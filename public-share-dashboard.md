# Plausible 公开报表与嵌入报表实现分析

## 一、概述

Plausible 的公开报表（Public Dashboard）、嵌入报表（Embedded Dashboard）和公开站点（Public Site）三种访问方式，通过复用私有 Dashboard 的查询逻辑和前端组件，同时在权限层面进行精细控制，实现不登录时展示有限指标的需求。

**核心发现（修正前一轮不准确结论）：**

1. **共享链接页面访问不经过 AuthPlug**：`:shared_link` pipeline 没有 `:fetch_session` 和 `AuthPlug`，即使浏览器有 session cookie 也不会被解析。

2. **已登录用户访问共享链接时存在不一致**：页面渲染时 `site_role = :public`，但 API 调用时（走 `:internal_stats_api` pipeline）会解析 session，获得完整成员权限。

3. **页面入口和 API 入口的鉴权链路完全不同**：需要分开描述两种入口的 pipeline 配置。

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
│  ⚠️  注意：已登录用户可通过 API 绕过（因为 API 走 :internal_stats_api pipeline）│
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
| **页面入口 Pipeline** | `:browser` | `:browser` | `:shared_link` | `:shared_link` |
| **需登录** | 是 | 否 | 否 | 否 |
| **页面渲染时 site_role** | `:admin/:editor/:viewer/:owner` | `:public` | `:public` | `:public` |
| **API 调用时 site_role (登录用户)** | 成员角色 | 成员角色 | **成员角色** (关键发现) | **成员角色** (关键发现) |
| **API 调用时 site_role (未登录用户)** | - | `:public` | `:public` | `:public` |
| **数据范围** | 完整站点数据 | 完整站点数据 | 可选：Segment 限制 | 可选：Segment 限制 |

---

## 三、页面入口、鉴权链路、统计接口入口的准确差异对照

### 3.1 Pipeline 配置详细对比

**关键发现：页面入口和 API 入口的 Pipeline 配置完全不同！**

| Pipeline | `:fetch_session` | `AuthPlug` | `AuthorizeSiteAccess` | `SecureEmbedHeaders` | 使用场景 |
|----------|-----------------|------------|----------------------|---------------------|---------|
| **`:browser`** | ✅ 有 | ✅ 有 | ⚠️ 仅在控制器 plug 中 | ❌ 无 | 私有 Dashboard 页面、公开站点页面 |
| **`:shared_link`** | ❌ **无** | ❌ **无** | ❌ **无** | ✅ 有 | 共享链接页面、嵌入模式页面 |
| **`:internal_stats_api`** | ✅ 有 | ✅ 有 | ✅ 有 | ❌ 无 | 所有统计 API 接口 |

---

### 3.2 四种访问方式的完整链路对照

#### 对照总表

| 维度 | 私有 Dashboard | 公开站点 | 共享链接 | 嵌入模式 |
|-----|--------------|----------|----------|---------|
| **页面入口** | | | | |
| URL 路径 | `/:domain` | `/:domain` | `/share/:domain?auth=slug` | `/share/:domain?auth=slug&embed=true` |
| Pipeline | `:browser` | `:browser` | `:shared_link` | `:shared_link` |
| `:fetch_session` | ✅ 有 | ✅ 有 | ❌ **无** | ❌ **无** |
| `AuthPlug` | ✅ 有 | ✅ 有 | ❌ **无** | ❌ **无** |
| `AuthorizeSiteAccess` | ⚠️ 控制器 plug | ⚠️ 控制器 plug | ❌ 无 | ❌ 无 |
| 控制器函数 | `StatsController.stats/2` | `StatsController.stats/2` | `StatsController.shared_link/2` | `StatsController.shared_link/2` |
| **鉴权链路 (页面渲染时)** | | | | |
| `conn.assigns.current_user` | 实际登录用户 | 实际登录用户 (若有) | **nil** (关键发现) | **nil** (关键发现) |
| `site_role` 来源 | `AuthorizeSiteAccess` | `AuthorizeSiteAccess` | 硬编码 `:public` | 硬编码 `:public` |
| 页面渲染时 `site_role` | 成员角色 | 未登录时 = `:public` 登录时 = 成员角色 | **`:public`** (始终) | **`:public`** (始终) |
| **统计接口入口** | | | | |
| API 端点 | `/api/stats/:domain/*` | `/api/stats/:domain/*` | `/api/stats/:domain/*` | `/api/stats/:domain/*` |
| Pipeline | `:internal_stats_api` | `:internal_stats_api` | `:internal_stats_api` | `:internal_stats_api` |
| `:fetch_session` | ✅ 有 | ✅ 有 | ✅ 有 | ✅ 有 |
| `AuthPlug` | ✅ 有 | ✅ 有 | ✅ 有 | ✅ 有 |
| `AuthorizeSiteAccess` | ✅ 有 | ✅ 有 | ✅ 有 | ✅ 有 |
| **鉴权链路 (API 调用时 - 登录用户)** | | | | |
| `conn.assigns.current_user` | 实际登录用户 | 实际登录用户 | **实际登录用户** (关键发现) | **实际登录用户** (关键发现) |
| `membership_role` | 存在 | 存在 | **存在** (关键发现) | **存在** (关键发现) |
| `site_role` | 成员角色 | 成员角色 | **成员角色** (关键发现) | **成员角色** (关键发现) |
| **鉴权链路 (API 调用时 - 未登录用户)** | | | | |
| `conn.assigns.current_user` | - | nil | nil | nil |
| `membership_role` | - | nil | nil | nil |
| `site_role` | - | `site.public=true` → `:public` | `shared_link` 存在 → `:public` | `shared_link` 存在 → `:public` |
| **数据范围限制** | | | | |
| Segment 限制 | 无 | 无 | 可选 (API 层强制验证) | 可选 (API 层强制验证) |
| 限制来源 | 无 | 无 | `validate_required_filters_plug` | `validate_required_filters_plug` |

---

### 3.3 关键代码证据

#### Pipeline 定义 (`lib/plausible_web/router.ex`)

**`:browser` pipeline (L7-19)**：
```elixir
pipeline :browser do
  plug :accepts, ["html"]
  plug :fetch_session                    # ✅ 有
  plug :fetch_live_flash
  plug :put_secure_browser_headers
  plug PlausibleWeb.Plugs.NoRobots
  on_ee(do: nil, else: plug(PlausibleWeb.FirstLaunchPlug, redirect_to: "/register"))
  plug PlausibleWeb.AuthPlug              # ✅ 有
  on_ee(do: plug(Plausible.Plugs.HandleExpiredSession))
  on_ee(do: plug(Plausible.Plugs.SSOTeamAccess))
  plug PlausibleWeb.Plugs.UserSessionTouch
  plug :put_root_layout, html: {PlausibleWeb.LayoutView, :app}
end
```

**`:shared_link` pipeline (L36-41)**：
```elixir
pipeline :shared_link do
  plug :accepts, ["html"]
  plug PlausibleWeb.Plugs.SecureEmbedHeaders  # ✅ 独有
  plug PlausibleWeb.Plugs.NoRobots
  plug :put_root_layout, html: {PlausibleWeb.LayoutView, :app}
  # ❌ 关键：没有 :fetch_session、AuthPlug、AuthorizeSiteAccess！
end
```

**`:internal_stats_api` pipeline (L72-78)**：
```elixir
pipeline :internal_stats_api do
  plug :accepts, ["json"]
  plug :fetch_session                    # ✅ 有
  plug PlausibleWeb.AuthPlug              # ✅ 有
  plug PlausibleWeb.Plugs.AuthorizeSiteAccess  # ✅ 有
  plug PlausibleWeb.Plugs.NoRobots
end
```

#### 路由匹配 (`lib/plausible_web/router.ex:486-491`)

```elixir
scope "/", PlausibleWeb do
  pipe_through [:shared_link]  # ⚠️ 共享链接页面走此 pipeline

  get "/share/:domain/*path", StatsController, :shared_link
  post "/share/:slug/authenticate", StatsController, :authenticate_shared_link
end
```

#### 共享链接控制器 (`lib/plausible_web/controllers/stats_controller.ex`)

**`shared_link/2` 函数**：
```elixir
def shared_link(conn, %{"domain" => domain, "auth" => auth}) do
  # 注意：conn.assigns[:current_user] 是 nil！
  # 因为 :shared_link pipeline 没有 AuthPlug
  
  with {:ok, shared_link} <- find_shared_link(domain, auth) do
    if Plausible.Site.SharedLink.password_protected?(shared_link) do
      render_password_protected_shared_link(conn, shared_link)
    else
      render_shared_link(conn, shared_link)
    end
  else
    {:error, :not_found} -> render_error(conn, 404)
  end
end
```

**`render_shared_link/2` 函数中的 site_role**：
```elixir
defp render_shared_link(conn, shared_link) do
  # ...
  site_role = get_fallback_site_role(conn)  # ← 返回 :public
  # ...
end

defp get_fallback_site_role(conn),
  do: if(role = conn.assigns[:site_role], do: role, else: :public)
```

#### AuthorizeSiteAccess 决策逻辑 (`lib/plausible_web/plugs/authorize_site_access.ex:98-106`)

```elixir
role =
  cond do
    # 1. 成员关系优先 (最高优先级)
    membership_role -> membership_role
    
    # 2. 超管
    Plausible.Auth.is_super_admin?(current_user) -> :super_admin
    
    # 3. 公开站点
    site.public -> :public
    
    # 4. 共享链接
    shared_link -> :public
    
    # 5. 拒绝访问
    true -> nil
  end
```

**关键**：`membership_role` 优先级最高！这意味着：
- 登录用户访问共享链接的 API 时
- `:internal_stats_api` pipeline 有 `AuthPlug` → `current_user` 被解析
- `Teams.Memberships.site_role(site, current_user)` 返回成员角色
- `membership_role` 存在 → `site_role = 成员角色` (不是 :public！)

---

## 四、已登录用户访问共享链接的特殊场景

### 4.1 场景描述

**用户状态**：已登录（浏览器有有效的 session cookie）
**访问 URL**：`/share/mydomain?auth=abc123`（共享链接）

### 4.2 完整执行流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    已登录用户访问共享链接的执行流程                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  阶段 1: 页面请求 (GET /share/mydomain?auth=abc123)                         │
│  ─────────────────────────────────────────────────────────────────────────   │
│                                                                              │
│  请求进入                                                                     │
│      │                                                                       │
│      ▼                                                                       │
│  Pipeline: :shared_link                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ plug :accepts, ["html"]                                               │   │
│  │ plug PlausibleWeb.Plugs.SecureEmbedHeaders                           │   │
│  │ plug PlausibleWeb.Plugs.NoRobots                                     │   │
│  │ plug :put_root_layout, html: {...}                                    │   │
│  │                                                                       │   │
│  │ ❌ 关键：没有 :fetch_session → session cookie 不解析                    │   │
│  │ ❌ 关键：没有 AuthPlug → current_user 不设置                           │   │
│  │ ❌ 关键：没有 AuthorizeSiteAccess                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│      │                                                                       │
│      ▼                                                                       │
│  控制器: StatsController.shared_link/2                                       │
│      │                                                                       │
│      ├─> conn.assigns[:current_user] = nil  (因为没有 AuthPlug)            │
│      ├─> site_role = :public (硬编码)                                       │
│      └─> render("stats.html",                                                │
│            site_role: :public,                                              │
│            shared_link_auth: "abc123",                                      │
│            ...                                                              │
│          )                                                                  │
│                                                                              │
│  结果：页面渲染时，前端认为这是公开访问 (site_role = :public)                │
│        可能会隐藏某些功能按钮（如"保存为 Segment"）                          │
│                                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  阶段 2: API 请求 (GET /api/stats/mydomain/sources?auth=abc123)             │
│  ─────────────────────────────────────────────────────────────────────────   │
│                                                                              │
│  前端发起请求                                                                 │
│  (携带: query auth=abc123, header X-Shared-Link-Auth: abc123,              │
│         以及浏览器自动携带的 session cookie)                                  │
│      │                                                                       │
│      ▼                                                                       │
│  Pipeline: :internal_stats_api                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ plug :accepts, ["json"]                                               │   │
│  │ plug :fetch_session          ✅ → session cookie 被解析！             │   │
│  │ plug PlausibleWeb.AuthPlug    ✅ → current_user 被设置！              │   │
│  │ plug PlausibleWeb.Plugs.AuthorizeSiteAccess                           │   │
│  │ plug PlausibleWeb.Plugs.NoRobots                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│      │                                                                       │
│      ▼                                                                       │
│  AuthorizeSiteAccess.call/2                                                  │
│      │                                                                       │
│      ├─> current_user = 实际登录用户 (从 session 解析)                       │
│      ├─> Teams.Memberships.site_role(site, current_user)                     │
│      │   └─> 返回 {:ok, {member_type, :admin}} (假设是管理员)                │
│      ├─> membership_role = :admin (存在！)                                   │
│      │                                                                       │
│      └─> role = cond do                                                     │
│            membership_role -> membership_role  # ← 这里匹配！返回 :admin    │
│            site.public -> :public                                          │
│            shared_link -> :public                                          │
│            true -> nil                                                      │
│          end                                                                 │
│      │                                                                       │
│      └─> site_role = :admin (成员角色！不是 :public)                        │
│                                                                              │
│  结果：API 调用时，已登录用户获得完整的成员权限！                              │
│        即使通过共享链接访问，也能执行成员操作。                                │
│                                                                              │
│  ⚠️  关键不一致：                                                            │
│  - 页面渲染时：site_role = :public (前端可能隐藏功能)                         │
│  - API 调用时：site_role = :admin (实际获得成员权限)                         │
│                                                                              │
│  这意味着：前端隐藏的功能，已登录用户可通过直接调用 API 绕过。                │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 4.3 此场景的关键结论

| 维度 | 页面渲染时 | API 调用时 | 差异说明 |
|-----|-----------|-----------|---------|
| Pipeline | `:shared_link` | `:internal_stats_api` | 完全不同 |
| `fetch_session` | ❌ 无 | ✅ 有 | 关键差异 |
| `AuthPlug` | ❌ 无 | ✅ 有 | 关键差异 |
| `current_user` | `nil` | **实际登录用户** | 关键差异 |
| `membership_role` | 不检查 | **存在** | 关键差异 |
| `site_role` | `:public` | **成员角色** | **不一致** |
| 实际权限 | 公开访问 | **成员权限** | **不一致** |

**安全提示**：
- 前端根据 `site_role = :public` 隐藏的功能，已登录用户可通过直接调用 API 绕过
- 唯一的数据级保护是 `shared_link.segment_id` 的 `validate_required_filters_plug` 验证
- 这是设计意图还是边界情况？需要确认业务需求

---

## 五、Token 与站点权限的分叉决策顺序

### 5.1 API 层的决策优先级（`:internal_stats_api` pipeline）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    API 层权限决策优先级 (从高到低)                            │
│              (AuthorizeSiteAccess.call/2 在 :internal_stats_api pipeline)   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  入口: AuthorizeSiteAccess.call(conn, {allowed_roles, site_param})         │
│                                                                              │
│  前提条件（因 :internal_stats_api pipeline）：                               │
│  - :fetch_session → session cookie 被解析                                    │
│  - AuthPlug → current_user 被设置（如果登录）                                 │
│                                                                              │
│  决策顺序：                                                                   │
│                                                                              │
│  1. 成员关系检查 (最高优先级)                                                 │
│     ───────────────────────────────────────────────────────────────────────  │
│     Teams.Memberships.site_role(site, current_user)                         │
│                                                                              │
│     如果 current_user 是 site 的成员：                                       │
│     ┌─────────────────────────────────────────────────────────────────────┐  │
│     │ membership_role = :owner/:admin/:editor/:viewer (存在)             │  │
│     │ role = membership_role (优先！)                                      │  │
│     │ → 忽略 site.public 和 shared_link！                                   │  │
│     │ → 获得完整成员权限                                                    │  │
│     └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│     ⚠️  关键：已登录用户访问共享链接的 API 时，会走此分支！                    │
│         因为 :internal_stats_api pipeline 有 AuthPlug                         │
│                                                                              │
│  2. 超管检查                                                                 │
│     ───────────────────────────────────────────────────────────────────────  │
│     if Plausible.Auth.is_super_admin?(current_user) do                      │
│       role = :super_admin                                                    │
│     end                                                                      │
│                                                                              │
│  3. 公开站点检查                                                             │
│     ───────────────────────────────────────────────────────────────────────  │
│     if site.public == true do                                               │
│       role = :public                                                        │
│       → 未登录用户访问公开站点的 API 时走此分支                               │
│     end                                                                      │
│                                                                              │
│  4. 共享链接检查                                                             │
│     ───────────────────────────────────────────────────────────────────────  │
│     从以下位置获取 token：                                                   │
│     - conn.path_params["slug"] (URL 路径参数)                                │
│     - conn.params["auth"] (查询参数 ?auth=)                                  │
│     - "X-Shared-Link-Auth" 请求头                                            │
│     - Cookie: shared-link-{slug} (密码保护验证后)                             │
│                                                                              │
│     如果找到有效的 SharedLink 记录：                                         │
│     ┌─────────────────────────────────────────────────────────────────────┐  │
│     │ shared_link = %SharedLink{} (存在)                                  │  │
│     │ role = :public                                                       │  │
│     │ → 未登录用户访问共享链接的 API 时走此分支                              │  │
│     │                                                                      │  │
│     │ ⚠️  注意：如果已登录且是成员，不会走到这里！                          │  │
│     │      因为 membership_role 优先级更高                                  │  │
│     └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  5. 都不满足                                                                 │
│     ───────────────────────────────────────────────────────────────────────  │
│     role = nil → 返回 404                                                   │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 决策优先级速查表

| 优先级 | 条件 | 结果 role | 适用场景 |
|-------|------|----------|---------|
| **1 (最高)** | `membership_role` 存在 | `:owner/:admin/:editor/:viewer` | 登录用户且是站点成员 |
| **2** | 超管 | `:super_admin` | 系统管理员 |
| **3** | `site.public == true` | `:public` | 未登录用户访问公开站点 |
| **4 (最低)** | 有效 `shared_link` 存在 | `:public` | 未登录用户访问共享链接 |
| **-** | 都不满足 | `nil` → 404 | 拒绝访问 |

### 5.3 关键决策代码

```elixir
# lib/plausible_web/plugs/authorize_site_access.ex:98-106

# 这是整个权限系统的核心分叉点
role =
  cond do
    # 分支 A: 有成员关系 → 使用成员角色 (最高优先级)
    # ⚠️  关键：即使同时有公开站点或共享链接，成员关系优先
    membership_role -> membership_role
    
    # 分支 B: 超管
    Plausible.Auth.is_super_admin?(current_user) -> :super_admin
    
    # 分支 C: 站点公开 → 任何人均可访问
    site.public -> :public
    
    # 分支 D: 有有效共享链接
    shared_link -> :public
    
    # 分支 E: 都不满足 → 拒绝访问
    true -> nil
  end
```

---

## 六、查询链路和前端组件的复用/差异对照

### 6.1 查询链路复用/差异对照表

| 层级 | 私有 Dashboard | 公开站点 (未登录) | 共享链接 (未登录) | 共享链接 (已登录) |
|-----|--------------|------------------|------------------|------------------|
| **页面访问** | | | | |
| Pipeline | `:browser` | `:browser` | `:shared_link` | `:shared_link` |
| `fetch_session` | ✅ 有 | ✅ 有 | ❌ 无 | ❌ 无 |
| `AuthPlug` | ✅ 有 | ✅ 有 | ❌ 无 | ❌ 无 |
| `current_user` | 登录用户 | `nil` | `nil` | `nil` (页面层面) |
| `site_role` (页面渲染) | 成员角色 | `:public` | `:public` | `:public` |
| **API 访问** | | | | |
| Pipeline | `:internal_stats_api` | `:internal_stats_api` | `:internal_stats_api` | `:internal_stats_api` |
| `fetch_session` | ✅ 有 | ✅ 有 | ✅ 有 | ✅ 有 |
| `AuthPlug` | ✅ 有 | ✅ 有 | ✅ 有 | ✅ 有 |
| `current_user` | 登录用户 | `nil` | `nil` | **登录用户** |
| `membership_role` | 存在 | `nil` | `nil` | **存在** |
| `site_role` (API 层) | 成员角色 | `:public` | `:public` | **成员角色** |
| **查询执行** | | | | |
| 核心函数 | `Plausible.Stats.*` | `Plausible.Stats.*` | `Plausible.Stats.*` | `Plausible.Stats.*` |
| 复用性 | **完全复用** | **完全复用** | **完全复用** | **完全复用** |
| **数据限制** | | | | |
| Segment 强制验证 | 无 | 无 | 可选 | 可选 |
| 限制来源 | 无 | 无 | `validate_required_filters_plug` | `validate_required_filters_plug` |

### 6.2 前端组件复用/差异对照表

| 组件 | 私有 Dashboard | 公开报表 (site_role=:public) | 差异控制条件 |
|-----|--------------|------------------------------|-------------|
| **完全复用的组件** | | | |
| Dashboard 主入口 | 完整功能 | 完整功能 | 无差异 |
| VisitorGraph | 完整数据 | 完整数据 | 基于数据 props |
| TopStats | 完整数据 | 完整数据 | 基于数据 props |
| Sources/Pages/Locations/Devices | 完整数据 | 完整数据 | 基于数据 props |
| **有条件差异的组件** | | | |
| SiteSwitcher | 下拉菜单，可切换 | 静态显示域名 | `user.loggedIn` + `user.role` |
| Segment 功能 | 可创建/编辑/查看详情 | 仅查看列表 | `ROLES_WITH_*` 常量 |
| FiltersBar | 显示"Save as segment" | 隐藏 | `showingSaveAsSegment` |
| TopBar (嵌入模式) | Sticky 导航 | Sticky 导航 (非嵌入) / 非 Sticky (嵌入) | `!site.embedded` |

### 6.3 关键复用证据

**所有统计指标共用相同的查询模式** (`lib/plausible_web/controllers/api/stats_controller.ex`)：

```elixir
# 以下所有函数使用完全相同的模式：
# 1. 从 conn.assigns 获取 site
# 2. 调用 Plausible.Stats.* 函数
# 3. 返回 JSON
#
# ⚠️  关键：没有任何基于 site_role 的分支判断！
#       所有访问方式走完全相同的代码路径

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
```

---

## 七、关键代码位置索引

### 7.1 后端关键文件

| 文件 | 功能 | 关键行 |
|-----|-----|-------|
| **Router (核心 Pipeline 配置)** | | |
| `lib/plausible_web/router.ex:7-19` | `:browser` pipeline | ✅ 有 `fetch_session`、`AuthPlug` |
| `lib/plausible_web/router.ex:36-41` | `:shared_link` pipeline | ❌ **无** `fetch_session`、`AuthPlug` (关键发现) |
| `lib/plausible_web/router.ex:72-78` | `:internal_stats_api` pipeline | ✅ 有 `fetch_session`、`AuthPlug`、`AuthorizeSiteAccess` |
| `lib/plausible_web/router.ex:486-491` | 共享链接路由匹配 | 使用 `:shared_link` pipeline |
| **权限核心** | | |
| `lib/plausible_web/plugs/authorize_site_access.ex:78-140` | `call/2` 主逻辑 | 完整权限决策 |
| `lib/plausible_web/plugs/authorize_site_access.ex:98-106` | role 条件判断 | 核心分叉点 |
| `lib/plausible_web/plugs/authorize_site_access.ex:176-190` | `get_site_with_role/3` | 成员关系检查 |
| `lib/plausible_web/plugs/authorize_site_access.ex:200-220` | `maybe_get_shared_link/2` | 共享链接检查 |
| **控制器** | | |
| `lib/plausible_web/controllers/stats_controller.ex:49-115` | `stats/2` | 私有 Dashboard + 公开站点页面 |
| `lib/plausible_web/controllers/stats_controller.ex:261-297` | `shared_link/2` | 共享链接页面入口 (关键：无 AuthPlug) |
| `lib/plausible_web/controllers/stats_controller.ex:469-535` | `render_shared_link/2` | 共享链接页面渲染 |
| `lib/plausible_web/controllers/stats_controller.ex:534-535` | `get_fallback_site_role/1` | 硬编码返回 `:public` |
| `lib/plausible_web/controllers/api/stats_controller.ex:1331-1351` | `validate_required_filters_plug/2` | Segment 强制验证 |

### 7.2 前端关键文件

| 文件 | 功能 | 关键行 |
|-----|-----|-------|
| **入口与初始化** | | |
| `assets/js/dashboard.tsx` | 前端入口初始化 | `setSharedLinkAuth` 设置 |
| `assets/js/dashboard/user-context.tsx` | 用户角色定义 | `Role` enum |
| `assets/js/dashboard/api.ts` | API 认证处理 | `SHARED_LINK_AUTH` 变量 |
| **组件** | | |
| `assets/js/dashboard/site-switcher.tsx` | 站点切换器 | 登录状态检查 |
| `assets/js/dashboard/filtering/segments.ts` | Segment 权限 | `ROLES_WITH_*` 常量 |

---

## 八、常见问题解答

### Q1: 共享链接页面访问是否经过 AuthPlug？

**答：不经过。**

**关键证据** (`lib/plausible_web/router.ex:36-41`)：
```elixir
pipeline :shared_link do
  plug :accepts, ["html"]
  plug PlausibleWeb.Plugs.SecureEmbedHeaders
  plug PlausibleWeb.Plugs.NoRobots
  plug :put_root_layout, html: {PlausibleWeb.LayoutView, :app}
  # ❌ 没有 :fetch_session、AuthPlug、AuthorizeSiteAccess！
end
```

这意味着：
1. 即使浏览器有 session cookie，也不会被解析（没有 `:fetch_session`）
2. `conn.assigns.current_user` = `nil`
3. 页面渲染时 `site_role` = `:public`（硬编码）

### Q2: API 接口访问是否经过 AuthPlug？

**答：经过。**

**关键证据** (`lib/plausible_web/router.ex:72-78`)：
```elixir
pipeline :internal_stats_api do
  plug :accepts, ["json"]
  plug :fetch_session              # ✅ 有
  plug PlausibleWeb.AuthPlug        # ✅ 有
  plug PlausibleWeb.Plugs.AuthorizeSiteAccess  # ✅ 有
  plug PlausibleWeb.Plugs.NoRobots
end
```

这意味着：
1. 浏览器的 session cookie 会被解析（有 `:fetch_session`）
2. `conn.assigns.current_user` 会被设置（如果登录）
3. `AuthorizeSiteAccess` 会检查 `membership_role`（优先级最高）

### Q3: 已登录用户访问共享链接时，权限如何？

**答：存在不一致。**

| 阶段 | Pipeline | `current_user` | `site_role` | 实际权限 |
|-----|----------|----------------|-------------|---------|
| **页面渲染** | `:shared_link` | `nil` | `:public` | 公开访问（前端可能隐藏功能） |
| **API 调用** | `:internal_stats_api` | **实际登录用户** | **成员角色** | **完整成员权限** |

**关键原因**：
- 页面入口和 API 入口使用完全不同的 Pipeline
- `:internal_stats_api` pipeline 有 `AuthPlug` 会解析 session
- `AuthorizeSiteAccess` 中 `membership_role` 优先级最高

**安全影响**：
- 前端根据 `site_role = :public` 隐藏的功能，已登录用户可通过直接调用 API 绕过
- 唯一的数据级保护是 `shared_link.segment_id` 的 `validate_required_filters_plug` 验证

### Q4: 如何实现"不登录时只能看到部分指标"？

**答：通过 Segment 限制，而不是角色。**

Plausible 没有基于角色的指标过滤机制。要实现"只显示部分指标"，需要：

1. 创建一个 Segment，定义过滤条件（如 `country = US` 或 `source = google`）
2. 创建 SharedLink 时关联此 Segment
3. 访问此共享链接时，API 层会通过 `validate_required_filters_plug` 强制验证第一个 filter 必须是该 Segment

**关键代码** (`lib/plausible_web/controllers/api/stats_controller.ex:1331-1351`)：
```elixir
defp validate_required_filters_plug(
       %Plug.Conn{assigns: %{shared_link: %Plausible.Site.SharedLink{segment_id: segment_id}}} = conn,
       _opts
     ) when is_integer(segment_id) do
  # 强制验证第一个 filter 必须是指定的 Segment
  case conn.params |> get_filters_param() |> ensure_expected_segment_filter_present(segment_id) do
    :ok -> conn
    :error -> bad_request(conn, "The first filter must be for the segment with id #{segment_id}")
  end
end
```

### Q5: 公开站点和共享链接的主要区别是什么？

| 维度 | 公开站点 (Public Site) | 共享链接 (Shared Link) |
|-----|----------------------|----------------------|
| **URL 路径** | `/:domain` (与私有相同) | `/share/:domain?auth=slug` |
| **页面入口 Pipeline** | `:browser` | `:shared_link` |
| **fetch_session** | ✅ 有 | ❌ 无 |
| **AuthPlug** | ✅ 有 | ❌ 无 |
| **登录用户访问** | 获得成员权限 | **页面 = :public，API = 成员权限** (不一致) |
| **细粒度控制** | 全站要么全公开要么全私有 | 可创建多个链接，每个可独立配置 |
| **密码保护** | 不支持 | 支持 |
| **Segment 限制** | 不支持 | 支持 |
| **适用场景** | 开源项目、社区项目 | 临时分享、客户报告 |

---

## 九、关键设计原则总结

### 9.1 核心设计理念

1. **查询逻辑完全复用**
   - 所有访问方式使用完全相同的 `Plausible.Stats.*` 查询函数
   - 没有基于 `site_role` 的数据过滤分支
   - 确保数据一致性和代码可维护性

2. **权限控制在入口层（但入口不止一个！）**
   - **页面入口**：
     - 私有 Dashboard / 公开站点：走 `:browser` pipeline（有 AuthPlug）
     - 共享链接 / 嵌入模式：走 `:shared_link` pipeline（**无 AuthPlug**）
   - **API 入口**：
     - 所有统计接口：走 `:internal_stats_api` pipeline（有 AuthPlug + AuthorizeSiteAccess）
   - **关键发现**：页面入口和 API 入口的权限配置不同，可能导致不一致

3. **唯一的数据级限制：Segment**
   - `shared_link.segment_id` 是唯一的数据范围限制机制
   - 通过 `validate_required_filters_plug` 强制验证
   - 用户无法绕过（API 层强制检查）

### 9.2 权限决策优先级速记

```
┌─────────────────────────────────────────────────────────────────┐
│                    API 层权限决策优先级                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. membership_role (最高优先级)                                │
│     └─> 登录用户且是站点成员 → 成员角色                          │
│         (即使通过共享链接访问 API)                               │
│                                                                 │
│  2. super_admin                                                 │
│     └─> 系统管理员                                              │
│                                                                 │
│  3. site.public                                                 │
│     └─> 未登录用户访问公开站点 → :public                         │
│                                                                 │
│  4. shared_link (最低优先级)                                    │
│     └─> 未登录用户访问共享链接 → :public                         │
│         (如果已登录且是成员，不会走到这里)                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 9.3 三种公开访问方式的适用场景

| 方式 | 适用场景 | 优势 | 注意事项 |
|-----|---------|------|---------|
| **公开站点** | 开源项目、社区项目、完全公开的数据 | URL 简洁，无需 token | 全站要么全公开要么全私有，无细粒度控制 |
| **共享链接** | 临时分享、客户报告、内部协作 | 可创建多个，可选密码保护，可选 Segment 限制 | **已登录用户 API 调用会获得成员权限** (页面渲染不一致) |
| **嵌入模式** | 嵌入第三方网站、SaaS 产品集成 | 支持 iframe，可自定义主题背景 | 是共享链接的变体，依赖共享链接机制，注意权限不一致问题 |

### 9.4 重要边界情况

| 场景 | 行为 | 说明 |
|-----|------|------|
| **已登录用户访问共享链接页面** | 页面渲染 `site_role = :public` | 因 `:shared_link` pipeline 无 AuthPlug |
| **已登录用户访问共享链接 API** | **`site_role = 成员角色`** | 因 `:internal_stats_api` pipeline 有 AuthPlug |
| **已登录用户访问公开站点页面** | `site_role = 成员角色` | 因 `:browser` pipeline 有 AuthPlug + AuthorizeSiteAccess |
| **未登录用户访问公开站点页面** | `site_role = :public` | 因 `site.public = true` |
| **未登录用户访问共享链接页面** | `site_role = :public` | 硬编码 |

---

## 十、修正前一轮不准确结论

### 10.1 不准确结论 1："共享链接页面访问经过 AuthPlug"

**修正后**：**`:shared_link` pipeline 没有 AuthPlug 和 fetch_session！**

**证据** (`lib/plausible_web/router.ex:36-41`)：
```elixir
pipeline :shared_link do
  plug :accepts, ["html"]
  plug PlausibleWeb.Plugs.SecureEmbedHeaders
  plug PlausibleWeb.Plugs.NoRobots
  plug :put_root_layout, html: {PlausibleWeb.LayoutView, :app}
  # ❌ 没有 :fetch_session、AuthPlug、AuthorizeSiteAccess！
end
```

**影响**：
1. 即使浏览器有 session cookie，也不会被解析
2. `conn.assigns.current_user` = `nil`
3. 页面渲染时 `site_role` 硬编码为 `:public`

### 10.2 不准确结论 2："三种访问方式的 site_role 都是 :public"

**修正后**：**需要区分页面渲染时和 API 调用时的 site_role！**

| 访问方式 | 页面渲染时 site_role | API 调用时 (登录用户) site_role | API 调用时 (未登录用户) site_role |
|---------|---------------------|-------------------------------|-------------------------------|
| 私有 Dashboard | 成员角色 | 成员角色 | - |
| 公开站点 | 未登录时 = `:public` 登录时 = 成员角色 | 成员角色 | `:public` |
| 共享链接 | **`:public`** (始终) | **成员角色** (关键修正) | `:public` |
| 嵌入模式 | **`:public`** (始终) | **成员角色** (关键修正) | `:public` |

**关键原因**：
- 页面入口和 API 入口使用完全不同的 Pipeline
- `:internal_stats_api` pipeline 有 `AuthPlug` 会解析 session
- `AuthorizeSiteAccess` 中 `membership_role` 优先级最高

### 10.3 不准确结论 3："三种访问方式的鉴权链路相同"

**修正后**：**页面入口和 API 入口的鉴权链路完全不同，需要分开描述！**

**页面入口 Pipeline 对比**：

| Pipeline | `fetch_session` | `AuthPlug` | `AuthorizeSiteAccess` | 适用场景 |
|----------|-----------------|------------|----------------------|---------|
| `:browser` | ✅ 有 | ✅ 有 | ⚠️ 仅在控制器 plug 中 | 私有 Dashboard 页面、公开站点页面 |
| `:shared_link` | ❌ **无** | ❌ **无** | ❌ **无** | 共享链接页面、嵌入模式页面 |

**API 入口 Pipeline 对比**：

| Pipeline | `fetch_session` | `AuthPlug` | `AuthorizeSiteAccess` | 适用场景 |
|----------|-----------------|------------|----------------------|---------|
| `:internal_stats_api` | ✅ 有 | ✅ 有 | ✅ 有 | 所有统计 API 接口 |

**关键发现**：
- 共享链接页面访问和 API 访问使用不同的 Pipeline
- 这导致已登录用户访问共享链接时出现权限不一致
- 页面渲染时 `site_role = :public`，但 API 调用时 `site_role = 成员角色`

---

## 十一、修正后的完整架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    修正后的完整架构图                                        │
│         (强调页面入口和 API 入口的 Pipeline 差异)                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  用户请求                                                                    │
│  ─────────                                                                   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         页面入口 (Page Entry)                          │   │
│  │ ────────────────────────────────────────────────────────────────────│   │
│  │                                                                      │   │
│  │  私有 Dashboard / 公开站点            共享链接 / 嵌入模式            │   │
│  │  ───────────────────────            ───────────────────            │   │
│  │  GET /mydomain.com                  GET /share/mydomain?auth=abc123 │   │
│  │       │                                      │                      │   │
│  │       ▼                                      ▼                      │   │
│  │  ┌───────────────┐                  ┌──────────────────┐           │   │
│  │  │ :browser      │                  │ :shared_link     │           │   │
│  │  │ Pipeline      │                  │ Pipeline          │           │   │
│  │  ├───────────────┤                  ├──────────────────┤           │   │
│  │  │ ✅ fetch_session │                  │ ❌ fetch_session │ ⚠️ 关键    │   │
│  │  │ ✅ AuthPlug    │                  │ ❌ AuthPlug     │ ⚠️ 关键    │   │
│  │  │ ⚠️ Authorize   │                  │ ❌ Authorize    │ ⚠️ 关键    │   │
│  │  │    (控制器 plug) │                  │    SiteAccess    │           │   │
│  │  └───────┬───────┘                  └─────────┬────────┘           │   │
│  │          │                                    │                     │   │
│  │          ▼                                    ▼                     │   │
│  │  ┌─────────────────────────────────────────────────────────────┐   │   │
│  │  │                    控制器层 (Controller Layer)                │   │   │
│  │  ├─────────────────────────────────────────────────────────────┤   │   │
│  │  │                                                                 │   │   │
│  │  │  StatsController.stats/2        StatsController.shared_link/2 │   │   │
│  │  │  ─────────────────────        ────────────────────────────── │   │   │
│  │  │  current_user = 登录用户         current_user = nil (⚠️)       │   │   │
│  │  │  site_role = 成员角色             site_role = :public (硬编码)  │   │   │
│  │  │                                                                 │   │   │
│  │  │  └─────────────────────────────┬─────────────────────────────┘   │   │
│  │  │                                │                                  │   │   │
│  │  │                                ▼                                  │   │   │
│  │  │              ┌─────────────────────────────┐                    │   │   │
│  │  │              │ 共用模板: stats.html.heex   │                    │   │   │
│  │  │              │ (但 current_user/site_role 不同)                    │   │   │
│  │  │              └───────────────┬─────────────┘                    │   │   │
│  │  └──────────────────────────────┼──────────────────────────────────┘   │   │
│  │                                 │                                       │   │
│  └─────────────────────────────────┼───────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         前端渲染 (Frontend Render)                    │   │
│  │ ────────────────────────────────────────────────────────────────────│   │
│  │                                                                      │   │
│  │  从 data-* attributes 获取:                                           │   │
│  │  ├─> data-current-user-role = "admin" / "public"                    │   │
│  │  ├─> data-logged-in = "true" / "false"                               │   │
│  │  └─> data-shared-link-auth = "abc123" / nil                          │   │
│  │                                                                      │   │
│  │  前端组件条件渲染:                                                     │   │
│  │  ├─> SiteSwitcher: user.loggedIn && user.role != Role.public          │   │
│  │  ├─> Segment 功能: ROLES_WITH_* 常量检查                               │   │
│  │  └─> API 调用时设置: api.setSharedLinkAuth(sharedLinkAuth)            │   │
│  │                                                                      │   │
│  │  ⚠️  关键：前端根据 site_role = :public 可能隐藏某些功能按钮           │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         API 入口 (API Entry)                          │   │
│  │ ────────────────────────────────────────────────────────────────────│   │
│  │                                                                      │   │
│  │  所有统计 API 走相同的 Pipeline！                                      │   │
│  │  GET /api/stats/:domain/sources?auth=abc123                          │   │
│  │       │                                                               │   │
│  │       ▼                                                               │   │
│  │  ┌─────────────────────────────────────────────────────────────────┐ │   │
│  │  │ :internal_stats_api Pipeline                                      │ │   │
│  │  ├─────────────────────────────────────────────────────────────────┤ │   │
│  │  │ ✅ fetch_session  ← 关键：浏览器的 session cookie 会被解析！      │ │   │
│  │  │ ✅ AuthPlug       ← 关键：current_user 会被设置！(如果登录)       │ │   │
│  │  │ ✅ AuthorizeSiteAccess                                            │ │   │
│  │  └───────────────────────┬─────────────────────────────────────────┘ │   │
│  │                          │                                            │   │
│  │                          ▼                                            │   │
│  │  ┌─────────────────────────────────────────────────────────────────┐ │   │
│  │  │         AuthorizeSiteAccess.call/2 (API 层权限决策)              │ │   │
│  │  ├─────────────────────────────────────────────────────────────────┤ │   │
│  │  │                                                                   │ │   │
│  │  │  ⚠️  关键：membership_role 优先级最高！                            │ │   │
│  │  │                                                                   │ │   │
│  │  │  场景 A: 未登录用户访问                                            │ │   │
│  │  │  ─────────────────────────────                                    │ │   │
│  │  │  current_user = nil                                                │ │   │
│  │  │  membership_role = nil                                             │ │   │
│  │  │  ├─> 公开站点: site.public = true → site_role = :public            │ │   │
│  │  │  └─> 共享链接: shared_link 存在 → site_role = :public              │ │   │
│  │  │                                                                   │ │   │
│  │  │  场景 B: 已登录用户访问共享链接 API (关键修正！)                    │ │   │
│  │  │  ─────────────────────────────────────────────────────────────── │ │   │
│  │  │  current_user = 实际登录用户 (从 session 解析)                     │ │   │
│  │  │  Teams.Memberships.site_role(site, current_user)                   │ │   │
│  │  │    └─> 返回 {:ok, {member_type, :admin}}                           │ │   │
│  │  │  membership_role = :admin (存在！)                                 │ │   │
│  │  │                                                                   │ │   │
│  │  │  role = cond do                                                   │ │   │
│  │  │    membership_role -> membership_role  # ← 这里匹配！返回 :admin   │ │   │
│  │  │    site.public -> :public                                         │ │   │
│  │  │    shared_link -> :public                                         │ │   │
│  │  │  end                                                              │ │   │
│  │  │                                                                   │ │   │
│  │  │  site_role = :admin (成员角色！不是 :public)                       │ │   │
│  │  │                                                                   │ │   │
│  │  │  ⚠️  关键不一致：                                                  │ │   │
│  │  │  - 页面渲染时：site_role = :public (前端可能隐藏功能)               │ │   │
│  │  │  - API 调用时：site_role = :admin (实际获得成员权限)               │ │   │
│  │  │                                                                   │ │   │
│  │  └─────────────────────────────────────────────────────────────────┘ │   │
│  │                          │                                            │   │
│  │                          ▼                                            │   │
│  │  ┌─────────────────────────────────────────────────────────────────┐ │   │
│  │  │         唯一的数据级限制：validate_required_filters_plug           │ │   │
│  │  ├─────────────────────────────────────────────────────────────────┤ │   │
│  │  │                                                                   │ │   │
│  │  │  如果 shared_link.segment_id 存在：                                │ │   │
│  │  │  - 强制验证第一个 filter 必须是指定的 Segment                        │ │   │
│  │  │  - 无法绕过（API 层强制检查）                                        │ │   │
│  │  │                                                                   │ │   │
│  │  │  ⚠️  这是唯一的数据级保护机制                                        │ │   │
│  │  │     角色限制只是隐藏功能按钮，不限制数据查询                          │ │   │
│  │  │                                                                   │ │   │
│  │  └─────────────────────────────────────────────────────────────────┘ │   │
│  │                          │                                            │   │
│  │                          ▼                                            │   │
│  │  ┌─────────────────────────────────────────────────────────────────┐ │   │
│  │  │              核心查询层 (完全复用)                                 │ │   │
│  │  ├─────────────────────────────────────────────────────────────────┤ │   │
│  │  │                                                                   │ │   │
│  │  │  Plausible.Stats.breakdown/4                                     │ │   │
│  │  │  Plausible.Stats.timeseries/3                                    │ │   │
│  │  │  Plausible.Stats.query/2                                          │ │   │
│  │  │  Plausible.Stats.aggregate/2                                      │ │   │
│  │  │                                                                   │ │   │
│  │  │  ⚠️  关键：所有访问方式走完全相同的代码路径！                         │ │   │
│  │  │     没有基于 site_role 的分支判断                                   │ │   │
│  │  │                                                                   │ │   │
│  │  └─────────────────────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 十二、最终总结

### 12.1 三个最重要的发现

1. **共享链接页面访问不经过 AuthPlug**
   - `:shared_link` pipeline 没有 `:fetch_session` 和 `AuthPlug`
   - 即使浏览器有 session cookie，也不会被解析
   - 页面渲染时 `conn.assigns.current_user` = `nil`，`site_role` 硬编码为 `:public`

2. **API 入口和页面入口的 Pipeline 完全不同**
   - **页面入口**：共享链接走 `:shared_link` pipeline（无 AuthPlug）
   - **API 入口**：所有统计接口走 `:internal_stats_api` pipeline（有 AuthPlug + AuthorizeSiteAccess）
   - 这导致已登录用户访问共享链接时出现**权限不一致**

3. **已登录用户访问共享链接时的权限不一致**
   - **页面渲染时**：`site_role = :public`（前端可能隐藏功能按钮）
   - **API 调用时**：`site_role = 成员角色`（因 `:internal_stats_api` pipeline 有 AuthPlug，`membership_role` 优先级最高）
   - **安全影响**：前端隐藏的功能，已登录用户可通过直接调用 API 绕过
   - **唯一的数据级保护**：`shared_link.segment_id` 的 `validate_required_filters_plug` 验证

### 12.2 权限决策速记

```
页面入口 Pipeline 差异：
┌─────────────────────────────────────────────────────────────────┐
│ :browser          │ :shared_link      │ :internal_stats_api    │
├─────────────────────────────────────────────────────────────────┤
│ ✅ fetch_session │ ❌ fetch_session │ ✅ fetch_session        │
│ ✅ AuthPlug      │ ❌ AuthPlug      │ ✅ AuthPlug           │
│ ⚠️ 控制器 plug   │ ❌ AuthorizeSA   │ ✅ AuthorizeSA        │
├─────────────────────────────────────────────────────────────────┤
│ 私有/公开页面    │ 共享/嵌入页面   │ 所有统计 API          │
└─────────────────────────────────────────────────────────────────┘

API 层权限决策优先级（从高到低）：
┌─────────────────────────────────────────────────────────────────┐
│ 1. membership_role (最高) → 登录用户且是站点成员 → 成员角色     │
│ 2. super_admin                → 系统管理员                        │
│ 3. site.public                → 未登录用户访问公开站点 → :public │
│ 4. shared_link (最低)         → 未登录用户访问共享链接 → :public │
│    ⚠️ 注意：如果已登录且是成员，不会走到这里！                    │
└─────────────────────────────────────────────────────────────────┘
```

### 12.3 设计意图 vs 实际行为

| 设计意图 | 实际行为 | 说明 |
|---------|---------|------|
| 共享链接应该是"公开访问" | **页面渲染时是，但 API 调用时已登录用户获得成员权限** | Pipeline 配置不同导致不一致 |
| `site_role = :public` 应该限制数据查询 | **角色限制只隐藏功能按钮，不限制数据查询** | 没有基于角色的行级过滤 |
| 唯一的数据级保护 | **`shared_link.segment_id` + `validate_required_filters_plug`** | 这是真正有效的数据限制机制 |

### 12.4 安全建议

1. **如果需要真正的数据限制**：
   - 必须使用 `shared_link.segment_id` 关联一个 Segment
   - `validate_required_filters_plug` 会强制验证第一个 filter 必须是该 Segment
   - 这是唯一无法绕过的数据级保护

2. **共享链接的"隐藏功能"只是 UI 隐藏**：
   - 已登录用户可通过直接调用 API 绕过
   - 如果需要真正限制功能，应该在 API 层添加额外验证

3. **公开站点 vs 共享链接的选择**：
   - **公开站点**：适合完全公开的数据，登录用户获得完整权限
   - **共享链接**：适合临时分享，注意已登录用户的 API 权限问题
   - **嵌入模式**：适合第三方集成，注意权限不一致问题
