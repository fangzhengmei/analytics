# 目标转化与过滤查询分析报告

## 1. 整体架构与数据流

### 1.1 数据流概览

从页面状态到分析查询的完整数据流分为以下几个阶段：

```
页面状态（LiveView/API参数）
    ↓
API Query Parser 解析
    ↓
ParsedQueryParams 结构
    ↓
Query Builder 构建与验证
    ↓
Query 结构
    ↓
SQL Query Builder 构建
    ↓
Ecto 查询 → 数据库执行
```

### 1.2 入口点分析

系统提供了两种主要的查询入口方式：

**方式一：新API查询（推荐）**
```elixir
# lib/plausible/stats/query.ex:50-59
def parse_and_build(site, params, opts \\ []) do
  with {:ok, %ParsedQueryParams{} = parsed_query_params} <-
         ApiQueryParser.parse(params, opts) do
    QueryBuilder.build(site, parsed_query_params, Keyword.get(opts, :debug_metadata, %{}))
  end
end
```

**方式二：旧API兼容查询**
```elixir
# lib/plausible/stats/query.ex:74-82
def from(site, params, opts \\ []) do
  Legacy.QueryBuilder.from(
    site,
    params,
    Keyword.get(opts, :debug_metadata, %{}),
    Keyword.get(opts, :now)
  )
end
```

**控制器中的实际使用**：
```elixir
# lib/plausible_web/controllers/api/stats_controller.ex:40-57
def query(conn, params) do
  site = conn.assigns.site
  now = conn.private[:now]

  with {:ok, %ParsedQueryParams{} = params} <- Dashboard.QueryParser.parse(params, now: now),
       {:ok, %Query{} = query} <- QueryBuilder.build(site, params, debug_metadata(conn)) do
    # ...
    json(conn, Plausible.Stats.query(site, query))
  end
end
```

## 2. 模块协作关系

### 2.1 核心模块职责划分

| 模块 | 文件位置 | 主要职责 |
|------|---------|---------|
| `ApiQueryParser` | `lib/plausible/stats/api_query_parser.ex` | 解析原始API参数，转换为结构化数据 |
| `ParsedQueryParams` | `lib/plausible/stats/parsed_query_params.ex` | 解析后参数的数据结构，提供辅助方法 |
| `QueryBuilder` | `lib/plausible/stats/query_builder.ex` | 构建Query结构，执行所有验证逻辑 |
| `Query` | `lib/plausible/stats/query.ex` | 最终查询结构，提供查询操作方法 |
| `Filters` | `lib/plausible/stats/filters/filters.ex` | 过滤器解析、遍历、转换工具函数 |
| `Goals` | `lib/plausible/stats/goals.ex` | 目标转化相关的查询逻辑 |
| `SQL.QueryBuilder` | `lib/plausible/stats/sql/query_builder.ex` | 构建Ecto SQL查询 |
| `SQL.WhereBuilder` | `lib/plausible/stats/sql/where_builder.ex` | 构建WHERE条件子句 |

### 2.2 模块协作流程

#### 阶段1：参数解析

`ApiQueryParser.parse` 负责将原始参数映射为内部结构：

```elixir
# lib/plausible/stats/api_query_parser.ex:25-46
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
     })}
  end
end
```

**过滤器解析细节**：
```elixir
# lib/plausible/stats/api_query_parser.ex:69-75
defp parse_filter(filter) do
  with {:ok, operator} <- parse_operator(filter),
       {:ok, second} <- parse_filter_second(operator, filter),
       {:ok, rest} <- parse_filter_rest(operator, filter) do
    {:ok, [operator, second | rest]}
  end
end
```

支持的操作符包括：
- 基础过滤：`:is`, `:is_not`, `:contains`, `:contains_not`
- 模式匹配：`:matches`, `:matches_not`, `:matches_wildcard`, `:matches_wildcard_not`
- 逻辑组合：`:and`, `:or`, `:not`
- 行为过滤：`:has_done`, `:has_not_done`

#### 阶段2：查询构建与验证

`QueryBuilder.build` 是核心的协调器，负责：

1. 解析段落在过滤器中的引用
2. 构建日期时间范围
3. 预加载目标和收入数据
4. 执行一系列验证
5. 应用额外的查询设置

```elixir
# lib/plausible/stats/query_builder.ex:29-60
def build(site, %ParsedQueryParams{} = parsed_query_params, debug_metadata) do
  with {:ok, parsed_query_params} <- resolve_segments_in_filters(parsed_query_params, site),
       query = do_build(parsed_query_params, site, debug_metadata),
       :ok <- validate_order_by(query),
       :ok <- validate_custom_props_access(site, query),
       :ok <- validate_case_sensitive_filter_modifier(query),
       :ok <- validate_toplevel_only_filter_dimension(query),
       :ok <- validate_time_dimension_granularity(query),
       :ok <- validate_special_metrics_filters(query),
       :ok <- validate_behavioral_filters(query),
       :ok <- validate_filtered_goals_exist(query, parsed_query_params),
       :ok <- validate_revenue_metrics_access(site, query),
       :ok <- validate_metrics(query),
       :ok <- validate_include(query) do
    query =
      query
      |> set_time_on_page_data(site)
      |> put_comparison_utc_time_range()
      |> Query.put_imported_opts(site)

    # ... 采样设置
    {:ok, query}
  end
end
```

**日期时间范围构建**：
```elixir
# lib/plausible/stats/query_builder.ex:88-150
defp build_datetime_range(input_date_range, _site, _relative_date, now)
     when input_date_range in [:realtime, :realtime_30m] do
  duration_minutes =
    case input_date_range do
      :realtime -> 5
      :realtime_30m -> 30
    end

  first_datetime = DateTime.shift(now, minute: -duration_minutes)
  last_datetime = DateTime.shift(now, second: 5)

  DateTimeRange.new!(first_datetime, last_datetime)
end
```

#### 阶段3：目标预加载

```elixir
# lib/plausible/stats/query_builder.ex:222-234
def preload_goals_and_revenue(site, metrics, filters, dimensions) do
  preloaded_goals =
    Plausible.Stats.Goals.preload_needed_goals(site, dimensions, filters)

  {revenue_warning, revenue_currencies} =
    preload_revenue(site, preloaded_goals, metrics, dimensions)

  {
    preloaded_goals,
    revenue_warning,
    revenue_currencies
  }
end
```

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
    %{
      all: [],
      matching_toplevel_filters: []
    }
  end
end
```

#### 阶段4：SQL查询构建

```elixir
# lib/plausible/stats/sql/query_builder.ex:17-28
def build(query, site) do
  query
  |> QueryOptimizer.split()
  |> Enum.map(fn {table_type, table_query} ->
    q = build_table_query(table_type, site, table_query)
    {table_type, table_query, q}
  end)
  |> join_query_results(query)
  |> build_order_by(query)
  |> paginate(query.pagination)
  |> select_total_rows(query.include.total_rows)
end
```

**事件表查询构建**：
```elixir
# lib/plausible/stats/sql/query_builder.ex:34-53
defp build_table_query(:events, site, events_query) do
  q =
    from(
      e in "events_v2",
      where: ^SQL.WhereBuilder.build(:events, events_query),
      where: ^SQL.WhereBuilder.derived_name_filter(events_query),
      select: ^select_event_metrics(events_query)
    )

  # ... 采样提示

  q
  |> join_sessions_if_needed(events_query)
  |> build_group_by(:events, events_query)
  |> merge_imported(site, events_query)
  |> SQL.SpecialMetrics.add(site, events_query)
  |> TimeOnPage.merge_legacy_time_on_page(events_query)
end
```

## 3. 边界条件与验证逻辑

### 3.1 目标存在性验证

**位置**：`lib/plausible/stats/query_builder.ex:409-444`

```elixir
defp validate_filtered_goals_exist(_query, %ParsedQueryParams{skip_goal_existence_check: true}),
  do: :ok

defp validate_filtered_goals_exist(query, %ParsedQueryParams{}) do
  goal_filter_clauses =
    query.filters
    |> Filters.all_leaf_filters()
    |> Enum.flat_map(fn
      [:is, "event:goal", clauses] -> clauses
      _ -> []
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

**设计要点**：
- 仅验证 `:is` 操作符的目标过滤器，`:contains` 操作符允许匹配空结果
- 提供 `skip_goal_existence_check` 选项用于兼容旧版API
- 使用预加载的目标列表进行验证，避免额外数据库查询

### 3.2 行为过滤器嵌套限制

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
      behavioral_depth == 0 ->
        {:cont, :ok}

      behavioral_depth > 1 ->
        {:halt,
         {:error,
          %QueryError{
            code: :invalid_filters,
            message: "Invalid filters. Behavioral filters (has_done, has_not_done) cannot be nested."
          }}}

      not String.starts_with?(dimension, "event:") ->
        {:halt,
         {:error,
          %QueryError{
            code: :invalid_filters,
            message: "Invalid filters. Behavioral filters (has_done, has_not_done) can only be used with event dimension filters."
          }}}

      true ->
        {:cont, :ok}
    end
  end)
end
```

**限制规则**：
1. 行为过滤器不能嵌套（深度>1时报错）
2. 行为过滤器只能用于事件维度（以 `event:` 开头）

**过滤器遍历机制**：
```elixir
# lib/plausible/stats/filters/filters.ex:199-217
def traverse(filters, state \\ nil, state_transformer \\ fn state, _ -> state end) do
  filters
  |> Enum.flat_map(&traverse_tree(&1, state, state_transformer))
end

defp traverse_tree(filter, state, state_transformer) do
  case filter do
    [operation, child_filter]
    when operation in [:not, :ignore_in_totals_query, :has_done, :has_not_done] ->
      traverse_tree(child_filter, state_transformer.(state, operation), state_transformer)

    [operation, filters] when operation in [:and, :or] ->
      traverse(filters, state_transformer.(state, operation), state_transformer)

    _ ->
      [{filter, state}]
  end
end
```

### 3.3 仅顶层可用的维度

**位置**：`lib/plausible/stats/query_builder.ex:314-331`

```elixir
@only_toplevel ["event:goal", "event:hostname"]

defp validate_toplevel_only_filter_dimension(query) do
  not_toplevel =
    query.filters
    |> Filters.dimensions_used_in_filters(min_depth: 1, behavioral_filters: :ignore)
    |> Enum.filter(&(&1 in @only_toplevel))

  if Enum.count(not_toplevel) > 0 do
    {:error,
     %QueryError{
       code: :invalid_filters,
       message: "Invalid filters. Dimension `#{List.first(not_toplevel)}` can only be filtered at the top level."
     }}
  else
    :ok
  end
end
```

**原因分析**：
- `event:goal`：目标过滤需要特殊处理，涉及预加载目标列表和构建复杂的SQL条件
- `event:hostname`：主机名过滤可能涉及特殊的路由逻辑

### 3.4 时间维度粒度限制

**位置**：`lib/plausible/stats/query_builder.ex:333-347`

```elixir
@max_hours_for_minute_interval 30

defp validate_time_dimension_granularity(query) do
  if Time.time_dimension(query) == "time:minute" and
       DateTimeRange.length(query.utc_time_range, :minute) > @max_hours_for_minute_interval * 60 do
    {:error,
     %QueryError{
       code: :invalid_dimensions,
       message: "Invalid dimensions. Dimension `time:minute` is only supported for time ranges up to 30 hours."
     }}
  else
    :ok
  end
end
```

**性能考量**：
- 分钟级时间维度会产生大量数据点
- 30小时的限制平衡了精度和性能
- 超过此范围应使用更粗的时间粒度（小时、天等）

### 3.5 特殊指标与深度过滤器冲突

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
    {:error,
     %QueryError{
       code: :invalid_filters,
       message: "Invalid filters. When `conversion_rate` or `group_conversion_rate` metrics are used, custom property filters can only be used on top level."
     }}
  else
    :ok
  end
end
```

**技术限制原因**：
- 转化率计算需要特殊的会话级别的聚合逻辑
- 深度嵌套的自定义属性过滤器在子查询中难以正确处理
- 这是SQL查询复杂性与性能之间的权衡

### 3.6 指标验证

**位置**：`lib/plausible/stats/query_builder.ex:475-570`

```elixir
defp validate_metrics(query) do
  with :ok <- validate_list(query.metrics, &validate_metric(&1, query)) do
    TableDecider.validate_no_metrics_dimensions_conflict(query)
  end
end

defp validate_metric(metric, query) when metric in [:conversion_rate, :group_conversion_rate] do
  if Enum.member?(query.dimensions, "event:goal") or
       Filters.filtering_on_dimension?(query, "event:goal", behavioral_filters: :ignore) do
    :ok
  else
    {:error,
     %QueryError{
       code: :invalid_metrics,
       message: "Metric `#{metric}` can only be queried with event:goal filters or dimensions."
     }}
  end
end

defp validate_metric(:scroll_depth = metric, query) do
  page_dimension? = Enum.member?(query.dimensions, "event:page")
  toplevel_page_filter? = not is_nil(Filters.get_toplevel_filter(query, "event:page"))

  if page_dimension? or toplevel_page_filter? do
    :ok
  else
    {:error,
     %QueryError{
       code: :invalid_metrics,
       message: "Metric `#{metric}` can only be queried with event:page filters or dimensions."
     }}
  end
end

defp validate_metric(:exit_rate = metric, query) do
  case {query.dimensions, TableDecider.sessions_join_events?(query)} do
    {["visit:exit_page"], false} ->
      :ok

    {["visit:exit_page"], true} ->
      {:error,
       %QueryError{
         code: :invalid_metrics,
         message: "Metric `#{metric}` cannot be queried when filtering on event dimensions."
       }}

    _ ->
      {:error,
       %QueryError{
         code: :invalid_metrics,
         message: "Metric `#{metric}` requires a `\"visit:exit_page\"` dimension. No other dimensions are allowed."
       }}
  end
end
```

### 3.7 自定义属性访问验证

**位置**：`lib/plausible/stats/query_builder.ex:446-473`

```elixir
defp validate_custom_props_access(site, query) do
  allowed_props = Plausible.Props.allowed_for(site, bypass_setup?: true)

  validate_custom_props_access(site, query, allowed_props)
end

defp validate_custom_props_access(_site, _query, :all), do: :ok

defp validate_custom_props_access(_site, query, allowed_props) do
  valid? =
    query.filters
    |> Filters.dimensions_used_in_filters()
    |> Enum.concat(query.dimensions)
    |> Enum.all?(fn
      "event:props:" <> prop -> prop in allowed_props
      _ -> true
    end)

  if valid? do
    :ok
  else
    {:error,
     %QueryError{
       code: :feature_access,
       message: "The owner of this site does not have access to the custom properties feature."
     }}
  end
end
```

## 4. 性能取舍与优化策略

### 4.1 派生名称过滤（Derived Name Filter）

**位置**：`lib/plausible/stats/sql/where_builder.ex:29-56`

```elixir
def derived_name_filter(query) do
  cond do
    Plausible.Stats.Filters.filtering_on_dimension?(query.filters, "event:goal") ->
      true

    query.preloaded_goals.matching_toplevel_filters != [] ->
      names = goal_event_names(query.preloaded_goals.matching_toplevel_filters)
      dynamic([e], e.name in ^names)

    :time_on_page not in query.metrics and :scroll_depth not in query.metrics ->
      dynamic([e], e.name != "engagement")

    true ->
      true
  end
end

defp goal_event_names(goals) do
  goals
  |> Enum.map(fn goal ->
    case Plausible.Goal.type(goal) do
      :event -> goal.event_name
      :page -> "pageview"
      :scroll -> "engagement"
    end
  end)
  |> Enum.uniq()
end
```

**优化效果**：
1. **有目标过滤时**：不添加额外限制，让`Goals.add_filter`处理精确条件
2. **有预加载目标时**：限制为相关的事件名称，减少扫描的数据量
3. **无特殊需求时**：排除`engagement`事件，这些事件只用于滚动深度和页面停留时间计算

### 4.2 目标过滤的特殊处理

**位置**：`lib/plausible/stats/sql/where_builder.ex:160-162`

```elixir
defp add_filter(:events, query, [_, "event:goal" | _rest] = filter) do
  Plausible.Stats.Goals.add_filter(query, filter)
end
```

**位置**：`lib/plausible/stats/goals.ex:52-64`

```elixir
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

**设计特点**：
1. **预加载目标**：目标列表在查询构建阶段就已预加载，避免运行时数据库查询
2. **构建动态条件**：根据目标类型（事件、页面、滚动）构建不同的SQL条件
3. **OR组合**：多个目标之间使用OR连接

### 4.3 目标分组的ARRAY JOIN优化

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

**性能优化**：
1. **ARRAY JOIN**：使用ClickHouse的ARRAY JOIN特性，一次性处理所有目标
2. **预计算数据**：将目标配置转换为数组形式，避免在SQL中进行复杂的条件判断
3. **分情况处理**：根据是否有自定义属性使用不同的JOIN表达式

### 4.4 会话表时间范围优化

**位置**：`lib/plausible/stats/sql/where_builder.ex:100-120`

```elixir
defp filter_time_range(:sessions, query) do
  {first_datetime, last_datetime} = utc_boundaries(query)

  dynamic(
    [s],
    s.start >= ^NaiveDateTime.add(first_datetime, -7, :day) and
      s.timestamp >= ^first_datetime and
      s.start <= ^last_datetime
  )
end
```

**优化原因**：
- ClickHouse的sessions表主键只包含`start`列
- 仅使用`timestamp`过滤会导致采样因子估计偏差
- 额外添加`start >= first_datetime - 7 days`确保主键条件存在
- 同时使用`start <= last_datetime`限制扫描范围

### 4.5 表连接决策

**位置**：`lib/plausible/stats/sql/query_builder.ex:74-107`

```elixir
defp join_sessions_if_needed(q, query) do
  if TableDecider.events_join_sessions?(query) do
    %{session: dimensions} = TableDecider.partition_dimensions(query)

    sessions_q =
      from(
        s in "sessions_v2",
        where: ^SQL.WhereBuilder.build(:sessions, query),
        where: s.sign == 1,
        select: %{session_id: s.session_id},
        group_by: s.session_id
      )

    sessions_q =
      Enum.reduce(dimensions, sessions_q, fn dimension, acc ->
        Plausible.Stats.SQL.Expression.select_dimension_internal(acc, dimension)
      end)

    from(
      e in q,
      join: sq in subquery(sessions_q),
      on: e.session_id == sq.session_id
    )
  else
    q
  end
end
```

**连接策略**：
1. 当查询包含仅会话表有的维度（如`entry_page`, `exit_page`）时，需要JOIN会话表
2. 先构建会话子查询，按session_id分组
3. 选择需要的会话维度列
4. 使用INNER JOIN（或LEFT JOIN，取决于`sql_join_type`）连接事件和会话

### 4.6 采样支持（企业版）

**位置**：`lib/plausible/stats/sql/query_builder.ex:43-45, 63-65, 95-97, 121-123`

```elixir
on_ee do
  q = Plausible.Stats.Sampling.add_query_hint(q, events_query)
end
```

**性能考虑**：
- 对于大数据量的站点，使用采样可以显著提高查询性能
- 采样阈值在`Query`结构中定义：`sample_threshold: 20_000_000`
- 通过添加查询提示（query hint）实现采样

### 4.7 自定义属性的高效查询

**位置**：`lib/plausible/stats/sql/where_builder.ex:217-297`

```elixir
defp filter_custom_prop(prop_name, column_name, [:is, _, clauses | _rest] = filter) do
  none_value_included = Enum.member?(clauses, "(none)")
  prop_value_expr = custom_prop_value(column_name, prop_name)

  dynamic(
    [t],
    (has_key(t, column_name, ^prop_name) and ^in_clause(prop_value_expr, filter)) or
      (^none_value_included and not has_key(t, column_name, ^prop_name))
  )
end
```

**使用的SQL函数**（在`Fragments`模块中定义）：
- `has_key(t, column, key)`：检查数组中是否存在键
- `get_by_key(t, column, key)`：按键获取值
- 利用ClickHouse的Array类型特性高效处理键值对

## 5. 数据结构定义

### 5.1 ParsedQueryParams

```elixir
# lib/plausible/stats/parsed_query_params.ex:4-22
defstruct input_date_range: nil,
          relative_date: nil,
          metrics: [],
          filters: [],
          dimensions: [],
          order_by: nil,
          pagination: nil,
          now: nil,
          include: %Plausible.Stats.QueryInclude{},
          skip_goal_existence_check: false
```

**辅助方法**：
- `add_or_replace_filter/2`：添加或替换过滤器，会移除同维度的现有过滤器
- `conversion_goal_filter?/1`：检查是否有目标转化过滤器

### 5.2 Query

```elixir
# lib/plausible/stats/query.ex:4-34
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
          preloaded_goals: [],
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

### 5.3 过滤器结构

过滤器是一个列表，格式如下：

```elixir
# 简单过滤
[:is, "visit:country", ["US", "CA"]]
[:is, "event:page", ["/checkout"], %{case_sensitive: false}]

# 目标过滤
[:is, "event:goal", ["Purchase", "Signup"]]

# 逻辑组合
[:and, [
  [:is, "visit:country", ["US"]],
  [:is, "event:goal", ["Purchase"]]
]]

# 行为过滤
[:has_done, [:is, "event:name", ["AddToCart"]]]

# 嵌套否定
[:not, [:is, "visit:device", ["Desktop"]]]
```

## 6. 关键协作流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           查询请求入口                                         │
│  StatsController.query / LiveView事件处理                                     │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      API Query Parser 阶段                                    │
│  lib/plausible/stats/api_query_parser.ex                                     │
│                                                                               │
│  职责：                                                                        │
│  - JSONSchema.validate 验证参数格式                                            │
│  - parse_input_date_range 解析日期范围                                         │
│  - parse_metrics 解析指标列表                                                  │
│  - parse_filters 解析过滤器（递归处理嵌套结构）                                 │
│  - parse_dimensions 解析维度                                                   │
│  - parse_order_by 解析排序                                                     │
│  - parse_pagination 解析分页                                                   │
│  - parse_include 解析附加选项                                                  │
│                                                                               │
│  输出：%ParsedQueryParams{}                                                   │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Query Builder 阶段                                        │
│  lib/plausible/stats/query_builder.ex                                        │
│                                                                               │
│  步骤：                                                                        │
│  1. resolve_segments_in_filters - 解析段落在过滤器中的引用                     │
│  2. do_build - 构建基础Query结构                                               │
│     - build_datetime_range - 构建UTC时间范围                                   │
│     - preload_goals_and_revenue - 预加载目标和收入数据                         │
│  3. validate_order_by - 验证排序字段                                           │
│  4. validate_custom_props_access - 验证自定义属性访问权限                       │
│  5. validate_case_sensitive_filter_modifier - 验证大小写修饰符                  │
│  6. validate_toplevel_only_filter_dimension - 验证仅顶层维度                   │
│  7. validate_time_dimension_granularity - 验证时间粒度                          │
│  8. validate_special_metrics_filters - 验证特殊指标与过滤器冲突                 │
│  9. validate_behavioral_filters - 验证行为过滤器                                │
│  10. validate_filtered_goals_exist - 验证目标存在性                            │
│  11. validate_revenue_metrics_access - 验证收入指标访问权限                     │
│  12. validate_metrics - 验证指标有效性                                          │
│  13. validate_include - 验证附加选项                                            │
│                                                                               │
│  后处理：                                                                      │
│  - set_time_on_page_data - 设置页面停留时间数据                                 │
│  - put_comparison_utc_time_range - 设置对比时间范围                              │
│  - Query.put_imported_opts - 设置导入数据选项                                   │
│  - Sampling.put_threshold - 设置采样阈值（企业版）                              │
│                                                                               │
│  输出：%Query{} 或 {:error, %QueryError{}}                                    │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      SQL Query Builder 阶段                                    │
│  lib/plausible/stats/sql/query_builder.ex                                    │
│                                                                               │
│  步骤：                                                                        │
│  1. QueryOptimizer.split - 优化查询，可能拆分为多个子查询                       │
│  2. build_table_query - 为每个表类型构建查询                                   │
│     - events_v2 表：                                                          │
│       - SQL.WhereBuilder.build - 构建WHERE条件                                │
│       - SQL.WhereBuilder.derived_name_filter - 派生名称过滤（优化）            │
│       - select_event_metrics - 选择指标                                        │
│       - join_sessions_if_needed - 按需JOIN会话表                               │
│       - build_group_by - 构建GROUP BY                                         │
│       - merge_imported - 合并导入数据                                          │
│       - SQL.SpecialMetrics.add - 添加特殊指标                                  │
│     - sessions_v2 表：                                                         │
│       - 类似流程，但针对会话数据                                                 │
│  3. join_query_results - 合并多表查询结果                                       │
│  4. build_order_by - 构建ORDER BY                                              │
│  5. paginate - 添加LIMIT/OFFSET                                                │
│  6. select_total_rows - 选择总行数（如果需要）                                  │
│                                                                               │
│  输出：Ecto.Query 结构                                                         │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      WHERE Builder 详细流程                                     │
│  lib/plausible/stats/sql/where_builder.ex                                    │
│                                                                               │
│  入口：build(table, query)                                                    │
│                                                                               │
│  基础条件：                                                                    │
│  - filter_site_time_range(table, query)                                      │
│    - filter_site_id: site_id 或 consolidated_site_ids IN (...)                │
│    - filter_time_range: 时间范围条件                                          │
│                                                                               │
│  过滤器处理（递归）：                                                           │
│  - :ignore_in_totals_query -> 递归处理内部过滤器                               │
│  - :not -> 取反内部条件                                                        │
│  - :and -> 多个条件AND组合                                                     │
│  - :or -> 多个条件OR组合                                                       │
│  - :has_done -> 子查询（session_id IN 子查询）                                 │
│  - :has_not_done -> 取反:has_done                                             │
│  - [:is, "event:goal", ...] -> Goals.add_filter 特殊处理                       │
│  - 其他 -> 常规字段过滤                                                         │
│                                                                               │
│  目标过滤特殊处理：                                                             │
│  lib/plausible/stats/goals.ex:add_filter                                      │
│  - 遍历每个目标条款                                                             │
│  - filter_preloaded: 从预加载列表中筛选匹配的目标                               │
│  - build_condition: 根据目标类型构建条件                                        │
│    - :event -> e.name == event_name AND 自定义属性条件（如果有）                │
│    - :page -> e.name == "pageview" AND pathname条件                           │
│    - :scroll -> e.name == "engagement" AND scroll_depth条件                   │
│  - OR组合所有条件                                                               │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 7. 边界条件与验证总结

| 验证项 | 位置 | 错误码 | 触发条件 |
|--------|------|--------|----------|
| 目标存在性 | `validate_filtered_goals_exist` | `:invalid_filters` | 使用未配置的目标名（:is操作符） |
| 行为过滤器嵌套 | `validate_behavioral_filters` | `:invalid_filters` | :has_done/:has_not_done嵌套深度>1 |
| 行为过滤器维度限制 | `validate_behavioral_filters` | `:invalid_filters` | 行为过滤器用于非event:维度 |
| 仅顶层维度 | `validate_toplevel_only_filter_dimension` | `:invalid_filters` | event:goal或event:hostname在嵌套过滤器中使用 |
| 时间粒度 | `validate_time_dimension_granularity` | `:invalid_dimensions` | time:minute用于超过30小时的范围 |
| 特殊指标与深度过滤 | `validate_special_metrics_filters` | `:invalid_filters` | conversion_rate与深度自定义属性过滤共用 |
| 转化率指标依赖 | `validate_metric(:conversion_rate)` | `:invalid_metrics` | 无event:goal过滤器或维度时使用转化率 |
| 滚动深度指标依赖 | `validate_metric(:scroll_depth)` | `:invalid_metrics` | 无event:page过滤器或维度时使用滚动深度 |
| 退出率指标限制 | `validate_metric(:exit_rate)` | `:invalid_metrics` | 维度不是[visit:exit_page]，或有事件维度过滤 |
| 自定义属性访问 | `validate_custom_props_access` | `:feature_access` | 无权限时使用自定义属性 |
| 大小写修饰符限制 | `validate_case_sensitive_filter_modifier` | `:invalid_filters` | 模式匹配操作符使用case_sensitive修饰符 |
| 排序字段验证 | `validate_order_by` | `:invalid_order_by` | 排序字段不是查询的指标或维度 |
| 包含选项验证 | `validate_include` | `:invalid_include` | time_labels需要时间维度 |

## 8. 性能优化策略总结

| 优化策略 | 位置 | 实现方式 | 效果 |
|----------|------|----------|------|
| 派生名称过滤 | `derived_name_filter` | 预加载目标事件名，添加`name IN (...)`条件 | 减少扫描的事件数量 |
| 目标预加载 | `preload_needed_goals` | 查询构建阶段一次性加载所有相关目标 | 避免运行时N+1查询 |
| ARRAY JOIN分组 | `event_goal_join` | 使用ClickHouse ARRAY JOIN一次性处理所有目标 | 避免多次扫描或复杂条件判断 |
| 会话表主键优化 | `filter_time_range(:sessions)` | 额外添加`start >= first_datetime - 7d` | 确保采样因子估计准确 |
| 按需表连接 | `join_sessions_if_needed` | 仅当需要会话维度时才JOIN | 避免不必要的JOIN开销 |
| 采样支持 | `Sampling.add_query_hint` | 大数据量时使用采样 | 显著提高查询性能（牺牲精度） |
| 自定义属性数组操作 | `filter_custom_prop` | 使用ClickHouse Array函数 | 高效处理键值对数据 |
| 分表查询优化 | `QueryOptimizer.split` | 可能拆分为events和sessions表分别查询 | 利用各自的索引和存储特性 |

## 9. 扩展阅读与相关文件

- `lib/plausible/stats/table_decider.ex` - 表连接决策逻辑
- `lib/plausible/stats/sql/expression.ex` - SQL表达式构建
- `lib/plausible/stats/query_optimizer.ex` - 查询优化器
- `lib/plausible/stats/imported/` - 导入数据处理
- `lib/plausible/stats/legacy/` - 旧版API兼容

## 10. 典型查询示例

### 示例1：目标转化查询

**输入参数**：
```json
{
  "site_id": "example.com",
  "date_range": "30d",
  "metrics": ["visitors", "conversion_rate"],
  "filters": [["is", "event:goal", ["Purchase", "Signup"]]],
  "dimensions": ["event:goal"]
}
```

**执行流程**：
1. `ApiQueryParser.parse` 解析为 `ParsedQueryParams`
2. `QueryBuilder.build` 验证并构建 `Query`：
   - 预加载目标配置（Purchase和Signup）
   - 验证 `conversion_rate` 需要 `event:goal` 维度（满足）
   - 验证目标存在性（假设已配置）
3. `SQL.QueryBuilder.build` 构建SQL：
   - `derived_name_filter` 限制 `name IN ("Purchase", "pageview", ...)`
   - `Goals.add_filter` 构建OR条件：
     - Purchase（事件目标）：`name = 'Purchase'`
     - Signup（页面目标）：`name = 'pageview' AND pathname = '/signup'`
   - `ARRAY JOIN` 进行目标分组

### 示例2：行为过滤器查询

**输入参数**：
```json
{
  "site_id": "example.com",
  "date_range": "7d",
  "metrics": ["visitors"],
  "filters": [
    ["has_done", ["is", "event:name", ["AddToCart"]]],
    ["is_not", "event:goal", ["Purchase"]]
  ],
  "dimensions": ["visit:source"]
}
```

**执行流程**：
1. 验证行为过滤器：
   - 检查是否嵌套（未嵌套，通过）
   - 检查是否用于event维度（是，通过）
2. 验证目标过滤器：
   - `:is_not` 操作符不触发存在性检查
3. SQL构建：
   - `:has_done` 转换为子查询：`session_id IN (SELECT session_id FROM events WHERE name = 'AddToCart')`
   - `:is_not` 目标过滤：`NOT (目标条件)`

---

**报告生成时间**：2026-05-03  
**分析范围**：目标转化与过滤查询从页面状态到分析查询的完整流程
