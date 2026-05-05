# Plausible GA 导入数据与原生事件混合查询机制

## 概述

当用户从 Google Analytics (GA4/UA) 导入历史数据后，Plausible Dashboard 需要同时查询：
1. **原生 ClickHouse 事件**（`events_v2` / `sessions_v2` 表）- Plausible 脚本收集的实时数据
2. **导入的 GA 数据**（`imported_*` 系列表）- 从 GA Data API 导入的历史聚合数据

本文档详细说明这两种数据源如何在查询层面合并、去重和对齐指标。

---

## 一、数据导入流程与存储结构

### 1.1 GA Data API 调用边界

**GA Data API 仅在导入阶段使用**，查询阶段完全不依赖外部 API。

```
┌─────────────────────────────────────────────────────────────────┐
│                        导入阶段 (Import Phase)                     │
├─────────────────────────────────────────────────────────────────┤
│  Google Analytics Data API                                        │
│         │                                                          │
│         ▼                                                          │
│  Plausible.Google.GA4.API.import_analytics()                     │
│         │                                                          │
│         ▼ (转换为 Plausible 数据模型)                               │
│  Plausible.Imported.Buffer (批量写入 ClickHouse)                   │
│         │                                                          │
│         ▼                                                          │
│  imported_* 系列表 (ClickHouse)                                    │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                        查询阶段 (Query Phase)                      │
├─────────────────────────────────────────────────────────────────┤
│  Dashboard API Request                                             │
│         │                                                          │
│         ▼                                                          │
│  Plausible.Stats.SQL.QueryBuilder                                  │
│         │                                                          │
│         ├───────────────┬───────────────┐                         │
│         ▼               ▼               ▼                         │
│  events_v2表      sessions_v2表    imported_*表                   │
│  (原生事件)         (原生会话)        (导入数据)                     │
│         │               │               │                         │
│         └───────────────┴───────────────┘                         │
│         │                                                          │
│         ▼ (SQL FULL JOIN / CROSS JOIN 合并)                        │
│  统一结果返回                                                        │
└─────────────────────────────────────────────────────────────────┘
```

**关键代码位置**：
- GA4 API 调用：`lib/plausible/google/ga4/api.ex`
- 数据转换：`lib/plausible/imported/google_analytics4.ex:from_report/4`
- 批量写入：`lib/plausible/imported/buffer.ex`

### 1.2 导入数据存储结构

GA 数据导入后存储在以下 ClickHouse 表中（均为聚合后的日级数据）：

| 表名 | 对应维度 | 主要指标字段 |
|------|----------|-------------|
| `imported_visitors` | 无（按日期聚合） | visitors, pageviews, bounces, visits, visit_duration |
| `imported_sources` | source, channel, referrer, utm_* | visitors, visits, pageviews, bounces, visit_duration |
| `imported_pages` | hostname, page | visitors, visits, pageviews, exits, total_time_on_page |
| `imported_entry_pages` | entry_page | visitors, entrances, visit_duration, pageviews, bounces |
| `imported_exit_pages` | exit_page | (GA4 API 当前无此数据) |
| `imported_custom_events` | name, link_url | visitors, events |
| `imported_locations` | country, region, city | visitors, visits, pageviews, bounces, visit_duration |
| `imported_devices` | device | visitors, visits, pageviews, bounces, visit_duration |
| `imported_browsers` | browser, browser_version | visitors, visits, pageviews, bounces, visit_duration |
| `imported_operating_systems` | operating_system, operating_system_version | visitors, visits, pageviews, bounces, visit_duration |

**表选择映射**（`lib/plausible/stats/imported/base.ex:14-48`）：
```elixir
@property_to_table_mappings %{
  "visit:source" => "imported_sources",
  "visit:channel" => "imported_sources",
  "visit:country" => "imported_locations",
  "visit:device" => "imported_devices",
  "visit:browser" => "imported_browsers",
  "visit:os" => "imported_operating_systems",
  "event:page" => "imported_pages",
  "event:name" => "imported_custom_events",
  "time:month" => "imported_visitors",
  "time:week" => "imported_visitors",
  "time:day" => "imported_visitors",
  "time:hour" => "imported_visitors"
}
```

### 1.3 导入数据的关键标识字段

每条导入记录包含以下关键字段：
- `site_id` - 站点 ID
- `import_id` - 导入批次 ID（对应 `site_imports` 表的主键）
- `date` - 数据日期（用户本地时区）

**import_id 的特殊处理**（`lib/plausible/imported.ex:67-79`）：
```elixir
def complete_import_ids(site) do
  imports = get_completed_imports(site)
  has_legacy? = Enum.any?(imports, fn %{legacy: legacy?} -> legacy? end)
  ids = Enum.map(imports, fn %{id: id} -> id end)

  # 兼容旧版导入（import_id = 0）
  if has_legacy? do
    [0 | ids]
  else
    ids
  end
end
```

---

## 二、查询时的数据合并机制

### 2.1 合并触发条件

查询是否合并导入数据由以下条件决定（`lib/plausible/stats/query.ex:149-170`）：

```elixir
def put_imported_opts(query, site) do
  requested? = query.include.imports  # API 参数 include.imports = true

  query =
    if site && Imported.schema_supports_interval?(query) do
      site = Plausible.Repo.preload(site, :completed_imports)
      struct!(query,
        imports_exist: Plausible.Imported.any_completed_imports?(site),
        imports_in_range: get_imports_in_range(site, query)
      )
    else
      query
    end

  skip_imported_reason = get_skip_imported_reason(query)

  struct!(query,
    include_imported: requested? and is_nil(skip_imported_reason),
    skip_imported_reason: skip_imported_reason
  )
end
```

### 2.2 导入合并的开启条件与降级条件

#### 开启导入合并的必要条件

查询是否会合并导入数据，需要**同时满足**以下所有条件（`lib/plausible/stats/query.ex:149-170`）：

```elixir
def put_imported_opts(query, site) do
  requested? = query.include.imports  # 条件1: 用户主动请求

  query =
    if site && Imported.schema_supports_interval?(query) do  # 条件2: 时间间隔支持
      site = Plausible.Repo.preload(site, :completed_imports)
      struct!(query,
        imports_exist: Plausible.Imported.any_completed_imports?(site),  # 条件3: 有导入数据
        imports_in_range: get_imports_in_range(site, query)  # 条件4: 范围内有数据
      )
    else
      query
    end

  skip_imported_reason = get_skip_imported_reason(query)  # 条件5: 查询模式支持

  struct!(query,
    include_imported: requested? and is_nil(skip_imported_reason),
    skip_imported_reason: skip_imported_reason
  )
end
```

**详细条件说明**：

| 条件 | 检查内容 | 代码位置 |
|------|----------|----------|
| **1. 用户主动请求** | `include.imports = true` 或 `include.imports_meta = true` | API 参数 |
| **2. 时间间隔支持** | 不使用 `time:minute` 或 `time:hour` 维度 | `schema_supports_interval?/1` |
| **3. 存在导入数据** | 站点有 `status = completed` 的导入记录 | `any_completed_imports?/1` |
| **4. 时间范围重叠** | 查询时间范围与导入数据范围有交集 | `completed_imports_in_query_range/2` |
| **5. 查询模式支持** | 维度/过滤器组合可被导入表支持 | `schema_supports_query?/1` |

#### API 参数控制

导入合并的开启由 API 的 `include` 参数控制：

```json
{
  "include": {
    "imports": true,        // 是否合并导入数据到结果
    "imports_meta": true    // 是否返回导入相关元数据（即使不合并）
  }
}
```

**参数行为**：
- `include.imports = true`：尝试合并导入数据，如果失败则降级为仅本地查询
- `include.imports_meta = true`：即使不合并数据，也在响应中返回 `imports_included`、`imports_skip_reason` 等元信息
- Dashboard 默认会设置 `include.imports = true`

#### 降级为仅本地查询的原因

当不满足上述条件时，查询会**降级为仅查询原生数据**，并在响应中返回 `imports_skip_reason`。

**所有降级原因**（`lib/plausible/stats/query.ex:191-210`）：

```elixir
@spec get_skip_imported_reason(t()) ::
        nil | :no_imported_data | :out_of_range | :unsupported_interval | :unsupported_query

def get_skip_imported_reason(query) do
  cond do
    not Imported.schema_supports_interval?(query) ->
      :unsupported_interval

    not query.imports_exist ->
      :no_imported_data

    query.imports_in_range == [] ->
      :out_of_range

    not Imported.schema_supports_query?(query) ->
      :unsupported_query

    true ->
      nil
  end
end
```

**详细降级原因表**：

| 原因值 | 触发条件 | 用户感知 |
|--------|----------|----------|
| `:unsupported_interval` | 使用 `time:minute` 或 `time:hour` 维度 | 导入数据只有日级，无法按小时/分钟合并 |
| `:no_imported_data` | 站点没有已完成的导入记录 | 用户还未导入任何 GA 数据 |
| `:out_of_range` | 查询时间范围与导入数据范围无重叠 | 例如：导入的是 2023 年数据，查询的是 2025 年 |
| `:unsupported_query` | 维度/过滤器组合不被导入表支持 | 见下文详细说明 |

#### `:unsupported_query` 的详细触发场景

当查询的维度或过滤器组合无法用预聚合的导入数据表示时，会触发此降级原因。

**场景 1：行为过滤器**（`lib/plausible/stats/imported/base.ex:77-91`）：
```elixir
def decide_tables(query) do
  behavioral_filters = dimensions_used_in_filters(query.filters, behavioral_filters: :only)

  cond do
    # 行为过滤器无法通过聚合的导入数据模拟
    length(behavioral_filters) > 0 ->
      []  # 返回空表列表，不支持导入查询
    ...
  end
end
```

**行为过滤器包括**：
- `visit:entry_page` - 入口页面（需要行为序列分析）
- `visit:exit_page` - 退出页面（需要行为序列分析）
- 某些复杂的自定义属性组合

**场景 2：多维度组合映射到不同表**：

导入数据是按维度预聚合到不同表的，一次查询只能从**单张** `imported_*` 表查询：

```elixir
# lib/plausible/stats/imported/base.ex:182-208
defp do_decide_tables(query) do
  table_candidates =
    dimensions_used_in_filters(query.filters)
    |> Enum.concat(query.dimensions)
    |> Enum.reject(&(&1 in @queriable_time_dimensions or &1 == "event:goal"))
    |> Enum.map(&@property_to_table_mappings[&1])

  case Enum.uniq(table_candidates) do
    [] -> ["imported_visitors"]      # 无维度：使用总表
    [nil] -> []                       # 存在不支持的维度
    [candidate] -> [candidate]        # 单一维度：使用对应表
    _ -> []                           # 多维度映射到不同表：不支持
  end
end
```

**示例**：
| 查询 | 支持？ | 原因 |
|------|--------|------|
| `dimensions: ["visit:source"]` | ✅ | 只需要 `imported_sources` |
| `dimensions: ["visit:country"]` | ✅ | 只需要 `imported_locations` |
| `dimensions: ["visit:source", "visit:country"]` | ❌ | `imported_sources` 和 `imported_locations` 是两张不同表 |

**场景 3：自定义属性的复杂使用**：

自定义属性导入支持有限（仅 `event:props:url` 和 `event:props:path`），且需要配合特定事件名过滤器：

```elixir
# lib/plausible/stats/imported/base.ex:125-150
defp do_decide_custom_prop_table(query, property) do
  "event:props:" <> prop_key = property

  has_required_event_or_goal_name_filter? =
    query.filters
    |> Enum.flat_map(fn
      [:is, "event:name", event_names | _rest] -> event_names
      [:is, "event:goal", goal_names | _rest] -> goal_names
      _ -> []
    end)
    |> Enum.any?(fn event_or_goal_name ->
      event_or_goal_name in Plausible.Event.SystemEvents.special_events_for_prop_key(prop_key)
    end)

  if has_required_event_or_goal_name_filter? and
       not has_unsupported_filters? do
    ["imported_custom_events"]
  else
    []
  end
end
```

**场景 4：实时查询**：

```elixir
# lib/plausible/stats/query.ex:172-175
defp get_imports_in_range(_site, %__MODULE__{input_date_range: period})
     when period in [:realtime, :realtime_30m] do
  []  # 实时查询不使用导入数据
end
```

---

## 三、API 响应元数据字段说明

### 3.1 元数据字段总览

当 `include.imports = true` 或 `include.imports_meta = true` 时，API 响应的 `meta` 字段会包含以下导入相关信息：

```json
{
  "results": [...],
  "meta": {
    "imports_included": true,
    "imports_skip_reason": null,
    "imports_warning": null,
    "metric_warnings": {
      "bounce_rate": {
        "code": "no_imported_bounce_rate",
        "warning": "imported bounce_rate is not available when using a page filter"
      }
    }
  }
}
```

**字段定义**（`lib/plausible/stats/query_result.ex:73-85`）：

```elixir
defp add_imports_meta(meta, %Query{include: include} = query) do
  if include.imports or include.imports_meta do
    %{
      imports_included: query.include_imported,
      imports_skip_reason: query.skip_imported_reason,
      imports_warning: @imports_warnings[query.skip_imported_reason]
    }
    |> Map.reject(fn {_key, value} -> is_nil(value) end)
    |> Map.merge(meta)
  else
    meta
  end
end
```

### 3.2 `imports_included`

**类型**：`boolean`

**含义**：查询结果是否实际包含了导入数据。

| 值 | 含义 |
|----|------|
| `true` | 结果是**原生数据 + 导入数据**的合并结果 |
| `false` | 结果**仅包含原生数据**（可能是降级，可能是用户没请求） |

**注意**：即使 `include.imports = true`，如果因为任何原因降级（如 `:unsupported_query`），`imports_included` 也会是 `false`。

### 3.3 `imports_skip_reason`

**类型**：`string | null`

**含义**：为什么没有包含导入数据的机器可读原因码。

| 值 | 含义 | 用户可采取的行动 |
|----|------|------------------|
| `null` | 已包含导入数据，或用户未请求 | 无 |
| `"unsupported_interval"` | 使用了小时/分钟维度 | 改为按日/周/月查询 |
| `"no_imported_data"` | 站点没有导入数据 | 先完成 GA 数据导入 |
| `"out_of_range"` | 查询范围无导入数据 | 调整查询时间范围 |
| `"unsupported_query"` | 查询模式不支持 | 简化维度/过滤器组合 |

### 3.4 `imports_warning`

**类型**：`string | null`

**含义**：面向用户的友好警告消息，解释为什么没有包含导入数据。

**警告消息映射**（`lib/plausible/stats/query_result.ex:18-24`）：

```elixir
@imports_warnings %{
  unsupported_query:
    "Imported stats are not included in the results because query parameters are not supported. " <>
      "For more information, see: https://plausible.io/docs/stats-api#filtering-imported-stats",
  unsupported_interval:
    "Imported stats are not included because the time dimension (i.e. the interval) is too short."
}
```

| `imports_skip_reason` | `imports_warning` 消息 |
|-----------------------|------------------------|
| `"unsupported_query"` | "Imported stats are not included in the results because query parameters are not supported..." |
| `"unsupported_interval"` | "Imported stats are not included because the time dimension (i.e. the interval) is too short." |
| `"no_imported_data"` | `null`（无警告消息） |
| `"out_of_range"` | `null`（无警告消息） |

**注意**：`no_imported_data` 和 `out_of_range` 不会产生 `imports_warning`，因为这些被认为是"正常"情况（用户可能知道自己在做什么）。

### 3.5 `metric_warnings`（指标级告警）

**类型**：`object | null`

**含义**：某些特定指标在当前查询上下文中存在限制或数据缺失的警告。

**结构**：
```json
{
  "metric_warnings": {
    "<metric_name>": {
      "code": "<warning_code>",
      "warning": "<human_readable_message>"
    }
  }
}
```

**警告类型总览**：

| 指标 | 警告代码 | 触发条件 | 消息 |
|------|----------|----------|------|
| `scroll_depth` | `no_imported_scroll_depth` | 包含导入数据但导入数据没有滚动深度信息 | "No imports with scroll depth data were found" |
| `bounce_rate` | `no_imported_bounce_rate` | 包含导入数据且使用了页面过滤器/维度 | "imported bounce_rate is not available when using a page filter" |
| `time_on_page` | `legacy_time_on_page_used` | 同时使用了新旧两种 time_on_page 计算方法 | "This period includes data calculated with the legacy time on page method up to..." |
| `revenue_*` | 多种 | EE 版本的收入指标相关 | 见下文 |

#### `scroll_depth` 警告

**代码位置**：`lib/plausible/stats/query_result.ex:275-279`

```elixir
defp metric_warning(:scroll_depth, %Query{} = query) do
  if query.include_imported and not Enum.any?(query.imports_in_range, & &1.has_scroll_depth) do
    @no_imported_scroll_depth_warning
  end
end
```

**触发条件**：
1. `include_imported = true`（正在使用导入数据）
2. 范围内的导入数据都没有 `has_scroll_depth = true`

**含义**：GA 导入数据中没有滚动深度指标，`scroll_depth` 指标将只包含原生数据部分。

#### `bounce_rate` 警告

**代码位置**：`lib/plausible/stats/query_result.ex:298-310`

```elixir
defp metric_warning(:bounce_rate, %Query{} = query) do
  page_filter_or_dimension? =
    Filters.filtering_on_dimension?(query, "event:page", behavioral_filters: :ignore) or
      "event:page" in query.dimensions

  if query.include_imported and page_filter_or_dimension? do
    @no_imported_bounce_rate_warning
  end
end
```

**触发条件**：
1. `include_imported = true`
2. 查询包含 `event:page` 过滤器 **或** 维度

**原因解释**（代码注释）：
> Native queries (i.e. ones that don't include imported data) allow querying bounce rate with an `event:page` filter or dimension. In those cases, an `event:page` gets treated as `visit:entry_page`. While theoretically possible, this behaviour does not yet exist for imported data, which is why we're returning a metric warning here.

**含义**：
- 原生查询中，`event:page` 过滤器会被当作 `visit:entry_page` 处理
- 导入数据查询**没有这个行为**
- 因此 `bounce_rate` 指标可能不准确（缺少导入数据部分）

#### `time_on_page` 警告

**代码位置**：`lib/plausible/stats/query_result.ex:281-296`

```elixir
defp metric_warning(:time_on_page, %Query{} = query) do
  case query.time_on_page_data do
    %{new_metric_visible: true, include_legacy_metric: true, cutoff: cutoff} ->
      cutoff_date =
        cutoff |> DateTime.shift_zone!(query.timezone) |> Calendar.strftime("%Y-%m-%d")

      %{
        code: :legacy_time_on_page_used,
        message:
          "This period includes data calculated with the legacy time on page method up to #{cutoff_date}"
      }

    _ ->
      nil
  end
end
```

**触发条件**：查询时间范围跨越了 `time_on_page` 计算方法变更的临界点。

**背景**：Plausible 在某个时间点改变了 `time_on_page` 的计算方法。如果查询范围同时包含两种方法的数据，会产生此警告。

#### 收入指标警告（EE 版本）

**代码位置**：`lib/plausible/stats/query_result.ex:250-273`

```elixir
on_ee do
  @revenue_metrics Plausible.Stats.Goal.Revenue.revenue_metrics()

  @revenue_metrics_warnings %{
    revenue_goals_unavailable:
      "The owner of this site does not have access to the revenue metrics feature.",
    no_single_revenue_currency:
      "Revenue metrics are null as there are multiple currencies for the selected event:goals.",
    no_revenue_goals_matching:
      "Revenue metrics are null as there are no matching revenue goals."
  }

  defp metric_warning(metric, %Query{} = query)
       when metric in @revenue_metrics do
    if query.revenue_warning do
      %{
        code: query.revenue_warning,
        warning: @revenue_metrics_warnings[query.revenue_warning]
      }
    else
      nil
    end
  end
end
```

**收入警告类型**：

| 警告代码 | 含义 |
|----------|------|
| `revenue_goals_unavailable` | 站点没有收入指标功能权限 |
| `no_single_revenue_currency` | 匹配的目标使用了多种货币，无法统一计算 |
| `no_revenue_goals_matching` | 没有匹配的收入目标 |

---

## 四、降级条件判定流程图

```
Dashboard API Request
        │
        ▼
┌───────────────────────┐
│ include.imports = true?│
└───────────────────────┘
        │
        ├─── No ──────────────────────────────────────►
        │                                              │
        ▼                                              ▼
┌───────────────────────┐                    ┌─────────────────────┐
│ schema_supports_      │                    │ imports_included:   │
│ interval?(query)       │                    │ false               │
│ (不是 minute/hour)     │                    │ (用户未请求)        │
└───────────────────────┘                    └─────────────────────┘
        │
        ├─── No ──────────────────────────────────────►
        │                                              │
        ▼                                              ▼
┌───────────────────────┐                    ┌─────────────────────┐
│ any_completed_        │                    │ imports_included:   │
│ imports?(site)         │                    │ false               │
│ (有已完成的导入)        │                    │ imports_skip_reason:│
└───────────────────────┘                    │ "unsupported_       │
        │                                    │ interval"           │
        ├─── No ──────────────────────────► └─────────────────────┘
        │
        ▼
┌───────────────────────┐
│ completed_imports_    │
│ in_query_range(...)   │
│ (范围内有导入数据)      │
└───────────────────────┘
        │
        ├─── No ──────────────────────────────────────►
        │                                              │
        ▼                                              ▼
┌───────────────────────┐                    ┌─────────────────────┐
│ schema_supports_      │                    │ imports_included:   │
│ query?(query)          │                    │ false               │
│ (维度/过滤器支持)        │                    │ imports_skip_reason:│
└───────────────────────┘                    │ "out_of_range"      │
        │                                    └─────────────────────┘
        ├─── No ──────────────────────────►
        │
        ▼                                              ▼
┌───────────────────────┐                    ┌─────────────────────┐
│ ✓ 所有条件满足         │                    │ imports_included:   │
│                       │                    │ false               │
│ imports_included:     │                    │ imports_skip_reason:│
│ true                  │                    │ "unsupported_query"  │
└───────────────────────┘                    │ imports_warning:    │
                                             │ "Imported stats are  │
                                             │ not included..."     │
                                             └─────────────────────┘
```

---

### 2.2 查询不支持导入数据的场景

**行为过滤器不支持**（`lib/plausible/stats/imported/base.ex:77-91`）：
```elixir
def decide_tables(query) do
  behavioral_filters = dimensions_used_in_filters(query.filters, behavioral_filters: :only)

  cond do
    # 行为过滤器无法通过聚合的导入数据模拟
    length(behavioral_filters) > 0 ->
      []
    ...
  end
end
```

**行为过滤器示例**：
- `visit:entry_page` - 入口页面（行为序列）
- `visit:exit_page` - 退出页面（行为序列）
- 自定义属性的复杂组合

**多维度组合限制**：
- 导入数据是预聚合的，一次查询只能从**单张** `imported_*` 表查询
- 如果查询涉及多个维度映射到不同表，则无法使用导入数据

### 2.3 核心合并函数 `merge_imported/3`

合并逻辑位于 `lib/plausible/stats/imported/imported.ex:218-311`。

#### 场景 1：无维度的聚合查询（`dimensions: []`）

使用 `CROSS JOIN` 合并两个单行结果：

```elixir
def merge_imported(q, site, %Query{dimensions: []} = query) do
  q = paginate_optimization(q, query)

  imported_q =
    site
    |> Imported.Base.query_imported(query)
    |> select_imported_metrics(query)
    |> paginate_optimization(query)

  from(
    s in subquery(q),           # 原生数据子查询
    cross_join: i in subquery(imported_q),  # 导入数据子查询
    select: %{}
  )
  |> select_joined_metrics(query)  # 合并指标
end
```

**SQL 模式**：
```sql
SELECT 
  -- 合并指标
  s.visitors + i.visitors AS visitors,
  s.visits + i.visits AS visits,
  ...
FROM (原生查询) AS s
CROSS JOIN (导入查询) AS i
```

#### 场景 2：按维度分组查询（有 dimensions）

使用 `FULL JOIN` 基于维度值匹配：

```elixir
def merge_imported(q, site, query) do
  if schema_supports_query?(query) do
    q = paginate_optimization(q, query)

    imported_q =
      site
      |> Imported.Base.query_imported(query)
      |> where([i], i.visitors > 0)
      |> group_imported_by(query)
      |> select_imported_metrics(query)
      |> paginate_optimization(query)

    from(s in subquery(q),
      full_join: i in subquery(imported_q),
      on: ^QueryBuilder.build_group_by_join(query),  # 维度值相等条件
      select: %{}
    )
    |> select_joined_dimensions(query)  # 合并维度值
    |> select_joined_metrics(query)     # 合并指标
  else
    q  # 不支持则只返回原生数据
  end
end
```

**JOIN 条件构建**（`lib/plausible/stats/sql/query_builder.ex:254-262`）：
```elixir
def build_group_by_join(%Query{dimensions: []}), do: true

def build_group_by_join(query) do
  query.dimensions
  |> Enum.map(fn dim ->
    dynamic([a, ..., b], field(a, ^shortname(query, dim)) == field(b, ^shortname(query, dim)))
  end)
  |> Enum.reduce(fn condition, acc -> dynamic([], ^acc and ^condition) end)
end
```

**SQL 模式**：
```sql
SELECT 
  -- 合并维度：优先使用原生数据的维度值，缺失时用导入数据
  if(not empty(s.source), s.source, i.source) AS source,
  
  -- 合并指标
  s.visitors + i.visitors AS visitors,
  ...
FROM (原生查询) AS s
FULL JOIN (导入查询) AS i
  ON s.source = i.source  -- 按维度值匹配
```

#### 场景 3：目标分组（`event:goal` 维度）

目标有特殊处理，因为目标可能是页面目标或事件目标，需要从不同的导入表查询：

```elixir
def merge_imported(q, site, %Query{dimensions: ["event:goal"]} = query) do
  goal_join_data = Plausible.Stats.Goals.goal_join_data(query)

  Imported.Base.decide_tables(query)
  |> Enum.map(fn
    "imported_custom_events" ->
      # 事件目标：从 imported_custom_events 查询
      Imported.Base.query_imported("imported_custom_events", site, query)
      |> where([i], i.visitors > 0)
      |> select_merge_as([i], %{
        dim0: fragment("indexOf(?, ?)", type(^goal_join_data.event_names_imports, {:array, :string}), i.name)
      })
      |> select_imported_metrics(query)
      |> group_by([], selected_as(:dim0))
      |> where([], selected_as(:dim0) != 0)

    "imported_pages" ->
      # 页面目标：从 imported_pages 查询，使用正则匹配
      Imported.Base.query_imported("imported_pages", site, query)
      |> where([i], i.visitors > 0)
      |> where([i], fragment("match(?, ?)", i.page, ?))  # 页面路径正则匹配
      ...
  end)
  |> Enum.reduce(q, fn imports_q, q ->
    naive_dimension_join(q, imports_q, query)
  end)
end
```

### 2.4 分页优化（高基数维度）

对于高基数维度（如 `event:page`），为避免 JOIN 性能问题，采用了**有损优化**：

```elixir
defp paginate_optimization(q, query) do
  if is_map(query.pagination) and can_order_by?(query) do
    n = (query.pagination.limit + query.pagination.offset) * 100
    q
    |> QueryBuilder.build_order_by(query)
    |> limit(^n)  # 限制子查询行数
  else
    q
  end
end
```

**优化说明**：
- 只对原生和导入数据各自的 **TOP N × 100** 行进行 JOIN
- 假设真正的 TOP N 不会出现在两个子查询的 TOP 100N 之外
- 这是一种**有损优化**，极端情况下可能导致排序不准确

---

## 三、去重机制

### 3.1 时间分区是主要去重策略

Plausible **没有**在查询层面做精确的行级别去重，而是依赖**时间分区**避免数据重叠。

**关键边界日期**（`lib/plausible/imported.ex:162-166`）：
```elixir
def get_cutoff_date(site) do
  Plausible.Sites.native_stats_start_date(site) ||
    DateTime.to_date(DateTime.now!(site.timezone))
end
```

`native_stats_start_date` 是站点**第一条原生事件**的日期：
```elixir
def native_stats_start_date(site) do
  Plausible.Stats.Clickhouse.pageview_start_date_local(site)
end
```

**时间分区示意**：
```
                    cutoff_date
                         │
    ┌────────────────────┼────────────────────┐
    │   导入数据日期      │    原生数据日期    │
    │  (imported_*表)    │  (events_v2表)    │
    │                    │                    │
    ◄─────── < cutoff_date ──────► >= cutoff_date ────►
    │                    │                    │
    │  来自 GA Data API  │  来自 Plausible   │
    │  日级聚合数据       │  实时事件数据      │
    └────────────────────┴────────────────────┘
```

### 3.2 导入时的日期范围检查

导入前会检查是否与已有导入数据重叠：

```elixir
def clamp_dates(site, start_date, end_date) do
  cutoff_date = get_cutoff_date(site)
  occupied_ranges = get_occupied_date_ranges(site)
  clamp_dates(occupied_ranges, cutoff_date, start_date, end_date)
end

defp find_free_ranges(import_range, d, [occupied_range | rest], result) do
  cond do
    # 跳过已被占用的范围
    Date.diff(occupied_range.last, d) <= 0 ->
      free_ranges(import_range, d, rest, result)
    
    # 与占用范围重叠或紧邻，跳到占用范围结束
    in_range?(d, occupied_range) or Date.diff(occupied_range.first, d) < 2 ->
      d = occupied_range.last
      free_ranges(import_range, d, rest, result)
    
    # 找到空闲范围
    true ->
      free_range = Date.range(d, occupied_range.first)
      result = result ++ [free_range]
      d = occupied_range.last
      free_ranges(import_range, d, rest, result)
  end
end
```

**设计意图**：
- 同一站点的多个导入**日期范围不能重叠**
- 导入数据必须在 `cutoff_date` **之前**
- 这样保证：`imported_*` 表的数据日期 < `events_v2` 表的数据日期

### 3.3 查询时的逻辑去重

由于时间分区的设计，查询时的 FULL JOIN 实际上：
- 对于任意维度值 + 日期组合，**只存在于原生数据或导入数据其中之一**
- `s.visitors + i.visitors` 其中一个必然是 0 或 NULL

**但如果出现日期重叠**（例如手动修改数据），会发生什么？

答案是：**指标会被重复计算**！这就是为什么导入时严格检查日期范围的原因。

### 3.4 同一日期的多批次导入

如果同一日期有多个导入批次（不同 `import_id`），查询时会合并：

```elixir
# lib/plausible/stats/imported/base.ex:62-75
def query_imported(table, site, query) do
  import_ids = Imported.complete_import_ids(site)
  
  from(i in table,
    where: i.site_id == ^site.id,
    where: i.import_id in ^import_ids,  # 包含所有完成的导入批次
    where: i.date >= ^date_from,
    where: i.date <= ^date_to,
    ...
  )
end
```

**多批次合并示意**：
```
imported_visitors 表：
┌───────────┬───────────┬──────────┬──────────┐
│ import_id │   date    │ visitors │ visits   │
├───────────┼───────────┼──────────┼──────────┤
│     1     │ 2024-01-01│   100    │   120    │  ← 批次1
│     2     │ 2024-01-01│    50    │    60    │  ← 批次2（同一日期！）
│     1     │ 2024-01-02│   200    │   220    │
└───────────┴───────────┴──────────┴──────────┘

查询时（GROUP BY date）：
2024-01-01: visitors = 100 + 50 = 150, visits = 120 + 60 = 180
```

**注意**：如果同一日期的同一维度值存在于多个导入批次，指标会被**相加**。这通常用于：
- 导入失败后重试（新 import_id）
- 补充导入不同维度的数据

---

## 四、指标对齐机制

### 4.1 指标映射：GA → Plausible

导入时，GA 指标会转换为 Plausible 的指标体系：

**GA4 → Plausible 映射**（`lib/plausible/imported/google_analytics4.ex:159-170`）：
```elixir
defp new_from_report(site_id, import_id, "imported_visitors", row) do
  %{
    site_id: site_id,
    import_id: import_id,
    date: get_date(row),
    visitors: row.metrics |> Map.fetch!("totalUsers") |> parse_number(),
    pageviews: row.metrics |> Map.fetch!("screenPageViews") |> parse_number(),
    bounces: row.metrics |> Map.fetch!("bounces") |> parse_number(),
    visits: row.metrics |> Map.fetch!("sessions") |> parse_number(),
    visit_duration: row.metrics |> Map.fetch!("userEngagementDuration") |> parse_number()
  }
end
```

**详细映射表**：

| Plausible 指标 | GA4 指标 | 说明 |
|---------------|----------|------|
| `visitors` | `totalUsers` | 独立用户 |
| `visits` | `sessions` | 会话数 |
| `pageviews` | `screenPageViews` | 页面浏览量 |
| `bounces` | `bounces` | 跳出数 |
| `visit_duration` | `userEngagementDuration` | 用户参与时长（秒） |

**注意**：
- GA4 的 `bounces` 定义可能与 Plausible 不同
- GA4 没有直接的 `exits` 指标（退出页面数据受限）
- 某些指标（如 `time_on_page`）在 GA 和 Plausible 间有差异

### 4.2 查询时的指标选择

导入数据查询时，根据表类型选择不同的指标计算方式：

**简单指标**（直接 `sum()`）：
```elixir
# lib/plausible/stats/imported/sql/expression.ex:42-72
defp select_metric(:visitors, _table, _query) do
  wrap_alias([i], %{visitors: sum(i.visitors)})
end

defp select_metric(:visits, _table, _query) do
  wrap_alias([i], %{visits: sum(i.visits)})
end

defp select_metric(:pageviews, "imported_custom_events", _query) do
  wrap_alias([i], %{pageviews: 0})  # 自定义事件没有 pageviews
end
```

**表特定的指标映射**：

| 表名 | `:visits` 来源 | `:events` 来源 |
|------|----------------|----------------|
| `imported_visitors` | `sum(i.visits)` | `sum(i.pageviews)` |
| `imported_entry_pages` | `sum(i.entrances)` | `sum(i.pageviews)` |
| `imported_exit_pages` | `sum(i.exits)` | `sum(i.pageviews)` |
| `imported_custom_events` | `sum(i.visits)` | `sum(i.events)` |

### 4.3 合并后的指标计算（原生 + 导入）

合并查询时，指标计算在 `select_joined_metrics/2` 中定义（`lib/plausible/stats/imported/sql/expression.ex:330-466`）。

#### 简单加法指标

```elixir
defp joined_metric(:visits, _query) do
  wrap_alias([s, i], %{visits: s.visits + i.visits})
end

defp joined_metric(:visitors, _query) do
  wrap_alias([s, i], %{visitors: s.visitors + i.visitors})
end

defp joined_metric(:events, _query) do
  wrap_alias([s, i], %{events: s.events + i.events})
end

defp joined_metric(:pageviews, _query) do
  wrap_alias([s, i], %{pageviews: s.pageviews + i.pageviews})
end
```

#### 复合比率指标（加权平均）

**跳出率 (bounce_rate)**：
```elixir
defp joined_metric(:bounce_rate, _query) do
  wrap_alias([s, i], %{
    bounce_rate:
      fragment(
        """
        if(? + ? > 0, round(100 * (? + (? * ? / 100)) / (? + ?)), 0)
        """,
        s.__internal_visits,   # 原生数据的分母（sessions）
        i.__internal_visits,   # 导入数据的分母
        i.bounces,             # 导入数据的跳出数
        s.bounce_rate,         # 原生数据的跳出率（已计算）
        s.__internal_visits,
        i.__internal_visits,
        s.__internal_visits
      )
  })
end
```

**公式解析**：
```
合并跳出率 = (原生跳出数 + 导入跳出数) / (原生会话数 + 导入会话数) × 100

其中：
- 原生跳出数 = 原生跳出率 × 原生会话数 / 100
- 导入跳出数 = i.bounces（直接存储）

SQL 展开：
if (原生会话数 + 导入会话数) > 0:
  round(100 × (导入跳出数 + (原生跳出率 × 原生会话数 / 100)) / (原生会话数 + 导入会话数))
else:
  0
```

**平均访问时长 (visit_duration)**：
```elixir
defp joined_metric(:visit_duration, _query) do
  wrap_alias([s, i], %{
    visit_duration:
      fragment(
        """
        if(
          ? + ? > 0,
          round((? + ? * ?) / (? + ?), 0),
          0
        )
        """,
        s.__internal_visits,
        i.__internal_visits,
        i.total_visit_duration,    # 导入数据的总时长
        s.visit_duration,          # 原生数据的平均时长
        s.__internal_visits,
        s.__internal_visits,
        i.__internal_visits
      )
  })
end
```

**公式解析**：
```
合并平均时长 = (原生总时长 + 导入总时长) / (原生会话数 + 导入会话数)

其中：
- 原生总时长 = 原生平均时长 × 原生会话数
- 导入总时长 = i.total_visit_duration（直接存储）
```

**每访问页面数 (views_per_visit)**：
```elixir
defp joined_metric(:views_per_visit, _query) do
  wrap_alias([s, i], %{
    views_per_visit:
      fragment(
        """
        if(? + ? > 0, round((? + ? * ?) / (? + ?), 2), 0)
        """,
        s.__internal_visits,
        i.__internal_visits,
        i.pageviews,
        s.views_per_visit,
        s.__internal_visits,
        i.__internal_visits,
        s.__internal_visits
      )
  })
end
```

#### 内部辅助字段

注意到复合指标使用了 `__internal_*` 字段，这些是在指标选择阶段预先计算的：

```elixir
# 跳出率的辅助字段
defp select_metric(:bounce_rate, _table, _query) do
  wrap_alias([i], %{
    bounces: sum(i.bounces),
    __internal_visits: sum(i.visits)  # 分母
  })
end

# 访问时长的辅助字段
defp select_metric(:visit_duration, _table, _query) do
  wrap_alias([i], %{
    visit_duration: fragment("ifNotFinite(round(? / ?), 0)", sum(i.visit_duration), sum(i.visits)),
    total_visit_duration: sum(i.visit_duration),  # 分子
    __internal_visits: sum(i.visits)              # 分母
  })
end
```

### 4.4 维度值的合并

FULL JOIN 后，维度值的选择策略：

```elixir
# lib/plausible/stats/imported/sql/expression.ex:306-328
def select_joined_dimensions(q, query) do
  Enum.reduce(query.dimensions, q, fn dimension, q ->
    select_joined_dimension(q, dimension, shortname(query, dimension))
  end)
end

defp select_joined_dimension(q, "visit:city", key) do
  select_merge_as(q, [s, i], %{
    key => fragment("greatest(?,?)", field(i, ^key), field(s, ^key))
  })
end

defp select_joined_dimension(q, "time:" <> _, key) do
  select_merge_as(q, [s, i], %{
    key => fragment("greatest(?, ?)", field(i, ^key), field(s, ^key))
  })
end

defp select_joined_dimension(q, _dimension, key) do
  select_merge_as(q, [s, i], %{
    key => fragment("if(empty(?), ?, ?)", field(s, ^key), field(i, ^key), field(s, ^key))
  })
end
```

**维度值选择规则**：

| 维度类型 | 选择策略 | SQL 表达式 |
|----------|----------|-----------|
| `visit:city` | 取较大值（数值） | `greatest(i.city, s.city)` |
| `time:*` | 取较大值（日期） | `greatest(i.date, s.date)` |
| 其他维度 | 优先原生，为空用导入 | `if(empty(s.value), i.value, s.value)` |

---

## 五、GA Data API 与本地存储的边界

### 5.1 清晰的职责划分

```
┌──────────────────────────────────────────────────────────────────────┐
│                         Google Analytics Data API                      │
│                         (仅在导入阶段使用)                               │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  调用时机：用户发起导入时（后台 Job）                                   │
│  调用频率：导入期间持续调用（分页获取数据）                               │
│  数据流向：GA API → 内存转换 → ClickHouse imported_* 表              │
│                                                                      │
│  涉及模块：                                                           │
│  - Plausible.Google.GA4.API                                          │
│  - Plausible.Google.GA4.HTTP                                         │
│  - Plausible.Google.GA4.ReportRequest                                │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
                              │
                              │ 导入完成后
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│                         ClickHouse 本地存储                             │
│                    (导入数据 + 原生数据，查询阶段使用)                    │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────┐    ┌─────────────────────┐                  │
│  │   imported_* 表      │    │ events_v2 /         │                  │
│  │  (GA 导入的聚合数据)  │    │ sessions_v2 表       │                  │
│  │                     │    │ (Plausible 原生事件) │                  │
│  ├─────────────────────┤    ├─────────────────────┤                  │
│  │ • 日级聚合           │    │ • 事件级别           │                  │
│  │ • 按维度预聚合        │    │ • 实时写入           │                  │
│  │ • 导入后只读          │    │ • 持续增长           │                  │
│  └─────────────────────┘    └─────────────────────┘                  │
│                                                                      │
│  查询时通过 SQL FULL JOIN 合并                                        │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 5.2 导入阶段的 API 交互

**GA4 报告请求构建**（`lib/plausible/google/ga4/report_request.ex`）：
```elixir
def build(date_range, property, dimensions, metrics, page_token, offset) do
  %{
    "dateRanges" => [%{"startDate" => format_date(date_range.first), "endDate" => format_date(date_range.last)}],
    "dimensions" => Enum.map(dimensions, &%{"name" => &1}),
    "metrics" => Enum.map(metrics, &%{"name" => &1}),
    "pageToken" => page_token,
    "offset" => to_string(offset),
    "limit" => to_string(@page_size),
    "orderBys" => ...
  }
end
```

**分页导入逻辑**（`lib/plausible/google/ga4/api.ex`）：
```elixir
defp fetch_and_persist_report(auth, property, request, table, persist_fn, opts) do
  # 循环获取数据直到没有下一页
  Stream.unfold({request, 0}, fn {req, offset} ->
    case run_report(auth, property, req, offset, opts) do
      {:ok, %{"rows" => rows} = response} ->
        persist_fn.(table, rows)  # 写入 ClickHouse
        
        next_page_token = response["nextPageToken"]
        offset = offset + length(rows)
        
        next_req = if next_page_token, do: %{req | "pageToken" => next_page_token}
        {{:ok, length(rows)}, {next_req, offset}}
      
      {:ok, %{}} ->  # 无数据
        nil
    end
  end)
end
```

### 5.3 查询阶段：完全本地化

查询阶段**不再调用任何外部 API**，所有数据都来自 ClickHouse：

```
Dashboard API Request
         │
         ▼
┌──────────────────────────────────────────────────────────┐
│  Plausible.Stats.SQL.QueryBuilder.build/2                │
│                                                           │
│  1. 解析查询参数                                           │
│  2. 检查是否需要包含导入数据 (include_imported)            │
│  3. 构建原生数据查询 (events_v2 / sessions_v2)            │
│  4. 如果 include_imported:                                │
│     a. 决定使用哪张 imported_* 表                          │
│     b. 构建导入数据查询                                     │
│     c. FULL JOIN 合并两个子查询                             │
│  5. 执行 ClickHouse 查询                                   │
└──────────────────────────────────────────────────────────┘
         │
         ▼
    ClickHouse (本地)
```

### 5.4 边界设计的优势

| 设计决策 | 优势 |
|----------|------|
| **导入后数据本地存储** | 查询性能不受外部 API 限制 |
| **日级预聚合** | 导入数据查询速度快，与原生数据复杂度相当 |
| **时间分区隔离** | 无需复杂的行级去重逻辑 |
| **查询时才合并** | 灵活控制是否使用导入数据（通过 `include.imports` 参数） |

### 5.5 边界设计的限制

| 限制 | 说明 |
|------|------|
| **导入数据只读** | 导入完成后无法修改，只能删除重导 |
| **只有日级粒度** | 无法按小时/分钟查询导入数据 |
| **预聚合限制** | 无法支持需要原始事件的行为分析（如漏斗） |
| **指标差异** | GA 和 Plausible 的指标定义不完全一致 |

---

## 六、代码路径总览

### 6.1 导入相关模块

```
lib/plausible/
├── imported/
│   ├── google_analytics4.ex    # GA4 导入实现（数据转换）
│   ├── universal_analytics.ex  # UA 导入实现
│   ├── importer.ex             # 导入行为定义（Behaviour）
│   ├── buffer.ex               # 批量写入缓冲
│   └── site_import.ex          # 导入记录 Schema
├── imported.ex                 # 导入上下文（管理导入状态）
└── google/
    ├── ga4/
    │   ├── api.ex              # GA4 API 封装
    │   ├── http.ex             # HTTP 客户端
    │   └── report_request.ex   # 报告请求构建
    └── api.ex                  # Google API 通用封装
```

### 6.2 查询合并相关模块

```
lib/plausible/stats/
├── imported/
│   ├── imported.ex             # 核心合并逻辑 (merge_imported/3)
│   ├── base.ex                 # 导入表查询基础（表选择、基础查询）
│   └── sql/
│       ├── expression.ex       # 指标选择、维度选择、合并计算
│       └── where_builder.ex    # 导入查询的 WHERE 条件构建
├── sql/
│   ├── query_builder.ex        # 主查询构建（调用 merge_imported）
│   └── expression.ex           # 原生数据的指标/维度表达式
├── query.ex                    # Query 结构体（include_imported 标志）
└── query_builder.ex            # Query 构建（设置 include_imported）
```

### 6.3 关键函数调用链

**查询时的合并路径**：
```
Plausible.Stats.Query.parse_and_build/3
  │
  └──▶ Plausible.Stats.QueryBuilder.build/3
         │
         └──▶ Plausible.Stats.Query.put_imported_opts/2  ◀─ 设置 include_imported
                │
         └──▶ Plausible.Stats.SQL.QueryBuilder.build/2
                │
                ├──▶ build_table_query(:events, ...)
                │      │
                │      └──▶ merge_imported(site, query)  ◀─ 核心合并
                │             │
                │             ├──▶ schema_supports_query?(query)
                │             ├──▶ Imported.Base.query_imported(site, query)
                │             ├──▶ group_imported_by(query)
                │             ├──▶ select_imported_metrics(query)
                │             └──▶ select_joined_metrics(query)
                │
                └──▶ build_table_query(:sessions, ...)
                       │
                       └──▶ merge_imported(site, query)  ◀─ 同上
```

---

## 七、总结

### 7.1 合并策略总览

| 方面 | 策略 | 实现位置 |
|------|------|----------|
| **数据合并** | FULL JOIN / CROSS JOIN 子查询 | `imported.ex:merge_imported/3` |
| **去重** | 时间分区（导入数据 < cutoff_date，原生 >=） | `imported.ex:get_cutoff_date/1` |
| **简单指标** | 直接相加 | `expression.ex:joined_metric/2` |
| **复合指标** | 加权平均（通过内部辅助字段） | `expression.ex:joined_metric/2` |
| **维度值** | 优先原生，缺失用导入 | `expression.ex:select_joined_dimension/3` |

### 7.2 去重的核心保障

Plausible 的去重不依赖查询时的复杂逻辑，而是依赖**数据导入时的约束**：

1. **时间分区**：导入数据必须在 `cutoff_date`（第一条原生事件日期）之前
2. **范围检查**：新导入的日期范围不能与已有导入重叠
3. **import_id 隔离**：不同批次的导入通过 import_id 区分，同一日期多批次时指标相加

### 7.3 GA Data API 的边界

- **导入阶段**：调用 GA Data API 获取聚合数据，转换后写入 ClickHouse 的 `imported_*` 表
- **查询阶段**：完全不涉及外部 API，所有合并都在 ClickHouse 本地通过 SQL 完成
- **数据所有权**：导入完成后，数据完全存储在 Plausible 基础设施中

### 7.4 潜在问题与注意事项

1. **指标定义差异**：GA 和 Plausible 的 `bounce_rate`、`visit_duration` 等指标定义可能不同
2. **高基数维度优化**：分页优化是有损的，极端场景下排序可能不准确
3. **无行为分析**：导入数据是聚合的，无法支持漏斗、路径分析等需要原始事件的功能
4. **手动数据修改风险**：如果手动修改 `imported_*` 表数据导致日期重叠，指标会被重复计算

---

## 附录：相关数据表 Schema

### imported_visitors（核心聚合表）
```sql
CREATE TABLE imported_visitors (
  site_id UInt64,
  import_id UInt64,
  date Date,
  visitors UInt64,
  pageviews UInt64,
  bounces UInt64,
  visits UInt64,
  visit_duration UInt64
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (site_id, import_id, date)
```

### 其他 imported_* 表
所有 `imported_*` 表都有类似结构，差异在于：
- 维度字段不同（如 `imported_sources` 有 `source`, `channel` 等）
- 指标字段略有差异（如 `imported_custom_events` 有 `events` 字段）

**共同特点**：
- 都有 `site_id`, `import_id`, `date` 字段
- 都是日级聚合数据
- 按 `(site_id, import_id, date, [维度字段])` 排序
