# Plausible Analytics 仪表盘查询深度分析（第二轮）

## 1. 缓存键构造与命中路径分析

### 1.1 查询键 (QueryKey) 结构

Plausible 使用 React Query 管理查询缓存，不同组件使用不同的 `queryKey` 结构：

| 组件/端点 | queryKey 结构 | 代码位置 |
|----------|--------------|---------|
| Top Stats | `['top-stats', { dashboardState }]` | `visitor-graph.tsx:47` |
| Main Graph | `['main-graph', { dashboardState, metric, interval }]` | `visitor-graph.tsx:65` |
| Breakdown | `[dimensionKey, statsQuery]` | `api-client.ts:62` |
| Map | `['countries', 'map', dashboardState]` | `map.tsx:70` |
| Sites | `['sites']` | `site-switcher.tsx:98` |

**核心结构分析** (`visitor-graph.tsx:65-88`):
```typescript
queryKey: [
  'main-graph',
  {
    dashboardState,  // 包含 period/date/filters/from/to 等
    metric,          // visitors/pageviews/events 等
    interval         // minute/hour/day/week/month
  }
]
```

### 1.2 站点切换：完整缓存失效

**站点切换机制** (`site-switcher.tsx:63-67`, `dashboard-state-context.tsx:143-145`):

```
┌─────────────────────────────────────────────────────────────┐
│                     站点切换流程                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  用户选择新站点                                              │
│        │                                                    │
│        ▼                                                    │
│  window.location.assign(`/${newDomain}`)  ← 整页刷新         │
│        │                                                    │
│        ▼                                                    │
│  浏览器加载新页面                                            │
│        │                                                    │
│        ▼                                                    │
│  dashboard.tsx 重新执行                                      │
│        │                                                    │
│        ├── QueryClient 新实例 (内存中创建)                    │
│        ├── timer.start() 重新启动 (每 30s tick)              │
│        ├── siteContext 包含新 site.domain                    │
│        └── DashboardStateContextProvider 挂载                │
│             │                                               │
│             └── api.cancelAll()  ← 取消所有进行中的请求       │
│                                                             │
│  结果：100% 缓存失效，无任何查询可命中                        │
└─────────────────────────────────────────────────────────────┘
```

**双重隔离机制**：
1. **整页刷新**：`window.location.assign()` 导致 React Query 内存缓存完全清空
2. **API 路径隔离**：`apiPath()` 函数构造 `/api/stats/:domain/` 路径

```typescript
// util/url.ts:3-8
export function apiPath(site: Pick<PlausibleSite, 'domain'>, path = ''): string {
  return `/api/stats/${encodeURIComponent(site.domain)}${path}/`
}
```

**关键结论**：站点切换 ≈ 全新会话，缓存完全不可复用。

---

### 1.3 过滤器变化：精确键失效

**过滤器存储结构** (`dashboard-state.ts:22-53`):
```typescript
export type DashboardState = {
  // ...
  filters: Filter[]           // 原始过滤器（含 segment ID）
  resolvedFilters: Filter[]   // 解析后的过滤器（segment 展开为子过滤器）
  // ...
}

type Filter = [FilterOperator, FilterKey, FilterClause[]]
// 例: ["is", "country", ["AT", "US"]]
//     ["is_not", "page", ["/login"]]
//     ["has_not_done", ["is", "event:goal", ["Purchase"]]]
```

**前端过滤器映射** (`util/filters.js:197-271`):
```
前端键名            → 后端 API 键名
─────────────────────────────────────
page                → event:page
goal                → event:goal
hostname            → event:hostname
name                → event:name
props:*             → event:props:*
segment             → segment (不变)
country/region/city → visit:country/region/city
source              → visit:source
browser             → visit:browser
...                 → visit:*
```

**缓存失效路径**：

```
用户添加过滤器
      │
      ▼
URL search params 更新 (?filters=[...])
      │
      ▼
useLocation().search 变化
      │
      ▼
DashboardStateContextProvider 重计算
      │
      ├── dashboardState.filters 变化
      └── queryKey[1].dashboardState 变化
            │
            ▼
      React Query 缓存键不匹配
            │
            ▼
      新查询触发，旧数据不再使用
```

**缓存命中场景**：
- 移除过滤器后又添加**完全相同**的过滤器 → 可能命中
- 过滤器顺序变化 → 不命中（数组顺序影响对象序列化）

---

### 1.4 时间范围变化：分层失效策略

**时间范围存储结构** (`dashboard-state.ts:34-53`):
```typescript
export type DashboardState = {
  period: DashboardPeriod        // 'day'|'7d'|'28d'|'realtime'|'custom'|...
  date: Dayjs | null             // 用于 day/month/year 类型的锚点日期
  from: Dayjs | null             // custom 范围起始
  to: Dayjs | null               // custom 范围结束
  comparison: ComparisonMode     // previous_period|year_over_year|custom|null
  compare_from: Dayjs | null     // 自定义比较起始
  compare_to: Dayjs | null       // 自定义比较结束
  match_day_of_week: boolean     // 比较时是否对齐周几
}
```

**staleTime 决策逻辑** (`api-client.ts:188-212`):

```typescript
export const getStaleTime = (props: DashboardTimeSettings): number => {
  // 规则 1: 实时模式 = 30秒
  if ([DashboardPeriod.realtime, DashboardPeriod.realtime_30m].includes(props.period)) {
    return CACHE_TTL_REALTIME  // = REALTIME_UPDATE_TIME_MS = 30_000
  }

  // 规则 2: 历史数据（不含今天）= 12小时
  if (isHistoricalPeriod(props)) {
    return CACHE_TTL_HISTORICAL  // 12h
  }

  // 规则 3: 包含今天
  const availableIntervals = validIntervals(props)
  
  if (availableIntervals.includes(Interval.day) ||
      availableIntervals.includes(Interval.hour) ||
      availableIntervals.includes(Interval.minute)) {
    // 规则 3a: 支持细粒度 = 5分钟
    return CACHE_TTL_SHORT_ONGOING  // 5m
  } else {
    // 规则 3b: 不支持细粒度（如年度视图）= 1小时
    return CACHE_TTL_LONG_ONGOING  // 1h
  }
}
```

**历史数据判定** (`dashboard-time-periods.ts:97-121`):
```typescript
export function isHistoricalPeriod(props) {
  const startOfDay = now(siteTimezoneOffset).startOf('day')

  const mainPeriodIncludesToday =
    period === 'custom' && to && from
      ? !to.isBefore(startOfDay)  // custom 结束时间 >= 今天
      : !(date?.isBefore(startOfDay) || 
          ['7d', '28d', '30d', '91d', '6mo', '12mo'].includes(period))

  const comparisonPeriodIncludesToday =
    comparison === 'custom' && compare_to && compare_from &&
    !compare_to.isBefore(startOfDay)

  return !(mainPeriodIncludesToday || comparisonPeriodIncludesToday)
}
```

**时间范围变化的缓存命中矩阵**：

| 原时间范围 | 新时间范围 | 缓存命中? | 原因 |
|-----------|-----------|----------|------|
| Today (5min TTL) | Today (同日期) | 可能 | queryKey 相同，取决于 TTL 是否过期 |
| Today | Yesterday | 否 | `date` 字段变化 |
| 28d (5min TTL) | 7d | 否 | `period` 字段变化 |
| custom [2024-01-01, 2024-01-31] | custom [2024-01-01, 2024-02-01] | 否 | `to` 字段变化 |
| Yesterday (12h TTL) | Yesterday | 可能 | 历史数据，12小时内可命中 |

---

### 1.5 缓存键变化汇总表

| 用户操作 | 影响的 queryKey 字段 | 缓存行为 |
|---------|---------------------|---------|
| 切换站点 | queryKey 完全重建（新页面） | **100% 失效** |
| 添加/移除/修改过滤器 | `dashboardState.filters` | **精确键失效** |
| 切换 period (day → 7d) | `dashboardState.period` | **键不匹配，失效** |
| 修改 custom 范围 (from/to) | `dashboardState.from/to` | **键不匹配，失效** |
| 时间自然流逝 (Today 内) | 字段不变 | **取决于 TTL，可能命中** |
| 实时模式 30s tick | 字段不变 → 主动 invalidate | **强制失效，静默刷新** |

---

## 2. 降级触发链路深度拆解

### 2.1 三级降级体系总览

```
┌─────────────────────────────────────────────────────────────────────┐
│                        请求进入                                       │
│                           │                                          │
│                           ▼                                          │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  LEVEL 1: 前端静默刷新 (仅实时模式)                           │    │
│  │  ┌─────────────────────────────────────────────────────┐    │    │
│  │  │ 触发条件:                                            │    │    │
│  │  │   • period = 'realtime' 或 'realtime_30m'           │    │    │
│  │  │   • 每 30s timer tick 事件                          │    │    │
│  │  │ 行为:                                                │    │    │
│  │  │   • placeholderData = previousData (显示旧数据)       │    │    │
│  │  │   • isRealtimeSilentUpdate 标记 (隐藏 spinner)       │    │    │
│  │  │   • invalidateQueries (强制重新请求)                  │    │    │
│  │  └─────────────────────────────────────────────────────┘    │    │
│  └────────────────────────────┬────────────────────────────────┘    │
│                               │                                      │
│                               ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  LEVEL 2: 后端查询拒绝 (11 项验证)                            │    │
│  │  ┌─────────────────────────────────────────────────────┐    │    │
│  │  │ 验证顺序:                                            │    │    │
│  │  │  1. validate_order_by                               │    │    │
│  │  │  2. validate_custom_props_access                    │    │    │
│  │  │  3. validate_case_sensitive_filter_modifier         │    │    │
│  │  │  4. validate_toplevel_only_filter_dimension         │    │    │
│  │  │  5. validate_time_dimension_granularity ⚡          │    │    │
│  │  │  6. validate_special_metrics_filters                │    │    │
│  │  │  7. validate_behavioral_filters ⚡                  │    │    │
│  │  │  8. validate_filtered_goals_exist                   │    │    │
│  │  │  9. validate_revenue_metrics_access                 │    │    │
│  │  │ 10. validate_metrics ⚡                             │    │    │
│  │  │ 11. validate_include                                │    │    │
│  │  │                                                      │    │    │
│  │  │ 行为: 返回 {:error, %QueryError{code, message}}     │    │    │
│  │  └─────────────────────────────────────────────────────┘    │    │
│  └────────────────────────────┬────────────────────────────────┘    │
│                               │                                      │
│                               ▼ 验证通过                               │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  LEVEL 3: 后端采样降级 (EE Only)                              │    │
│  │  ┌─────────────────────────────────────────────────────┐    │    │
│  │  │ 触发条件:                                            │    │    │
│  │  │   • 查询范围 >= 1 天                                 │    │    │
│  │  │   • 估算流量 > 25M (fraction < 0.4)                  │    │    │
│  │  │ 行为:                                                │    │    │
│  │  │   • sample_threshold = max(fraction, 0.013)         │    │    │
│  │  │   • SQL 添加 "SAMPLE <threshold>" hint              │    │    │
│  │  │   • 结果返回 sample_percent 指标                    │    │    │
│  │  └─────────────────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 2.2 Level 1: 前端静默刷新

**触发条件**：实时模式 + 30s tick

```typescript
// 定时器: util/realtime-update-timer.js
export const REALTIME_UPDATE_TIME_MS = 30_000

export function start() {
  setInterval(() => {
    document.dispatchEvent(new Event('tick'))
  }, REALTIME_UPDATE_TIME_MS)
}
```

**Main Graph 静默刷新逻辑** (`visitor-graph.tsx:156-178`):

```typescript
useEffect(() => {
  const onTick = () => {
    // 设置静默刷新标记
    setIsRealtimeSilentUpdate({ topStats: true, mainGraph: true })
    
    // 主动使缓存失效（不等待 TTL）
    queryClient.invalidateQueries({
      predicate: ({ queryKey }) => {
        return ['top-stats', 'main-graph'].includes(queryKey[0]) &&
          queryKey[1]?.dashboardState?.period === DashboardPeriod.realtime
      }
    })
  }

  // 注册 tick 监听（仅实时模式）
  if (isRealtime) {
    document.addEventListener('tick', onTick)
    return () => document.removeEventListener('tick', onTick)
  }
}, [queryClient, isRealtime])
```

**静默刷新的关键要素**：

| 要素 | 实现方式 | 效果 |
|-----|---------|------|
| 旧数据显示 | `placeholderData: (prev) => prev` | 请求期间继续显示旧数据 |
| 隐藏加载态 | `isRealtimeSilentUpdate` 条件判断 | 不显示 spinner |
| 强制更新 | `queryClient.invalidateQueries()` | 忽略 TTL，立即请求 |
| 请求取消 | `api.cancelAll()` + `AbortController` | 新请求到达时取消旧请求 |

**实时模式 vs 普通模式的缓存行为对比**：

| 特性 | 实时模式 (realtime) | 普通模式 (如 28d) |
|-----|-------------------|------------------|
| staleTime | 30秒 (等于刷新间隔) | 5分钟 |
| 触发机制 | 主动 invalidate + tick 事件 | 缓存过期 + 组件重挂 |
| 旧数据显示 | placeholderData + 静默标记 | 无特殊处理 |
| 查询时间范围 | 5分钟 (实时) → 30分钟 (图表) | 完整 period |

**注意**：实时图表实际查询 `realtime_30m` 以获得更完整的趋势线 (`fetch-main-graph.ts:32-34`)：
```typescript
if (isRealTimeDashboard(dashboardState)) {
  statsQuery.date_range = DashboardPeriod.realtime_30m
}
```

---

### 2.3 Level 2: 后端查询拒绝

**验证流水线** (`query_builder.ex:31-60`):

```elixir
def build(site, parsed_query_params, debug_metadata) do
  with {:ok, parsed_query_params} <- resolve_segments_in_filters(parsed_query_params, site),
       query = do_build(parsed_query_params, site, debug_metadata),
       :ok <- validate_order_by(query),
       :ok <- validate_custom_props_access(site, query),
       :ok <- validate_case_sensitive_filter_modifier(query),
       :ok <- validate_toplevel_only_filter_dimension(query),
       :ok <- validate_time_dimension_granularity(query),       # ⚡ 粒度限制
       :ok <- validate_special_metrics_filters(query),
       :ok <- validate_behavioral_filters(query),               # ⚡ 行为过滤
       :ok <- validate_filtered_goals_exist(query, parsed_query_params),
       :ok <- validate_revenue_metrics_access(site, query),
       :ok <- validate_metrics(query),                         # ⚡ 指标验证
       :ok <- validate_include(query) do
    # 验证通过后才设置采样率
    on_ee do
      query = Plausible.Stats.Sampling.put_threshold(query, site, %{})
    end
    {:ok, query}
  end
end
```

**关键拒绝规则详情**：

#### 规则 5: 时间粒度限制 (`validate_time_dimension_granularity`)

```elixir
@max_hours_for_minute_interval 30  # minute 粒度最多 30 小时

defp validate_time_dimension_granularity(query) do
  if Time.time_dimension(query) == "time:minute" and
       DateTimeRange.length(query.utc_time_range, :minute) > @max_hours_for_minute_interval * 60 do
    {:error, %QueryError{
      code: :invalid_dimensions,
      message: "Dimension `time:minute` is only supported for time ranges up to 30 hours."
    }}
  else
    :ok
  end
end
```

**粒度-范围对应关系**（查询优化器自动选择）：

| 查询范围 | 自动选择粒度 | 代码位置 |
|---------|-------------|---------|
| ≤ 48 小时 | `time:hour` | `query_optimizer.ex:95-97` |
| ≤ 40 天 | `time:day` | `query_optimizer.ex:97-98` |
| ≤ 52 周 | `time:week` | `query_optimizer.ex:98-99` |
| > 52 周 | `time:month` | `query_optimizer.ex:99-100` |

#### 规则 7: 行为过滤器限制 (`validate_behavioral_filters`)

```elixir
# 禁止: has_done / has_not_done 嵌套超过 1 层
# 禁止: 非 event:* 维度使用行为过滤器
# 例: 禁止 "has_done visit:country US"
```

#### 规则 10: 指标-维度兼容性 (`validate_metrics` + TableDecider)

```elixir
# validate_metric 中的规则:
# - conversion_rate/group_conversion_rate 必须与 event:goal 同时使用
# - scroll_depth 必须与 event:page 过滤或维度同时使用
# - exit_rate 必须与 event:page 维度或过滤同时使用
# - time_on_page 必须与 event:page 维度或过滤同时使用

# TableDecider.validate_no_metrics_dimensions_conflict 中的规则:
# - Session metrics (bounce_rate, visit_duration, etc.) 
#   不能与 event:* 维度同时使用（event:page 除外）
# - Event metrics 不能与 session:* 维度同时使用
```

**指标-维度兼容性矩阵**：

| 指标类型 | 指标 | 可搭配维度 | 不可搭配维度 |
|---------|------|----------|------------|
| Event | `pageviews`, `events`, `scroll_depth` | 任意 | session:* 维度 |
| Session | `bounce_rate`, `visit_duration`, `views_per_visit`, `exit_rate` | visit:* + time:* | event:* (event:page 除外) |
| Either | `visitors`, `visits`, `conversion_rate` | 任意 | 无 |
| Revenue | `total_revenue`, `average_revenue` | 任意 (EE) | 无 (CE) |

---

### 2.4 Level 3: 后端采样降级 (EE Only)

**采样决策流程** (`sampling.ex:56-104`):

```
1. 获取 30 天流量估算 (SamplingCache)
   ├── 普通站点: SamplingCache.get(site.id)
   └── 聚合站点: SamplingCache.consolidated_get([site_ids])

2. 按查询范围缩放
   duration = Date.diff(date_range.last, date_range.first)
   duration_adjusted_traffic = traffic_30_day / 30.0 * duration

3. 按过滤器数量衰减
   filtered_estimate = duration_adjusted_traffic * (1/4) ** min(filters_count, 2)

4. 计算采样率
   fraction = 10,000,000 / estimated_traffic

5. 决策
   ├── 范围 < 1 天 → :no_sampling
   ├── fraction > 0.4 → :no_sampling  (采样效果不显著)
   └── 否则 → max(fraction, 0.013)  (最低 1.3%)
```

**采样率计算示例**：

```
假设: 站点 30 天流量 = 750 万 (7,500,000)

场景 1: 查询 364 天，无过滤器
  duration_adjusted = 7.5M / 30 * 364 = 91M
  fraction = 10M / 91M = 0.11
  → 采样 11%

场景 2: 查询 900 天，无过滤器
  duration_adjusted = 7.5M / 30 * 900 = 225M
  fraction = 10M / 225M = 0.044
  → 采样 4.4%

场景 3: 查询 2900 天 (约 8 年)
  duration_adjusted = 7.5M / 30 * 2900 = 725M
  fraction = 10M / 725M = 0.0138
  → 采样 1.38% (接近最低 1.3%)

场景 4: 查询 30 天，5M 流量 (阈值 10M 的一半)
  fraction = 10M / 5M = 2.0 (>> 0.4)
  → :no_sampling

场景 5: 实时查询 (5 分钟)
  duration < 1 天
  → :no_sampling
```

**SQL 层集成** (`base.ex:36-38`):

```elixir
defp query_events(query) do
  q = from(e in "events_v2",
    where: ^SQL.WhereBuilder.build(:events, query)
  )

  on_ee do
    q = Plausible.Stats.Sampling.add_query_hint(q, query)
  end

  q
end

# 最终 SQL: SELECT ... FROM events_v2 SAMPLE 0.11 ...
```

**采样状态传递给前端**：
- `sample_percent` 指标返回实际采样率
- 前端可显示 "数据基于 X% 采样估算" 提示

---

## 3. 决策矩阵

### 3.1 缓存命中/失效决策矩阵

| 场景 | queryKey 变化? | staleTime | 命中? | 说明 |
|-----|---------------|-----------|-------|------|
| **站点切换** | ✅ 变化 (新页面) | - | ❌ 否 | 整页刷新，QueryClient 重建 |
| **过滤器增删改** | ✅ 变化 (filters 字段) | - | ❌ 否 | 对象引用变化 |
| **period 切换** | ✅ 变化 (period 字段) | - | ❌ 否 | 键不匹配 |
| **custom 范围变化** | ✅ 变化 (from/to 字段) | - | ❌ 否 | 键不匹配 |
| **Today 内快速往返** | ❌ 不变 | 5分钟 | ✅ 可能 | 5分钟内返回同一筛选条件 |
| **Yesterday → Today** | ✅ 变化 (date 字段) | - | ❌ 否 | 日期锚点变化 |
| **历史数据重复访问** | ❌ 不变 | 12小时 | ✅ 可能 | 12小时内无需重查 |
| **实时模式 30s 内** | ❌ 不变 | 30秒 | ✅ 是 | tick 前使用缓存 |
| **实时模式 tick 后** | ❌ 不变 → invalidate | - | ❌ 否 | 主动强制刷新 |
| **组件卸载重挂** | ❌ 不变 | 取决于 period | ✅ 可能 | React Query 缓存持久化 |

### 3.2 降级类型决策矩阵

| 条件 | 降级类型 | 行为 | 用户可见 |
|-----|---------|------|---------|
| `period ∈ [realtime, realtime_30m]` | **静默刷新** (前端) | 显示旧数据，后台请求 | 无 spinner，数据无缝更新 |
| `time:minute` 且 范围 > 30h | **拒绝查询** (后端) | 返回 QueryError | 显示错误提示 |
| `conversion_rate` 无 `event:goal` | **拒绝查询** (后端) | 返回 QueryError | 显示错误提示 |
| 行为过滤器嵌套 > 1 层 | **拒绝查询** (后端) | 返回 QueryError | 显示错误提示 |
| 范围 ≥ 1 天 **且** 估算流量 > 25M | **采样** (EE, 后端) | SAMPLE fraction | 显示采样百分比提示 |
| 范围 < 1 天 | **无降级** | 全量查询 | 无 |
| 估算流量 ≤ 25M (fraction > 0.4) | **无降级** | 全量查询 | 无 |
| SamplingCache 无数据 | **无降级** | 全量查询 | 无 |

### 3.3 时间范围 → staleTime 映射矩阵

| 时间范围类型 | 示例 | 是否含今天 | 支持粒度 | staleTime |
|-------------|------|-----------|---------|-----------|
| **实时** | `realtime`, `realtime_30m` | ✅ 是 | minute | **30 秒** |
| **短周期正在进行** | `day`, `24h`, `7d`, `28d`, `30d` | ✅ 是 | hour/day | **5 分钟** |
| **长周期正在进行** | `12mo`, `year` | ✅ 是 | week/month | **1 小时** |
| **历史数据** | 昨天、上月、2023 全年 | ❌ 否 | 任意 | **12 小时** |

### 3.4 流量-采样率映射矩阵

| 站点 30 天流量 | 查询范围 | 估算流量 (缩放后) | fraction | 采样率 | 说明 |
|---------------|---------|-----------------|----------|--------|------|
| < 25M | 30 天 | < 25M | > 0.4 | 无 | 采样效果不显著 |
| 25M | 30 天 | 25M | 0.4 | 无 | 临界值 |
| 50M | 30 天 | 50M | 0.2 | 20% | - |
| 100M | 30 天 | 100M | 0.1 | 10% | - |
| 100M | 365 天 | ~1.2B | 0.008 | **1.3%** | 触底 |
| 任意 | 实时 (5 分钟) | - | - | 无 | 范围 < 1 天 |
| 100M | 30 天 + 1 过滤器 | 25M | 0.4 | 无 | 过滤器衰减 (×1/4) |
| 100M | 30 天 + 2+ 过滤器 | 6.25M | 1.6 | 无 | 两层衰减 (×1/16) |

---

## 4. 端到端请求-响应链路

### 4.1 实时刷新链路

```
用户在实时视图
      │
      │ 每 30 秒
      ▼
timer.js: setInterval → dispatchEvent('tick')
      │
      ▼
visitor-graph.tsx: onTick() 监听
      │
      ├── setIsRealtimeSilentUpdate({...})  ← 隐藏 spinner
      │
      └── queryClient.invalidateQueries(predicate)  ← 强制失效
            │
            ▼
      React Query 检查:
        • queryKey 不变
        • placeholderData = previousData  ← 显示旧数据
        • 忽略 staleTime，立即请求
            │
            ▼
      api.stats() → AbortController 取消旧请求
            │
            ▼
      POST /api/stats/:domain/query
            │
            ├─ 后端: date_range = :realtime_30m (图表)
            │              = :realtime (顶栏)
            │
            ├─ 后端: duration < 1 天 → :no_sampling
            │
            └─ 后端: 返回最新数据
                  │
                  ▼
            React Query 更新缓存
                  │
                  ▼
            组件重新渲染 (无 spinner)
```

### 4.2 大查询采样链路

```
用户选择 "All Time" 视图 (100M 流量站点)
      │
      ▼
dashboardState.period = 'all'
      │
      ▼
queryKey: ['main-graph', { dashboardState, metric: 'visitors', interval: 'month' }]
      │
      ├─ 缓存检查: 若无，发起请求
      │
      ▼
POST /api/stats/:domain/query
      │
      ▼
后端 QueryBuilder.build():
  ├── 验证通过 (时间粒度自动选择 month)
  │
  └── Sampling.put_threshold():
        ├── SamplingCache.get(site.id) → 100M
        ├── duration = Date.diff(all_time_end, all_time_start) = 1095 天
        ├── estimated = 100M / 30 * 1095 = 3.65B
        ├── fraction = 10M / 3.65B = 0.0027
        └── sample_threshold = max(0.0027, 0.013) = 0.013
      │
      ▼
SQL 执行:
  SELECT count(DISTINCT visitor_id)
  FROM sessions_v2
  SAMPLE 0.013  ← 1.3% 采样
  WHERE ...
      │
      ▼
返回响应:
  {
    results: [...],
    query: { sample_percent: 1.3 }  ← 前端可显示提示
  }
```

### 4.3 非法查询拒绝链路

```
用户尝试: "time:minute" 粒度 + "7d" 范围
      │
      ▼
后端 QueryBuilder.build():
  │
  └── validate_time_dimension_granularity(query)
        │
        ├── Time.time_dimension(query) = "time:minute"
        ├── DateTimeRange.length(range, :minute) = 7*24*60 = 10080
        ├── @max_hours_for_minute_interval = 30 → 1800 分钟
        └── 10080 > 1800 → :error
      │
      ▼
返回 HTTP 400:
  {
    "error": "Dimension `time:minute` is only supported for time ranges up to 30 hours."
  }
      │
      ▼
前端 ErrorBoundary 或组件显示错误信息
```

---

## 5. 关键代码索引

| 功能 | 文件 | 行号 |
|-----|------|------|
| **缓存键构造** | | |
| Top Stats queryKey | `assets/js/dashboard/stats/graph/visitor-graph.tsx` | 47 |
| Main Graph queryKey | `assets/js/dashboard/stats/graph/visitor-graph.tsx` | 65 |
| Breakdown queryKey | `assets/js/dashboard/hooks/api-client.ts` | 62 |
| **站点切换** | | |
| 站点切换 URL 构造 | `assets/js/dashboard/site-switcher.tsx` | 59-68 |
| 整页刷新执行 | `assets/js/dashboard/site-switcher.tsx` | 135, 261 |
| API 路径域名隔离 | `assets/js/dashboard/util/url.ts` | 3-8 |
| 挂载时取消所有请求 | `assets/js/dashboard/dashboard-state-context.tsx` | 143-145 |
| **过滤器处理** | | |
| DashboardState filters | `assets/js/dashboard/dashboard-state.ts` | 34-53 |
| 前端→后端 filter 映射 | `assets/js/dashboard/util/filters.js` | 197-271 |
| createStatsQuery filters | `assets/js/dashboard/stats-query.ts` | 65-88 |
| **时间范围处理** | | |
| staleTime 决策函数 | `assets/js/dashboard/hooks/api-client.ts` | 188-212 |
| 历史数据判定 | `assets/js/dashboard/dashboard-time-periods.ts` | 97-121 |
| CACHE_TTL 常量 | `assets/js/dashboard/hooks/api-client.ts` | 20-23 |
| 自动粒度选择 | `lib/plausible/stats/query_optimizer.ex` | 95-102 |
| **静默刷新** | | |
| 实时定时器 | `assets/js/dashboard/util/realtime-update-timer.js` | 1-8 |
| Main Graph tick 处理 | `assets/js/dashboard/stats/graph/visitor-graph.tsx` | 156-178 |
| placeholderData 设置 | `assets/js/dashboard/stats/graph/visitor-graph.tsx` | 63 |
| 实时图表 30m 范围 | `assets/js/dashboard/stats/graph/fetch-main-graph.ts` | 32-34 |
| **查询拒绝** | | |
| 验证流水线入口 | `lib/plausible/stats/query_builder.ex` | 31-60 |
| 时间粒度限制 | `lib/plausible/stats/query_builder.ex` | 335-350 |
| 指标-维度兼容 | `lib/plausible/stats/query_builder.ex` | 475-570 |
| 表选择兼容检查 | `lib/plausible/stats/table_decider.ex` | 44-77 |
| **采样降级** | | |
| 采样决策核心 | `extra/lib/plausible/stats/sampling.ex` | 56-104 |
| 采样率计算 | `extra/lib/plausible/stats/sampling.ex` | 71-90 |
| 流量估算 (含过滤器) | `extra/lib/plausible/stats/sampling.ex` | 94-104 |
| SQL SAMPLE hint | `extra/lib/plausible/stats/sampling.ex` | 27-30 |
| SamplingCache | `extra/lib/plausible/stats/sampling_cache.ex` | 1-59 |
