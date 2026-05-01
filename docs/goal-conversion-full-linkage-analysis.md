# Plausible Analytics 目标转化全链路深度分析

## 1. 概述

本文档深入分析 Plausible Analytics 中目标转化功能的完整链路，重点关注：

1. **事件上报全链路**：从前端上报到 ClickHouse 入库的完整流程
2. **目标去重规则**：同名事件在自定义属性不同情况下的判重方式
3. **目标匹配逻辑**：事件如何与配置的目标进行匹配并参与统计

---

## 2. 事件上报全链路

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              前端 Tracker 层                                  │
│  tracker/src/plausible.js                                                    │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                     │
│  │  pageview   │    │ custom event│    │  engagement │                     │
│  │  (页面访问)  │    │ (自定义事件) │    │  (滚动/停留) │                     │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘                     │
│         └──────────────────┼──────────────────┘                            │
│                            ▼                                                 │
│              POST /api/event (JSON body)                                    │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              控制器层                                        │
│  lib/plausible_web/controllers/api/external_controller.ex                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ def event(conn, _params) do                                          │   │
│  │   {:ok, request, conn} <- Ingestion.Request.build(conn)              │   │
│  │   Ingestion.Event.build_and_buffer(request)                          │   │
│  │ end                                                                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              事件处理 Pipeline                                │
│  lib/plausible/ingestion/event.ex                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ defp pipeline() do                                                    │   │
│  │   [                                                                   │   │
│  │     drop_verification_agent: &drop_verification_agent/2,           │   │
│  │     drop_datacenter_ip: &drop_datacenter_ip/2,                       │   │
│  │     drop_threat_ip: &drop_threat_ip/2,                               │   │
│  │     put_geolocation: &put_geolocation/2,                              │   │
│  │     put_user_agent: &put_user_agent/2,                                │   │
│  │     put_basic_info: &put_basic_info/2,                                │   │
│  │     put_props: &put_props/2,                    ← 自定义属性处理      │   │
│  │     put_revenue: &put_revenue/2,                                      │   │
│  │     put_user_id: &put_user_id/2,                                      │   │
│  │     register_session: &register_session/2       ← 持久化到 ClickHouse │   │
│  │   ]                                                                   │   │
│  │ end                                                                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              持久化层                                        │
│  lib/plausible/ingestion/persistor.ex                                       │
│  lib/plausible/clickhouse_event_v2.ex                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 写入 ClickHouse 的 events_v2 表                                       │   │
│  │  - name: 事件名称 (pageview, engagement, 自定义事件名)                │   │
│  │  - meta.key: 自定义属性键数组                                          │   │
│  │  - meta.value: 自定义属性值数组                                        │   │
│  │  - pathname: 页面路径                                                  │   │
│  │  - scroll_depth: 滚动深度                                              │   │
│  │  - user_id/session_id: 用户/会话标识                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 前端事件上报

#### 2.2.1 事件类型

Plausible Tracker 支持以下事件类型：

| 事件类型 | 触发条件 | 事件名称 | 说明 |
|---------|---------|---------|------|
| **页面访问** | 页面加载/路由变化 | `pageview` | 基础页面访问事件 |
| **自定义事件** | 调用 `plausible('EventName')` | 用户定义 | 如 `Signup`, `Purchase` 等 |
| **滚动事件** | 页面滚动达到阈值 | `engagement` | 包含滚动深度 |
| ** outbound links** | 点击外部链接 | `Outbound Link: Click` | 自动追踪 |
| **文件下载** | 点击下载链接 | `File Download` | 自动追踪 |
| **表单提交** | 表单提交 | `Form: Submission` | 自动追踪 |
| **404 页面** | 访问不存在页面 | `404` | 自动追踪 |

#### 2.2.2 自定义事件上报示例

```javascript
// 基础自定义事件
plausible('Signup');

// 带自定义属性的事件
plausible('Purchase', {
  props: {
    product: 'Premium Plan',
    amount: '99',
    currency: 'USD'
  }
});
```

### 2.3 事件接收与处理

#### 2.3.1 控制器入口

事件通过 `/api/event` 端点接收：

```elixir
# lib/plausible_web/router.ex:417
scope "/api", PlausibleWeb do
  pipe_through :external_api
  post "/event", Api.ExternalController, :event
  # ...
end
```
[router.ex:417](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible_web/router.ex#L417)

控制器处理逻辑：

```elixir
# lib/plausible_web/controllers/api/external_controller.ex:13-45
def event(conn, _params) do
  with {:ok, request, conn} <- Ingestion.Request.build(conn),
       _ <- Sentry.Context.set_extra_context(%{request: request}) do
    case Ingestion.Event.build_and_buffer(request) do
      {:ok, %{dropped: [], buffered: _buffered}} ->
        conn
        |> put_status(202)
        |> text("ok")
      # ... 错误处理
    end
  end
end
```
[external_controller.ex:13-45](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible_web/controllers/api/external_controller.ex#L13-L45)

### 2.4 事件标准化 Pipeline

事件处理 Pipeline 定义在 `lib/plausible/ingestion/event.ex` 中，包含以下步骤：

#### 2.4.1 Pipeline 步骤详解

```elixir
# lib/plausible/ingestion/event.ex:130-151
defp pipeline() do
  [
    drop_verification_agent: &drop_verification_agent/2,
    drop_datacenter_ip: &drop_datacenter_ip/2,
    drop_threat_ip: &drop_threat_ip/2,
    drop_shield_rule_hostname: &drop_shield_rule_hostname/2,
    drop_shield_rule_page: &drop_shield_rule_page/2,
    drop_shield_rule_ip: &drop_shield_rule_ip/2,
    put_geolocation: &put_geolocation/2,
    drop_shield_rule_country: &drop_shield_rule_country/2,
    put_user_agent: &put_user_agent/2,
    put_basic_info: &put_basic_info/2,
    put_source_info: &put_source_info/2,
    maybe_infer_medium: &maybe_infer_medium/2,
    put_props: &put_props/2,              # 自定义属性处理
    put_revenue: &put_revenue/2,
    put_salts: &put_salts/2,
    put_user_id: &put_user_id/2,
    validate_clickhouse_event: &validate_clickhouse_event/2,
    register_session: &register_session/2  # 持久化
  ]
end
```
[event.ex:130-151](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/ingestion/event.ex#L130-L151)

#### 2.4.2 自定义属性标准化

自定义属性的处理是关键步骤，将上报的属性对象转换为有序数组：

```elixir
# lib/plausible/ingestion/event.ex:362-372
defp put_props(%__MODULE__{request: %{props: %{} = props}} = event, _context) do
  # defensive: ensuring the keys/values are always in the same order
  {keys, values} = Enum.unzip(props)

  update_event_attrs(event, %{
    "meta.key": keys,
    "meta.value": values
  })
end

defp put_props(%__MODULE__{} = event, _context), do: event
```
[event.ex:362-372](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/ingestion/event.ex#L362-L372)

**关键点**：
- 使用 `Enum.unzip/1` 确保键值对的顺序一致
- 存储为两个并行数组：`meta.key` 和 `meta.value`
- 这种存储方式便于 ClickHouse 进行数组索引查询

#### 2.4.3 基本信息填充

```elixir
# lib/plausible/ingestion/event.ex:278-290
defp put_basic_info(%__MODULE__{} = event, _context) do
  update_event_attrs(event, %{
    domain: event.domain,
    site_id: event.site.id,
    timestamp: event.request.timestamp,
    name: event.request.event_name,      # 事件名称
    hostname: event.request.hostname,
    pathname: event.request.pathname,
    scroll_depth: event.request.scroll_depth,
    engagement_time: event.request.engagement_time,
    interactive?: event.request.interactive?
  })
end
```
[event.ex:278-290](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/ingestion/event.ex#L278-L290)

### 2.5 事件数据结构

事件最终存储在 ClickHouse 的 `events_v2` 表中，Schema 定义如下：

```elixir
# lib/plausible/clickhouse_event_v2.ex:8-54
schema "events_v2" do
  field :name, Ch, type: "LowCardinality(String)"
  field :site_id, Ch, type: "UInt64"
  field :hostname, :string
  field :pathname, :string
  field :user_id, Ch, type: "UInt64"
  field :session_id, Ch, type: "UInt64"
  field :timestamp, :naive_datetime

  # 自定义属性 - 并行数组存储
  field :"meta.key", {:array, :string}
  field :"meta.value", {:array, :string}

  field :scroll_depth, Ch, type: "UInt8"
  field :engagement_time, Ch, type: "UInt32"

  # 收入相关（企业版）
  field :revenue_source_amount, Ch, type: "Nullable(Decimal64(3))"
  field :revenue_source_currency, Ch, type: "FixedString(3)"

  # 会话属性（反规范化存储）
  field :referrer, :string
  field :referrer_source, :string
  field :utm_medium, :string
  field :utm_source, :string
  field :utm_campaign, :string
  # ... 更多会话属性

  field :country_code, Ch, type: "FixedString(2)"
  field :screen_size, Ch, type: "LowCardinality(String)"
  field :operating_system, Ch, type: "LowCardinality(String)"
  field :browser, Ch, type: "LowCardinality(String)"
  # ...
end
```
[clickhouse_event_v2.ex:8-54](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/clickhouse_event_v2.ex#L8-L54)

---

## 3. 目标去重规则详解

### 3.1 唯一约束的演进

目标去重规则通过数据库唯一约束实现，经历了多次迁移演进：

#### 3.1.1 初始约束（无自定义属性）

```elixir
# 早期版本的约束
unique_index(:goals, [:site_id, :event_name], name: :goals_event_name_unique)
unique_index(:goals, [:site_id, :page_path, :scroll_threshold], 
  name: :goals_page_path_and_scroll_threshold_unique)
```

#### 3.1.2 添加自定义属性支持（2025.12.09 迁移）

```elixir
# priv/repo/migrations/20251209120138_goals_custom_props.exs:10-40
def change do
  # 删除旧约束
  drop(unique_index(:goals, [:site_id, :event_name], name: :goals_event_name_unique))
  drop(unique_index(:goals, [:site_id, :page_path, :scroll_threshold],
    name: :goals_page_path_and_scroll_threshold_unique))

  # 添加 custom_props 字段
  alter table(:goals) do
    add(:custom_props, :map)
  end

  # 创建新的唯一约束 - 包含 custom_props
  create(
    unique_index(:goals, [:site_id, :event_name, :custom_props],
      where: "event_name IS NOT NULL",
      name: :goals_event_config_unique
    )
  )

  create(
    unique_index(:goals, [:site_id, :page_path, :scroll_threshold],
      where: "page_path IS NOT NULL",
      name: :goals_pageview_config_unique
    )
  )

  create(unique_index(:goals, [:site_id, :display_name], name: :goals_display_name_unique))
end
```
[20251209120138_goals_custom_props.exs:10-40](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/priv/repo/migrations/20251209120138_goals_custom_props.exs#L10-L40)

#### 3.1.3 页面目标也添加自定义属性约束（2026.01.05 迁移）

```elixir
# priv/repo/migrations/20260105075211_update_goals_pageview_config_unique_constraint.exs:7-22
def change do
  drop(
    unique_index(:goals, [:site_id, :page_path, :scroll_threshold],
      where: "page_path IS NOT NULL",
      name: :goals_pageview_config_unique
    )
  )

  create(
    unique_index(:goals, [:site_id, :page_path, :scroll_threshold, :custom_props],
      where: "page_path IS NOT NULL",
      name: :goals_pageview_config_unique
    )
  )
end
```
[20260105075211_update_goals_pageview_config_unique_constraint.exs:7-22](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/priv/repo/migrations/20260105075211_update_goals_pageview_config_unique_constraint.exs#L7-L22)

### 3.2 最终唯一约束

经过演进后，目标表有三个独立的唯一约束：

| 约束名称 | 约束字段 | 适用条件 | 说明 |
|---------|---------|---------|------|
| `goals_event_config_unique` | `site_id`, `event_name`, `custom_props` | `event_name IS NOT NULL` | 事件目标去重 |
| `goals_pageview_config_unique` | `site_id`, `page_path`, `scroll_threshold`, `custom_props` | `page_path IS NOT NULL` | 页面/滚动目标去重 |
| `goals_display_name_unique` | `site_id`, `display_name` | 无条件 | 显示名称全局唯一 |

### 3.3 去重规则详解

#### 3.3.1 事件目标去重规则

**判断逻辑**：同一站点内，**事件名称 + 自定义属性组合** 必须唯一

```elixir
# lib/plausible/goal.ex:58-59
|> unique_constraint(:event_name, name: :goals_event_config_unique)
```
[goal.ex:58-59](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/goal.ex#L58-L59)

**示例场景**：

| 场景 | 事件名 | 自定义属性 | 结果 |
|-----|-------|-----------|------|
| 场景 1 | `Purchase` | `{}` | ✅ 创建成功 |
| 场景 2 | `Purchase` | `{product: "tablet"}` | ✅ 创建成功（不同属性） |
| 场景 3 | `Purchase` | `{product: "speaker"}` | ✅ 创建成功（不同属性值） |
| 场景 4 | `Purchase` | `{product: "tablet"}` | ❌ 报错：已存在（完全相同） |
| 场景 5 | `Purchase` | `{product: "tablet", color: "black"}` | ✅ 创建成功（更多属性） |

**测试用例验证**：

```elixir
# test/plausible/goals_test.exs:110-132
test "create/2 succeeds to create the same custom event goal thrice with different custom props and different display names each" do
  site = new_site()

  {:ok, _} =
    Goals.create(site, %{
      "event_name" => "Purchase",
      "display_name" => "Tablet Purchase",
      "custom_props" => %{"product" => "tablet"}
    })

  {:ok, _} =
    Goals.create(site, %{
      "event_name" => "Purchase",
      "display_name" => "Speaker Purchase",
      "custom_props" => %{"product" => "speaker"}
    })

  {:ok, _} =
    Goals.create(site, %{
      "event_name" => "Purchase",
      "display_name" => "General Purchase"
    })
end
```
[goals_test.exs:110-132](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/test/plausible/goals_test.exs#L110-L132)

**重复检测验证**：

```elixir
# test/plausible/goals_test.exs:282-300
test "create/2 fails when same custom props config exists with different display name" do
  site = new_site()

  {:ok, _} =
    Goals.create(site, %{
      "event_name" => "Purchase",
      "display_name" => "Purchase Event",
      "custom_props" => %{"product" => "tablet"}
    })

  {:error, changeset} =
    Goals.create(site, %{
      "event_name" => "Purchase",
      "display_name" => "Different Display Name",
      "custom_props" => %{"product" => "tablet"}
    })

  assert {"has already been taken", _} = changeset.errors[:event_name]
end
```
[goals_test.exs:282-300](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/test/plausible/goals_test.exs#L282-L300)

#### 3.3.2 页面/滚动目标去重规则

**判断逻辑**：同一站点内，**页面路径 + 滚动阈值 + 自定义属性组合** 必须唯一

```elixir
# lib/plausible/goal.ex:60-62
|> unique_constraint([:page_path, :scroll_threshold],
  name: :goals_pageview_config_unique
)
```
[goal.ex:60-62](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/goal.ex#L60-L62)

**示例场景**：

| 场景 | 页面路径 | 滚动阈值 | 自定义属性 | 结果 |
|-----|---------|---------|-----------|------|
| 场景 1 | `/blog` | `-1` (页面目标) | `{}` | ✅ 创建成功 |
| 场景 2 | `/blog` | `50` (滚动目标) | `{}` | ✅ 创建成功（不同阈值） |
| 场景 3 | `/blog` | `-1` | `{variant: "A"}` | ✅ 创建成功（不同属性） |
| 场景 4 | `/blog` | `-1` | `{}` | ❌ 报错：已存在 |

**测试用例验证**：

```elixir
# test/plausible/goals_test.exs:244-255
test "create/2 succeeds to create two pageview goals with different displayn names and custom props each" do
  site = new_site()

  {:ok, _} = Goals.create(site, %{"page_path" => "/index", "display_name" => "Index"})

  {:ok, _} =
    Goals.create(site, %{
      "page_path" => "/index",
      "display_name" => "Index 2",
      "custom_props" => %{"foo" => "bar"}
    })
end
```
[goals_test.exs:244-255](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/test/plausible/goals_test.exs#L244-L255)

#### 3.3.3 显示名称全局唯一

**判断逻辑**：同一站点内，**显示名称** 必须全局唯一，与目标类型无关

```elixir
# lib/plausible/goal.ex:58
|> unique_constraint(:display_name, name: :goals_display_name_unique)
```
[goal.ex:58](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/goal.ex#L58)

**示例场景**：

| 场景 | 目标类型 | 事件名/路径 | 显示名称 | 结果 |
|-----|---------|------------|---------|------|
| 场景 1 | 事件目标 | `Signup` | `User Action` | ✅ 创建成功 |
| 场景 2 | 页面目标 | `/signup` | `User Action` | ❌ 报错：显示名称已存在 |

**测试用例验证**：

```elixir
# test/plausible/goals_test.exs:322-338
test "create/2 fails when same display name exists between event and pageview goals" do
  site = new_site()

  {:ok, _} =
    Goals.create(site, %{
      "event_name" => "Signup",
      "display_name" => "User Action"
    })

  {:error, changeset} =
    Goals.create(site, %{
      "page_path" => "/signup",
      "display_name" => "User Action"
    })

  assert {"has already been taken", _} = changeset.errors[:display_name]
end
```
[goals_test.exs:322-338](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/test/plausible/goals_test.exs#L322-L338)

### 3.4 自定义属性在去重中的特殊规则

#### 3.4.1 空属性 vs 无属性

**关键点**：`custom_props = %{}`（空 Map）和 `custom_props = nil`（未设置）在 PostgreSQL 中被视为不同的值

但是从测试用例来看：

```elixir
# test/plausible/goals_test.exs:134-150
test "create/2 fails to create the same custom event goal twice with different display names but no props each" do
  site = new_site()

  {:ok, _} =
    Goals.create(site, %{
      "event_name" => "Purchase",
      "display_name" => "General Purchase"
    })

  {:error, changeset} =
    Goals.create(site, %{
      "event_name" => "Purchase",
      "display_name" => "General Purchase 2"
    })

  assert {"has already been taken", _} = changeset.errors[:event_name]
end
```
[goals_test.exs:134-150](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/test/plausible/goals_test.exs#L134-L150)

**分析**：
- 未显式设置 `custom_props` 时，使用默认值 `%{}`
- Schema 定义：`field :custom_props, :map, default: %{}`
- 因此两次创建都使用 `custom_props = %{}`，触发唯一约束

#### 3.4.2 属性数量/顺序影响

从迁移和约束定义来看：

- PostgreSQL 的唯一约束对 JSONB 类型进行值比较
- `{a: 1, b: 2}` 和 `{b: 2, a: 1}` 被视为相同（JSONB 不保留键顺序）
- `{a: 1}` 和 `{a: 1, b: 2}` 被视为不同

**测试用例验证**：

```elixir
# test/plausible/goals_test.exs:226-242
test "create/2 succeeds to create the same custom event twice with different props and different display names each" do
  site = new_site()

  {:ok, _} =
    Goals.create(site, %{
      "event_name" => "Purchase",
      "display_name" => "General Purchase",
      "custom_props" => %{"variant" => "A"}
    })

  {:ok, _} =
    Goals.create(site, %{
      "event_name" => "Purchase",
      "display_name" => "General Purchase 2",
      "custom_props" => %{"variant" => "A", "foo" => "bar"}
    })
end
```
[goals_test.exs:226-242](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/test/plausible/goals_test.exs#L226-L242)

---

## 4. 目标匹配逻辑

### 4.1 目标匹配架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           查询构建阶段                                        │
│  lib/plausible/stats/query_builder.ex                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ preload_goals_and_revenue(site, metrics, filters, dimensions)       │   │
│  │   └─► Goals.preload_needed_goals(site, dimensions, filters)         │   │
│  │       └─► 检测是否需要目标维度/过滤器                                  │   │
│  │           └─► 从数据库预加载所有目标                                  │   │
│  │               └─► 筛选匹配顶层过滤器的目标                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           WHERE 条件构建阶段                                  │
│  lib/plausible/stats/sql/where_builder.ex                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ add_filter(:events, query, [operation, "event:goal", clauses | _]) │   │
│  │   └─► Goals.add_filter(query, filter)                                │   │
│  │       └─► 从预加载目标中筛选匹配的目标                                │   │
│  │           └─► 为每个目标构建匹配条件                                  │   │
│  │               └─► 使用 OR 连接多个条件                               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           目标条件构建                                        │
│  lib/plausible/stats/goals.ex                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ goal_condition(goal, imported?)                                      │   │
│  │   ├─► :event  → e.name == event_name [AND custom_props 条件]        │   │
│  │   ├─► :page   → e.name == "pageview" AND pathname == page_path     │   │
│  │   │              [AND custom_props 条件]                             │   │
│  │   └─► :scroll → e.name == "engagement" AND pathname == page_path    │   │
│  │                  AND scroll_depth >= threshold [AND custom_props]    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 目标预加载

#### 4.2.1 预加载触发条件

```elixir
# lib/plausible/stats/goals.ex:14-34
def preload_needed_goals(site, dimensions, filters) do
  if Enum.member?(dimensions, "event:goal") or
       Filters.filtering_on_dimension?(filters, "event:goal") do
    site = Plausible.Repo.preload(site, :team)
    props_available? = Plausible.Billing.Feature.Props.check_availability(site.team) == :ok
    goals = Plausible.Goals.for_site(site, include_goals_with_custom_props?: props_available?)

    %{
      matching_toplevel_filters: goals_matching_toplevel_filters(goals, filters),
      all: goals
    }
  else
    %{all: [], matching_toplevel_filters: []}
  end
end
```
[goals.ex:14-34](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/goals.ex#L14-L34)

**触发条件**：
1. 查询维度包含 `"event:goal"`
2. 或查询过滤器包含 `"event:goal"` 维度

#### 4.2.2 目标筛选逻辑

```elixir
# lib/plausible/stats/goals.ex:146-161
defp goals_matching_toplevel_filters(goals, filters) do
  Enum.reduce(filters, goals, fn
    [_, "event:goal" | _] = filter, goals ->
      goals_matching_any_clause(goals, filter)

    _filter, goals ->
      goals
  end)
end

defp goals_matching_any_clause(goals, [_, _, clauses | _] = filter) do
  goals
  |> Enum.filter(fn goal ->
    Enum.any?(clauses, fn clause -> matches?(goal, filter, clause) end)
  end)
end
```
[goals.ex:146-161](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/goals.ex#L146-L161)

**目标匹配（按显示名称）**：

```elixir
# lib/plausible/stats/goals.ex:163-178
defp matches?(goal, [operation | _rest] = filter, clause) do
  goal_name =
    goal
    |> Plausible.Goal.display_name()
    |> mod(filter)

  clause = mod(clause, filter)

  case operation do
    :is ->
      goal_name == clause

    :contains ->
      String.contains?(goal_name, clause)
  end
end
```
[goals.ex:163-178](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/goals.ex#L163-L178)

**关键点**：
- 目标过滤是通过 **显示名称（display_name）** 匹配的
- 支持两种操作：`:is`（精确匹配）和 `:contains`（包含匹配）
- 这就是为什么显示名称需要全局唯一

### 4.3 目标条件构建

#### 4.3.1 事件目标条件

```elixir
# lib/plausible/stats/goals.ex:202-211
defp goal_condition(:event, goal, _) do
  name_condition = dynamic([e], e.name == ^goal.event_name)

  if Plausible.Goal.has_custom_props?(goal) do
    custom_props_condition = build_custom_props_condition(goal.custom_props)
    dynamic([e], ^name_condition and ^custom_props_condition)
  else
    name_condition
  end
end
```
[goals.ex:202-211](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/goals.ex#L202-L211)

#### 4.3.2 页面目标条件

```elixir
# lib/plausible/stats/goals.ex:234-245
defp goal_condition(:page, goal, false = _imported?) do
  name_condition = dynamic([e], e.name == "pageview")
  pathname_condition = page_path_condition(goal.page_path, _imported? = false)
  base_condition = dynamic([e], ^pathname_condition and ^name_condition)

  if Plausible.Goal.has_custom_props?(goal) do
    custom_props_condition = build_custom_props_condition(goal.custom_props)
    dynamic([e], ^base_condition and ^custom_props_condition)
  else
    base_condition
  end
end
```
[goals.ex:234-245](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/goals.ex#L234-L245)

#### 4.3.3 滚动目标条件

```elixir
# lib/plausible/stats/goals.ex:213-228
defp goal_condition(:scroll, goal, false = _imported?) do
  pathname_condition = page_path_condition(goal.page_path, _imported? = false)
  name_condition = dynamic([e], e.name == "engagement")

  scroll_condition =
    dynamic([e], e.scroll_depth <= 100 and e.scroll_depth >= ^goal.scroll_threshold)

  base_condition = dynamic([e], ^pathname_condition and ^name_condition and ^scroll_condition)

  if Plausible.Goal.has_custom_props?(goal) do
    custom_props_condition = build_custom_props_condition(goal.custom_props)
    dynamic([e], ^base_condition and ^custom_props_condition)
  else
    base_condition
  end
end
```
[goals.ex:213-228](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/goals.ex#L213-L228)

### 4.4 自定义属性匹配条件

**这是目标匹配的核心逻辑**：

```elixir
# lib/plausible/stats/goals.ex:257-273
defp build_custom_props_condition(custom_props) do
  Enum.reduce(custom_props, true, fn {prop_key, prop_value}, acc ->
    condition =
      dynamic(
        [e],
        fragment(
          "?[indexOf(?, ?)] = ?",
          field(e, :"meta.value"),
          field(e, :"meta.key"),
          ^prop_key,
          ^prop_value
        )
      )

    dynamic([e], ^acc and ^condition)
  end)
end
```
[goals.ex:257-273](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/goals.ex#L257-L273)

#### 4.4.1 ClickHouse 数组索引查询解析

这段代码生成的 SQL 片段非常关键，让我详细解析：

```sql
-- 假设目标定义了 custom_props = {product: "Premium Plan", variant: "A"}
-- 生成的 SQL 条件逻辑：

meta.value[indexOf(meta.key, 'product')] = 'Premium Plan'
AND
meta.value[indexOf(meta.key, 'variant')] = 'A'
```

**工作原理**：

1. **`meta.key`**：事件的自定义属性键数组，如 `["product", "variant", "amount"]`
2. **`meta.value`**：事件的自定义属性值数组，如 `["Premium Plan", "A", "99"]`
3. **`indexOf(meta.key, 'product')`**：在键数组中查找 `'product'` 的索引位置
4. **`meta.value[indexOf(...)]`**：使用该索引从值数组中获取对应的值
5. **`= 'Premium Plan'`**：比较值是否匹配

#### 4.4.2 匹配规则详解

| 事件的自定义属性 | 目标的自定义属性 | 匹配结果 | 说明 |
|----------------|----------------|---------|------|
| `{product: "Premium"}` | `{product: "Premium"}` | ✅ 匹配 | 完全匹配 |
| `{product: "Premium", variant: "A"}` | `{product: "Premium"}` | ✅ 匹配 | 目标属性是事件属性的子集 |
| `{product: "Premium"}` | `{product: "Premium", variant: "A"}` | ❌ 不匹配 | 事件缺少 `variant` 属性 |
| `{product: "Basic"}` | `{product: "Premium"}` | ❌ 不匹配 | 属性值不同 |
| `{product: "Premium", variant: "A"}` | `{product: "Premium", variant: "B"}` | ❌ 不匹配 | 某个属性值不同 |
| `{}`（无自定义属性） | `{product: "Premium"}` | ❌ 不匹配 | 事件缺少属性 |
| `{product: "Premium"}` | `{}`（无属性） | ✅ 匹配 | 目标无属性条件，只匹配事件名 |

**关键点**：
- 目标的自定义属性条件是 **"所有定义的属性必须存在且值匹配"**
- 事件可以有更多属性，只要包含目标定义的所有属性即可
- 无自定义属性的目标只匹配事件名/路径

### 4.5 多个目标的 OR 连接

当查询过滤多个目标时，使用 OR 连接：

```elixir
# lib/plausible/stats/goals.ex:52-64
def add_filter(query, [operation, "event:goal", clauses | _] = filter, opts \\ [])
    when operation in [:is, :contains] do
  imported? = Keyword.get(opts, :imported?, false)

  Enum.reduce(clauses, false, fn clause, dynamic_statement ->
    condition =
      query.preloaded_goals.all
      |> filter_preloaded(filter, clause)
      |> build_condition(imported?)

    dynamic([e], ^condition or ^dynamic_statement)
  end)
end
```
[goals.ex:52-64](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/goals.ex#L52-L64)

**示例**：
- 查询过滤目标 `["Tablet Purchase", "Speaker Purchase"]`
- 生成的 SQL 条件：`(event_name='Purchase' AND product='tablet') OR (event_name='Purchase' AND product='speaker')`

---

## 5. 事件参与统计查询

### 5.1 查询执行流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         查询入口                                             │
│  lib/plausible/stats/query_runner.ex:run/2                                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SQL 构建                                             │
│  lib/plausible/stats/sql/query_builder.ex:build/2                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 1. 构建基础查询 (Base.base_event_query)                               │   │
│  │    └─► WHERE site_id = ? AND timestamp BETWEEN ? AND ?               │   │
│  │                                                                       │   │
│  │ 2. 添加过滤器 (WhereBuilder.build)                                    │   │
│  │    └─► 处理 event:goal 过滤器 → 调用 Goals.add_filter                 │   │
│  │                                                                       │   │
│  │ 3. 构建 GROUP BY                                                       │   │
│  │    └─► 如果维度是 event:goal → 特殊处理                                │   │
│  │                                                                       │   │
│  │ 4. 添加特殊指标 (SpecialMetrics.add)                                   │   │
│  │    └─► conversion_rate, group_conversion_rate 等                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ClickHouse 执行                                       │
│  执行 SQL 查询，返回聚合结果                                                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         结果转换                                             │
│  lib/plausible/stats/query_runner.ex:build_from_ch/2                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ - 维度标签转换：goal 索引 → 目标 display_name                         │   │
│  │ - 指标值提取：从查询结果中获取 metrics                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 按目标维度分组

当查询维度包含 `"event:goal"` 时，需要特殊的 GROUP BY 处理：

```elixir
# lib/plausible/stats/sql/query_builder.ex:173-197
defp dimension_group_by(q, :events, query, "event:goal" = dimension) do
  goal_join_data = Plausible.Stats.Goals.goal_join_data(query)

  if Enum.all?(goal_join_data.custom_props_keys, &Enum.empty?/1) do
    from(e in q,
      join: goal in Expression.event_goal_join_no_props(goal_join_data),
      hints: "ARRAY",
      on: true,
      select_merge: %{
        ^shortname(query, dimension) => fragment("?", goal)
      },
      group_by: goal
    )
  else
    from(e in q,
      join: goal in Expression.event_goal_join(goal_join_data),
      hints: "ARRAY",
      on: true,
      select_merge: %{
        ^shortname(query, dimension) => fragment("?", goal)
      },
      group_by: goal
    )
  end
end
```
[sql/query_builder.ex:173-197](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/sql/query_builder.ex#L173-L197)

### 5.3 目标分组数据结构

```elixir
# lib/plausible/stats/goals.ex:66-75
@type goal_join_data() :: %{
        indices: [non_neg_integer()],
        types: [String.t()],
        event_names_imports: [String.t()],
        event_names_by_type: [String.t()],
        page_regexes: [String.t()],
        scroll_thresholds: [non_neg_integer()],
        custom_props_keys: [[String.t()]],
        custom_props_values: [[String.t()]]
      }
```
[goals.ex:66-75](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/goals.ex#L66-L75)

### 5.4 结果转换 - 目标索引到名称

```elixir
# lib/plausible/stats/query_runner.ex:164-215
defp dimension_label("event:goal", entry, query) do
  get_dimension_goal(entry, query)
    |> Plausible.Goal.display_name()
end

defp get_dimension_goal(entry, query) do
  goal_index = Map.get(entry, Util.shortname(query, "event:goal"))

  query.preloaded_goals.matching_toplevel_filters
    |> Enum.at(goal_index - 1)
end
```
[query_runner.ex:164-215](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/query_runner.ex#L164-L215)

---

## 6. 完整示例：从事件上报到目标统计

### 6.1 场景描述

假设我们有一个电商网站，配置了以下目标：

| 目标名称 | 类型 | 事件名/路径 | 自定义属性 | 说明 |
|---------|------|------------|-----------|------|
| `Tablet Purchase` | 事件目标 | `Purchase` | `{product: "tablet"}` | 平板购买 |
| `Speaker Purchase` | 事件目标 | `Purchase` | `{product: "speaker"}` | 音箱购买 |
| `General Purchase` | 事件目标 | `Purchase` | `{}` | 任意购买 |
| `Checkout Page` | 页面目标 | `/checkout` | `{}` | 访问结算页 |

### 6.2 事件上报

```javascript
// 用户 1 购买平板
plausible('Purchase', {
  props: { product: 'tablet', amount: '299' }
});

// 用户 2 购买音箱
plausible('Purchase', {
  props: { product: 'speaker', amount: '199' }
});

// 用户 3 访问结算页（没有购买）
// 自动上报 pageview 事件，pathname = '/checkout'
```

### 6.3 事件存储

这些事件在 ClickHouse 中的存储：

| name | meta.key | meta.value | pathname |
|------|----------|------------|----------|
| `Purchase` | `["product", "amount"]` | `["tablet", "299"]` | `/checkout` |
| `Purchase` | `["product", "amount"]` | `["speaker", "199"]` | `/checkout` |
| `pageview` | `[]` | `[]` | `/checkout` |

### 6.4 查询目标统计

**查询**：按目标维度统计转化率

```elixir
# 查询参数
%{
  metrics: [:visitors, :conversion_rate],
  dimensions: ["event:goal"],
  filters: [[:is, "event:goal", ["Tablet Purchase", "Speaker Purchase", "Checkout Page"]]]
}
```

**执行流程**：

1. **预加载目标**：从数据库加载所有目标，筛选匹配显示名称的目标
   - `Tablet Purchase` → `event_name='Purchase', custom_props={product: "tablet"}`
   - `Speaker Purchase` → `event_name='Purchase', custom_props={product: "speaker"}`
   - `Checkout Page` → `page_path='/checkout', scroll_threshold=-1, custom_props={}`

2. **构建 WHERE 条件**：

```sql
-- 目标 1: Tablet Purchase
(e.name = 'Purchase' 
 AND meta.value[indexOf(meta.key, 'product')] = 'tablet')

OR

-- 目标 2: Speaker Purchase
(e.name = 'Purchase' 
 AND meta.value[indexOf(meta.key, 'product')] = 'speaker')

OR

-- 目标 3: Checkout Page
(e.name = 'pageview' AND e.pathname = '/checkout')
```

3. **构建 GROUP BY**：使用 ClickHouse 数组 JOIN 技术，将事件按匹配的目标分组

4. **计算转化率**：

```sql
-- 总访客数（移除目标过滤器）
SELECT uniq(user_id) as total_visitors FROM events_v2 WHERE ...

-- 转化率 = 目标访客数 / 总访客数 * 100
```

### 6.5 查询结果

| 目标 | visitors | conversion_rate |
|-----|----------|-----------------|
| `Tablet Purchase` | 1 | 33.33 |
| `Speaker Purchase` | 1 | 33.33 |
| `Checkout Page` | 3 | 100.00 |

**注意**：`General Purchase` 目标（无自定义属性）会匹配所有 `Purchase` 事件，但在这个查询中没有被过滤，所以不显示。

---

## 7. 关键设计要点总结

### 7.1 事件存储设计

| 设计选择 | 原因 |
|---------|------|
| **并行数组存储自定义属性** (`meta.key` + `meta.value`) | ClickHouse 列式存储优化，便于数组索引查询 |
| **事件名低基数化** (`LowCardinality(String)`) | 减少存储，提高查询性能 |
| **会话属性反规范化** | 避免 JOIN，提高查询性能 |
| **scroll_depth 单独字段** | 滚动目标专用，便于范围查询 |

### 7.2 目标去重设计

| 设计选择 | 原因 |
|---------|------|
| **三级唯一约束** | 分别处理事件目标、页面目标、显示名称的唯一性 |
| **custom_props 参与唯一约束** | 允许同名事件不同属性定义为不同目标 |
| **显示名称全局唯一** | 目标过滤通过显示名称匹配，必须唯一 |

### 7.3 目标匹配设计

| 设计选择 | 原因 |
|---------|------|
| **目标预加载** | 减少数据库查询，在应用层处理目标匹配 |
| **显示名称匹配** | 用户友好，避免暴露内部实现 |
| **自定义属性子集匹配** | 目标定义的属性必须全部存在，事件可以有更多属性 |
| **ClickHouse 数组索引查询** | `meta.value[indexOf(meta.key, 'key')]` 高效匹配并行数组 |
| **多目标 OR 连接** | 符合用户预期，任意目标匹配即可 |

### 7.4 自定义属性的关键作用

自定义属性在整个链路中的作用：

1. **事件上报**：用户可以附加任意键值对
2. **事件存储**：转换为并行数组，便于 ClickHouse 查询
3. **目标定义**：参与唯一约束，同名事件不同属性是不同目标
4. **目标匹配**：使用数组索引查询，精确匹配属性值
5. **统计查询**：作为过滤条件和聚合维度

---

## 8. 相关文件索引

| 文件路径 | 功能描述 |
|---------|---------|
| `tracker/src/plausible.js` | 前端 Tracker，事件上报 |
| `lib/plausible_web/router.ex:417` | 事件路由定义 |
| `lib/plausible_web/controllers/api/external_controller.ex` | 事件控制器 |
| `lib/plausible/ingestion/event.ex` | 事件处理 Pipeline |
| `lib/plausible/ingestion/persistor.ex` | 事件持久化 |
| `lib/plausible/clickhouse_event_v2.ex` | 事件数据结构 |
| `lib/plausible/goal.ex` | 目标数据模型、唯一约束 |
| `lib/plausible/goals/goals.ex` | 目标 CRUD、去重逻辑 |
| `lib/plausible/stats/goals.ex` | 目标预加载、匹配条件构建 |
| `lib/plausible/stats/sql/where_builder.ex` | WHERE 条件构建 |
| `lib/plausible/stats/sql/query_builder.ex` | SQL 查询构建 |
| `lib/plausible/stats/query_runner.ex` | 查询执行、结果转换 |
| `priv/repo/migrations/20251209120138_goals_custom_props.exs` | 自定义属性约束迁移 |
| `priv/repo/migrations/20260105075211_update_goals_pageview_config_unique_constraint.exs` | 页面目标约束更新 |
