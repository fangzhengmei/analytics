# Plausible 公开报表与嵌入报表实现分析

## 一、概述

Plausible 的公开报表（Public Dashboard）、嵌入报表（Embedded Dashboard）和公开站点（Public Site）三种访问方式，通过复用私有 Dashboard 的查询逻辑和前端组件，同时在权限层面进行精细控制，实现不登录时展示有限指标的需求。

**核心设计原则：**
- **查询逻辑完全复用**：不区分访问类型，所有数据查询走相同代码路径
- **权限控制在入口层**：通过角色 (`site_role`) 控制访问权限和 UI 组件
- **唯一数据级限制**：Segment 过滤（可选）

---

## 二、不登录时的有限指标：三种访问方式详解

### 2.1 核心结论：没有基于角色的行级数据过滤

**Plausible 没有基于角色的行级数据过滤！** "有限指标"通过两种独立机制实现：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           限制机制分层                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Layer 3: UI / 功能限制 (由 site_role = :public 控制)                        │
│  ─────────────────────────────────────────────────────────────────────────   │
│  ✓ 无法创建/编辑 Segments                                                   │
│  ✓ 无法切换站点                                                             │
│  ✓ 无法访问站点设置                                                         │
│  ✓ 无法保存自定义过滤器                                                     │
│  ✓ 隐藏部分菜单选项                                                         │
│                                                                              │
│  Layer 2: 数据范围限制 (由 shared_link.segment_id 控制，可选)                │
│  ─────────────────────────────────────────────────────────────────────────   │
│  ✓ 强制应用 Segment 定义的过滤条件                                          │
│  ✓ 无法绕过（API 层强制验证）                                                │
│  ✓ 这是唯一的数据级限制方式                                                 │
│                                                                              │
│  Layer 1: 访问权限控制 (能否进入报表)                                        │
│  ─────────────────────────────────────────────────────────────────────────   │
│  ✓ 成员关系验证                                                             │
│  ✓ SharedLink 存在性验证                                                    │
│  ✓ site.public 检查                                                         │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 三种公开访问方式对比

| 特性 | 私有 Dashboard | 公开站点 (Public Site) | 共享链接 (Shared Link) | 嵌入模式 (Embedded) |
|-----|--------------|----------------------|----------------------|-------------------|
| **启用方式** | 登录 + 成员关系 | `site.public = true` | 创建 SharedLink 记录 | 共享链接 + `?embed=true` |
| **URL 路径** | `/:domain` | `/:domain` | `/share/:domain?auth=slug` | `/share/:domain?auth=slug&embed=true` |
| **需登录** | 是 | 否 | 否 | 否 |
| **site_role** | `:admin/:editor/:viewer/:owner` | `:public` | `:public` | `:public` |
| **数据范围** | 完整站点数据 | 完整站点数据 | 可选：Segment 限制 | 可选：Segment 限制 |

### 2.3 指标可见性详细对比

| 指标/功能 | 私有 Dashboard | 公开站点 | 共享链接 (无 Segment) | 共享链接 (有 Segment) | 嵌入模式 | 限制来源 |
|----------|---------------|----------|---------------------|----------------------|---------|---------|
| **数据范围** | | | | | | |
| 访客数 (Visitors) | 完整数据 | 完整数据 | 完整数据 | **Segment 过滤后** | **Segment 过滤后** | Segment (若有) |
| 页面浏览 (Pageviews) | 完整数据 | 完整数据 | 完整数据 | **Segment 过滤后** | **Segment 过滤后** | Segment (若有) |
| 访问时长 (Avg Duration) | 完整数据 | 完整数据 | 完整数据 | **Segment 过滤后** | **Segment 过滤后** | Segment (若有) |
| 跳出率 (Bounce Rate) | 完整数据 | 完整数据 | 完整数据 | **Segment 过滤后** | **Segment 过滤后** | Segment (若有) |
| 渠道来源 (Sources) | 完整数据 | 完整数据 | 完整数据 | **Segment 过滤后** | **Segment 过滤后** | Segment (若有) |
| 页面列表 (Pages) | 完整数据 | 完整数据 | 完整数据 | **Segment 过滤后** | **Segment 过滤后** | Segment (若有) |
| 地理位置 (Locations) | 完整数据 | 完整数据 | 完整数据 | **Segment 过滤后** | **Segment 过滤后** | Segment (若有) |
| 设备信息 (Devices) | 完整数据 | 完整数据 | 完整数据 | **Segment 过滤后** | **Segment 过滤后** | Segment (若有) |
| 转化目标 (Goals) | 完整数据 | 完整数据 | 完整数据 | **Segment 过滤后** | **Segment 过滤后** | Segment (若有) |
| **功能限制** | | | | | | |
| 切换站点 | 是 | 否 | 否 | 否 | 否 | 角色 (:public) |
| 创建 Site Segment | 是 (`:editor+`) | 否 | 否 | 否 | 否 | 角色 (:public) |
| 创建 Personal Segment | 是 (`:viewer+`) | 否 | 否 | 否 | 否 | 角色 (:public) |
| 保存过滤器 | 是 | 否 | 否 | 否 | 否 | 角色 (:public) |
| 查看 Segment 详情 | 是 | 否 | 否 | 否 | 否 | 角色 (:public) |
| 导出 CSV | 是 | 否 | 否 | 否 | 否 | 角色 (:public) |
| **UI 限制** | | | | | | |
| 显示页脚 | 是 | EE: 否 / CE: 是 | EE: 否 / CE: 是 | EE: 否 / CE: 是 | **否** | 角色 + embedded |
| Sticky 导航 | 是 | 是 | 是 | 是 | **否** | embedded 参数 |
| iframe 嵌入 | 被阻止 | 被阻止 | 被阻止 | 被阻止 | **允许** | embedded 参数 |

### 2.4 角色限制 vs Segment 限制对比

| 维度 | 角色限制 (`site_role = :public`) | Segment 限制 (`shared_link.segment_id`) |
|-----|-------------------------------|---------------------------------------|
| **控制层面** | UI / 功能层 | 数据层 |
| **影响范围** | 隐藏功能入口，不影响数据查询 | 强制过滤数据范围 |
| **是否可绕过** | 可通过直接调用 API 绕过 (但功能已隐藏) | 不可绕过 (API 层强制验证) |
| **作用** | 限制"能做什么" (创建、编辑、导出等) | 限制"能看什么数据" |
| **实现位置** | 前端组件条件渲染 + 后端路由权限 | `validate_required_filters_plug` |
| **可选性** | 所有公开访问都有此限制 | 仅当 SharedLink 关联 Segment 时生效 |

---

## 三、Token 与站点权限的分叉决策顺序

### 3.1 决策优先级总览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    权限决策优先级 (从高到低)                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  入口: AuthorizeSiteAccess.call(conn, {allowed_roles, site_param})         │
│                                                                              │
│  1. 成员关系 (最高优先级)                                                     │
│     └─> membership_role 存在 → 使用成员角色                                  │
│         (即使同时有公开站点或共享链接，成员关系优先)                          │
│                                                                              │
│  2. 超管                                                                     │
│     └─> is_super_admin?(current_user) → :super_admin                        │
│                                                                              │
│  3. 公开站点                                                                 │
│     └─> site.public == true → :public (任何人可通过 /:domain 访问)           │
│                                                                              │
│  4. 共享链接                                                                 │
│     └─> 有有效 shared_link → :public (通过 /share/:domain?auth= 访问)        │
│                                                                              │
│  5. 拒绝访问 (404)                                                            │
│     └─> 以上都不满足 → nil                                                   │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 核心决策代码

**`AuthorizeSiteAccess.call/2`** (`lib/plausible_web/plugs/authorize_site_access.ex:98-106`)：

```elixir
# 这是整个权限系统的核心分叉点
role =
  cond do
    # 分支 A: 有成员关系 → 使用成员角色
    # 即使同时有公开站点或共享链接，成员关系优先
    membership_role -> membership_role
    
    # 分支 B: 超管
    Plausible.Auth.is_super_admin?(current_user) -> :super_admin
    
    # 分支 C: 站点公开 → 任何人均可通过 /:domain 访问
    site.public -> :public
    
    # 分支 D: 有有效共享链接 → 通过 /share/:domain?auth=slug 访问
    shared_link -> :public
    
    # 都不满足 → 拒绝访问
    true -> nil
  end
```

### 3.3 不同场景的决策路径示例

**场景 1: 登录用户访问私有站点**
```
URL: GET /myprivate.com (site.public = false)
用户: 已登录，是站点成员

决策路径:
1. membership_role = :admin (存在)
2. 结果 role = :admin
3. 允许访问
```

**场景 2: 未登录用户访问公开站点**
```
URL: GET /mypublic.com (site.public = true)
用户: 未登录

决策路径:
1. membership_role = nil (无成员关系)
2. 非超管
3. site.public = true ✓
4. 结果 role = :public
5. 允许访问
```

**场景 3: 未登录用户访问共享链接**
```
URL: GET /share/myprivate.com?auth=abc123
用户: 未登录
SharedLink: 存在且无密码保护

决策路径:
1. membership_role = nil (无成员关系)
2. 非超管
3. site.public = false (不满足)
4. shared_link = %SharedLink{} (存在) ✓
5. 结果 role = :public
6. 允许访问
```

**场景 4: 登录用户同时有成员关系和共享链接**
```
URL: GET /share/mydomain.com?auth=abc123
用户: 已登录，是站点成员 (:admin)
SharedLink: 存在

决策路径:
1. membership_role = :admin (存在) ✓
2. 结果 role = :admin (忽略 shared_link)
3. 允许访问

关键: 成员关系优先级高于共享链接！即使通过 /share 路径访问，
      登录用户仍以成员身份访问，获得完整权限。
```

---

## 四、查询链路复用与差异对照

### 4.1 复用/差异对照表

| 层级 | 私有 Dashboard | 公开站点 | 共享链接 (有 Segment) | 相同点 |
|-----|--------------|----------|---------------------|--------|
| **页面访问入口** | | | | |
| URL 路径 | `/:domain` | `/:domain` | `/share/:domain?auth=slug` | - |
| Pipeline | `:browser` | `:browser` | `:shared_link` | 都走 `AuthPlug` |
| 控制器函数 | `stats/2` | `stats/2` | `shared_link/2` | 最终都调用 `render("stats.html")` |
| **模板渲染** | | | | |
| 模板文件 | `stats.html.heex` | `stats.html.heex` | `stats.html.heex` | **完全相同** |
| `site_role` | `:admin/:editor/:viewer/:owner` | `:public` | `:public` | - |
| `shared_link_auth` | `nil` | `nil` | `slug` 值 | - |
| `limited_to_segment_id` | `nil` | `nil` | 关联的 Segment ID | - |
| **API 调用** | | | | |
| API 端点 | `/api/stats/:domain/*` | `/api/stats/:domain/*` | `/api/stats/:domain/*` | **完全相同** |
| Query 参数 | 无特殊 | 无特殊 | `?auth=slug` | - |
| Header | 无特殊 | 无特殊 | `X-Shared-Link-Auth: slug` | - |
| Pipeline | `:internal_stats_api` | `:internal_stats_api` | `:internal_stats_api` | **完全相同** |
| **查询执行** | | | | |
| 核心函数 | `Plausible.Stats.breakdown/4` | `Plausible.Stats.breakdown/4` | `Plausible.Stats.breakdown/4` | **完全相同** |
| `timeseries/3` | 相同 | 相同 | 相同 | **完全相同** |
| `aggregate/3` | 相同 | 相同 | 相同 | **完全相同** |
| **数据过滤** | | | | |
| Segment 强制验证 | 无 | 无 | **有** (`validate_required_filters_plug`) | - |
| 数据范围 | 完整站点数据 | 完整站点数据 | **Segment 过滤后** | - |

### 4.2 关键复用证据

**所有统计指标共用相同的查询模式** (`lib/plausible_web/controllers/api/stats_controller.ex`)：

```elixir
# 以下所有函数使用完全相同的模式：
# 1. 从 conn.assigns 获取 site
# 2. 调用 Plausible.Stats.* 函数
# 3. 返回 JSON

def current_visitors(conn, _params) do
  json(conn, Plausible.Stats.current_visitors(conn.assigns.site, conn))
end

def sources(conn, _params) do
  json(conn, Plausible.Stats.breakdown(conn.assigns.site, conn, :source, ...))
end

def pages(conn, _params) do
  json(conn, Plausible.Stats.breakdown(conn.assigns.site, conn, :page, ...))
end

# ... 其他所有指标

# 关键：没有任何基于 site_role 的分支！
# 所有访问方式走完全相同的代码路径
```

---

## 五、前端组件复用与差异对照

### 5.1 组件复用/差异对照表

| 组件 | 私有 Dashboard | 公开报表 | 嵌入模式 | 差异控制条件 |
|-----|---------------|----------|---------|-------------|
| **完全复用的组件** | | | | |
| Dashboard 主入口 | 完整功能 | 完整功能 | 完整功能 | 无差异 |
| VisitorGraph | 完整数据 | 完整数据 | Segment 过滤后 | 基于数据 props |
| TopStats | 完整数据 | 完整数据 | Segment 过滤后 | 基于数据 props |
| Sources | 完整数据 | 完整数据 | Segment 过滤后 | 基于数据 props |
| Pages | 完整数据 | 完整数据 | Segment 过滤后 | 基于数据 props |
| Locations | 完整数据 | 完整数据 | Segment 过滤后 | 基于数据 props |
| Devices | 完整数据 | 完整数据 | Segment 过滤后 | 基于数据 props |
| Behaviours | 完整数据 | 完整数据 | Segment 过滤后 | 基于数据 props |
| Conversions | 完整数据 | 完整数据 | Segment 过滤后 | 基于数据 props |
| **有条件差异的组件** | | | | |
| SiteSwitcher | 下拉菜单，可切换 | 静态显示域名 | 静态显示域名 | `user.loggedIn` + `user.role` |
| Segment 功能 | 可创建/编辑/查看详情 | 仅查看列表 | 仅查看列表 | `ROLES_WITH_*` 常量 |
| FiltersBar | 显示"Save as segment" | 隐藏 | 隐藏 | `showingSaveAsSegment` |
| TopBar | Sticky 导航 | Sticky 导航 | **非 Sticky** | `!site.embedded` |
| **路由与 API** | | | | |
| 路由 basepath | `/:domain` | `/share/:domain` | `/share/:domain` | `site.shared` |
| API 认证 | 无 token | `?auth=` + Header | `?auth=` + Header | `SHARED_LINK_AUTH` |

### 5.2 关键代码证据

**Segment 权限控制** (`assets/js/dashboard/filtering/segments.ts:8-21`)：

```typescript
// 可创建 Site Segment 的角色
const ROLES_WITH_MAYBE_SITE_SEGMENTS = [Role.admin, Role.editor, Role.owner]

// 可创建 Personal Segment 的角色
const ROLES_WITH_PERSONAL_SEGMENTS = [
  Role.billing, Role.viewer, Role.admin, Role.editor, Role.owner
]

// public 角色不在以上任一列表中！

// 能否保存为 Segment？
export const canSeeSaveAsSegmentAction = ({ user }) =>
  user.loggedIn && (
    ROLES_WITH_MAYBE_SITE_SEGMENTS.includes(user.role) ||
    ROLES_WITH_PERSONAL_SEGMENTS.includes(user.role)
  )
// → public 角色: false (且 not loggedIn)
```

---

## 六、关键代码位置索引

### 6.1 后端关键文件

| 文件 | 功能 | 关键行 |
|-----|-----|-------|
| **权限核心** | | |
| `lib/plausible_web/plugs/authorize_site_access.ex` | 核心权限检查插件 | L78-140 (`call/2`), L98-106 (role 条件判断) |
| `lib/plausible_web/router.ex` | 路由配置 | L36-41 (`:shared_link` pipeline), L72-78 (`:internal_stats_api`) |
| **控制器** | | |
| `lib/plausible_web/controllers/stats_controller.ex` | 页面渲染 | L49-115 (`stats/2`), L261-297 (`shared_link/2`) |
| `lib/plausible_web/controllers/site_controller.ex` | 站点管理 | L316-335 (`make_public/make_private`) |
| `lib/plausible_web/controllers/api/stats_controller.ex` | Stats API | L1331-1351 (`validate_required_filters_plug`) |
| **模型** | | |
| `lib/plausible/site.ex` | Site 模型 | L17 (`public` 字段), L165-171 (`make_public/make_private`) |
| `lib/plausible/site/shared_link.ex` | SharedLink 模型 | L1-58 (schema + 辅助函数) |

### 6.2 前端关键文件

| 文件 | 功能 | 关键行 |
|-----|-----|-------|
| **入口与初始化** | | |
| `assets/js/dashboard.tsx` | 前端入口初始化 | L1-112 (完整初始化流程) |
| `assets/js/dashboard/index.tsx` | Dashboard 入口 | 完整文件 (组件复用) |
| **Context 与状态** | | |
| `assets/js/dashboard/user-context.tsx` | 用户角色定义 | L3-10 (`Role` enum) |
| `assets/js/dashboard/site-context.tsx` | 站点配置解析 | L3-27 (`parseSiteFromDataset`) |
| **组件与 API** | | |
| `assets/js/dashboard/api.ts` | API 认证处理 | L9-107 (`SHARED_LINK_AUTH`, `setSharedLinkAuth`) |
| `assets/js/dashboard/router.tsx` | 路由配置 | L177-184 (`getRouterBasepath`) |
| `assets/js/dashboard/site-switcher.tsx` | 站点切换器 | L90-108 (登录状态检查) |
| `assets/js/dashboard/filtering/segments.ts` | Segment 权限 | L8-21 (`ROLES_WITH_*` 常量) |

---

## 七、常见问题解答

### Q1: 不登录时，数据是否真的被限制了？

**答：视情况而定。**

- 如果没有设置 Segment 限制：**数据完全可见**，只是功能被限制（无法创建 Segment、切换站点等）
- 如果设置了 Segment 限制：**数据被 Segment 过滤条件限制**，且无法绕过

**关键证据**：`AuthorizeSiteAccess` 只控制访问权限，不修改查询逻辑；所有 stats 接口共用相同的 `Plausible.Stats.*` 函数。

### Q2: 如何实现"不登录时只能看到部分指标"？

**答：通过 Segment 限制，而不是角色。**

Plausible 没有基于角色的指标过滤机制。要实现"只显示部分指标"，需要：

1. 创建一个 Segment，定义过滤条件（如 `country = US` 或 `source = google`）
2. 创建 SharedLink 时关联此 Segment
3. 访问此共享链接时，数据会被强制过滤

### Q3: 嵌入模式和普通共享链接的主要区别是什么？

**答：嵌入模式只是 UI 适配，权限逻辑相同。**

主要差异：
1. **iframe 支持**：`SecureEmbedHeaders` 插件移除 `X-Frame-Options`
2. **UI 适配**：禁用 sticky 导航、隐藏页脚
3. **自定义选项**：支持 `?theme=` 和 `?background=` 参数

权限逻辑完全复用共享链接机制。

---

## 八、关键设计原则总结

### 8.1 核心设计理念

1. **查询逻辑完全复用**
   - 所有访问方式使用完全相同的 `Plausible.Stats.*` 查询函数
   - 没有基于 `site_role` 的数据过滤分支
   - 确保数据一致性和代码可维护性

2. **权限控制在入口层**
   - `AuthorizeSiteAccess` 插件是唯一的权限决策点
   - 通过 `site_role` 控制 UI 组件和功能入口
   - 不影响底层数据查询

3. **唯一的数据级限制：Segment**
   - `shared_link.segment_id` 是唯一的数据范围限制机制
   - 通过 `validate_required_filters_plug` 强制验证
   - 用户无法绕过（API 层强制检查）

### 8.2 三种公开访问方式的适用场景

| 方式 | 适用场景 | 优势 | 劣势 |
|-----|---------|------|------|
| **公开站点** | 开源项目、社区项目、完全公开的数据 | URL 简洁，无需 token | 全站要么全公开要么全私有，无细粒度控制 |
| **共享链接** | 临时分享、客户报告、内部协作 | 可创建多个，可选密码保护，可选 Segment 限制 | URL 较长，需要管理 token |
| **嵌入模式** | 嵌入第三方网站、SaaS 产品集成 | 支持 iframe，可自定义主题背景 | 是共享链接的变体，依赖共享链接机制 |
