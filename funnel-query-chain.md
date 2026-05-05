# Plausible 漏斗分析查询链详解

本文档详细介绍 Plausible 漏斗分析从前端配置到后端 ClickHouse 查询再到 React 组件展示的完整流程。

## 一、整体架构概览

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                    前端层 (React + LiveView)                            │
├─────────────────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────────────────┐   │
│  │ FunnelSettings   │    │ Funnel Component │    │ DashboardState Context       │   │
│  │ (LiveView 配置)  │────▶│ (React 展示)    │◀───│ (时间范围/过滤器共享)        │   │
│  └──────────────────┘    └──────────────────┘    └──────────────────────────────┘   │
│           │                        │                              │                   │
└───────────┼────────────────────────┼──────────────────────────────┼───────────────────┘
            │                        │                              │
            ▼                        ▼                              ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                    API 层                                               │
├─────────────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────┐    ┌─────────────────────────────────────────┐      │
│  │ Plugins API (Funnels CRUD)  │    │ Stats API (funnel/2 数据查询)          │      │
│  │ Funnels.create/get/delete   │    │ dashboardState 作为查询参数传递          │      │
│  └─────────────────────────────┘    └─────────────────────────────────────────┘      │
└───────────┬────────────────────────────────────────────────────────┬───────────────────┘
            │                                                        │
            ▼                                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                    业务逻辑层                                           │
├─────────────────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────────────────┐   │
│  │ Plausible.Funnel │    │ Plausible.Funnels│    │ Plausible.Stats.QueryBuilder │   │
│  │ (Schema 定义)    │    │ (存储/检索)      │    │ (Query 结构构建)             │   │
│  └──────────────────┘    └──────────────────┘    └──────────────────────────────┘   │
│                                                        │                              │
│                                                        ▼                              │
│  ┌──────────────────────────────────────────────────────────────────────────────┐   │
│  │                    Plausible.Stats.Funnel (核心查询逻辑)                      │   │
│  │  - funnel/3: 入口函数，处理 funnel_id → Funnel 结构                           │   │
│  │  - funnel_query/2: 构建 ClickHouse windowFunnel 查询                         │   │
│  │  - select_funnel/2: 动态拼接 windowFunnel 条件                               │   │
│  │  - backfill_steps/2: 回填步骤数据，计算转化率/流失率                          │   │
│  └──────────────────────────────────────────────────────────────────────────────┘   │
└───────────┬────────────────────────────────────────────────────────┬───────────────────┘
            │                                                        │
            ▼                                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                               ClickHouse 查询层                                         │
├─────────────────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────────────────┐   │
│  │ Plausible.Stats. │    │ Plausible.Stats. │    │ Plausible.ClickhouseRepo     │   │
│  │ Base             │    │ SQL.WhereBuilder  │    │ - parallel_tasks/2 (并发)   │   │
│  │ - base_event_query│    │ (过滤条件构建)   │    │ - prepare_query (日志追踪)  │   │
│  └──────────────────┘    └──────────────────┘    └──────────────────────────────┘   │
│                                                                                         │
│  ┌──────────────────────────────────────────────────────────────────────────────┐   │
│  │                    ClickHouse SQL (windowFunnel 函数)                          │   │
│  │  windowFunnel(86400, 'strict_order')(timestamp, condition1, condition2, ...) │   │
│  │  - 86400: 时间窗口 (24小时)                                                    │   │
│  │  - strict_order: 严格顺序模式                                                  │   │
│  │  - 条件按步骤顺序定义，返回用户达到的最远步骤索引                               │   │
│  └──────────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 二、前端配置流程

### 2.1 LiveView 配置界面

漏斗配置通过 `PlausibleWeb.Live.FunnelSettings` LiveView 组件实现：

**文件位置**: `extra/lib/plausible_web/live/funnel_settings.ex:1-175`

**核心功能**：
```elixir
def mount(_params, %{"site_id" => site_id, "domain" => domain}, socket) do
  socket =
    socket
    |> assign_new(:site, fn %{current_user: current_user} ->
      Plausible.Sites.get_for_user!(current_user, domain,
        roles: [:owner, :admin, :editor, :super_admin]
      )
    end)
    |> assign_new(:all_funnels, fn %{site: %{id: ^site_id} = site} ->
      Funnels.list(site)
    end)
    |> assign_new(:goal_count, fn %{site: site} ->
      Goals.count(site)
    end)
  # ...
end
```

**配置流程**：
1. 用户通过 LiveView 界面选择已配置的 Goals 作为漏斗步骤
2. 步骤顺序可调整，支持 **2-8 个步骤** (`Funnel.min_steps()` 到 `Funnel.max_steps()`)
   - `Funnel.Const.min_steps()` = 2
   - `Funnel.Const.max_steps()` = 8
3. 支持 `strict_order` 模式（严格顺序）和非严格顺序
4. 配置保存到 PostgreSQL 数据库的 `funnels` 表

> **注意**：Exploration（用户行为探索）功能有不同的限制：
> - 最大支持 20 个步骤 (`@max_steps 20`)
> - `interesting_funnel` 默认使用 6 个步骤

### 2.2 Dashboard 状态共享机制

**文件位置**: `assets/js/dashboard/dashboard-state.ts:1-235`

`DashboardState` 是整个 Dashboard 共享的核心状态，包含：

```typescript
export type DashboardState = {
  period: DashboardPeriod           // 时间周期 (如 '28d', 'day', 'month')
  comparison: ComparisonMode | null  // 对比模式
  match_day_of_week: boolean         // 对比时匹配星期
  date: Dayjs | null                  // 参考日期
  from: Dayjs | null                  // 自定义开始日期
  to: Dayjs | null                    // 自定义结束日期
  compare_from: Dayjs | null          // 对比开始日期
  compare_to: Dayjs | null            // 对比结束日期
  filters: Filter[]                   // 过滤器列表 (关键!)
  resolvedFilters: Filter[]           // 解析后的过滤器 (Segment 展开)
  labels: FilterClauseLabels          // 过滤器显示标签
  with_imported: boolean              // 是否包含导入数据
}
```

**过滤器结构** (`Filter`):
```typescript
// 格式: [operator, dimension, clauses]
// 示例: ["is", "visit:country", ["CN", "US"]]
// 示例: ["contains", "event:page", ["/checkout"]]
export type Filter = [FilterOperator, FilterKey, FilterClause[]]
```

### 2.3 前端漏斗组件

**文件位置**: `assets/js/dashboard/extra/funnel.js:1-417`

漏斗数据获取逻辑：

```javascript
// 从 Context 获取共享状态
const { dashboardState } = useDashboardStateContext()

// 当 dashboardState 变化时重新获取数据
useEffect(() => {
  if (visible) {
    setLoading(true)
    fetchFunnel()
      .then((res) => {
        setFunnel(res)
        setError(undefined)
      })
      .catch((error) => {
        setError(error)
      })
      .finally(() => {
        setLoading(false)
      })
  }
}, [dashboardState, funnelName, visible, isSmallScreen])

// 调用 API 时传递 dashboardState
const fetchFunnel = async () => {
  const funnelMeta = getFunnel()
  return api.get(
    `/api/stats/${encodeURIComponent(site.domain)}/funnels/${funnelMeta.id}`,
    dashboardState  // 关键：传递整个 dashboardState 作为查询参数
  )
}
```

### 2.4 API 参数序列化

**文件位置**: `assets/js/dashboard/api.ts:53-103`

`dashboardStateToParams` 函数将状态转换为 URL 查询参数：

```javascript
export function dashboardStateToParams(
  dashboardState: DashboardState,
  extraQuery: unknown[] = []
): Record<string, string> {
  const queryObj: Record<string, string> = {}
  
  // 时间参数
  if (dashboardState.period) queryObj.period = dashboardState.period
  if (dashboardState.date) queryObj.date = formatISO(dashboardState.date)
  if (dashboardState.from) queryObj.from = formatISO(dashboardState.from)
  if (dashboardState.to) queryObj.to = formatISO(dashboardState.to)
  
  // 关键：过滤器参数
  if (dashboardState.filters) {
    queryObj.filters = serializeApiFilters(dashboardState.filters)
  }
  
  // 导入数据
  if (dashboardState.with_imported) {
    queryObj.with_imported = String(dashboardState.with_imported)
  }
  
  // 对比参数
  if (dashboardState.comparison) {
    queryObj.comparison = dashboardState.comparison
    // ...
  }
  
  return queryObj
}
```

---

## 三、API 层处理

### 3.1 漏斗数据查询 API

**文件位置**: `lib/plausible_web/controllers/api/stats_controller.ex:261-295`

```elixir
def funnel(conn, %{"id" => funnel_id} = params) do
  site = Plausible.Repo.preload(conn.assigns.site, :team)

  with :ok <- Plausible.Billing.Feature.Funnels.check_availability(site.team),
       # 步骤1: 从 params 构建 Query 结构
       query <- Query.from(site, params, debug_metadata: debug_metadata(conn)),
       # 步骤2: 验证漏斗查询的过滤条件限制
       :ok <- validate_funnel_query(query),
       {funnel_id, ""} <- Integer.parse(funnel_id),
       # 步骤3: 执行漏斗查询
       {:ok, funnel} <- Stats.funnel(site, query, funnel_id) do
    json(conn, funnel)
  else
    {:error, {:invalid_funnel_query, due_to}} ->
      bad_request(
        conn,
        "We are unable to show funnels when the dashboard is filtered by #{due_to}",
        %{level: :normal}
      )
    # ... 其他错误处理
  end
end
```

### 3.2 漏斗查询限制验证

**文件位置**: `lib/plausible_web/controllers/api/stats_controller.ex:297-311`

```elixir
defp validate_funnel_query(query) do
  cond do
    # 限制1: 不能过滤 goals (漏斗本身就是基于 goals 的)
    toplevel_goal_filter?(query) ->
      {:error, {:invalid_funnel_query, "goals"}}
    
    # 限制2: 不能过滤 pages
    Filters.filtering_on_dimension?(query, "event:page") ->
      {:error, {:invalid_funnel_query, "pages"}}
    
    # 限制3: 不支持实时模式
    query.input_date_range == :realtime ->
      {:error, {:invalid_funnel_query, "realtime period"}}
    
    true ->
      :ok
  end
end
```

> **重要**：漏斗查询**不支持实时模式** (`realtime` period)。当用户选择实时时间范围时，漏斗组件会返回错误。这是因为：
> 1. 漏斗分析需要完整的会话数据来计算用户旅程
> 2. 实时数据是增量更新的，无法保证完整的用户旅程
> 3. `windowFunnel` 函数依赖完整的事件序列

### 3.3 Query 结构构建

**文件位置**: `lib/plausible/stats/query_builder.ex:1-596`

`Query.from/3` 是构建查询的入口：

```elixir
def build(site, %ParsedQueryParams{} = parsed_query_params, debug_metadata) do
  with {:ok, parsed_query_params} <- resolve_segments_in_filters(parsed_query_params, site),
       query = do_build(parsed_query_params, site, debug_metadata),
       :ok <- validate_order_by(query),
       :ok <- validate_custom_props_access(site, query),
       :ok <- validate_case_sensitive_filter_modifier(query),
       # ... 更多验证
       do
    query =
      query
      |> set_time_on_page_data(site)
      |> put_comparison_utc_time_range()
      |> Query.put_imported_opts(site)

    {:ok, query}
  end
end
```

**Query 结构核心字段** (`lib/plausible/stats/query.ex`):
```elixir
%Query{
  site_id: integer(),           # 站点 ID
  utc_time_range: DateTimeRange.t(),  # UTC 时间范围
  timezone: String.t(),         # 站点时区
  metrics: [atom()],            # 指标列表
  dimensions: [String.t()],     # 维度列表
  filters: Filters.t(),         # 过滤器树 (关键!)
  preloaded_goals: map(),       # 预加载的 goals
  consolidated_site_ids: [integer()] | nil,  # 合并站点
  # ...
}
```

---

## 四、ClickHouse 查询执行

### 4.1 漏斗查询核心入口

**文件位置**: `extra/lib/plausible/stats/funnel.ex:1-171`

```elixir
defmodule Plausible.Stats.Funnel do
  @funnel_window_duration 86_400  # 24小时窗口 (秒)

  @spec funnel(Plausible.Site.t(), Plausible.Stats.Query.t(), Funnel.t() | pos_integer()) ::
          {:ok, map()} | {:error, :funnel_not_found}
  
  # 重载1: 通过 funnel_id 查询
  def funnel(site, query, funnel_id) when is_integer(funnel_id) do
    case Funnels.get(site.id, funnel_id) do
      %Funnel{} = funnel -> funnel(site, query, funnel)
      nil -> {:error, :funnel_not_found}
    end
  end

  # 重载2: 核心执行函数
  def funnel(_site, query, %Funnel{} = funnel) do
    goals = Enum.map(funnel.steps, & &1.goal)

    funnel_data =
      query
      # 步骤1: 设置预加载的 goals (用于过滤条件)
      |> Query.set(preloaded_goals: %{all: [], matching_toplevel_filters: goals})
      # 步骤2: 构建基础事件查询 (应用 filters)
      |> Base.base_event_query()
      # 步骤3: 构建 funnel 特定查询
      |> funnel_query(funnel)
      # 步骤4: 执行 ClickHouse 查询
      |> ClickhouseRepo.all(query: query)

    # 步骤5: 处理结果，回填步骤数据
    steps = backfill_steps(funnel_data, funnel)

    {:ok,
     %{
       name: funnel.name,
       steps: steps,
       all_visitors: all_visitors,
       entering_visitors: visitors_at_first_step,
       entering_visitors_percentage: percentage(visitors_at_first_step, all_visitors),
       # ...
     }}
  end
```

### 4.2 基础事件查询构建

**文件位置**: `lib/plausible/stats/base.ex:1-52`

```elixir
def base_event_query(query) do
  events_q = query_events(query)

  # 判断是否需要 join sessions 表
  if TableDecider.events_join_sessions?(query) do
    sessions_q =
      from(
        s in query_sessions(query),
        select: %{session_id: s.session_id},
        where: s.sign == 1,
        group_by: s.session_id
      )

    from(
      e in events_q,
      join: sq in subquery(sessions_q),
      on: e.session_id == sq.session_id
    )
  else
    events_q
  end
end

defp query_events(query) do
  q =
    from(e in "events_v2",
      # 关键：应用 Dashboard 过滤器 (WHERE 条件)
      where: ^SQL.WhereBuilder.build(:events, query),
      where: ^SQL.WhereBuilder.derived_name_filter(query)
    )

  on_ee do
    # 企业版：添加采样提示
    q = Plausible.Stats.Sampling.add_query_hint(q, query)
  end

  q
end
```

### 4.3 Funnel 特定查询构建

**文件位置**: `extra/lib/plausible/stats/funnel.ex:68-119`

```elixir
defp funnel_query(query, funnel_definition) do
  # 子查询1: 计算每个 user_id 达到的最远步骤
  q_events =
    from(e in query,
      select: %{user_id: e.user_id, _sample_factor: fragment("any(_sample_factor)")},
      where: e.site_id == ^funnel_definition.site_id,
      group_by: e.user_id,
      order_by: [desc: fragment("step")]
    )
    |> select_funnel(funnel_definition)  # 注入 windowFunnel

  # 外层查询: 按步骤聚合用户数
  from(f in subquery(q_events),
    select: {f.step, total()},  # {步骤索引, 用户数}
    group_by: f.step
  )
end

defp select_funnel(db_query, funnel_definition) do
  # 动态构建 windowFunnel 的条件参数
  window_funnel_steps =
    Enum.reduce(funnel_definition.steps, nil, fn step, acc ->
      # 每个步骤的 goal 条件 (如 "pageview on /checkout")
      goal_condition = Plausible.Stats.Goals.goal_condition(step.goal)

      if acc do
        # 拼接多个条件: fragment("?, ?", previous, current)
        dynamic([q], fragment("?, ?", ^acc, ^goal_condition))
      else
        # 第一个条件
        dynamic([q], fragment("?", ^goal_condition))
      end
    end)

  # 选择是否使用 strict_order 模式
  dynamic_window_funnel =
    if funnel_definition.strict_order do
      dynamic(
        [q],
        fragment(
          "windowFunnel(?, 'strict_order')(timestamp, ?)",
          @funnel_window_duration,  # 86400 秒 = 24小时
          ^window_funnel_steps
        )
      )
    else
      dynamic(
        [q],
        fragment("windowFunnel(?)(timestamp, ?)", @funnel_window_duration, ^window_funnel_steps)
      )
    end

  # 将 step 字段注入 SELECT
  from(q in db_query,
    select_merge:
      ^%{
        step: dynamic_window_funnel
      }
  )
end
```

### 4.4 结果回填与计算

**文件位置**: `extra/lib/plausible/stats/funnel.ex:121-170`

```elixir
defp backfill_steps(funnel_result, funnel) do
  # ClickHouse 返回的是 {step_idx, visitor_count} 元组
  # 例如: [{0, 100}, {1, 80}, {2, 50}, {3, 30}]
  # 其中 step=0 表示未进入漏斗的用户
  
  funnel_result = Enum.into(funnel_result, %{})  # 转为 Map
  max_step = Enum.max_by(funnel.steps, & &1.step_order).step_order

  # 遍历漏斗定义的步骤，回填数据
  funnel
  |> Map.fetch!(:steps)
  |> Enum.reduce({nil, nil, []}, fn step, {total_visitors, visitors_at_previous, acc} ->
    # 关键：累加当前及后续步骤的用户数
    # 因为 windowFunnel 返回的是"达到的最远步骤"
    # 所以 step=3 的用户也经过了 step=1, 2
    visitors_at_step =
      step.step_order..max_step
      |> Enum.map(&Map.get(funnel_result, &1, 0))
      |> Enum.sum()

    # 第一步的用户数作为总数
    total_visitors = total_visitors || current_visitors

    # 计算流失数和转化率
    dropoff = if visitors_at_previous, do: visitors_at_previous - current_visitors, else: 0
    dropoff_percentage = percentage(dropoff, visitors_at_previous)
    conversion_rate = percentage(current_visitors, total_visitors)
    conversion_rate_step = percentage(current_visitors, visitors_at_previous)

    step = %{
      dropoff: dropoff,
      dropoff_percentage: dropoff_percentage,
      conversion_rate: conversion_rate,
      conversion_rate_step: conversion_rate_step,
      visitors: visitors_at_step,
      label: to_string(step.goal)
    }

    {total_visitors, current_visitors, [step | acc]}
  end)
  |> elem(2)
  |> Enum.reverse()
end
```

---

## 五、并发查询机制与结果拼装

### 5.1 并发查询场景概览

**重要澄清**：漏斗查询本身（`Stats.funnel/3`）是**单次 ClickHouse 查询**，使用 `windowFunnel` 函数一次返回所有步骤的数据。它**不使用并发查询**。

并发查询机制 (`ClickhouseRepo.parallel_tasks/2`) 主要用于以下场景：

| 场景 | 函数位置 | 并发方式 | 结果拼装方式 |
|-----|---------|---------|-------------|
| CSV 多维度导出 | `StatsController` L177 | `parallel_tasks/2` (默认 3 并发) | `Enum.zip` + 按 key 匹配 |
| 多站点用量统计 | `Clickhouse.usage_breakdown` L71 | `parallel_tasks/2` (10 并发) | `Enum.reduce` 累加 |
| 多站点概览 | `Sparkline.parallel_overview` L13 | `Task.async_stream` | 直接返回 List |
| 账单周期统计 | `Billing.usage_cycle` L370 | `Task.async_stream` | `Enum.into` 转为 Map |

### 5.2 ClickHouseRepo 并发任务实现

**文件位置**: `lib/plausible/clickhouse_repo.ex:1-79`

```elixir
defmodule Plausible.ClickhouseRepo do
  @task_timeout 60_000  # 60秒超时

  def parallel_tasks(queries, opts \\ []) do
    ctx = OpenTelemetry.Ctx.get_current()

    # 保持 OpenTelemetry 上下文
    execute_with_tracing = fn fun ->
      OpenTelemetry.Ctx.attach(ctx)
      fun.()
    end

    # 最大并发数 (默认 3)
    max_concurrency = Keyword.get(opts, :max_concurrency, 3)

    # 计算任务超时时间
    task_timeout =
      on_ee do
        @task_timeout
      else
        # 开源版: 基于数据库超时计算 (保持 4 倍比例)
        ch_timeout = Keyword.fetch!(config(), :timeout)
        max(ch_timeout * 4, @task_timeout)
      end

    # 使用 Elixir Task.async_stream 并发执行
    Task.async_stream(queries, execute_with_tracing,
      max_concurrency: max_concurrency,
      timeout: task_timeout
    )
    |> Enum.to_list()
    |> Keyword.values()
  end
end
```

**设计要点**：
1. `queries` 参数是一个函数列表，每个函数包装一个查询
2. 使用 `Task.async_stream` 实现背压控制
3. `max_concurrency` 控制最大并发数（默认 3）
4. 保持 OpenTelemetry 追踪上下文，确保分布式追踪的连续性
5. 返回结果是按输入顺序排列的列表

### 5.3 场景一：CSV 多维度导出（结果按 key 匹配）

**文件位置**: `lib/plausible_web/controllers/stats_controller.ex:172-185`

```elixir
# 定义多个 CSV 查询任务
csvs = %{
  ~c"visitors.csv" => fn -> Api.StatsController.visitors(conn, params) end,
  ~c"pageviews.csv" => fn -> Api.StatsController.pageviews(conn, params) end,
  ~c"sources.csv" => fn -> Api.StatsController.sources(conn, params) end,
  ~c"custom_props.csv" => fn -> Api.StatsController.all_custom_prop_values(conn, params) end
}

# 并发执行所有查询
csv_values =
  Map.values(csvs)
  |> Plausible.ClickhouseRepo.parallel_tasks()

# 结果拼装：按 key 匹配
csvs =
  Map.keys(csvs)
  |> Enum.zip(csv_values)
  |> Enum.reject(fn {_k, v} -> is_nil(v) end)
  |> Map.new()
```

**拼装逻辑**：
1. 输入：`%{key1: fun1, key2: fun2, key3: fun3}`
2. 并发执行：`[fun1_result, fun2_result, fun3_result]`（顺序保持）
3. 拼装：`Enum.zip([key1, key2, key3], [result1, result2, result3])`
4. 输出：`%{key1: result1, key2: result2, key3: result3}`

### 5.4 场景二：多站点用量统计（结果累加）

**文件位置**: `lib/plausible/stats/clickhouse.ex:53-75`

```elixir
def usage_breakdown([sid | _] = site_ids, date_range) when is_integer(sid) do
  # 分块处理 (每块 1000 个 site_id)
  Enum.chunk_every(site_ids, 1000)
  |> Enum.map(fn site_ids ->
    # 返回一个函数，将被并发执行
    fn ->
      ClickhouseRepo.one(
        from(e in "events_v2",
          where: e.site_id in ^site_ids,
          # 统计 pageviews 和 custom_events
          select: %{
            pageviews:
              sum(fragment("if(? = 'pageview', 1, 0)", e.name)) / e._sample_factor,
            custom_events:
              sum(fragment("if(? != 'pageview' AND ? != 'engagement', 1, 0)", e.name, e.name))
              / e._sample_factor
          }
        )
      )
    end
  end)
  # 并发执行所有函数 (最大 10 并发)
  |> ClickhouseRepo.parallel_tasks(max_concurrency: 10)
  # 结果拼装：累加所有块的统计结果
  |> Enum.reduce(fn {pageviews, custom_events}, {pageviews_total, custom_events_total} ->
    {pageviews_total + pageviews, custom_events_total + custom_events}
  end)
end
```

**拼装逻辑**：
1. 输入：`[site_ids1, site_ids2, site_ids3]`（分块）
2. 并发执行：返回 `[{pv1, ce1}, {pv2, ce2}, {pv3, ce3}]`
3. 拼装：`Enum.reduce` 累加所有块的 pageviews 和 custom_events
4. 输出：`{total_pv, total_ce}`

### 5.5 场景三：Exploration 功能（迭代式，非并发）

**重要**：Exploration（用户行为探索）使用的是**迭代式查询**，不是**并发查询**。

**文件位置**: `extra/lib/plausible/stats/exploration.ex:146-226`

```elixir
def interesting_funnel(query, opts \\ []) do
  max_steps = min(Keyword.get(opts, :max_steps, 6), @max_steps)
  max_candidates = min(Keyword.get(opts, :max_candidates, 10), @max_candidates)

  # 迭代构建旅程
  with {:ok, result} <-
         build_interesting_journey(query, max_steps, max_candidates, include_wildcard?),
       # 最后再调用一次 journey_funnel 获取完整漏斗数据
       {:ok, funnel} <- journey_funnel(query, result.journey) do
    {:ok, %{funnel: funnel, candidates: result.candidates}}
  end
end

defp do_build_journey(
       query,
       journey,
       step_candidates,
       seen,
       max_steps,
       max_candidates,
       include_wildcard?
     )
     when length(journey) >= max_steps do
  %{journey: journey, candidates: step_candidates}
end

defp do_build_journey(query, journey, step_candidates, seen, max_steps, max_candidates, include_wildcard?) do
  # 每次迭代调用 next_steps 获取下一步候选
  {:ok, candidates} =
    next_steps(query, journey,
      max_candidates: max_candidates,
      include_wildcard?: include_wildcard?
    )

  case find_unseen_step(candidates, seen) do
    nil ->
      # 没有新步骤，结束迭代
      %{journey: journey, candidates: step_candidates}

    step ->
      # 选择一个新步骤，继续迭代
      new_seen = MapSet.put(seen, normalize_step_key(step))

      do_build_journey(
        query,
        journey ++ [step],
        step_candidates ++ [candidates],
        new_seen,
        max_steps,
        max_candidates,
        include_wildcard?
      )
  end
end
```

**执行流程**：
1. 初始状态：`journey = []`
2. 迭代 1：`next_steps(query, [])` → 获取候选步骤 → 选择 step1 → `journey = [step1]`
3. 迭代 2：`next_steps(query, [step1])` → 获取候选步骤 → 选择 step2 → `journey = [step1, step2]`
4. ... 继续直到 max_steps
5. 最后：`journey_funnel(query, journey)` → 计算完整漏斗数据

这是**顺序迭代**，不是**并发执行**。每次查询依赖上一次的结果。

### 5.6 前端多组件并行请求

**文件位置**: `assets/js/dashboard/extra/funnel.js` 和相关组件

前端层面，Dashboard 加载时多个组件会**并行发起 HTTP 请求**：

```javascript
// 主图表组件
useEffect(() => {
  fetchMainGraph(dashboardState)
}, [dashboardState])

// 来源列表组件
useEffect(() => {
  fetchSources(dashboardState)
}, [dashboardState])

// 漏斗组件
useEffect(() => {
  fetchFunnel(dashboardState)
}, [dashboardState])
```

**管理机制**：
1. 使用 `AbortController` 统一管理所有请求
2. 当 `dashboardState` 变化时，取消所有旧请求，发起新请求
3. 避免旧请求覆盖新数据

但这是**浏览器层面的并行 HTTP 请求**，不是**后端层面的并发查询**。每个请求在后端都是独立处理的。

### 5.7 查询追踪与日志

**文件位置**: `lib/plausible/clickhouse_repo.ex:46-67`

```elixir
@impl true
def prepare_query(_operation, query, opts) do
  # 从 opts 中提取 Query 结构
  {plausible_query, opts} = Keyword.pop(opts, :query)

  # 获取当前追踪 ID
  trace_id = Plausible.OpenTelemetry.current_trace_id()

  # 构建 log_comment 数据 (用于 ClickHouse 查询日志)
  log_comment_data =
    if plausible_query do
      Map.put(plausible_query.debug_metadata, :trace_id, trace_id)
    else
      %{trace_id: trace_id}
    end

  log_comment = Jason.encode!(log_comment_data)

  # 将 log_comment 添加到查询设置
  opts =
    Keyword.update(opts, :settings, [log_comment: log_comment], fn settings ->
      [{:log_comment, log_comment} | settings]
    end)

  {query, opts}
end
```

---

## 六、过滤维度共享详解

### 6.1 过滤器传递链条

```
前端 DashboardState.filters
       │
       ▼
api.get(url, dashboardState)
       │
       ▼
dashboardStateToParams() 序列化为 URL 参数
       │
       ▼
后端 Query.from(site, params) 解析
       │
       ▼
ParsedQueryParams.filters
       │
       ▼
QueryBuilder.build() 验证和处理
       │
       ▼
Query.filters (过滤器树结构)
       │
       ▼
SQL.WhereBuilder.build(:events, query) 生成 WHERE 子句
       │
       ▼
ClickHouse 查询执行
```

### 6.2 过滤器解析示例

假设前端有以下过滤器：
```javascript
// dashboardState.filters
[
  ["is", "visit:country", ["CN", "US"]],
  ["contains", "event:page", ["/checkout"]]
]
```

**序列化后** (`serializeApiFilters`):
```
"visit:country==CN|US;event:page=~/checkout"
```

**后端解析** (`lib/plausible/stats/filters/filters.ex`):
- 解析为嵌套的 filter 结构
- 支持 `:is`, `:is_not`, `:contains`, `:contains_not`, `:matches`, `:matches_not` 等操作符
- 支持逻辑组合 (`:and`, `:or`)
- 支持行为过滤器 (`:has_done`, `:has_not_done`)

### 6.3 漏斗查询的过滤器限制

如前文 `validate_funnel_query/1` 所示，漏斗查询有特殊限制：

| 过滤器类型 | 是否允许 | 原因 |
|-----------|---------|------|
| `event:goal` | ❌ 不允许 | 漏斗本身基于 goals 定义，会造成冲突 |
| `event:page` | ❌ 不允许 | 页面过滤会干扰漏斗步骤的 pageview 匹配 |
| `visit:source`, `visit:country` 等 | ✅ 允许 | 这些是用户/会话级别的过滤，不影响步骤匹配 |
| `event:props:*` | ✅ 允许 (顶层) | 自定义属性过滤需在顶层使用 |

### 6.4 预加载 Goals 机制

**文件位置**: `extra/lib/plausible/stats/funnel.ex:31-41`

```elixir
funnel_data =
  query
  # 关键：将漏斗步骤的 goals 注入 preloaded_goals
  |> Query.set(preloaded_goals: %{all: [], matching_toplevel_filters: goals})
  |> Base.base_event_query()
  # ...
```

这确保了：
1. `SQL.WhereBuilder` 能正确识别和处理 goal 相关的条件
2. 漏斗步骤的 goal 条件能正确应用到事件查询中

---

## 七、完整执行流程图

### 7.1 单次漏斗查询执行流程

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           用户查看漏斗 Dashboard                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  1. React Funnel 组件检测到 dashboardState 变化 (useEffect)                          │
│     - 时间范围改变                                                                      │
│     - 过滤器添加/移除                                                                   │
│     - 漏斗名称切换                                                                      │
│                                                                                         │
│  ⚠️ 注意：漏斗不支持实时模式 (realtime period)                                         │
│     当 query.input_date_range == :realtime 时会返回错误                              │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  2. 调用 api.get(`/api/stats/:domain/funnels/:id`, dashboardState)                  │
│     - dashboardState 包含 filters, period, from/to 等                               │
│     - 通过 dashboardStateToParams() 序列化为 URL 查询参数                            │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  3. 后端 StatsController.funnel/2 处理                                                │
│     a. 检查 Funnels 功能权限 (Business Plan)                                          │
│     b. Query.from(site, params) 构建 Query 结构                                      │
│     c. validate_funnel_query(query) 验证过滤限制                                      │
│        - 不允许 goal 过滤                                                              │
│        - 不允许 page 过滤                                                              │
│        - 不允许 realtime period                                                        │
│     d. Stats.funnel(site, query, funnel_id) 执行查询                                 │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  4. Plausible.Stats.Funnel.funnel/3 核心逻辑                                         │
│     a. Funnels.get() 从 PostgreSQL 读取漏斗定义 (steps, strict_order)                │
│        - 步骤数量限制：2-8 步                                                          │
│     b. Query.set(preloaded_goals: ...) 注入步骤 goals                               │
│     c. Base.base_event_query() 应用 dashboard 过滤器                                 │
│     d. funnel_query() 构建 windowFunnel SQL                                          │
│     e. ClickhouseRepo.all() 执行查询 (单次查询，非并发)                                │
│     f. backfill_steps() 回填数据，计算转化率                                         │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  5. ClickHouse 执行 SQL (核心)                                                        │
│                                                                                         │
│  生成的 SQL 大致如下:                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────────────┐   │
│  │ SELECT f.step, count(*) AS total                                               │   │
│  │ FROM (                                                                          │   │
│  │   SELECT                                                                        │   │
│  │     e.user_id,                                                                  │   │
│  │     windowFunnel(86400, 'strict_order')(                                       │   │
│  │       e.timestamp,                                                              │   │
│  │       e.name = 'pageview' AND path = '/landing',  -- 步骤1 条件              │   │
│  │       e.name = 'pageview' AND path = '/product',  -- 步骤2 条件              │   │
│  │       e.name = 'Purchase'                        -- 步骤3 条件 (自定义事件)   │   │
│  │       -- ... 支持最多 8 个步骤                                                  │   │
│  │     ) AS step                                                                    │   │
│  │   FROM events_v2 e                                                               │   │
│  │   WHERE e.site_id = 123                                                          │   │
│  │     AND e.timestamp >= '2024-01-01'                                             │   │
│  │     AND e.timestamp < '2024-02-01'                                              │   │
│  │     AND visit:country IN ('CN', 'US')  -- Dashboard 过滤器应用                  │   │
│  │   GROUP BY e.user_id                                                             │   │
│  │ ) f                                                                               │   │
│  │ GROUP BY f.step                                                                   │   │
│  └──────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                         │
│  ClickHouse windowFunnel 函数说明:                                                     │
│  - 按时间顺序扫描每个 user_id 的事件                                                   │
│  - 返回用户达到的最远步骤索引 (0 表示未进入任何步骤)                                   │
│  - 'strict_order' 模式要求事件按严格顺序发生，中间不能有其他步骤的事件                │
│  - 时间窗口 86400秒 (24小时) 内完成才算有效                                           │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  6. 结果处理与返回                                                                      │
│     a. backfill_steps() 将 {step_idx, count} 转为完整步骤数据                        │
│        - 累加当前及后续步骤的用户数 (因为 step=3 的用户也经过了 1,2)                │
│        - 计算 dropoff (流失数) 和 conversion_rate (转化率)                           │
│     b. 返回 JSON 给前端                                                                │
│        {                                                                               │
│          "name": "Checkout Funnel",                                                   │
│          "steps": [                                                                    │
│            {"label": "Visit Landing", "visitors": 1000, "conversion_rate": 100},   │
│            {"label": "View Product", "visitors": 800, "conversion_rate": 80},      │
│            {"label": "Purchase", "visitors": 200, "conversion_rate": 20}           │
│          ]                                                                             │
│        }                                                                               │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  7. 前端渲染 (Chart.js)                                                                │
│     - 绘制漏斗图 (水平堆叠柱状图)                                                      │
│     - 显示每个步骤的访客数、转化率、流失率                                            │
│     - 支持响应式布局 (小屏幕显示简化版)                                               │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 并发查询场景对比

| 场景 | 查询类型 | 执行方式 | 结果拼装 |
|-----|---------|---------|---------|
| **标准漏斗查询** | 单次 `windowFunnel` | 顺序执行 | 无需拼装，直接返回 |
| **CSV 多维度导出** | 多次独立查询 | `parallel_tasks` (后端并发) | 按 key 匹配 `Enum.zip` |
| **多站点用量统计** | 多次分块查询 | `parallel_tasks` (后端并发) | 累加 `Enum.reduce` |
| **Exploration** | 多次依赖查询 | 顺序迭代 | 逐步构建 journey |
| **Dashboard 多组件** | 多次独立 HTTP 请求 | 浏览器并行 | 各组件独立处理 |

---

## 八、关键代码位置索引

| 功能模块 | 文件路径 | 关键函数/行号 |
|---------|---------|--------------|
| 前端漏斗组件 | `assets/js/dashboard/extra/funnel.js` | `fetchFunnel()` L149, useEffect L57 |
| Dashboard 状态 | `assets/js/dashboard/dashboard-state.ts` | `DashboardState` type L34 |
| API 参数序列化 | `assets/js/dashboard/api.ts` | `dashboardStateToParams()` L60 |
| 漏斗配置 LiveView | `extra/lib/plausible_web/live/funnel_settings.ex` | `mount/3` L11, 事件处理 L112+ |
| 漏斗数据 API | `lib/plausible_web/controllers/api/stats_controller.ex` | `funnel/2` L261, `validate_funnel_query/1` L297 |
| 漏斗查询核心 | `extra/lib/plausible/stats/funnel.ex` | `funnel/3` L19, `funnel_query/2` L68, `backfill_steps/2` L121 |
| 漏斗步骤常量 | `lib/plausible/funnel/const.ex` | `min_steps()`, `max_steps()` (2-8) |
| 基础查询构建 | `lib/plausible/stats/base.ex` | `base_event_query/1` L7 |
| 查询构建器 | `lib/plausible/stats/query_builder.ex` | `build/3` L29 |
| ClickHouse 并发 | `lib/plausible/clickhouse_repo.ex` | `parallel_tasks/2` L18, `prepare_query/3` L47 |
| Exploration 功能 | `extra/lib/plausible/stats/exploration.ex` | `interesting_funnel/2` L146, `do_build_journey/7` L181 |
| 漏斗存储 | `extra/lib/plausible/funnels.ex` | `create/4`, `get/2`, `list/1` |

---

## 九、总结与关键点

### 9.1 过滤器共享机制

1. **单一数据源**: 所有组件共享同一个 `DashboardState.filters`
2. **自动传递**: 组件调用 API 时只需传递 `dashboardState` 对象
3. **统一验证**: 后端 `QueryBuilder` 统一验证和处理所有过滤器
4. **漏斗特殊限制**: 
   - 漏斗查询不允许 `event:goal` 和 `event:page` 过滤
   - **不支持实时模式** (`realtime` period)

### 9.2 步骤数量限制

| 功能 | 最小步骤 | 最大步骤 | 说明 |
|-----|---------|---------|------|
| **标准漏斗** | 2 | 8 | `Funnel.min_steps()` 到 `Funnel.max_steps()` |
| **Exploration** | 1 | 20 | `@max_steps 20`，`interesting_funnel` 默认 6 步 |

### 9.3 并发查询机制澄清

**重要区别**：

| 类型 | 说明 | 示例场景 |
|-----|------|---------|
| **后端并发查询** | 使用 `parallel_tasks/2` 或 `Task.async_stream` | CSV 导出、多站点统计 |
| **前端并行请求** | 浏览器同时发起多个 HTTP 请求 | Dashboard 多组件加载 |
| **迭代式查询** | 顺序执行，每次依赖前一次结果 | Exploration 功能 |
| **单次查询** | 一个 SQL 完成所有工作 | 标准漏斗查询 (`windowFunnel`) |

**漏斗查询本身**：
- 是**单次 `windowFunnel` 调用**，不是并发查询
- 利用 ClickHouse 的 `windowFunnel` 函数一次计算所有步骤
- 结果通过 `backfill_steps/2` 回填，不是通过并行查询拼装

**真正使用后端并发的场景**：
1. **CSV 多维度导出**：按 key 匹配拼装
2. **多站点用量统计**：累加拼装
3. **多站点概览**：直接返回列表
4. **账单周期统计**：转为 Map

### 9.4 windowFunnel 工作原理

1. **用户分组**: 按 `user_id` 分组，每组独立分析
2. **时间顺序**: 按 `timestamp` 排序扫描事件
3. **步骤匹配**: 依次检查每个步骤的条件是否满足
4. **严格模式**: `strict_order` 要求步骤间不能有其他步骤的事件
5. **时间窗口**: 24小时内完成的步骤才算有效
6. **结果含义**: 返回值是用户达到的**最远**步骤索引

### 9.5 结果回填逻辑

```
原始 ClickHouse 结果 (windowFunnel 返回):
{0: 100, 1: 200, 2: 150, 3: 50}
  │      │      │      │
  │      │      │      └── 达到步骤3 (完成所有步骤): 50人
  │      │      └───────── 达到步骤2但未到3: 150人
  │      └──────────────── 达到步骤1但未到2: 200人
  └─────────────────────── 未进入任何步骤: 100人

回填后 (累加逻辑):
步骤1 访客数 = 200 + 150 + 50 = 400人 (所有至少进入步骤1的)
步骤2 访客数 = 150 + 50 = 200人
步骤3 访客数 = 50人
```

### 9.6 性能优化考虑

1. **采样机制**: 企业版支持 `Sampling.add_query_hint()` 处理大数据量
2. **合并站点**: 支持 `consolidated_site_ids` 查询跨站点数据
3. **导入数据**: `with_imported` 参数控制是否包含历史导入数据
4. **查询追踪**: `log_comment` 用于在 ClickHouse 日志中追踪查询来源

---

*文档生成时间: 2026-05-05*
*基于 Plausible 代码库分析*
