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

### 5.1 完整时序流程图

```
用户操作 → 参数写回 → 页面状态重算 → 查询触发 → 服务端解析 → 数据返回
   ↓           ↓              ↓              ↓            ↓            ↓
点击菜单   构建search      URL变化         queryKey     解析参数    执行SQL
或快捷键   更新URL         解析参数         变化         构建查询    返回结果
```

### 5.2 步骤1：用户操作触发

**方式1：通过菜单选择**

用户点击时间范围菜单中的选项，如 "Last 7 Days" 或 "Month to Date"。

**核心组件**: `DashboardPeriodMenu`

```typescript
// assets/js/dashboard/nav-menu/query-periods/dashboard-period-menu.tsx:200-216
<AppNavigationLink
  key={label}
  data-selected={isActive({ site, dashboardState })}
  className={linkClassName}
  search={search}
  onClick={onEvent && ((e) => onEvent(e))}
>
  {label}
  {!!keyboardKey && (
    <KeybindHint>{keyboardKey}</KeybindHint>
  )}
</AppNavigationLink>
```

**方式2：通过键盘快捷键**

项目支持键盘快捷键快速切换时间范围，如 `D` 切换到 "Today"，`W` 切换到 "Last 7 Days"。

```typescript
// assets/js/dashboard/nav-menu/query-periods/dashboard-period-menu.tsx:41-81
function DashboardPeriodMenuKeybinds({
  closeDropdown,
  groups
}: {
  groups: LinkItem[][]
  closeDropdown: () => void
}) {
  const dashboardRouteMatch = useMatch(rootRoute.path)
  const navigate = useAppNavigate()

  if (!dashboardRouteMatch) {
    return null
  }
  return (
    <>
      {groups.flatMap((group) =>
        group
          .filter(([[_name, keyboardKey]]) => !!keyboardKey)
          .map(([[_name, keyboardKey], { search, onEvent }]) => (
            <Keybind
              key={keyboardKey}
              keyboardKey={keyboardKey}
              type="keydown"
              handler={(e) => {
                if (typeof search === 'function') {
                  navigate({ search })  // 直接触发导航
                }
                if (typeof onEvent === 'function') {
                  onEvent(e)
                } else {
                  closeDropdown()
                }
              }}
              // ...
            />
          ))
      )}
    </>
  )
}
```

**方式3：选择自定义日期范围**

用户通过日历组件选择自定义日期范围。

```typescript
// assets/js/dashboard/nav-menu/query-periods/dashboard-period-menu.tsx:244-259
<DateRangeCalendar
  id="calendar"
  onCloseWithSelection={(selection) => {
    navigate({
      search: getSearchToApplyCustomDates(selection)
    })
    closeDropdown()
  }}
  // ...
/>
```

### 5.3 步骤2：参数写回 URL

当用户选择时间范围后，系统会构建新的搜索参数并更新 URL。

**1. 构建搜索参数**

每种时间范围选项都有对应的 `search` 函数，用于生成新的 URL 参数。

```typescript
// assets/js/dashboard/dashboard-time-periods.ts:326-446
export const getDatePeriodGroups = ({
  site,
  onEvent,
  extraItemsInLastGroup = [],
  extraGroups = []
}: {
  // ...
}): LinkItem[][] => {
  const groups: LinkItem[][] = [
    [
      [
        ['Today', 'D'],
        {
          search: (s) => ({
            ...s,
            ...clearedDateSearch,
            period: DashboardPeriod.day,
            date: formatISO(now(site.offset)),
            keybindHint: 'D'
          }),
          isActive: ({ dashboardState }) =>
            dashboardState.period === DashboardPeriod.day &&
            isSameDate(dashboardState.date, now(site.offset)),
          onEvent
        }
      ],
      [
        ['Last 7 Days', 'W'],
        {
          search: (s) => ({
            ...s,
            ...clearedDateSearch,
            period: DashboardPeriod['7d'],
            keybindHint: 'W'
          }),
          isActive: ({ dashboardState }) =>
            dashboardState.period === DashboardPeriod['7d'],
          onEvent
        }
      ],
      // ... 更多时间范围选项
    ]
  ]
  // ...
}
```

**自定义日期范围参数构建**：

```typescript
// assets/js/dashboard/dashboard-time-periods.ts:249-277
export const getSearchToApplyCustomDates = ([selectionStart, selectionEnd]: [
  Date,
  Date
]): AppNavigationTarget['search'] => {
  const [from, to] = [
    parseNaiveDate(selectionStart),
    parseNaiveDate(selectionEnd)
  ]
  const singleDaySelected = from.isSame(to, 'day')

  if (singleDaySelected) {
    return (search) => ({
      ...search,
      ...clearedDateSearch,
      period: DashboardPeriod.day,
      date: formatISO(from),
      keybindHint: 'C'
    })
  }

  return (search) => ({
    ...search,
    ...clearedDateSearch,
    period: DashboardPeriod.custom,
    from: formatISO(from),
    to: formatISO(to),
    keybindHint: 'C'
  })
}
```

**2. 执行导航更新 URL**

使用 `useAppNavigate` 钩子执行导航，将新参数写回 URL。

```typescript
// assets/js/dashboard/navigation/use-app-navigate.tsx:71-86
export const useAppNavigate = () => {
  const _navigate = useNavigate()
  const getToOptions = useGetNavigateOptions()
  const navigate = useCallback(
    ({
      path,
      params,
      search,
      ...options
    }: AppNavigationTarget & NavigateOptions) => {
      return _navigate(getToOptions({ path, params, search }), options)
    },
    [getToOptions, _navigate]
  )
  return navigate
}
```

**3. 构建导航选项**

`getNavigateToOptions` 函数负责解析当前 URL 并应用新的搜索参数。

```typescript
// assets/js/dashboard/navigation/use-app-navigate.tsx:34-45
const getNavigateToOptions = (
  currentSearchString: string,
  { path, params, search }: AppNavigationTarget
) => {
  const searchRecord = parseSearch(currentSearchString)  // 解析当前 URL
  const updatedSearchRecord = search && search(searchRecord)  // 应用新的搜索函数
  const updatedPath = path && generatePath(path, params)
  return {
    pathname: updatedPath,
    search: updatedSearchRecord && stringifySearch(updatedSearchRecord)  // 序列化新参数
  }
}
```

**4. 序列化 URL 参数**

`stringifySearch` 函数将搜索记录序列化为 URL 参数字符串。

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

**示例 URL 变化**：
- 从 `?period=7d` 切换到 "Last 28 Days" → `?period=28d`
- 选择自定义日期范围（2024-01-01 到 2024-01-15）→ `?period=custom&from=2024-01-01&to=2024-01-15`

### 5.4 步骤3：页面状态重算

URL 更新后，React 会检测到变化并触发状态重算。

**1. 检测 URL 变化**

```typescript
// assets/js/dashboard/dashboard-state-context.tsx:47
const location = useLocation()  // React Router 钩子，URL 变化时触发重渲染
```

**2. 解析新的 URL 参数**

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

**3. 解析逻辑详情**

`parseSearch` 函数将 URL 参数字符串解析为对象：

```typescript
// assets/js/dashboard/util/url-search-params.ts:56-105
export function parseSearch(searchString: string): Record<string, unknown> {
  const searchRecord: Record<string, string | boolean> = {}
  const filters: Filter[] = []
  const labels: FilterClauseLabels = {}

  const normalizedSearchString = normalizeSearchString(searchString)

  if (!normalizedSearchString.length) {
    return searchRecord
  }

  const meaningfulParams = normalizedSearchString
    .split('&')
    .filter((i) => i.length > 0)

  for (const param of meaningfulParams) {
    const [key, rawValue = ''] = param.split('=')
    switch (key) {
      case FILTER_URL_PARAM_NAME: {  // 处理过滤器参数
        const filter = parseFilter(rawValue)
        if (filter.length === 3 && filter[2].length) {
          filters.push(filter)
        }
        break
      }
      case LABEL_URL_PARAM_NAME: {  // 处理标签参数
        const [labelKey, labelValue] = parseLabelsEntry(rawValue)
        if (labelKey.length && labelValue.length) {
          labels[labelKey] = labelValue
        }
        break
      }
      case '': {
        break
      }
      default: {  // 处理其他参数（如 period, date, from, to 等）
        const parsedValue = parseSimpleSearchEntry(rawValue)
        if (parsedValue !== null) {
          searchRecord[decodeURIComponent(key)] = parsedValue
        }
      }
    }
  }

  return {
    ...searchRecord,
    ...(filters.length && { filters }),
    ...(Object.keys(labels).length && { labels })
  }
}
```

**4. 重新计算 dashboardState**

当 `location.search` 变化时，`useMemo` 的依赖项变化，触发 `dashboardState` 重新计算。

```typescript
// assets/js/dashboard/dashboard-state-context.tsx:68-133
const dashboardState = useMemo(() => {
  const defaultValues = dashboardStateDefaultValue
  const storedValues = getSavedTimePreferencesFromStorage({ site })
  
  // 计算时间设置（优先级：URL > 存储 > 默认值）
  const timeSettings = getDashboardTimeSettings({
    site,
    searchValues: { period, comparison, match_day_of_week },
    storedValues,
    defaultValues,
    segmentIsExpanded: !!expandedSegment
  })

  const filters = Array.isArray(rawFilters)
    ? postProcessFilters(rawFilters as Filter[])
    : defaultValues.filters

  const resolvedFilters = resolveFilters(filters, segmentsContext.segments)

  return {
    ...timeSettings,
    compare_from:
      typeof compare_from === 'string' && compare_from.length
        ? dayjs.utc(compare_from)
        : defaultValues.compare_from,
    compare_to:
      typeof compare_to === 'string' && compare_to.length
        ? dayjs.utc(compare_to)
        : defaultValues.compare_to,
    date:
      typeof date === 'string' && date.length
        ? dayjs.utc(date)
        : now(site.offset).startOf('day'),
    from:
      typeof from === 'string' && from.length
        ? dayjs.utc(from)
        : timeSettings.period === DashboardPeriod.custom
          ? yesterday(site.offset)
          : defaultValues.from,
    to:
      typeof to === 'string' && to.length
        ? dayjs.utc(to)
        : timeSettings.period === DashboardPeriod.custom
          ? now(site.offset)
          : defaultValues.to,
    with_imported: [true, false].includes(with_imported as boolean)
      ? (with_imported as boolean)
      : defaultValues.with_imported,
    filters,
    resolvedFilters,
    labels: (labels as FilterClauseLabels) || defaultValues.labels
  }
}, [
  compare_from,
  compare_to,
  comparison,
  date,
  rawFilters,
  from,
  labels,
  match_day_of_week,
  period,
  to,
  with_imported,
  site,
  expandedSegment,
  segmentsContext.segments
])
```

**5. 时间设置计算优先级**

`getDashboardTimeSettings` 函数实现了三层优先级机制：

```typescript
// assets/js/dashboard/dashboard-time-periods.ts:616-670
export function getDashboardTimeSettings({
  site,
  searchValues,
  storedValues,
  defaultValues,
  segmentIsExpanded
}: {
  // ...
}): Pick<DashboardState, 'period' | 'comparison' | 'match_day_of_week'> {
  let period: DashboardPeriod
  // 优先级：URL 参数 > 存储值 > 特殊情况 > 默认值
  if (isValidPeriod(searchValues.period)) {
    period = searchValues.period
  } else if (isValidPeriod(storedValues.period)) {
    period = storedValues.period
  } else if (isTodayOrYesterday(site.nativeStatsBegin)) {
    period = DashboardPeriod.day
  } else {
    period = defaultValues.period
  }

  let comparison: ComparisonMode | null
  // 检查是否禁用比较（实时模式或分段展开时）
  if (isComparisonForbidden({ period, segmentIsExpanded })) {
    comparison = null
  } else {
    // 优先级：URL 参数 > 存储值
    comparison = isValidComparison(searchValues.comparison)
      ? searchValues.comparison
      : storedValues.comparison

    if (!isComparisonEnabled(comparison)) {
      comparison = null
    }
  }

  // match_day_of_week 优先级：URL > 存储 > 默认
  const match_day_of_week = isValidMatchDayOfWeek(
    searchValues.match_day_of_week
  )
    ? (searchValues.match_day_of_week as boolean)
    : isValidMatchDayOfWeek(storedValues.match_day_of_week)
      ? (storedValues.match_day_of_week as boolean)
      : defaultValues.match_day_of_week

  return {
    period,
    comparison,
    match_day_of_week
  }
}
```

### 5.5 步骤4：查询触发

`dashboardState` 更新后，由于查询键（queryKey）依赖 `dashboardState`，`@tanstack/react-query` 会自动触发新的数据获取。

**1. 查询键依赖机制**

```typescript
// assets/js/dashboard/stats/graph/visitor-graph.tsx:46-61
const topStatsQuery = useQuery({
  queryKey: ['top-stats', { dashboardState }] as const,  // 查询键包含 dashboardState
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
  ] as const,  // 查询键包含 dashboardState
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

**2. 触发查询执行**

当 `queryKey` 变化时，`useQuery` 会：
1. 检查缓存中是否有对应的数据
2. 如果没有或已过期，执行 `queryFn`
3. 更新缓存并触发组件重渲染

**3. 构建 API 查询参数**

`queryFn` 执行时，会调用 `createStatsQuery` 将 `dashboardState` 转换为 API 所需的查询参数。

```typescript
// assets/js/dashboard/stats-query.ts:40-61
export function createStatsQuery(
  dashboardState: DashboardState,
  reportParams: ReportParams
): StatsQuery {
  return {
    date_range: createDateRange(dashboardState),  // 构建时间范围
    relative_date: dashboardState.date ? formatISO(dashboardState.date) : null,
    dimensions: reportParams.dimensions || [],
    metrics: reportParams.metrics,
    filters: remapToApiFilters(dashboardState.filters),
    include: {
      imports: dashboardState.with_imported,
      imports_meta: reportParams.include?.imports_meta || false,
      time_labels: reportParams.include?.time_labels || false,
      partial_time_labels: reportParams.include?.partial_time_labels || false,
      compare: createIncludeCompare(dashboardState),  // 构建比较模式
      compare_match_day_of_week: dashboardState.match_day_of_week,
      empty_metrics: reportParams.include?.empty_metrics || false,
      present_index: reportParams.include?.present_index || false
    }
  }
}
```

**4. 时间范围转换**

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

**5. 比较模式转换**

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

**6. 发送 API 请求**

最终通过 `api.stats` 发送请求到后端。

```typescript
// assets/js/dashboard/stats/graph/fetch-main-graph.ts:9-36
export function fetchMainGraph(
  site: PlausibleSite,
  dashboardState: DashboardState,
  metric: Metric,
  interval: string
): Promise<MainGraphResponse> {
  const metricToQuery =
    metric === 'conversion_rate' ? 'group_conversion_rate' : metric

  const reportParams: ReportParams = {
    metrics: [metricToQuery],
    dimensions: [`time:${interval}`],
    include: {
      time_labels: true,
      partial_time_labels: true,
      empty_metrics: true,
      present_index: true
    }
  }

  const statsQuery = createStatsQuery(dashboardState, reportParams)

  // 实时模式特殊处理
  if (isRealTimeDashboard(dashboardState)) {
    statsQuery.date_range = DashboardPeriod.realtime_30m
  }

  return api.stats(site, statsQuery)
}
```

### 5.6 步骤5：服务端解析与处理

**1. API 端点接收请求**

请求发送到后端 API 端点，通常是 `/api/stats/:domain/` 路径。

**2. 解析查询参数**

后端使用 `Dashboard.QueryParser.parse` 解析前端传来的查询参数。

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

**3. 解析时间范围**

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

**4. 解析相对日期**

```elixir
# lib/plausible/stats/dashboard/query_parser.ex:51-62
defp parse_relative_date(%{"relative_date" => date}) when is_binary(date) do
  case Date.from_iso8601(date) do
    {:ok, date} ->
      {:ok, date}

    _ ->
      {:error,
       %QueryError{code: :invalid_relative_date, message: "Failed to convert '#{date}' to date"}}
  end
end

defp parse_relative_date(_), do: {:ok, nil}
```

**5. 解析 include 选项（比较模式等）**

```elixir
# lib/plausible/stats/dashboard/query_parser.ex:76-122
defp parse_include(params) do
  with {:ok, compare} <- parse_include_compare(params["include"]) do
    {:ok,
     %QueryInclude{
       imports: params["include"]["imports"] == true,
       imports_meta: params["include"]["imports_meta"] == true,
       compare: compare,
       compare_match_day_of_week: params["include"]["compare_match_day_of_week"] == true,
       time_labels: params["include"]["time_labels"] == true,
       partial_time_labels: params["include"]["partial_time_labels"] == true,
       present_index: params["include"]["present_index"] == true,
       empty_metrics: params["include"]["empty_metrics"] == true,
       trim_relative_date_range: true,
       drop_unavailable_time_on_page: true,
       drop_unavailable_revenue_metrics: true
     }}
  end
end

defp parse_include_compare(%{"compare" => compare})
     when compare in @valid_comparison_shorthand_keys do
  {:ok, @valid_comparison_shorthands[compare]}
end

defp parse_include_compare(%{"compare" => [from, to] = compare})
     when is_binary(from) and is_binary(to) do
  case ApiQueryParser.parse_date_strings(from, to) do
    {:ok, compare} ->
      {:ok, compare}

    {:error, _} ->
      {:error,
       %QueryError{
         code: :invalid_include,
         message: "Invalid include.compare '#{inspect(compare)}'"
       }}
  end
end
```

**6. 构建 DateTimeRange**

解析后的参数用于构建 `DateTimeRange` 结构体。

```elixir
# lib/plausible/stats/datetime_range.ex:23-58
defmodule Plausible.Stats.DateTimeRange do
  @enforce_keys [:first, :last]
  defstruct [:first, :last]

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

  def new!(%DateTime{} = first, %DateTime{} = last) do
    first = DateTime.truncate(first, :second)
    last = DateTime.truncate(last, :second)

    if DateTime.before?(first, last) do
      %__MODULE__{first: first, last: last}
    else
      %__MODULE__{first: last, last: first}
    end
  end
end
```

**7. 生成 SQL 查询**

基于 `DateTimeRange` 和其他参数，后端构建 SQL 查询。

**8. 执行查询并返回结果**

查询执行后，结果被格式化并返回给前端。

### 5.7 步骤6：数据返回与界面更新

**1. 前端接收响应**

`@tanstack/react-query` 自动处理响应，更新缓存并触发组件重渲染。

**2. 图表组件重渲染**

`VisitorGraph` 组件检测到数据变化，重新渲染图表。

```typescript
// assets/js/dashboard/stats/graph/visitor-graph.tsx:213-254
return (
  <div className="col-span-full relative w-full bg-white rounded-md shadow-sm dark:bg-gray-900">
    <>
      <div
        id="top-stats-container"
        className="flex flex-wrap relative"
        ref={topStatsBoundary}
      >
        {topStatsQuery.data ? (
          <TopStats
            data={topStatsQuery.data}
            selectedMetric={selectedMetric}
            onMetricClick={onMetricClick}
            tooltipBoundary={topStatsBoundary.current}
          />
        ) : (
          // 加载状态
          <div style={{ height: `${heightPx}px` }}></div>
        )}
      </div>
      <div className="relative flex flex-col pl-3 pr-4">
        <MainGraphContainer ref={mainGraphContainer}>
          {!!mainGraphQuery.data && !!width && (
            <>
              {!showGraphLoader && (
                <MainGraph width={width} data={mainGraphQuery.data} />
              )}
              {showGraphLoader && <Loader />}
            </>
          )}
        </MainGraphContainer>
      </div>
    </>
    {(!(topStatsQuery.data && mainGraphQuery.data) || showFullLoader) && (
      <Loader />
    )}
  </div>
)
```

### 5.8 实时更新机制

对于实时时间范围（`period: realtime`），项目实现了特殊的定时更新机制。

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

**静默更新机制**：

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

### 5.9 时序总结表

| 阶段 | 关键组件/函数 | 核心操作 | 输入 | 输出 |
|------|--------------|----------|------|------|
| **用户操作** | `DashboardPeriodMenu` 或 `Keybind` | 用户选择时间范围或按快捷键 | 无 | 触发导航 |
| **参数写回** | `useAppNavigate` + `stringifySearch` | 构建新搜索参数并更新 URL | `search` 函数 | 新的 URL（如 `?period=28d`） |
| **状态重算** | `DashboardStateContextProvider` | 解析 URL，重新计算 `dashboardState` | `location.search` | 新的 `dashboardState` 对象 |
| **查询触发** | `useQuery` + `createStatsQuery` | 检测查询键变化，构建 API 参数 | `dashboardState` | API 请求参数 |
| **服务端解析** | `Dashboard.QueryParser.parse` | 解析请求参数，构建 `DateTimeRange` | HTTP 请求体 | `ParsedQueryParams` 结构体 |
| **数据返回** | `@tanstack/react-query` | 接收响应，更新缓存 | API 响应 | 更新后的 `query.data` |

### 5.10 关键数据流示例

假设用户从 "Last 7 Days" 切换到 "Last 28 Days"：

1. **URL 变化**：`?period=7d` → `?period=28d`
2. **dashboardState 变化**：
   ```typescript
   // 之前
   { period: '7d', date: null, from: null, to: null, ... }
   
   // 之后
   { period: '28d', date: null, from: null, to: null, ... }
   ```
3. **查询键变化**：
   ```typescript
   // 之前
   ['top-stats', { dashboardState: { period: '7d', ... } }]
   
   // 之后
   ['top-stats', { dashboardState: { period: '28d', ... } }]
   ```
4. **API 请求参数**：
   ```typescript
   {
     date_range: '28d',
     relative_date: null,
     // ... 其他参数
   }
   ```
5. **后端查询**：查询过去 28 天的数据

## 6. 快速连续切换时间范围：旧请求取消与结果覆盖防护

当用户快速连续切换时间范围时（例如，从 "Last 7 Days" 快速点击到 "Last 28 Days" 再到 "Last 91 Days"），系统需要处理两个关键问题：
1. **旧请求取消**：防止已经不需要的请求继续执行，浪费资源
2. **结果覆盖防护**：防止旧请求的响应在新请求之后到达，导致显示错误的数据

### 6.1 双层请求取消机制

项目实现了双层请求取消机制：**API 层的 AbortController** 和 **React Query 层的自动取消**。

#### 6.1.1 API 层：全局 AbortController

**核心实现**：

```typescript
// assets/js/dashboard/api.ts:9-51
let abortController = new AbortController()
let SHARED_LINK_AUTH: null | string = null

// ...

export function cancelAll() {
  abortController.abort()  // 取消所有进行中的请求
  abortController = new AbortController()  // 创建新的 AbortController 供后续请求使用
}
```

**所有 API 请求都使用同一个 AbortController 的 signal**：

```typescript
// assets/js/dashboard/api.ts:122-140
export async function stats(site: PlausibleSite, statsQuery: StatsQuery) {
  const sharedLinkParams = getSharedLinkSearchParams()
  const queryString = sharedLinkParams.auth
    ? new URLSearchParams(sharedLinkParams).toString()
    : ''
  const path = url.apiPath(site, '/query')
  const response = await fetch(queryString ? `${path}?${queryString}` : path, {
    method: 'POST',
    signal: abortController.signal,  // 使用全局 AbortController 的 signal
    headers: {
      ...getHeaders(),
      'Content-Type': 'application/json',
      Accept: 'application/json'
    },
    body: JSON.stringify(statsQuery)
  })

  return handleApiResponse(response)
}
```

**其他 API 方法也使用相同的 signal**：

```typescript
// assets/js/dashboard/api.ts:142-157
export async function get(
  url: string,
  dashboardState?: DashboardState,
  ...extraQueryParams: unknown[]
) {
  const queryString = dashboardState
    ? dashboardStateToSearchParams(dashboardState, [...extraQueryParams])
    : serializeUrlParams(getSharedLinkSearchParams())

  const response = await fetch(queryString ? `${url}?${queryString}` : url, {
    signal: abortController.signal,  // 同样使用全局 signal
    headers: { ...getHeaders(), Accept: 'application/json' }
  })

  return handleApiResponse(response)
}
```

#### 6.1.2 React Query 层：自动取消机制

`@tanstack/react-query` 内置了请求取消机制，当查询键变化时会自动取消正在进行的请求。

**查询键依赖机制**：

```typescript
// assets/js/dashboard/stats/graph/visitor-graph.tsx:46-95
const topStatsQuery = useQuery({
  queryKey: ['top-stats', { dashboardState }] as const,  // 查询键包含 dashboardState
  queryFn: async ({ queryKey }) => {
    const [_, opts] = queryKey
    return await fetchTopStats(site, opts.dashboardState)
  },
  // ...
})

const mainGraphQuery = useQuery({
  enabled: !!selectedMetric,
  queryKey: [
    'main-graph',
    { dashboardState, metric: selectedMetric!, interval: selectedInterval }
  ] as const,  // 查询键包含 dashboardState
  queryFn: async ({ queryKey }) => {
    // ...
  },
  // ...
})
```

**工作原理**：
1. 当 `dashboardState` 变化时，查询键（`queryKey`）随之变化
2. `@tanstack/react-query` 检测到查询键变化，自动取消正在进行的旧查询
3. 新查询开始执行

#### 6.1.3 取消时机

**1. 组件挂载时**：

```typescript
// assets/js/dashboard/dashboard-state-context.tsx:143-145
useMountedEffect(() => {
  api.cancelAll()  // 组件挂载时取消所有进行中的请求
}, [])
```

**2. 查询键变化时**：
- 由 `@tanstack/react-query` 自动处理
- 当 `queryKey` 变化时，旧查询被取消

### 6.2 结果覆盖防护机制

为了防止旧请求的响应在新请求之后到达，导致显示错误数据，项目使用了以下机制：

#### 6.2.1 placeholderData：保持旧数据显示

**核心实现**：

```typescript
// assets/js/dashboard/stats/graph/visitor-graph.tsx:52
placeholderData: (previousData) => previousData
```

```typescript
// assets/js/dashboard/hooks/api-client.ts:112
placeholderData: (previousData) => previousData
```

**工作原理**：
1. 当新查询正在执行时，`placeholderData` 返回之前的数据
2. 组件继续显示旧数据，直到新查询完成
3. 即使旧请求的响应在新请求之后到达，由于查询键不同，`@tanstack/react-query` 会将其存储在不同的缓存条目下，不会覆盖新查询的结果

#### 6.2.2 查询键隔离

每个查询键对应独立的缓存条目：

```typescript
// 查询键示例
['top-stats', { dashboardState: { period: '7d', ... } }]  // 缓存条目 A
['top-stats', { dashboardState: { period: '28d', ... } }] // 缓存条目 B
['top-stats', { dashboardState: { period: '91d', ... } }] // 缓存条目 C
```

**工作原理**：
1. 每个不同的 `dashboardState` 对应不同的查询键
2. 每个查询键有独立的缓存条目
3. 旧请求的响应会存储在对应的缓存条目下，不会影响新请求的结果
4. 组件只使用当前查询键对应的数据

### 6.3 快速连续切换场景示例

**场景**：用户快速点击 "Last 7 Days" → "Last 28 Days" → "Last 91 Days"

**时序流程**：

```
时间点 T0: 用户点击 "Last 7 Days"
  ↓
- URL 变为 ?period=7d
- dashboardState 更新 { period: '7d' }
- 查询键变为 ['top-stats', { dashboardState: { period: '7d', ... } }]
- 开始执行请求 A (7d 数据)
  ↓

时间点 T1 (请求 A 还在进行中): 用户快速点击 "Last 28 Days"
  ↓
- URL 变为 ?period=28d
- dashboardState 更新 { period: '28d' }
- 查询键变为 ['top-stats', { dashboardState: { period: '28d', ... } }]
- @tanstack/react-query 检测到查询键变化，自动取消请求 A
- 开始执行请求 B (28d 数据)
- 由于 placeholderData，组件继续显示请求 A 之前的数据（或请求 A 已有的部分数据）
  ↓

时间点 T2 (请求 B 还在进行中): 用户快速点击 "Last 91 Days"
  ↓
- URL 变为 ?period=91d
- dashboardState 更新 { period: '91d' }
- 查询键变为 ['top-stats', { dashboardState: { period: '91d', ... } }]
- @tanstack/react-query 检测到查询键变化，自动取消请求 B
- 开始执行请求 C (91d 数据)
- 组件继续显示旧数据（placeholderData）
  ↓

时间点 T3: 请求 C 完成
  ↓
- 缓存更新，查询键 C 对应的数据可用
- 组件显示 91d 的数据
```

**结果覆盖防护**：
- 即使请求 A 或 B 在请求 C 之后"完成"（实际上已被取消），它们的响应会被忽略或存储在不同的缓存条目
- 组件始终使用当前查询键（C）对应的数据
- 用户最终看到的是正确的 91d 数据

## 7. 查询失败后的状态流转

当 API 请求失败时，系统需要处理错误状态，确保用户体验良好，并且有清晰的错误信息展示。

### 7.1 查询状态机

`@tanstack/react-query` 使用状态机管理查询生命周期：

```
┌─────────┐     ┌─────────┐     ┌─────────┐
│  idle   │────▶│ loading │────▶│ success │
└─────────┘     └─────────┘     └─────────┘
                      │
                      ▼
                 ┌─────────┐
                 │  error  │
                 └─────────┘
```

**状态定义**：
- `idle`：查询尚未开始
- `loading`：查询正在执行中
- `success`：查询成功完成
- `error`：查询失败

### 7.2 API 错误处理

#### 7.2.1 ApiError 类

项目定义了自定义的 `ApiError` 类来包装 API 错误：

```typescript
// assets/js/dashboard/api.ts:26-33
export class ApiError extends Error {
  payload: unknown
  constructor(message: string, payload: unknown) {
    super(message)
    this.name = 'ApiError'
    this.payload = payload  // 包含完整的错误 payload
  }
}
```

#### 7.2.2 响应错误处理

```typescript
// assets/js/dashboard/api.ts:109-116
async function handleApiResponse(response: Response) {
  const payload = await response.json()
  if (!response.ok) {
    throw new ApiError(payload.error, payload)  // 抛出 ApiError
  }

  return payload
}
```

### 7.3 组件层错误处理

#### 7.3.1 BreakdownTable 错误展示

在模态框表格组件中，错误状态有明确的 UI 展示：

```typescript
// assets/js/dashboard/stats/modals/breakdown-table.tsx:11-89
export const BreakdownTable = <TListItem extends { name: string }>({
  title,
  // ...
  status,         // QueryStatus: 'idle' | 'loading' | 'error' | 'success'
  error,          // Error | null
  displayError,   // 是否显示错误
  // ...
}: {
  // ...
  status?: QueryStatus
  error?: Error | null
  /** Controls whether the component displays API request errors or ignores them. */
  displayError?: boolean
  // ...
}) => {
  // ...

  return (
    <div className="min-h-[66vh] md:min-h-120 flex flex-col flex-1">
      {/* ... */}
      <div className="flex-1 overflow-auto pr-4 -mr-4">
        {displayError && status === 'error' && <ErrorMessage error={error} />}
        {isPending && <InitialLoadingSpinner />}
        {data && <Table<TListItem> data={data} columns={columns} />}
        {/* ... */}
      </div>
    </div>
  )
}
```

#### 7.3.2 错误消息组件

```typescript
// assets/js/dashboard/stats/modals/breakdown-table.tsx:105-116
const ErrorMessage = ({ error }: { error?: unknown }) => (
  <div className="grid grid-rows-2 text-gray-700 dark:text-gray-300">
    <div className="text-center self-end">
      <RocketIcon />
    </div>
    <div className="text-lg text-center">
      {error
        ? (error as { message: string }).message
        : 'Error loading data. Refresh the page to try again'}
    </div>
  </div>
)
```

#### 7.3.3 搜索输入禁用

当查询失败时，搜索输入会被禁用，防止用户在错误状态下继续操作：

```typescript
// assets/js/dashboard/stats/modals/breakdown-table.tsx:53-63
{!!onSearch && (
  <SearchInput
    searchRef={searchRef}
    onSearch={onSearch}
    className={
      displayError && status === 'error'
        ? '[&_input]:pointer-events-none'  // 禁用输入
        : ''
    }
  />
)}
```

### 7.4 React 组件树错误边界

项目使用 `ErrorBoundary` 捕获 React 组件树中的错误：

```typescript
// assets/js/dashboard/error/error-boundary.tsx:1-29
import React, { ReactNode, ReactElement } from 'react'

type ErrorBoundaryProps = {
  children: ReactNode
  renderFallbackComponent: (props: { error?: unknown }) => ReactElement
}

type ErrorBoundaryState = { error: null | unknown }

export default class ErrorBoundary extends React.Component<
  ErrorBoundaryProps,
  ErrorBoundaryState
> {
  constructor(props: ErrorBoundaryProps) {
    super(props)
    this.state = { error: null }
  }

  static getDerivedStateFromError(error: unknown) {
    return { error }
  }

  render() {
    if (this.state.error) {
      return this.props.renderFallbackComponent({ error: this.state.error })
    }
    return this.props.children
  }
}
```

### 7.5 查询失败后的状态流转示例

**场景**：用户切换到 "Last 91 Days"，但 API 请求失败

**时序流程**：

```
时间点 T0: 用户点击 "Last 91 Days"
  ↓
- URL 变为 ?period=91d
- dashboardState 更新 { period: '91d' }
- 查询键变为 ['top-stats', { dashboardState: { period: '91d', ... } }]
- 查询状态: loading
- 开始执行请求 C
- 组件显示加载状态（Loader）
  ↓

时间点 T1: 请求 C 失败（网络错误或服务器错误）
  ↓
- handleApiResponse 检测到 response.ok 为 false
- 抛出 ApiError
- @tanstack/react-query 捕获错误
- 查询状态: error
  ↓

时间点 T2: 组件处理错误状态
  ↓
- 在 BreakdownTable 等组件中：
  - 如果 displayError 为 true，显示 ErrorMessage 组件
  - 搜索输入被禁用（如果有）
- 错误信息展示给用户："Error loading data. Refresh the page to try again"
  ↓

用户操作: 刷新页面或重新选择时间范围
  ↓
- 如果用户重新选择时间范围：
  - URL 更新
  - dashboardState 更新
  - 查询键变化
  - 查询状态: loading
  - 重新开始查询流程
```

### 7.6 错误恢复机制

**1. 重新选择时间范围**：
- 用户选择不同的时间范围会触发新的查询
- 新查询有独立的查询键和缓存条目
- 错误状态不会影响新查询

**2. 页面刷新**：
- 页面刷新会重新初始化所有状态
- 所有查询重新执行

**3. placeholderData 的缓冲作用**：
- 如果之前有成功的查询，`placeholderData` 会保持旧数据显示
- 用户不会看到完全空白的页面

## 8. 数据缓存与性能优化

### 8.1 查询缓存

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

### 8.2 智能缓存过期策略

根据时间范围类型，缓存有不同的过期时间：

```typescript
// assets/js/dashboard/hooks/api-client.ts:17-21
// define (in ms) when query API responses should become stale
export const CACHE_TTL_REALTIME = REALTIME_UPDATE_TIME_MS  // 实时模式：短 TTL
export const CACHE_TTL_SHORT_ONGOING = 5 * 60 * 1000     // 5 分钟
export const CACHE_TTL_LONG_ONGOING = 60 * 60 * 1000     // 1 小时
export const CACHE_TTL_HISTORICAL = 12 * 60 * 60 * 1000  // 12 小时（历史数据）
```

```typescript
// assets/js/dashboard/hooks/api-client.ts:138-162
export const getStaleTime = (props: DashboardTimeSettings): number => {
  if (
    [DashboardPeriod.realtime, DashboardPeriod.realtime_30m].includes(
      props.period
    )
  ) {
    return CACHE_TTL_REALTIME
  }

  if (isHistoricalPeriod(props)) {
    return CACHE_TTL_HISTORICAL  // 历史数据：12 小时
  }

  const availableIntervals = validIntervals(props)

  if (
    availableIntervals.includes(Interval.day) ||
    availableIntervals.includes(Interval.hour) ||
    availableIntervals.includes(Interval.minute)
  ) {
    return CACHE_TTL_SHORT_ONGOING  // 包含今天的数据：5 分钟
  } else {
    return CACHE_TTL_LONG_ONGOING   // 其他：1 小时
  }
}
```

### 8.3 静默更新

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

## 9. 总结

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