# Plausible 公开报表与嵌入报表实现分析

## 一、概述

Plausible 的公开报表（Public Dashboard）和嵌入报表（Embedded Dashboard）基于共享链接（Shared Link）机制实现，通过复用私有 Dashboard 的查询逻辑和前端组件，同时在权限层面进行精细控制，实现不登录时展示有限指标的需求。

---

## 二、权限控制架构

### 2.1 核心权限分叉点

**1. 路由层分叉 (`lib/plausible_web/router.ex`)**

```
┌─────────────────────────────────────────────────────────────────┐
│                        路由入口层                                │
├─────────────────────────────────────────────────────────────────┤
│  私有 Dashboard:                                                 │
│  GET /:domain/*path                                              │
│    └─ pipeline: :browser + AuthorizeSiteAccess                  │
│         (需要登录 + 站点成员权限)                                 │
├─────────────────────────────────────────────────────────────────┤
│  公开/嵌入报表:                                                   │
│  GET /share/:domain/*path                                        │
│    └─ pipeline: :shared_link                                     │
│         (无 AuthorizeSiteAccess, 无 AuthPlug)                   │
│         ├─ SecureEmbedHeaders (移除 X-Frame-Options)            │
│         └─ NoRobots                                               │
└─────────────────────────────────────────────────────────────────┘
```

**2. 控制器层分叉 (`lib/plausible_web/controllers/stats_controller.ex`)**

| 入口函数 | 调用场景 | 权限检查 |
|---------|---------|---------|
| `stats/2` | 私有 Dashboard | `AuthorizeSiteAccess` + `:admin/:editor/:viewer/:owner` |
| `shared_link/2` | 公开/嵌入报表 | 检查 SharedLink 存在性 + 密码保护 |

**关键代码 - 控制器入口差异：**
```elixir
# 私有 Dashboard - 需要授权插件
plug(PlausibleWeb.Plugs.AuthorizeSiteAccess when action in [:stats, :csv_export])

# 公开报表 - 无授权插件，自行处理
def shared_link(conn, %{"domain" => domain, "auth" => auth}) do
  case find_shared_link(domain, auth) do
    {:ok, shared_link} ->
      if password_protected?(shared_link) do
        render_password_protected_shared_link(conn, shared_link)
      else
        render_shared_link(conn, shared_link)  # 核心渲染函数
      end
    {:error, :not_found} -> render_error(conn, 404)
  end
end
```

---

### 2.2 SharedLink 模型

**模型定义 (`lib/plausible/site/shared_link.ex`)：**
```elixir
schema "shared_links" do
  belongs_to :site, Plausible.Site
  field :name, :string           # 链接名称
  field :slug, :string           # 公开 token (auth 参数)
  field :password_hash, :string  # 密码保护 (可选)
  belongs_to :segment, Plausible.Segments.Segment  # 限制到特定 segment
  timestamps()
end
```

**核心属性：**
- `slug` - 公开访问的 token，URL 中以 `?auth=slug` 形式传递
- `segment_id` - 可选，限制报表只展示特定 segment 的数据
- `password_hash` - 可选，启用密码保护

---

### 2.3 权限判定核心逻辑

**`AuthorizeSiteAccess` 插件 (`lib/plausible_web/plugs/authorize_site_access.ex:78-140`)：**

这是 API 层的权限分叉点，私有和公开报表共用此插件，但行为不同：

```elixir
def call(conn, {allowed_roles, site_param}) do
  current_user = conn.assigns[:current_user]
  
  with {:ok, domain} <- get_domain(conn, site_param),
       {:ok, %{site: site, role: membership_role, member_type: member_type}} <-
         get_site_with_role(conn, current_user, domain),
       :ok <- ensure_consolidated_view_access(conn, site),
       {:ok, shared_link} <- maybe_get_shared_link(conn, site) do
    
    # 核心权限判定逻辑
    role =
      cond do
        membership_role -> membership_role                    # 1. 有成员关系 → 使用成员角色
        Plausible.Auth.is_super_admin?(current_user) -> :super_admin  # 2. 超管
        site.public -> :public                                # 3. 站点公开 → public 角色
        shared_link -> :public                                # 4. 有共享链接 → public 角色
        true -> nil
      end
    
    if role in allowed_roles do
      # 授权通过，设置 conn.assigns
      merge_assigns(conn, site: site, site_role: role, shared_link: shared_link)
    else
      error_not_found(conn)
    end
  end
end
```

**权限判定优先级：**
1. **成员关系角色** > 2. **超管** > 3. **公开站点** > 4. **共享链接**

**共享链接获取逻辑 (`maybe_get_shared_link/2`)：**
```elixir
defp maybe_get_shared_link(conn, site) do
  slug = conn.path_params["slug"] || conn.params["auth"]  # 从 URL 获取 token
  
  if valid_path_fragment?(slug) do
    with %Plausible.Site.SharedLink{} = shared_link <-
           Repo.get_by(Plausible.Site.SharedLink, slug: slug, site_id: site.id),
         # 检查密码保护
         {%{password_protected?: true}, shared_link} <-
           {%{password_protected?: password_protected?(shared_link)}, shared_link},
         {:ok, shared_link} <-
           PlausibleWeb.StatsController.validate_shared_link_password(conn, shared_link) do
      {:ok, shared_link}
    else
      {%{password_protected?: false}, shared_link} -> {:ok, shared_link}  # 无密码直接通过
      {:error, :unauthorized} -> error_not_found(conn)
      nil -> error_not_found(conn)
    end
  else
    {:ok, nil}  # 无 token，继续其他检查
  end
end
```

---

## 三、页面渲染与组件复用

### 3.1 渲染入口统一

**私有和公开报表共用同一个模板！**

```
私有 Dashboard:
  └─ stats/2 → render("stats.html", ...)

公开/嵌入报表:
  └─ shared_link/2 → render_shared_link/2 → render("stats.html", ...)
```

**`render_shared_link/2` 核心代码 (`lib/plausible_web/controllers/stats_controller.ex:442-532`)：**

```elixir
defp render_shared_link(conn, shared_link) do
  # 1. 检查团队锁定和共享链接功能可用性
  cond do
    Teams.locked?(shared_link.site.team) -> render "site_locked.html"
    not shared_links_feature_access? -> render "site_locked.html" (仅共享链接权限缺失)
    not Teams.locked?(shared_link.site.team) ->
      # 2. 获取基础数据（与私有 Dashboard 相同）
      current_user = conn.assigns[:current_user]
      site_role = get_fallback_site_role(conn)  # → :public
      stats_start_date = Plausible.Sites.stats_start_date(shared_link.site)
      
      # 3. 处理 Segment 限制
      limited_to_segment_id =
        if Plausible.Site.SharedLink.limited_to_segment?(shared_link) do
          shared_link.segment.id
        else
          nil
        end
      
      # 4. 检测嵌入模式
      embedded? = conn.params["embed"] == "true"
      
      # 5. 渲染相同的模板！
      render(conn, "stats.html",
        site: shared_link.site,
        site_role: site_role,                    # ← :public
        has_goals: Sites.has_goals?(shared_link.site),
        # ... 其他相同参数
        shared_link_auth: shared_link.slug,     # ← 关键：传递 token
        embedded: embedded?,                     # ← 嵌入标识
        background: conn.params["background"],
        theme: conn.params["theme"],
        # ...
        limited_to_segment_id: limited_to_segment_id  # ← Segment 限制
      )
  end
end

# 获取 fallback 角色
defp get_fallback_site_role(conn),
  do: if(role = conn.assigns[:site_role], do: role, else: :public)
```

### 3.2 模板参数对比

| 参数 | 私有 Dashboard | 公开报表 | 嵌入报表 |
|-----|--------------|---------|---------|
| `site_role` | `:admin/:editor/:viewer/:owner` | `:public` | `:public` |
| `shared_link_auth` | `nil` | `slug` 值 | `slug` 值 |
| `embedded` | `nil` | `nil` | `true` |
| `hide_footer?` | 基于 `site_role != :public` | `true` (CE 除外) | `true` |
| `limited_to_segment_id` | `nil` | 可能有值 | 可能有值 |

**关键差异 - hide_footer 参数：**
```elixir
# 私有 Dashboard
hide_footer?: if(ce?() || demo, do: false, else: site_role != :public)

# 公开/嵌入报表
hide_footer?: if(ce?(), do: embedded?, else: embedded? || site_role != :public)
# 即: 嵌入模式 或 非 public 角色时隐藏
# 公开报表 (site_role=:public, embedded=false): hide_footer? = false (CE: false / EE: true)
```

---

### 3.3 模板传递到前端

**`stats.html.heex` 模板中的 data 属性：**

```heex
<div id="stats-react-container"
     data-domain={@site.domain}
     data-has-goals={to_string(@has_goals)}
     data-logged-in={to_string(!!@conn.assigns[:current_user])}
     data-shared-link-auth={assigns[:shared_link_auth]}  <%-- 关键：token --%>
     data-embedded={to_string(@conn.assigns[:embedded])}    <%-- 嵌入标识 --%>
     data-background={@conn.assigns[:background]}
     data-current-user-role={@site_role}                   <%-- :public --%>
     data-current-user-id={...}
     data-flags={Jason.encode!(@flags)}
     data-segments={Jason.encode!(@segments)}
     data-limited-to-segment-id={Jason.encode!(@limited_to_segment_id)}>
</div>
```

---

## 四、前端组件与条件渲染

### 4.1 前端初始化流程

**`assets/js/dashboard.tsx`：**

```typescript
const container = document.getElementById('stats-react-container')

if (container && container.dataset) {
  // 1. 解析站点配置
  const site = parseSiteFromDataset(container.dataset)
  
  // 2. 设置共享链接 token (关键！)
  const sharedLinkAuth = container.dataset.sharedLinkAuth
  if (sharedLinkAuth) {
    api.setSharedLinkAuth(sharedLinkAuth)  // 所有后续 API 调用携带此 token
  }
  
  // 3. 解析 Segment 限制
  const limitedToSegmentId = parseLimitedToSegmentId(container.dataset)
  const limitedToSegment = getLimitedToSegment(limitedToSegmentId, preloadedSegments)
  
  // 4. 初始化用户上下文
  <UserContextProvider
    user={
      container.dataset.loggedIn === 'true'
        ? { loggedIn: true, id: ..., role: dataset.currentUserRole as Role, ... }
        : { loggedIn: false, id: null, role: dataset.currentUserRole as Role, ... }
    }
  >
    {/* 5. 创建路由 (路径区分) */}
    <RouterProvider router={createAppRouter(site)} />
  </UserContextProvider>
}
```

### 4.2 角色定义与权限控制

**`assets/js/dashboard/user-context.tsx`：**

```typescript
export enum Role {
  owner = 'owner',
  admin = 'admin',
  viewer = 'viewer',
  editor = 'editor',
  public = 'public',    // ← 公开/嵌入报表使用此角色
  billing = 'billing'
}
```

**基于角色的权限控制 (`assets/js/dashboard/filtering/segments.ts`)：**

```typescript
// 可创建 Site Segment 的角色
const ROLES_WITH_MAYBE_SITE_SEGMENTS = [Role.admin, Role.editor, Role.owner]

// 可创建 Personal Segment 的角色
const ROLES_WITH_PERSONAL_SEGMENTS = [
  Role.billing, Role.viewer, Role.admin, Role.editor, Role.owner
]

// 能否保存为 Segment？
export const canSeeSaveAsSegmentAction = ({ user }) =>
  user.loggedIn && (
    ROLES_WITH_MAYBE_SITE_SEGMENTS.includes(user.role) ||
    ROLES_WITH_PERSONAL_SEGMENTS.includes(user.role)
  )
// → public 角色: false (且 not loggedIn)

// 能否查看 Segment 详情？
export function canSeeSegmentDetails({ user }) {
  return user.loggedIn && user.role !== Role.public
}
// → public 角色: false

// 能否列出 Personal Segment？
export function isListableSegment({ segment, site, user }) {
  if (segment.type === SegmentType.personal) {
    if (!user.loggedIn || user.id === null || user.role === Role.public) {
      return false  // public 角色看不到 personal segments
    }
    return segment.owner_id === user.id
  }
  return false
}
```

### 4.3 SiteContext 解析

**`assets/js/dashboard/site-context.tsx`：**

```typescript
export function parseSiteFromDataset(dataset: DOMStringMap): PlausibleSite {
  return {
    domain: dataset.domain!,
    // ...
    embedded: dataset.embedded === 'true',        // ← 嵌入标识
    background: dataset.background,
    shared: !!dataset.sharedLinkAuth,              // ← 是否为共享链接
    isConsolidatedView: dataset.isConsolidatedView === 'true'
  }
}

// site.shared = true 时，表示是公开报表访问
```

### 4.4 路由路径区分

**`assets/js/dashboard/router.tsx`：**

```typescript
export function getRouterBasepath(
  site: Pick<PlausibleSite, 'shared' | 'domain'>
): string {
  const basepath = site.shared
    ? `/share/${encodeURIComponent(site.domain)}`  // 公开报表路径
    : `/${encodeURIComponent(site.domain)}`         // 私有报表路径
  return basepath
}
```

### 4.5 组件条件渲染示例

**SiteSwitcher 组件 (`assets/js/dashboard/site-switcher.tsx:90-108`)：**

```typescript
export const SiteSwitcher = () => {
  const user = useUserContext()
  
  // 只有登录用户才能获取站点列表和切换
  const sitesQuery = useQuery({
    enabled: user.loggedIn,  // ← public 角色: not loggedIn → 禁用
    queryKey: ['sites'],
    queryFn: async () => await get('/api/sites')
  })

  if (!user.loggedIn) {
    return <SiteSwitcherStatic />  // ← 静态展示，无下拉菜单
  }
  
  // 登录用户：展示完整的站点切换器
  const canSeeSiteSettings: boolean =
    user.loggedIn &&
    [Role.owner, Role.admin, Role.editor, 'super_admin'].includes(user.role)
  // ...
}
```

**TopBar 组件 (`assets/js/dashboard/nav-menu/top-bar.tsx:33-43`)：**

```typescript
// 嵌入模式下禁用 sticky 效果
className={classNames(
  'col-span-full relative top-0 py-2 -my-3 sm:-my-4 z-10',
  !site.embedded && !inView && 'sticky bg-gray-50 ...'  // ← 嵌入时不 sticky
)}
```

---

## 五、API 层认证机制

### 5.1 API 调用流程

**前端 API 层 (`assets/js/dashboard/api.ts`)：**

```typescript
let SHARED_LINK_AUTH: null | string = null

// 初始化时设置
export function setSharedLinkAuth(auth: string) {
  SHARED_LINK_AUTH = auth
}

// 每次请求添加认证信息
function getHeaders(): Record<string, string> {
  return SHARED_LINK_AUTH ? { 'X-Shared-Link-Auth': SHARED_LINK_AUTH } : {}
}

// 同时在查询参数中添加 auth (双重保障)
function getSharedLinkSearchParams(): Record<string, string> {
  return SHARED_LINK_AUTH ? { auth: SHARED_LINK_AUTH } : {}
}

export async function get(url, dashboardState?, ...extraQueryParams) {
  const queryString = dashboardState
    ? dashboardStateToSearchParams(dashboardState, [...extraQueryParams])
    : serializeUrlParams(getSharedLinkSearchParams())  // 添加 auth=slug

  const response = await fetch(queryString ? `${url}?${queryString}` : url, {
    headers: { ...getHeaders(), Accept: 'application/json' }  // 添加 X-Shared-Link-Auth 头
  })
  return handleApiResponse(response)
}
```

### 5.2 后端 API 权限检查

**路由配置 (`lib/plausible_web/router.ex:72-78`)：**

```elixir
pipeline :internal_stats_api do
  plug :accepts, ["json"]
  plug :fetch_session
  plug PlausibleWeb.AuthPlug
  plug PlausibleWeb.Plugs.AuthorizeSiteAccess  # ← 关键！所有 stats API 都要经过
  plug PlausibleWeb.Plugs.NoRobots
end

# 所有 stats API 走此 pipeline
scope "/api" do
  pipe_through :internal_stats_api
  
  scope "/stats", PlausibleWeb.Api do
    post "/:domain/query", StatsController, :query
    get "/:domain/current-visitors", StatsController, :current_visitors
    get "/:domain/sources", StatsController, :sources
    # ... 所有其他指标接口
  end
end
```

### 5.3 API 权限分叉点详解

这是最关键的分叉点：**私有和公开报表使用完全相同的 API 端点，但通过不同的认证方式通过 `AuthorizeSiteAccess` 检查。**

**对比：**

| 访问类型 | 认证方式 | AuthorizeSiteAccess 中的 role |
|---------|---------|------------------------------|
| 私有 Dashboard | Session Cookie + User + Site Membership | `:admin/:editor/:viewer/:owner` |
| 公开报表 | `?auth=slug` 或 `X-Shared-Link-Auth` | `:public` (来自 shared_link) |
| 公开站点 | `site.public = true` (无 token) | `:public` |

**AuthorizeSiteAccess 如何处理 API 请求中的 token：**

```elixir
# 步骤 1: 从请求中获取 site (通过 domain)
{:ok, %{site: site, role: membership_role, member_type: member_type}} <-
  get_site_with_role(conn, current_user, domain)
# 此时 membership_role 可能是 nil (非成员访问)

# 步骤 2: 查找 shared_link (关键！)
{:ok, shared_link} <- maybe_get_shared_link(conn, site)
# 从 conn.params["auth"] (查询参数) 获取 token

# 步骤 3: 确定角色
role =
  cond do
    membership_role -> membership_role  # 成员优先
    # ... 超管检查 ...
    site.public -> :public              # 公开站点
    shared_link -> :public              # ← 有 shared_link → public 角色！
    true -> nil
  end

# 步骤 4: 检查角色是否在允许列表中
if role in allowed_roles do
  # 允许访问。allowed_roles 默认为 @all_roles，包含 :public
  merge_assigns(conn, site: site, site_role: role, shared_link: shared_link)
else
  error_not_found(conn)
end
```

**关键设计：**
- `AuthorizeSiteAccess` 默认允许所有角色（包含 `:public`）
- 只要有有效的 `shared_link`，就会获得 `:public` 角色
- 所有 `/api/stats/*` 端点使用相同的查询逻辑（`Plausible.Stats.breakdown/4` 等）

### 5.4 Segment 限制的 API 层验证

**`lib/plausible_web/controllers/api/stats_controller.ex:1331-1351`：**

当 SharedLink 关联了 Segment 时，API 会强制验证第一个 filter 必须是该 Segment：

```elixir
defp validate_required_filters_plug(
       %Plug.Conn{assigns: %{shared_link: %Plausible.Site.SharedLink{segment_id: segment_id}}} = conn,
       _opts
     ) when is_integer(segment_id) do
  case conn.params
       |> get_filters_param()
       |> ensure_expected_segment_filter_present(segment_id) do
    :ok -> conn
    :error ->
      bad_request(
        conn,
        "The first filter must be for the segment with id #{segment_id}"
      )
  end
end

defp validate_required_filters_plug(conn, _opts), do: conn  # 无 Segment 限制时不检查

# 验证逻辑
defp ensure_expected_segment_filter_present(filters, expected_segment_id) when is_list(filters) do
  case filters do
    [["is", "segment", [segment_id]] | _other_filters] when segment_id == expected_segment_id ->
      :ok
    _ -> :error
  end
end
```

---

## 六、嵌入报表 (Embedded) 特殊处理

### 6.1 启用方式

通过在共享链接 URL 后添加 `?embed=true` 参数：

```
普通公开报表:   /share/mydomain.com?auth=abc123
嵌入报表:       /share/mydomain.com?auth=abc123&embed=true&theme=dark&background=%23ffffff
```

### 6.2 额外支持的参数

| 参数 | 作用 | 示例 |
|-----|-----|-----|
| `embed` | 启用嵌入模式 | `true` |
| `theme` | 主题设置 | `light` / `dark` |
| `background` | 背景色 (URL 编码) | `%23ffffff` (白色) |

### 6.3 后端处理

**`render_shared_link/2` 中：**

```elixir
embedded? = conn.params["embed"] == "true"

render(conn, "stats.html",
  # ...
  embedded: embedded?,
  background: conn.params["background"],
  theme: conn.params["theme"],
  hide_footer?: if(ce?(), do: embedded?, else: embedded? || site_role != :public)
)
```

### 6.4 前端嵌入适配

**路由中的 Secure Headers：**
```elixir
# router.ex
pipeline :shared_link do
  plug PlausibleWeb.Plugs.SecureEmbedHeaders  # ← 关键：允许 iframe 嵌入
  # ...
end
```

**`SecureEmbedHeaders` 插件行为：**
- 移除 `X-Frame-Options` 响应头（或设置为 `ALLOW-FROM`）
- 允许页面在 iframe 中加载

---

## 七、数据查询逻辑复用

### 7.1 查询层完全复用

公开报表和私有报表使用**完全相同**的底层查询函数：

```
API 控制器层
├── PlausibleWeb.Api.StatsController
│   ├── sources/2      → Plausible.Stats.breakdown/4
│   ├── pages/2        → Plausible.Stats.breakdown/4
│   ├── countries/2    → Plausible.Stats.breakdown/4
│   └── ... 所有其他指标
│
└── Plausible.Stats (核心查询层，完全复用)
    ├── breakdown/4
    ├── timeseries/3
    ├── aggregate/3
    └── query/2
```

### 7.2 唯一的数据限制方式

**数据限制仅通过两种方式实现：**

1. **Site 本身的公开/私有设置**（`site.public`）
   - 不影响数据查询范围，只影响访问权限

2. **SharedLink 的 Segment 限制**（`shared_link.segment_id`）
   - 通过 API 层的 `validate_required_filters_plug` 强制
   - 前端会自动添加 `limited_to_segment_id` 对应的 filter
   - 查询时 Segment 会被解析为其包含的实际 filters

**Segment 解析过程 (`assets/js/dashboard/filtering/segments.ts:167-187`)：**

```typescript
export function resolveFilters(
  filters: Filter[],
  segments: Array<Pick<SavedSegment, 'id'> & { segment_data: SegmentData }>
): Filter[] {
  return filters.flatMap((filter): Filter[] => {
    if (isSegmentFilter(filter)) {  // ["is", "segment", [id]]
      const segment = segments.find(
        (segment) => String(segment.id) == String(clauses[0])
      )
      return segment ? segment.segment_data.filters : [filter]  // 展开为实际 filters
    } else {
      return [filter]
    }
  })
}
```

---

## 八、前端组件复用清单

### 8.1 完全复用的组件

所有核心可视化组件完全复用，通过 props/context 控制行为：

| 组件 | 路径 | 公开报表行为 |
|-----|-----|------------|
| Dashboard 主入口 | `dashboard/index.tsx` | 完全复用，无差异 |
| VisitorGraph | `dashboard/stats/graph/visitor-graph.tsx` | 完全复用 |
| TopStats | `dashboard/stats/graph/top-stats.js` | 完全复用 |
| Sources | `dashboard/stats/sources/index.js` | 完全复用 |
| Pages | `dashboard/stats/pages/index.js` | 完全复用 |
| Locations | `dashboard/stats/locations/index.js` | 完全复用 |
| Devices | `dashboard/stats/devices/index.js` | 完全复用 |
| Behaviours | `dashboard/stats/behaviours/index.js` | 完全复用 |
| Conversions | `dashboard/stats/behaviours/conversions.js` | 完全复用 |

### 8.2 有条件渲染差异的组件

| 组件 | 差异点 | 控制条件 |
|-----|-------|---------|
| SiteSwitcher | 登录用户有下拉菜单，public 角色只显示静态域名 | `user.loggedIn` + `user.role` |
| SegmentMenu | 可创建/编辑 Segments 的功能隐藏 | `canSeeSaveAsSegmentAction(user)` |
| FiltersBar | "Save as segment" 菜单项隐藏 | `showingSaveAsSegment` 条件 |
| DashboardOptionsMenu | 部分选项可能隐藏 | 基于 `site_role` |
| TopBar | 嵌入模式下禁用 sticky 效果 | `!site.embedded` |

### 8.3 嵌入模式下的 UI 差异

通过 `site.embedded` 和 `site.background` 控制：

1. **无 sticky 导航** - 滚动时导航栏不固定
2. **可自定义背景** - 通过 `background` 参数设置
3. **主题控制** - 通过 `theme` 参数选择亮/暗主题
4. **隐藏页脚** - `hide_footer?: true`

---

## 九、关键代码位置索引

### 9.1 后端关键文件

| 文件 | 功能 | 关键行 |
|-----|-----|-------|
| `lib/plausible_web/router.ex` | 路由配置，pipeline 定义 | L36-41 (`:shared_link`), L72-78 (`:internal_stats_api`) |
| `lib/plausible_web/controllers/stats_controller.ex` | 页面渲染控制器 | L49-115 (`stats/2`), L261-297 (`shared_link/2`), L442-532 (`render_shared_link/2`) |
| `lib/plausible_web/plugs/authorize_site_access.ex` | 核心权限检查插件 | L78-140 (`call/2`), L200-220 (`maybe_get_shared_link/2`) |
| `lib/plausible/site/shared_link.ex` | SharedLink 模型 | L1-58 (schema + 辅助函数) |
| `lib/plausible_web/controllers/api/stats_controller.ex` | Stats API 控制器 | L1331-1351 (`validate_required_filters_plug/2`) |

### 9.2 前端关键文件

| 文件 | 功能 | 关键行 |
|-----|-----|-------|
| `assets/js/dashboard.tsx` | 前端入口初始化 | L1-112 (完整初始化流程) |
| `assets/js/dashboard/user-context.tsx` | 用户角色定义 | L3-10 (`Role` enum) |
| `assets/js/dashboard/site-context.tsx` | 站点配置解析 | L3-27 (`parseSiteFromDataset`) |
| `assets/js/dashboard/api.ts` | API 层认证处理 | L9-107 (`SHARED_LINK_AUTH`, `getHeaders`, `setSharedLinkAuth`) |
| `assets/js/dashboard/filtering/segments.ts` | Segment 权限控制 | L8-21 (`ROLES_WITH_*`), L189-196 (`canSeeSaveAsSegmentAction`) |
| `assets/js/dashboard/router.tsx` | 路由路径区分 | L177-184 (`getRouterBasepath`) |
| `assets/js/dashboard/site-switcher.tsx` | 站点切换器 | L90-108 (登录状态检查) |

---

## 十、总结架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           访问入口层                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  私有 Dashboard                    公开报表                      嵌入报表   │
│  ──────────────                    ────────                      ────────   │
│  GET /mydomain.com                GET /share/mydomain.com      同上 +      │
│                                    ?auth=abc123                  ?embed=true │
│                                                                              │
│  ┌─────────────────┐              ┌─────────────────┐          ┌─────────┐ │
│  │ :browser        │              │ :shared_link    │          │  同上   │ │
│  │ + AuthPlug      │              │ (无 AuthPlug)   │          │         │ │
│  │ + AuthorizeSite │              │                 │          │         │ │
│  │   Access        │              │                 │          │         │ │
│  └────────┬────────┘              └────────┬────────┘          └────┬────┘ │
└───────────┼────────────────────────────────┼────────────────────────┼───────┘
            │                                │                        │
            ▼                                ▼                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           控制器层                                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  stats/2                        shared_link/2                               │
│  ───────                        ─────────────                               │
│  检查成员关系                   检查 SharedLink 存在性                        │
│  site_role = :admin/:editor    可选：密码验证                               │
│                :viewer/:owner  site_role = :public                          │
│                                 shared_link_auth = slug                       │
│                                 embedded = (embed==true)                      │
│                                                                              │
│  └───────────────────────┬──────────────────────────┘                       │
│                          ▼                                                     │
│              ┌─────────────────────┐                                          │
│              │ render("stats.html")│ ← 共用同一个模板！                        │
│              └──────────┬──────────┘                                          │
└─────────────────────────┼─────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           前端初始化层                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  dataset.currentUserRole  →  Role.public/Role.admin/etc.                    │
│  dataset.sharedLinkAuth   →  api.setSharedLinkAuth()                        │
│  dataset.embedded         →  site.embedded = true                            │
│  dataset.limitedToSegment →  强制 Segment 过滤                               │
│                                                                              │
│  └───────────────────────────────────┬───────────────────────────────────────┘
│                                      ▼
│                    ┌──────────────────────────────┐
│                    │ 路由 basepath 区分            │
│                    │ 私有: /domain                 │
│                    │ 公开: /share/domain          │
│                    └──────────────┬───────────────┘
└───────────────────────────────────┼───────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           API 调用层                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  所有请求携带:                                                                │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ Query:  ?auth=slug (如果有 shared_link_auth)                          │  │
│  │ Header: X-Shared-Link-Auth: slug (如果有 shared_link_auth)           │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  GET /api/stats/:domain/sources                                              │
│  GET /api/stats/:domain/pages                                                │
│  POST /api/stats/:domain/query                                               │
│  ...                                                                         │
│                                                                              │
│  └──────────────────────────────┬───────────────────────────────────────────┘
│                                 ▼
┌─────────────────────────────────┼───────────────────────────────────────────┐
│         :internal_stats_api     │                                           │
│         pipeline                │                                           │
│  ┌──────────────────────────────┴──────────────────────────────────────┐   │
│  │ AuthorizeSiteAccess 插件 (核心分叉点)                                 │   │
│  │ ─────────────────────────────────────                                 │   │
│  │                                                                       │   │
│  │ 有成员关系? ──Yes──► role = 成员角色                                  │   │
│  │       │                                                               │   │
│  │       No                                                            │   │
│  │       │                                                               │   │
│  │       ▼                                                               │   │
│  │ 有 ?auth=slug? ──Yes──► 查找 SharedLink ──存在──► role = :public    │   │
│  │       │                            │                                 │   │
│  │       No                          不存在                             │   │
│  │       │                            │                                 │   │
│  │       ▼                            ▼                                 │   │
│  │ site.public=true? ──Yes──► role=:public                        404  │   │
│  │       │                                                               │   │
│  │       No                                                             │   │
│  │       │                                                               │   │
│  │       ▼                                                               │   │
│  │     404                                                              │   │
│  └───────────────────────────────────────────────────────────────────────┘   │
│                                      │                                          │
│                                      ▼                                          │
│                    ┌─────────────────────────────┐                              │
│                    │ StatsController 处理函数     │                              │
│                    │ (私有/公开完全相同)           │                              │
│                    └──────────────┬──────────────┘                              │
└───────────────────────────────────┼───────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           核心查询层 (完全复用)                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Plausible.Stats.breakdown/4    ← 所有 breakdown 类报表                     │
│  Plausible.Stats.timeseries/3   ← 时间序列图表                              │
│  Plausible.Stats.query/2        ← 通用查询接口                              │
│  Plausible.Stats.aggregate/2    ← 聚合指标                                  │
│                                                                              │
│  唯一的限制: 如果 SharedLink 有 segment_id                                   │
│              → validate_required_filters_plug 强制验证 filter                │
│              → 查询时 Segment 展开为实际的过滤条件                            │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 十一、关键设计要点

### 11.1 权限分叉点总结

| 层级 | 私有 Dashboard | 公开/嵌入报表 | 分叉位置 |
|-----|--------------|--------------|---------|
| **路由 URL** | `/:domain` | `/share/:domain?auth=slug` | `router.ex:489-490` |
| **Pipeline** | `:browser` + `AuthorizeSiteAccess` | `:shared_link` (无 AuthPlug) | `router.ex:36-41` |
| **控制器** | `stats/2` | `shared_link/2` | `stats_controller.ex:261-297` |
| **角色** | `:admin/:editor/:viewer/:owner` | `:public` | `authorize_site_access.ex:86-102` |
| **API 认证** | Session Cookie | `?auth=slug` + `X-Shared-Link-Auth` | `api.ts:9-107` |
| **API 授权** | 成员关系验证 | SharedLink 存在性验证 | `authorize_site_access.ex:200-220` |

### 11.2 组件复用策略

1. **模板完全复用** - `stats.html.heex` 同时服务私有和公开场景
2. **API 端点完全复用** - `/api/stats/*` 端点不区分访问类型
3. **查询逻辑完全复用** - `Plausible.Stats.*` 函数无差别
4. **前端组件完全复用** - 所有可视化组件相同
5. **通过 Context/Props 控制差异** - `site_role`, `site.shared`, `site.embedded`

### 11.3 数据限制机制

**没有基于角色的行级过滤！** 数据限制仅通过以下方式：

1. **Segment 限制** - SharedLink 可关联一个 Segment，强制应用其过滤条件
2. **访问权限控制** - 只控制"能不能看"，不控制"能看多少"
3. **公开站点** - `site.public=true` 时任何人都可查看完整数据

---

## 十二、密码保护流程

对于带密码保护的 SharedLink：

```
1. 用户访问 /share/mydomain.com?auth=slug
   └─> shared_link/2 检测 password_hash 存在

2. 渲染 password 表单
   └─> render_password_protected_shared_link/2

3. 用户提交密码
   └─> POST /share/:slug/authenticate
       └─> authenticate_shared_link/2
           └─> 密码匹配
               └─> 设置 cookie: shared-link-{slug} = signed_token
               └─> 重定向回 /share/mydomain.com?auth=slug

4. 后续访问
   └─> maybe_get_shared_link/2 检测 cookie
       └─> validate_shared_link_password/2 验证 cookie
           └─> 成功 → 允许访问
```

**密码验证逻辑 (`stats_controller.ex:299-307`)：**
```elixir
def validate_shared_link_password(conn, shared_link) do
  with {:ok, token} <- Map.fetch(conn.req_cookies, shared_link_cookie_name(shared_link.slug)),
       {:ok, %{slug: token_slug}} <- Plausible.Auth.Token.verify_shared_link(token),
       true <- token_slug == shared_link.slug do
    {:ok, shared_link}
  else
    _e -> {:error, :unauthorized}
  end
end
```
