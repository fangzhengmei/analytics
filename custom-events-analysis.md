# 自定义事件、属性和 Revenue 指标查询链路分析报告

## 1. 概述

本文档详细分析了 Plausible Analytics 中自定义事件、自定义属性和 Revenue 指标的完整查询链路，包括采集校验、存储映射、报表聚合和扩展成本四个核心环节。重点澄清了 **Revenue 事件只有命中对应转化目标才会入库** 的关键逻辑，以及源币种与目标币种一致和不一致两种分支的处理方式。

---

## 2. 采集校验环节

### 2.1 前端数据采集

**核心文件**: `tracker/src/track.js`

#### 2.1.1 事件采集流程

前端通过 `track()` 函数采集事件数据，支持以下参数：

| 参数 | 说明 | 对应字段 |
|------|------|----------|
| `eventName` | 事件名称（如 'pageview', 'signup'） | `n` / `name` |
| `options.u` / `options.url` | 自定义 URL | `u` |
| `options.props` | 自定义属性 | `p` / `props` |
| `options.revenue` | Revenue 数据 | `$` / `revenue` |
| `options.interactive` | 是否为交互事件 | `i` |

#### 2.1.2 Revenue 数据格式

前端发送 Revenue 数据时需要包含 `amount` 和 `currency` 两个字段：

```javascript
// 示例：追踪一个购买事件
plausible.track('Purchase', {
  revenue: {
    amount: 99.99,
    currency: 'USD'
  }
})
```

#### 2.1.3 关键校验点

```javascript
// 本地环境过滤
if (/^localhost$|^127(\.[0-9]+){0,2}\.[0-9]+$|^\[::1?\]$/.test(location.hostname)) {
  return onIgnoredEvent(eventName, options, 'localhost')
}

// 爬虫/自动化工具过滤
if ((window._phantom || window.__nightmare || window.navigator.webdriver || window.Cypress) && !window.__plausible) {
  return onIgnoredEvent(eventName, options)
}

// localStorage 忽略标记
if (window.localStorage.plausible_ignore === 'true') {
  return onIgnoredEvent(eventName, options, 'localStorage flag')
}
```

### 2.2 后端请求校验

**核心文件**: `lib/plausible/ingestion/request.ex`

#### 2.2.1 请求构建流程

```elixir
def build(%Plug.Conn{} = conn, now \\ NaiveDateTime.utc_now()) do
  changeset =
    %__MODULE__{}
    |> Changeset.change()
    |> Changeset.put_change(:timestamp, NaiveDateTime.truncate(now, :second))

  case parse_body(conn) do
    {:ok, request_body, conn} ->
      request =
        changeset
        |> put_ip_classification(conn)
        |> put_remote_ip(conn)
        |> put_uri(request_body)
        |> put_hostname()
        |> put_user_agent(conn)
        |> put_request_params(request_body)
        |> put_referrer(request_body)
        |> put_pathname()
        |> put_props(request_body)        # 自定义属性处理
        |> put_engagement_fields(request_body)
        |> put_query_params()
        |> put_revenue_source(request_body) # Revenue 处理
        |> put_interactive(request_body)
        |> put_tracker_script_version(request_body)
        |> map_domains(request_body)
        |> Changeset.validate_required([:event_name, :hostname, :pathname, :timestamp])
        |> Changeset.validate_length(:event_name, max: 120)  # 事件名称长度限制
        |> Changeset.apply_action(nil)
      ...
  end
end
```

#### 2.2.2 自定义属性校验

```elixir
@max_props 30  # 每个事件最多 30 个属性
@max_prop_key_length Plausible.Props.max_prop_key_length()  # 300 字节
@max_prop_value_length Plausible.Props.max_prop_value_length()  # 2000 字节

defp put_props(changeset, %{} = request_body) do
  props =
    (request_body["m"] || request_body["meta"] || request_body["p"] || request_body["props"])
    |> Plausible.Helpers.JSON.decode_or_fallback()
    |> Enum.reduce([], &filter_bad_props/2)  # 过滤无效属性
    |> maybe_set_props_path_to_pathname(changeset)
    |> Enum.take(@max_props)  # 限制数量
    |> Map.new()

  changeset
  |> Changeset.put_change(:props, props)
  |> validate_props()  # 校验键值长度
end

defp filter_bad_props({k, v}, acc) do
  cond do
    # 过滤列表和映射类型
    Enum.any?([k, v], &(is_list(&1) or is_map(&1))) -> acc
    # 过滤空字符串
    Enum.any?([k, v], &(String.trim_leading(to_string(&1)) == "")) -> acc
    true -> [{to_string(k), to_string(v)} | acc]
  end
end
```

#### 2.2.3 Revenue 数据校验

**核心文件**: `extra/lib/plausible/ingestion/request/revenue.ex`

```elixir
def put_revenue_source(%Ecto.Changeset{} = changeset, %{} = request_body) do
  with revenue_source <- request_body["revenue"] || request_body["$"],
       %{"amount" => _, "currency" => _} = revenue_source <-
         Plausible.Helpers.JSON.decode_or_fallback(revenue_source) do
    parse_revenue_source(changeset, revenue_source)
  else
    _any -> changeset
  end
end

@valid_currencies Plausible.Goal.Revenue.valid_currencies()
defp parse_revenue_source(changeset, %{"amount" => amount, "currency" => currency}) do
  with true <- currency in @valid_currencies,  # 校验货币有效性
       {%Decimal{} = amount, _rest} <- parse_decimal(amount),  # 解析金额
       %Money{} = amount <- Money.new(currency, amount) do
    Ecto.Changeset.put_change(changeset, :revenue_source, amount)
  else
    _any -> changeset
  end
end
```

**⚠️ 重要注意**: 此阶段仅校验 Revenue 数据的格式有效性，**并不保证数据会入库**。是否入库取决于后续的「目标命中检查」。

### 2.3 校验规则汇总

| 数据类型 | 校验规则 | 限制值 |
|----------|----------|--------|
| 事件名称 | 必填、长度限制 | 最大 120 字节 |
| 自定义属性数量 | 每个事件最多 | 30 个 |
| 属性键长度 | 字节限制 | 最大 300 字节 |
| 属性值长度 | 字节限制 | 最大 2000 字节 |
| 属性类型 | 仅允许简单类型 | 不允许列表/映射 |
| Revenue 货币 | 必须为有效货币 | ISO 4217 代码 |
| Revenue 金额 | 必须可解析为 Decimal | 支持整数/浮点数/字符串 |

---

## 3. 存储映射环节

### 3.1 事件处理管道

**核心文件**: `lib/plausible/ingestion/event.ex`

#### 3.1.1 处理流程

```elixir
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
    put_props: &put_props/2,              # 自定义属性映射
    put_revenue: &put_revenue/2,          # Revenue 映射 - ⚠️ 关键步骤
    put_salts: &put_salts/2,
    put_user_id: &put_user_id/2,
    validate_clickhouse_event: &validate_clickhouse_event/2,
    register_session: &register_session/2
  ]
end
```

#### 3.1.2 自定义属性存储映射

```elixir
defp put_props(%__MODULE__{request: %{props: %{} = props}} = event, _context) do
  # 确保键值顺序一致
  {keys, values} = Enum.unzip(props)

  update_event_attrs(event, %{
    "meta.key": keys,
    "meta.value": values
  })
end
```

**存储设计**: 自定义属性采用**两个并行数组**存储，而非键值对结构，这是 ClickHouse 列式存储的最佳实践。

### 3.2 Revenue 存储映射（重写版）

**核心文件**: `extra/lib/plausible/ingestion/event/revenue.ex`

#### 3.2.1 关键逻辑：目标命中检查

**⚠️ 核心发现**: Revenue 数据**只有在事件命中对应配置的 Revenue Goal 时才会入库**。

```elixir
def get_revenue_attrs(
      %Plausible.Ingestion.Event{
        request: %{revenue_source: %Money{} = revenue_source}
      } = event
    ) do
  # 🔴 关键步骤：检查事件名称是否匹配任何已配置的 Revenue Goal
  matching_goal =
    Enum.find(
      event.site.revenue_goals,
      &(&1.event_name == event.clickhouse_event_attrs.name)
    )

  cond do
    # 🚫 分支 1：没有匹配的 Goal → 不入库任何 Revenue 字段
    is_nil(matching_goal) ->
      %{}

    # ✅ 分支 2：有匹配的 Goal，且币种一致
    matching_goal.currency == revenue_source.currency ->
      %{
        revenue_source_amount: Money.to_decimal(revenue_source),
        revenue_source_currency: to_string(revenue_source.currency),
        revenue_reporting_amount: Money.to_decimal(revenue_source),
        revenue_reporting_currency: to_string(revenue_source.currency)
      }

    # 🔄 分支 3：有匹配的 Goal，但币种不一致 → 执行汇率转换
    matching_goal.currency != revenue_source.currency ->
      converted =
        Money.to_currency!(revenue_source, matching_goal.currency)

      %{
        revenue_source_amount: Money.to_decimal(revenue_source),      # 原始金额
        revenue_source_currency: to_string(revenue_source.currency),   # 原始币种
        revenue_reporting_amount: Money.to_decimal(converted),         # 转换后金额
        revenue_reporting_currency: to_string(converted.currency)       # 目标币种
      }
  end
end

# 如果请求中没有 Revenue 数据，返回空 map
def get_revenue_attrs(_event), do: %{}
```

#### 3.2.2 三个分支的详细说明

| 分支 | 条件 | 结果 | Revenue 字段 |
|------|------|------|-------------|
| **1. 无匹配 Goal** | `matching_goal == nil` | ❌ 不入库 | 所有 Revenue 字段为 NULL |
| **2. 币种一致** | `goal.currency == source.currency` | ✅ 直接入库 | source = reporting |
| **3. 币种不一致** | `goal.currency != source.currency` | 🔄 转换后入库 | source 保留原值，reporting 为转换值 |

#### 3.2.3 Site 模型中的 Revenue Goals 关联

**核心文件**: `lib/plausible/site.ex`

```elixir
schema "sites" do
  # ... 其他字段

  # 普通 Goal（可能有或没有 currency）
  has_many :goals, Plausible.Goal, preload_order: [desc: :id]
  
  # 🔴 Revenue Goal：只包含 currency 不为 nil 的 Goal
  has_many :revenue_goals, Plausible.Goal, where: [currency: {:not, nil}]
  
  # ... 其他字段
end
```

**关键理解**: 
- `revenue_goals` 是一个**过滤后的关联**，只包含设置了 `currency` 的 Goal
- 这意味着：**只有先在后台配置一个带有货币的 Goal，Revenue 事件才可能被入库**

#### 3.2.4 完整的 Revenue 入库决策流程

```
前端发送 Revenue 事件
         │
         ▼
┌─────────────────────────────────┐
│  1. 请求校验阶段                  │
│  - 检查 currency 是否有效         │
│  - 检查 amount 是否可解析         │
│  - 解析为 Money 结构体            │
└───────────────┬─────────────────┘
                │
                ▼
┌─────────────────────────────────┐
│  2. 存储映射阶段（关键决策点）     │
│  extra/lib/plausible/ingestion/  │
│  event/revenue.ex                │
│                                  │
│  查找 matching_goal:             │
│  Enum.find(site.revenue_goals,   │
│    &(&1.event_name == event.name))│
└───────────────┬─────────────────┘
                │
         ┌──────┴──────┐
         │             │
         ▼             ▼
  ┌──────────┐   ┌──────────────┐
  │ 无匹配    │   │ 有匹配 Goal  │
  │ Goal     │   │              │
  └────┬─────┘   └──────┬───────┘
       │                │
       ▼                ▼
  ┌──────────┐    ┌─────────────┐
  │ 返回 %{} │    │ 检查币种    │
  │          │    │ 是否一致    │
  └────┬─────┘    └──────┬──────┘
       │                 │
       ▼            ┌────┴────┐
  ┌──────────────┐  │         │
  │ ❌ Revenue   │  ▼         ▼
  │ 字段不入库   │ ┌─────┐  ┌────────┐
  │              │ │一致 │  │不一致  │
  │ (NULL)       │ └──┬──┘  └───┬────┘
  └──────────────┘    │          │
                      ▼          ▼
                 ┌─────────┐ ┌──────────┐
                 │ source =│ │ 汇率转换 │
                 │reporting│ │          │
                 └────┬────┘ └────┬─────┘
                      │           │
                      ▼           ▼
                 ┌─────────────────┐
                 │ ✅ 4 个 Revenue │
                 │ 字段全部入库     │
                 └─────────────────┘
```

### 3.3 ClickHouse 数据模型

**核心文件**: `lib/plausible/clickhouse_event_v2.ex`

#### 3.3.1 表结构定义

```elixir
schema "events_v2" do
  # 基础字段
  field :name, Ch, type: "LowCardinality(String)"  # 事件名称
  field :site_id, Ch, type: "UInt64"
  field :timestamp, :naive_datetime

  # 自定义属性 - 并行数组设计
  field :"meta.key", {:array, :string}
  field :"meta.value", {:array, :string}

  # 🔴 Revenue 字段 - 双货币设计
  # 注意：这些字段只有在命中 Revenue Goal 时才会有值
  field :revenue_source_amount, Ch, type: "Nullable(Decimal64(3))"
  field :revenue_source_currency, Ch, type: "FixedString(3)"
  field :revenue_reporting_amount, Ch, type: "Nullable(Decimal64(3))"
  field :revenue_reporting_currency, Ch, type: "FixedString(3)"

  # 会话属性（反规范化）
  field :referrer, :string
  field :referrer_source, :string
  field :country_code, Ch, type: "FixedString(2)"
  field :screen_size, Ch, type: "LowCardinality(String)"
  field :operating_system, Ch, type: "LowCardinality(String)"
  field :browser, Ch, type: "LowCardinality(String)"
  # ... 更多字段
end
```

#### 3.3.2 Revenue 字段设计意图

| 字段名 | 类型 | 说明 |
|--------|------|------|
| `revenue_source_amount` | Nullable(Decimal64(3)) | 原始交易金额 |
| `revenue_source_currency` | FixedString(3) | 原始交易货币 |
| `revenue_reporting_amount` | Nullable(Decimal64(3)) | 报表金额（可能经过转换） |
| `revenue_reporting_currency` | FixedString(3) | 报表货币（Goal 配置的货币） |

**设计意图**:
- **source 字段**：保留原始交易数据，用于审计和追溯
- **reporting 字段**：统一货币，用于聚合计算和报表展示

#### 3.3.3 存储设计要点

| 设计决策 | 说明 | 优势 |
|----------|------|------|
| 并行数组存储属性 | `meta.key` 和 `meta.value` 两个数组 | 列式存储友好，查询高效 |
| LowCardinality 类型 | 高基数重复字段使用此类型 | 减少存储，提高压缩率 |
| 双货币 Revenue 设计 | source + reporting | 支持原始货币和报表货币 |
| Decimal64(3) | 金额使用定点数 | 避免浮点精度问题 |
| Nullable 类型 | Revenue 字段可为空 | 未命中 Goal 时为 NULL |
| 会话属性反规范化 | 会话字段直接存储在事件表 | 避免 JOIN，查询性能高 |

---

## 4. 报表聚合环节

### 4.1 目标过滤机制

**核心文件**: `lib/plausible/stats/goals.ex`

#### 4.1.1 目标预加载

```elixir
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

#### 4.1.2 目标条件构建

```elixir
def goal_condition(goal, imported? \\ false) do
  type = Plausible.Goal.type(goal)
  goal_condition(type, goal, imported?)
end

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

#### 4.1.3 自定义属性过滤条件

```elixir
defp build_custom_props_condition(custom_props) do
  Enum.reduce(custom_props, true, fn {prop_key, prop_value}, acc ->
    condition =
      dynamic(
        [e],
        fragment(
          "?[indexOf(?, ?)] = ?",
          field(e, :"meta.value"),  # 值数组
          field(e, :"meta.key"),    # 键数组
          ^prop_key,                 # 目标键
          ^prop_value                # 目标值
        )
      )

    dynamic([e], ^acc and ^condition)
  end)
end
```

**查询原理**: 使用 `indexOf(meta.key, prop_key)` 找到键在数组中的索引，然后用该索引从 `meta.value` 数组中获取对应的值进行比较。

### 4.2 Revenue 指标聚合（重写版）

**核心文件**: `extra/lib/plausible/stats/goal/revenue.ex`

#### 4.2.1 支持的 Revenue 指标

```elixir
@revenue_metrics [:average_revenue, :total_revenue]

def revenue_metrics(), do: @revenue_metrics
```

#### 4.2.2 货币处理逻辑与入库条件的关联

**重要理解**: 报表聚合阶段的行为**直接依赖于存储阶段的入库决策**。

```elixir
def preload(site, preloaded_goals, metrics, dimensions) do
  cond do
    # 没有请求 Revenue 指标
    not requested?(metrics) -> {nil, %{}}
    
    # 站点没有 Revenue Goals 功能权限
    not available?(site) -> {:revenue_goals_unavailable, %{}}
    
    # 正常流程：检查匹配的 Goal
    true -> preload(preloaded_goals.matching_toplevel_filters, dimensions)
  end
end

defp preload(goals, dimensions) do
  # 构建 Goal 到货币的映射
  goal_currency_map =
    goals
    |> Map.new(fn goal -> {Plausible.Goal.display_name(goal), goal.currency} end)
    |> Map.reject(fn {_goal, currency} -> is_nil(currency) end)

  currencies = goal_currency_map |> Map.values() |> Enum.uniq()
  goal_dimension? = "event:goal" in dimensions

  case {currencies, goal_dimension?} do
    # ✅ 情况 1：单一货币，且不按 Goal 分组
    # 使用 reporting 字段聚合
    {[currency], false} -> 
      {nil, %{default: currency}}
    
    # ❌ 情况 2：没有匹配的 Revenue Goal
    # 意味着：要么没配置 Goal，要么配置了但没有事件命中
    {[], _} -> 
      {:no_revenue_goals_matching, %{}}
    
    # ✅ 情况 3：多货币，但按 Goal 分组
    # 每个 Goal 使用自己的 reporting 货币
    {_, true} -> 
      {nil, goal_currency_map}
    
    # ❌ 情况 4：多货币，且不按 Goal 分组
    # 无法聚合（不同货币不能直接相加）
    _ -> 
      {:no_single_revenue_currency, %{}}
  end
end
```

#### 4.2.3 入库条件对报表聚合的影响矩阵

| 入库阶段情况 | 数据状态 | 报表聚合结果 |
|-------------|---------|-------------|
| **无配置的 Revenue Goal** | 所有 Revenue 字段为 NULL | `no_revenue_goals_matching` 警告 |
| **配置了 Goal，但无事件命中** | 对应事件的 Revenue 字段为 NULL | `no_revenue_goals_matching` 警告 |
| **单一货币，事件命中 Goal** | reporting 字段有值 | ✅ 正常聚合（sum/avg） |
| **多货币，按 Goal 分组** | 每个 Goal 的 reporting 有值 | ✅ 按分组分别聚合 |
| **多货币，不按 Goal 分组** | 各事件 reporting 货币不同 | ❌ `no_single_revenue_currency` 错误 |

#### 4.2.4 聚合时使用的字段

**报表聚合始终使用 `revenue_reporting_*` 字段**，而非 `revenue_source_*` 字段。

这意味着：
- 如果币种一致：`reporting` = `source`，直接使用原始值
- 如果币种不一致：`reporting` 是转换后的值，使用转换后的值进行聚合

#### 4.2.5 Revenue 格式化

```elixir
def format_revenue_metric(value, currency) do
  if currency do
    money = Money.new!(value || 0, currency)

    %{
      short: Money.to_string!(money, format: :short, fractional_digits: 1),
      long: Money.to_string!(money),
      value: Decimal.to_float(money.amount),
      currency: currency
    }
  else
    value
  end
end
```

### 4.3 自定义属性查询

**核心文件**: `lib/plausible/stats/custom_props.ex`

#### 4.3.1 获取属性名列表

```elixir
def fetch_prop_names(site, query) do
  case Filters.get_toplevel_filter(query, "event:props:") do
    [_op, "event:props:" <> key | _rest] ->
      [key]  # 已指定具体属性

    _ ->
      from(e in base_event_query(query),
        join: meta in fragment("meta"),  # 数组展开
        hints: "ARRAY",
        on: true,
        select: meta.key,
        distinct: true
      )
      |> maybe_allowed_props_only(site)  # 权限过滤
      |> ClickhouseRepo.all()
  end
end

def maybe_allowed_props_only(q, site) do
  case Plausible.Props.allowed_for(site) do
    :all -> q
    allowed_props -> from [..., m] in q, where: m.key in ^allowed_props
  end
end
```

### 4.4 完整的 Revenue 查询链路

```
用户查询 Revenue 指标
         │
         ▼
┌─────────────────────────────────────┐
│ 1. 查询构建阶段                      │
│ lib/plausible/stats/query_builder.ex│
│                                     │
│ - 检查是否有 RevenueGoals 功能      │
│ - 预加载匹配的 Goals 和货币信息     │
│ - 验证多货币场景是否可聚合           │
└───────────────┬─────────────────────┘
                │
                ▼
┌─────────────────────────────────────┐
│ 2. SQL 执行阶段                      │
│                                     │
│ 使用的字段：                         │
│ - revenue_reporting_amount  ← 聚合  │
│ - revenue_reporting_currency ← 展示 │
│                                     │
│ 注意：这些字段是否有值，取决于       │
│ 「入库阶段是否命中了 Revenue Goal」 │
└───────────────┬─────────────────────┘
                │
         ┌──────┴──────┐
         │             │
         ▼             ▼
┌────────────────┐  ┌──────────────────┐
│ 字段有值       │  │ 字段为 NULL      │
│ (命中 Goal)   │  │ (未命中 Goal)    │
└───────┬────────┘  └────────┬─────────┘
        │                      │
        ▼                      ▼
┌────────────────┐  ┌──────────────────┐
│ ✅ 正常聚合     │  │ ⚠️ 返回警告      │
│ sum/avg 计算   │  │ no_revenue_goals │
│                │  │ _matching         │
└────────────────┘  └──────────────────┘
```

### 4.5 聚合查询流程

```
用户查询
    │
    ▼
┌─────────────────┐
│ Query.parse()   │ 解析查询参数
│  - 指标解析      │
│  - 维度解析      │
│  - 过滤器解析    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ QueryRunner.run()│ 执行查询
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ QueryBuilder    │ 构建 SQL
│  - 选择表        │
│  - 构建 WHERE    │
│  - 构建 GROUP BY │
│  - 聚合函数      │
│  - Revenue 货币检查│
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ ClickHouse      │ 执行查询
│  - 数组索引查询  │
│  - 聚合计算      │
│  - 使用 reporting 字段│
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 结果格式化      │
│  - Revenue 格式化│
│  - 百分比计算    │
└────────┬────────┘
         │
         ▼
      返回结果
```

---

## 5. 扩展成本分析（重写版）

### 5.1 资源限制

**核心文件**: `lib/plausible/props.ex`

#### 5.1.1 配置限制

```elixir
@max_props 300          # 每个站点最多 300 个可配置属性
@max_prop_key_length 300    # 属性键最大长度
@max_prop_value_length 2000  # 属性值最大长度
```

#### 5.1.2 运行时限制

| 限制项 | 值 | 说明 |
|--------|-----|------|
| 单事件属性数量 | 30 | `put_props` 中 `Enum.take(@max_props)` |
| 站点可配置属性 | 300 | `allowed_event_props` 列表最大长度 |
| 属性键长度 | 300 字节 | 校验时检查 |
| 属性值长度 | 2000 字节 | 校验时检查 |
| 事件名称长度 | 120 字节 | Changeset 校验 |

### 5.2 计费特性

#### 5.2.1 自定义属性特性

```elixir
def allowed_for(site, opts \\ []) do
  site = Plausible.Repo.preload(site, :team)
  internal_keys = Plausible.Props.internal_keys()
  props_enabled? = Plausible.Billing.Feature.Props.check_availability(site.team) == :ok
  bypass_setup? = Keyword.get(opts, :bypass_setup?)

  cond do
    props_enabled? && is_nil(site.allowed_event_props) -> :all
    props_enabled? && bypass_setup? -> :all
    props_enabled? -> site.allowed_event_props ++ internal_keys
    true -> internal_keys  # 仅内部属性
  end
end
```

#### 5.2.2 Revenue Goals 特性

```elixir
def available?(site) do
  site = Plausible.Repo.preload(site, :team)
  Plausible.Billing.Feature.RevenueGoals.check_availability(site.team) == :ok
end
```

#### 5.2.3 特性依赖矩阵

| 功能 | 所需特性 | 内部属性访问 | 额外配置要求 |
|------|----------|-------------|-------------|
| 页面浏览事件 | 无 | 总是可用 | 无 |
| 自定义事件追踪 | 无 | 总是可用 | 无 |
| 自定义属性查询 | `Props` | 仅 `url`, `path`, `search_query` | 配置 `allowed_event_props` |
| **Revenue 事件入库** | `RevenueGoals` | 不可用 | **必须配置带 currency 的 Goal** |
| **Revenue 报表聚合** | `RevenueGoals` | 不可用 | **必须有事件命中 Goal** |

### 5.3 性能成本（重写版）

#### 5.3.1 存储成本

| 数据类型 | 存储开销 | 说明 |
|----------|----------|------|
| 自定义属性 | 每个属性存储为两个字符串（键+值） | 使用数组压缩，实际开销取决于基数 |
| **Revenue（命中 Goal）** | 4 个字段 ≈ 16 字节/事件 | 2 个 Decimal64(3) + 2 个 FixedString(3) |
| **Revenue（未命中 Goal）** | 0 额外字节 | 所有字段为 NULL，不占用实际存储 |
| LowCardinality 字段 | 字典编码 | 重复值高的字段显著减少存储 |

**重要发现**: 
- Revenue 字段**只有在命中 Goal 时才占用存储**
- 这是一种**惰性存储**策略，避免浪费存储空间

#### 5.3.2 查询成本

| 查询类型 | 复杂度 | 说明 | 受入库条件影响 |
|----------|--------|------|---------------|
| 按事件名称过滤 | O(log n) | 有索引，高效 | 否 |
| 按单个属性过滤 | O(n) | 需扫描数组，无直接索引 | 否 |
| 按多个属性过滤 | O(n * k) | k 为属性数量 | 否 |
| 按属性分组 | O(n) | 需展开数组 | 否 |
| **Revenue 聚合** | O(n) | 需扫描相关事件 | **是** |

**Revenue 聚合的额外成本因素**:

1. **NULL 值过滤**: 聚合时需要过滤掉 `revenue_reporting_amount IS NULL` 的行
2. **汇率转换的滞后影响**: 
   - 如果币种不一致，入库时已完成转换
   - 但如果汇率更新，**历史数据不会重新计算**
3. **多货币检查**: 查询构建阶段需要检查货币一致性

#### 5.3.3 索引策略

当前设计中，`meta.key` 和 `meta.value` 数组**没有直接索引**，属性过滤依赖：

1. **分区裁剪**: 按 `site_id` + `timestamp` 分区
2. **前置过滤**: 先按事件名称、时间范围等过滤，减少扫描行数
3. **向量化执行**: ClickHouse 的向量化处理缓解数组扫描开销

**Revenue 字段同样没有专门索引**，依赖：
- 事件名称过滤（有索引）
- 时间范围过滤（有分区）

### 5.4 入库条件对扩展成本的影响（新增）

#### 5.4.1 配置复杂度影响

| 场景 | 配置复杂度 | 维护成本 |
|------|-----------|---------|
| 单 Revenue Goal，单货币 | 低 | 低 |
| 多 Revenue Goal，统一货币 | 中 | 中 |
| 多 Revenue Goal，多货币，按 Goal 分组 | 高 | 中 |
| 多 Revenue Goal，多货币，不按 Goal 分组 | **无法聚合** | - |

#### 5.4.2 汇率转换的成本

**币种不一致时的处理逻辑（代码可证）**:

**已实现的代码逻辑**:

| 代码位置 | 实现内容 |
|----------|----------|
| `extra/lib/plausible/ingestion/event/revenue.ex:26` | 调用 `Money.to_currency!(revenue_source, matching_goal.currency)` 进行汇率转换 |
| `config/runtime.exs:687-689` | 配置 `ex_money` 库使用 Open Exchange Rates API，设置 `retrieve_every: :timer.hours(24)` 每24小时获取一次汇率 |

**风险提示（基于代码逻辑的推断）**:

1. **汇率时效性风险**:
   - 汇率按固定周期（24小时）批量获取，而非实时
   - 入库时使用的汇率可能与交易实际时间的汇率有差异
   - 一旦入库，`revenue_reporting_amount` 字段**不会**因后续汇率更新而回溯调整

2. **外部依赖风险**:
   - 汇率数据依赖外部 API（Open Exchange Rates）
   - 代码中未显式处理 API 不可用或返回失败的场景（使用 `to_currency!` 而非 `to_currency`，失败会抛出异常）

3. **存储成本**:
   - 需要同时存储 source 和 reporting 两套数据（4 个字段：`revenue_source_amount`, `revenue_source_currency`, `revenue_reporting_amount`, `revenue_reporting_currency`）
   - 每个事件的额外存储开销约为 16 字节（2 个 Decimal64 + 2 个 FixedString(3)）

#### 5.4.3 调试和排查成本

**Revenue 数据不显示的可能原因排查链**:

```
用户报告 "Revenue 指标为空"
         │
         ▼
┌─────────────────────────────────┐
│ 1. 检查功能权限                   │
│    - 团队是否有 RevenueGoals 功能│
└───────────────┬─────────────────┘
                │
                ▼
┌─────────────────────────────────┐
│ 2. 检查 Goal 配置                │
│    - 是否配置了带 currency 的 Goal│
│    - Goal 的 event_name 是否正确  │
└───────────────┬─────────────────┘
                │
                ▼
┌─────────────────────────────────┐
│ 3. 检查事件数据                  │
│    - 事件名称是否与 Goal 匹配     │
│    - revenue_source_amount 是否  │
│      为 NULL（未命中 Goal）      │
└───────────────┬─────────────────┘
                │
                ▼
┌─────────────────────────────────┐
│ 4. 检查查询条件                  │
│    - 多货币场景是否按 Goal 分组   │
│    - 时间范围是否包含有效数据     │
└─────────────────────────────────┘
```

**这比普通事件排查复杂得多**，因为涉及：
- 功能权限检查
- Goal 配置检查
- 入库条件检查
- 多货币场景检查

### 5.5 扩展建议（重写版）

#### 5.5.1 Revenue 事件的最佳实践

| 建议 | 原因 |
|------|------|
| **先配置 Goal，再发送事件** | 否则 Revenue 数据不会入库 |
| **使用统一货币** | 避免汇率转换的复杂性和成本 |
| **按 Goal 分组查询多货币场景** | 否则无法聚合 |
| **确保事件名称与 Goal 完全匹配** | 大小写敏感，完全匹配才会入库 |

#### 5.5.2 高基数属性优化

对于高基数属性（如 `user_id`、`order_id`），建议：

1. **避免存储为自定义属性**: 考虑使用专门字段或外部系统
2. **控制属性值长度**: 长值会增加存储和扫描成本
3. **限制单事件属性数量**: 过多属性会增加查询复杂度

#### 5.5.3 常见查询模式优化

| 场景 | 建议 |
|------|------|
| 频繁按某个属性过滤 | 考虑添加专门字段或使用 Materialized View |
| 多属性组合查询 | 评估是否需要更结构化的存储 |
| 高吞吐量事件 | 评估采样策略或数据保留策略 |
| **Revenue 多货币场景** | **确保所有目标使用统一货币，或按 `event:goal` 分组** |

#### 5.5.4 多货币场景的架构建议

如果业务确实需要支持多货币，建议：

1. **方案 A：应用层统一货币**
   - 在发送事件前，应用层统一转换为报表货币
   - 数据库中只存储单一货币
   - 简化查询和聚合

2. **方案 B：按 Goal 分组**
   - 每个 Goal 使用自己的货币
   - 查询时必须按 `event:goal` 分组
   - 应用层处理跨 Goal 的汇总（如果需要）

3. **方案 C：使用独立的财务系统**
   - Plausible 仅用于趋势分析
   - 精确的财务汇总使用专门的财务系统
   - 避免在分析系统中处理复杂的货币转换

---

## 6. 关键代码位置汇总

### 6.1 采集与存储模块（入库链路）

| 功能模块 | 文件路径 | 关键函数/模块 |
|----------|----------|--------------|
| 前端采集 | `tracker/src/track.js` | `track()` |
| 请求解析 | `lib/plausible/ingestion/request.ex` | `build/2` |
| 属性校验 | `lib/plausible/ingestion/request.ex` | `put_props/2`, `validate_props/1` |
| Revenue 校验 | `extra/lib/plausible/ingestion/request/revenue.ex` | `put_revenue_source/2` |
| 事件处理管道 | `lib/plausible/ingestion/event.ex` | `pipeline/0` |
| 属性存储映射 | `lib/plausible/ingestion/event.ex` | `put_props/2` |
| **Revenue 存储映射（关键）** | `extra/lib/plausible/ingestion/event/revenue.ex` | `get_revenue_attrs/1` |
| Site Goal 关联 | `lib/plausible/site.ex` | `has_many :revenue_goals` |
| 数据模型 | `lib/plausible/clickhouse_event_v2.ex` | `schema "events_v2"` |

### 6.2 查询与聚合模块（查询链路）

| 功能模块 | 文件路径 | 关键函数/模块 |
|----------|----------|--------------|
| 查询解析 | `lib/plausible/stats/query.ex` | `parse_and_build/2`, `parse_and_build!/3` |
| API 查询解析 | `lib/plausible/stats/api_query_parser.ex` | `parse/2`, `parse_metrics/1`, `parse_filters/1` |
| 查询构建 | `lib/plausible/stats/query_builder.ex` | `build/3`, `build!/3` |
| SQL 查询构建 | `lib/plausible/stats/sql/query_builder.ex` | `build/2` |
| 查询执行 | `lib/plausible/stats/query_runner.ex` | `run/2` |
| 目标过滤 | `lib/plausible/stats/goals.ex` | `add_filter/3`, `goal_condition/2` |
| **Revenue 聚合** | `extra/lib/plausible/stats/goal/revenue.ex` | `preload/4`, `format_revenue_metric/3` |
| 聚合查询入口 | `lib/plausible/stats/aggregate.ex` | `aggregate/3` |
| 属性查询 | `lib/plausible/stats/custom_props.ex` | `fetch_prop_names/2` |

### 6.3 目标与配置模块

| 功能模块 | 文件路径 | 关键函数/模块 |
|----------|----------|--------------|
| 目标定义 | `lib/plausible/goal.ex` | `defstruct`, `type/1` |
| Revenue 目标 | `extra/lib/plausible/goal/revenue.ex` | `valid_currencies/0`, `type/1` |
| 目标管理 | `lib/plausible/goals/goals.ex` | `for_site/2` |
| 自定义属性配置 | `lib/plausible/props.ex` | `allowed_for/2`, `max_prop_key_length/0` |

### 6.4 汇率转换配置

| 功能模块 | 文件路径 | 关键配置项 |
|----------|----------|-----------|
| 汇率 API 配置 | `config/runtime.exs:687-689` | `:ex_money` 配置 `open_exchange_rates_app_id` 和 `retrieve_every: :timer.hours(24)` |
| 测试环境 Mock | `config/test.exs:34` | `api_module: Plausible.ExchangeRateMock` |
| 依赖声明 | `mix.exs:155` | `{:ex_money, "~> 5.12"}` |

---

## 7. 总结

### 7.1 核心发现

#### Revenue 事件的关键特性

1. **目标命中是前提**
   - Revenue 数据**只有在事件名称匹配配置的 Revenue Goal 时才会入库**
   - 没有配置 Goal 或事件名称不匹配 → 所有 Revenue 字段为 NULL

2. **双分支币种处理**
   - **币种一致**: `source` = `reporting`，直接存储
   - **币种不一致**: 执行汇率转换，`source` 保留原值，`reporting` 存储转换值

3. **报表聚合依赖入库状态**
   - 聚合时使用 `revenue_reporting_*` 字段
   - 如果入库时为 NULL，聚合结果为空或返回警告

### 7.2 架构优势

1. **灵活的采集层**: 支持多种属性传递方式（`props`, `meta`, `p`）
2. **严格的校验机制**: 多层校验确保数据质量
3. **ClickHouse 优化设计**: 并行数组、LowCardinality 等设计适应列式存储
4. **特性控制**: 基于计费特性的精细访问控制
5. **惰性存储**: Revenue 字段只有在命中 Goal 时才占用存储

### 7.3 潜在风险

1. **配置复杂度**: 
   - Revenue 事件需要先配置 Goal，增加了上手难度
   - 多货币场景需要按 Goal 分组，增加了查询复杂度

2. **查询性能**:
   - 数组索引查询无直接索引，高基数场景可能较慢
   - 多货币且不分组时无法聚合

3. **数据一致性**:
   - 汇率转换是时点值，历史数据不会随汇率更新
   - 可能导致与财务系统的差异

4. **排查成本**:
   - Revenue 数据不显示的原因链较长
   - 需要检查功能权限、Goal 配置、事件匹配、查询条件等多个环节

### 7.4 扩展建议

1. **配置策略**:
   - 先配置 Goal，再发送 Revenue 事件
   - 建议使用统一货币简化聚合
   - 多货币场景必须按 `event:goal` 分组查询

2. **性能优化**:
   - 监控属性基数，定期检查高基数属性
   - 识别高频属性查询，考虑 Materialized View
   - 根据事件量和属性数量预估存储增长

3. **架构选择**:
   - 简单场景：单一货币 + 少量 Goal
   - 复杂场景：考虑应用层统一货币或使用专门财务系统
   - 避免在分析系统中处理复杂的货币转换逻辑

---

## 附录：Revenue 事件完整生命周期示例

### 场景 1：成功入库（币种一致）

```
1. 配置阶段
   - 在后台创建 Revenue Goal:
     - event_name: "Purchase"
     - currency: "USD"

2. 前端发送
   plausible.track('Purchase', {
     revenue: { amount: 99.99, currency: 'USD' }
   })

3. 后端处理
   - 请求校验: ✓ currency 有效，amount 可解析
   - 目标命中: ✓ "Purchase" 匹配配置的 Goal
   - 币种检查: ✓ USD == USD
   - 入库字段:
     revenue_source_amount: 99.99
     revenue_source_currency: "USD"
     revenue_reporting_amount: 99.99
     revenue_reporting_currency: "USD"

4. 报表查询
   - 查询 total_revenue: ✓ 99.99
   - 查询 average_revenue: ✓ 99.99
```

### 场景 2：成功入库（币种不一致）

```
1. 配置阶段
   - Goal: event_name: "Purchase", currency: "EUR"

2. 前端发送
   plausible.track('Purchase', {
     revenue: { amount: 99.99, currency: 'USD' }
   })

3. 后端处理
   - 目标命中: ✓
   - 币种检查: ✗ USD != EUR
   - 汇率转换: 假设 1 USD = 0.92 EUR
   - 入库字段:
     revenue_source_amount: 99.99
     revenue_source_currency: "USD"
     revenue_reporting_amount: 91.99  (99.99 * 0.92)
     revenue_reporting_currency: "EUR"

4. 报表查询
   - total_revenue: 91.99 EUR (使用 reporting 字段)
```

### 场景 3：不入库（无匹配 Goal）

```
1. 配置阶段
   - 没有配置任何 Revenue Goal
   - 或配置的 Goal event_name 是 "Checkout"

2. 前端发送
   plausible.track('Purchase', {
     revenue: { amount: 99.99, currency: 'USD' }
   })

3. 后端处理
   - 请求校验: ✓
   - 目标命中: ✗ 没有匹配的 Goal
   - 返回: %{}
   - 入库字段: 所有 Revenue 字段为 NULL

4. 报表查询
   - 查询 total_revenue:
     ⚠️ 警告: "no_revenue_goals_matching"
     - 结果: null
```

### 场景 4：无法聚合（多货币不分组）

```
1. 配置阶段
   - Goal 1: event_name: "Purchase_US", currency: "USD"
   - Goal 2: event_name: "Purchase_EU", currency: "EUR"

2. 前端发送
   plausible.track('Purchase_US', { revenue: { amount: 99.99, currency: 'USD' } })
   plausible.track('Purchase_EU', { revenue: { amount: 89.99, currency: 'EUR' } })

3. 后端处理
   - 两个事件都命中各自的 Goal
   - 分别入库（USD 和 EUR）

4. 报表查询
   - 查询 1:  metrics: [total_revenue], dimensions: ["event:goal"]
     ✓ 按 Goal 分组，可以聚合
     - "Purchase_US": 99.99 USD
     - "Purchase_EU": 89.99 EUR

   - 查询 2:  metrics: [total_revenue], dimensions: []
     ✗ 不分组，多货币
     ⚠️ 警告: "no_single_revenue_currency"
     - 结果: null（无法直接相加 USD 和 EUR）
```

---

*报告生成时间: 2026-05-03*
*基于代码版本: 当前工作目录*
*更新内容: 重写 Revenue 查询链路，澄清目标命中逻辑、币种转换分支及影响*
