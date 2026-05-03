# 实时访客指标刷新机制分析报告

## 概述

本报告详细分析了 Plausible Analytics 系统中实时访客指标的刷新机制，**重点区分了两条独立的实时查询链路**，纠正了窗口计算的归属，明确了成本计数链路与实时访客数的关系，并标清了直接数据源和旁路监控。

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

## 2. 窗口计算归属纠正

### 2.1 之前的误解

之前的分析可能存在以下误解：
- ❌ 认为所有实时窗口计算都在 `query_builder.ex` 中
- ❌ 认为 `current_visitors.ex` 使用了 `query_builder.ex` 的窗口计算
- ❌ 混淆了两条链路的窗口大小

### 2.2 正确的归属关系

#### 2.2.1 归属一：实时访客接口的窗口计算

**归属模块**: `lib/plausible/stats/current_visitors.ex`

**实现细节**:
```elixir
def current_visitors(site, duration \\ Duration.new!(minute: -5)) do
  first_datetime =
    NaiveDateTime.utc_now()
    |> NaiveDateTime.shift(duration)  # 默认: -5分钟
    |> NaiveDateTime.truncate(:second)
  # ... 查询
end
```

**特点**:
- ✅ 默认窗口：5 分钟 (`Duration.new!(minute: -5)`)
- ✅ 可自定义：通过 `duration` 参数调整
- ✅ 独立实现：不依赖 `QueryBuilder`
- ✅ 滑动窗口：每次查询基于 `NaiveDateTime.utc_now()` 重新计算

#### 2.2.2 归属二：实时看板的窗口计算

**归属模块**: `lib/plausible/stats/query_builder.ex`

**实现细节**:
```elixir
defp build_datetime_range(input_date_range, _site, _relative_date, now)
     when input_date_range in [:realtime, :realtime_30m] do
  duration_minutes =
    case input_date_range do
      :realtime -> 5          # 理论值，实际不使用
      :realtime_30m -> 30     # 实际使用值
    end

  first_datetime = DateTime.shift(now, minute: -duration_minutes)
  last_datetime = DateTime.shift(now, second: 5)  # +5秒缓冲

  DateTimeRange.new!(first_datetime, last_datetime)
end
```

**特点**:
- ✅ `:realtime` → 5 分钟 (理论定义，实际查询不使用)
- ✅ `:realtime_30m` → 30 分钟 (实际查询使用)
- ✅ 统一接口：与其他时间周期 (day, month, year 等) 使用相同的构建逻辑
- ✅ 时间缓冲：`last_datetime = now + 5 seconds` 确保最新数据被包含

### 2.3 为什么有两套独立实现？

#### 设计原因分析

| 维度 | 实时访客接口 | 实时看板 |
|-----|------------|---------|
| **响应时间要求** | 极快 (导航栏指示器需要即时响应) | 较快 (页面加载可接受短暂延迟) |
| **查询复杂度** | 极简单 (仅 `uniq(user_id)`) | 复杂 (多维度、多指标、筛选) |
| **缓存策略** | 可独立优化 | 依赖通用缓存机制 |
| **演化历史** | 早期简单实现 | 后期统一查询框架 |

#### 技术债务考虑

两套独立实现可能带来的问题：
1. **维护成本**：需要同时维护两套窗口逻辑
2. **行为差异**：5分钟 vs 30分钟窗口可能导致用户困惑
3. **代码复用**：无法共享查询优化和 bug 修复

---

## 3. 成本计数链路与实时访客数的关系

### 3.1 成本计数链路的真实用途

**成本计数链路** (`lib/plausible/ingestion/counters.ex` 及其子模块) 是一个**旁路监控系统**，**完全不用于实时访客数的计算**。

#### 3.1.1 链路架构

```
事件摄入管道 (Plausible.Ingestion.Event)
    ↓ 发射 telemetry 事件
Telemetry 事件: [:plausible, :ingestion, :event, :buffered/:dropped]
    ↓
TelemetryHandler 捕获事件 [telemetry_handler.ex:18-64]
    ↓
Buffer.aggregate/5 聚合计数 [buffer.ex]
    ↓ 每 10 秒刷新
Counters.handle_cycle/2 刷新缓冲区 [counters.ex:66-99]
    ↓ 异步插入
AsyncInsertRepo.insert_all(Record, records)
    ↓
ClickHouse ingest_counters 表 [record.ex:10-17]
```

#### 3.1.2 核心实现

**Telemetry 事件订阅** (`lib/plausible/ingestion/counters/telemetry_handler.ex:12-15`):
```elixir
@event_dropped Event.telemetry_event_dropped()
@event_buffered Event.telemetry_event_buffered()

@telemetry_events [@event_dropped, @event_buffered]
```

**事件处理** (`lib/plausible/ingestion/counters/telemetry_handler.ex:52-64`):
```elixir
def handle_event(
      @event_buffered,
      _measurements,
      %{
        domain: domain,
        request_timestamp: timestamp,
        tracker_script_version: tracker_script_version
      },
      buffer
    ) do
  Counters.Buffer.aggregate(buffer, "buffered", domain, timestamp, tracker_script_version)
  :ok
end
```

**数据结构** (`lib/plausible/ingestion/counters/record.ex:10-17`):
```elixir
schema "ingest_counters" do
  field :event_timebucket, :utc_datetime
  field :site_id, Ch, type: "Nullable(UInt64)"
  field :domain, Ch, type: "LowCardinality(String)"
  field :metric, Ch, type: "LowCardinality(String)"
  field :value, Ch, type: "UInt64"
  field :tracker_script_version, Ch, type: "UInt16"
end
```

#### 3.1.3 收集的指标

| Metric 名称 | 含义 | 触发条件 |
|------------|------|---------|
| `buffered` | 成功缓冲的事件数 | 事件被成功写入摄入管道 |
| `dropped_#{reason}` | 因特定原因丢弃的事件数 | 事件被丢弃 (如 `dropped_robot`, `dropped_rate_limited` 等) |

### 3.2 实时访客数的真实数据源

**实时访客数**的计算**完全依赖 `events_v2` 表**，与 `ingest_counters` 表没有任何关系。

#### 3.2.1 直接数据源

**表名**: `events_v2`

**查询条件** (`current_visitors.ex:12-18`):
```elixir
from e in "events_v2",
  where: ^Plausible.Sites.site_id_query_filter(site),  # 按站点过滤
  where: e.timestamp >= ^first_datetime,                # 时间窗口 (默认5分钟)
  where: e.name != "engagement",                         # 排除互动事件
  select: uniq(e.user_id)                                # 统计唯一用户数
```

#### 3.2.2 为什么 events_v2 是直接数据源？

| 原因 | 说明 |
|-----|------|
| **数据完整性** | `events_v2` 存储了所有原始事件数据 |
| **实时性** | 事件摄入后近实时写入 `events_v2` |
| **准确性** | `uniq(user_id)` 直接基于原始事件计算，无聚合误差 |
| **灵活性** | 可根据需要调整时间窗口和过滤条件 |

### 3.3 两条链路的关系图解

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           事件摄入管道                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   用户访问 → 跟踪脚本 → 事件接收 → 事件处理 → 写入 events_v2 表          │
│                                              │                            │
│                                              │ 发射 telemetry 事件         │
│                                              ▼                            │
│                                    ┌─────────────────┐                   │
│                                    │ Telemetry 事件  │                   │
│                                    │ (旁路监控)       │                   │
│                                    └────────┬────────┘                   │
│                                             │                             │
│                                             ▼                             │
│                                    ┌─────────────────┐                   │
│                                    │  Counters 链路   │                   │
│                                    │ (成本计数/监控)  │                   │
│                                    └────────┬────────┘                   │
│                                             │                             │
│                                             ▼                             │
│                                    ┌─────────────────┐                   │
│                                    │ ingest_counters │                   │
│                                    │ 表 (旁路存储)    │                   │
│                                    └─────────────────┘                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                           查询层                                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────┐         ┌─────────────────────────┐    │
│   │  实时访客接口链路        │         │  实时看板通用查询链路      │    │
│   │  (Current Visitors)     │         │  (Realtime Dashboard)    │    │
│   ├─────────────────────────┤         ├─────────────────────────┤    │
│   │                         │         │                         │    │
│   │  API: /current-visitors │         │  API: /query, /breakdown│    │
│   │  窗口: current_visitors │         │  窗口: query_builder    │    │
│   │        .ex (5分钟)       │         │        .ex (30分钟)      │    │
│   │                         │         │                         │    │
│   └───────────┬─────────────┘         └───────────┬─────────────┘    │
│               │                                     │                    │
│               └─────────────────┬───────────────────┘                    │
│                                 │                                          │
│                                 ▼                                          │
│                        ┌─────────────────┐                               │
│                        │   events_v2 表   │                               │
│                        │  (直接数据源)     │                               │
│                        └─────────────────┘                               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

关键关系:
✓ 实时访客接口 ←→ events_v2 (直接查询)
✓ 实时看板 ←→ events_v2 (通过 QueryBuilder 查询)
✗ 实时访客数 ←→ ingest_counters (无任何关系！)
✓ counters 链路 ←→ 旁路监控 (仅用于监控摄入管道健康)
```

### 3.4 关键结论

| 结论 | 说明 |
|-----|------|
| ✅ **实时访客数不使用 counters 数据** | `ingest_counters` 表与实时访客计算完全无关 |
| ✅ **events_v2 是唯一直接数据源** | 两条实时查询链路都直接查询 `events_v2` 表 |
| ✅ **counters 是旁路监控系统** | 仅用于监控事件摄入管道的健康状况 |
| ✅ **counters 指标含义不同** | 统计的是"事件数"，不是"访客数" |
| ⚠️ **容易混淆的命名** | `Counters` 模块名可能让人误解为与访客计数有关 |

---

## 4. 直接数据源 vs 旁路监控

### 4.1 直接数据源：events_v2 表

#### 4.1.1 表用途
存储所有原始事件数据，是**所有业务指标的唯一真实来源**。

#### 4.1.2 使用场景

| 使用场景 | 链路 | 查询方式 |
|---------|------|---------|
| 实时访客数 | 实时访客接口 | `current_visitors.ex` 直接查询 |
| 实时看板指标 | 实时看板通用查询 | `QueryBuilder` 构建查询 |
| 历史报表 | 所有历史查询 | 统一查询机制 |
| 自定义报表 | API 查询 | 统一查询机制 |

#### 4.1.3 数据特点

| 特点 | 说明 |
|-----|------|
| **数据完整性** | 包含所有原始事件字段 |
| **实时性** | 近实时写入 (摄入后立即可查) |
| **数据量** | 巨大 (每个页面浏览、事件都产生记录) |
| **查询成本** | 较高 (需要扫描大量数据) |
| **存储成本** | 较高 (需要保留历史数据) |

### 4.2 旁路监控：ingest_counters 表

#### 4.2.1 表用途
存储**摄入管道的内部监控指标**，用于：
- 监控事件摄入速率
- 追踪事件丢弃原因
- 分析跟踪脚本版本分布
- 容量规划和性能优化

#### 4.2.2 不使用场景

| ❌ 不用于 | 原因 |
|----------|------|
| 实时访客数 | 统计的是事件数，不是访客数 |
| 业务指标报表 | 数据粒度和维度不匹配 |
| 转化率计算 | 缺少用户级别的追踪 |
| 漏斗分析 | 缺少事件序列信息 |

#### 4.2.3 数据特点

| 特点 | 说明 |
|-----|------|
| **聚合粒度** | 分钟级聚合 (`event_timebucket`) |
| **数据量** | 小得多 (聚合后的数据) |
| **维度** | domain, metric, tracker_script_version |
| **查询目的** | 监控、告警、容量规划 |
| **存储引擎** | SummingMergeTree (支持自动聚合) |

### 4.3 为什么需要旁路监控？

#### 4.3.1 架构设计考虑

```
┌──────────────────────────────────────────────────────────────────────┐
│                        主数据流 (业务指标)                              │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   事件摄入 → events_v2 表 → 业务查询 (实时访客、报表等)                │
│                                                                      │
│   特点:                                                              │
│   • 数据完整                                                         │
│   • 查询灵活                                                         │
│   • 成本较高 (存储和查询)                                             │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                        旁路数据流 (监控指标)                            │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   事件摄入 → telemetry 事件 → 内存聚合 → ingest_counters 表 → 监控    │
│                                                                      │
│   特点:                                                              │
│   • 数据聚合 (分钟级)                                                 │
│   • 查询高效                                                         │
│   • 成本低廉                                                         │
│   • 不影响主数据流                                                    │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

#### 4.3.2 旁路监控的优势

| 优势 | 说明 |
|-----|------|
| **不影响主路径** | 通过 telemetry 事件异步收集，不阻塞事件摄入 |
| **成本可控** | 内存聚合 + 批量写入 + SummingMergeTree 自动聚合 |
| **实时监控** | 10 秒刷新周期，近实时反映管道健康状况 |
| **维度丰富** | 支持按 domain、metric、tracker_script_version 分析 |
| **故障排查** | `dropped_#{reason}` 指标帮助定位事件丢弃原因 |

---

## 5. 页面状态更新机制

### 5.1 实时访客组件的更新机制

#### 5.1.1 全局定时器

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

#### 5.1.2 组件事件监听

**位置**: `assets/js/dashboard/stats/current-visitors.js`

```javascript
useEffect(() => {
  document.addEventListener('tick', updateCount)

  return () => {
    document.removeEventListener('tick', updateCount)
  }
}, [updateCount])
```

#### 5.1.3 更新触发条件

| 触发条件 | 实现位置 | 说明 |
|---------|---------|------|
| 全局 `tick` 事件 | `useEffect` 监听 | 每 30 秒自动触发 |
| 仪表板状态变化 | `useEffect` 依赖 `dashboardState` | 筛选器、时间范围变化时 |
| 组件挂载 | 第二个 `useEffect` | 首次渲染时获取初始数据 |

### 5.2 事件驱动架构的优势

```
┌────────────────────────────────────────────────────────────┐
│                    事件驱动架构                              │
├────────────────────────────────────────────────────────────┤
│                                                            │
│   定时器模块 (realtime-update-timer.js)                   │
│              │                                             │
│              │ 每 30 秒                                    │
│              ▼                                             │
│   ┌─────────────────────┐                                 │
│   │ 全局 'tick' 事件     │                                 │
│   └──────────┬──────────┘                                 │
│              │                                             │
│     ┌────────┼────────┐                                   │
│     │        │        │                                   │
│     ▼        ▼        ▼                                   │
│   ┌─────┐ ┌─────┐ ┌─────┐                               │
│   │组件1│ │组件2│ │组件3│  (解耦：组件之间互不感知)       │
│   └─────┘ └─────┘ └─────┘                               │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

**优势**:
1. **解耦**：定时器与更新逻辑完全分离
2. **可扩展**：新组件只需监听 `tick` 事件即可
3. **统一控制**：更新频率在一处配置
4. **易于测试**：可以手动触发 `tick` 事件进行测试

---

## 6. 完整架构总结

### 6.1 系统架构全景图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              前端层                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────────────┐        ┌──────────────────────┐                 │
│  │ CurrentVisitors 组件  │        │   实时看板组件        │                 │
│  │ (导航栏指示器)        │        │   (所有图表/表格)     │                 │
│  ├──────────────────────┤        ├──────────────────────┤                 │
│  │ • 监听 'tick' 事件   │        │ • 依赖各自刷新机制    │                 │
│  │ • 30秒自动刷新       │        │ • 使用 period 参数    │                 │
│  │ • 调用专用 API       │        │ • 调用通用查询 API    │                 │
│  └──────────┬───────────┘        └──────────┬───────────┘                 │
│             │                                 │                              │
│             ▼                                 ▼                              │
│  ┌───────────────────────────────────────────────────────────────┐         │
│  │                    全局定时器 (30秒)                            │         │
│  │              realtime-update-timer.js                          │         │
│  └───────────────────────────────────────────────────────────────┘         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              API 层                                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────────────────────┐    ┌──────────────────────────────┐     │
│  │ GET /api/stats/{domain}/     │    │ 通用查询端点                  │     │
│  │     current-visitors         │    │ /api/stats/query, /breakdown  │     │
│  ├──────────────────────────────┤    ├──────────────────────────────┤     │
│  │ • 专用端点                   │    │ • 统一查询框架                │     │
│  │ • 无参数                     │    │ • 支持 period=realtime_30m   │     │
│  │ • 返回单个整数               │    │ • 支持复杂筛选和聚合          │     │
│  └──────────────┬───────────────┘    └──────────────┬───────────────┘     │
│                 │                                     │                      │
│                 ▼                                     ▼                      │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                    StatsController                                      │ │
│  │  • current_visitors/2 → Stats.current_visitors/1                       │ │
│  │  • query/2, breakdown/2 → QueryBuilder.build/3                        │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              业务逻辑层                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────────────────────┐    ┌──────────────────────────────┐     │
│  │  Stats.current_visitors/1    │    │    QueryBuilder.build/3      │     │
│  │  (stats.ex:30-32)            │    │    (query_builder.ex)        │     │
│  ├──────────────────────────────┤    ├──────────────────────────────┤     │
│  │ • 代理到 CurrentVisitors      │    │ • 统一查询构建               │     │
│  │ • 默认 5 分钟窗口             │    │ • 处理 :realtime_30m        │     │
│  │ • 支持自定义 duration         │    │ • 30 分钟窗口               │     │
│  └──────────────┬───────────────┘    └──────────────┬───────────────┘     │
│                 │                                     │                      │
│                 ▼                                     ▼                      │
│  ┌──────────────────────────────┐    ┌──────────────────────────────┐     │
│  │ CurrentVisitors.             │    │    统一 Query 执行            │     │
│  │   current_visitors/2         │    │                               │     │
│  │ (current_visitors.ex)        │    │                               │     │
│  ├──────────────────────────────┤    ├──────────────────────────────┤     │
│  │ • 独立窗口计算 (5分钟)        │    │ • query_builder 窗口计算      │     │
│  │ • 直接写 Ecto 查询           │    │ • 30分钟窗口                  │     │
│  │ • 不使用 QueryBuilder        │    │ • 完整的查询构建流程          │     │
│  └──────────────┬───────────────┘    └──────────────┬───────────────┘     │
│                 │                                     │                      │
│                 └─────────────────┬───────────────────┘                      │
│                                   │                                            │
│                                   ▼                                            │
│                        ┌──────────────────────┐                              │
│                        │   events_v2 表        │                              │
│                        │   (直接数据源)        │                              │
│                        └──────────────────────┘                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                              旁路监控层 (独立)                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                         事件摄入管道                                     │ │
│  │  Plausible.Ingestion.Event                                             │ │
│  └───────────────────────────────┬───────────────────────────────────────┘ │
│                                  │                                            │
│                                  │ 发射 telemetry 事件                        │
│                                  ▼                                            │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                   TelemetryHandler (telemetry_handler.ex)              │ │
│  │  • 监听 [:plausible, :ingestion, :event, :buffered/:dropped]          │ │
│  │  • 调用 Buffer.aggregate/5                                              │ │
│  └───────────────────────────────┬───────────────────────────────────────┘ │
│                                  │                                            │
│                                  ▼                                            │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                   Counters.Buffer (buffer.ex)                          │ │
│  │  • 内存聚合计数                                                         │ │
│  │  • 按 timebucket、metric、domain 维度聚合                               │ │
│  └───────────────────────────────┬───────────────────────────────────────┘ │
│                                  │                                            │
│                                  │ 每 10 秒刷新                               │
│                                  ▼                                            │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                   Counters.handle_cycle/2 (counters.ex:66-99)         │ │
│  │  • 刷新缓冲区                                                            │ │
│  │  • 转换为分钟级时间桶                                                    │ │
│  │  • 异步插入到数据库                                                      │ │
│  └───────────────────────────────┬───────────────────────────────────────┘ │
│                                  │                                            │
│                                  ▼                                            │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                   ingest_counters 表 (record.ex)                       │ │
│  │  • SummingMergeTree 引擎                                                │ │
│  │  • 自动聚合                                                              │ │
│  │  • 仅用于监控，不用于业务指标                                             │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 关键数据对照表

| 数据项 | 实时访客接口链路 | 实时看板通用查询链路 | 成本计数链路 |
|-------|-----------------|---------------------|-------------|
| **直接数据源** | `events_v2` | `events_v2` | 无 (旁路) |
| **存储表** | 无 (直接查询) | 无 (直接查询) | `ingest_counters` |
| **窗口大小** | 5 分钟 (默认) | 30 分钟 | 分钟级聚合 |
| **窗口计算位置** | `current_visitors.ex` | `query_builder.ex` | `counters.ex` |
| **刷新频率** | 30 秒 (前端) | 依赖组件 | 10 秒 (后端刷新) |
| **聚合方式** | `uniq(user_id)` | 多种指标 | `SummingMergeTree` |
| **用途** | 当前在线访客数 | 实时看板所有指标 | 摄入管道监控 |

### 6.3 常见误区澄清

| 误区 | 真相 |
|-----|------|
| ❌ `current_visitors.ex` 使用 `query_builder.ex` 的窗口计算 | ✅ 两套完全独立的实现 |
| ❌ 实时访客数使用 `ingest_counters` 表的数据 | ✅ 直接查询 `events_v2` 表 |
| ❌ `realtime` 和 `realtime_30m` 是同一回事 | ✅ 前端显示用 `realtime`，实际查询用 `realtime_30m` (30分钟) |
| ❌ 成本计数链路用于业务指标 | ✅ 仅用于监控摄入管道健康状况 |
| ❌ 所有实时查询都走统一 QueryBuilder | ✅ 实时访客接口是独立实现 |

---

## 7. 代码位置速查

### 7.1 实时访客接口链路

| 功能 | 文件路径 | 行号 |
|-----|---------|------|
| 控制器端点 | `lib/plausible_web/controllers/api/stats_controller.ex` | 1236-1239 |
| Stats 模块代理 | `lib/plausible/stats.ex` | 30-32 |
| 核心查询实现 | `lib/plausible/stats/current_visitors.ex` | 6-19 |
| 前端组件 | `assets/js/dashboard/stats/current-visitors.js` | 12-81 |
| 全局定时器 | `assets/js/dashboard/util/realtime-update-timer.js` | 1-8 |

### 7.2 实时看板通用查询链路

| 功能 | 文件路径 | 行号 |
|-----|---------|------|
| 前端周期定义 | `assets/js/dashboard/dashboard-time-periods.ts` | 28-43 |
| 前端周期转换 | `assets/js/dashboard/stats/graph/fetch-main-graph.ts` | 32 |
| 后端周期解析 | `lib/plausible/stats/dashboard/query_parser.ex` | 41 |
| 统一窗口计算 | `lib/plausible/stats/query_builder.ex` | 88-100 |

### 7.3 成本计数链路 (旁路监控)

| 功能 | 文件路径 | 行号 |
|-----|---------|------|
| 主模块 | `lib/plausible/ingestion/counters.ex` | 1-130 |
| 缓冲刷新逻辑 | `lib/plausible/ingestion/counters.ex` | 66-99 |
| Telemetry 处理器 | `lib/plausible/ingestion/counters/telemetry_handler.ex` | 18-64 |
| 数据结构定义 | `lib/plausible/ingestion/counters/record.ex` | 10-17 |
| 缓冲模块 | `lib/plausible/ingestion/counters/buffer.ex` | 完整文件 |

---

## 8. 总结

### 8.1 核心结论

1. **两条独立的实时查询链路**：
   - 实时访客接口链路：专用、简单、5分钟窗口、独立实现
   - 实时看板通用查询链路：通用、复杂、30分钟窗口、统一 QueryBuilder

2. **窗口计算归属明确**：
   - `current_visitors.ex` 有自己独立的窗口计算（默认5分钟）
   - `query_builder.ex` 中的窗口计算用于实时看板（30分钟）
   - 两套实现完全独立，没有依赖关系

3. **成本计数链路是旁路监控**：
   - 与实时访客数计算**完全无关**
   - 仅用于监控事件摄入管道的健康状况
   - 数据存储在 `ingest_counters` 表，不用于业务指标

4. **直接数据源是 events_v2 表**：
   - 两条实时查询链路都直接查询 `events_v2` 表
   - `ingest_counters` 是旁路存储，仅用于监控

### 8.2 设计权衡

| 设计决策 | 优势 | 劣势 |
|---------|------|------|
| 两套独立实时链路 | 实时访客接口简单高效 | 维护成本高，行为可能不一致 |
| 旁路监控架构 | 不影响主路径，成本可控 | 需要维护额外的监控系统 |
| 30秒刷新频率 | 平衡实时性和资源消耗 | 对于某些场景可能不够实时 |
| 5分钟 vs 30分钟窗口 | 各场景针对性优化 | 用户可能困惑于差异 |

### 8.3 未来优化建议

1. **统一窗口计算**：考虑将实时访客接口迁移到统一的 QueryBuilder 框架
2. **明确窗口差异**：在 UI 中明确显示不同实时视图的窗口大小
3. **监控告警**：利用 `ingest_counters` 数据建立自动告警机制
4. **性能优化**：考虑为高频的实时访客查询添加缓存层

---

**报告完成时间**: 2026-05-03

**分析范围**: Plausible Analytics 实时访客指标刷新机制
