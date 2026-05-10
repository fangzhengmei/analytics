# Plausible Analytics 请求取消机制最终校准（第四轮）

## 0. 最终结论速览

### 0.1 最关键的发现

**`useMountedEffect(fn, [])` 的 `fn` 永远不会执行！**

这是一个代码层面的死代码问题，意味着：
- `DashboardStateContextProvider` 中的 `api.cancelAll()` **不会被调用**
- `LastLoadContextProvider` 中的 `updateTimestamp()` **不会被调用**
- 全局 `abortController` **永远不会被代码主动取消**（除非整页刷新）

### 0.2 修正后的请求取消矩阵

| 场景 | 之前的结论 | 最终校准 | 原因 |
|-----|-----------|---------|------|
| 页面首次加载后 | ✅ 会取消 | ❌ **不会取消** | `useMountedEffect(fn, [])` 的 `fn` 永远不执行 |
| 路由 path 变化 | ❌ 不会取消 | ❌ **不会取消** | Provider 不卸载 |
| search params 变化 | ❌ 不会取消 | ❌ **不会取消** | 仅重渲染 |
| 实时 tick 刷新 | ❌ 不会取消 | ❌ **不会取消** | 无 `cancelAll()` 调用 |
| 站点切换（整页刷新） | ✅ 会取消 | ✅ **会取消** | 浏览器原生请求取消（非代码主动调用） |
| 显式调用 `cancelAll()` | 会取消 | 会取消 | 函数本身功能正常，但没人调用它 |

### 0.3 三份报告的错误修正历史

| 报告 | 关于请求取消的错误 |
|-----|-------------------|
| analysis.md | 声称"路由变化时取消所有请求" → 错误 |
| analysis-round2.md | 声称"首次挂载后调用 cancelAll" → 错误 |
| analysis-round3.md | 声称"首次挂载后调用 cancelAll" → 错误 |
| **analysis-round4.md** | ✅ 最终校准正确 |

---

## 1. useMountedEffect 行为精确分析

### 1.1 源码实现

```javascript
// assets/js/dashboard/custom-hooks.js:5-16
// 注释明确说明: "the function does not run on the initial render"
export function useMountedEffect(fn, deps) {
  const mounted = useRef(false)  // 初始值为 false

  useEffect(() => {
    if (mounted.current) {
      fn()
    } else {
      mounted.current = true
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, deps)  // deps 由调用者传入
}
```

### 1.2 当 `deps = []` 时的执行时序

```
时间线：
─────────────────────────────────────────────────────────────────→

T0: 组件首次渲染
    │
    ├── useMountedEffect(fn, []) 初始化
    │      ├── mounted.current = false
    │      └── deps = []
    │
T1: 首次挂载完成 → React 执行 useEffect
    │
    ├── 检查 if (mounted.current)
    │      └── mounted.current = false → 条件为假
    │
    ├── 执行 else 分支
    │      └── mounted.current = true
    │
    ├── ❌ fn() 不会被调用
    │
T2: 后续（只要 deps 不变）
    │
    ├── React 检查 deps = []
    │      └── 没有变化 → useEffect 不再执行
    │
    ├── ❌ fn() 永远不会被调用
```

### 1.3 关键逻辑推导

| 步骤 | 条件 | 结果 |
|-----|------|------|
| 1 | `deps = []` | useEffect 只在首次挂载时执行一次 |
| 2 | 首次 useEffect 执行时 | `mounted.current = false` |
| 3 | `if (mounted.current)` | 条件为假 |
| 4 | 执行 `else` 分支 | `mounted.current = true` |
| 5 | 后续 deps 无变化 | useEffect 不再执行 |
| 6 | `mounted.current = true` 的值 | 永远不会被再次检查 |
| **结论** | - | **`fn()` 永远不会执行** |

### 1.4 何时 `fn()` 才能执行？

要让 `useMountedEffect` 的 `fn()` 执行，必须同时满足：

1. **deps 不为空** → useEffect 会在 deps 变化时重新执行
2. **第二次及以后的 effect 执行** → `mounted.current` 已经是 `true`

**示例：`useMountedEffect(fn, [location.search])`**

```
T0: 首次渲染 → useEffect 执行
    ├── mounted.current = false
    ├── 设置 mounted.current = true
    └── ❌ fn 不执行

T1: location.search 变化 → deps 变化 → useEffect 重新执行
    ├── mounted.current = true（保持上一次的值）
    ├── if (mounted.current) 为真
    └── ✅ fn 执行
```

### 1.5 两个使用场景对比

| 使用场景 | 代码 | deps | fn 是否执行 |
|---------|------|------|------------|
| DashboardStateContextProvider | `useMountedEffect(() => { api.cancelAll() }, [])` | `[]` | ❌ **永远不执行** |
| LastLoadContextProvider | `useMountedEffect(() => { updateTimestamp() }, [])` | `[]` | ❌ **永远不执行** |

**代码位置**：
- `assets/js/dashboard/dashboard-state-context.tsx:143-145`
- `assets/js/dashboard/last-load-context.tsx:36-38`

---

## 2. AbortController 机制重新审视

### 2.1 全局单例设计

```typescript
// assets/js/dashboard/api.ts:9
let abortController = new AbortController()  // 模块级单例

// assets/js/dashboard/api.ts:68-71
export function cancelAll() {
  abortController.abort()          // 取消当前所有绑定此 signal 的请求
  abortController = new AbortController()  // 创建新实例
}
```

### 2.2 函数功能 vs 实际调用

| 项目 | 状态 |
|-----|------|
| `cancelAll()` 函数定义 | ✅ 存在且功能正常 |
| 代码中调用 `cancelAll()` 的地方 | 仅 1 处（`dashboard-state-context.tsx:144`） |
| 该调用是否会执行 | ❌ **不会**（`useMountedEffect(fn, [])` 的 `fn` 不执行） |
| 实际被调用的次数 | **0 次**（除非手动在控制台调用） |

### 2.3 所有请求的实际行为

```typescript
// assets/js/dashboard/api.ts:150, 172, 187, 225
signal: abortController.signal  // 绑定到全局单例

// 但由于 cancelAll() 从未被调用...

结果：
- 所有请求共享同一个 signal
- 这个 signal 永远不会被 abort（除非整页刷新）
- 没有任何请求会被代码主动取消
```

### 2.4 时间线示例（实时模式）

```
页面加载:
  abortController = new AbortController() → signal A

T0:  实时 tick → 请求 1 发起 (signal A)
T30s: 实时 tick → 请求 2 发起 (signal A)  ← 请求 1 可能还在进行
T60s: 实时 tick → 请求 3 发起 (signal A)  ← 请求 1、2 可能还在进行
T90s: 实时 tick → 请求 4 发起 (signal A)  ← 请求 1、2、3 可能还在进行
...

问题：
- 没有 cancelAll() 调用
- 所有请求共享 signal A
- signal A 永远不会被 abort
- 可能存在请求堆积
```

---

## 3. 三个核心场景的最终校准

### 3.1 场景一：实时刷新

#### 之前的错误描述
> "实时 tick 时会调用 `cancelAll()` 取消旧请求"

#### 最终校准分析

**实时刷新代码** (`visitor-graph.tsx:155-178`):
```typescript
useEffect(() => {
  const onTick = () => {
    setIsRealtimeSilentUpdate({ topStats: true, mainGraph: true })
    queryClient.invalidateQueries({...})  // 触发重新获取
    // ❌ 注意：这里没有调用 api.cancelAll()
  }

  if (isRealtime) {
    document.addEventListener('tick', onTick)
  }

  return () => {
    document.removeEventListener('tick', onTick)
  }
}, [queryClient, isRealtime])
```

**关键事实**：
1. `invalidateQueries()` 只标记查询为过期，不会主动取消请求
2. 代码中没有任何 `cancelAll()` 调用
3. 旧请求继续进行，新请求并行发起

**最终结论**：
| 项目 | 状态 |
|-----|------|
| 是否显示旧数据 | ✅ 是（`placeholderData = previousData`） |
| 是否隐藏 spinner | ✅ 是（`isRealtimeSilentUpdate`） |
| 是否取消旧请求 | ❌ **否** |
| 是否可能请求堆积 | ✅ **是**（每 30 秒发起新请求） |

---

### 3.2 场景二：路由变更

#### 之前的错误描述
> "路由变化时 Provider 卸载重挂，`useMountedEffect` 执行 `cancelAll()`"

#### 最终校准分析

**路由结构** (`router.tsx:40-55`):
```typescript
function DashboardElement() {
  return (
    <QueryClientProvider client={queryClient}>
      <RoutelessModalsContextProvider>
        <DashboardStateContextProvider>  {/* ← 根级 Provider */}
          <LastLoadContextProvider>
            <Dashboard />
            <Outlet />  {/* ← 子路由渲染点 */}
            <RoutelessSegmentModals />
          </LastLoadContextProvider>
        </DashboardStateContextProvider>  {/* ← 不会因子路由变化卸载 */}
      </RoutelessModalsContextProvider>
    </QueryClientProvider>
  )
}
```

**路由配置** (`router.tsx:197-243`):
```typescript
export function createAppRouter(site: PlausibleSite) {
  return createBrowserRouter(
    [
      {
        ...rootRoute,        // path: '/', element: <DashboardElement />
        children: [
          { index: true, element: <DashboardKeybinds /> },
          sourcesRoute,      // path: 'sources'
          countriesRoute,    // path: 'countries'
          // ... 更多子路由
        ]
      }
    ],
    { basename: basepath }
  )
}
```

**关键事实**：
1. `DashboardStateContextProvider` 在**根路由** (`path: '/'`)
2. 子路由通过 `<Outlet />` 渲染，**不会导致父组件卸载**
3. 即使 `useMountedEffect` 的 `fn` 会执行（实际上不会），也没有触发机会

**useLocation 行为** (`dashboard-state-context.tsx:47,66`):
```typescript
const location = useLocation()  // ← 响应路径变化

} = useMemo(() => parseSearch(location.search), [location.search])
// ↓
// 这会导致重渲染，但不会导致卸载重挂载
```

**最终结论**：
| 路由变化类型 | Provider 状态 | useMountedEffect | 是否取消请求 |
|-------------|--------------|-----------------|------------|
| `/` → `/sources` | ✅ 保持挂载 | ❌ 不执行 | ❌ 不取消 |
| `/sources` → `/` | ✅ 保持挂载 | ❌ 不执行 | ❌ 不取消 |
| `?period=7d` → `?period=28d` | ✅ 保持挂载 | ❌ 不执行 | ❌ 不取消 |
| 过滤器增删改 | ✅ 保持挂载 | ❌ 不执行 | ❌ 不取消 |

---

### 3.3 场景三：站点切换

#### 之前的描述（部分正确）
> "站点切换时调用 `window.location.assign()` 整页刷新，会取消所有请求"

#### 最终校准分析

**站点切换代码** (`site-switcher.tsx:59-68, 130-136`):
```typescript
const getSwitchToSiteURL = (currentSite, site): string | null => {
  if (currentSite.domain === site.domain) {
    return null
  }
  return `/${encodeURIComponent(site.domain)}`
}

// 切换逻辑
const url = getSwitchToSiteURL(currentSite, { domain })
if (!url) {
  closePopover()
} else {
  closePopover()
  window.location.assign(url)  // ← 整页刷新
}
```

**整页刷新时的请求取消机制**：
```
用户点击切换站点
    │
    ▼
window.location.assign('/new-site')
    │
    ├── 浏览器导航到新 URL
    │
    ├── 当前页面即将卸载
    │
    ├── 浏览器原生取消所有进行中的 fetch 请求
    │
    ├── 浏览器加载新页面
    │
    ├── 新页面的 JS 重新执行
    │
    └── abortController = new AbortController()（新模块实例）
```

**关键事实**：
1. 请求取消是**浏览器原生行为**（页面卸载时自动取消）
2. 不是 `api.cancelAll()` 的功劳（它从未被调用）
3. 新页面加载后，`abortController` 是**新的模块实例**

**最终结论**：
| 项目 | 状态 |
|-----|------|
| 是否取消旧请求 | ✅ 是（浏览器原生，非代码主动） |
| `cancelAll()` 是否被调用 | ❌ 否（useMountedEffect 的 fn 不执行） |
| QueryClient 缓存是否清空 | ✅ 是（整页刷新，内存缓存消失） |
| abortController 是否重建 | ✅ 是（新页面的 JS 模块重新初始化） |

---

## 4. 最终请求取消触发条件矩阵

### 4.1 会取消请求的场景

| 场景 | 触发机制 | 说明 | 代码证据 |
|-----|---------|------|---------|
| **站点切换（整页刷新）** | `window.location.assign()` → 浏览器页面卸载 → 原生取消所有请求 | 唯一真正会取消请求的场景 | `site-switcher.tsx:135` |
| **用户关闭/刷新页面** | 浏览器页面卸载 | 原生行为，与代码无关 | - |
| **手动在控制台调用 `api.cancelAll()`** | 开发者调试 | 非生产场景 | `api.ts:68-71` |

### 4.2 不会取消请求的场景

| 场景 | 之前错误结论 | 最终结论 | 原因 |
|-----|-------------|---------|------|
| **页面首次加载后** | 会取消（useMountedEffect） | ❌ 不会取消 | `useMountedEffect(fn, [])` 的 `fn` 永远不执行 |
| **路由 path 变化** | 会取消 | ❌ 不会取消 | Provider 不卸载 |
| **search params 变化** | 会取消 | ❌ 不会取消 | 仅重渲染 |
| **过滤器变化** | 会取消 | ❌ 不会取消 | queryKey 变化触发新查询，旧请求并行 |
| **时间范围变化** | 会取消 | ❌ 不会取消 | 同上 |
| **实时 tick 刷新** | 会取消 | ❌ 不会取消 | `invalidateQueries()` 不配合 `cancelAll()` |
| **组件卸载** | 会取消 | ❌ 不会取消 | 没有 cleanup 函数调用 `cancelAll()` |
| **子路由模态框打开/关闭** | 会取消 | ❌ 不会取消 | Provider 保持挂载 |

### 4.3 完整决策树

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        请求取消决策树（最终版）                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  有请求正在进行？                                                        │
│        │                                                                 │
│        ├── 否 → 无操作                                                   │
│        │                                                                 │
│        └── 是                                                           │
│             │                                                            │
│             ├── 整页刷新？（window.location.assign / 用户刷新）           │
│             │    │                                                       │
│             │    ├── 是 → ✅ 浏览器原生取消所有请求                       │
│             │    │                                                       │
│             │    └── 否 → 继续检查                                       │
│             │                                                            │
│             ├── useMountedEffect(fn, []) 会执行吗？                       │
│             │    │                                                       │
│             │    ├── 是 → 不可能（见下方注释）                           │
│             │    │                                                       │
│             │    └── 否 → ❌ 不取消                                      │
│             │                                                            │
│             ├── 有显式 cancelAll() 调用吗？                               │
│             │    │                                                       │
│             │    ├── 是 → ✅ 取消所有请求（仅控制台调试场景）             │
│             │    │                                                       │
│             │    └── 否 → ❌ 不取消                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

注释：
useMountedEffect(fn, []) 的 fn 永远不会执行，因为：
  1. deps = [] → useEffect 只执行一次（首次挂载）
  2. 首次执行时 mounted.current = false
  3. if (mounted.current) 为假 → 设置 mounted.current = true
  4. fn 被跳过
  5. 后续无 deps 变化 → useEffect 不再执行
```

---

## 5. 死代码问题分析

### 5.1 识别出的死代码

| 文件 | 行号 | 代码 | 问题 |
|-----|------|------|------|
| `dashboard-state-context.tsx` | 143-145 | `useMountedEffect(() => { api.cancelAll() }, [])` | `api.cancelAll()` 永远不会执行 |
| `last-load-context.tsx` | 36-38 | `useMountedEffect(() => { updateTimestamp() }, [])` | `updateTimestamp()` 永远不会执行 |

### 5.2 LastLoadContextProvider 的影响

```typescript
// last-load-context.tsx:28-38
useEffect(() => {
  document.addEventListener('tick', updateTimestamp)  // ✅ 会执行
  return () => {
    document.removeEventListener('tick', updateTimestamp)
  }
}, [updateTimestamp])

useMountedEffect(() => {
  updateTimestamp()  // ❌ 永远不会执行
}, [])
```

**影响分析**：
- `tick` 事件监听是有效的（每 30 秒更新一次）
- 但页面刚加载时，`updateTimestamp()` 不会立即执行一次
- 因此 `LastLoadContext` 的初始值是 `new Date()`（组件创建时），而不是页面加载完成时
- 实际影响很小（最多相差几十毫秒）

### 5.3 代码意图推测

```typescript
// 可能的原始意图：在页面加载完成后立即更新 timestamp
useMountedEffect(() => {
  updateTimestamp()
}, [])

// 但由于 deps = []，实际效果是：永远不执行

// 要实现意图，应该写成：
useEffect(() => {
  updateTimestamp()
}, [])  // 直接使用 useEffect，跳过首次渲染检测
```

---

## 6. 前报告错误修正对照表

### 6.1 analysis.md 中的错误

| 原结论 | 位置 | 错误类型 | 修正 |
|-------|------|---------|------|
| "但前端使用 `AbortController` 取消旧请求" | 第 184 行 | 功能不存在 | 没有代码主动调用 `cancelAll()`，请求不会被取消 |
| "L5: 请求取消：路由变化时取消所有进行中的请求" | 第 198 行 | 功能不存在 | 路由变化不会取消请求 |
| "AbortController: 路由变化时取消" | 第 301 行 | 功能不存在 | 同上 |
| "请求层：取消过时请求避免资源浪费" | 第 377 行 | 意图与实现不符 | 设计意图存在，但实现有 bug（`useMountedEffect(fn, [])` 的 fn 不执行） |

### 6.2 analysis-round2.md 中的错误

| 原结论 | 位置 | 错误类型 | 修正 |
|-------|------|---------|------|
| "api.cancelAll() ← 取消所有进行中的请求" | 第 54 行 | 功能不存在 | 这行代码存在，但永远不会执行 |
| "请求取消：api.cancelAll() + AbortController" | 第 335 行 | 功能不存在 | 同上 |
| "api.stats() → AbortController 取消旧请求" | 第 603 行 | 功能不存在 | 新请求使用同一 signal，但没有 abort 调用 |
| "挂载时取消所有请求" | 第 703 行 | 部分错误 | "挂载后"不会执行，只有整页刷新才会取消 |

### 6.3 analysis-round3.md 中的错误

| 原结论 | 位置 | 错误类型 | 修正 |
|-------|------|---------|------|
| "页面首次加载后会调用 api.cancelAll()" | 多处 | 功能不存在 | `useMountedEffect(fn, [])` 的 fn 永远不执行 |
| "`useMountedEffect` 在组件首次挂载完成后执行一次" | 多处 | 部分错误 | 执行的是 useEffect 的回调主体，但 `fn()` 被跳过 |

---

## 7. 修正后的降级决策矩阵（请求取消相关）

| 条件 | 降级类型 | 是否取消旧请求 | 最终说明 |
|-----|---------|--------------|---------|
| 页面首次加载 | 无 | ❌ 不会 | `useMountedEffect(fn, [])` 的 fn 不执行 |
| 站点切换（整页刷新） | 无 | ✅ 会 | 浏览器原生行为（非代码主动） |
| `period ∈ [realtime, realtime_30m]` | 静默刷新 (前端) | ❌ 不会 | `invalidateQueries()` 不配合 `cancelAll()` |
| 过滤器/时间范围变化 | 无 | ❌ 不会 | 仅 queryKey 变化，无显式取消 |
| 路由 path 变化 | 无 | ❌ 不会 | Provider 不卸载 |
| 非法查询 (如 minute 粒度 > 30h) | 拒绝查询 (后端) | 不相关 | 请求被后端拒绝，前端无特殊处理 |
| 大查询采样 | 采样降级 (EE, 后端) | ❌ 不会 | 采样在后端执行，前端不取消请求 |

---

## 8. 潜在问题与改进建议

### 8.1 已确认的问题

| 问题 | 严重程度 | 说明 |
|-----|---------|------|
| `useMountedEffect(fn, [])` 的 fn 不执行 | 中 | 死代码，cancelAll 永远不会被调用 |
| 实时模式请求堆积 | 中 | 每 30 秒发起新请求，旧请求不取消 |
| 快速切换时的竞态条件 | 低 | 旧请求可能在新请求之后返回，覆盖新数据 |

### 8.2 改进方案

**方案 A：修复 useMountedEffect 的使用**

```typescript
// dashboard-state-context.tsx
// 改为使用 useEffect，跳过 mounted 检测
useEffect(() => {
  api.cancelAll()
}, [location.pathname, location.search])
```

**方案 B：在 useAppNavigate 中添加取消逻辑**

```typescript
// navigation/use-app-navigate.tsx
export const useAppNavigate = () => {
  const _navigate = useNavigate()
  const navigate = useCallback((opts) => {
    api.cancelAll()  // 导航前取消所有请求
    return _navigate(...)
  }, [_navigate])
  return navigate
}
```

**方案 C：使用 React Query 内置的请求取消**

React Query 的 `useQuery` 已经内置了请求取消机制，但需要每个查询使用独立的 `AbortController`，而不是全局单例。

---

## 9. 最终代码索引（校准版）

| 功能 | 文件 | 行号 | 最终状态 |
|-----|------|------|---------|
| **useMountedEffect 实现** | | | |
| Hook 定义 | `assets/js/dashboard/custom-hooks.js` | 5-16 | 逻辑正确，但 deps=[] 导致死代码 |
| DashboardState 中的使用 | `assets/js/dashboard/dashboard-state-context.tsx` | 143-145 | ❌ 死代码，cancelAll 不执行 |
| LastLoadContext 中的使用 | `assets/js/dashboard/last-load-context.tsx` | 36-38 | ❌ 死代码，updateTimestamp 不执行 |
| **AbortController** | | | |
| 全局单例 | `assets/js/dashboard/api.ts` | 9 | 存在但从未被 abort |
| cancelAll 实现 | `assets/js/dashboard/api.ts` | 68-71 | 功能正确但无人调用 |
| 请求绑定 signal | `assets/js/dashboard/api.ts` | 150, 172, 187, 225 | 绑定但 signal 不 abort |
| **路由与站点切换** | | | |
| DashboardElement 定义 | `assets/js/dashboard/router.tsx` | 40-55 | Provider 在根级，不卸载 |
| Outlet 使用 | `assets/js/dashboard/router.tsx` | 49 | 子路由不影响父 Provider |
| 站点切换 URL 构造 | `assets/js/dashboard/site-switcher.tsx` | 59-68 | 构造新站点路径 |
| 站点切换执行 | `assets/js/dashboard/site-switcher.tsx` | 135 | `window.location.assign()` 整页刷新 |
| **实时刷新** | | | |
| tick 监听 | `assets/js/dashboard/stats/graph/visitor-graph.tsx` | 155-178 | 正常，但无 cancelAll 调用 |
| invalidateQueries | `assets/js/dashboard/stats/graph/visitor-graph.tsx` | 158-168 | 只标记过期，不取消请求 |

---

## 10. 总结

### 10.1 四轮分析的演进

| 轮次 | 核心发现 |
|-----|---------|
| Round 1 | 分析了缓存策略、采样机制、实时刷新架构 |
| Round 2 | 深入分析了站点/过滤器/时间范围变化对缓存的影响，建立了降级决策矩阵 |
| Round 3 | 发现 Provider 不随路由变化卸载，但仍误以为 `useMountedEffect(fn, [])` 会执行一次 |
| **Round 4** | ✅ 最终发现 `useMountedEffect(fn, [])` 的 `fn` **永远不会执行**，`cancelAll()` 是死代码 |

### 10.2 最终核心结论

1. **`useMountedEffect(fn, [])` 的 `fn` 永远不会执行**：这是一个死代码问题
2. **`api.cancelAll()` 在生产环境中从未被调用**：所有请求共享的 signal 永远不会被 abort
3. **唯一会取消请求的场景是整页刷新**：由浏览器原生行为实现，与代码无关
4. **实时模式下可能存在请求堆积**：每 30 秒发起新请求，旧请求不取消
5. **两份之前的报告（round2、round3）包含不准确的结论**：都误以为 `useMountedEffect(fn, [])` 会在首次挂载后执行一次
