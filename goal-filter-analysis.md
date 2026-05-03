# 目标转化与过滤查询分析报告

## 1. 四条查询入口的完整对比

### 1.1 入口总览

系统存在**四条独立的查询入口**，每条入口的参数解析路径、验证逻辑和目标校验行为都不同：

| 入口名称 | 路由端点 | 控制器 | 解析器 | skip_goal_existence_check | 目标存在性校验 |
|---------|---------|--------|--------|---------------------------|----------------|
| **1. 内部新API (Dashboard Query API)** | `POST /api/stats/:domain/query` | `StatsController.query/2` | `Dashboard.QueryParser` | `true` (显式设置) | **跳过** |
| **2. 内部旧API (Legacy Dashboard API)** | `GET /api/stats/:domain/sources`, `/channels` 等 | `StatsController.sources/2` 等 | `Legacy.QueryBuilder` | 不相关 | **无此验证** |
| **3. 公开API v1 (External Stats API v1)** | `GET /api/v1/stats/aggregate`, `/breakdown`, `/timeseries` | `ExternalStatsController.aggregate/2` 等 | `Legacy.QueryBuilder` | 不相关 | **控制器层面强制验证** |
| **4. 公开API v2 (External Query API)** | `POST /api/v2/query` | `ExternalQueryApiController.query/2` | `ApiQueryParser` | `false` (默认值) | **QueryBuilder层面验证** |

### 1.2 入口1：内部新API (Dashboard Query API)

**路由位置**：`router.ex:297`
```elixir
post "/:domain/query", StatsController, :query
```

**控制器位置**：`lib/plausible_web/controllers/api/stats_controller.ex:40-57`
```elixir
def query(conn, params) do
  site = conn.assigns.site
  now = conn.private[:now]

  with {:ok, %ParsedQueryParams{} = params} <- Dashboard.QueryParser.parse(params, now: now),
       {:ok, %Query{} = query} <- QueryBuilder.build(site, params, debug_metadata(conn)) do
    json(conn, Plausible.Stats.query(site, query))
  else
    {:error, %QueryError{message: message}} -> bad_request(conn, message)
  end
end
```

**解析器位置**：`lib/plausible/stats/dashboard/query_parser.ex:17-36`
```elixir
def parse(params, opts \\ []) do
  with {:ok, input_date_range} <- parse_input_date_range(params),
       {:ok, relative_date} <- parse_relative_date(params),
       {:ok, dimensions} <- ApiQueryParser.parse_dimensions(params["dimensions"]),
       {:ok, filters} <- ApiQueryParser.parse_filters(params["filters"]),
       {:ok, metrics} <- parse_metrics(params),
       {:ok, include} <- parse_include(params) do
    {:ok,
     ParsedQueryParams.new!(%{
       input_date_range: input_date_range,
       relative_date: relative_date,
       dimensions: dimensions,
       filters: filters,
       metrics: metrics,
       include: include,
       skip_goal_existence_check: true,  # 关键：显式设置为 true
       now: Keyword.get(opts, :now)
     })}
  end
end
```

**关键特征**：
- 使用 `Dashboard.QueryParser`，显式设置 `skip_goal_existence_check: true`
- 经过 `QueryBuilder.build/3` 的14项验证流程
- `validate_filtered_goals_exist` 函数直接返回 `:ok`，跳过目标验证

### 1.3 入口2：内部旧API (Legacy Dashboard API)

**路由位置**：`router.ex:299-324`
```elixir
get "/:domain/current-visitors", StatsController, :current_visitors
get "/:domain/sources", StatsController, :sources
get "/:domain/channels", StatsController, :channels
get "/:domain/pages", StatsController, :pages
# ... 共约20个端点
```

**控制器位置**：`lib/plausible_web/controllers/api/stats_controller.ex:59-97` (以`sources/2`为例)
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

**解析器位置**：`lib/plausible/stats/query.ex:71-82`
```elixir
@doc """
Builds query from old-style stats APIv1 params. New code should use `Query.parse_and_build`
or `QueryBuilder.build` with already parsed params.
"""
def from(site, params, opts \\ []) do
  Legacy.QueryBuilder.from(
    site,
    params,
    Keyword.get(opts, :debug_metadata, %{}),
    Keyword.get(opts, :now)
  )
end
```

**解析器位置**：`lib/plausible/stats/legacy/legacy_query_builder.ex:14-44`
```elixir
def from(site, params, debug_metadata, now \\ nil) do
  now = now || DateTime.utc_now(:second)

  query =
    Query
    |> struct!(
      now: now,
      debug_metadata: debug_metadata,
      site_id: site.id,
      site_native_stats_start_at: site.native_stats_start_at
    )
    |> put_input_date_range(site, params)
    |> put_timezone(site)
    |> put_dimensions(params)
    |> put_interval(params)
    |> put_parsed_filters(params)
    |> resolve_segments(site)
    |> preload_goals_and_revenue(site)
    |> put_consolidated_site_ids(site)
    |> put_order_by(params)
    |> put_include(params)
    |> QueryBuilder.put_comparison_utc_time_range()
    |> Query.put_imported_opts(site)
    |> QueryBuilder.set_time_on_page_data(site)

  # ... 采样设置
  query
end
```

**关键特征**：
- 使用 `Legacy.QueryBuilder` 直接构建 `%Query{}` 结构
- **完全绕过** `QueryBuilder.build/3` 的14项验证流程
- 没有任何目标存在性验证
- 预加载目标仅用于构建SQL查询条件，不用于验证

### 1.4 入口3：公开API v1 (External Stats API v1)

**路由位置**：`router.ex:337-345`
```elixir
scope "/api/v1/stats", PlausibleWeb.Api,
  assigns: %{api_scope: "stats:read:*", api_context: :site} do
  pipe_through [:public_api, PlausibleWeb.Plugs.AuthorizePublicAPI]

  get "/realtime/visitors", ExternalStatsController, :realtime_visitors
  get "/aggregate", ExternalStatsController, :aggregate
  get "/breakdown", ExternalStatsController, :breakdown
  get "/timeseries", ExternalStatsController, :timeseries
end
```

**控制器位置**：`lib/plausible_web/controllers/api/external_stats_controller.ex:33-55` (以`breakdown/2`为例)
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
    # ... 执行查询
  end
end
```

**验证函数位置**：`lib/plausible_web/controllers/api/external_stats_controller.ex:330-359`
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

    {:error, msg}  # 返回错误
  else
    :ok
  end
end
```

**关键特征**：
- 使用 `Legacy.QueryBuilder` 构建 `%Query{}`（同入口2）
- **控制器层面**有独立的 `validate_filters/2` 函数
- 验证所有 `event:goal` 过滤器，**不区分** `:is` 和 `:contains` 操作符
- 发现未配置目标立即返回400错误，**SQL查询不会执行**

### 1.5 入口4：公开API v2 (External Query API)

**路由位置**：`router.ex:347-358`
```elixir
scope "/api/v2", PlausibleWeb.Api,
  private: %{allow_consolidated_views: true},
  assigns: %{api_scope: "stats:read:*", api_context: :site} do
  pipe_through [:public_api, PlausibleWeb.Plugs.AuthorizePublicAPI]

  post "/query", ExternalQueryApiController, :query
end
```

**控制器位置**：`lib/plausible_web/controllers/api/external_query_api_controller.ex:9-25`
```elixir
def query(conn, params) do
  site = Repo.preload(conn.assigns.site, :owners)

  case Query.parse_and_build(site, params,
         debug_metadata: debug_metadata(conn),
         now: conn.private[:now]
       ) do
    {:ok, query} ->
      results = Plausible.Stats.query(site, query)
      json(conn, results)

    {:error, %QueryError{message: message}} ->
      conn
      |> put_status(400)
      |> json(%{error: message})
  end
end
```

**解析器位置**：`lib/plausible/stats/query.ex:50-59`
```elixir
def parse_and_build(
      %Plausible.Site{domain: domain} = site,
      %{"site_id" => domain} = params,
      opts \\ []
    ) do
  with {:ok, %ParsedQueryParams{} = parsed_query_params} <-
         ApiQueryParser.parse(params, opts) do  # 注意：使用 ApiQueryParser，不是 Dashboard.QueryParser
    QueryBuilder.build(site, parsed_query_params, Keyword.get(opts, :debug_metadata, %{}))
  end
end
```

**解析器位置**：`lib/plausible/stats/api_query_parser.ex:25-46`
```elixir
def parse(params, opts \\ []) when is_map(params) do
  with :ok <- JSONSchema.validate(params),
       {:ok, input_date_range} <- parse_input_date_range(params["date_range"]),
       {:ok, metrics} <- parse_metrics(Map.fetch!(params, "metrics")),
       {:ok, filters} <- parse_filters(params["filters"]),
       {:ok, dimensions} <- parse_dimensions(params["dimensions"]),
       {:ok, order_by} <- parse_order_by(params["order_by"]),
       {:ok, pagination} <- parse_pagination(params["pagination"]),
       {:ok, include} <- parse_include(params["include"]) do
    {:ok,
     Plausible.Stats.ParsedQueryParams.new!(%{
       input_date_range: input_date_range,
       metrics: metrics,
       filters: filters,
       dimensions: dimensions,
       order_by: order_by,
       pagination: pagination,
       include: include,
       now: Keyword.get(opts, :now)
       # 注意：没有设置 skip_goal_existence_check，使用默认值 false
     })}
  end
end
```

**数据结构默认值位置**：`lib/plausible/stats/parsed_query_params.ex:4-22`
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
          skip_goal_existence_check: false  # 默认值是 false！
```

**关键特征**：
- 使用 `ApiQueryParser`（不是 `Dashboard.QueryParser`）
- **没有设置** `skip_goal_existence_check`，使用默认值 `false`
- 经过 `QueryBuilder.build/3` 的完整14项验证流程
- `validate_filtered_goals_exist` 函数**会执行验证**
- 仅验证 `:is` 操作符的 `event:goal` 过滤器（不验证 `:contains`）

---

## 2. 目标存在性校验的完整真相

### 2.1 验证函数分析

**验证函数位置**：`lib/plausible/stats/query_builder.ex:409-444`
```elixir
defp validate_filtered_goals_exist(_query, %ParsedQueryParams{skip_goal_existence_check: true}),
  do: :ok  # 短路返回，直接跳过

defp validate_filtered_goals_exist(query, %ParsedQueryParams{}) do
  # 仅检查 :is 操作符的 event:goal 过滤器
  goal_filter_clauses =
    query.filters
    |> Filters.all_leaf_filters()
    |> Enum.flat_map(fn
      [:is, "event:goal", clauses] -> clauses  # 只检查 :is
      _ -> []  # :contains, :is_not 等其他操作符不检查
    end)

  if length(goal_filter_clauses) > 0 do
    configured_goal_names =
      query.preloaded_goals.all
      |> Enum.map(&Plausible.Goal.display_name/1)

    validate_list(goal_filter_clauses, &validate_goal_filter(&1, configured_goal_names))
  else
    :ok
  end
end

defp validate_goal_filter(clause, configured_goal_names) do
  if Enum.member?(configured_goal_names, clause) do
    :ok
  else
    {:error,
     %QueryError{
       code: :invalid_filters,
       message: "Invalid filters. The goal `#{clause}` is not configured for this site."
     }}
  end
end
```

### 2.2 四条入口的校验行为对比表

| 入口 | 使用的解析器 | skip_goal_existence_check | 校验位置 | 校验范围 | 校验结果 |
|------|-------------|---------------------------|---------|---------|---------|
| **1. 内部新API** | `Dashboard.QueryParser` | `true` (显式设置) | QueryBuilder | 不执行 | **跳过** |
| **2. 内部旧API** | `Legacy.QueryBuilder` | 不相关 | 无 | 无 | **无此验证** |
| **3. 公开API v1** | `Legacy.QueryBuilder` | 不相关 | 控制器层面 | 所有 `event:goal` 过滤器 (含 `:contains`) | **强制校验** |
| **4. 公开API v2** | `ApiQueryParser` | `false` (默认值) | QueryBuilder | 仅 `:is` 操作符的 `event:goal` | **校验执行** |

### 2.3 之前报告的错误纠正

**错误说法1**：
> "唯一会在查询执行前拒绝未配置目标的入口"

**纠正**：
有**两条入口**会执行目标校验：
- **入口3 (公开API v1)**：控制器层面强制校验所有 `event:goal` 过滤器
- **入口4 (公开API v2)**：QueryBuilder层面校验 `:is` 操作符的 `event:goal` 过滤器

**错误说法2**：
> "QueryBuilder.validate_filtered_goals_exist 会验证目标存在性"

**纠正**：
这个函数**默认启用**，但在实际使用中：
- 入口1：`Dashboard.QueryParser` 设置 `skip_goal_existence_check: true`，函数直接返回 `:ok`
- 入口2：不经过 `QueryBuilder.build`，函数不被调用
- 入口3：控制器层面有自己的 `validate_filters/2`，不使用 QueryBuilder 的这个函数
- 入口4：函数正常执行

**错误说法3**：
> "混淆不同查询入口的行为"

**纠正**：
四条入口的行为**完全不同**：
- 入口1和2：**不校验**（一个跳过，一个无此验证）
- 入口3和4：**校验**（但校验位置、范围、方式都不同）

---

## 3. 前端URL状态到后端查询构建的完整链路

### 3.1 链路总览

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
│          ┌──────────────────────┼──────────────────────┐                          │
│          │                      │                      │                          │
│          ▼                      ▼                      ▼                          │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐            │
│  │  内部新API      │  │  内部旧API      │  │  公开API v1 & v2   │            │
│  │  POST /query    │  │  GET /sources等 │  │  GET/POST          │            │
│  └────────┬────────┘  └────────┬────────┘  └──────────┬──────────┘            │
│           │                      │                      │                       │
│           ▼                      ▼                      ▼                       │
│  ┌───────────────────────┐  ┌───────────────────────┐  ┌───────────────────┐  │
│  │ Dashboard.QueryParser │  │ Legacy.QueryBuilder   │  │ 两条不同路径:     │  │
│  │ skip_goal: true       │  │ 绕过QueryBuilder      │  │ v1: 控制器校验    │  │
│  └────────┬──────────────┘  └────────┬──────────────┘  │ v2: ApiQueryParser│  │
│           │                      │                     └─────────┬─────────┘  │
│           ▼                      ▼                               │              │
│  ┌───────────────────────────────────────────────────────────────┴──────────┐ │
│  │                      QueryBuilder.build (14项验证)                           │ │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐│ │
│  │  │ 1. validate_order_by                           ││ │
│  │  │ 2. validate_custom_props_access               ││ │
│  │  │ 3. validate_case_sensitive_filter_modifier     ││ │
│  │  │ 4. validate_toplevel_only_filter_dimension     ││ │
│  │  │ 5. validate_time_dimension_granularity         ││ │
│  │  │ 6. validate_special_metrics_filters            ││ │
│  │  │ 7. validate_behavioral_filters                 ││ │
│  │  │ 8. validate_filtered_goals_exist              ││ │
│  │  │    → 入口1: skip_goal=true → 跳过             ││ │
│  │  │    → 入口4: skip_goal=false → 执行             ││ │
│  │  │ 9. validate_revenue_metrics_access             ││ │
│  │  │ 10. validate_metrics                            ││ │
│  │  │ 11. validate_include                           ││ │
│  │  └─────────────────────────────────────────────────────────────────────────┘│ │
│  └────────────────────────────────────────────────────────────────────────────────┘ │
│                                              │                                       │
│                                              ▼                                       │
│                                     ┌─────────────────┐                              │
│                                     │  %Query{} 结构  │                              │
│                                     └────────┬────────┘                              │
│                                              │                                       │
│                                              ▼                                       │
│                                     ┌─────────────────┐                              │
│                                     │ SQL.QueryBuilder│                              │
│                                     │ - 构建Ecto查询   │                              │
│                                     │ - Goals.add_    │                              │
│                                     │   filter()      │                              │
│                                     └────────┬────────┘                              │
│                                              │                                       │
│                                              ▼                                       │
│                                     ┌─────────────────┐                              │
│                                     │  ClickHouse     │                              │
│                                     │  - 执行SQL查询   │                              │
│                                     │  - 未配置目标    │                              │
│                                     │    匹配不到数据  │                              │
│                                     └─────────────────┘                              │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 前端URL参数编码详解

**位置**：`assets/js/dashboard/util/url-search-params.ts`

#### 自定义序列化格式（可读性优先）

```typescript
// URL地址栏使用自定义逗号分隔格式
// 示例: ?f=is,event:goal,Purchase&period=30d

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
    queryObj.filters = serializeApiFilters(dashboardState.filters)  // 转换为JSON字符串
  }
  // ...
  return queryObj
}
```

**重要差异**：
- **URL地址栏**：自定义逗号分隔格式（可读性优先）
- **API请求**：JSON格式（结构化优先）

### 3.3 四条入口的参数解析路径

#### 路径1：内部新API → Dashboard.QueryParser

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
   - 调用 ApiQueryParser.parse 解析内部字段
   - 设置 skip_goal_existence_check: true
3. QueryBuilder.build(site, parsed_params)
   - 所有14项验证执行...
   - validate_filtered_goals_exist → 因为 skip_goal_existence_check: true，直接返回 :ok
4. 返回 %Query{}

目标校验: 跳过
```

#### 路径2：内部旧API → Legacy.QueryBuilder

```
前端发送:
GET /api/stats/example.com/sources
  ?period=30d
  &property=visit:source
  &filters=%5B%5B%22is%22%2C%22event%3Agoal%22%2C%5B%22Purchase%22%5D%5D%5D

后端处理:
1. Phoenix解析URL query params → conn.query_params
   - "filters" = "[[\"is\",\"event:goal\",[\"Purchase\"]]]" (JSON字符串)
2. Query.from(site, params) → Legacy.QueryBuilder.from(...)
   - put_parsed_filters: Filters.parse(params["filters"]) → 解析为 [[:is, "event:goal", ["Purchase"]]]
   - 直接构建 %Query{}，**完全绕过** QueryBuilder.build
3. 返回 %Query{}

目标校验: 无此验证（不经过 QueryBuilder.build）
```

#### 路径3：公开API v1 → 控制器层面验证

```
前端发送:
GET /api/v1/stats/example.com/breakdown
  ?period=30d
  &property=visit:source
  &metrics=visitors,conversion_rate
  &filters=%5B%5B%22is%22%2C%22event%3Agoal%22%2C%5B%22Purchase%22%2C%22UnknownGoal%22%5D%5D%5D

后端处理:
1. Phoenix解析URL query params → conn.query_params
2. Query.from(site, params) → Legacy.QueryBuilder.from(...)
   - 同路径2，构建 %Query{}，无验证
3. ExternalStatsController.validate_filters(site, query.filters)
   - 遍历所有 filters
   - 对每个 "event:goal" 过滤器调用 validate_filter
   - 加载已配置目标: ["Purchase", "Signup"]
   - 检查过滤器中的目标: ["Purchase", "UnknownGoal"]
   - 发现 "UnknownGoal" 未配置
4. 返回错误:
   {
     "error": "The goal `UnknownGoal` is not configured for this site. 
              Find out how to configure goals here: ..."
   }

目标校验: 强制执行（控制器层面）
```

#### 路径4：公开API v2 → ApiQueryParser + QueryBuilder

```
前端发送:
POST /api/v2/query
Content-Type: application/json
Body: {
  "site_id": "example.com",
  "date_range": "30d",
  "metrics": ["visitors", "conversion_rate"],
  "filters": [["is", "event:goal", ["Purchase", "UnknownGoal"]]],
  "dimensions": ["event:goal"],
  "order_by": null,
  "pagination": null,
  "include": null
}

后端处理:
1. Phoenix解析JSON → conn.body_params (Map)
2. Query.parse_and_build(site, params)
   → ApiQueryParser.parse(params, opts)
      - 不设置 skip_goal_existence_check，默认值 false
   → QueryBuilder.build(site, parsed_params, ...)
      - validate_filtered_goals_exist:
        - 提取 [:is, "event:goal", ...] 的 clauses: ["Purchase", "UnknownGoal"]
        - 检查 preloaded_goals.all: ["Purchase", "Signup"]
        - 发现 "UnknownGoal" 未配置
3. 返回错误:
   {
     "error": "Invalid filters. The goal `UnknownGoal` is not configured for this site."
   }

目标校验: 执行（QueryBuilder层面，仅校验 :is 操作符）
```

### 3.4 入口与前端链路的对应关系

| 前端API调用 | HTTP方法 | 后端入口 | 目标校验 |
|------------|---------|---------|---------|
| `stats(site, statsQuery)` | POST | 内部新API | **跳过** |
| `get(url, dashboardState)` | GET | 内部旧API | **无此验证** |
| 公开API v1调用 | GET | 公开API v1 | **强制校验** |
| 公开API v2调用 | POST | 公开API v2 | **校验执行** |

**关键观察**：
- 前端仪表盘的主要数据流使用**内部新API**和**内部旧API**，都**不校验**目标存在性
- 这是设计意图：允许用户探索数据，未配置的目标匹配空结果而非报错
- 公开API（v1和v2）用于外部集成，**强制校验**目标存在性

---

## 4. 设计意图与边界条件

### 4.1 为什么内部API不校验目标？

**用户体验优先**：
- 仪表盘用户可能正在探索数据，使用 `:contains` 或模糊匹配
- 即使目标不存在，也应返回空结果而非错误
- 例如：用户想查看所有包含"Purchase"的目标，但具体目标名可能记错

**向后兼容**：
- 旧版API端点（如 `top_stats`）的行为是"缺失的目标匹配空"
- 新API保持一致的行为，避免破坏性变更

**灵活性**：
- 允许用户查询可能尚未配置但预期存在的目标
- 目标配置可能还在设置中，但用户想提前验证查询逻辑

### 4.2 为什么公开API强制校验？

**API契约严格**：
- 外部API应该有明确的契约，无效参数应返回错误
- 集成方依赖API的可预测行为

**调试友好**：
- 集成方可以立即发现配置错误，而不是困惑于空结果
- 例如：第三方BI工具集成时，错误提示比空结果更有意义

**文档明确**：
- 公开API文档明确说明了目标需要先配置
- 校验行为符合文档预期

### 4.3 为什么v1和v2的校验方式不同？

**v1（控制器层面校验）**：
- 使用旧的 `Legacy.QueryBuilder`，绕过 `QueryBuilder.build`
- 需要在控制器层面添加独立的校验逻辑
- 校验**所有** `event:goal` 过滤器，不区分操作符

**v2（QueryBuilder层面校验）**：
- 使用新的 `ApiQueryParser` + `QueryBuilder.build` 路径
- `skip_goal_existence_check` 默认为 `false`
- 仅校验 `:is` 操作符的 `event:goal` 过滤器
- 这是更细粒度的控制：`:contains` 操作符允许模糊匹配，可能故意匹配不存在的目标

### 4.4 边界条件总结

| 边界条件 | 内部新API | 内部旧API | 公开API v1 | 公开API v2 |
|---------|----------|----------|-----------|-----------|
| 未配置目标使用 `:is` | 空结果 | 空结果 | 400错误 | 400错误 |
| 未配置目标使用 `:contains` | 空结果 | 空结果 | 400错误 | 空结果 |
| 目标过滤器嵌套在 `:and`/`:or` 内 | 空结果 | 空结果 | 400错误 | 400错误（且触发"仅顶层维度"错误） |
| 使用 `:is_not` 操作符 | 空结果（匹配所有其他） | 空结果 | 400错误 | 空结果 |

### 4.5 关键验证规则详解

#### 规则1：仅顶层可用的维度

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
- `event:goal` 或 `event:hostname` 出现在嵌套过滤器中（`min_depth: 1`）
- 例如：`[:and, [[:is, "visit:country", ["US"]], [:is, "event:goal", ["Purchase"]]]]`

**影响范围**：
- 内部新API：触发（因为经过 `QueryBuilder.build`）
- 内部旧API：不触发（因为绕过 `QueryBuilder.build`）
- 公开API v1：不触发（控制器层面的校验不检查这条）
- 公开API v2：触发（经过 `QueryBuilder.build`）

#### 规则2：行为过滤器限制

**位置**：`lib/plausible/stats/query_builder.ex:370-407`

**限制**：
1. **不能嵌套**：`[:has_done, [:has_done, ...]]` → 错误
2. **只能用于事件维度**：`[:has_done, [:is, "visit:country", ...]]` → 错误

**影响范围**：
- 内部新API：触发
- 内部旧API：不触发
- 公开API v1：不触发
- 公开API v2：触发

#### 规则3：特殊指标与深度过滤器冲突

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

**影响范围**：
- 内部新API：触发
- 内部旧API：不触发
- 公开API v1：不触发
- 公开API v2：触发

---

## 5. 性能优化策略

### 5.1 派生名称过滤（Derived Name Filter）

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

**优化效果**：
- **情况2**：通过预加载的目标知道相关的事件名称，添加 `name IN (...)` 条件
- **情况3**：`engagement` 事件仅用于滚动深度和页面停留时间，默认排除以减少扫描数据量

### 5.2 目标预加载

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

**预加载的用途**：
1. **构建SQL查询条件** (`Goals.add_filter/3`)
2. **派生名称过滤优化** (`SQL.WhereBuilder.derived_name_filter/1`)
3. **ARRAY JOIN分组数据** (`Goals.goal_join_data/1`)
4. **验证**（仅在 `skip_goal_existence_check: false` 时）

### 5.3 ARRAY JOIN 分组优化

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

**ClickHouse优化**：
- 使用 `hints: "ARRAY"` 提示ClickHouse优化器
- 一次JOIN完成所有目标的分组，避免多次扫描

---

## 6. 完整总结与决策树

### 6.1 四条入口决策树

```
收到查询请求
    │
    ▼
路由判断:
    │
    ├── POST /api/stats/:domain/query → 内部新API
    │         │
    │         ▼
    │    Dashboard.QueryParser
    │         │
    │         ├── skip_goal_existence_check: true
    │         │
    │         ▼
    │    QueryBuilder.build
    │         │
    │         └── validate_filtered_goals_exist → 跳过
    │
    ├── GET /api/stats/:domain/* (非 /query) → 内部旧API
    │         │
    │         ▼
    │    Legacy.QueryBuilder
    │         │
    │         └── 绕过 QueryBuilder.build
    │         │
    │         └── 无目标验证
    │
    ├── GET /api/v1/stats/* → 公开API v1
    │         │
    │         ▼
    │    Legacy.QueryBuilder
    │         │
    │         └── 绕过 QueryBuilder.build
    │         │
    │         ▼
    │    控制器层面 validate_filters
    │         │
    │         └── 强制验证所有 event:goal 过滤器
    │
    └── POST /api/v2/query → 公开API v2
              │
              ▼
         ApiQueryParser
              │
              ├── skip_goal_existence_check: false (默认)
              │
              ▼
         QueryBuilder.build
              │
              └── validate_filtered_goals_exist
                   │
                   └── 验证 :is 操作符的 event:goal 过滤器
```

### 6.2 关键纠正汇总

| 之前的错误说法 | 纠正后的正确事实 |
|---------------|-----------------|
| "唯一会在查询执行前拒绝未配置目标的入口" | **两条入口**会校验：公开API v1（控制器层面）和公开API v2（QueryBuilder层面） |
| "QueryBuilder.validate_filtered_goals_exist 会验证目标存在性" | 这个函数**默认启用**，但：<br>• 内部新API：被 `skip_goal_existence_check: true` 跳过<br>• 内部旧API：不经过 `QueryBuilder.build`<br>• 公开API v1：控制器有自己的校验，不使用这个函数<br>• 公开API v2：函数正常执行 |
| "三条查询入口使用相同的验证逻辑" | **四条入口**的行为完全不同：<br>• 内部新API：跳过<br>• 内部旧API：无此验证<br>• 公开API v1：控制器层面强制校验所有<br>• 公开API v2：QueryBuilder层面校验 :is 操作符 |
| "预加载目标用于验证" | 预加载目标**主要用于**：<br>• 构建SQL查询条件<br>• 派生名称过滤优化<br>• ARRAY JOIN分组<br>• **验证只是副产品**（且可跳过） |

### 6.3 设计意图总结

| 设计决策 | 原因 |
|---------|------|
| 内部API不校验目标 | 用户体验优先、向后兼容、灵活性 |
| 公开API强制校验 | API契约严格、调试友好、文档明确 |
| v1和v2校验方式不同 | v1继承旧架构，v2使用新的QueryBuilder路径 |
| v2只校验 `:is` 操作符 | `:contains` 允许模糊匹配，可能故意匹配不存在的目标 |
| `skip_goal_existence_check` 标志 | 允许同一套验证逻辑在不同场景有不同行为 |

---

**报告生成时间**：2026-05-03  
**分析范围**：前端URL参数编码 → 后端四条查询入口 → 目标存在性校验 → 验证规则 → 性能优化

**关键文件参考**：
- `assets/js/dashboard/util/url-search-params.ts` - 前端URL参数编码
- `lib/plausible/stats/dashboard/query_parser.ex` - 内部新API解析器（设置 `skip_goal_existence_check: true`）
- `lib/plausible/stats/legacy/legacy_query_builder.ex` - 内部旧API构建器（绕过验证）
- `lib/plausible/stats/api_query_parser.ex` - 公开API v2解析器（使用默认 `skip_goal_existence_check: false`）
- `lib/plausible/stats/query_builder.ex` - 查询构建与验证（包含 `validate_filtered_goals_exist`）
- `lib/plausible_web/controllers/api/external_stats_controller.ex` - 公开API v1控制器（独立的 `validate_filters`）
- `lib/plausible_web/controllers/api/external_query_api_controller.ex` - 公开API v2控制器
