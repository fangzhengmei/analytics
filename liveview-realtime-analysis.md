# 实时访客指标刷新机制分析报告

## 概述

本报告详细分析了 Plausible Analytics 系统中实时访客指标的刷新机制，**重点区分了两条独立的实时查询链路**，纠正了窗口计算的归属，明确了成本计数链路与实时访客数的关系，**逐项分析了实时看板各模块的时间窗口与刷新触发条件**，并补充了缓存时效与全局刷新节奏的配合关系。

---

## 1. 两条独立的实时查询链路

系统中存在**两条完全独立**的实时查询链路，它们有不同的用途、实现方式和数据源。

### 1.1 链路一：实时访客接口链路 (Current Visitors API)

#### 1.1.1 链路用途
专门用于获取**当前在线访客数**这单一指标，显示在导航栏的实时指示器中。

#### 1.1.2 完整调用链

```
前端组件 (current-visitors.js)
    ↓ 30秒定时触发
API 调用: GET /api/stats/{domain}/current-visitors
    ↓
StatsController.current_visitors/2 [stats_controller.ex:1236-1239]
    ↓
Plausible.Stats.current_visitors/1 [stats.ex:30-32]
    ↓
Plausible.Stats.CurrentVisitors.current_visitors/2 [current_visitors.ex:6-19]
    ↓
直接查询 ClickHouse events_v2 表
```

#### 1.1.3 核心实现

**控制器层** (`lib/plausible_web/controllers/api/stats_controller.ex:1236-1239`):
```elixir
def current_visitors(conn, _) do
  site = conn.assigns[:site]
  json(conn, Stats.current_visitors(site))
end
```

**Stats 模块代理** (`lib/plausible/stats.ex:30-32`):
```elixir
def current_visitors(site, duration \\ Duration.new!(minute: -5)) do
  CurrentVisitors.current_visitors(site, duration)
end
```

**独立查询实现** (`lib/plausible/stats/current_visitors.ex:6-19`):
```elixir
def current_visitors(site, duration \\ Duration.new!(minute: -5)) do
  first_datetime =
    NaiveDateTime.utc_now()
    |> NaiveDateTime.shift(duration)
    |> NaiveDateTime.truncate(:second)

  ClickhouseRepo.one(
    from e in "events_v2",
      where: ^Plausible.Sites.site_id_query_filter(site),
      where: e.timestamp >= ^first_datetime,
      where: e.name != "engagement",
      select: uniq(e.user_id)
  )
end
```

#### 1.1.4 链路特点

| 特性 | 值 |
|-----|---|
| API 路径 | `/api/stats/{domain}/current-visitors` |
| 默认窗口 | 5 分钟 |
| 窗口计算位置 | `current_visitors.ex` (独立实现) |
| 聚合方式 | `uniq(user_id)` - 唯一用户数 |
| 数据源 | 直接查询 `events_v2` 表 |
| 使用组件 | `CurrentVisitors` (导航栏指示器) |

---

### 1.2 链路二：实时看板通用查询链路 (Realtime Dashboard)

#### 1.2.1 链路用途
用于**实时看板**模式下的所有指标查询，包括访客图表、热门页面、来源分析等。

#### 1.2.2 完整调用链

```
前端时间周期选择: period = "realtime" 或 "realtime_30m"
    ↓
前端转换: "realtime" → "realtime_30m" [fetch-main-graph.ts:32, fetch-top-stats.ts:136]
    ↓
API 调用 (通用查询端点)
    ↓
QueryParser 解析: "realtime_30m" → :realtime_30m [query_parser.ex:41]
    ↓
QueryBuilder.build/3 构建查询
    ↓
QueryBuilder.build_datetime_range/4 计算时间范围 [query_builder.ex:89-99]
    ↓
通过统一 Query 机制查询 events_v2 表
```

#### 1.2.3 核心实现

**前端时间周期定义** (`assets/js/dashboard/dashboard-time-periods.ts:28-43`):
```typescript
export enum DashboardPeriod {
  'realtime' = 'realtime',
  'realtime_30m' = 'realtime_30m',
  'day' = 'day',
  'month' = 'month',
  // ... 其他周期
}
```

**前端实时周期转换** (`assets/js/dashboard/stats/graph/fetch-main-graph.ts:32`):
```typescript
statsQuery.date_range = DashboardPeriod.realtime_30m
```

**后端解析** (`lib/plausible/stats/dashboard/query_parser.ex:41`):
```elixir
"realtime_30m" -> {:ok, :realtime_30m}
```

**统一窗口计算** (`lib/plausible/stats/query_builder.ex:88-100`):
```elixir
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

#### 1.2.4 链路特点

| 特性 | 值 |
|-----|---|
| 前端周期 | `realtime` (显示用)、`realtime_30m` (实际查询用) |
| 实际窗口 | 30 分钟 (`realtime_30m`) |
| 窗口计算位置 | `query_builder.ex` (统一查询构建器) |
| 聚合方式 | 通过统一 Query 机制支持多种指标 |
| 数据源 | 通过 `QueryBuilder` 构建查询，最终查询 `events_v2` 表 |
| 使用场景 | 实时看板的所有图表和表格 |

---

### 1.3 两条链路对比总结

| 对比维度 | 实时访客接口链路 | 实时看板通用查询链路 |
|---------|-----------------|---------------------|
| **用途** | 单一指标：当前在线访客数 | 实时看板所有指标 |
| **API 端点** | `/api/stats/{domain}/current-visitors` (专用) | 通用查询端点 (`/api/stats/query` 等) |
| **窗口大小** | 默认 5 分钟 (可配置) | 30 分钟 (`realtime_30m`) |
| **窗口计算位置** | `current_visitors.ex` (独立实现) | `query_builder.ex` (统一构建器) |
| **查询构建** | 直接写 Ecto 查询，不使用 QueryBuilder | 使用完整的 QueryBuilder 流程 |
| **聚合方式** | 仅 `uniq(user_id)` | 支持多种指标聚合 |
| **前端刷新触发** | 全局 `tick` 事件 (30秒) | 依赖各组件自身的刷新机制 |
| **显示位置** | 导航栏实时指示器 | 实时看板页面 |

**关键结论**：
> 这是**两套完全独立**的实现！`current_visitors.ex` 中的窗口计算与 `query_builder.ex` 中的窗口计算**没有任何关系**。

---

## 2. 实时看板各模块分析

### 2.1 实时看板模块全景

实时看板包含以下核心模块，各模块有不同的时间窗口转换和刷新触发条件：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           实时看板页面结构                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      Top Stats (顶部统计)                          │   │
│  │  • Current visitors (当前访客数) - 特殊处理                        │   │
│  │  • Visitors (访客数)                                              │   │
│  │  • Pageviews (页面浏览量)                                         │   │
│  │  • 其他指标 (依筛选条件而定)                                        │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      Main Graph (主图表)                           │   │
│  │  • 访客趋势图表                                                    │   │
│  │  • 支持多种指标切换                                                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌───────────────────────┬───────────────────────────────────────────┐ │
│  │   Sources (来源)      │       Pages (页面)                        │ │
│  │   • Top Sources       │       • Top Pages                         │ │
│  │   • Top Channels      │       • Entry Pages                       │ │
│  │   • Top Referrers     │       • Exit Pages                        │ │
│  └───────────────────────┴───────────────────────────────────────────┘ │
│                                                                         │
│  ┌───────────────────────┬───────────────────────────────────────────┐ │
│  │   Locations (地理位置) │       Devices (设备)                      │ │
│  │   • Countries         │       • Browsers                          │ │
│  │   • Regions           │       • Operating Systems                 │ │
│  │   • Cities            │       • Screen Sizes                      │ │
│  └───────────────────────┴───────────────────────────────────────────┘ │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      Behaviours (行为)                             │   │
│  │  • Props (自定义属性)                                             │   │
│  │  • Conversions (转化) - 仅当有目标筛选时显示                        │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 模块时间窗口转换分析

#### 2.2.1 关键转换机制

**前端周期转换规则**：
- 用户选择 `period = "realtime"`（显示用）
- 实际查询时转换为 `date_range = "realtime_30m"`（30分钟窗口）

**转换位置**：
- `fetch-main-graph.ts:32` - 主图表
- `fetch-top-stats.ts:136` - 顶部统计

#### 2.2.2 各模块时间窗口对照表

| 模块 | 文件位置 | 是否转换为 realtime_30m | 实际窗口大小 | 说明 |
|-----|---------|------------------------|-------------|------|
| **Top Stats (顶部统计)** | `fetch-top-stats.ts:136` | ✅ 是 | 30 分钟 | 显式转换 `date_range = DashboardPeriod.realtime_30m` |
| **Main Graph (主图表)** | `fetch-main-graph.ts:32` | ✅ 是 | 30 分钟 | 显式转换 `date_range = DashboardPeriod.realtime_30m` |
| **Sources (来源)** | 通用查询机制 | ✅ 是 | 30 分钟 | 通过统一 Query 机制，使用 `dashboardState.period` |
| **Pages (页面)** | 通用查询机制 | ✅ 是 | 30 分钟 | 通过统一 Query 机制，使用 `dashboardState.period` |
| **Locations (地理位置)** | 通用查询机制 | ✅ 是 | 30 分钟 | 通过统一 Query 机制，使用 `dashboardState.period` |
| **Devices (设备)** | 通用查询机制 | ✅ 是 | 30 分钟 | 通过统一 Query 机制，使用 `dashboardState.period` |
| **Behaviours (行为)** | 通用查询机制 | ✅ 是 | 30 分钟 | 通过统一 Query 机制，使用 `dashboardState.period` |

**关键发现**：
> 所有实时看板模块在查询时**都使用 30 分钟窗口**（`realtime_30m`），而不是用户界面上显示的 "realtime"。

### 2.3 Top Stats 模块特殊处理

#### 2.3.1 Current Visitors 特殊处理

**位置**: `fetch-top-stats.ts:19-35`

```typescript
export function topStatsQueries(
  dashboardState: DashboardState,
  metrics: Metric[]
): [StatsQuery, StatsQuery | null] {
  let currentVisitorsQuery = null

  if (isRealTimeDashboard(dashboardState)) {
    currentVisitorsQuery = createStatsQuery(dashboardState, {
      metrics: ['visitors']
    })

    currentVisitorsQuery.filters = []  // 清除筛选器
  }
  const topStatsQuery = constructTopStatsQuery(dashboardState, metrics)

  return [topStatsQuery, currentVisitorsQuery]
}
```

**特殊处理说明**：
- 在实时看板中，`Current visitors` 指标使用**单独的查询**
- 该查询**清除了所有筛选器**（`filters = []`）
- 但**仍会使用 30 分钟窗口**（通过后续的 `date_range` 转换）

#### 2.3.2 指标选择逻辑

**位置**: `fetch-top-stats.ts:77-112`

```typescript
export function chooseMetrics(
  site: Pick<PlausibleSite, 'revenueGoals'>,
  dashboardState: DashboardState
): Metric[] {
  // ...
  if (
    isRealTimeDashboard(dashboardState) &&
    hasConversionGoalFilter(dashboardState)
  ) {
    return ['visitors', 'events']
  } else if (isRealTimeDashboard(dashboardState)) {
    return ['visitors', 'pageviews']
  }
  // ... 其他周期的指标选择
}
```

**实时看板指标选择**：
- 无筛选器：`['visitors', 'pageviews']`
- 有目标筛选器：`['visitors', 'events']`

#### 2.3.3 时间窗口转换

**位置**: `fetch-top-stats.ts:135-137`

```typescript
if (isRealTimeDashboard(dashboardState)) {
  statsQuery.date_range = DashboardPeriod.realtime_30m
}
```

**关键**：无论主查询还是 `Current visitors` 子查询，**最终都会使用 30 分钟窗口**。

### 2.4 Main Graph 模块

#### 2.4.1 时间窗口转换

**位置**: `fetch-main-graph.ts:31-33`

```typescript
if (isRealTimeDashboard(dashboardState)) {
  statsQuery.date_range = DashboardPeriod.realtime_30m
}
```

#### 2.4.2 实时看板特殊显示

**位置**: `main-graph.tsx:574-589`

```typescript
case Interval.minute: {
  if (period === DashboardPeriod.realtime) {
    const minutesAgo = totalBuckets - bucketIndex
    return `-${minutesAgo}m`  // 显示为 "-Xm" 格式
  }
  // ... 其他周期的显示逻辑
}
```

**显示特点**：
- 实时看板使用 `minute` 时间间隔
- X 轴标签显示为 `-Xm`（如 "-1m", "-2m"），表示"X分钟前"
- 实际查询使用 30 分钟窗口

### 2.5 其他模块 (Sources, Pages, Locations, Devices, Behaviours)

这些模块使用**统一的查询机制**，时间窗口由 `dashboardState.period` 决定。

#### 2.5.1 周期判断

**位置**: `util/filters.js:141-143`

```javascript
export function isRealTimeDashboard(dashboardState) {
  return dashboardState?.period === 'realtime'
}
```

#### 2.5.2 实时看板特殊逻辑

**位置**: `stats/behaviours/index.js:436-438`

```javascript
function isRealtime() {
  return dashboardState.period === 'realtime'
}
```

**用途**：
- 控制某些 UI 元素的显示/隐藏
- 控制某些功能的启用/禁用

### 2.6 实时看板 vs 非实时看板对比

| 维度 | 实时看板 (period=realtime) | 非实时看板 (period=day/month/等) |
|-----|---------------------------|---------------------------------|
| **实际查询窗口** | 30 分钟 (realtime_30m) | 对应周期的时间范围 |
| **时间间隔** | minute (分钟级) | 依周期而定 (hour, day, week 等) |
| **X 轴标签** | "-Xm" (X分钟前) | 日期/时间格式 |
| **可用指标** | visitors, pageviews/events | 更多指标 (bounce_rate, visit_duration 等) |
| **闪烁圆点** | 显示 pulsating-circle | 不显示 |
| **比较模式** | 禁用 | 可用 |

---

## 3. 刷新触发条件分析

### 3.1 全局刷新机制

#### 3.1.1 全局定时器

**位置**: `assets/js/dashboard/util/realtime-update-timer.js`

```javascript
export const REALTIME_UPDATE_TIME_MS = 30_000  // 30秒
const tickEvent = new Event('tick')

export function start() {
  setInterval(() => {
    document.dispatchEvent(tickEvent)
  }, REALTIME_UPDATE_TIME_MS)
}
```

#### 3.1.2 事件驱动架构

```
┌─────────────────────────────────────────────────────────────┐
│                    全局刷新机制                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  realtime-update-timer.js (定时器模块)                       │
│              │                                              │
│              │ 每 30 秒                                     │
│              ▼                                              │
│  ┌───────────────────────┐                                 │
│  │  全局 'tick' 事件       │                                 │
│  └───────────┬───────────┘                                 │
│              │                                              │
│    ┌─────────┼─────────┐                                   │
│    │         │         │                                   │
│    ▼         ▼         ▼                                   │
│  ┌─────┐ ┌─────┐ ┌─────┐                                 │
│  │组件A│ │组件B│ │组件C│  (组件自主决定是否响应)            │
│  └─────┘ └─────┘ └─────┘                                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 各模块刷新触发条件

#### 3.2.1 CurrentVisitors 组件 (导航栏指示器)

**位置**: `current-visitors.js`

```javascript
useEffect(() => {
  document.addEventListener('tick', updateCount)  // 监听全局 tick 事件
  return () => {
    document.removeEventListener('tick', updateCount)
  }
}, [updateCount])

useEffect(() => {
  updateCount()  // 组件挂载或状态变化时刷新
}, [dashboardState, updateCount])
```

**刷新触发条件**：

| 触发条件 | 类型 | 说明 |
|---------|------|------|
| 全局 `tick` 事件 | 定时 | 每 30 秒自动刷新 |
| `dashboardState` 变化 | 状态 | 筛选器、时间范围等变化时 |
| 组件挂载 | 生命周期 | 首次渲染时获取初始数据 |

**链路类型**：实时访客接口链路（独立 API）

#### 3.2.2 实时看板模块

**刷新机制**：实时看板模块**不直接监听全局 `tick` 事件**，而是依赖 React Query 的缓存机制。

**刷新触发条件**：

| 触发条件 | 类型 | 说明 |
|---------|------|------|
| 缓存过期 | 自动 | 当 `staleTime` 到期后，下次访问时重新获取 |
| 组件挂载 | 生命周期 | 首次渲染时检查缓存 |
| `dashboardState` 变化 | 状态 | 查询参数变化时触发新查询 |
| 用户交互 | 手动 | 切换指标、展开详情等操作 |

**关键**：实时看板的刷新**依赖缓存时效**，而不是全局定时器。

### 3.3 刷新触发条件对照表

| 模块 | 监听全局 tick | 刷新触发条件 | 链路类型 |
|-----|-------------|-------------|---------|
| **CurrentVisitors (导航栏)** | ✅ 是 | 30秒定时 + 状态变化 | 实时访客接口链路 |
| **Top Stats (实时看板)** | ❌ 否 | 缓存过期 + 组件挂载 + 状态变化 | 实时看板通用查询链路 |
| **Main Graph (实时看板)** | ❌ 否 | 缓存过期 + 组件挂载 + 状态变化 | 实时看板通用查询链路 |
| **Sources (实时看板)** | ❌ 否 | 缓存过期 + 组件挂载 + 状态变化 | 实时看板通用查询链路 |
| **Pages (实时看板)** | ❌ 否 | 缓存过期 + 组件挂载 + 状态变化 | 实时看板通用查询链路 |
| **Locations (实时看板)** | ❌ 否 | 缓存过期 + 组件挂载 + 状态变化 | 实时看板通用查询链路 |
| **Devices (实时看板)** | ❌ 否 | 缓存过期 + 组件挂载 + 状态变化 | 实时看板通用查询链路 |
| **Behaviours (实时看板)** | ❌ 否 | 缓存过期 + 组件挂载 + 状态变化 | 实时看板通用查询链路 |

**重要发现**：
> 导航栏的 `CurrentVisitors` 组件**直接监听全局 `tick` 事件**，每 30 秒强制刷新；而实时看板内部的模块**不监听 `tick` 事件**，完全依赖 React Query 的缓存机制。

---

## 4. 缓存时效与全局刷新节奏的配合

### 4.1 缓存时效配置

#### 4.1.1 缓存常量定义

**位置**: `assets/js/dashboard/hooks/api-client.ts:17-21`

```typescript
// define (in ms) when query API responses should become stale
export const CACHE_TTL_REALTIME = REALTIME_UPDATE_TIME_MS  // 30_000 ms = 30秒
export const CACHE_TTL_SHORT_ONGOING = 5 * 60 * 1000       // 5分钟
export const CACHE_TTL_LONG_ONGOING = 60 * 60 * 1000       // 1小时
export const CACHE_TTL_HISTORICAL = 12 * 60 * 60 * 1000    // 12小时
```

#### 4.1.2 关键常量关系

```typescript
// realtime-update-timer.js
export const REALTIME_UPDATE_TIME_MS = 30_000  // 30秒

// api-client.ts
export const CACHE_TTL_REALTIME = REALTIME_UPDATE_TIME_MS  // 30秒（与全局刷新同步）
```

**关键设计**：
> `CACHE_TTL_REALTIME` **完全等于** `REALTIME_UPDATE_TIME_MS`，确保缓存时效与全局刷新节奏**完全同步**。

### 4.2 缓存时效计算逻辑

#### 4.2.1 getStaleTime 函数

**位置**: `assets/js/dashboard/hooks/api-client.ts:138-162`

```typescript
export const getStaleTime = (props: DashboardTimeSettings): number => {
  // 实时周期 (realtime 或 realtime_30m)
  if (
    [DashboardPeriod.realtime, DashboardPeriod.realtime_30m].includes(
      props.period
    )
  ) {
    return CACHE_TTL_REALTIME  // 30秒
  }

  // 历史周期 (不包含今天)
  if (isHistoricalPeriod(props)) {
    return CACHE_TTL_HISTORICAL  // 12小时
  }

  // 进行中的周期 (包含今天)
  const availableIntervals = validIntervals(props)

  if (
    availableIntervals.includes(Interval.day) ||
    availableIntervals.includes(Interval.hour) ||
    availableIntervals.includes(Interval.minute)
  ) {
    return CACHE_TTL_SHORT_ONGOING  // 5分钟
  } else {
    return CACHE_TTL_LONG_ONGOING   // 1小时
  }
}
```

#### 4.2.2 缓存时效对照表

| 周期类型 | period 值 | staleTime (缓存时效) | 说明 |
|---------|-----------|---------------------|------|
| **实时周期** | `realtime` / `realtime_30m` | **30 秒** | 与全局刷新节奏完全同步 |
| **历史周期** | 不包含今天的周期 | **12 小时** | 数据不再变化，缓存时间长 |
| **短期进行中** | day, 7d 等 (支持 day/hour/minute 间隔) | **5 分钟** | 数据仍在变化，需要较频繁刷新 |
| **长期进行中** | 12mo, year 等 (不支持短间隔) | **1 小时** | 数据变化较慢，缓存时间较长 |

### 4.3 缓存时效与全局刷新的配合机制

#### 4.3.1 架构图解

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    缓存时效与全局刷新配合机制                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    全局刷新节奏 (30秒)                            │   │
│  │                                                                   │   │
│  │  realtime-update-timer.js                                        │   │
│  │         │                                                         │   │
│  │         │ 每 30 秒                                                │   │
│  │         ▼                                                         │   │
│  │  ┌─────────────────┐                                              │   │
│  │  │ 全局 'tick' 事件 │ ←── 仅 CurrentVisitors 组件监听             │   │
│  │  └─────────────────┘                                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    缓存时效配置 (React Query)                      │   │
│  │                                                                   │   │
│  │  实时周期: staleTime = 30 秒 (CACHE_TTL_REALTIME)                │   │
│  │         │                                                         │   │
│  │         │ 与全局刷新节奏完全同步                                   │   │
│  │         ▼                                                         │   │
│  │  ┌─────────────────────────────────────────────────────────┐   │   │
│  │  │  实际效果:                                                 │   │   │
│  │  │  • 30秒后缓存过期                                          │   │   │
│  │  │  • 下次访问时重新获取数据                                   │   │   │
│  │  │  • 效果上等同于每30秒刷新一次                               │   │   │
│  │  └─────────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    配合效果                                        │   │
│  │                                                                   │   │
│  │  全局定时器 (30秒)                                                │   │
│  │         │                                                         │   │
│  │         ├───→ CurrentVisitors 组件: 强制刷新 (通过 tick 事件)   │   │
│  │         │                                                         │   │
│  │         └───→ 实时看板模块: 缓存过期 (通过 staleTime)            │   │
│  │              下次访问时自动刷新                                   │   │
│  │                                                                   │   │
│  │  结果: 所有实时数据每 30 秒都会更新                               │   │
│  │                                                                   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 4.3.2 时间线示意图

```
时间轴: 0s ────────── 30s ────────── 60s ────────── 90s ──────────→

全局定时器:
         │              │              │              │
         ├──── tick ────┼──── tick ────┼──── tick ────┼────→
         │              │              │              │

CurrentVisitors 组件 (导航栏):
         │              │              │              │
         ├──── 刷新 ────┼──── 刷新 ────┼──── 刷新 ────┼────→
         (监听 tick 事件，强制刷新)

实时看板模块 (通过缓存机制):
         │              │              │              │
         │  缓存新鲜    │  缓存过期    │  缓存过期    │
         │  (0-30s)    │  (30s后)    │  (60s后)    │
         │              │              │              │
         ├──────────────┼──────────────┼──────────────┼────→
                        │              │              │
                        └── 下次访问时 └── 下次访问时 └──→
                            重新获取        重新获取

关键:
• CurrentVisitors: 每 30 秒强制刷新 (主动)
• 实时看板模块: 每 30 秒缓存过期，下次访问时刷新 (被动)
• 两者的刷新节奏完全同步 (都是 30 秒)
```

### 4.4 为什么这样设计？

#### 4.4.1 设计考量

| 设计决策 | 原因 |
|---------|------|
| **CurrentVisitors 直接监听 tick** | 导航栏指示器需要**即时响应**，用户期望看到实时更新 |
| **实时看板依赖缓存机制