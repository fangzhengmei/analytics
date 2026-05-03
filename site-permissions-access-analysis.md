# Plausible Analytics 站点权限与访问机制分析

## 概述

本文档深入分析 Plausible Analytics 中四种访问控制机制的叠加关系、公开报表的无登录查询实现、前后端权限数据来源一致性，以及成员角色变更对共享链接的影响。

---

## 一、四种权限机制的叠加关系

### 1.1 核心发现：两种完全不同的代码路径

**关键区别**：普通 Dashboard 访问和共享链接访问走的是**完全不同的代码路径**，使用不同的权限判定逻辑。

| 访问类型 | URL 格式 | Controller Action | 是否使用 `AuthorizeSiteAccess` |
|---------|---------|------------------|-------------------------------|
| 普通访问 | `/mydomain.com` | `stats/2` | ✅ **是** |
| 共享链接访问 | `/share/mydomain.com?auth=xxx` | `shared_link/2` | ❌ **否** |

**路由配置证据**（`lib/plausible_web/router.ex:72-78, 486-491`）：

```elixir
# API 路由 - 所有请求都走 AuthorizeSiteAccess
pipeline :internal_stats_api do
  plug :accepts, ["json"]
  plug :fetch_session
  plug PlausibleWeb.AuthPlug
  plug PlausibleWeb.Plugs.AuthorizeSiteAccess  # ✅ 使用
  plug PlausibleWeb.Plugs.NoRobots
end

# 共享链接页面路由 - 不走 AuthorizeSiteAccess
scope "/", PlausibleWeb do
  pipe_through [:shared_link]  # 只包含 SecureEmbedHeaders 和 NoRobots
  get "/share/:domain/*path", StatsController, :shared_link  # ❌ 不使用 AuthorizeSiteAccess
end
```

**Controller plug 配置证据**（`lib/plausible_web/controllers/stats_controller.ex:49`）：

```elixir
# 只对 :stats 和 :csv_export action 使用 AuthorizeSiteAccess
# :shared_link action 不使用！
plug(PlausibleWeb.Plugs.AuthorizeSiteAccess when action in [:stats, :csv_export])
```

---

### 1.2 页面渲染阶段的权限判定

#### 场景 A：普通访问（`/mydomain.com`）

**执行流程**：

```
用户访问 /mydomain.com
    ↓
走 :browser pipeline
    ↓
StatsController.stats/2
    ↓
触发 plug AuthorizeSiteAccess（因为 action in [:stats, :csv_export]）
    ↓
权限判定（见下文）
    ↓
渲染 stats.html，传递 site_role
```

**权限判定逻辑**（`lib/plausible_web/plugs/authorize_site_access.ex:86-102`）：

```elixir
role =
  cond do
    membership_role ->
      membership_role      # 第1优先级：团队成员角色

    Plausible.Auth.is_super_admin?(current_user) ->
      :super_admin         # 第2优先级：超级管理员

    site.public ->
      :public              # 第3优先级：公开站点

    shared_link ->
      :public              # 第4优先级：共享链接

    true ->
      nil                  # 无权限
  end
```

**关键注意**：普通访问 URL 中**没有** `slug` 或 `auth` 参数，所以 `maybe_get_shared_link` 总是返回 `{:ok, nil}`。也就是说，**普通访问不会触发共享链接的权限判定**。

---

#### 场景 B：共享链接访问（`/share/mydomain.com?auth=xxx`）

**执行流程**：

```
用户访问 /share/mydomain.com?auth=xxx
    ↓
走 :shared_link pipeline（无 AuthorizeSiteAccess）
    ↓
StatsController.shared_link/2
    ↓
直接调用 find_shared_link(domain, auth) 检查链接是否存在
    ↓
检查共享链接是否存在、密码是否正确
    ↓
调用 render_shared_link(conn, shared_link)
    ↓
site_role = get_fallback_site_role(conn)  # 关键！
    ↓
渲染 stats.html，传递 site_role: :public
```

**关键代码证据**（`lib/plausible_web/controllers/stats_controller.ex:466-541`）：

```elixir
defp render_shared_link(conn, shared_link) do
  # ... 检查站点锁定、功能可用性等
  
  current_user = conn.assigns[:current_user]  # 可能有值（用户已登录）
  site_role = get_fallback_site_role(conn)     # 关键！
  
  # ...
  
  render("stats.html",
    site: shared_link.site,
    site_role: site_role,           # 传递给前端
    shared_link_auth: shared_link.slug,  # 共享链接标识
    # ...
  )
end

defp get_fallback_site_role(conn),
  do: if(role = conn.assigns[:site_role], do: role, else: :public)
```

**严格纠偏后的结论**：

1. **`shared_link/2` action 不使用 `AuthorizeSiteAccess` plug**
2. 因为没走 `AuthorizeSiteAccess`，`conn.assigns[:site_role]` 是 `nil`
3. `get_fallback_site_role(conn)` 检查 `conn.assigns[:site_role]`，如果没有则返回 `:public`
4. 所以 **`site_role` 被硬编码为 `:public`**
5. **即使 `current_user` 存在且是团队成员，页面渲染阶段也完全忽略这个事实**

---

### 1.3 API 调用阶段的权限判定

前端加载后，会通过 API 获取实际数据。**所有 API 调用**都走 `internal_stats_api` pipeline，包含 `AuthorizeSiteAccess`。

#### 前端如何传递共享链接认证

**两种方式**（`assets/js/dashboard/api.ts:44-107, 122-157`）：

1. **URL 查询参数**：`?auth=xxx`
   ```typescript
   const sharedLinkParams = getSharedLinkSearchParams()
   if (sharedLinkParams.auth) {
     queryObj.auth = sharedLinkParams.auth
   }
   ```

2. **HTTP 头**：`X-Shared-Link-Auth: xxx`
   ```typescript
   function getHeaders(): Record<string, string> {
     return SHARED_LINK_AUTH ? { 'X-Shared-Link-Auth': SHARED_LINK_AUTH } : {}
   }
   ```

**前端初始化证据**（`assets/js/dashboard.tsx:38-42`）：

```typescript
const sharedLinkAuth = container.dataset.sharedLinkAuth

if (sharedLinkAuth) {
  api.setSharedLinkAuth(sharedLinkAuth)  // 设置全局变量，所有 API 调用都会携带
}
```

---

#### API 端 `AuthorizeSiteAccess` 的完整逻辑

**关键代码**（`lib/plausible_web/plugs/authorize_site_access.ex:78-140`）：

```elixir
def call(conn, {allowed_roles, site_param}) do
  current_user = conn.assigns[:current_user]

  with {:ok, domain} <- get_domain(conn, site_param),
       # 第1步：获取成员角色（独立于共享链接）
       {:ok, %{site: site, role: membership_role, member_type: member_type}} <-
         get_site_with_role(conn, current_user, domain),
       :ok <- ensure_consolidated_view_access(conn, site),
       # 第2步：获取共享链接（独立于成员角色）
       {:ok, shared_link} <- maybe_get_shared_link(conn, site) do
    
    # 第3步：角色判定（优先级从高到低）
    role =
      cond do
        membership_role ->
          membership_role      # 第1优先级：成员角色

        Plausible.Auth.is_super_admin?(current_user) ->
          :super_admin         # 第2优先级：超级管理员

        site.public ->
          :public              # 第3优先级：公开站点

        shared_link ->
          :public              # 第4优先级：共享链接

        true ->
          nil                  # 无权限
      end

    if role in allowed_roles do
      # ...
      
      # 关键！无论 role 是什么，shared_link 都会被赋值
      conn = merge_assigns(conn, site: site, site_role: role, shared_link: shared_link)
      
      # ...
    else
      error_not_found(conn)
    end
  end
end
```

**`maybe_get_shared_link` 的实现**（`lib/plausible_web/plugs/authorize_site_access.ex:200-220`）：

```elixir
defp maybe_get_shared_link(conn, site) do
  # 关键！auth 可以来自 URL 查询参数或 POST body
  slug = conn.path_params["slug"] || conn.params["auth"]

  if valid_path_fragment?(slug) do
    # 查询 shared_links 表
    with %Plausible.Site.SharedLink{} = shared_link <-
           Repo.get_by(Plausible.Site.SharedLink, slug: slug, site_id: site.id),
         # ... 密码验证
      {:ok, shared_link}
    else
      # ... 错误处理
    end
  else
    {:ok, nil}
  end
end
```

---

#### 严格纠偏后的 API 权限判定规则

**规则 1：`membership_role` 和 `shared_link` 独立获取**

- `membership_role`：通过 `get_site_with_role` 获取，检查 `current_user` 是否是团队成员
- `shared_link`：通过 `maybe_get_shared_link` 获取，检查 `conn.params["auth"]`
- 两者**完全独立**，互不影响

**规则 2：`role` 判定时 `membership_role` 优先级最高**

```elixir
cond do
  membership_role ->
    membership_role      # ✅ 如果用户是成员，使用成员角色，忽略 shared_link

  # ... 其他条件

  shared_link ->
    :public              # ❌ 只有前面都不满足时才使用共享链接
end
```

**规则 3：`conn.assigns[:shared_link]` 总是被赋值**

```elixir
# 无论 role 是什么，shared_link 都会被赋值
conn = merge_assigns(conn, site: site, site_role: role, shared_link: shared_link)
```

这意味着：
- 如果 `membership_role` 存在 → `site_role = membership_role`，`shared_link = %SharedLink{}`
- 如果 `membership_role` 不存在但 `shared_link` 存在 → `site_role = :public`，`shared_link = %SharedLink{}`

---

### 1.4 `validate_required_filters_plug`：segment 强制过滤约束

#### 触发条件

**代码证据**（`lib/plausible_web/controllers/api/stats_controller.ex:38, 1331-1351`）：

```elixir
# 只对除 :current_visitors 外的 action 触发
plug(:validate_required_filters_plug when action not in [:current_visitors])

defp validate_required_filters_plug(
       # 模式匹配：只在以下条件都满足时触发
       %Plug.Conn{assigns: %{shared_link: %Plausible.Site.SharedLink{segment_id: segment_id}}} =
         conn,
       _opts
     )
     when is_integer(segment_id) do  # segment_id 必须是整数（不是 nil）
  # ... 强制第一个 filter 是该 segment
end

# 默认：不做任何事
defp validate_required_filters_plug(conn, _opts), do: conn
```

**严格纠偏后的触发条件**：

| 条件 | 是否必须满足 |
|------|-------------|
| `conn.assigns[:shared_link]` 是 `%Plausible.Site.SharedLink{}`（不是 `nil`） | ✅ 是 |
| `shared_link.segment_id` 是整数（不是 `nil`） | ✅ 是 |
| `site_role` 是什么 | ❌ 完全无关 |

**关键结论**：`validate_required_filters_plug` 的触发**与 `site_role` 完全无关**！只看 `shared_link` 是否存在且有 `segment_id`。

---

#### 实际约束效果

**代码证据**（`lib/plausible_web/controllers/api/stats_controller.ex:1365-1381`）：

```elixir
defp ensure_expected_segment_filter_present(
       filters,
       expected_segment_id
     )
     when is_list(filters) do
  case filters do
    # ✅ 允许：第一个 filter 是该 segment，后面可以跟任意其他 filter
    [["is", "segment", [segment_id]] | _other_filters] when segment_id == expected_segment_id ->
      :ok

    # ❌ 拒绝：其他任何情况
    _ ->
      :error
  end
end
```

**严格纠偏后的约束效果**：

1. **只检查第一个 filter**：必须是 `["is", "segment", [expected_segment_id]]`
2. **允许其他 filter**：`_other_filters` 可以是任意其他 filter
3. **不检查实际数据范围**：没有代码验证返回的数据是否在 segment 范围内

**这意味着**：
- 如果用户满足第一个 filter 是该 segment，他们可以添加任意其他 filter
- 这不是一个严格的数据访问限制，而是一个"默认视图"的强制

---

### 1.5 关键场景详细分析

#### 场景 1：未登录用户通过共享链接访问

| 阶段 | 权限判定 | 结果 |
|------|---------|------|
| 页面渲染 | `shared_link/2` 硬编码 `:public` | `site_role = :public` |
| API 调用 | `membership_role = nil`，`shared_link` 存在 | `role = :public` |
| segment 约束（如有） | `shared_link.segment_id` 存在 | 触发 `validate_required_filters_plug` |

**一致性**：页面渲染和 API 调用都使用 `:public` 角色 ✅

---

#### 场景 2：已登录的团队成员通过普通 URL 访问

| 阶段 | 权限判定 | 结果 |
|------|---------|------|
| 页面渲染 | `AuthorizeSiteAccess` 中 `membership_role` 存在 | `site_role = :admin` |
| API 调用 | `membership_role` 存在，优先级最高 | `role = :admin` |
| segment 约束 | URL 中无 `auth`，`shared_link = nil` | 不触发 |

**一致性**：页面渲染和 API 调用都使用成员角色 ✅

---

#### 场景 3：已登录的团队成员通过共享链接 URL 访问 ⚠️

**这是最关键的场景**，需要**严格纠偏**。

##### 页面渲染阶段：

| 项目 | 值 | 说明 |
|------|-----|------|
| URL | `/share/mydomain.com?auth=xxx` | |
| Action | `shared_link/2` | |
| `AuthorizeSiteAccess` | 不使用 | |
| `conn.assigns[:site_role]` | `nil` | 没走 `AuthorizeSiteAccess` |
| `get_fallback_site_role` | `:public` | 硬编码 |
| **最终 `site_role`** | **`:public`** | 与用户实际身份无关 |

##### API 调用阶段：

| 项目 | 值 | 说明 |
|------|-----|------|
| URL | `/api/stats/mydomain.com/top-stats?auth=xxx` | |
| `AuthorizeSiteAccess` | 使用 | |
| `get_site_with_role` | `membership_role = :admin` | 用户是成员 |
| `maybe_get_shared_link` | `shared_link = %SharedLink{segment_id: 123}` | 有 segment 限制 |
| `role` 判定 | `role = :admin` | `membership_role` 优先级高 |
| `conn.assigns[:shared_link]` | `%SharedLink{segment_id: 123}` | 总是被赋值 |
| `validate_required_filters_plug` | 触发 | `shared_link.segment_id` 是整数 |

##### 严格纠偏后的完整结论：

| 层面 | 实际值 | 说明 |
|------|--------|------|
| 页面渲染 `site_role` | `:public` | 硬编码，与用户身份无关 |
| API `site_role` | `:admin`（成员角色） | `membership_role` 优先级最高 |
| `conn.assigns[:shared_link]` | `%SharedLink{segment_id: 123}` | 总是被赋值，与 `role` 无关 |
| `validate_required_filters_plug` | 触发 | 只看 `shared_link`，与 `site_role` 无关 |

##### 这意味着什么？

1. **UI 层面**：前端看到 `site_role = :public`，所以：
   - 隐藏设置入口
   - 隐藏邀请成员按钮
   - 显示共享链接视图（只读模式）

2. **数据访问权限**：API 实际使用 `site_role = :admin`，所以：
   - 拥有完整的数据访问权限
   - 不是绕过，而是用户本来就有权限

3. **segment 约束**：如果共享链接有 segment 限制：
   - `validate_required_filters_plug` 会触发
   - 强制第一个 filter 是该 segment
   - 但允许添加其他 filter
   - 这是"默认视图"的强制，不是严格的数据限制

##### 这是有意设计吗？

从代码逻辑来看，这似乎是**有意的设计**：

1. **页面渲染阶段**：用户通过共享链接访问，应该看到分享的视图（只读模式）
2. **API 调用阶段**：
   - 如果用户本来就是团队成员，他们应该拥有完整的数据访问权限
   - 但如果共享链接有 segment 限制，应该尊重这个限制（至少在默认视图上）
3. **这种设计避免了**："用户明明有权限却因为用了分享链接而被降权"的问题

---

### 1.6 权限叠加规则总结

#### 规则 1：页面渲染阶段

| 访问路径 | `site_role` 来源 | 实际 `site_role` |
|---------|-----------------|-----------------|
| `/mydomain.com`（普通） | `AuthorizeSiteAccess` 判定 | 成员角色 或 `:public` |
| `/share/mydomain.com`（共享链接） | `get_fallback_site_role` 硬编码 | **总是 `:public`** |

**关键**：共享链接访问的页面渲染阶段**完全忽略**用户的实际成员身份。

---

#### 规则 2：API 调用阶段 - `role` 判定

**优先级从高到低**：

```
1. membership_role（团队成员角色）← 最高优先级，与 shared_link 无关
         ↓
2. :super_admin（超级管理员）
         ↓
3. site.public（公开站点）
         ↓
4. shared_link（共享链接）← 最低优先级，只有前面都不满足时才用
         ↓
5. 拒绝访问
```

**关键**：`membership_role` 和 `shared_link` **独立获取**，`role` 判定时 `membership_role` 优先级更高。

---

#### 规则 3：API 调用阶段 - `shared_link` 赋值

```elixir
# 无论 role 是什么，shared_link 总是被赋值
conn = merge_assigns(conn, site: site, site_role: role, shared_link: shared_link)
```

| 场景 | `site_role` | `shared_link` |
|------|-------------|---------------|
| 成员通过普通 URL 访问 | `:admin` | `nil` |
| 成员通过共享链接 URL 访问 | `:admin` | `%SharedLink{}` |
| 非成员通过共享链接访问 | `:public` | `%SharedLink{}` |

---

#### 规则 4：`validate_required_filters_plug` 触发条件

| 条件 | 是否触发 |
|------|---------|
| `shared_link` 存在且 `segment_id` 是整数 | ✅ 触发 |
| `shared_link` 为 `nil` | ❌ 不触发 |
| `shared_link.segment_id` 为 `nil` | ❌ 不触发 |
| `site_role` 是 `:admin` | 无关，不影响触发 |
| `site_role` 是 `:public` | 无关，不影响触发 |

**关键**：触发条件**与 `site_role` 完全无关**！

---

#### 规则 5：成员角色与共享链接同时存在时的最终效果

当**已登录的团队成员**通过**共享链接 URL**访问时：

| 维度 | 实际值 | 效果 |
|------|--------|------|
| 前端 UI `siteRole` | `'public'` | 只读模式，隐藏管理功能 |
| API `site_role` | 成员角色（`:admin`） | 完整数据访问权限 |
| `shared_link`（如有） | 存在 | 影响 `validate_required_filters_plug` |
| `segment_id`（如有） | 存在 | 强制第一个 filter 是该 segment |

**这不是绕过**：用户本来就是成员，有权限访问所有数据。共享链接的 segment 限制只是"默认视图"的强制，不是严格的数据访问限制。

---

## 二、公开报表无登录复用查询能力的实现

### 2.1 核心设计思想

**同一套查询逻辑，不同的权限入口**

所有统计数据查询都通过 `Plausible.Stats` 模块实现，无论用户是否登录，查询逻辑完全相同。区别仅在于：
1. 如何获取 `site` 对象
2. 如何进行权限验证
3. 如何处理共享链接的特殊限制（如 segment 强制 filter）

---

### 2.2 实现架构

#### (1) 查询逻辑的统一入口

**位置**：`lib/plausible_web/controllers/api/stats_controller.ex`

所有 API 端点都使用相同的模式：

```elixir
def sources(conn, params) do
  site = conn.assigns[:site]  # 从 assigns 获取，与权限来源无关
  params = Map.put(params, "property", "visit:source")
  query = Query.from(site, params, debug_metadata: debug_metadata(conn))
  
  # 同一套查询逻辑
  %{results: results, meta: meta} = Stats.breakdown(site, query, metrics, pagination)
  
  json(conn, %{results: results, meta: meta})
end
```

---

#### (2) 权限验证的不同入口

| 访问方式 | 权限验证入口 | site 来源 |
|---------|-------------|----------|
| 普通访问（已登录） | `AuthorizeSiteAccess` 检查成员角色 | `get_site_with_role` |
| 公开站点（未登录） | `AuthorizeSiteAccess` 检查 `site.public` | 直接查询 `sites` 表 |
| 共享链接（页面渲染） | `StatsController.shared_link/2` | `find_shared_link` 预加载 |
| 共享链接（API 调用） | `AuthorizeSiteAccess` 检查 `shared_link` | `get_site_with_role` + `maybe_get_shared_link` |

---

#### (3) 前端如何感知访问模式

前端通过页面渲染时注入的变量来区分：

| 变量 | 普通访问 | 公开站点 | 共享链接 |
|------|---------|---------|---------|
| `site_role` | 成员角色（`:admin` 等） | `:public` | `:public` |
| `shared_link_auth` | `nil` | `nil` | 有值（slug） |
| `limited_to_segment_id` | `nil` | `nil` | segment ID（如有） |

---

### 2.3 无登录访问的关键技术点

#### (1) Session 处理

`:internal_stats_api` pipeline 包含 `:fetch_session` 和 `AuthPlug`：

```elixir
pipeline :internal_stats_api do
  plug :accepts, ["json"]
  plug :fetch_session      # 允许读取 session
  plug PlausibleWeb.AuthPlug  # 尝试获取 current_user
  plug PlausibleWeb.Plugs.AuthorizeSiteAccess
  plug PlausibleWeb.Plugs.NoRobots
end
```

这意味着：
- 对于已登录用户，`current_user` 会被设置
- 对于未登录用户，`current_user` 是 `nil`
- `AuthorizeSiteAccess` 会处理这两种情况

---

#### (2) 共享链接认证的无状态性

共享链接认证不依赖 session（除了密码保护的 cookie）：

- **无密码保护**：只需要 URL 中的 `?auth=xxx` 参数
- **有密码保护**：验证通过后设置 cookie `shared-link-<slug>`，有效期由 `Plausible.Auth.Token.sign_shared_link/1` 控制

**密码验证逻辑**（`lib/plausible_web/controllers/stats_controller.ex:298-306`）：

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

---

### 2.4 共享链接 segment 限制的特殊性

#### 严格纠偏后的理解

之前可能存在一个误解：共享链接的 segment 限制是一个严格的数据访问控制。

**实际情况**（基于代码证据）：

1. **`validate_required_filters_plug` 只强制第一个 filter 是该 segment**
2. **它允许添加任意其他 filter**
3. **它不检查实际返回的数据范围**
4. **对于成员用户，`site_role` 是成员角色，拥有完整权限**

#### 这意味着什么？

对于**非成员用户**通过共享链接访问：
- `site_role = :public`
- 如果有 segment 限制，强制第一个 filter 是该 segment
- 但可以添加其他 filter
- 这不是一个完美的数据隔离机制

对于**成员用户**通过共享链接访问：
- `site_role = 成员角色`（完整权限）
- 如果有 segment 限制，强制第一个 filter 是该 segment
- 但用户本来就有权限访问所有数据
- 这只是"默认视图"的强制，不是限制

---

## 三、前后端权限数据来源一致性

### 3.1 服务端权限数据来源

服务端有两个主要的权限数据获取入口：

#### (1) 站点级别的角色（`site_role`）

**来源**：`AuthorizeSiteAccess.call/2`

**数据来源**：
- 成员角色：`Teams.Memberships.site_role/2` → 直接查询 `team_memberships` 和 `guest_memberships` 表
- 共享链接：`maybe_get_shared_link` → 查询 `shared_links` 表
- 公开站点：`site.public` → `sites` 表字段

---

#### (2) 团队级别的角色（`current_team_role`）

**来源**：`AuthPlug.call/2`

**数据来源**：从 `user.team_memberships` 关联获取（Ecto 预加载）

```elixir
team_membership =
  Enum.find(user.team_memberships, %{}, &(&1.team.identifier == current_team_id))

{Map.get(team_membership, :team), Map.get(team_membership, :role)}
```

---

### 3.2 前端权限数据来源

前端获取权限数据有两个途径：

#### (1) 页面渲染时注入

**位置**：`stats.html.eex` 模板

模板从 Controller 接收：
- `@site_role` - 访问者角色
- `@shared_link_auth` - 共享链接 slug（如有）
- `@limited_to_segment_id` - segment 限制（如有）

---

#### (2) 前端 Context

**位置**：`assets/js/dashboard/site-context.tsx`

前端创建一个 Context 来存储站点信息，包括：

```typescript
interface PlausibleSite {
  domain: string
  siteRole: SiteRole  // 'admin' | 'owner' | 'viewer' | 'editor' | 'super_admin' | 'public'
  sharedLinkAuth?: string
  // ...
}
```

---

### 3.3 一致性分析

#### 情况 1：普通访问（已登录）

| 层面 | 数据来源 | 一致性 |
|------|---------|--------|
| 服务端 `site_role` | `AuthorizeSiteAccess` 实时查询 | ✅ 一致 |
| 前端 `siteRole` | 页面渲染时注入 | ✅ 与服务端一致 |
| API 调用 | `AuthorizeSiteAccess` 实时查询 | ✅ 与页面渲染一致 |

---

#### 情况 2：共享链接访问（未登录）

| 层面 | 数据来源 | 一致性 |
|------|---------|--------|
| 页面渲染 `site_role` | `shared_link/2` 硬编码 `:public` | ✅ 一致 |
| 前端 `siteRole` | 页面渲染时注入 | ✅ `'public'` |
| API 调用 `role` | `AuthorizeSiteAccess` 判定 `:public` | ✅ 与页面渲染一致 |

---

#### 情况 3：共享链接访问（已登录成员）⚠️

| 层面 | 数据来源 | 结果 |
|------|---------|------|
| 页面渲染 `site_role` | `shared_link/2` 硬编码 | `:public` |
| 前端 `siteRole` | 页面渲染时注入 | `'public'` |
| API 调用 `role` | `AuthorizeSiteAccess` 中 `membership_role` 优先 | 实际成员角色（`:admin` 等） |
| API `shared_link` | `maybe_get_shared_link` | 存在（影响 segment 约束） |

**严格纠偏后的不一致性说明**：

这是一个**有意的设计**，而不是 bug：

1. **UI 层面**：用户通过共享链接访问，应该看到分享的视图（只读模式）
2. **数据层面**：
   - 用户本来就是团队成员，不应该因为用了分享链接而被降权
   - 但如果共享链接有 segment 限制，`validate_required_filters_plug` 会强制第一个 filter 是该 segment
3. **segment 限制的本质**：
   - 这不是严格的数据访问控制
   - 这是"默认视图"的强制
   - 用户可以添加其他 filter

**安全保障**：
- 对于非成员用户，`site_role = :public`，即使绕过 segment filter，也没有额外权限
- 对于成员用户，他们本来就有权限访问所有数据，这不是安全问题

---

### 3.4 潜在的不一致场景

#### (1) 角色变更后的缓存问题

- `AuthPlug` 使用预加载的 `user.team_memberships`
- `AuthorizeSiteAccess` 每次请求实时查询
- 如果角色在会话期间变更，`AuthPlug` 的数据可能滞后

**影响**：
- `current_team_role`（团队级别）可能滞后
- `site_role`（站点级别）是实时查询的，总是准确的

---

#### (2) 团队角色 vs 站点角色

- `current_team_role` 是团队级别的角色
- `site_role` 是站点级别的角色（考虑 guest membership）

**区别**：
- 对于普通成员，两者相同
- 对于 `:guest` 角色：
  - `current_team_role = :guest`
  - `site_role` 取决于是否有 `guest_membership` 关联到该站点

---

## 四、成员角色变更对共享链接的影响

### 4.1 共享链接的独立权限模型

**核心结论**：共享链接是**完全独立**的权限授予机制，**不依赖于**创建者的当前角色或成员状态。

---

#### (1) 数据模型分析

**位置**：`lib/plausible/site/shared_link.ex`

```elixir
schema "shared_links" do
  belongs_to :site, Plausible.Site
  field :name, :string
  field :slug, :string
  field :password_hash, :string
  field :password, :string, virtual: true
  belongs_to :segment, Plausible.Segments.Segment
  timestamps()
end
```

**关键观察**：
- ❌ 没有 `creator_id` 或 `created_by` 字段
- ❌ 没有记录创建者的角色
- ✅ 只关联到 `site_id`
- ✅ 权限完全由记录是否存在决定

---

#### (2) 验证逻辑分析

**位置**：`lib/plausible_web/plugs/authorize_site_access.ex:200-220`

```elixir
defp maybe_get_shared_link(conn, site) do
  slug = conn.path_params["slug"] || conn.params["auth"]

  if valid_path_fragment?(slug) do
    with %Plausible.Site.SharedLink{} = shared_link <-
           Repo.get_by(Plausible.Site.SharedLink, slug: slug, site_id: site.id),
         # ... 密码验证
      {:ok, shared_link}
    else
      # ... 错误处理
    end
  else
    {:ok, nil}
  end
end
```

**验证步骤**：
1. 从 URL 获取 `slug`（或 `auth` 查询参数）
2. 查询 `shared_links` 表，检查是否存在匹配的记录（`slug` + `site_id`）
3. 如果有密码保护，验证密码 cookie
4. 返回共享链接对象或错误

**关键点**：**没有任何步骤检查创建者的角色或状态！**

---

### 4.2 影响共享链接可用性的因素

以下因素会影响共享链接的可用性：

| 因素 | 是否影响 | 说明 |
|------|---------|------|
| 共享链接记录被删除 | ✅ 是 | 管理员可以在设置中撤销 |
| 站点被锁定（订阅过期） | ✅ 是 | 团队欠费或订阅过期 |
| 共享链接功能不可用 | ✅ 是 | 需要 Business 计划 |
| 密码被修改 | ✅ 是 | 访问者需要重新输入新密码 |
| 创建者角色变更 | ❌ 否 | 共享链接独立于创建者权限 |
| 创建者被移出团队 | ❌ 否 | 共享链接仍然有效 |
| 站点不再公开 | ❌ 否 | 共享链接不受 `site.public` 影响 |
| 创建者被降级为 `:viewer` | ❌ 否 | 不影响已创建的链接 |
| 创建者被降级为 `:guest` | ❌ 否 | 不影响已创建的链接 |

---

### 4.3 实际场景分析

#### 场景 1：管理员创建共享链接后被降级为查看者

**结果**：共享链接 **仍然可用**

原因：
- 共享链接记录存在于数据库
- 验证逻辑不检查创建者角色
- 即使创建者变成 `:viewer`，链接仍然有效

---

#### 场景 2：管理员创建共享链接后被移出团队

**结果**：共享链接 **仍然可用**

原因：
- 共享链接没有 `creator_id` 字段
- 验证逻辑只检查链接本身是否存在
- 创建者是否在团队中不影响链接有效性

---

#### 场景 3：成员被降级为 guest，其创建的共享链接

**结果**：共享链接 **仍然可用**

原因：
- `:guest` 角色只是限制该成员的访问
- 已创建的共享链接是独立的权限授予

---

#### 场景 4：管理员撤销共享链接

**结果**：共享链接 **立即失效**

原因：
- 管理员在设置中删除共享链接记录
- `maybe_get_shared_link` 查询不到记录，返回 404

---

### 4.4 共享链接的撤销机制

由于共享链接不依赖创建者权限，撤销只能通过以下方式：

#### (1) 管理员手动删除

任何拥有站点管理权限的用户（`:owner`、`:admin`）都可以在站点设置中查看和删除所有共享链接。

---

#### (2) 修改共享链接的密码

对于有密码保护的共享链接，修改密码会使所有旧的认证 cookie 失效。访问者需要重新输入新密码。

---

#### (3) 站点层面的撤销

- 锁定站点（团队订阅过期）
- 取消共享链接功能（降级计划）

---

### 4.5 安全考虑

#### (1) 离职员工问题

如果一个员工创建了共享链接后离开公司（被移出团队）：
- 共享链接**仍然有效**
- 这可能是预期行为（分享给外部合作伙伴的链接不应该因为员工离职而失效）
- 但也可能是安全风险（如果链接包含敏感数据）

**建议**：
- 定期审计共享链接
- 使用密码保护和过期时间（如果支持）
- 员工离职时检查并撤销其创建的共享链接

---

#### (2) 权限撤销的粒度

当前实现：
- 只能撤销整个共享链接
- 不能部分撤销（如限制某些访问者）

---

### 4.6 设计意图分析

共享链接的独立权限模型是**有意的设计**：

1. **分享的本质**：共享链接应该是一个独立的"访问令牌"，而不是创建者权限的代理
2. **使用场景**：
   - 分享给外部客户/合作伙伴 → 不应该因为员工离职而失效
   - 嵌入到其他网站 → 需要长期稳定的访问
3. **管理控制**：管理员仍然可以随时撤销任何共享链接

---

## 五、总结

### 5.1 权限叠加机制总结

#### 页面渲染阶段

| 访问路径 | 成员角色是否生效 | 共享链接是否生效 |
|---------|-----------------|-----------------|
| `/mydomain.com`（普通） | ✅ 是，优先级最高 | ❌ URL 中无 auth 参数 |
| `/share/mydomain.com`（共享链接） | ❌ **完全忽略**，硬编码 `:public` | ✅ 是，但页面只使用 `:public` |

**关键**：共享链接访问的页面渲染阶段**完全忽略**用户的实际成员身份，`site_role` 总是 `:public`。

---

#### API 调用阶段（所有路径）

**`role` 判定优先级**（从高到低）：

```
1. membership_role（团队成员角色）← 最高优先级，与 shared_link 无关
         ↓
2. :super_admin（超级管理员）
         ↓
3. site.public（公开站点）
         ↓
4. shared_link（共享链接）← 最低优先级，只有前面都不满足时才用
         ↓
5. 拒绝访问
```

**`conn.assigns` 的设置**：
- `site_role` = 上述判定的 `role`
- `shared_link` = `maybe_get_shared_link` 的返回值（**总是被赋值，与 role 无关**）

---

#### `validate_required_filters_plug`（segment 强制过滤）

**触发条件**（与 `site_role` 完全无关）：
- `conn.assigns[:shared_link]` 是 `%SharedLink{}`（不是 `nil`）
- `shared_link.segment_id` 是整数（不是 `nil`）

**实际效果**：
- 只强制**第一个** filter 是该 segment
- **允许**后面跟任意其他 filter
- 不检查实际返回的数据范围
- 这是"默认视图"的强制，不是严格的数据访问控制

---

#### 关键不一致场景：已登录成员通过共享链接访问

| 层面 | 实际值 | 说明 |
|------|--------|------|
| 页面渲染 `site_role` | `:public` | 硬编码，与用户身份无关 |
| API `site_role` | 成员角色（`:admin` 等） | `membership_role` 优先级最高 |
| `conn.assigns[:shared_link]` | 存在 | 总是被赋值，与 `role` 无关 |
| `validate_required_filters_plug`（如有 segment） | 触发 | 只看 `shared_link`，与 `site_role` 无关 |

**设计意图**：
- UI 层面：用户通过分享链接访问时看到分享视图（只读模式）
- 数据层面：
  - 如果用户本来就是团队成员，拥有完整数据访问权限
  - 如果共享链接有 segment 限制，强制第一个 filter 是该 segment（默认视图）
- 这不是绕过，而是用户本来就有权限

---

### 5.2 公开报表无登录查询实现总结

**核心原理**：

1. **统一的查询入口**：所有统计查询都通过 `Plausible.Stats` 模块
2. **统一的上下文传递**：无论权限来源，`site` 都赋值到 `conn.assigns`
3. **不同的权限前置**：通过不同的 plug 和 controller action 进行权限验证
4. **前端感知差异**：通过 `site_role`、`shared_link_auth`、`limited_to_segment_id` 区分访问模式

**好处**：
- 代码复用：一套查询逻辑支持所有访问模式
- 维护简单：修改查询逻辑只需改一处
- 功能一致：公开访问和登录访问看到相同的数据（除了 UI 限制）

**segment 限制的特殊性**：
- 不是严格的数据访问控制
- 只是"默认视图"的强制
- 对于成员用户，他们本来就有权限访问所有数据

---

### 5.3 前后端权限数据一致性总结

| 维度 | 服务端 | 前端 |
|------|--------|------|
| 数据来源 | `team_memberships`、`shared_links`、`sites` 表 | 页面渲染时注入 |
| 获取方式 | `AuthorizeSiteAccess` 实时查询 | 模板变量赋值 |
| 更新时机 | 每次请求 | 页面刷新时 |

**潜在不一致**：
- 已登录成员通过共享链接访问时：
  - 页面渲染：`site_role = :public`
  - API 调用：`site_role = 成员角色`，`shared_link` 存在
- 这是有意的设计，不是 bug

**安全保障**：
- 对于非成员用户，`site_role = :public`，没有额外权限
- 对于成员用户，他们本来就有权限访问所有数据
- segment 限制只是"默认视图"的强制，不是安全问题

---

### 5.4 成员角色变更对共享链接影响总结

**核心结论**：成员角色变更**完全不影响**已创建的共享链接。

**原因**：
1. 共享链接模型**没有** `creator_id` 字段
2. 验证逻辑**只检查**链接记录是否存在
3. 共享链接是**独立的**权限授予，不是创建者权限的代理

**影响共享链接的因素**：
- ✅ 链接被管理员删除
- ✅ 站点被锁定（订阅过期）
- ✅ 密码被修改（有密码保护的链接）
- ❌ 创建者角色变更
- ❌ 创建者被移出团队

**管理建议**：
- 定期审计共享链接
- 员工离职时检查并撤销其创建的链接
- 敏感数据使用密码保护

---

## 六、关键代码位置索引

| 功能模块 | 文件路径 | 关键行号 |
|---------|---------|---------|
| 权限判定核心逻辑 | `lib/plausible_web/plugs/authorize_site_access.ex` | 78-140 |
| 角色判定 cond 表达式 | `lib/plausible_web/plugs/authorize_site_access.ex` | 86-102 |
| conn.assigns 赋值 | `lib/plausible_web/plugs/authorize_site_access.ex` | 121 |
| 共享链接获取逻辑 | `lib/plausible_web/plugs/authorize_site_access.ex` | 200-220 |
| 共享链接页面渲染 | `lib/plausible_web/controllers/stats_controller.ex` | 260-296, 441-538 |
| get_fallback_site_role 硬编码 | `lib/plausible_web/controllers/stats_controller.ex` | 540-541 |
| Controller plug 配置 | `lib/plausible_web/controllers/stats_controller.ex` | 49 |
| validate_required_filters_plug 定义 | `lib/plausible_web/controllers/api/stats_controller.ex` | 38, 1331-1351 |
| ensure_expected_segment_filter_present | `lib/plausible_web/controllers/api/stats_controller.ex` | 1365-1381 |
| 共享链接模型 | `lib/plausible/site/shared_link.ex` | 1-58 |
| 团队角色获取 | `lib/plausible/teams/memberships.ex` | 80-101 |
| 登录用户权限填充 | `lib/plausible_web/plugs/auth_plug.ex` | 1-94 |
| 路由配置 | `lib/plausible_web/router.ex` | 72-78, 486-491 |
| 前端 API 认证传递 | `assets/js/dashboard/api.ts` | 44-107, 122-157 |
| 前端 sharedLinkAuth 初始化 | `assets/js/dashboard.tsx` | 38-42 |
