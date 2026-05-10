# Plausible Analytics 请求取消机制核查（第三轮）

## 1. 核心发现摘要

### 1.1 关键修正结论

| 之前的结论（analysis.md/round2） | 核查后的实际行为 |
|--------------------------------|----------------|
| 路由变化时会调用 `api.cancelAll()` 取消所有请求 | ❌ **不会** |
| `useMountedEffect` 在每次路由变化时执行 | ❌ **只在页面首次加载后执行一次** |
| `DashboardStateContextProvider` 在路由变更时卸载重挂 | ❌ **不会**，它在根路由，子路由通过 Outlet 渲染 |

### 1.2 实际触发请求取消的场景

**只有以下场景会真正取消请求：**

| 场景 | 是否取消 | 触发机制 |
|-----|---------|---------|
| 页面**首次加载后**（刚打开仪表盘） | ✅ **会** | `useMountedEffect` + `api.cancelAll()` |
| 站点**切换**（整页刷新） | ✅ **会** | 浏览器原生请求取消 + `abortController` 重建 |
| 同一查询的**后续请求**覆盖 | ✅ **会** | 全局单一 `abortController`，新请求使用同一 signal |
| 路由 path 变化（如 `/` → `/sources`） | ❌ **不会** | Provider 不卸载，`useMountedEffect` 不触发 |
| search params 变化（过滤器/时间范围） | ❌ **不会** | 仅重渲染，`useMountedEffect` 不触发 |

---

## 2. useMountedEffect 行为深度分析

### 2.1 源码实现

```javascript
// assets/js/dashboard/custom-hooks.js:5-16
export function useMountedEffect(fn, deps) {
  const mounted = useRef(false)

  useEffect(() => {
    if (mounted.current) {
      fn()
    } else {
      mounted.current = true
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, deps)
}
```

### 2.2 行为时序图

```
页面加载
    │
    ▼
第一次渲染
    │
    ├── useMountedEffect 初始化
    │      ├── mounted.current = false
    │      └── deps = []
    │
    ├── useEffect 执行（挂载阶段）
    │      ├── mounted.current 为 false
    │      ├── 设置 mounted.current = true
    │      └── ❌ 不执行 fn()
    │
    ▼
首次渲染完成（组件已挂载）
    │
    └── （无 deps 变化，useEffect 不再执行）

后续路由变化 (search params / path)
    │
    ├── 组件重渲染（useLocation 响应）
    │
    ├── useEffect 依赖检查
    │      └── deps = [] → 无变化
    │
    └── ❌ useEffect 不执行 → fn() 不调用
```

### 2.3 在 DashboardStateContextProvider 中的使用

```typescript
// assets/js/dashboard/dashboard-state-context.tsx:143-145
useMountedEffect(() => {
  api.cancelAll()
}, [])  // ← 注意：deps 为空数组！
```

**结论**：
- `api.cancelAll()` 只在**页面首次挂载完成后**执行一次
- 目的：取消页面加载期间可能残留的旧请求（如服务端渲染、预加载等）
- **不会**在路由变更时触发

---

## 3. 路由结构与 Provider 挂载分析

### 3.1 路由树结构

```typescript
// assets/js/dashboard/router.tsx:40-55
function DashboardElement() {
  return (
    <QueryClientProvider client={queryClient}>
      <RoutelessModalsContextProvider>
        <DashboardStateContextProvider>     {/* ← 根级 Provider，始终挂载 */}
          <LastLoadContextProvider>
            <Dashboard />                  {/* 主仪表盘 */}
            <Outlet />                     {/* 子路由渲染点 */}
            <RoutelessSegmentModals />
          </LastLoadContextProvider>
        </DashboardStateContextProvider>  {/* ← 不会因子路由变化卸载 */}
      </RoutelessModalsContextProvider>
    </QueryClientProvider>
  )
}
```

### 3.2 路由配置

```typescript
// assets/js/dashboard/router.tsx:197-243
export function createAppRouter(site: PlausibleSite) {
  return createBrowserRouter(
    [
      {
        ...rootRoute,        // path: '/', element: <DashboardElement />
        children: [
          { index: true, element: <DashboardKeybinds /> },
          sourcesRoute,      // path: 'sources'
          channelsRoute,     // path: 'channels'
          countriesRoute,    // path: 'countries'
          // ... 更多子路由
        ]
      }
    ],
    { basename: basepath }
  )
}
```

### 3.3 路由变化时的挂载行为

| 导航类型 | 例子 | Provider 状态 | useMountedEffect |
|---------|------|--------------|-----------------|
| 根路径 → 子路由 | `/` → `/sources` | ✅ 保持挂载 | ❌ 不执行 |
| 子路由 → 根路径 | `/sources` → `/` | ✅ 保持挂载 | ❌ 不执行 |
| 子路由之间 | `/sources` → `/countries` | ✅ 保持挂载 | ❌ 不执行 |
| search params 变化 | `?period=7d` → `?period=28d` | ✅ 保持挂载 | ❌ 不执行 |
| 站点切换（整页刷新） | `siteA.com` → `siteB.com` | 🔥 新实例 | ✅ 新页面的 mounted effect |

### 3.4 useLocation 行为

```typescript
// assets/js/dashboard/dashboard-state-context.tsx:47
const location = useLocation()  // ← 响应所有路径变化

// assets/js/dashboard/dashboard-state-context.tsx:66
} = useMemo(() => parseSearch(location.search), [location.search])
```

**行为**：
- `useLocation()` 返回的 `location` 对象在**每次路径变化时**都会更新
- `location.search` 变化 → `useMemo` 重新计算 → `dashboardState` 变化
- 但这是**重渲染**，不是**卸载重挂载**
- `useMountedEffect` 的 `deps=[]` 意味着它只对首次挂载敏感

---

## 4. AbortController 机制分析

### 4.1 全局单例设计

```typescript
// assets/js/dashboard/api.ts:9
let abortController = new AbortController()  // ← 模块级单例

// assets/js/dashboard/api.ts:68-71
export function cancelAll() {
  abortController.abort()          // 取消当前所有绑定此 signal 的请求
  abortController = new AbortController()  // 创建新实例，后续请求使用新 signal
}
```

### 4.2 请求绑定方式

```typescript
// assets/js/dashboard/api.ts:148-157  (stats POST)
const response = await fetch(path, {
  method: 'POST',
  signal: abortController.signal,  // ← 绑定到全局单例
  // ...
})

// assets/js/dashboard/api.ts:171-174  (GET)
const response = await fetch(url, {
  signal: abortController.signal,  // ← 同一 signal
  // ...
})
```

### 4.3 关键特性：单一 Signal 覆盖

```
时间线：
─────────────────────────────────────────────────────────────────→

T0:  abortController = new AbortController()  →  signal A
     │
T1:  fetch('/api/stats/siteA/query', { signal: signal A })  →  请求 1
     │
T2:  fetch('/api/stats/siteA/query', { signal: signal A })  →  请求 2
     │
T3:  fetch('/api/stats/siteA/query', { signal: signal A })  →  请求 3
     │
T4:  cancelAll() 被调用
     │
     ├── abortController.abort() → 同时取消 请求 1, 请求 2, 请求 3
     │
     └── abortController = new AbortController() →  signal B
     │
T5:  fetch('/api/stats/siteA/query', { signal: signal B })  →  请求 4 (使用新 signal)
```

**结论**：
- 所有请求共享同一个 `abortController.signal`
- 一旦调用 `cancelAll()`，**所有正在进行的请求**都会被取消
- 但由于 `cancelAll()` 只在**页面首次加载后**调用一次，这个设计的实际价值有限

### 4.4 与 React Query 的交互

```typescript
// assets/js/dashboard/stats/graph/visitor-graph.tsx:63
placeholderData: (previousData) => previousData  // ← 显示旧数据

// assets/js/dashboard/stats/graph/visitor-graph.tsx:156-178
useEffect(() => {
  const onTick = () => {
    setIsRealtimeSilentUpdate({...})
    queryClient.invalidateQueries({...})  // ← 触发重新获取
  }
  // ...
}, [])
```

**实时刷新场景下的请求取消情况**：

| 场景 | 是否取消旧请求 | 原因 |
|-----|---------------|------|
| 实时 tick 触发 invalidate | ❌ 不会 | 没有调用 `cancelAll()` |
| queryKey 变化触发新查询 | ❌ 不会 | 没有调用 `cancelAll()` |
| 页面刚加载 | ✅ 会 | `useMountedEffect` 调用 `cancelAll()` |

**实际上，实时刷新时旧请求不会被取消，可能出现请求堆积**：

```
T0:  tick → 请求 A 发起 (使用 signal A)
T30s: tick → 请求 B 发起 (使用 signal A)  ← 请求 A 可能还在进行
T60s: tick → 请求 C 发起 (使用 signal A)  ← 请求 A、B 可能还在进行

没有 cancelAll() 调用 → 三个请求同时进行 → 可能的资源浪费
```

---

## 5. 请求取消触发条件对照表（最终版）

### 5.1 会取消请求的场景

| 场景 | 触发代码 | 取消范围 | 说明 |
|-----|---------|---------|------|
| **页面首次加载后** | `dashboard-state-context.tsx:143-145` | 所有正在进行的请求 | `useMountedEffect` + `deps=[]` → 仅一次 |
| **站点切换（整页刷新）** | `site-switcher.tsx:135,261` | 所有请求 | `window.location.assign()` 浏览器原生取消 |
| **显式调用 cancelAll()** | 无（仅上述两处） | 所有绑定 signal 的请求 | 模块级单例 abort |

### 5.2 不会取消请求的场景

| 场景 | 之前错误结论 | 实际行为 | 原因 |
|-----|-------------|---------|------|
| **路由 path 变化** | 会取消 | ❌ 不会取消 | Provider 不卸载，`useMountedEffect` deps=[] |
| **search params 变化** | 会取消 | ❌ 不会取消 | 仅重渲染，无 mount effect 触发 |
| **过滤器添加/移除** | 会取消 | ❌ 不会取消 | `queryKey` 变化触发新查询，旧查询无显式取消 |
| **时间范围变化** | 会取消 | ❌ 不会取消 | 同上 |
| **实时 tick 刷新** | 会取消 | ❌ 不会取消 | `invalidateQueries()` 不配合 `cancelAll()` |
| **组件卸载** | 会取消 | ❌ 不会取消 | 没有 cleanup 函数调用 `cancelAll()` |

### 5.3 完整决策矩阵

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          请求取消决策树                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  有请求正在进行？                                                        │
│        │                                                                 │
│        ├── 否 → 无操作                                                   │
│        │                                                                 │
│        └── 是 → 检查是否触发 cancelAll()                                 │
│                   │                                                      │
│                   ├── 页面刚加载？                                        │
│                   │    ├── 是 → ✅ cancelAll() → 取消所有请求            │
│                   │    └── 否 → 继续检查                                 │
│                   │                                                      │
│                   ├── 路由 path 变化？                                    │
│                   │    ├── 是 → ❌ 不取消（Provider 不卸载）             │
│                   │    └── 否 → 继续检查                                 │
│                   │                                                      │
│                   ├── search params 变化？                                │
│                   │    ├── 是 → ❌ 不取消（仅重渲染）                    │
│                   │    └── 否 → 继续检查                                 │
│                   │                                                      │
│                   ├── 站点切换？                                          │
│                   │    ├── 是 → ✅ 浏览器原生取消 + 新页面 cancelAll()   │
│                   │    └── 否 → ❌ 不取消                                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 6. 前报告修正对照表

### 6.1 analysis.md 中的错误结论

| 原结论 (analysis.md) | 位置 | 修正后的实际行为 |
|---------------------|------|----------------|
| "但前端使用 `AbortController` 取消旧请求" | 第 184 行 | ❌ 只有 `cancelAll()` 被调用时才会取消，大多数场景不会 |
| "L5: 请求取消：路由变化时取消所有进行中的请求" | 第 198 行 | ❌ 路由变化不会取消请求 |
| "AbortController: 路由变化时取消" | 第 301 行 | ❌ 路由变化不会触发 |
| "请求层：取消过时请求避免资源浪费" | 第 377 行 | ⚠️ 设计意图存在，但实际触发条件非常有限 |

### 6.2 analysis-round2.md 中的错误结论

| 原结论 (analysis-round2.md) | 位置 | 修正后的实际行为 |
|---------------------------|------|----------------|
| "api.cancelAll() ← 取消所有进行中的请求" (站点切换流程图) | 第 54 行 | ⚠️ 站点切换时确实会取消，但原因是整页刷新，不是这行代码；这行代码只在新页面加载后执行一次 |
| "请求取消：api.cancelAll() + AbortController" (静默刷新要素表) | 第 335 行 | ❌ 实时静默刷新**不会**取消旧请求 |
| "api.stats() → AbortController 取消旧请求" (实时刷新链路) | 第 603 行 | ❌ 不会取消，新请求使用同一 signal，但没有 abort 调用 |
| "挂载时取消所有请求" (代码索引说明) | 第 703 行 | ✅ 这个是对的，但只在**首次**挂载时，不是每次路由变化 |

---

## 7. 设计意图 vs 实际行为分析

### 7.1 设计意图推测

```
预期的请求取消流程（设计意图）：

1. 路由变化 → Provider 卸载重挂
   ↓
2. useMountedEffect 执行
   ↓
3. api.cancelAll() 取消旧请求
   ↓
4. 新路由发起新请求

结果：避免旧请求浪费资源
```

### 7.2 实际行为

```
实际流程（当前实现）：

1. 路由变化 → Provider 保持挂载（Outlet 模式）
   ↓
2. useMountedEffect 不执行（deps=[]）
   ↓
3. 旧请求继续进行
   ↓
4. 新请求发起，可能与旧请求并行

结果：可能存在请求堆积，特别是实时模式下
```

### 7.3 潜在问题

| 问题 | 影响 | 场景 |
|-----|------|------|
| 请求堆积 | 网络资源浪费 | 实时模式下每 30 秒发起新请求 |
| 竞态条件 | 旧数据覆盖新数据 | 慢请求在新请求之后返回 |
| 服务器压力 | 不必要的请求处理 | 用户快速切换过滤器/时间范围 |

### 7.4 可能的改进方向

**方案 A：在 useLocation 变化时取消请求**

```typescript
// 潜在改进：在 location 变化时取消请求
useEffect(() => {
  return () => {
    // location 变化时的 cleanup
    api.cancelAll()
  }
}, [location.pathname, location.search])
```

**方案 B：使用查询级别的 AbortController（React Query 内置）**

React Query 的 `useQuery` 已经内置了请求取消机制，当组件卸载或 queryKey 变化时会自动取消。但 Plausible 当前使用全局 `abortController` 与这个机制冲突。

**方案 C：在导航钩子中取消**

```typescript
// 潜在改进：在 useAppNavigate 中包装取消逻辑
export const useAppNavigate = () => {
  const _navigate = useNavigate()
  const navigate = useCallback((opts) => {
    api.cancelAll()  // 导航前取消所有请求
    return _navigate(...)
  }, [_navigate])
  return navigate
}
```

---

## 8. 关键代码索引（核查版）

| 功能 | 文件 | 行号 | 说明 |
|-----|------|------|------|
| **useMountedEffect 实现** | | | |
| Hook 定义 | `assets/js/dashboard/custom-hooks.js` | 5-16 | `deps` 参数控制触发时机 |
| 在 DashboardState 中的使用 | `assets/js/dashboard/dashboard-state-context.tsx` | 143-145 | `deps=[]` → 仅首次挂载后执行 |
| **AbortController** | | | |
| 全局单例 | `assets/js/dashboard/api.ts` | 9 | 模块级变量 |
| cancelAll 实现 | `assets/js/dashboard/api.ts` | 68-71 | abort + 重建 |
| 请求绑定 signal | `assets/js/dashboard/api.ts` | 150, 172, 187, 225 | 所有 API 使用同一 signal |
| **路由结构** | | | |
| DashboardElement 定义 | `assets/js/dashboard/router.tsx` | 40-55 | Provider 在根级 |
| Outlet 使用 | `assets/js/dashboard/router.tsx` | 49 | 子路由不影响父 Provider |
| 子路由配置 | `assets/js/dashboard/router.tsx` | 204-229 | 所有子路由共享根 Provider |

---

## 9. 总结

### 9.1 核心发现

1. **`useMountedEffect(fn, [])` 只执行一次**：在组件首次挂载完成后，不会因路由变化触发
2. **`DashboardStateContextProvider` 不会因路由变化卸载**：它在根路由，子路由通过 `<Outlet />` 渲染
3. **`api.cancelAll()` 只在页面刚加载后调用一次**：目的是清除加载期间的残留请求
4. **大多数场景下旧请求不会被取消**：包括路由变化、过滤器变化、实时刷新等

### 9.2 修正后的降级决策矩阵（与请求取消相关部分）

| 条件 | 降级类型 | 是否取消旧请求 | 说明 |
|-----|---------|--------------|------|
| 页面首次加载 | 无 | ✅ 会 | `useMountedEffect` + `cancelAll()` |
| 站点切换（整页刷新） | 无 | ✅ 会 | 浏览器原生 + 新页面的 cancelAll |
| `period ∈ [realtime, realtime_30m]` | 静默刷新 (前端) | ❌ 不会 | `invalidateQueries` 不配合 cancelAll |
| 过滤器/时间范围变化 | 无 | ❌ 不会 | 仅 queryKey 变化，无显式取消 |
| 路由 path 变化 | 无 | ❌ 不会 | Provider 不卸载 |
| 非法查询 (如 minute 粒度 > 30h) | 拒绝查询 (后端) | 不相关 | 请求被后端拒绝，前端无特殊处理 |
| 大查询采样 | 采样降级 (EE, 后端) | ❌ 不会 | 采样在后端执行，前端不取消请求 |

### 9.3 建议

如果需要实现"路由变化时取消请求"的预期行为，建议：
1. 在 `useAppNavigate` 包装器中添加 `api.cancelAll()` 调用，或者
2. 利用 React Query 内置的请求取消机制（让每个查询使用独立的 AbortController）
