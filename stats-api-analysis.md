# API Token、Stats API 与界面权限的查询能力复用分析

## 1. 整体架构概览

### 1.1 查询能力复用架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           入口层 (Entry Layer)                            │
├─────────────────────────────────────┬─────────────────────────────────────┤
│         API Token 入口              │         界面权限入口                 │
│  (AuthorizePublicAPI)               │  (AuthorizeSiteAccess)              │
├─────────────────────────────────────┼─────────────────────────────────────┤
│  • Bearer Token 验证                │  • 用户会话认证                      │
│  • Scopes 范围检查                   │  • 角色权限检查                      │
│  • 速率限制检查                      │  • 共享链接支持                      │
│  • StatsAPI 功能检查                 │  • 团队锁定检查                      │
└───────────────────┬─────────────────┴───────────────────┬─────────────────┘
                    │                                     │
                    ▼                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        控制器层 (Controller Layer)                         │
├─────────────────────────────────────┬─────────────────────────────────────┤
│   Api.StatsController               │   StatsController                    │
│   (API 端点处理)                      │   (Web 界面处理)                     │
│   ExternalStatsController           │                                      │
│   (Public API v1/v2)                 │                                      │
├─────────────────────────────────────┼─────────────────────────────────────┤
│  • 解析 API 参数                     │  • 解析前端状态                      │
│  • Query.from/3 构建查询            │  • 直接调用 API 控制器方法           │
│  • 调用 Stats.breakdown/4 等        │  • (如 csv_export 重用 sources 等)  │
└───────────────────┬─────────────────┴───────────────────┬─────────────────┘
                    │                                     │
                    ▼                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        核心查询层 (Core Query Layer)                       │
├─────────────────────────────────────────────────────────────────────────┤
│                           Plausible.Stats                                 │
├─────────────────────────────────────────────────────────────────────────┤
│  • query/2          → QueryRunner.run/2                                 │
│  • breakdown/4      → Breakdown.breakdown/4                             │
│  • aggregate/3      → Aggregate.aggregate/3                              │
│  • timeseries/3     → Timeseries.timeseries/3                            │
│  • current_visitors → CurrentVisitors.current_visitors/2                 │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 关键数据流

无论是 API Token 还是界面权限，最终的数据流向都是：

1. **授权层** → 验证身份和权限，设置 `conn.assigns[:site]`
2. **控制器层** → 构建 `Query` 结构体
3. **核心层** → 执行实际的数据库查询

---

## 2. 授权差异分析

### 2.1 API Token 授权机制 (AuthorizePublicAPI)

**文件位置**: `lib/plausible_web/plugs/authorize_public_api.ex`

#### 2.1.1 授权流程

```elixir
# authorize_public_api.ex:50-65
def call(conn, _opts) do
  requested_scope = Map.fetch!(conn.assigns, :api_scope)
  context = conn.assigns[:api_context]

  with {:ok, token} <- get_bearer_token(conn),
       {:ok, api_key, limit_key, hourly_limit} <- find_api_key(conn, token, context),
       :ok <- check_api_key_rate_limit(limit_key, hourly_limit),
       :ok <- check_api_key_burst_limit(limit_key),
       {:ok, conn} <- verify_by_scope(conn, api_key, requested_scope) do
    conn
    |> assign(:current_user, api_key.user)
    |> assign(:current_team, api_key.team)
  else
    error -> send_error(conn, requested_scope, error)
  end
end
```

#### 2.1.2 关键授权检查点

| 检查项 | 实现位置 | 说明 |
|--------|----------|------|
| Bearer Token 提取 | L175-185 | 从 Authorization 头提取 `Bearer <token>` |
| API Key 查找 | L69-115 | 支持团队和遗留两种模式 |
| 速率限制检查 | L187-214 | 每小时限制 + 突发请求限制 |
| Scopes 范围检查 | L156-173 | 前缀匹配，支持通配符 |
| 站点访问验证 | L117-131 | 检查成员资格、订阅功能 |
| 团队访问验证 | L133-154 | 检查团队成员资格 |

#### 2.1.3 Scopes 机制

```elixir
# 隐式授权的 scopes - 所有有效 API key 都拥有
@implicit_scopes ["stats:read:*", "sites:read:*"]

# 范围检查使用前缀匹配
# 例如: API key 有 "stats:*"，可以访问 "stats:read:*"
defp check_scope(api_key, required_scope) do
  found? =
    Enum.any?(api_key.scopes, fn scope ->
      scope = String.trim_trailing(scope, "*")
      String.starts_with?(required_scope, scope)
    end)
end
```

#### 2.1.4 速率限制策略

```elixir
# 检查每小时请求限制
defp check_api_key_rate_limit(limit_key, hourly_limit) do
  case RateLimit.check_rate(limit_key, to_timeout(hour: 1), hourly_limit) do
    {:allow, _} -> :ok
    {:deny, _} -> {:error, :rate_limit, "..."}
  end
end

# 检查突发请求限制
defp check_api_key_burst_limit(limit_key) do
  burst_period_seconds = Auth.ApiKey.burst_period_seconds()
  burst_request_limit = Auth.ApiKey.burst_request_limit()
  # ...
end
```

#### 2.1.5 站点访问验证（关键：订阅功能检查）

```elixir
# L257-290
defp verify_site_access(opts) do
  site = Keyword.fetch!(opts, :site)
  api_key = Keyword.fetch!(opts, :api_key)
  feature = Keyword.fetch!(opts, :feature)  # 如 StatsAPI

  team = Repo.preload(site, :team).team

  is_member? = Plausible.Teams.Memberships.site_member?(site, api_key.user)
  is_super_admin? = Auth.is_super_admin?(api_key.user_id)

  cond do
    # 1. 合并视图检查
    Plausible.Sites.consolidated?(site) && !allow_consolidated_views ->
      {:error, :unavailable_for_consolidated_view}

    # 2. 超级管理员绕过
    is_super_admin? -> :ok

    # 3. API key 所属团队检查
    api_key.team_id && api_key.team_id != site.team_id ->
      {:error, :invalid_api_key}

    # 4. 团队锁定检查
    Teams.locked?(team) -> {:error, :site_locked}

    # 5. ⚠️ 关键：StatsAPI 功能可用性检查
    # 这是 API Token 特有的检查，需要高级订阅
    feature.check_availability(team) !== :ok ->
      {:error, :upgrade_required}

    # 6. 成员资格检查
    is_member? -> :ok

    true -> {:error, :invalid_api_key}
  end
end
```

### 2.2 界面权限机制 (AuthorizeSiteAccess + StatsController)

**文件位置**: 
- `lib/plausible_web/plugs/authorize_site_access.ex`
- `lib/plausible_web/controllers/stats_controller.ex`

#### 2.2.1 授权流程

```elixir
# authorize_site_access.ex:78-140
def call(conn, {allowed_roles, site_param}) do
  current_user = conn.assigns[:current_user]

  with {:ok, domain} <- get_domain(conn, site_param),
       {:ok, %{site: site, role: membership_role, member_type: member_type}} <-
         get_site_with_role(conn, current_user, domain),
       :ok <- ensure_consolidated_view_access(conn, site),
       {:ok, shared_link} <- maybe_get_shared_link(conn, site) do
    # 确定最终角色
    role =
      cond do
        membership_role -> membership_role
        Plausible.Auth.is_super_admin?(current_user) -> :super_admin
        site.public -> :public
        shared_link -> :public
        true -> nil
      end

    if role in allowed_roles do
      # 设置 conn.assigns
      merge_assigns(conn, site: site, site_role: role, shared_link: shared_link)
    else
      error_not_found(conn)
    end
  end
end
```

#### 2.2.2 角色系统

```elixir
# 支持的所有角色
@all_roles [:public, :viewer, :admin, :editor, :super_admin, :owner, :billing]

# 角色优先级（从高到低）:
# 1. :super_admin - 超级管理员（绕过所有检查）
# 2. :owner - 团队所有者
# 3. :admin - 管理员
# 4. :editor - 编辑者
# 5. :viewer - 查看者
# 6. :billing - 账单管理员
# 7. :public - 公开访问（通过共享链接或公开站点）
```

#### 2.2.3 ⚠️ 界面权限的订阅检查（修正前偏差）

**之前的错误结论**: 界面权限"隐式通过会话处理"订阅检查

**正确的事实**:

1. **AuthorizeSiteAccess 插件**：**不**检查 `Feature.StatsAPI` 功能可用性
2. **StatsController**：在 `stats/2` 动作中检查 `Teams.locked?(site.team)`

```elixir
# stats_controller.ex:51-114
def stats(%{assigns: %{site: site}} = conn, _params) do
  site = Plausible.Repo.preload(site, :owners)
  site_role = conn.assigns[:site_role]
  current_user = conn.assigns[:current_user]
  stats_start_date = Plausible.Sites.stats_start_date(site)
  
  # ⚠️ 关键：只检查团队是否被锁定，不检查 StatsAPI 功能
  can_see_stats? = not Teams.locked?(site.team) or site_role == :super_admin
  
  # ...
  
  cond do
    # ...
    Teams.locked?(site.team) ->
      site = Plausible.Repo.preload(site, :owners)
      render(conn, "site_locked.html", site: site, dogfood_page_path: dogfood_page_path)
  end
end
```

#### 2.2.4 共享链接支持

```elixir
# L200-220
defp maybe_get_shared_link(conn, site) do
  slug = conn.path_params["slug"] || conn.params["auth"]

  if valid_path_fragment?(slug) do
    with %Plausible.Site.SharedLink{} = shared_link <-
           Repo.get_by(Plausible.Site.SharedLink, slug: slug, site_id: site.id),
         # 密码保护检查
         {%{password_protected?: true}, shared_link} <-
           {%{password_protected?: Plausible.Site.SharedLink.password_protected?(shared_link)},
            shared_link},
         {:ok, shared_link} <-
           PlausibleWeb.StatsController.validate_shared_link_password(conn, shared_link) do
      {:ok, shared_link}
    else
      # 无密码保护的共享链接直接通过
      {%{password_protected?: false}, shared_link} -> {:ok, shared_link}
      {:error, :unauthorized} -> error_not_found(conn)
      nil -> error_not_found(conn)
    end
  else
    {:ok, nil}
  end
end
```

### 2.3 授权机制对比表（修正后）

| 特性 | API Token 授权 | 界面权限授权 |
|------|---------------|-------------|
| **认证方式** | Bearer Token (HTTP Header) | 用户 Session (Cookie) |
| **权限检查** | Scopes 范围匹配 | 角色列表匹配 |
| **速率限制** | 有（每小时 + 突发） | 无 |
| **共享链接** | 不支持 | 支持（可密码保护） |
| **订阅检查 - 团队锁定** | ✅ 检查 `Teams.locked?` | ✅ 检查 `Teams.locked?` |
| **订阅检查 - StatsAPI 功能** | ✅ 检查 `Feature.StatsAPI` | ❌ **不检查** |
| **团队切换** | 基于 API key 绑定 | 自动切换到站点所属团队 |
| **错误响应** | JSON 格式 | 404 页面或 JSON（根据格式） |
| **超级管理员** | 绕过所有检查 | 绕过所有检查 |

**关键差异说明**:
- **API Token**: 需要 `StatsAPI` 功能可用（通常需要高级订阅如 Business/Enterprise）
- **界面权限**: 只需要团队没有被锁定（有活跃订阅即可，包括基础版）

---

## 3. Stats 各接口权限声明映射

### 3.1 路由 Pipeline 概览

根据 `lib/plausible_web/router.ex`，Stats API 有三个主要的访问入口：

| Pipeline | 授权机制 | 适用场景 |
|----------|----------|----------|
| `:internal_stats_api` | `AuthorizeSiteAccess` (默认 `:all_roles`) | 前端界面调用 |
| `:public_api` + `AuthorizePublicAPI` | Bearer Token + Scopes | 公共 API v1/v2 |
| `:docs_stats_api` | `AuthorizeSiteAccess` (限制 `[:admin, :editor, :super_admin, :owner]`) | API 文档测试 |

### 3.2 公共 API v1/v2 权限映射 (API Token)

#### 3.2.1 API v1 端点 (`/api/v1/stats`)

**Scope**: `stats:read:*`

| 端点 | HTTP 方法 | 控制器动作 | 所需 Scope | 功能检查 |
|------|-----------|-----------|------------|----------|
| `/api/v1/stats/realtime/visitors` | GET | `ExternalStatsController.realtime_visitors/2` | `stats:read:*` | StatsAPI |
| `/api/v1/stats/aggregate` | GET | `ExternalStatsController.aggregate/2` | `stats:read:*` | StatsAPI |
| `/api/v1/stats/breakdown` | GET | `ExternalStatsController.breakdown/2` | `stats:read:*` | StatsAPI |
| `/api/v1/stats/timeseries` | GET | `ExternalStatsController.timeseries/2` | `stats:read:*` | StatsAPI |

#### 3.2.2 API v2 端点 (`/api/v2`)

**Scope**: `stats:read:*`

| 端点 | HTTP 方法 | 控制器动作 | 所需 Scope | 功能检查 |
|------|-----------|-----------|------------|----------|
| `/api/v2/query` | POST | `ExternalQueryApiController.query/2` | `stats:read:*` | StatsAPI |

#### 3.2.3 Sites API 端点 (`/api/v1/sites`)

| 端点组 | Scope | 功能检查 |
|--------|-------|----------|
| 读取类 (index, teams_index, goals_index 等) | `sites:read:*` | StatsAPI |
| 管理类 (create_site, update_site, delete_site 等) | `sites:provision:*` | StatsAPI + SitesAPI |

### 3.3 内部 Stats API 权限映射 (界面权限)

**Pipeline**: `internal_stats_api`
**授权插件**: `AuthorizeSiteAccess` (默认 `:all_roles`)

#### 3.3.1 核心查询端点

| 端点 | HTTP 方法 | 控制器动作 | 允许角色 | 团队锁定检查 | StatsAPI 检查 |
|------|-----------|-----------|----------|-------------|---------------|
| `/api/stats/:domain/query` | POST | `Api.StatsController.query/2` | 所有角色 | ❌ **不检查** | ❌ |
| `/api/stats/:domain/current-visitors` | GET | `Api.StatsController.current_visitors/2` | 所有角色 | ❌ **不检查** | ❌ |

#### 3.3.2 Breakdown 类端点

| 端点 | HTTP 方法 | 控制器动作 | 对应维度 |
|------|-----------|-----------|----------|
| `/api/stats/:domain/sources` | GET | `StatsController.sources/2` | `visit:source` |
| `/api/stats/:domain/channels` | GET | `StatsController.channels/2` | `visit:channel` |
| `/api/stats/:domain/pages` | GET | `StatsController.pages/2` | `event:page` |
| `/api/stats/:domain/entry-pages` | GET | `StatsController.entry_pages/2` | `visit:entry_page` |
| `/api/stats/:domain/exit-pages` | GET | `StatsController.exit_pages/2` | `visit:exit_page` |
| `/api/stats/:domain/countries` | GET | `StatsController.countries/2` | `visit:country` |
| `/api/stats/:domain/regions` | GET | `StatsController.regions/2` | `visit:region` |
| `/api/stats/:domain/cities` | GET | `StatsController.cities/2` | `visit:city` |
| `/api/stats/:domain/browsers` | GET | `StatsController.browsers/2` | `visit:browser` |
| `/api/stats/:domain/browser-versions` | GET | `StatsController.browser_versions/2` | `visit:browser_version` |
| `/api/stats/:domain/operating-systems` | GET | `StatsController.operating_systems/2` | `visit:os` |
| `/api/stats/:domain/operating-system-versions` | GET | `StatsController.operating_system_versions/2` | `visit:os_version` |
| `/api/stats/:domain/screen-sizes` | GET | `StatsController.screen_sizes/2` | `visit:device` |
| `/api/stats/:domain/conversions` | GET | `StatsController.conversions/2` | `event:goal` |

#### 3.3.3 UTM 类端点

| 端点 | HTTP 方法 | 控制器动作 | 对应维度 |
|------|-----------|-----------|----------|
| `/api/stats/:domain/utm_mediums` | GET | `StatsController.utm_mediums/2` | `visit:utm_medium` |
| `/api/stats/:domain/utm_sources` | GET | `StatsController.utm_sources/2` | `visit:utm_source` |
| `/api/stats/:domain/utm_campaigns` | GET | `StatsController.utm_campaigns/2` | `visit:utm_campaign` |
| `/api/stats/:domain/utm_contents` | GET | `StatsController.utm_contents/2` | `visit:utm_content` |
| `/api/stats/:domain/utm_terms` | GET | `StatsController.utm_terms/2` | `visit:utm_term` |

#### 3.3.4 自定义属性端点

| 端点 | HTTP 方法 | 控制器动作 | 说明 |
|------|-----------|-----------|------|
| `/api/stats/:domain/custom-prop-values/:prop_key` | GET | `StatsController.custom_prop_values/2` | 需要检查 Props 功能可用性 |
| `/api/stats/:domain/suggestions/custom-prop-values/:prop_key` | GET | `StatsController.custom_prop_value_filter_suggestions/2` | 属性值建议 |

#### 3.3.5 企业版特有端点 (EE Only)

| 端点 | HTTP 方法 | 控制器动作 | 额外权限 |
|------|-----------|-----------|----------|
| `/api/stats/:domain/funnels/:id` | GET | `StatsController.funnel/2` | 需要 Funnels 功能 |
| `/api/stats/:domain/exploration/*` | POST | `StatsController.exploration_*` | 需要 Super Admin |

### 3.4 Web 界面端点权限映射

| 端点 | HTTP 方法 | 控制器动作 | 允许角色 | 说明 |
|------|-----------|-----------|----------|------|
| `/:domain/*path` | GET | `StatsController.stats/2` | 所有角色 | 主统计面板 |
| `/:domain/export` | GET | `StatsController.csv_export/2` | 所有角色 | CSV 导出（重用 API 控制器方法） |
| `/share/:domain/*path` | GET | `StatsController.shared_link/2` | 公开 | 共享链接访问 |

### 3.5 权限声明总表

#### 3.5.1 API Token 方式

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        API Token 权限声明模型                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  层级1: Bearer Token 验证                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  Authorization: Bearer <api_key>                                    │  │
│  │  → 验证 key_hash 是否存在                                            │  │
│  │  → 获取关联的 user 和 team                                           │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
│                                    ▼                                        │
│  层级2: 速率限制检查                                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  • 每小时请求限制 (基于 team 或 user)                                 │  │
│  │  • 突发请求限制 (防止瞬间流量冲击)                                     │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
│                                    ▼                                        │
│  层级3: Scopes 范围匹配                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  路由 assigns: %{api_scope: "stats:read:*"}                         │  │
│  │  vs                                                                    │  │
│  │  API Key scopes: ["stats:read:*", "sites:provision:*"]              │  │
│  │                                                                       │  │
│  │  匹配规则: 前缀匹配 (支持通配符 *)                                      │  │
│  │  隐式 scopes: ["stats:read:*", "sites:read:*"] (所有 API key 都有)   │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
│                                    ▼                                        │
│  层级4: 站点/团队访问验证                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  1. 成员资格检查: 用户是否是站点/团队成员                               │  │
│  │  2. 团队锁定检查: Teams.locked?(team)                                 │  │
│  │  3. 功能检查: Feature.StatsAPI.check_availability(team)              │  │
│  │     → 这是 API Token 特有的严格检查                                    │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 3.5.2 界面权限方式

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        界面权限声明模型                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  层级1: Session 认证                                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  Cookie 中的 session token                                            │  │
│  │  → AuthPlug 填充 conn.assigns[:current_user, :current_team, ...]    │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
│                                    ▼                                        │
│  层级2: 角色检查 (AuthorizeSiteAccess)                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  确定用户的 site_role:                                                 │  │
│  │    1. 成员角色 (:owner, :admin, :editor, :viewer, :billing)         │  │
│  │    2. :super_admin (超级管理员)                                        │  │
│  │    3. :public (公开站点或共享链接)                                     │  │
│  │                                                                       │  │
│  │  检查: role in allowed_roles                                          │  │
│  │  - internal_stats_api: :all_roles (所有角色都允许)                    │  │
│  │  - docs_stats_api: [:admin, :editor, :super_admin, :owner]          │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
│                                    ▼                                        │
│  层级3: 访问验证                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  1. 合并视图检查: consolidated_view_available?                        │  │
│  │  2. 共享链接检查: 密码保护验证 (如有)                                   │  │
│  │  3. 团队锁定检查: Teams.locked?(team) (在 StatsController 中)          │  │
│  │                                                                       │  │
│  │  ⚠️ 注意: 没有 StatsAPI 功能检查!                                      │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. 参数流转分析

### 4.1 API 层参数流转 (Api.StatsController)

**文件位置**: `lib/plausible_web/controllers/api/stats_controller.ex`

#### 4.1.1 统一的查询构建模式

```elixir
# 以 sources 端点为例 (L59-97)
def sources(conn, params) do
  site = conn.assigns[:site]  # 来自授权层
  params = Map.put(params, "property", "visit:source")
  
  # 核心：构建 Query 结构体
  query = Query.from(site, params, debug_metadata: debug_metadata(conn))
  
  pagination = parse_pagination(params)
  
  # 核心：调用统一的查询函数
  %{results: results, meta: meta} = Stats.breakdown(site, query, metrics, pagination)
  
  # 结果转换和返回
  json(conn, %{
    results: results,
    meta: Stats.Breakdown.formatted_date_ranges(query),
    skip_imported_reason: meta[:imports_skip_reason]
  })
end
```

#### 4.1.2 Query.from/3 调用链

```
Query.from(site, params, opts)
    │
    ▼
Legacy.QueryBuilder.from(site, params, debug_metadata, now)
    │
    ▼
# 构建完整的 Query 结构体
%Plausible.Stats.Query{
  utc_time_range: ...,        # UTC 时间范围
  input_date_range: ...,       # 原始输入日期范围
  filters: [...],              # 过滤条件列表
  dimensions: [...],           # 维度列表
  metrics: [...],              # 指标列表
  include: %{...},             # 包含选项（导入数据、比较等）
  timezone: ...,               # 时区
  sample_threshold: 20_000_000, # 采样阈值
  ...
}
```

#### 4.1.3 新的 query 端点 (L40-57)

```elixir
# 使用新的解析器
def query(conn, params) do
  site = conn.assigns.site
  now = conn.private[:now]

  with {:ok, %ParsedQueryParams{} = params} <- Dashboard.QueryParser.parse(params, now: now),
       {:ok, %Query{} = query} <- QueryBuilder.build(site, params, debug_metadata(conn)) do
    json(conn, Plausible.Stats.query(site, query))
  else
    {:error, %QueryError{message: message}} -> bad_request(conn, message)
  end
end
```

### 4.2 界面层参数流转 (StatsController + 前端)

#### 4.2.1 Web 控制器重用 API 控制器

```elixir
# lib/plausible_web/controllers/stats_controller.ex:134-196
def csv_export(conn, params) do
  site = Plausible.Repo.preload(conn.assigns.site, :owners)
  query = Query.from(site, params, debug_metadata: debug_metadata(conn))

  # 直接调用 API 控制器的方法，实现代码复用
  csvs = %{
    ~c"visitors.csv" => fn -> main_graph_csv(site, query) end,
    ~c"sources.csv" => fn -> Api.StatsController.sources(conn, params) end,
    ~c"channels.csv" => fn -> Api.StatsController.channels(conn, params) end,
    ~c"pages.csv" => fn -> Api.StatsController.pages(conn, limited_params) end,
    # ... 更多复用
  }
end
```

#### 4.2.2 前端参数构建 (stats-query.ts)

**文件位置**: `assets/js/dashboard/stats-query.ts`

```typescript
export function createStatsQuery(
  dashboardState: DashboardState,
  reportParams: ReportParams
): StatsQuery {
  return {
    date_range: createDateRange(dashboardState),
    relative_date: dashboardState.date ? formatISO(dashboardState.date) : null,
    dimensions: reportParams.dimensions || [],
    metrics: reportParams.metrics,
    filters: remapToApiFilters(dashboardState.filters),  // 过滤器转换
    include: {
      imports: dashboardState.with_imported,
      imports_meta: reportParams.include?.imports_meta || false,
      time_labels: reportParams.include?.time_labels || false,
      compare: createIncludeCompare(dashboardState),
      compare_match_day_of_week: dashboardState.match_day_of_week,
      // ...
    }
  }
}
```

### 4.3 参数流转统一模式

```
┌────────────────────────────────────────────────────────────────────────────┐
│                        参数流转统一模式                                        │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────┐                                                           │
│  │  授权层完成   │  → 设置 conn.assigns[:site]                              │
│  └──────────────┘                                                           │
│         │                                                                    │
│         ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                         参数来源                                        │  │
│  ├───────────────────────┬──────────────────────────────────────────────┤  │
│  │    API Token 方式      │              界面权限方式                      │  │
│  ├───────────────────────┼──────────────────────────────────────────────┤  │
│  │ • URL 查询参数         │ • DashboardState (React 状态)                │  │
│  │ • Request Body         │ • ReportParams (报告配置)                    │  │
│  │ • 路径参数 (site_id)   │ • remapToApiFilters 转换                    │  │
│  └───────────────────────┴──────────────────────────────────────────────┘  │
│         │                                                                    │
│         ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                    Query 结构体构建                                     │  │
│  ├──────────────────────────────────────────────────────────────────────┤  │
│  │  方式1: Query.from(site, params, opts)  ← 遗留 API v1                │  │
│  │         → Legacy.QueryBuilder.from()                                   │  │
│  │                                                                         │  │
│  │  方式2: Query.parse_and_build(site, params, opts)  ← 新 API          │  │
│  │         → ApiQueryParser.parse()                                       │  │
│  │         → QueryBuilder.build()                                         │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│         │                                                                    │
│         ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                    核心查询函数调用                                     │  │
│  ├──────────────────────────────────────────────────────────────────────┤  │
│  │  Stats.breakdown(site, query, metrics, pagination)                   │  │
│  │  Stats.aggregate(site, query, metrics)                                │  │
│  │  Stats.timeseries(site, query, metrics)                               │  │
│  │  Stats.query(site, query)                                             │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### 4.4 Query 结构体核心字段

**文件位置**: `lib/plausible/stats/query.ex:4-34`

```elixir
defstruct utc_time_range: nil,           # UTC 时间范围 (实际查询使用)
          comparison_utc_time_range: nil, # 比较期时间范围
          interval: nil,                  # 时间间隔 (day, hour, month 等)
          input_date_range: nil,          # 原始输入日期范围
          dimensions: [],                  # 维度列表 (如 ["visit:source"])
          filters: [],                     # 过滤条件列表
          sample_threshold: 20_000_000,   # 采样阈值
          imports_exist: false,            # 是否存在导入数据
          imports_in_range: [],            # 范围内的导入数据
          include_imported: false,         # 是否包含导入数据
          skip_imported_reason: nil,       # 跳过导入数据的原因
          now: nil,                        # 当前时间
          metrics: [],                     # 指标列表
          order_by: nil,                   # 排序方式
          timezone: nil,                   # 时区
          legacy_breakdown: false,         # 是否使用遗留 breakdown
          include: Plausible.Stats.ApiQueryParser.default_include(),
          # include 包含: imports, imports_meta, time_labels, compare 等
          debug_metadata: %{},             # 调试元数据
          pagination: nil,                 # 分页信息
          revenue_currencies: %{},         # 收入指标货币信息
          revenue_warning: nil,            # 收入警告
          site_id: nil,                    # 站点 ID
          consolidated_site_ids: nil,      # 合并视图站点 ID 列表
          site_native_stats_start_at: nil, # 站点原生统计开始时间
          time_on_page_data: %{},          # 页面停留时间数据
          sql_join_type: :left,            # SQL 连接类型
          smear_session_metrics: false     # 是否涂抹会话指标
```

---

## 5. 两种权限模型在查询复用上的边界与限制

### 5.1 复用边界总览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    查询能力复用边界与限制                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                      完全复用的层 (Shared Layers)                     │  │
│  ├─────────────────────────────────────────────────────────────────────┤  │
│  │                                                                       │  │
│  │  核心查询层 (Plausible.Stats)                                          │  │
│  │  ┌─────────────────────────────────────────────────────────────┐    │  │
│  │  │ Stats.breakdown/4, Stats.aggregate/3, Stats.timeseries/3   │    │  │
│  │  │ Stats.query/2, Stats.current_visitors/2                      │    │  │
│  │  └─────────────────────────────────────────────────────────────┘    │  │
│  │                              ▲                                         │  │
│  │                              │                                         │  │
│  │  Query 结构体层                                               │  │
│  │  ┌─────────────────────────────────────────────────────────────┐    │  │
│  │  │ Query.from/3, Query.parse_and_build/3                        │    │  │
│  │  │ Query 结构体 (filters, dimensions, metrics, 等)               │    │  │
│  │  └─────────────────────────────────────────────────────────────┘    │  │
│  │                              ▲                                         │  │
│  │                              │                                         │  │
│  │  参数解析层 (ApiQueryParser, Legacy.QueryBuilder)                    │  │
│  │  ┌─────────────────────────────────────────────────────────────┐    │  │
│  │  │ 过滤器解析、日期范围解析、维度/指标验证                         │    │  │
│  │  └─────────────────────────────────────────────────────────────┘    │  │
│  │                                                                       │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                    部分复用的层 (Partially Shared)                    │  │
│  ├─────────────────────────────────────────────────────────────────────┤  │
│  │                                                                       │  │
│  │  控制器层 (Controllers)                                                │  │
│  │  ┌──────────────────┬──────────────────────────────────────────┐   │  │
│  │  │ Api.StatsController │ StatsController                        │   │  │
│  │  │ (内部 API)         │ (Web 界面)                               │   │  │
│  │  ├──────────────────┼──────────────────────────────────────────┤   │  │
│  │  │ • 所有 stats 端点  │ • stats/2 (渲染面板)                     │   │  │
│  │  │ • 被 csv_export    │ • csv_export/2 (重用 API 控制器方法)    │   │  │
│  │  │   直接调用         │                                          │   │  │
│  │  └──────────────────┴──────────────────────────────────────────┘   │  │
│  │                                                                       │  │
│  │  ExternalStatsController / ExternalQueryApiController                │  │
│  │  ┌─────────────────────────────────────────────────────────────┐    │  │
│  │  │ 公共 API v1/v2 专用，不被界面复用                               │    │  │
│  │  │ 不同的参数格式、不同的错误处理                                    │    │  │
│  │  └─────────────────────────────────────────────────────────────┘    │  │
│  │                                                                       │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                    完全隔离的层 (Completely Isolated)                 │  │
│  ├─────────────────────────────────────────────────────────────────────┤  │
│  │                                                                       │  │
│  │  授权层 (Authorization Plugs)                                          │
│  │  ┌──────────────────────────┬──────────────────────────────────┐   │  │
│  │  │ AuthorizePublicAPI       │ AuthorizeSiteAccess              │   │  │
│  │  ├──────────────────────────┼──────────────────────────────────┤   │  │
│  │  │ • Bearer Token 验证       │ • Session 认证                    │   │  │
│  │  │ • Scopes 范围匹配         │ • 角色列表匹配                    │   │  │
│  │  │ • 速率限制检查            │ • 无速率限制                       │   │  │
│  │  │ • StatsAPI 功能检查       │ • 仅团队锁定检查                   │   │  │
│  │  │ • 详细 JSON 错误响应      │ • 404 页面或模糊 JSON              │   │  │
│  │  └──────────────────────────┴──────────────────────────────────┘   │  │
│  │                                                                       │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 核心复用能力

| 能力层级 | API Token | 界面权限 | 复用方式 |
|----------|-----------|----------|----------|
| **Query 结构体** | ✅ 使用 | ✅ 使用 | 完全相同的结构体定义 |
| **Query 构建** | ✅ `Query.from/3`, `Query.parse_and_build/3` | ✅ 相同函数 | 完全相同的构建逻辑 |
| **核心查询函数** | ✅ `Stats.breakdown/4` 等 | ✅ 相同函数 | 完全相同的实现 |
| **稳定性机制** | ✅ 采样、导入数据跳过 | ✅ 相同机制 | 完全相同的保护 |
| **参数解析** | ✅ `ApiQueryParser`, `Legacy.QueryBuilder` | ✅ 相同解析器 | 完全相同的解析逻辑 |

### 5.3 边界与限制

#### 5.3.1 授权边界（不可逾越）

| 限制项 | API Token | 界面权限 | 说明 |
|--------|-----------|----------|------|
| **StatsAPI 功能要求** | ✅ 必须 | ❌ 不需要 | API Token 需要高级订阅 |
| **速率限制** | ✅ 强制 | ❌ 无 | API 有严格的请求配额 |
| **认证方式** | Bearer Token | Session Cookie | 完全不同的认证机制 |
| **权限模型** | Scopes (字符串前缀匹配) | Roles (原子枚举值) | 不同的权限表达 |
| **共享链接** | ❌ 不支持 | ✅ 支持 | 界面特有的公开访问方式 |

#### 5.3.2 功能可用性边界

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        功能可用性边界                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  订阅功能依赖 (Feature Availability)                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                                                                       │  │
│  │  API Token 访问                                                        │
│  │  ┌──────────────────────────────────────────────────────────────┐   │  │
│  │  │ 需要: Feature.StatsAPI.check_availability(team) == :ok       │   │  │
│  │  │                                                                  │   │  │
│  │  │ 通常需要: Business 或 Enterprise 订阅                           │   │  │
│  │  │ 包含:                                                           │   │  │
│  │  │   • /api/v1/stats/* (aggregate, breakdown, timeseries)        │   │  │
│  │  │   • /api/v2/query                                               │   │  │
│  │  │   • /api/v1/sites/* (需要 StatsAPI + SitesAPI)                 │   │  │
│  │  └──────────────────────────────────────────────────────────────┘   │  │
│  │                              │                                         │  │
│  │                              ▼                                         │  │
│  │  界面访问                                                               │
│  │  ┌──────────────────────────────────────────────────────────────┐   │  │
│  │  │ 只需要: not Teams.locked?(team)                                │   │  │
│  │  │                                                                  │   │  │
│  │  │ 任何活跃订阅都可以:                                             │   │  │
│  │  │   • Starter, Business, Enterprise 都支持                        │   │  │
│  │  │   • 只要订阅没有过期 (团队没被锁定)                              │   │  │
│  │  │                                                                  │   │  │
│  │  │ 额外功能检查 (在控制器中):                                       │   │  │
│  │  │   • Funnels: 需要 Feature.Funnels (EE only)                   │   │  │
│  │  │   • Props: 需要 Feature.Props (Business+)                      │   │  │
│  │  │   • ConsolidatedView: 需要 Feature.ConsolidatedView            │   │  │
│  │  └──────────────────────────────────────────────────────────────┘   │  │
│  │                                                                       │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  关键差异:                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                                                                       │  │
│  │  场景: 团队有 Starter 订阅 (活跃，但不含 StatsAPI)                    │  │
│  │                                                                       │  │
│  │    API Token 访问 → ❌ 402 Payment Required (需要 StatsAPI)          │  │
│  │    界面访问     → ✅ 正常使用 (只需要团队没被锁定)                    │  │
│  │                                                                       │  │
│  │  这是设计上的边界，不是 Bug！                                          │  │
│  │  - API Token 设计为第三方集成，需要明确的功能授权                      │  │
│  │  - 界面是产品的核心体验，应该对所有付费用户开放                        │  │
│  │                                                                       │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 5.3.3 端点可访问性边界

| 端点组 | API Token (Public API) | 界面权限 (Internal API) | 说明 |
|--------|------------------------|-------------------------|------|
| `/api/v1/stats/*` | ✅ 可用 | ❌ 不存在 | 公共 API v1 专用端点 |
| `/api/v2/query` | ✅ 可用 | ❌ 不存在 | 公共 API v2 专用端点 |
| `/api/stats/:domain/query` | ❌ 不存在 | ✅ 可用 | 内部 API 专用端点 |
| `/api/stats/:domain/*` (breakdown) | ❌ 不存在 | ✅ 可用 | 内部 API 专用端点 |
| `/:domain/*` (Web 界面) | ❌ 不适用 | ✅ 可用 | 仅浏览器访问 |
| `/share/:domain/*` (共享链接) | ❌ 不支持 | ✅ 可用 | 界面特有功能 |

#### 5.3.4 错误处理边界

| 错误场景 | API Token 响应 | 界面权限响应 | 边界原因 |
|----------|---------------|-------------|----------|
| 无效凭据 | 401 `{"error": "Invalid API key..."}` | 404 页面 | 安全考虑：界面不暴露详细错误 |
| 缺少权限 | 401 `{"error": "Invalid API key..."}` | 404 页面 | 同上 |
| 速率限制 | 429 `{"error": "Too many API requests..."}` | N/A | API 特有保护 |
| 升级要求 | 402 `{"error": "The account that owns..."}` | 页面提示 | API 明确，界面友好 |
| 站点锁定 | 402 `{"error": "This Plausible site is locked..."}` | `site_locked.html` | 相同含义，不同表现 |
| 查询参数错误 | 400 `{"error": "..."}` | 400 或静默失败 | 界面需要更好的用户体验 |

### 5.4 复用限制的设计意图

| 限制项 | 设计意图 | 架构决策 |
|--------|----------|----------|
| **StatsAPI 功能检查** | API Token 作为付费功能 | 区分"使用界面"和"API 集成"两种使用模式 |
| **速率限制** | 保护系统免受脚本滥用 | API 更易被自动化工具调用 |
| **不同的端点路径** | 清晰的职责分离 | Public API  vs Internal API |
| **不同的错误格式** | 不同的消费方 | API 面向机器（JSON），界面向用户（HTML） |
| **共享链接支持** | 界面特有的公开访问 | API Token 已提供类似能力（生成受限 key） |

### 5.5 跨越边界的场景

#### 5.5.1 可以跨越边界的场景

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        可以跨越边界的场景                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. 核心查询逻辑的完全复用                                                   │
│     ┌─────────────────────────────────────────────────────────────────┐   │
│     │ Stats.breakdown(site, query, metrics, pagination)               │   │
│     │                                                                   │   │
│     │ 无论是 API Token 还是界面权限，只要：                              │   │
│     │   • 提供相同的 site 对象                                          │   │
│     │   • 构建相同的 Query 结构体                                       │   │
│     │   • 传入相同的 metrics 和 pagination                              │   │
│     │                                                                   │   │
│     │ 就会得到完全相同的查询结果！                                       │   │
│     └─────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  2. CSV 导出中的控制器方法复用                                               │
│     ┌─────────────────────────────────────────────────────────────────┐   │
│     │ StatsController.csv_export/2                                     │   │
│     │   │                                                               │   │
│     │   ├──► Api.StatsController.sources(conn, params)                 │   │
│     │   ├──► Api.StatsController.channels(conn, params)                │   │
│     │   ├──► Api.StatsController.pages(conn, params)                   │   │
│     │   └──► ... (所有 breakdown 方法)                                   │   │
│     │                                                                   │   │
│     │ 界面控制器直接调用 API 控制器的方法，实现代码级复用                  │   │
│     └─────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  3. Query 结构体的完全互操作性                                               │
│     ┌─────────────────────────────────────────────────────────────────┐   │
│     │ 用 API Token 的参数构建的 Query，在界面上下文中完全可用            │   │
│     │ 反之亦然                                                          │   │
│     │                                                                   │   │
│     │ Query 结构体不包含任何授权相关的信息                                │   │
│     │ 它是纯粹的"查询描述"数据结构                                       │   │
│     └─────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 5.5.2 无法跨越边界的场景

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        无法跨越边界的场景                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. 授权层的本质差异                                                         │
│     ┌─────────────────────────────────────────────────────────────────┐   │
│     │ 无法用 Bearer Token "模拟" Session 认证                           │   │
│     │ 无法用 Session Cookie "模拟" API Token 认证                        │   │
│     │                                                                   │   │
│     │ 原因：                                                             │   │
│     │   • 不同的认证插件 (AuthorizePublicAPI vs AuthorizeSiteAccess)   │   │
│     │   • 不同的权限模型 (Scopes vs Roles)                              │   │
│     │   • 不同的 conn.assigns 结构                                       │   │
│     │       - API: :current_user, :current_team, :site                 │   │
│     │       - 界面: :current_user, :current_team, :site, :site_role,   │   │
│     │                :shared_link                                        │   │
│     └─────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  2. 端点路径的硬性隔离                                                       │
│     ┌─────────────────────────────────────────────────────────────────┐   │
│     │ /api/v1/stats/breakdown 只能通过 API Token 访问                   │   │
│     │ /api/stats/:domain/sources 只能通过界面权限访问                    │   │
│     │                                                                   │   │
│     │ 即使你有有效的 API Token，也无法调用内部 API 端点                   │   │
│     │ 即使你已登录界面，也无法调用公共 API 端点                           │   │
│     │                                                                   │   │
│     │ 原因：不同的路由 pipeline                                           │   │
│     │   - Public API: pipe_through [:public_api, AuthorizePublicAPI]   │   │
│     │   - Internal API: pipe_through :internal_stats_api                │   │
│     │                (包含 AuthPlug + AuthorizeSiteAccess)               │   │
│     └─────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  3. 功能可用性的硬性检查                                                     │
│     ┌─────────────────────────────────────────────────────────────────┐   │
│     │ 场景：Starter 订阅团队（无 StatsAPI 功能）                         │   │
│     │                                                                   │   │
│     │ 界面访问：✅ 正常工作                                              │   │
│     │   - 只检查 Teams.locked?(team)                                    │   │
│     │                                                                   │   │
│     │ API Token 访问：❌ 402 Payment Required                          │   │
│     │   - 检查 Feature.StatsAPI.check_availability(team)               │   │
│     │   - 返回 {:error, :upgrade_required}                              │   │
│     │                                                                   │   │
│     │ 这是设计决策，不是缺陷！                                            │   │
│     └─────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. 稳定性取舍分析

### 6.1 API Token 稳定性保障

#### 6.1.1 速率限制策略

| 限制类型 | 实现位置 | 目的 |
|----------|----------|------|
| **每小时限制** | `check_api_key_rate_limit/2` | 防止滥用，保护系统资源 |
| **突发请求限制** | `check_api_key_burst_limit/1` | 防止瞬间流量冲击 |

```elixir
# 两种限制的组合使用
with {:ok, api_key, limit_key, hourly_limit} <- find_api_key(conn, token, context),
     :ok <- check_api_key_rate_limit(limit_key, hourly_limit),    # 第一步：每小时
     :ok <- check_api_key_burst_limit(limit_key),                  # 第二步：突发
     {:ok, conn} <- verify_by_scope(conn, api_key, requested_scope) do
  # 继续处理
end
```

#### 6.1.2 速率限制 Key 策略

```elixir
# 遗留 API key (无 team_id): 基于用户限制
{:ok, api_key, Auth.ApiKey.legacy_limit_key(api_key.user),
 Auth.ApiKey.legacy_hourly_request_limit()}

# 新团队 API key: 基于团队限制
{:ok, api_key, Auth.ApiKey.limit_key(team), team.hourly_api_request_limit}
```

**设计考量**:
- **遗留模式**: 每个用户独立限制，适合个人开发者
- **团队模式**: 整个团队共享配额，适合企业用户

#### 6.1.3 订阅功能检查

```elixir
# 必须检查 StatsAPI 功能是否可用
:ok <- verify_site_access(
  site: site,
  api_key: api_key,
  feature: Plausible.Billing.Feature.StatsAPI,  # 关键：功能检查
  allow_consolidated_views: conn.private[:allow_consolidated_views]
)
```

**稳定性影响**:
- **优点**: 防止未付费用户滥用付费功能
- **缺点**: 增加了数据库查询开销（检查团队订阅状态）

### 6.2 界面权限稳定性考量

#### 6.2.1 无速率限制的设计决策

界面权限（通过用户会话）**没有速率限制**，原因：

1. **用户体验优先**: 频繁的统计数据刷新是正常操作
2. **会话过期机制**: 通过 Cookie 过期自然限制
3. **超级管理员检查**: `Auth.is_super_admin?()` 是轻量级检查

```elixir
# authorize_site_access.ex:87-102
role =
  cond do
    membership_role -> membership_role
    Plausible.Auth.is_super_admin?(current_user) -> :super_admin  # 轻量级
    site.public -> :public
    shared_link -> :public
    true -> nil
  end
```

#### 6.2.2 共享链接的安全/稳定权衡

| 特性 | 安全性 | 稳定性影响 |
|------|--------|-----------|
| 无密码共享链接 | 较低 | 可能被滥用，但 URL 难以猜测 |
| 有密码共享链接 | 较高 | Cookie 有效期 24 小时 |
| Segment 限制共享链接 | 最高 | 只能查看特定数据段 |

```elixir
# 密码保护的共享链接使用 JWT Token 签名
def authenticate_shared_link(conn, %{"slug" => slug, "password" => password}) do
  # ... 密码验证成功后
  token = Plausible.Auth.Token.sign_shared_link(slug)
  
  conn
  |> put_resp_cookie(shared_link_cookie_name(slug), token)  # 24 小时有效期
  # ...
end
```

### 6.3 查询层稳定性机制

#### 6.3.1 采样机制

```elixir
# query.ex:10
sample_threshold: 20_000_000,  # 2000 万行后启用采样
```

**目的**: 防止大数据量查询拖慢系统

#### 6.3.2 导入数据跳过机制

```elixir
# query.ex:191-210
@spec get_skip_imported_reason(t()) ::
        nil | :no_imported_data | :out_of_range | :unsupported_interval | :unsupported_query

def get_skip_imported_reason(query) do
  cond do
    not Imported.schema_supports_interval?(query) -> :unsupported_interval
    not query.imports_exist -> :no_imported_data
    query.imports_in_range == [] -> :out_of_range
    not Imported.schema_supports_query?(query) -> :unsupported_query
    true -> nil
  end
end
```

**返回给前端的信息**:
```json
{
  "results": [...],
  "meta": {...},
  "skip_imported_reason": "no_imported_data"  // 或其他原因
}
```

#### 6.3.3 错误处理策略

| 错误类型 | API Token 响应 | 界面权限响应 |
|----------|---------------|-------------|
| 无效凭据 | 401 Unauthorized | 404 页面 |
| 缺少权限 | 401 Unauthorized | 404 页面 |
| 速率限制 | 429 Too Many Requests | N/A |
| 升级要求 | 402 Payment Required | 页面提示 |
| 站点锁定 | 402 Payment Required | 专用锁定页面 |
| 查询参数错误 | 400 Bad Request | 400 或静默失败 |

### 6.4 稳定性取舍总结表

| 维度 | API Token 策略 | 界面权限策略 | 设计考量 |
|------|---------------|-------------|---------|
| **速率限制** | 强制启用（每小时 + 突发） | 无速率限制 | API 更易被脚本滥用 |
| **功能检查** | 显式检查 StatsAPI 功能 | 仅检查团队锁定 | API 需要严格计费，界面是核心体验 |
| **错误信息** | 详细 JSON 响应 | 模糊 404 或用户友好页面 | 安全性 vs 用户体验 |
| **数据采样** | 共享采样机制 | 共享采样机制 | 统一保护查询性能 |
| **导入数据** | 条件性包含 | 条件性包含 | 兼容性 vs 完整性 |
| **超级管理员** | 绕过所有检查 | 绕过所有检查 | 运维便利性 |

---

## 7. 关键代码引用

### 7.1 授权层

| 文件 | 关键函数/结构 | 行号 |
|------|--------------|------|
| `authorize_public_api.ex` | `call/2` (主流程) | 50-65 |
| `authorize_public_api.ex` | `verify_site_access/1` (站点验证) | 257-290 |
| `authorize_public_api.ex` | `check_api_key_rate_limit/2` (速率限制) | 187-196 |
| `authorize_site_access.ex` | `call/2` (主流程) | 78-140 |
| `authorize_site_access.ex` | `maybe_get_shared_link/2` (共享链接) | 200-220 |
| `auth_plug.ex` | `call/2` (会话填充) | 18-93 |
| `stats_controller.ex` | `stats/2` (团队锁定检查) | 51-114 |

### 7.2 路由层

| 文件 | 关键配置 | 行号 |
|------|----------|------|
| `router.ex` | `:internal_stats_api` pipeline | 72-78 |
| `router.ex` | `:public_api` + `AuthorizePublicAPI` | 91-93, 337-358 |
| `router.ex` | `:docs_stats_api` pipeline | 80-89 |
| `router.ex` | `/api/v1/stats` 路由 | 337-345 |
| `router.ex` | `/api/v2/query` 路由 | 347-358 |
| `router.ex` | `/api/stats/:domain` 路由 | 277-327 |

### 7.3 控制器层

| 文件 | 关键函数/结构 | 行号 |
|------|--------------|------|
| `api/stats_controller.ex` | `sources/2` (示例端点) | 59-97 |
| `api/stats_controller.ex` | `query/2` (新查询端点) | 40-57 |
| `stats_controller.ex` | `csv_export/2` (复用 API 方法) | 134-196 |
| `api/external_stats_controller.ex` | 公共 API v1 端点 | - |
| `api/external_query_api_controller.ex` | 公共 API v2 端点 | - |

### 7.4 核心查询层

| 文件 | 关键函数/结构 | 行号 |
|------|--------------|------|
| `stats.ex` | `breakdown/4`, `aggregate/3`, `timeseries/3` | 18-28 |
| `stats/query.ex` | `defstruct` (Query 结构体) | 4-34 |
| `stats/query.ex` | `from/3` (查询构建入口) | 75-82 |
| `stats/filters.ex` | `parse/1` (过滤器解析) | 69-82 |

### 7.5 前端层

| 文件 | 关键函数/结构 | 行号 |
|------|--------------|------|
| `stats-query.ts` | `createStatsQuery/2` (参数构建) | 40-61 |
| `stats-query.ts` | `StatsQuery` 类型 | 31-38 |

---

## 8. 架构优势与潜在改进点

### 8.1 当前架构优势

1. **查询能力完全复用**: API 和界面使用相同的 `Stats.breakdown/4` 等核心函数
2. **授权层清晰分离**: 两种授权方式独立实现，但最终都设置相同的 `conn.assigns`
3. **参数统一抽象**: `Query` 结构体作为中间层，隔离了入口差异
4. **渐进式演进**: 支持遗留 `Query.from/3` 和新的 `Query.parse_and_build/3` 两种方式
5. **稳定保障机制**: 采样、速率限制、导入数据跳过等多层保护
6. **明确的权限边界**: API Token 和界面权限有清晰的职责划分和功能边界

### 8.2 潜在改进点

1. **速率限制一致性**: 界面权限也可考虑添加软速率限制，防止恶意刷新
2. **错误处理统一**: 当前 API 返回详细 JSON，界面返回页面，可考虑统一错误格式
3. **监控可观测性**: 可考虑在授权层添加更多 metrics，区分 API 和界面的访问模式
4. **缓存策略**: 两种入口可共享查询结果缓存，进一步提升性能
5. **端点命名统一**: 考虑是否需要统一 Public API 和 Internal API 的端点路径设计

---

## 9. 总结

### 9.1 核心复用机制

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           核心复用机制                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   入口差异层                    统一抽象层                   核心实现层    │
│  ┌─────────────┐              ┌─────────────┐              ┌─────────┐ │
│  │ API Token   │              │             │              │         │ │
│  │ (Bearer)    │──assigns──► │  conn.site  │──Query──►   │ Stats.  │ │
│  │             │              │  conn.user  │   struct     │breakdown│ │
│  │ 🔍 Scopes   │              │ conn.team   │              │         │ │
│  │ 🔍 StatsAPI │              │             │              │Stats.   │ │
│  │ 🔍 速率限制 │              │             │              │aggregate│ │
│  ├─────────────┤              ├─────────────┤              │         │ │
│  │             │              │             │              │Stats.   │ │
│  │ 界面权限     │──assigns──► │  conn.site  │──Query──►   │ Stats.  │ │
  │ (Session)   │              │  conn.user  │   struct     │timeseries│ │
  │ 🔍 角色      │              │ conn.role   │              │         │ │
  │ 🔍 团队锁定  │              │ shared_link │              │Stats.   │ │
  │ 🔍 共享链接  │              │             │              │ query/2 │ │
  └─────────────┘              └─────────────┘              └─────────┘ │
                                                                          │
  关键差异点 (授权层边界):                                                  │
  • API Token: 需要 StatsAPI 功能（高级订阅）                              │
  • 界面权限: 只需要团队没有被锁定（基础订阅即可）                           │
  • API Token: 有速率限制                                                   │
  • 界面权限: 无速率限制                                                     │
                                                                          │
  关键复用点 (核心层共享):                                                  │
  • 相同的 conn.assigns 结构 (site, user, team)                          │
  • 相同的 Query 结构体构建                                                │
  • 相同的 Stats 核心函数调用                                              │
  • 相同的稳定性机制 (采样、导入数据跳过)                                  │
                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 9.2 关键修正点总结

#### 9.2.1 订阅检查的事实偏差修正

**之前的错误结论**:
> 界面权限"隐式通过会话处理"订阅检查

**正确的事实**:

| 检查项 | API Token 授权 | 界面权限授权 |
|--------|---------------|-------------|
| `Teams.locked?(team)` | ✅ 检查 | ✅ 检查 |
| `Feature.StatsAPI.check_availability(team)` | ✅ 检查 | ❌ **不检查** |

**场景示例** (团队有 Starter 订阅，活跃但不含 StatsAPI 功能):
- **API Token 访问**: ❌ 402 Payment Required
- **界面访问**: ✅ 正常使用

**设计意图**:
- API Token 设计为"高级集成功能"，需要明确的功能授权
- 界面是产品的"核心体验"，对所有付费用户开放

#### 9.2.2 权限声明映射

**API Token 方式**:
- 端点: `/api/v1/stats/*`, `/api/v2/query`
- 所需 Scope: `stats:read:*`
- 功能检查: `Feature.StatsAPI`
- 速率限制: 有

**界面权限方式**:
- 端点: `/api/stats/:domain/*`, `/:domain/*`
- 所需角色: `:all_roles` (所有角色)
- 功能检查: 仅 `Teams.locked?`
- 速率限制: 无

### 9.3 最终结论

这套架构通过以下层次实现了查询能力的复用和隔离：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        复用与隔离的层次设计                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  🔒 隔离层 (授权/路由)                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  AuthorizePublicAPI    │  AuthorizeSiteAccess                       │  │
│  │  ┌─────────────────┐   │  ┌─────────────────┐                       │  │
│  │  • Bearer Token   │   │  • Session       │                       │  │
│  │  • Scopes 匹配    │   │  • 角色 匹配      │                       │  │
│  │  • 速率限制       │   │  • 无速率限制     │                       │  │
│  │  • StatsAPI 功能  │   │  • 仅团队锁定检查  │                       │  │
│  │  └─────────────────┘   │  └─────────────────┘                       │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
│                                    ▼                                        │
│  🔄 中间适配层 (控制器/参数)                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  ExternalStatsController  │  Api.StatsController  │  StatsController │  │
│  │  (Public API v1/v2)      │  (Internal API)       │  (Web 界面)      │  │
│  ├───────────────────────────┼───────────────────────┼───────────────────┤  │
│  │  不同的参数格式           │  相同的 Query 构建    │  重用 API 控制器  │  │
│  │  不同的错误处理           │  相同的错误处理       │  不同的渲染方式   │  │
│  └───────────────────────────┴───────────────────────┴───────────────────┘  │
│                                    │                                        │
│                                    ▼                                        │
│  ✅ 完全复用层 (核心查询)                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                                                                       │  │
│  │  Query 结构体 ──► Stats.breakdown/4, Stats.aggregate/3, 等          │  │
│  │                                                                       │  │
│  │  • 相同的查询构建逻辑                                                  │  │
│  │  • 相同的数据库查询执行                                                │  │
│  │  • 相同的稳定性机制 (采样、导入数据跳过)                                │  │
│  │                                                                       │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**关键洞察**:

1. **授权层是唯一的实质性差异**
   - API Token 需要 `StatsAPI` 功能（高级订阅）
   - 界面权限只需要团队没有被锁定（基础订阅即可）
   - 这是设计决策，不是缺陷

2. **核心查询能力完全复用**
   - 从 `Query` 结构体开始，所有代码都是共享的
   - 相同的查询参数、相同的查询执行、相同的稳定性保障

3. **边界清晰，易于维护**
   - 授权层与查询层完全解耦
   - 可以独立调整授权策略而不影响查询逻辑
   - 可以独立优化查询性能而不影响授权机制

这套架构成功地在"严格的 API 集成控制"和"流畅的用户界面体验"之间找到了平衡，同时保持了核心查询能力的高度复用。