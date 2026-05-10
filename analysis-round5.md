# Plausible Analytics 请求取消机制最终校准（第五轮）

## 0. 执行摘要

### 0.1 核心发现

经过对 React Query 官方语义与项目实际实现的对比分析，得出以下核心结论：

| 维度 | React Query 库语义 | 项目实际实现 | 偏差 |
|-----|------------------|---------------|------|
| `invalidateQueries` | 只标记 stale + 触发后台 refetch，**不取消**进行中的请求 | ✅ 一致 | 无偏差 |
| `cancelQueries` | 专门用于取消进行中的请求 | ❌ 项目未使用 | 大偏差 |
| `signal` 参数 | `queryFn` 接收 `signal`，查询过期/组件卸载时 `signal` 被 abort | ❌ 项目未接线 | 大偏差 |
| 全局 `AbortController` | 无此设计模式 | ✅ 项目有，但 `cancelAll()` 是死代码 | 设计意图未实现 |

### 0.2 最终请求取消结论（定性）

**项目中**没有任何代码路径**会主动取消进行中的请求，除了浏览器原生的整页刷新。

---

## 1. React Query 官方语义深度解析

### 1.1 两个容易混淆的 API

#### `invalidateQueries`（标记过期）

**官方文档**（React Query v5）：

> "When a query is invalidated with invalidateQueries, two things happen:
> 1. It is marked as stale.
> 2. If the query is currently being rendered via useQuery, it will also be refetched in the background."

**核心语义**：
- ✅ 标记为 stale（覆盖 `staleTime`）
- ✅ 后台发起新请求（如果查询当前在被渲染）
- ❌ **不会取消**正在进行的请求**

```
时间线（invalidateQueries 场景）：
─────────────────────────────────────────────────────────────────→

T0:  请求 A 发起
      ────────────────────────────────────────────────────────►
           │
           │
T1:  invalidateQueries 调用
           │
           ├── 标记为 stale
           │
           └── 发起请求 B（后台 refetch）
           │
T2:  请求 A 仍在进行中（未被取消）
      ────────────────────────────────────────────────────────►
           │
T3:  请求 B 正在进行
      ────────────────────────────────────────────────────────►
           │
T4:  请求 A 完成 → 数据写入缓存（可能是旧数据）
           │
T5:  请求 B 完成 → 覆盖旧数据

结果：两个请求都执行，无取消行为
```

#### `cancelQueries`（主动取消）

**官方文档**（React Query v5）：

> "queryClient.cancelQueries({ queryKey }), which will cancel the query and revert it back to its previous state. If you have consumed the signal passed to the query function, TanStack Query will additionally also cancel the Promise."

**核心语义**：
- ✅ 取消进行中的请求
- ✅ 恢复查询状态到之前的状态
- ✅ 只有在 `queryFn` 使用了 `signal` 时，才会真正 abort 底层的 fetch

### 1.2 Query Cancellation 机制

**官方文档**（React Query v5）：

> "TanStack Query provides each query function with an AbortSignal instance. When a query becomes out-of-date or inactive, this signal will become aborted."

> "By default, queries that unmount or become unused before their promises are resolved are *not* cancelled. However, if you consume the AbortSignal, the Promise will be cancelled."

**关键条件**：
- React Query **会**在以下情况 abort `signal`：
  - 查询过期（out-of-date）
  - 查询不再使用（inactive）
  - 组件卸载
- 但**只有** `queryFn` 使用了这个 `signal`，底层请求才会被取消

**正确使用方式**（React Query 推荐）：
```typescript
const query = useQuery({
  queryKey: ['todos'],
  queryFn: async ({ signal }) => {  // ← 解构 signal
    const resp = await fetch('/todos', { signal })  // ← 传递给 fetch
    return resp.json()
  },
})
```

---

## 2. 项目代码实际实现分析

### 2.1 QueryFn 未使用 Signal

**Visitor Graph 中的实现** (`visitor-graph.tsx:46-61）：
```typescript
const topStatsQuery = useQuery({
  queryKey: ['top-stats', { dashboardState }] as const,
  queryFn: async ({ queryKey }) => {  // ← 只解构了 queryKey
    const [_, opts] = queryKey
    return await fetchTopStats(site, opts.dashboardState)
  },
  // ...
})

const mainGraphQuery = useQuery({
  queryKey: ['main-graph', { dashboardState, ... }] as const,
  queryFn: async ({ queryKey }) => {  // ← 只解构了 queryKey
    const [_, opts] = queryKey
    const data = await fetchMainGraph(site, ...)
    return data
  },
  // ...
})
```

**关键**：`queryFn` 参数中**没有**解构 `signal`！

### 2.2 PaginatedQuery 中的实现** (`api-client.ts:61-82`）：
```typescript
return useInfiniteQuery({
  queryKey: [dimensionKey, statsQuery],
  queryFn: async ({ pageParam }): Promise<api.QueryApiResponse> => {  // ← 只解构了 pageParam
    return api.stats(site, {
      ...statsQuery,
      pagination: { limit: PAGINATION_LIMIT, offset: pageParam as number }
    })
  },
  // ...
})
```

**关键**：同样没有解构 `signal`！

### 2.3 fetch 调用链

**fetchTopStats** (`fetch-top-stats.ts:36-72`）：
```typescript
export async function fetchTopStats(
  site: PlausibleSite,
  dashboardState: DashboardState
) {
  // ...
  const topStatsPromise = api.stats(site, topStatsQuery)
  // ...
  const [topStatsResponse, currentVisitorsResponse] = await Promise.all([
    topStatsPromise,
    currentVisitorsPromise
  ])
  // ...
}
```

**api.stats** (`api.ts:146-161`）：
```typescript
export async function stats(
  site: PlausibleSite,
  statsQuery: StatsQuery
): Promise<QueryApiResponse> {
  const path = apiPath(site, 'query')
  const response = await fetch(path, {
    method: 'POST',
    signal: abortController.signal,  // ← 使用全局 signal
    // ...
  })
  // ...
}
```

**关键发现**：

| 层级 | Signal 来源 | 是否会被 React Query 控制 |
|-----|------------|-------------------------|
| React Query `queryFn` 参数 | React Query 内部 `signal` | ❌ 未使用 |
| 全局 `abortController.signal` | 项目自己的模块级单例 | ❌ 永远不会被 abort（`cancelAll()` 是死代码 |

---

## 3. 两套 AbortController 系统完全脱节

### 3.1 系统对比

| 系统 | 控制方 | 项目状态 |
|-----|--------|-----------|
| **React Query 内置** | React Query 内部 | `signal` 参数未接线，没用到 |
| **项目全局** | `api.cancelAll()` | `cancelAll()` 是死代码，永远不执行 |

### 3.2 架构图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         请求取消架构（项目实际）                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  React Query 层                                                      │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │  useQuery({                                             │  │
│  │    queryKey: [...],                                     │  │
│  │    queryFn: async ({ queryKey }) => { ... }  ← 未使用 signal │  │
│  │  })                                                       │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                           │                                           │
│                           ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │  React Query 内部 AbortController                          │  │
│  │  ├── 组件卸载时 → 内部 signal.abort()                     │  │
│  │  ├── 查询过期时 → 内部 signal.abort()                     │  │
│  │  └── cancelQueries() → 内部 signal.abort()                 │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                           │                                           │
│                           │ ❌ 未接线！                             │
│                           │                                           │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │  api.ts: 全局 AbortController 模块级单例                    │  │
│  │  ├── let abortController = new AbortController()          │  │
│  │  ├── cancelAll() {                                      │  │
│  │  │   abortController.abort()                               │  │
│  │  │   abortController = new AbortController()               │  │
│  │  │ }                                                     │  │
│  │  └── fetch(..., { signal: abortController.signal })      │  │
│  │  │                                                        │  │
│  │  └── ❌ cancelAll() 被 useMountedEffect 调用        │  │
│  │      └── useMountedEffect(fn, []) 的 fn 永远不执行       │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  结果：                                                             │
│  ├── React Query 的取消能力 ❌ 未接线（未使用 signal 参数）             │
│  ├── 项目自己的取消机制 ❌ 无法触发（cancelAll() 是死代码）           │
│  └── ❌ 没有任何代码路径会主动取消请求                               │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. 三个核心场景的最终校准

### 4.1 场景一：实时刷新

**代码** (`visitor-graph.tsx:155-178`）：
```typescript
useEffect(() => {
  const onTick = () => {
    setIsRealtimeSilentUpdate({ topStats: true, mainGraph: true })
    queryClient.invalidateQueries({
      predicate: ({ queryKey }) =>
        ['top-stats', 'main-graph'].includes(queryKey[0]) &&
        queryKey[1]?.dashboardState?.period === DashboardPeriod.realtime
    })
  }
  // ...
}, [])
```

**最终结论**：

| 维度 | 状态 | 原因 |
|-----|------|------|
| 发起新请求 | ✅ 是 | `invalidateQueries` 触发后台 refetch |
| 取消旧请求 | ❌ 否 | `invalidateQueries` 语义上不取消 |
| 使用 React Query signal | ❌ 否 | `queryFn` 未接线 |
| 使用项目全局 abort | ❌ 否 | `cancelAll()` 是死代码 |
| 请求堆积风险 | ✅ 是 | 每 30 秒发起新请求，旧请求并行 |

**时间线（实时刷新）**：
```
T0:  请求 A 发起（top-stats + main-graph）
T30s: tick → invalidateQueries()
         ├── 标记为 stale
         └── 发起请求 B（top-stats + main-graph）
         └── ❌ 请求 A 未被取消
T60s: tick → invalidateQueries()
         ├── 标记为 stale
         └── 发起请求 C
         └── ❌ 请求 A、B 可能仍在进行
T90s: tick → ...

结果：可能同时有多个并行请求
```

### 4.2 场景二：路由变更

**路由结构** (`router.tsx:40-55`）：
```typescript
function DashboardElement() {
  return (
    <QueryClientProvider client={queryClient}>
      <DashboardStateContextProvider>  {/* 根级，不卸载 */}
        <Outlet />  {/* 子路由渲染点 */}
      </DashboardStateContextProvider>
    </QueryClientProvider>
  )
}
```

**最终结论**：

| 维度 | 状态 | 原因 |
|-----|------|------|
| Provider 卸载 | ❌ 否 | Provider 在根级，Outlet 模式 |
| `useMountedEffect` 执行 | ❌ 否 | `deps=[]`，`fn` 永远不执行 |
| `cancelAll()` 调用 | ❌ 否 | 同上 |
| React Query 取消 | ❌ 否 | `queryFn` 未使用 signal |
| queryKey 变化时 | ✅ 是（当 path/filter 变化时 React Query 自动取消） | ❌ 但 `queryFn` 未使用 signal |

**关键补充**：

React Query 在 `queryKey` 变化时，**确实会**使旧查询变成 "inactive"，如果 `queryFn` 接线了 `signal`，就会取消。

```
React Query 官方文档："When a query becomes out-of-date or inactive, this signal will become aborted."
```

但项目中 `queryFn` 未使用 `signal`，所以即使 `queryKey` 变化时：
- React Query 会让旧查询变成 inactive
- React Query 会 abort 内部的 signal
- 但 `queryFn` 没使用这个 signal
- ❌ 底层的 fetch 请求**不会被取消**

### 4.3 场景三：站点切换

**代码** (`site-switcher.tsx:135`）：
```typescript
window.location.assign(url)  // 整页刷新
```

**最终结论**：

| 维度 | 状态 | 原因 |
|-----|------|------|
| 取消旧请求 | ✅ 是 | 浏览器原生行为（页面卸载） |
| `cancelAll()` 调用 | ❌ 否 | 浏览器原生取消，不是代码调用 |
| 新页面 `cancelAll()` | ❌ 否 | 新页面的 `useMountedEffect` 的 `fn` 不执行 |
| QueryClient 清空 | ✅ 是 | 整页刷新，内存缓存消失 |
| 全局 AbortController | ✅ 是 | 新页面的 JS 模块重新初始化 |

---

## 5. 库语义 vs 项目实际行为对照矩阵

### 5.1 React Query 两个核心 API 对照

| 功能 | React Query 库语义 | 项目实际实现 | 一致性 |
|-----|------------------|--------------|--------|
| `invalidateQueries` | 标记 stale + 后台 refetch，**不取消**进行中的请求 | `queryKey` 不变的情况下，旧请求继续，新请求发起 | ✅ 一致 |
| `cancelQueries` | 取消进行中的请求 | ❌ 项目未使用 | ❌ 不一致 |
| `queryFn` 的 `signal` | 查询过期/组件卸载时 abort | ❌ 项目未使用 | ❌ 不一致 |

### 5.2 两套 AbortController 对照

| 系统 | React Query 内置 | 项目全局单例 |
|-----|----------------|-------------|
| 创建者 | React Query 内部 | `api.ts:9 |
| 触发方式 | 查询过期、组件卸载、`cancelQueries` | `api.cancelAll()` |
| 项目中触发次数 | 理论上很多，但没用到 | 0（死代码） |
| 是否接线到 fetch | ❌ 项目未用 | ✅ 接了但无法触发 |
| 实际效果 | ❌ 无取消 | ❌ 无取消 |

### 5.3 场景对照（最终版）

| 场景 | React Query 预期行为 | 项目实际行为 | 差异原因 |
|-----|-------------------|--------------|-----------|
| 实时 tick (`invalidateQueries`) | 不取消旧请求 | 不取消旧请求 | 库语义就是不取消 |
| queryKey 变化（过滤器/时间范围） | 如果接线了 signal，会取消 | 不取消（未接线） | `queryFn` 未使用 signal 参数 |
| 组件卸载（子路由） | 如果接线了 signal，会取消 | 不取消（未接线） | `queryFn` 未使用 signal 参数 |
| 路由 path 变化 | Provider 不卸载，查询可能变为 inactive | 不取消（未接线） | Provider 在根级 + 未使用 signal |
| 站点切换（整页刷新） | 浏览器原生取消所有请求 | 浏览器原生取消所有请求 | 一致 |
| `cancelQueries` 显式调用 | 取消匹配的查询 | 项目未使用此 API | 项目未调用 |

---

## 6. 最终请求取消触发条件矩阵（最终版）

### 6.1 会取消请求的场景

| 场景 | 触发机制 | 定性结论 | 证据 |
|-----|---------|---------|------|
| **站点切换（整页刷新） | 浏览器页面卸载 → 原生取消所有 fetch 请求 | ✅ 会取消 | `site-switcher.tsx:135` |
| **用户关闭/刷新页面** | 浏览器页面卸载 | ✅ 会取消 | 浏览器原生行为 |
| **手动在控制台调用 `api.cancelAll()` | 开发者调试 | ✅ 会取消 | `api.ts:68-71` |

### 6.2 不会取消请求的场景

| 场景 | 之前结论 | 最终校准 | 关键原因 |
|-----|---------|----------|---------|
| **实时 tick 刷新** | 会取消（错误） | ❌ **不会取消** | `invalidateQueries` 语义上不取消 |
| **queryKey 变化（过滤器变化） | 不会取消 | ❌ **不会取消** | `queryFn` 未使用 React Query 的 signal |
| **路由 path 变化** | 不会取消 | ❌ **不会取消** | Provider 在根级，查询变为 inactive，但未使用 signal |
| **组件卸载** | 不会取消 | ❌ **不会取消** | `queryFn` 未使用 signal |
| **页面首次加载后** | 会取消（错误） | ❌ **不会取消** | `useMountedEffect(fn, [])` 的 `fn` 不执行 |
| **`invalidateQueries` 调用** | 隐含会取消 | ❌ **不会取消** | 库语义就是不取消 |

### 6.3 如果 `queryFn` 接线了 signal 的理论行为

**假设项目修复了代码，`queryFn` 使用了 signal：

| 场景 | 理论行为 |
|-----|----------|
| queryKey 变化 | ✅ 会取消（查询变为 inactive） |
| 组件卸载 | ✅ 会取消（查询变为 inactive） |
| `cancelQueries()` | ✅ 会取消 |
| `invalidateQueries()`（同一 queryKey） | ❌ 不会取消（库语义） |

---

## 7. 五轮分析演进总结

### 7.1 各轮核心发现

| 轮次 | 核心发现 |
|-----|---------|
| Round 1 | 分析了缓存策略、采样机制、实时刷新架构 |
| Round 2 | 深入分析了站点/过滤器/时间范围变化对缓存的影响，建立了降级决策矩阵 |
| Round 3 | 发现 Provider 不随路由变化卸载，但仍误以为 `useMountedEffect(fn, [])` 会执行一次 |
| Round 4 | ✅ 发现 `useMountedEffect(fn, [])` 的 `fn` 永远不会执行，`cancelAll()` 是死代码 |
| **Round 5** | ✅ 明确 React Query 的 `invalidateQueries` 不取消请求，`queryFn` 未使用 signal |

### 7.2 请求取消结论的演进

| 结论 | Round 1 | Round 2 | Round 3 | Round 4 | Round 5 |
|-----|--------|--------|--------|--------|--------|
| 实时刷新会取消请求 | ❌ 是 | ❌ 是 | ❌ 否 | ❌ 否 | ❌ **否（库语义）** |
| 路由变化会取消请求 | ❌ 是 | ❌ 否 | ❌ 否 | ❌ 否 | ❌ **否（未接线 signal）** |
| 站点切换会取消请求 | ✅ 是 | ✅ 是 | ✅ 是 | ✅ 是 | ✅ **是（浏览器原生）** |
| `cancelAll()` 会被调用 | ❌ 是 | ❌ 是 | ❌ 是 | ❌ 否 | ❌ **否（死代码）** |
| React Query 能取消请求 | - | - | - | - | ❌ **理论上可以，但项目未接线 signal）** |

---

## 8. 代码问题总结

### 8.1 已确认的问题

| 问题 | 严重程度 | 影响 |
|-----|---------|------|
| `useMountedEffect(fn, [])` 死代码 | 中 | `cancelAll()` 永远不执行 |
| `queryFn` 未使用 React Query signal | 中 | React Query 的取消能力未利用 |
| 两套 AbortController 脱节 | 中 | 一套没用到，一套无法触发 |
| 实时模式请求堆积 | 低 | 每 30 秒并行请求可能累积 |

### 8.2 修复建议

**方案 A：接线 React Query signal（推荐）**

```typescript
// visitor-graph.tsx
const topStatsQuery = useQuery({
  queryKey: ['top-stats', { dashboardState }] as const,
  queryFn: async ({ queryKey, signal }) => {  // ← 添加 signal
    const [_, opts] = queryKey
    return await fetchTopStats(site, opts.dashboardState, signal)  // ← 传递 signal
  },
  // ...
})

// fetch-top-stats.ts
export async function fetchTopStats(
  site: PlausibleSite,
  dashboardState: DashboardState,
  signal?: AbortSignal  // ← 接收 signal
) {
  // ...
  const topStatsPromise = api.stats(site, topStatsQuery, signal)  // ← 传递 signal
  // ...
}

// api.ts
export async function stats(
  site: PlausibleSite,
  statsQuery: StatsQuery,
  signal?: AbortSignal  // ← 接收 signal
): Promise<QueryApiResponse> {
  const path = apiPath(site, 'query')
  const response = await fetch(path, {
    method: 'POST',
    signal: signal ?? abortController.signal,  // ← 优先使用传入的 signal
    // ...
  })
  // ...
}
```

**方案 B：在导航时显式取消

```typescript
// 在 useAppNavigate 或路由守卫中
queryClient.cancelQueries()  // ← 调用 cancelQueries 取消所有进行中的请求
```

---

## 9. 最终定性结论

### 9.1 核心结论

1. **`invalidateQueries` 不会取消进行中的请求**：这是 React Query 的设计语义，不是项目实现问题。

2. **`queryFn` 未使用 React Query 的 `signal` 参数**：项目放弃了 React Query 的内置取消能力。

3. **项目自己的全局 AbortController 是死代码**：`useMountedEffect(fn, [])` 的 `fn` 永远不会执行。

4. **唯一会取消请求的场景只有整页刷新**：由浏览器原生行为实现。

5. **实时模式可能存在请求堆积**：每 30 秒发起新请求，旧请求并行执行。

### 9.2 最终请求取消定性结论

**项目中没有任何代码路径会主动取消进行中的请求**，除了浏览器原生的整页刷新。
