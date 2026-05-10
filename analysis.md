# Plausible Analytics 仪表盘查询分析报告

## 1. 缓存策略分析

### 1.1 前端 React Query 缓存层级

Plausible Analytics 在前端使用 `@tanstack/react-query` 管理查询缓存，根据时间范围类型使用不同的缓存过期（staleTime）策略：

| 策略名称 | 缓存时长 | 适用场景 | 代码位置 |
|---------|---------|---------|---------|
| `CACHE_TTL_REALTIME` | 30 秒 | 实时仪表盘 (`realtime` / `realtime_30m`) | `assets/js/dashboard/hooks/api-client.ts:20` |
| `CACHE_TTL_SHORT_ONGOING` | 5 分钟 | 包含今天且支持 `day`/`hour`/`minute` 粒度的时间段 | `assets/js/dashboard/hooks/api-client.ts:21` |
| `CACHE_TTL_LONG_ONGOING` | 1 小时 | 包含今天但不支持短粒度的时间段（如年度视图） | `assets/js/dashboard/hooks/api-client.ts:22` |
| `CACHE_TTL_HISTORICAL` | 12 小时 | 不包含今天的历史时间段 | `assets/js/dashboard/hooks/api-client.ts:23` |

### 1.2 缓存过期决策逻辑

核心逻辑位于 `getStaleTime` 函数 (`assets/js/dashboard/hooks/api-client.ts:188-212`)：

```typescript
export const getStaleTime = (props: DashboardTimeSettings): number => {
  // 1. 实时模式：30秒
  if ([DashboardPeriod.realtime, DashboardPeriod.realtime_30m].includes(props.period)) {
    return CACHE_TTL_REALTIME  // 30_000ms
  }

  // 2. 历史数据（不包含今天）：12小时
  if (isHistoricalPeriod(props)) {
    return CACHE_TTL_HISTORICAL  // 12h
  }

  // 3. 包含今天的数据
  const availableIntervals = validIntervals(props)
  
  // 3a. 支持 day/hour/minute 粒度：5分钟
  if (availableIntervals.includes(Interval.day) ||
      availableIntervals.includes(Interval.hour) ||
      availableIntervals.includes(Interval.minute)) {
    return CACHE_TTL_SHORT_ONGOING  // 5m
  } 
  // 3b. 不支持短粒度（如年度视图）：1小时
  else {
    return CACHE_TTL_LONG_ONGOING  // 1h
  }
}
```

### 1.3 后端采样缓存 (EE Only)

在企业版中，Plausible 使用 `SamplingCache` 缓存站点过去 30 天的事件摄入估算值，用于决定是否对查询进行采样：

**缓存结构** (`extra/lib/plausible/stats/sampling_cache.ex`):
- 缓存名称: `:stats_sampling_cache`
- 数据来源: `Ingestion.Counters.Record` (ClickHouse)
- 查询频率: 定期刷新（`refresh_all` / `refresh_updated_recently`）
- 键: `site_id`
- 值: 过去 30 天的事件总数

**查询逻辑**:
```elixir
# 过去30天按 site_id 聚合事件数
from(r in Ingestion.Counters.Record,
  select: {
    r.site_id,
    selected_as(fragment("sumIf(value, metric = 'buffered')"), :events_ingested)
  },
  where: fragment("toDate(event_timebucket) >= ?", ^thirty_days_ago()),
  group_by: r.site_id
)
```

---

## 2. 采样（降级）机制分析

### 2.1 采样触发条件

采样逻辑位于 `extra/lib/plausible/stats/sampling.ex`，基于以下因素决定采样率：

1. **站点 30 天流量估算** (`traffic_30_day`)
   - 从 `SamplingCache` 获取
   - 对于聚合站点（consolidated），累加所有子站点的流量

2. **查询时间范围** (`duration`)
   - 按比例估算查询范围内的流量
   - 公式: `duration_adjusted_traffic = traffic_30_day / 30.0 * duration`

3. **过滤器数量**
   - 每个过滤器应用 1/4 的流量乘数（假设过滤器缩小 75% 数据量）
   - 最多考虑 2 个过滤器
   - 公式: `estimation * (1/4) ** min(filters_count, 2)`

4. **默认采样阈值**: 10,000,000 行

### 2.2 采样率计算算法

```
fraction = default_sample_threshold(10M) / estimated_traffic

决策规则:
- 如果 duration < 1 天: 不采样 (:no_sampling)
- 如果 fraction > 0.4: 不采样（采样效果不显著）
- 否则: 使用 max(fraction, 0.013) — 最小 1.3% 采样率
```

**测试用例验证** (`test/plausible/stats/sampling_test.exs`):

| 30天流量 | 时间范围 | 预期采样率 |
|---------|---------|-----------|
| 7.5M | 100天 | 0.4 (不采样临界) |
| 7.5M | 364天 | 0.11 |
| 7.5M | 900天 | 0.04 |
| 7.5M | 2900天 | 0.013 (最小值) |
| 50M (5×阈值) | 30天, 2个过滤器 | 0.32 (不采样) |
| 500M (50×阈值) | 30天 | 0.013 (最小值) |

### 2.3 ClickHouse 查询提示

当启用采样时，SQL 查询会添加 `SAMPLE <threshold>` 提示：

```elixir
# extra/lib/plausible/stats/sql/query_builder.ex:44,64,96,122
on_ee do
  q = Plausible.Stats.Sampling.add_query_hint(q, query)
end
```

---

## 3. 实时刷新机制分析

### 3.1 刷新触发机制

实时刷新通过全局定时器触发：

**定时器** (`assets/js/dashboard/util/realtime-update-timer.js`):
```javascript
export const REALTIME_UPDATE_TIME_MS = 30_000

export function start() {
  setInterval(() => {
    document.dispatchEvent(new Event('tick'))
  }, REALTIME_UPDATE_TIME_MS)
}
```

**各组件监听刷新**：

1. **主图表和顶栏统计** (`assets/js/dashboard/stats/graph/visitor-graph.tsx:156-178`)
   ```typescript
   useEffect(() => {
     const onTick = () => {
       setIsRealtimeSilentUpdate({ topStats: true, mainGraph: true })
       queryClient.invalidateQueries({
         predicate: ({ queryKey }) => {
           return ['top-stats', 'main-graph'].includes(queryKey[0]) &&
             queryKey[1]?.dashboardState?.period === DashboardPeriod.realtime
         }
       })
     }
     // 监听 'tick' 事件
   }, [queryClient, isRealtime])
   ```

2. **Breakdown 组件** (`assets/js/dashboard/stats/reports/index-breakdown.tsx:121-138`)
   - 类似逻辑，仅使实时相关的 breakdown 查询失效

### 3.2 缓存命中对实时刷新的影响

**关键设计：实时数据不依赖缓存命中**

实时模式下的缓存策略：
1. **staleTime = 30秒**：与刷新间隔完全一致
2. **主动失效**：每 30 秒通过 `queryClient.invalidateQueries` 主动使缓存失效
3. **静默更新**：使用 `isRealtimeSilentUpdate` 标志避免显示 loading spinner

**用户体验效果**：
- 初次加载：显示完整加载动画
- 后续刷新（每 30 秒）：静默更新数据，不显示加载状态
- 缓存中的旧数据继续显示，新数据到达后无缝替换

**潜在问题**：
- 如果网络请求超过 30 秒，可能出现请求堆积
- 但前端使用 `AbortController` 取消旧请求 (`assets/js/dashboard/api.ts:9,68-71`)

---

## 4. 大查询保护机制

### 4.1 多级保护体系

| 保护层级 | 机制 | 代码位置 |
|---------|------|---------|
| **L1: 查询验证** | 时间维度粒度限制、指标/维度冲突检查 | `lib/plausible/stats/query_builder.ex:333-584` |
| **L2: 采样降级** | 基于流量估算的 ClickHouse SAMPLE | `extra/lib/plausible/stats/sampling.ex` |
| **L3: 分页限制** | 前端分页每页 100 条 | `assets/js/dashboard/hooks/api-client.ts:26` |
| **L4: 缓存层** | 前端 React Query 缓存减少重复查询 | `assets/js/dashboard/hooks/api-client.ts` |
| **L5: 请求取消** | 路由变化时取消所有进行中的请求 | `assets/js/dashboard/api.ts:68-71` |

### 4.2 详细保护规则

#### 4.2.1 时间粒度限制 (`validate_time_dimension_granularity`)
```elixir
# time:minute 维度仅支持最多 30 小时范围
@max_hours_for_minute_interval 30

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

#### 4.2.2 指标-维度兼容性检查
```elixir
# 示例：conversion_rate 只能与 event:goal 一起使用
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
```

#### 4.2.3 行为过滤器嵌套限制
```elixir
# 行为过滤器 (has_done, has_not_done) 不能嵌套超过 1 层
# 且只能用于 event: 维度
defp validate_behavioral_filters(query) do
  # ... 检查 depth > 1 时报错
end
```

### 4.3 查询优化器流水线

`QueryOptimizer.optimize/1` (`lib/plausible/stats/query_optimizer.ex:32-64`) 在执行前应用以下优化：

```
流水线步骤:
1. update_group_by_time          - 根据时间范围自动选择粒度 (hour/day/week/month)
2. add_missing_order_by          - 添加默认排序
3. update_time_in_order_by       - 更新排序中的时间维度
4. extend_hostname_filters_to_visit - 跨表 filter 同步
5. set_time_on_page_data         - 页面停留时间数据准备
6. remove_time_on_page_if_unavailable - 移除不可用的 metrics
7. remove_revenue_metrics_if_unavailable - 移除不可用的收入指标
8. trim_relative_date_range      - 截断未来的时间桶
9. set_sql_join_type             - 设置 JOIN 类型 (left/full)
```

**自动粒度选择逻辑** (`update_group_by_time`):
```elixir
defp resolve_time_dimension(first, last) do
  cond do
    DateTime.diff(last, first, :hour) <= 48 -> "time:hour"
    DateTime.diff(last, first, :day) <= 40 -> "time:day"
    Plausible.Times.diff(last, first, :week) <= 52 -> "time:week"
    true -> "time:month"
  end
end
```

---

## 5. 数据流架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        前端 (React + React Query)                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────┐   30s tick   ┌───────────────┐                    │
│  │ 实时定时器    │─────────────▶│  QueryClient  │                    │
│  │ (setInterval)│              │ invalidateQueries()               │
│  └──────────────┘              └───────┬───────┘                    │
│                                        │                            │
│                                        ▼                            │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │                    React Query Cache                        │     │
│  │  ┌─────────┬─────────┬─────────┬─────────────────────────┐ │     │
│  │  │ realtime│ short   │ long    │ historical              │ │     │
│  │  │  30s    │  5min   │  1hour  │  12hour                │ │     │
│  │  └─────────┴─────────┴─────────┴─────────────────────────┘ │     │
│  └────────────────────────────────────────────────────────────┘     │
│                                        │                            │
│                                        ▼ 未命中/过期                 │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │                    API 请求 (/query)                        │     │
│  │  - POST JSON body: {date_range, filters, metrics, ...}    │     │
│  │  - AbortController: 路由变化时取消                          │     │
│  └────────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼ HTTP
┌─────────────────────────────────────────────────────────────────────┐
│                         后端 (Elixir + ClickHouse)                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │                    QueryBuilder.build()                      │     │
│  │  1. 解析 params → ParsedQueryParams                         │     │
│  │  2. 验证: 粒度/指标/过滤器/权限                               │     │
│  │  3. EE: Sampling.put_threshold() 决定采样率                  │     │
│  └───────────────────────────┬────────────────────────────────┘     │
│                              │                                       │
│                              ▼                                       │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │                    QueryOptimizer.optimize()                │     │
│  │  - 自动时间粒度选择                                          │     │
│  │  - 去除不可用 metrics                                        │     │
│  │  - 截断未来时间桶                                            │     │
│  └───────────────────────────┬────────────────────────────────┘     │
│                              │                                       │
│                              ▼                                       │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │                    SamplingCache (EE Only)                  │     │
│  │  - 缓存各站点过去 30 天事件数                                 │     │
│  │  - 用于计算采样率 fraction                                   │     │
│  └───────────────────────────┬────────────────────────────────┘     │
│                              │                                       │
│                              ▼                                       │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │                    SQL.QueryBuilder.build()                 │     │
│  │  - 拆分 events/sessions 子查询                               │     │
│  │  - EE: 添加 SAMPLE <fraction> hint                          │     │
│  │  - 合并 imported 数据                                       │     │
│  └───────────────────────────┬────────────────────────────────┘     │
│                              │                                       │
│                              ▼                                       │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │                    ClickHouse 执行                          │     │
│  │  - events_v2 / sessions_v2 表                               │     │
│  │  - SAMPLE 子句: 按比例采样行                                 │     │
│  │  - 聚合 + GROUP BY + ORDER BY + LIMIT                       │     │
│  └────────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 6. 关键结论

### 6.1 缓存 vs 降级的权衡

| 场景 | 策略 | 理由 |
|-----|------|------|
| **历史数据（不含今天）** | 12小时缓存，不采样 | 数据不会变化，最大化缓存命中率 |
| **包含今天的短范围** | 5分钟缓存，低概率采样 | 数据可能变化但频率低 |
| **包含今天的长范围** | 1小时缓存，可能采样 | 数据量大，优先保护 ClickHouse |
| **实时模式** | 30秒缓存 + 主动失效 | 需要最新数据，容忍频繁请求 |

### 6.2 实时刷新的缓存行为

- **缓存不是为了减少实时请求**：30秒的 staleTime 恰好等于刷新间隔
- **缓存用于**：
  1. 避免用户快速切换路由时重复请求
  2. 提供 `placeholderData` 实现静默刷新
  3. 组件重新挂载时立即显示数据
- **实际实时请求频率**：每 30 秒一次，与缓存 TTL 同步

### 6.3 大查询保护层次

1. **预防层**：查询验证拒绝不可能完成的查询（如 1 年的 minute 粒度）
2. **缓解层**：采样让大数据量查询以近似结果快速返回
3. **缓存层**：减少重复查询的压力
4. **请求层**：取消过时请求避免资源浪费

### 6.4 潜在改进空间

1. **后端查询结果缓存**：当前没有后端级别的查询结果缓存，每次都查询 ClickHouse
2. **采样率自适应**：当前采样率仅基于预估值，不考虑实际查询执行时间
3. **查询队列/限流**：缺少对并发大查询的限制机制

---

## 7. 关键代码索引

| 功能 | 文件路径 | 行号 |
|-----|---------|------|
| 前端缓存 TTL 定义 | `assets/js/dashboard/hooks/api-client.ts` | 20-23 |
| 缓存过期决策函数 | `assets/js/dashboard/hooks/api-client.ts` | 188-212 |
| 实时定时器 | `assets/js/dashboard/util/realtime-update-timer.js` | 1-8 |
| 主图表实时刷新 | `assets/js/dashboard/stats/graph/visitor-graph.tsx` | 156-178 |
| 采样率计算 | `extra/lib/plausible/stats/sampling.ex` | 56-104 |
| 采样缓存 | `extra/lib/plausible/stats/sampling_cache.ex` | 1-59 |
| SQL 采样 hint | `extra/lib/plausible/stats/sql/query_builder.ex` | 44, 64, 96, 122 |
| 查询验证流水线 | `lib/plausible/stats/query_builder.ex` | 333-584 |
| 查询优化流水线 | `lib/plausible/stats/query_optimizer.ex` | 52-64 |
| 时间粒度自动选择 | `lib/plausible/stats/query_optimizer.ex` | 95-102 |
