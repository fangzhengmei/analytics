# 目标转化与过滤查询分析报告

## 1. 整体架构与查询入口分类

### 1.1 三种主要查询入口

系统存在**三个独立的查询入口**，各自有不同的参数处理路径和验证规则：

| 入口类型 | 控制器 | 端点示例 | 解析器 | 目标存在性检查 |
|---------|--------|---------|--------|---------------|
| 内部仪表盘新API | `Api.StatsController.query/2` | `POST /api/stats/:domain/query` | `Dashboard.QueryParser` | **跳过** (`skip_goal_existence_check: true`) |
| 内部仪表盘旧API | `Api.StatsController.sources/2` 等 | `GET /api/stats/:domain/sources` | `Legacy.QueryBuilder` | **无显式验证**（依赖SQL匹配） |
| 公开API v1 | `Api.ExternalStatsController.aggregate/2` 等 | `GET /api/v1/stats/:domain/aggregate` | `Legacy.QueryBuilder` | **强制验证**（控制器层面） |

### 1.2 查询入口对比详解

#### 入口1：内部仪表盘新API (`/query`)

**位置**：`lib/plausible_web/controllers/api/stats_controller.ex:40-57`

```elixir
def query(conn, params) do
  site = conn.assigns.site
  now = conn.private[:now]

  with {:ok, %ParsedQueryParams{} = params} <- Dashboard.QueryParser.parse(params, now: now),
       {:ok, %Query{} = query} <- QueryBuilder.build(site, params, debug_metadata(conn)) do
    # ...
    json(conn, Plausible.Stats.query(site, query))
  else
    {:error, %QueryError{message: message}} -> bad_request(conn, message)
  end
end
```

**关键特征**：
- 使用 `Dashboard.QueryParser` 解析参数
- 该解析器设置 `skip_goal_existence_check: true`
- 目标过滤器不进行存在性验证，未配置的目标在SQL层面匹配不到数据

#### 入口2：内部仪表盘旧API (`/sources`, `/channels` 等)

**位置**：`lib/plausible_web/controllers/api/stats_controller.ex:59-97` (以`sources/2`为例)

```elixir
def sources(conn, params) do
  site = conn.assigns[:site]
  params = Map.put(params, "property", "visit:source")
  query = Query.from(site, params, debug_metadata: debug_metadata(conn))
  # ...
  %{results: results, meta: meta} = Stats.breakdown(site, query, metrics, pagination)
  # ...
end
```

**位置**：`lib/plausible/stats/query.ex:74-82`

```elixir
def from(site, params, opts \\ []) do
  Legacy.QueryBuilder.from(
    site,
    params,
    Keyword.get(opts, :debug_metadata, %{}),
    Keyword.get(opts, :now)
  )
end
```

**关键特征**：
- 使用 `Legacy.QueryBuilder` 直接构建 `%Query{}` 结构
- **不经过** `QueryBuilder.build/3` 的验证流程
- 目标过滤器没有显式的存在性验证，依赖SQL层面的匹配
- 预加载目标用于构建查询条件，但不验证请求的目标是否存在

#### 入口3：公开API v1 (`/api/v1/stats`)

**位置**：`lib/plausible_web/controllers/api/external_stats_controller.ex:33-55` (以`breakdown/2`为例)

```elixir
def breakdown(conn, params) do
  site = Repo.preload(conn.assigns.site, :owners)

  with :ok <- validate_period(params),
       :ok <- validate_date(params),
       :ok <- validate_property(params),
       query <- Query.from(site, params, debug_metadata: debug_metadata(conn)),
       :ok <- validate_filters(site, query.filters),  # 强制验证！
       {:ok, metrics} <- parse_and_validate_metrics(params, query),
       {:ok, limit} <- validate_or_default_limit(params),
       :ok <- ensure_custom_props_access(site, query) do
    # ...
  end
end
```

**位置**：`lib/plausible_web/controllers/api/external_stats_controller.ex:330-359`

```elixir
defp validate_filters(site, filters) do
  Enum.reduce_while(filters, :ok, fn filter, _ ->
    case validate_filter(site, filter) do
      :ok -> {:cont, :ok}
      {:error, reason} -> {:halt, {:error, reason}}
    end
  end)
end

defp validate_filter(site, [_type, "event:goal", goal_filter | _rest]) do
  site = Plausible.Repo.preload(site, :team)
  props_available? = Plausible.Billing.Feature.Props.check_availability(site.team) == :ok

  configured_goals =
    site
    |> Plausible.Goals.for_site(include_goals_with_custom_props?: props_available?)
    |> Enum.map(& &1.display_name)

  goals_in_filter = List.wrap(goal_filter)

  if found = Enum.find(goals_in_filter, &(&1 not in configured_goals)) do
    msg =
      goal_not_configured_message(found) <>
        "Find out how to configure goals here: https://plausible.io/docs/stats-api#filtering-by-goals"

    {:error, msg}
  else
    :ok
  end
end
```

**关键特征**：
- 控制器层面有**强制的目标存在性验证**
- 使用 `Enum.find(goals_in_filter, &(&1 not in configured_goals))` 检查
- 未配置的目标会立即返回错误响应，**不会到达SQL执行阶段**
- 这是**唯一**会在查询执行前拒绝未配置目标的入口

---

## 2. 前端URL参数到后端查询的完整路径

### 2.1 完整数据流图

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              前端层 (Browser)                                         │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌─────────────────┐                                                                │
│  │  URL地址栏       │                                                                │
│  │  可读格式参数     │                                                                │
│  │  ?f=is,goal,Purchase&period=30d                  │                                                                │
│  └────────┬────────┘                                                                │
│           │                                                                          │
│           ▼                                                                          │
│  ┌──────────────────────────────────────────────────────────────┐                  │
│  │  url-search-params.ts                                         │                  │
│  │  - parseSearch(): 解析URL参数 → DashboardState                │                  │
│  │  - stringifySearch(): DashboardState → URL参数                │                  │
│  │  - 自定义序列化格式 (非JSON)                                   │                  │
│  └──────────────────────────────┬───────────────────────────────┘                  │
│                                 │                                                    │
│                                 ▼                                                    │
│  ┌──────────────────────────────────────────────────────────────┐                  │
│  │  dashboard-state.ts                                            │                  │
│  │  - DashboardState: period, date, filters, labels 等           │                  │
│  │  - filters: [["is", "event:goal", ["Purchase"]]]              │                  │
│  └──────────────────────────────┬───────────────────────────────┘                  │
│                                 │                                                    │
│                                 ▼                                                    │
│  ┌──────────────────────────────────────────────────────────────┐                  │
│  │  api.ts                                                        │                  │
│  │  - dashboardStateToParams(): 转换为API请求参数                 │                  │
│  │  - serializeApiFilters(): 过滤器序列化为JSON字符串            │                  │
│  │  - stats(): POST /api/stats/:domain/query (JSON body)        │                  │
│  │  - get(): GET /api/stats/:domain/sources (URL query params)   │                  │
│  └──────────────────────────────┬───────────────────────────────┘                  │
│                                 │                                                    │
└─────────────────────────────────┼────────────────────────────────────────────────────┘
                                  │
                                  │ HTTP Request
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              后端层 (Phoenix Server)                                  │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐                  │
│  │  Phoenix Router / Plug                                        │                  │
│  │  - 解析URL查询参数 (conn.query_params)                        │                  │
│  │  - 解析JSON请求体 (conn.body_params)                          │                  │
│  │  - 参数自动解码 (URI.decode_www_form)                         │                  │
│  └──────────────────────────────┬───────────────────────────────┘                  │
│                                 │                                                    │
│                                 ▼                                                    │
│          ┌──────────────────────┴──────────────────────┐                          │
│          │                                             │                          │
│          ▼                                             ▼                          │
│  ┌───────────────────┐                    ┌──────────────────────────┐          │
│  │  新API /query     │                    │  旧API /sources等        │          │
│  │  (POST JSON)      │                    │  (GET query params)      │          │
│  └─────────┬─────────┘                    └─────────────┬────────────┘          │
│            │                                            │                         │
│            ▼                                            ▼                         │
│  ┌─────────────────────────┐              ┌─────────────────────────────┐       │
│  │ Dashboard.QueryParser   │              │ Legacy.QueryBuilder         │       │
│  │ - 解析JSON格式参数       │              │ - 解析URL query params      │       │
│  │ - skip_goal_existence_  │              │ - 直接构建%Query{}          │       │
│  │   check: true           │              │ - 不经过QueryBuilder.build   │       │
│  └─────────┬───────────────┘              └─────────────┬───────────────┘       │
│            │                                            │                         │
│            ▼                                            │                         │
│  ┌─────────────────────────┐                           │                         │
│  │ QueryBuilder.build      │                           │                         │
│  │ - 14项验证检查          │                           │                         │
│  │ - 目标检查被跳过        │                           │                         │
│  └─────────┬───────────────┘                           │                         │
│            │                                            │                         │
│            └──────────────────────┬─────────────────────┘                         │
│                                   │                                                  │
│                                   ▼                                                  │
│                          ┌─────────────────┐                                        │
│                          │  %Query{} 结构   │                                        │
│                          │  - filters       │                                        │
│                          │  - preloaded_    │                                        │
│                          │    goals         │                                        │
│                          └────────┬────────┘                                        │
│                                   │                                                  │
│                                   ▼                                                  │
│                          ┌─────────────────┐                                        │
│                          │ SQL.QueryBuilder │                                        │
│                          │ - 构建Ecto查询   │                                        │
│                          │ - Goals.add_     │                                        │
│                          │   filter()       │                                        │
│                          └────────┬────────┘                                        │
│                                   │                                                  │
│                                   ▼                                                  │
│                          ┌─────────────────┐                                        │
│                          │  ClickHouse     │                                        │
│                          │  - 执行SQL查询   │                                        │
│                          │  - 未配置目标    │                                        │
│                          │    匹配不到数据  │                                        │
│                          └─────────────────┘                                        │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 前端URL参数编码详解

**位置**：`assets/js/dashboard/util/url-search-params.ts`

#### 自定义序列化格式（可读性优先）

为了URL的可读性，系统使用**自定义逗号分隔格式**而非标准JSON：

```typescript
// 序列化示例
// 输入: [["is", "event:goal", ["Purchase", "Signup"]]]
// 输出: f=is,event:goal,Purchase,Signup

export function serializeFilter([operator, dimension, clauses]: Filter) {
  const serializedFilter = [
    encodeURIComponentPermissive(operator, NOT_URL_ENCODED_CHARACTERS),
    encodeURIComponentPermissive(dimension, NOT_URL_ENCODED_CHARACTERS),
    ...clauses.map((clause) =>
      encodeURIComponentPermissive(
        clause.toString(),
        NOT_URL_ENCODED_CHARACTERS
      )
    )
  ].join(',')
  return serializedFilter
}
```

**关键设计**：
- `NOT_URL_ENCODED_CHARACTERS = ':/'` - 这些字符不进行URL编码以提高可读性
- 格式：`f=<operator>,<dimension>,<clause1>,<clause2>,...`
- 示例：`?f=is,event:page,/blog,/news&period=30d`

#### API请求时的参数转换

**位置**：`assets/js/dashboard/api.ts:60-103`

```typescript
export function dashboardStateToParams(
  dashboardState: DashboardState,
  extraQuery: unknown[] = []
): Record<string, string> {
  const queryObj: Record<string, string> = {}
  if (dashboardState.period) {
    queryObj.period = dashboardState.period
  }
  // ...
  if (dashboardState.filters) {
    queryObj.filters = serializeApiFilters(dashboardState.filters)  // 序列化为JSON!
  }
  // ...
  return queryObj
}
```

**重要差异**：
- URL地址栏：自定义逗号分隔格式
- API请求：JSON格式（通过`serializeApiFilters`）

### 2.3 后端参数解析路径

#### 路径A：新API `/query` (POST JSON)

```
前端发送:
POST /api/stats/example.com/query
Content-Type: application/json
Body: {
  "date_range": "30d",
  "metrics": ["visitors", "conversion_rate"],
  "filters": [["is", "event:goal", ["Purchase"]]],
  "dimensions": ["event:goal"]
}

后端处理:
1. Phoenix解析JSON → conn.body_params (Map)
2. Dashboard.QueryParser.parse(params)
   - parse_input_date_range("30d") → {:last_n_days, 30}
   - ApiQueryParser.parse_filters([["is", "event:goal", ["Purchase"]]])
     → [[:is, "event:goal", ["Purchase"]]]
   - 设置 skip_goal_existence_check: true
3. QueryBuilder.build(site, parsed_params)
   - 所有验证执行...
   - validate_filtered_goals_exist → 跳过 (skip_goal_existence_check: true)
4. 返回 %Query{}
```

#### 路径B：旧API `/sources` (GET query params)

```
前端发送:
GET /api/stats/example.com/sources?period=30d&filters=%5B%5B%22is%22%2C%22event%3Agoal%22%2C%5B%22Purchase%22%5D%5D%5D

后端处理:
1. Phoenix解析URL query params → conn.query_params
   - "filters" = "[[\"is\",\"event:goal\",[\"Purchase\"]]]" (JSON字符串)
2. Query.from(site, params) → Legacy.QueryBuilder.from(...)
   - put_parsed_filters: Filters.parse(params["filters"])
     → 解析JSON字符串 → [[:is, "event:goal", ["Purchase"]]]
   - 直接构建%Query{}，**不经过QueryBuilder.build的验证**
3. 返回 %Query{} (无目标存在性验证)
```

#### 路径C：公开API v1 `/api/v1/stats/breakdown`

```
前端发送:
GET /api/v1/stats/example.com/breakdown
  ?period=30d
  &property=visit:source
  &filters=%5B%5B%22is%22%2C%22event%3Agoal%22%2C%5B%22Purchase%22%5D%5D%5D

后端处理:
1. Phoenix解析URL query params → conn.query_params
2. Query.from(site, params) → Legacy.QueryBuilder.from(...)
   - 同路径B，构建%Query{}
3. ExternalStatsController.validate_filters(site, query.filters)
   - 遍历filters，检查每个"event:goal"过滤器
   - 从数据库加载已配置的目标
   - Enum.find(goals_in_filter, &(&1 not in configured_goals))
   - 找到未配置目标 → {:error, msg} → 返回400错误
4. 验证通过才执行查询
```

---

## 3. 目标存在性检查的真相

### 3.1 之前报告的错误口径

**错误说法**：
> "QueryBuilder.validate_filtered_goals_exist 会验证目标存在性，`:is`操作符需要配置的目标名"

**实际真相**：
这个验证**只在特定条件下执行**，且三个查询入口的行为完全不同。

### 3.2 验证逻辑的实际执行条件

**位置**：`lib/plausible/stats/query_builder.ex:409-431`

```elixir
defp validate_filtered_goals_exist(_query, %ParsedQueryParams{skip_goal_existence_check: true}),
  do: :ok  # 直接跳过！

defp validate_filtered_goals_exist(query, %ParsedQueryParams{}) do
  # 仅检查 :is 操作符的 event:goal 过滤器
  goal_filter_clauses =
    query.filters
    |> Filters.all_leaf_filters()
    |> Enum.flat_map(fn
      [:is, "event:goal", clauses] -> clauses  # 只检查 :is
      _ -> []  # :contains 等其他操作符不检查
    end)

  # 验证是否在 preloaded_goals 中
  # ...
end
```

**关键条件**：
1. `skip_goal_existence_check` 必须为 `false`（默认值）
2. 仅检查 `:is` 操作符，**不检查** `:contains` 操作符
3. 仅 `Dashboard.QueryParser` 会设置 `skip_goal_existence_check: true`

### 3.3 三个入口的验证行为总结

| 入口 | 是否经过 QueryBuilder.build | skip_goal_existence_check | 实际是否验证目标 |
|------|----------------------------|---------------------------|-----------------|
| 内部新API `/query` | **是** | `true` (设置于 Dashboard.QueryParser) | **否，跳过** |
| 内部旧API `/sources`等 | **否** (直接用Legacy.QueryBuilder) | 不相关 | **否，无此验证** |
| 公开API v1 | **否** (直接用Legacy.QueryBuilder) | 不相关 | **是，控制器层面验证** |

### 3.4 公开API v1的验证逻辑详解

**位置**：`lib/plausible_web/controllers/api/external_stats_controller.ex:339-359`

```elixir
defp validate_filter(site, [_type, "event:goal", goal_filter | _rest]) do
  site = Plausible.Repo.preload(site, :team)
  props_available? = Plausible.Billing.Feature.Props.check_availability(site.team) == :ok

  configured_goals =
    site
    |> Plausible.Goals.for_site(include_goals_with_custom_props?: props_available?)
    |> Enum.map(& &1.display_name)

  goals_in_filter = List.wrap(goal_filter)

  if found = Enum.find(goals_in_filter, &(&1 not in configured_goals)) do
    msg =
      goal_not_configured_message(found) <>
        "Find out how to configure goals here: https://plausible.io/docs/stats-api#filtering-by-goals"

    {:error, msg}  # 返回错误，不会执行SQL
  else
    :ok
  end
end
```

**与QueryBuilder验证的区别**：
1. **位置不同**：控制器层面 vs 查询构建层面
2. **覆盖范围**：公开API验证所有`event:goal`过滤器，QueryBuilder只验证`:is`操作符
3. **错误处理**：公开API立即返回400错误，QueryBuilder在构建阶段返回错误

---

## 4. 模块协作与数据结构

### 4.1 核心模块职责

| 模块 | 文件位置 | 主要职责 | 入口使用情况 |
|------|---------|---------|-------------|
| `Dashboard.QueryParser` | `lib/plausible/stats/dashboard/query_parser.ex` | 解析仪表盘API的JSON参数，设置`skip_goal_existence_check: true` | **仅**新API `/query` |
| `ApiQueryParser` | `lib/plausible/stats/api_query_parser.ex` | 解析标准API参数格式，通用解析器 | 被Dashboard.QueryParser内部调用 |
| `Legacy.QueryBuilder` | `lib/plausible/stats/legacy/legacy_query_builder.ex` | 直接从URL params构建Query，**绕过**大部分验证 | 旧API、公开API v1 |
| `QueryBuilder` | `lib/plausible/stats/query_builder.ex` | 执行14项验证，构建最终Query | **仅**新API `/query` |

### 4.2 ParsedQueryParams 结构

**位置**：`lib/plausible/stats/parsed_query_params.ex:4-22`

```elixir
defstruct input_date_range: nil,
          relative_date: nil,
          metrics: [],
          filters: [],
          dimensions: [],
          order_by: nil,
          pagination: nil,
          now: nil,
          include: %Plausible.Stats.QueryInclude{},
          skip_goal_existence_check: false  # 关键标志！
```

**关键方法**：

```elixir
# 检查是否有目标转化过滤器
def conversion_goal_filter?(%__MODULE__{filters: filters}) do
  Plausible.Stats.Filters.filtering_on_dimension?(filters, "event:goal",
    max_depth: 0,
    behavioral_filters: :ignore
  )
end
```

### 4.3 Query 结构

**位置**：`lib/plausible/stats/query.ex:4-34`

```elixir
defstruct utc_time_range: nil,
          comparison_utc_time_range: nil,
          interval: nil,
          input_date_range: nil,
          dimensions: [],
          filters: [],
          sample_threshold: 20_000_000,
          imports_exist: false,
          imports_in_range: [],
          include_imported: false,
          skip_imported_reason: nil,
          now: nil,
          metrics: [],
          order_by: nil,
          timezone: nil,
          legacy_breakdown: false,
          preloaded_goals: [],  # 预加载的目标（用于构建查询条件，非验证）
          include: Plausible.Stats.ApiQueryParser.default_include(),
          debug_metadata: %{},
          pagination: nil,
          revenue_currencies: %{},
          revenue_warning: nil,
          site_id: nil,
          consolidated_site_ids: nil,
          site_native_stats_start_at: nil,
          time_on_page_data: %{},
          sql_join_type: :left,
          smear_session_metrics: false
```

### 4.4 预加载目标的作用

**位置**：`lib/plausible/stats/goals.ex:14-34`

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
    %{
      all: [],
      matching_toplevel_filters: []
    }
  end
end
```

**预加载目标的用途**：
1. **构建SQL查询条件** (`Goals.add_filter/3`) - 根据目标类型生成不同的WHERE条件
2. **派生名称过滤优化** (`SQL.WhereBuilder.derived_name_filter/1`) - 限制查询的事件名称范围
3. **ARRAY JOIN分组数据** (`Goals.goal_join_data/1`) - 用于按目标分组
4. **验证**（仅QueryBuilder在`skip_goal_existence_check: false`时）

**重要**：预加载目标**不代表**会验证请求的目标是否存在。预加载的是**所有已配置的目标**，用于构建查询条件。

---

## 5. 边界条件与验证逻辑

### 5.1 QueryBuilder的14项验证（仅新API `/query`）

**位置**：`lib/plausible/stats/query_builder.ex:31-60`

```elixir
def build(site, %ParsedQueryParams{} = parsed_query_params, debug_metadata) do
  with {:ok, parsed_query_params} <- resolve_segments_in_filters(parsed_query_params, site),
       query = do_build(parsed_query_params, site, debug_metadata),
       :ok <- validate_order_by(query),                           # 1. 排序字段验证
       :ok <- validate_custom_props_access(site, query),         # 2. 自定义属性访问
       :ok <- validate_case_sensitive_filter_modifier(query),     # 3. 大小写修饰符
       :ok <- validate_toplevel_only_filter_dimension(query),     # 4. 仅顶层维度
       :ok <- validate_time_dimension_granularity(query),         # 5. 时间粒度
       :ok <- validate_special_metrics_filters(query),            # 6. 特殊指标过滤器
       :ok <- validate_behavioral_filters(query),                 # 7. 行为过滤器
       :ok <- validate_filtered_goals_exist(query, parsed_query_params),  # 8. 目标存在性
       :ok <- validate_revenue_metrics_access(site, query),       # 9. 收入指标访问
       :ok <- validate_metrics(query),                             # 10. 指标验证
       :ok <- validate_include(query) do                           # 11-14? 实际计数
    # ...
  end
end
```

### 5.2 关键验证详解

#### 验证1：仅顶层可用的维度

**位置**：`lib/plausible/stats/query_builder.ex:314-331`

```elixir
@only_toplevel ["event:goal", "event:hostname"]

defp validate_toplevel_only_filter_dimension(query) do
  not_toplevel =
    query.filters
    |> Filters.dimensions_used_in_filters(min_depth: 1, behavioral_filters: :ignore)
    |> Enum.filter(&(&1 in @only_toplevel))

  if Enum.count(not_toplevel) > 0 do
    {:error, %QueryError{
      code: :invalid_filters,
      message: "Dimension `#{List.first(not_toplevel)}` can only be filtered at the top level."
    }}
  else
    :ok
  end
end
```

**触发条件**：
- `event:goal` 或 `event:hostname` 出现在嵌套过滤器中（min_depth: 1）
- 例如：`[:and, [[:is, "visit:country", ["US"]], [:is, "event:goal", ["Purchase"]]]]`

**原因**：
- 目标过滤需要特殊的SQL构建逻辑，嵌套时难以正确处理
- 主机名过滤可能涉及特殊的路由逻辑

#### 验证2：行为过滤器限制

**位置**：`lib/plausible/stats/query_builder.ex:370-407`

```elixir
defp validate_behavioral_filters(query) do
  query.filters
    |> Filters.traverse(0, fn behavioral_depth, operator ->
      if operator in [:has_done, :has_not_done] do
        behavioral_depth + 1
      else
        behavioral_depth
      end
    end)
    |> Enum.reduce_while(:ok, fn {[_operator, dimension | _rest], behavioral_depth}, :ok ->
      cond do
        behavioral_depth == 0 -> {:cont, :ok}
        behavioral_depth > 1 ->
          {:halt, {:error, %QueryError{
            code: :invalid_filters,
            message: "Behavioral filters cannot be nested."
          }}}
        not String.starts_with?(dimension, "event:") ->
          {:halt, {:error, %QueryError{
            code: :invalid_filters,
            message: "Behavioral filters can only be used with event dimension filters."
          }}}
        true -> {:cont, :ok}
      end
    end)
end
```

**限制规则**：
1. **不能嵌套**：`[:has_done, [:has_done, ...]]` → 错误
2. **只能用于事件维度**：`[:has_done, [:is, "visit:country", ...]]` → 错误

#### 验证3：特殊指标与深度过滤器冲突

**位置**：`lib/plausible/stats/query_builder.ex:349-368`

```elixir
@special_metrics [:conversion_rate, :group_conversion_rate]

defp validate_special_metrics_filters(query) do
  special_metric? = Enum.any?(@special_metrics, &(&1 in query.metrics))

  deep_custom_property? =
    query.filters
      |> Filters.dimensions_used_in_filters(min_depth: 1)
      |> Enum.any?(fn dimension -> String.starts_with?(dimension, "event:props:") end)

  if special_metric? and deep_custom_property? do
    {:error, %QueryError{
      code: :invalid_filters,
      message: "When `conversion_rate` metrics are used, custom property filters can only be used on top level."
    }}
  else
    :ok
  end
end
```

**技术原因**：
- 转化率计算需要特殊的会话级聚合
- 深度嵌套的自定义属性过滤器在子查询中难以正确处理
- SQL复杂度与性能之间的权衡

#### 验证4：指标依赖验证

**位置**：`lib/plausible/stats/query_builder.ex:481-570`

```elixir
# 转化率指标
defp validate_metric(metric, query) when metric in [:conversion_rate, :group_conversion_rate] do
  if Enum.member?(query.dimensions, "event:goal") or
       Filters.filtering_on_dimension?(query, "event:goal", behavioral_filters: :ignore) do
    :ok
  else
    {:error, %QueryError{
      code: :invalid_metrics,
      message: "Metric `#{metric}` can only be queried with event:goal filters or dimensions."
    }}
  end
end

# 滚动深度指标
defp validate_metric(:scroll_depth = metric, query) do
  page_dimension? = Enum.member?(query.dimensions, "event:page")
  toplevel_page_filter? = not is_nil(Filters.get_toplevel_filter(query, "event:page"))

  if page_dimension? or toplevel_page_filter? do
    :ok
  else
    {:error, %QueryError{
      code: :invalid_metrics,
      message: "Metric `#{metric}` can only be queried with event:page filters or dimensions."
    }}
  end
end
```

### 5.3 过滤器深度计算

**位置**：`lib/plausible/stats/filters/filters.ex:91-115`

```elixir
def dimensions_used_in_filters(filters, opts \\ []) do
  min_depth = Keyword.get(opts, :min_depth, 0)
  max_depth = Keyword.get(opts, :max_depth, 999)
  behavioral_filter_option = Keyword.get(opts, :behavioral_filters, nil)

  filters
    |> traverse(
      {0, false},
      fn {depth, is_behavioral_filter}, operator ->
        {depth + 1, is_behavioral_filter or operator in [:has_done, :has_not_done]}
      end
    )
    |> Enum.filter(fn {_filter, {depth, is_behavioral_filter}} ->
      # ...
    end)
end
```

**深度计算规则**：
- 顶层过滤器：depth = 0
- 逻辑组合内的过滤器：depth = 1 及以上
- 行为过滤器（`:has_done`, `:has_not_done`）会标记 `is_behavioral_filter = true`

**示例**：
```elixir
# 顶层 - depth 0
[[:is, "event:goal", ["Purchase"]]]

# 嵌套在AND内 - depth 1
[[:and, [
  [:is, "visit:country", ["US"]],       # depth 1
  [:is, "event:goal", ["Purchase"]]     # depth 1 (触发错误!)
]]]

# 行为过滤器内 - 标记为 behavioral
[[:has_done, [:is, "event:name", ["AddToCart"]]]]
```

---

## 6. 性能取舍与优化策略

### 6.1 派生名称过滤（Derived Name Filter）

**位置**：`lib/plausible/stats/sql/where_builder.ex:29-56`

```elixir
def derived_name_filter(query) do
  cond do
    # 情况1：有目标过滤器 - 不添加额外限制
    Plausible.Stats.Filters.filtering_on_dimension?(query.filters, "event:goal") ->
      true

    # 情况2：有预加载目标 - 限制为相关事件名称
    query.preloaded_goals.matching_toplevel_filters != [] ->
      names = goal_event_names(query.preloaded_goals.matching_toplevel_filters)
      dynamic([e], e.name in ^names)

    # 情况3：无特殊需求 - 排除engagement事件
    :time_on_page not in query.metrics and :scroll_depth not in query.metrics ->
      dynamic([e], e.name != "engagement")

    true ->
      true
  end
end
```

**性能优化效果**：
- **情况2**：通过预加载的目标知道相关的事件名称，添加 `name IN (...)` 条件
- **情况3**：`engagement` 事件仅用于滚动深度和页面停留时间，默认排除以减少扫描数据量
- **情况1**：目标过滤器本身会添加精确的名称条件，无需额外限制

### 6.2 目标过滤的SQL构建

**位置**：`lib/plausible/stats/goals.ex:52-64`

```elixir
def add_filter(query, [operation, "event:goal", clauses | _] = filter, opts \\ [])
    when operation in [:is, :contains] do
  imported? = Keyword.get(opts, :imported?, false)

  Enum.reduce(clauses, false, fn clause, dynamic_statement ->
    condition =
      query.preloaded_goals.all
        |> filter_preloaded(filter, clause)  # 从预加载列表筛选匹配的目标
        |> build_condition(imported?)       # 构建该目标的SQL条件

    dynamic([e], ^condition or ^dynamic_statement)  # OR组合所有目标条件
  end)
end
```

**目标类型对应的SQL条件**：

```elixir
# 事件目标
defp goal_condition(:event, goal, _) do
  name_condition = dynamic([e], e.name == ^goal.event_name)

  if Plausible.Goal.has_custom_props?(goal) do
    custom_props_condition = build_custom_props_condition(goal.custom_props)
    dynamic([e], ^name_condition and ^custom_props_condition)
  else
    name_condition
  end
end

# 页面目标
defp goal_condition(:page, goal, false = _imported?) do
  name_condition = dynamic([e], e.name == "pageview")
  pathname_condition = page_path_condition(goal.page_path, _imported? = false)
  base_condition = dynamic([e], ^pathname_condition and ^name_condition)
  # ... 自定义属性条件
end

# 滚动目标
defp goal_condition(:scroll, goal, false = _imported?) do
  pathname_condition = page_path_condition(goal.page_path, _imported? = false)
  name_condition = dynamic([e], e.name == "engagement")

  scroll_condition =
    dynamic([e], e.scroll_depth <= 100 and e.scroll_depth >= ^goal.scroll_threshold)
  # ...
end
```

**性能考虑**：
- 预加载目标避免了运行时的数据库查询
- 使用Ecto Dynamic构建条件，保持查询的可组合性
- OR组合允许一次查询匹配多个目标

### 6.3 ARRAY JOIN 分组优化

**位置**：`lib/plausible/stats/sql/query_builder.ex:173-197`

```elixir
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

**位置**：`lib/plausible/stats/goals.ex:80-118`

```elixir
@spec goal_join_data(Plausible.Stats.Query.t()) :: goal_join_data()
def goal_join_data(query) do
  goals = query.preloaded_goals.matching_toplevel_filters

  goals
    |> Enum.with_index(1)
    |> Enum.reduce(
      %{
        indices: [],
        types: [],
        event_names_imports: [],
        event_names_by_type: [],
        page_regexes: [],
        scroll_thresholds: [],
        custom_props_keys: [],
        custom_props_values: []
      },
      fn {goal, idx}, acc ->
        goal_type = Plausible.Goal.type(goal)
        {prop_keys, prop_values} = Enum.unzip(goal.custom_props)

        %{
          indices: [idx | acc.indices],
          types: [to_string(goal_type) | acc.types],
          event_names_imports: [to_string(goal.event_name) | acc.event_names_imports],
          event_names_by_type: [event_name_by_type(goal_type, goal) | acc.event_names_by_type],
          page_regexes: [page_regex_for_goal(goal_type, goal) | acc.page_regexes],
          scroll_thresholds: [goal.scroll_threshold | acc.scroll_thresholds],
          custom_props_keys: [prop_keys | acc.custom_props_keys],
          custom_props_values: [prop_values | acc.custom_props_values]
        }
      end
    )
    |> Enum.map(fn {key, list} -> {key, Enum.reverse(list)} end)
    |> Map.new()
end
```

**ClickHouse ARRAY JOIN 优化**：
- 将所有目标配置转换为并行数组
- 使用 `hints: "ARRAY"` 提示ClickHouse优化器
- 一次JOIN完成所有目标的分组，避免多次扫描
- 按是否有自定义属性选择不同的JOIN表达式

### 6.4 会话表时间范围优化

**位置**：`lib/plausible/stats/sql/where_builder.ex:100-120`

```elixir
defp filter_time_range(:sessions, query) do
  {first_datetime, last_datetime} = utc_boundaries(query)

  dynamic(
    [s],
    # 额外添加 start 条件以确保主键过滤
    s.start >= ^NaiveDateTime.add(first_datetime, -7, :day) and
      s.timestamp >= ^first_datetime and
      s.start <= ^last_datetime
  )
end
```

**优化原因**：
- ClickHouse的 `sessions_v2` 表主键只包含 `start` 列
- 仅使用 `timestamp` 过滤会导致采样因子估计偏差
- 额外添加 `start >= first_datetime - 7 days` 确保主键条件存在
- 同时使用 `start <= last_datetime` 限制扫描范围

---

## 7. 典型查询场景分析

### 7.1 场景1：仪表盘查询目标转化

**请求路径**：新API `/query`

```
前端状态:
DashboardState {
  period: "30d",
  filters: [["is", "event:goal", ["Purchase", "Signup"]]],
  metrics: ["visitors", "conversion_rate"],
  dimensions: ["event:goal"]
}

API请求:
POST /api/stats/example.com/query
Body: {
  "date_range": "30d",
  "filters": [["is", "event:goal", ["Purchase", "Signup"]]],
  "metrics": ["visitors", "conversion_rate"],
  "dimensions": ["event:goal"]
}

后端处理:
1. Dashboard.QueryParser.parse
   - skip_goal_existence_check: true
2. QueryBuilder.build
   - validate_filtered_goals_exist → 跳过!
   - preload_goals_and_revenue → 加载所有已配置目标
3. 假设 "UnknownGoal" 未配置:
   - 不会报错
   - SQL中该目标条件匹配不到数据
   - 返回结果中不包含该目标
```

### 7.2 场景2：公开API查询目标转化

**请求路径**：公开API v1 `/api/v1/stats/breakdown`

```
API请求:
GET /api/v1/stats/example.com/breakdown
  ?period=30d
  &property=visit:source
  &metrics=visitors,conversion_rate
  &filters=%5B%5B%22is%22%2C%22event%3Agoal%22%2C%5B%22Purchase%22%2C%22UnknownGoal%22%5D%5D%5D

后端处理:
1. Query.from → Legacy.QueryBuilder (无验证)
2. ExternalStatsController.validate_filters
   - 加载已配置目标: ["Purchase", "Signup"]
   - 检查过滤器中的目标: ["Purchase", "UnknownGoal"]
   - 发现 "UnknownGoal" 未配置
3. 返回错误:
   {
     "error": "The goal `UnknownGoal` is not configured for this site. 
              Find out how to configure goals here: ..."
   }
   
4. SQL查询**不会执行**
```

### 7.3 场景3：使用 `:contains` 操作符

**请求路径**：任意入口

```
过滤器: [[:contains, "event:goal", ["Purchase"]]]

验证行为:
- QueryBuilder.validate_filtered_goals_exist:
  - 只检查 [:is, "event:goal", clauses]
  - :contains 操作符返回 [] → 不检查
- ExternalStatsController.validate_filter:
  - 检查所有 "event:goal" 过滤器
  - :contains 也会验证

关键差异:
- 新API /query: :contains 不会验证目标存在性
- 公开API v1: :contains 仍会验证目标存在性
```

---

## 8. 关键纠正与总结

### 8.1 之前报告的主要错误

| 错误说法 | 正确事实 |
|---------|---------|
| "QueryBuilder.validate_filtered_goals_exist 会验证目标存在性" | 这个验证**默认启用**，但 `Dashboard.QueryParser` 会设置 `skip_goal_existence_check: true` 来**跳过**它 |
| 三个查询入口使用相同的验证逻辑 | 三个入口**完全不同**：新API跳过验证、旧API无此验证、公开API强制验证 |
| `:is` 和 `:contains` 都验证 | QueryBuilder只验证`:is`，但公开API验证所有`event:goal`过滤器 |
| 预加载目标用于验证 | 预加载目标**主要用于构建查询条件**，验证只是副产品（且可跳过） |

### 8.2 查询入口决策树

```
收到查询请求
    │
    ▼
是 POST /api/stats/:domain/query ?
    │
    ├── 是 → 新API入口
    │         │
    │         ▼
    │    Dashboard.QueryParser
    │         │
    │         ├── skip_goal_existence_check: true
    │         │
    │         ▼
    │    QueryBuilder.build
    │         │
    │         ├── 14项验证执行
    │         └── 目标验证被跳过
    │         │
    │         ▼
    │    未配置目标 → SQL层面匹配不到数据（不报错）
    │
    └── 否
         │
         ▼
    是 GET /api/v1/stats/* ?
         │
         ├── 是 → 公开API v1入口
         │         │
         │         ▼
         │    Legacy.QueryBuilder (无验证)
         │         │
         │         ▼
         │    ExternalStatsController.validate_filters
         │         │
         │         └── 强制验证目标存在性
         │         │
         │         ▼
         │    未配置目标 → 立即返回400错误（不执行SQL）
         │
         └── 否 → 内部旧API入口
                   │
                   ▼
              Legacy.QueryBuilder
                   │
                   └── 直接构建Query，无目标验证
                   │
                   ▼
              未配置目标 → SQL层面匹配不到数据（不报错）
```

### 8.3 设计意图分析

**为什么新API跳过目标验证？**

1. **用户体验优先**：仪表盘用户可能正在探索数据，使用`:contains`或模糊匹配时，即使目标不存在也应返回空结果而非错误
2. **向后兼容**：旧版API端点（如`top_stats`）的行为是"缺失的目标匹配空"，新API保持一致
3. **灵活性**：允许用户查询可能尚未配置但预期存在的目标

**为什么公开API强制验证？**

1. **API契约严格**：外部API应该有明确的契约，无效参数应返回错误
2. **调试友好**：集成方可以立即发现配置错误，而不是困惑于空结果
3. **文档明确**：公开API文档明确说明了目标需要先配置

### 8.4 性能优化总结

| 优化策略 | 位置 | 效果 |
|---------|------|------|
| 派生名称过滤 | `derived_name_filter/1` | 减少扫描的事件数量，排除无关事件类型 |
| 目标预加载 | `preload_needed_goals/3` | 避免运行时N+1数据库查询 |
| ARRAY JOIN分组 | `event_goal_join/1` | 利用ClickHouse特性，一次JOIN完成所有目标分组 |
| 会话表主键条件 | `filter_time_range(:sessions, query)` | 确保采样因子估计准确，避免过度采样 |
| 按需表连接 | `join_sessions_if_needed/2` | 仅在需要会话维度时才JOIN，避免不必要开销 |
| 自定义属性数组操作 | `filter_custom_prop/4` | 利用ClickHouse Array类型高效处理键值对 |

---

**报告生成时间**：2026-05-03  
**分析范围**：前端URL参数编码 → 后端参数解析 → 三个查询入口 → 验证逻辑 → SQL构建 → 性能优化

**关键文件参考**：
- `assets/js/dashboard/util/url-search-params.ts` - 前端URL参数编码
- `lib/plausible/stats/dashboard/query_parser.ex` - 仪表盘API参数解析
- `lib/plausible/stats/legacy/legacy_query_builder.ex` - 旧版查询构建器
- `lib/plausible/stats/query_builder.ex` - 查询构建与验证
- `lib/plausible_web/controllers/api/external_stats_controller.ex` - 公开API控制器（强制验证）
