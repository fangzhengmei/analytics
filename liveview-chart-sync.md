# LiveView 中 URL 参数与图表组件的同步机制

## 1. 概述

本报告详细分析了 Plausible Analytics 项目中，前端 React 应用如何通过 URL 参数与图表组件的显示状态保持同步，以及切换时间范围时背后的查询联动机制。

## 2. 前端状态管理与 URL 参数同步

### 2.1 状态管理架构

项目使用 React Context API 进行状态管理，核心组件是 `DashboardStateContextProvider`。

**核心文件**: 
- `assets/js/dashboard/dashboard-state-context.tsx`
- `assets/js/dashboard/dashboard-state.ts`

**状态定义**:
```typescript
// assets/js/dashboard/dashboard-state.ts:34-53
export type DashboardState = {
  period: DashboardPeriod
  comparison: ComparisonMode | null
  match_day_of_week: boolean
  date: Dayjs | null
  from: Dayjs | null
  to: Dayjs | null
  compare_from: Dayjs | null
  compare_to: Dayjs | null
  filters: Filter[]
  resolvedFilters: Filter[]
  labels: FilterClauseLabels
  with_imported: boolean
}
```

### 2.2 URL 参数解析

**核心文件**: `assets/js/dashboard/util/url-search-params.ts`

项目实现了自定义的 URL 参数序列化和解析逻辑，支持向后兼容的多个版本。

**解析流程**:
1. **使用 `useLocation` 钩子获取当前 URL**
   ```typescript
   // assets/js/dashboard/dashboard-state-context.tsx:47
   const location = useLocation()
   ```

2. **解析 URL 搜索参数**
   ```typescript
   // assets/js/dashboard/dashboard-state-context.tsx:66
   const {
     compare_from,
     compare_to,
     comparison,
     date,
     filters: rawFilters,
     from,
     labels,
     match_day_of_week,
     period,
     to,
     with_imported,
     ...otherSearch
   } = useMemo(() => parseSearch(location.search), [location.search])
   ```

3. **构建 `dashboardState` 对象**
   ```typescript
   // assets/js/dashboard/dashboard-state-context.tsx:68-133
   const dashboardState = useMemo(() => {
     const defaultValues = dashboardStateDefaultValue
     const storedValues = getSavedTimePreferencesFromStorage({ site })
     const timeSettings = getDashboardTimeSettings({
       site,
       searchValues: { period, comparison, match_day_of_week },
       storedValues,
       defaultValues,
       segmentIsExpanded: !!expandedSegment
     })
     
     // ... 处理其他状态字段
     
     return {
       ...timeSettings,
       compare_from: typeof compare_from === 'string' && compare_from.length
         ? dayjs.utc(compare_from)
         : defaultValues.compare_from,
       // ... 其他字段
       filters,
       resolvedFilters,
       labels: (labels as FilterClauseLabels) || defaultValues.labels
     }
   }, [/* 依赖项 */])
   ```

### 2.3 URL 参数序列化

**核心函数**: `stringifySearch`

```typescript
// assets/js/dashboard/util/url-search-params.ts:26-50
export function stringifySearch(
  searchRecord: Record<string, null | undefined | number | string | unknown>
): '' | string {
  const { filters, labels, ...rest } = searchRecord ?? {}
  const definedSearchEntries = Object.entries(rest)
    .map(serializeSimpleSearchEntry)
    .filter(isSearchEntryDefined)
    .map(([k, v]) => `${k}=${v}`)

  if (!Array.isArray(filters) || !filters.length) {
    return definedSearchEntries.length
      ? `?${definedSearchEntries.join('&')}`
      : ''
  }

  const serializedFilters = Array.isArray(filters)
    ? filters.map((f) => `${FILTER_URL_PARAM_NAME}=${serializeFilter(f)}`)
    : []

  const serializedLabels = Object.entries(labels ?? {}).map(
    (entry) => `${LABEL_URL_PARAM_NAME}=${serializeLabelsEntry(entry)}`
  )

  return `?${serializedFilters.concat(serializedLabels).concat(definedSearchEntries).join('&')}`
}
```

### 2.4 版本兼容性

项目支持多个版本的 URL 格式，并提供了向后兼容的重定向功能：

- **v1**: 自定义编码模式，如 `?page=/blog`
- **v2**: 使用 jsonurl 库序列化状态
- **当前版本**: 自定义编码，更具可读性

```typescript
// assets/js/dashboard/util/url-search-params.ts:237-267
export function maybeGetLatestReadableSearch(
  searchString: string
): null | string {
  const searchParams = new URLSearchParams(searchString)
  if (isAlreadyRedirected(searchParams)) {
    return null
  }
  const isCurrentVersion = searchParams.get(FILTER_URL_PARAM_NAME)
  if (isCurrentVersion) {
    return null
  }

  const isV2 = v2.isV2(searchParams)
  const isV1 = v1.isV1(searchParams)

  if (isV2) {
    return stringifySearch({
      ...v2.parseSearch(searchString),
      [REDIRECTED_SEARCH_PARAM_NAME]: 'v2'
    })
  }

  if (isV1) {
    return stringifySearch({
      ...v1.parseSearch(searchString),
      [REDIRECTED_SEARCH_PARAM_NAME]: 'v1'
    })
  }

  return null
}
```

## 3. 图表组件数据获取机制

### 3.1 核心图表组件

**核心文件**: `assets/js/dashboard/stats/graph/visitor-graph.tsx`

图表组件使用 `@tanstack/react-query` 进行数据获取和缓存管理。

### 3.2 查询键依赖机制

图表组件通过将 `dashboardState` 包含在查询键中，实现状态变化时的自动重新获取数据：

```typescript
// assets/js/dashboard/stats/graph/visitor-graph.tsx:46-61
const topStatsQuery = useQuery({
  queryKey: ['top-stats', { dashboardState }] as const,
  queryFn: async ({ queryKey }) => {
    const [_, opts] = queryKey
    return await fetchTopStats(site, opts.dashboardState)
  },
  placeholderData: (previousData) => previousData,
  staleTime: ({ queryKey }) => {
    const [_, opts] = queryKey
    return getStaleTime({
      siteTimezoneOffset: site.offset,
      siteStatsBegin: site.statsBegin,
      ...opts.dashboardState
    })
  }
})

// assets/js/dashboard/stats/graph/visitor-graph.tsx:63-95
const mainGraphQuery = useQuery({
  enabled: !!selectedMetric,
  queryKey: [
    'main-graph',
    { dashboardState, metric: selectedMetric!, interval: selectedInterval }
  ] as const,
  queryFn: async ({ queryKey }) => {
    const [_, opts] = queryKey
    const data = await fetchMainGraph(
      site,
      opts.dashboardState,
      opts.metric,
      opts.interval
    )
    return {
      ...data,
      period: opts.dashboardState.period,
      interval: opts.interval
    }
  },
  // ... 其他配置
})
```

### 3.3 数据获取流程

1. **构建查询参数**
   ```typescript
   // assets/js/dashboard/stats-query.ts:40-61
   export function createStatsQuery(
     dashboardState: DashboardState,
     reportParams: ReportParams
   ): StatsQuery {
     return {
       date_range: createDateRange(dashboardState),
       relative_date: dashboardState.date ? formatISO(dashboardState.date) : null,
       dimensions: reportParams.dimensions || [],
       metrics: reportParams.metrics,
       filters: remapToApiFilters(dashboardState.filters),
       include: {
         imports: dashboardState.with_imported,
         imports_meta: reportParams.include?.imports_meta || false,
         time_labels: reportParams.include?.time_labels || false,
         partial_time_labels: reportParams.include?.partial_time_labels || false,
         compare: createIncludeCompare(dashboardState),
         compare_match_day_of_week: dashboardState.match_day_of_week,
         empty_metrics: reportParams.include?.empty_metrics || false,
         present_index: reportParams.include?.present_index || false
       }
     }
   }
   ```

2. **时间范围处理**
   ```typescript
   // assets/js/dashboard/stats-query.ts:63-69
   function createDateRange(dashboardState: DashboardState): DateRange {
     if (dashboardState.period === DashboardPeriod.custom) {
       return [formatISO(dashboardState.from), formatISO(dashboardState.to)]
     } else {
       return dashboardState.period
     }
   }
   ```

3. **比较模式处理**
   ```typescript
   // assets/js/dashboard/stats-query.ts:71-88
   function createIncludeCompare(dashboardState: DashboardState) {
     switch (dashboardState.comparison) {
       case ComparisonMode.custom:
         return [
           formatISO(dashboardState.compare_from),
           formatISO(dashboardState.compare_to)
         ]
       case ComparisonMode.previous_period:
         return ComparisonMode.previous_period
       case ComparisonMode.year_over_year:
         return ComparisonMode.year_over_year
       default:
         return null
     }
   }
   ```

## 4. 后端查询处理机制

### 4.1 查询参数解析

**核心文件**: 
- `lib/plausible/stats/dashboard/query_parser.ex`
- `lib/plausible/stats/parsed_query_params.ex`

**解析流程**:
1. **解析 URL 参数**
   ```elixir
   # lib/plausible/stats/dashboard/query_parser.ex:17-36
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
          skip_goal_existence_check: true,
          now: Keyword.get(opts, :now)
        })}
     end
   end
   ```

2. **时间范围解析**
   ```elixir
   # lib/plausible/stats/dashboard/query_parser.ex:38-49
   defp parse_input_date_range(%{"date_range" => date_range}) do
     case date_range do
       "realtime" -> {:ok, :realtime}
       "realtime_30m" -> {:ok, :realtime_30m}
       date_range -> ApiQueryParser.parse_input_date_range(date_range)
     end
   end
   ```

### 4.2 日期时间范围处理

**核心文件**: `lib/plausible/stats/datetime_range.ex`

```elixir
# lib/plausible/stats/datetime_range.ex:8-78
defmodule Plausible.Stats.DateTimeRange do
  @enforce_keys [:first, :last]
  defstruct [:first, :last]

  @type t() :: %__MODULE__{
          first: %DateTime{},
          last: %DateTime{}
        }

  @doc """
  Creates a `DateTimeRange` struct from the given `%Date{}` structs.
  The first datetime will become the first date at 00:00:00, and the last datetime
  will become the last date at 23:59:59.
  """
  def new!(%Date{} = first, last, timezone) do
    first =
      case DateTime.new(first, ~T[00:00:00], timezone) do
        {:ok, datetime} -> datetime
        {:gap, _just_before, just_after} -> just_after
        {:ambiguous, _first_datetime, second_datetime} -> second_datetime
      end

    new!(first, last, timezone)
  end

  # ... 其他构造函数和方法
end
```

### 4.3 时间维度处理

**核心文件**: `lib/plausible/stats/time.ex`

时间维度支持多种粒度：
- `time:month` - 按月
- `time:week` - 按周
- `time:day` - 按天
- `time:hour` - 按小时
- `time:minute` - 按分钟

**时间标签生成**:
```elixir
# lib/plausible/stats/time.ex:48-121
@doc """
Returns list of time bucket labels for the given query.
"""
def time_labels(query) do
  time_labels_for_dimension(time_dimension(query), query)
end

defp time_labels_for_dimension("time:month", query) do
  date_range = Query.date_range(query)
  n_buckets = Plausible.Times.diff(
    date_range.last,
    Date.beginning_of_month(date_range.first),
    :month
  )
  Enum.map(n_buckets..0//-1, fn shift ->
    date_range.last
    |> Date.beginning_of_month()
    |> Date.shift(month: -shift)
    |> format_datetime()
  end)
end

# ... 其他时间维度的标签生成函数
```

## 5. 时间范围切换时的查询联动机制

### 5.1 前端触发机制

当用户切换时间范围时，会触发以下流程：

1. **更新 URL 参数**
   - 当用户选择不同的时间范围（如从 "7天" 切换到 "30天"），前端会更新 URL 参数
   - 使用 React Router 的导航功能更新 URL

2. **URL 变化触发状态更新**
   - `useLocation` 钩子检测到 URL 变化
   - `parseSearch` 重新解析 URL 参数
   - `dashboardState` 依赖项变化，触发重新计算

3. **查询键变化触发数据重新获取**
   - `dashboardState` 变化导致 `top-stats` 和 `main-graph` 查询键变化
   - `@tanstack/react-query` 自动触发新的数据获取请求

### 5.2 后端查询联动

1. **接收新的查询参数**
   - API 端点接收包含新时间范围的查询参数
   - `Dashboard.QueryParser.parse` 解析新参数

2. **构建新的日期时间范围**
   ```elixir
   # 基于 period 和相对日期构建 DateTimeRange
   ```

3. **生成新的 SQL 查询**
   - 使用新的时间范围构建 SQL 查询
   - 根据时间维度（如 day, hour）确定分组粒度

4. **执行查询并返回结果**
   - 执行新的数据库查询
   - 返回包含新时间范围数据的响应

### 5.3 实时更新机制

对于实时时间范围（`period: realtime`），项目实现了特殊的更新机制：

```typescript
// assets/js/dashboard/stats/graph/visitor-graph.tsx:155-178
useEffect(() => {
  const onTick = () => {
    setIsRealtimeSilentUpdate({ topStats: true, mainGraph: true })
    queryClient.invalidateQueries({
      predicate: ({ queryKey }) => {
        const realtimeTopStatsOrMainGraphQuery =
          ['top-stats', 'main-graph'].includes(queryKey[0] as string) &&
          typeof queryKey[1] === 'object' &&
          (queryKey[1] as { dashboardState?: DashboardState })?.dashboardState
            ?.period === DashboardPeriod.realtime

        return realtimeTopStatsOrMainGraphQuery
      }
    })
  }

  if (isRealtime) {
    document.addEventListener('tick', onTick)
  }

  return () => {
    document.removeEventListener('tick', onTick)
  }
}, [queryClient, isRealtime])
```

当处于实时模式时，组件会监听 `tick` 事件，触发查询失效，从而自动刷新数据。

## 6. 数据缓存与性能优化

### 6.1 查询缓存

项目使用 `@tanstack/react-query` 进行查询缓存，通过 `staleTime` 配置优化性能：

```typescript
// assets/js/dashboard/stats/graph/visitor-graph.tsx:52-61
staleTime: ({ queryKey }) => {
  const [_, opts] = queryKey
  return getStaleTime({
    siteTimezoneOffset: site.offset,
    siteStatsBegin: site.statsBegin,
    ...opts.dashboardState
  })
}
```

### 6.2 静默更新

对于实时数据更新，项目实现了静默更新机制，避免加载指示器闪烁：

```typescript
// assets/js/dashboard/stats/graph/visitor-graph.tsx:117-144
const [isRealtimeSilentUpdate, setIsRealtimeSilentUpdate] = useState({
  topStats: false,
  mainGraph: false
})

// ... 相关 useEffect 逻辑

const showGraphLoader =
  mainGraphQuery.isFetching &&
  mainGraphQuery.isStale &&
  !isRealtimeSilentUpdate.mainGraph &&
  !showFullLoader
```

## 7. 总结

### 7.1 同步机制核心要点

1. **URL 作为单一事实来源**
   - 所有图表显示状态都反映在 URL 参数中
   - 页面刷新或分享链接时能保持相同的显示状态

2. **响应式状态管理**
   - 使用 React Context 和 useMemo 实现状态的响应式更新
   - URL 变化自动触发状态更新

3. **声明式数据获取**
   - 使用 `@tanstack/react-query` 实现声明式数据获取
   - 查询键包含 `dashboardState`，状态变化自动触发重新获取

4. **向后兼容性**
   - 支持多个版本的 URL 格式
   - 提供自动重定向机制，确保旧链接仍然有效

### 7.2 时间范围切换流程

1. **用户操作** → 选择不同的时间范围
2. **URL 更新** → 前端更新 URL 参数
3. **状态同步** → URL 变化触发 `dashboardState` 更新
4. **查询触发** → 查询键变化触发数据重新获取
5. **后端处理** → 解析新参数，构建并执行新查询
6. **结果返回** → 前端接收新数据，更新图表显示

### 7.3 技术亮点

1. **自定义 URL 编码**
   - 实现了简洁、可读的 URL 参数格式
   - 处理了特殊字符和复杂数据结构的序列化

2. **时间维度智能处理**
   - 支持多种时间粒度（年、月、周、天、小时、分钟）
   - 智能生成时间标签和处理部分时间范围

3. **实时更新机制**
   - 特殊处理实时数据的更新逻辑
   - 静默更新避免用户体验中断

4. **性能优化**
   - 智能缓存策略（`staleTime`）
   - 条件渲染加载指示器

## 8. 关键文件索引

| 功能 | 文件路径 |
|------|----------|
| 状态管理 | `assets/js/dashboard/dashboard-state-context.tsx` |
| 状态定义 | `assets/js/dashboard/dashboard-state.ts` |
| URL 参数处理 | `assets/js/dashboard/util/url-search-params.ts` |
| 查询参数构建 | `assets/js/dashboard/stats-query.ts` |
| 图表组件 | `assets/js/dashboard/stats/graph/visitor-graph.tsx` |
| 主图数据获取 | `assets/js/dashboard/stats/graph/fetch-main-graph.ts` |
| 顶部统计获取 | `assets/js/dashboard/stats/graph/fetch-top-stats.ts` |
| 后端查询解析 | `lib/plausible/stats/dashboard/query_parser.ex` |
| 查询参数结构 | `lib/plausible/stats/parsed_query_params.ex` |
| 日期时间范围 | `lib/plausible/stats/datetime_range.ex` |
| 时间工具函数 | `lib/plausible/stats/time.ex` |