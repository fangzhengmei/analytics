# 实时访客指标刷新机制分析报告

## 概述

本报告详细分析了 Plausible Analytics 系统中实时访客指标的刷新机制，重点关注以下四个核心方面：
1. 实时窗口管理
2. 聚合查询实现
3. 页面状态更新
4. 成本控制策略

---

## 1. 实时窗口管理

### 1.1 窗口大小配置

系统支持两种实时窗口大小：

| 窗口类型 | 时长 | 定义位置 |
|---------|------|---------|
| 默认实时窗口 | 5 分钟 | `current_visitors.ex:6` |
| 扩展实时窗口 | 30 分钟 | `query_builder.ex:92-94` |

### 1.2 窗口计算逻辑

在 `lib/plausible/stats/query_builder.ex:88-100` 中实现了实时时间范围的构建：

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

**关键点分析**：
- **起始时间**：`now - duration_minutes` - 从当前时间向前推指定分钟数
- **结束时间**：`now + 5 seconds` - 包含一个小的缓冲时间，确保最新数据被包含
- **时间精度**：使用 UTC 时间，确保跨时区一致性

### 1.3 窗口滑动机制

在 `lib/plausible/stats/current_visitors.ex:6-19` 中实现了实时访客查询的窗口滑动：

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

**窗口特性**：
- **滑动窗口**：每次查询都基于当前时间重新计算窗口
- **动态窗口**：窗口随时间自然滑动，无需显式维护
- **默认 5 分钟**：默认使用 5 分钟的滑动窗口

---

## 2. 聚合查询实现

### 2.1 查询架构

实时访客查询采用了以下架构层次：

```
前端组件 (current-visitors.js)
    ↓ API 调用
API 控制器 (stats_controller.ex:1236-1239)
    ↓ 调用
统计模块 (plausible/stats.ex)
    ↓ 调用
当前访客模块 (current_visitors.ex)
    ↓ 执行
ClickHouse 查询
```

### 2.2 核心查询逻辑

在 `lib/plausible/stats/current_visitors.ex` 中实现了核心查询：

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

**查询分析**：
- **数据源**：`events_v2` 表 - 存储所有事件数据
- **过滤条件**：
  1. `site_id_query_filter(site)` - 按站点过滤
  2. `timestamp >= ^first_datetime` - 时间窗口过滤
  3. `name != "engagement"` - 排除互动事件
- **聚合方式**：`uniq(e.user_id)` - 统计唯一用户数
- **返回值**：单个整数值 - 当前访客数量

### 2.3 查询构建器支持

在 `lib/plausible/stats/query_builder.ex` 中，实时窗口的构建被集成到统一的查询构建流程中：

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

**设计优势**：
- **统一接口**：实时窗口与其他时间范围（日、月、年等）使用相同的构建逻辑
- **可扩展性**：易于添加新的实时窗口大小
- **时间缓冲**：`last_datetime = now + 5 seconds` 确保最新数据被包含

### 2.4 API 端点

在 `lib/plausible_web/controllers/api/stats_controller.ex:1236-1239` 中定义了实时访客 API：

```elixir
def current_visitors(conn, _) do
  site = conn.assigns[:site]
  json(conn, Stats.current_visitors(site))
end
```

**端点特性**：
- **路径**：`/api/stats/{domain}/current-visitors`
- **方法**：GET
- **参数**：无（从上下文获取站点信息）
- **返回**：JSON 格式的整数 - 当前访客数量

---

## 3. 页面状态更新

### 3.1 更新架构

前端实时更新采用了事件驱动架构：

```
定时器模块 (realtime-update-timer.js)
    ↓ 每 30 秒触发
全局 'tick' 事件
    ↓ 组件监听
CurrentVisitors 组件 (current-visitors.js)
    ↓ API 调用
获取最新数据
    ↓ 更新状态
React 状态更新
    ↓ 渲染
UI 刷新
```

### 3.2 定时器实现

在 `assets/js/dashboard/util/realtime-update-timer.js` 中实现了全局定时器：

```javascript
export const REALTIME_UPDATE_TIME_MS = 30_000
const tickEvent = new Event('tick')

export function start() {
  setInterval(() => {
    document.dispatchEvent(tickEvent)
  }, REALTIME_UPDATE_TIME_MS)
}
```

**定时器分析**：
- **更新频率**：30 秒（`REALTIME_UPDATE_TIME_MS = 30_000`）
- **事件机制**：使用自定义 `tick` 事件，解耦定时器与更新逻辑
- **全局通知**：通过 `document.dispatchEvent` 通知所有监听组件

### 3.3 组件更新逻辑

在 `assets/js/dashboard/stats/current-visitors.js` 中实现了组件更新：

```javascript
export default function CurrentVisitors({ className = '' }) {
  const { dashboardState } = useDashboardStateContext()
  const lastLoadTimestamp = useLastLoadContext()
  const site = useSiteContext()
  const [currentVisitors, setCurrentVisitors] = useState(null)

  const updateCount = useCallback(() => {
    api
      .get(`/api/stats/${encodeURIComponent(site.domain)}/current-visitors`)
      .then((res) => setCurrentVisitors(res))
  }, [site.domain])

  useEffect(() => {
    document.addEventListener('tick', updateCount)

    return () => {
      document.removeEventListener('tick', updateCount)
    }
  }, [updateCount])

  useEffect(() => {
    updateCount()
  }, [dashboardState, updateCount])
  
  // 渲染逻辑...
}
```

**更新机制分析**：

#### 3.3.1 事件监听机制
- **生命周期**：组件挂载时添加监听，卸载时移除监听
- **事件源**：监听全局 `tick` 事件
- **回调函数**：`updateCount` - 执行 API 调用并更新状态

#### 3.3.2 状态更新触发条件

| 触发条件 | 实现位置 | 说明 |
|---------|---------|------|
| 定时器事件 | `useEffect` 监听 `tick` | 每 30 秒自动更新 |
| 仪表板状态变化 | `useEffect` 依赖 `dashboardState` | 筛选器、时间范围等变化时更新 |
| 组件挂载 | 第二个 `useEffect` 初始调用 | 首次渲染时获取数据 |

#### 3.3.3 状态管理
- **状态变量**：`currentVisitors` - 存储当前访客数量
- **初始值**：`null` - 表示未加载
- **更新函数**：`setCurrentVisitors` - 从 API 响应更新状态

### 3.4 渲染逻辑

组件渲染逻辑包含以下特性：

```javascript
if (currentVisitors !== null && dashboardState.filters.length === 0) {
  return (
    <Tooltip
      info={
        <div>
          <p className="whitespace-nowrap text-small">
            Last updated{' '}
            <SecondsSinceLastLoad lastLoadTimestamp={lastLoadTimestamp} />s
            ago
          </p>
          <p className="whitespace-nowrap font-normal text-xs">
            Click to view realtime dashboard
          </p>
        </div>
      }
    >
      <AppNavigationLink
        search={(prev) => ({ ...prev, period: 'realtime' })}
        // ... 样式和内容
      >
        <svg ... /> {/* 绿色状态指示器 */}
        <div>
          {currentVisitors}
          <span> current visitor{currentVisitors === 1 ? '' : 's'}</span>
        </div>
      </AppNavigationLink>
    </Tooltip>
  )
} else {
  return null
}
```

**渲染特性**：
- **条件渲染**：只在有数据且无筛选器时显示
- **实时状态指示器**：绿色圆点表示实时状态
- **工具提示**：显示最后更新时间和操作提示
- **导航链接**：点击可跳转到实时仪表板
- **单复数处理**：`visitor` vs `visitors`

### 3.5 更新频率权衡

**为什么选择 30 秒更新频率？**

1. **用户体验**：30 秒是实时性与资源消耗的平衡点
2. **API 负载**：避免过于频繁的请求对服务器造成压力
3. **数据新鲜度**：对于大多数实时监控场景，30 秒延迟是可接受的
4. **与后端窗口匹配**：与 5 分钟的统计窗口相比，30 秒更新频率足够精细

---

## 4. 成本控制策略

### 4.1 成本控制架构

系统采用了多层次的成本控制策略：

```
事件采集层
    ↓ 缓冲聚合
计数器缓冲 (Counters.Buffer)
    ↓ 定时刷新
计数器模块 (Counters)
    ↓ 异步插入
AsyncInsertRepo
    ↓ ClickHouse
SummingMergeTree 表
    ↓ 自动聚合
ClickHouse 后台合并
```

### 4.2 计数器模块实现

在 `lib/plausible/ingestion/counters.ex` 中实现了成本控制的核心逻辑：

```elixir
defmodule Plausible.Ingestion.Counters do
  @moduledoc """
  This is instrumentation necessary for keeping track of per-domain
  internal metrics. Due to metric labels cardinality (domain x metric_name),
  these statistics are not suitable for prometheus/grafana exposure,
  hence an internal storage is used.
  """

  @behaviour :gen_cycle
  @interval :timer.seconds(10)

  # ... 实现代码
end
```

### 4.3 缓冲聚合机制

#### 4.3.1 模块设计

计数器模块采用了以下设计模式：

| 组件 | 职责 | 文件位置 |
|-----|------|---------|
| `Counters` | 主模块，管理生命周期和刷新逻辑 | `counters.ex` |
| `Counters.Buffer` | 内存缓冲，聚合计数器数据 | `counters/buffer.ex` |
| `Counters.Record` | 数据记录结构 | `counters/record.ex` |
| `Counters.TelemetryHandler` | 遥测事件处理器 | `counters/telemetry_handler.ex` |
| `AsyncInsertRepo` | 异步插入仓库 | 全局配置 |

#### 4.3.2 刷新周期

```elixir
@interval :timer.seconds(10)
```

- **刷新频率**：每 10 秒刷新一次缓冲区
- **实现方式**：使用 `:gen_cycle` 行为实现周期性任务
- **优势**：减少数据库写入次数，降低 I/O 成本

#### 4.3.3 缓冲刷新逻辑

在 `lib/plausible/ingestion/counters.ex:66-99` 中实现了缓冲刷新：

```elixir
@impl true
def handle_cycle(buffer, now \\ DateTime.utc_now()) do
  case Buffer.flush(buffer, now) do
    [] ->
      :noop

    records ->
      records =
        Enum.map(records, fn {bucket, metric, domain, tracker_script_version, value} ->
          %{
            event_timebucket: to_0_minute_datetime(bucket),
            metric: metric,
            site_id: Plausible.Site.Cache.get_site_id(domain),
            domain: domain,
            tracker_script_version: tracker_script_version,
            value: value
          }
        end)

      try do
        {_, _} = AsyncInsertRepo.insert_all(Record, records)
      catch
        _, thrown ->
          Sentry.capture_message(
            "Caught an error when trying to flush ingest counters.",
            extra: %{
              number_of_records: Enum.count(records),
              error: inspect(thrown)
            }
          )
      end
  end

  {:continue_hibernated, buffer}
end
```

**刷新流程分析**：

1. **缓冲区刷新**：调用 `Buffer.flush(buffer, now)` 获取待写入记录
2. **空缓冲区处理**：如果没有记录，执行 `:noop` 不做任何操作
3. **数据转换**：
   - 将时间桶转换为分钟精度（`to_0_minute_datetime`）
   - 解析站点 ID（`Site.Cache.get_site_id`）
   - 构建完整的记录结构
4. **异步插入**：使用 `AsyncInsertRepo.insert_all` 异步写入数据库
5. **错误处理**：捕获异常并记录到 Sentry
6. **状态保持**：返回 `{:continue_hibernated, buffer}` 保持状态并休眠

### 4.4 数据粒度优化

#### 4.4.1 时间粒度转换

```elixir
defp to_0_minute_datetime(unix_ts) when is_integer(unix_ts) do
  unix_ts
  |> DateTime.from_unix!()
  |> DateTime.truncate(:second)
  |> Map.replace(:second, 0)
end
```

**粒度优化策略**：
- **输入**：Unix 时间戳（秒级精度）
- **处理**：
  1. 转换为 DateTime
  2. 截断到秒级
  3. 将秒数设置为 0（分钟级精度）
- **输出**：分钟级精度的时间桶

**成本优势**：
- **减少数据量**：分钟级精度比秒级精度减少 60 倍数据量
- **更好的聚合**：与 ClickHouse 的 `SummingMergeTree` 引擎配合，实现自动聚合
- **查询效率**：更粗的时间粒度意味着更少的查询扫描范围

#### 4.4.2 ClickHouse 引擎选择

根据模块注释：

```elixir
@moduledoc """
The underlying database schema is running `SummingMergeTree` engine.
To take advantage of automatic roll-ups it provides, upon dispatching the
buffered records to Clickhouse this module transforms each `event_timebucket`
aggregate into a 1-minute resolution.
"""
```

**SummingMergeTree 优势**：
- **自动聚合**：ClickHouse 后台自动合并相同主键的行
- **空间效率**：聚合后的数据占用更少存储空间
- **查询性能**：查询时直接读取聚合结果，无需实时计算

### 4.5 异步写入机制

#### 4.5.1 AsyncInsertRepo 配置

根据模块注释：

```elixir
@moduledoc """
Clickhouse connection is set to insert counters asynchronously every time
a pool checkout is made. Those properties are reverted once the insert is done
(or naturally, if the connection crashes).
"""
```

**异步写入特性**：
- **非阻塞**：写入操作不阻塞主流程
- **连接池优化**：每次从连接池获取连接时设置异步插入
- **自动恢复**：插入完成或连接崩溃后自动恢复连接属性

#### 4.5.2 写入操作

```elixir
{_, _} = AsyncInsertRepo.insert_all(Record, records)
```

- **批量插入**：使用 `insert_all` 批量写入多条记录
- **无需返回**：不关心返回值，只关心是否成功
- **异常捕获**：使用 `try/catch` 捕获所有可能的异常

### 4.6 优雅停机处理

在 `lib/plausible/ingestion/counters.ex:111-116` 中实现了优雅停机：

```elixir
@impl true
def terminate(_reason, buffer) do
  # we'll travel in time to flush everything regardless of current bucket completion
  future = DateTime.utc_now() |> DateTime.add(60, :second)
  handle_cycle(buffer, future)
  :ok
end
```

**停机策略**：
- **时间旅行**：将当前时间向前推进 60 秒
- **强制刷新**：触发 `handle_cycle` 刷新所有缓冲区
- **数据完整性**：确保停机前所有缓冲数据都被写入数据库

**为什么使用 60 秒？**
- 与分钟级时间粒度匹配
- 确保所有当前时间桶的数据都被视为"完成"
- 避免部分写入导致的数据不一致

### 4.7 成本控制总结

| 策略 | 实现方式 | 成本节约效果 |
|-----|---------|------------|
| 缓冲聚合 | 10 秒刷新周期，内存缓冲 | 减少数据库写入次数 |
| 时间粒度优化 | 分钟级时间桶 | 减少 60 倍数据量 |
| 异步写入 | AsyncInsertRepo | 非阻塞，提高吞吐量 |
| 数据库引擎 | SummingMergeTree | 自动聚合，减少存储 |
| 错误处理 | Sentry 监控 + 异常捕获 | 避免数据丢失 |
| 优雅停机 | terminate 回调强制刷新 | 确保数据完整性 |

---

## 5. 完整数据流

### 5.1 数据采集到展示的完整流程

```
1. 用户访问网站
    ↓
2. 跟踪脚本发送事件
    ↓
3. 事件被摄入系统处理
    ↓
4. 事件写入 ClickHouse events_v2 表
    ↓
5. 同时，计数器模块缓冲内部指标
    ↓ 每 10 秒
6. 计数器刷新到 SummingMergeTree 表
    ↓ ClickHouse 后台
7. SummingMergeTree 自动聚合数据
    ↓ 每 30 秒
8. 前端定时器触发 'tick' 事件
    ↓
9. CurrentVisitors 组件监听事件
    ↓
10. 调用 API: /api/stats/{domain}/current-visitors
    ↓
11. StatsController 调用 Stats.current_visitors
    ↓
12. CurrentVisitors 模块执行 ClickHouse 查询
    ↓
13. 查询返回唯一用户数 (uniq(user_id))
    ↓
14. API 返回 JSON 响应
    ↓
15. 组件更新 React 状态
    ↓
16. UI 重新渲染，显示最新访客数
```

### 5.2 关键时间节点

| 操作 | 频率/延迟 | 位置 |
|-----|----------|------|
| 事件采集 | 实时 | 跟踪脚本 |
| 事件写入 | 近实时 | 摄入管道 |
| 计数器刷新 | 每 10 秒 | Counters 模块 |
| 前端更新 | 每 30 秒 | realtime-update-timer.js |
| 实时窗口 | 滑动 5 分钟 | current_visitors.ex |
| 数据聚合 | 后台持续 | ClickHouse SummingMergeTree |

---

## 6. 技术架构总结

### 6.1 架构优势

1. **分层设计**：
   - 数据层：ClickHouse 提供高性能存储和查询
   - 服务层：Elixir 模块提供业务逻辑抽象
   - 展示层：React 组件提供用户界面

2. **事件驱动**：
   - 前端使用自定义事件解耦定时器与更新逻辑
   - 后端使用 OTP 行为实现周期性任务

3. **成本优化**：
   - 多层缓冲减少数据库压力
   - 时间粒度优化减少数据量
   - 异步写入提高吞吐量

4. **实时与平衡**：
   - 30 秒更新频率平衡实时性与资源消耗
   - 5 分钟滑动窗口提供有意义的实时指标

### 6.2 关键技术选型

| 技术 | 用途 | 选型理由 |
|-----|------|---------|
| ClickHouse | 数据存储 | 列式存储，高性能聚合查询 |
| SummingMergeTree | 计数器表引擎 | 自动聚合，减少存储空间 |
| Elixir/OTP | 后端语言 | 并发能力强，适合实时系统 |
| React | 前端框架 | 组件化，状态管理清晰 |
| 事件驱动 | 前端更新机制 | 解耦，易于扩展 |

### 6.3 可扩展性考虑

1. **水平扩展**：
   - ClickHouse 支持集群部署
   - Elixir 应用可以水平扩展
   - 无状态 API 设计支持负载均衡

2. **功能扩展**：
   - 实时窗口大小可配置（5 分钟、30 分钟）
   - 组件化设计易于添加新的实时指标
   - 事件驱动架构支持多个组件监听更新

3. **性能优化**：
   - 可调整刷新频率（10 秒、30 秒）
   - 可调整时间粒度（分钟级、小时级）
   - 可增加缓存层减少数据库查询

---

## 7. 结论

Plausible Analytics 的实时访客指标刷新机制是一个经过精心设计的系统，在以下四个方面达到了优秀的平衡：

### 7.1 实时窗口管理
- **滑动窗口**：5 分钟默认窗口，支持 30 分钟扩展
- **动态计算**：每次查询基于当前时间重新计算
- **时间缓冲**：+5 秒缓冲确保最新数据被包含

### 7.2 聚合查询实现
- **简单高效**：`uniq(user_id)` 聚合，条件过滤清晰
- **统一接口**：与其他时间范围使用相同的查询构建器
- **API 友好**：RESTful 端点，易于集成

### 7.3 页面状态更新
- **事件驱动**：30 秒定时器触发全局 `tick` 事件
- **多条件触发**：定时器、状态变化、组件挂载
- **用户体验**：实时状态指示器，更新时间提示

### 7.4 成本控制策略
- **多层缓冲**：10 秒刷新周期，内存聚合
- **粒度优化**：分钟级时间桶，减少数据量
- **异步写入**：非阻塞操作，提高吞吐量
- **智能存储**：SummingMergeTree 自动聚合
- **优雅停机**：确保数据完整性

### 7.5 整体评价

该系统在**实时性**、**准确性**、**性能**和**成本**之间找到了最佳平衡点：

- **对于用户**：30 秒更新频率提供足够的实时感知
- **对于业务**：5 分钟窗口提供有意义的访客统计
- **对于运维**：多层成本控制确保系统可扩展
- **对于开发**：清晰的架构和模块化设计易于维护

这种设计思路值得在其他需要实时指标的系统中借鉴。

---

## 附录：关键代码位置速查

| 功能 | 文件路径 | 行号范围 |
|-----|---------|---------|
| 实时访客查询 | `lib/plausible/stats/current_visitors.ex` | 1-20 |
| 实时时间范围构建 | `lib/plausible/stats/query_builder.ex` | 88-100 |
| API 端点 | `lib/plausible_web/controllers/api/stats_controller.ex` | 1236-1239 |
| 前端组件 | `assets/js/dashboard/stats/current-visitors.js` | 1-81 |
| 全局定时器 | `assets/js/dashboard/util/realtime-update-timer.js` | 1-8 |
| 计数器模块 | `lib/plausible/ingestion/counters.ex` | 1-130 |
| 缓冲刷新逻辑 | `lib/plausible/ingestion/counters.ex` | 66-99 |
| 优雅停机 | `lib/plausible/ingestion/counters.ex` | 111-116 |
