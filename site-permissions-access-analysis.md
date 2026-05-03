# Plausible Analytics 站点权限与访问机制分析

## 概述

本文档深入分析 Plausible Analytics 中四种访问控制机制的叠加关系、公开报表的无登录查询实现、前后端权限数据来源一致性，以及成员角色变更对共享链接的影响。

---

## 一、四种权限机制的叠加关系

### 1.1 权限判定优先级

在 `AuthorizeSiteAccess.call/2` 中实现了权限的优先级判定逻辑，优先级从高到低如下：

```
团队成员角色 > 超级管理员 > 公开站点 > 共享链接
```

**核心代码位置**：`lib/plausible_web/plugs/authorize_site_access.ex:86-102`

```elixir
role =
  cond do
    membership_role ->
      membership_role  # 第1优先级：团队成员角色

    Plausible.Auth.is_super_admin?(current_user) ->
      :super_admin     # 第2优先级：超级管理员

    site.public ->
      :public          # 第3优先级：公开站点

    shared_link ->
      :public          # 第4优先级：共享链接

    true ->
      nil              # 无权限
  end
```

### 1.2 各机制详细说明

#### (1) 团队成员角色（Team Membership Roles）

**来源**：`Teams.Memberships.site_role/2` 函数

**角色层级**（从高到低）：

| 角色 | 权限范围 | 说明 |
|------|---------|------|
| `:owner` | 最高权限 | 团队所有者，可管理所有成员和设置 |
| `:admin` | 高权限 | 管理员，可邀请成员、修改设置 |
| `:editor` | 编辑权限 | 可创建目标、漏斗等 |
| `:viewer` | 只读权限 | 仅可查看统计数据 |
| `:guest` | 访客权限 | 只能访问被明确授权的站点 |

**关键实现**：`lib/plausible/teams/memberships.ex:80-101`

```elixir
def site_role(site, user) do
  result =
    from(u in Auth.User,
      inner_join: tm in assoc(u, :team_memberships),
      left_join: gm in assoc(tm, :guest_memberships),
      where: tm.team_id == ^site.team_id and tm.user_id == ^user.id,
      where: tm.role != :guest or gm.site_id == ^site.id,
      select: {tm.role, gm.role}
    )
    |> Repo.one()

  case result do
    {:guest, role} -> {:ok, {:guest_member, role}}
    {role, _} -> {:ok, {:team_member, role}}
    _ -> {:error, :not_a_member}
  end
end
```

**特殊说明**：
- `:guest` 角色需要通过 `GuestMembership` 关联到特定站点才能访问
- 返回值包含 `member_type`（`:team_member` 或 `:guest_member`）用于区分成员类型

#### (2) 超级管理员（Super Admin）

**判定方式**：`Plausible.Auth.is_super_admin?(current_user)`

**权限特点**：
- 系统级别的最高权限
- 可访问所有站点
- 不受团队角色限制

#### (3) 公开站点（Public Site）

**触发条件**：`site.public == true`

**权限特点**：
- 站点级别的公开设置
- 任何访问者都以 `:public` 角色访问
- 无需登录即可查看统计数据

#### (4) 共享链接（Shared Link）

**触发条件**：URL 中包含有效的 `slug` 或 `auth` 参数

**权限特点**：
- 通过独立的 `shared_links` 表管理
- 访问者以 `:public` 角色访问
- 可设置密码保护
- 可限制到特定 segment（数据片段）

---

## 二、公开报表无登录复用查询能力的实现

### 2.1 核心设计思想

**同一套查询逻辑，不同的权限入口**

所有统计数据查询都通过 `Plausible.Stats` 模块实现，无论用户是否登录，查询逻辑完全相同。区别仅在于：
1. 如何获取 `site` 对象
2. 如何进行权限验证

### 2.2 实现架构

#### (1) Dashboard 页面渲染流程

**入口**：`StatsController.stats/2`

```
用户访问 /:domain
    ↓
AuthPlug 检查登录状态（可选）
    ↓
AuthorizeSiteAccess 验证权限
    ↓
获取 site 并赋值到 conn.assigns
    ↓
渲染 stats.html，传递 site_role
    ↓
前端 React 应用加载
    ↓
前端调用 API 获取数据
```

#### (2) API 数据获取流程

**入口**：`Api.StatsController` 的各个 action

```
前端调用 /api/stats/:domain/xxx
    ↓
AuthorizeSiteAccess 再次验证权限
    ↓
从 conn.assigns[:site] 获取站点
    ↓
调用 Plausible.Stats.breakdown/query 等函数
    ↓
返回 JSON 数据
```

### 2.3 关键代码分析

#### (1) 权限验证后的站点赋值

**位置**：`lib/plausible_web/plugs/authorize_site_access.ex:121`

```elixir
conn = merge_assigns(conn, site: site, site_role: role, shared_link: shared_link)
```

无论权限来源是什么（团队成员、公开站点、共享链接），最终都会将：
- `:site` - 站点对象
- `:site_role` - 访问者角色
- `:shared_link` - 共享链接对象（如果通过共享链接访问）

赋值到 `conn.assigns`。

#### (2) API 控制器使用相同的站点

**位置**：`lib/plausible_web/controllers/api/stats_controller.ex:40-56`

```elixir
def query(conn, params) do
  site = conn.assigns.site  # 从 assigns 获取，与权限来源无关
  now = conn.private[:now]

  with {:ok, %ParsedQueryParams{} = params} <- Dashboard.QueryParser.parse(params, now: now),
       {:ok, %Query{} = query} <- QueryBuilder.build(site, params, debug_metadata(conn)) do
    json(conn, Plausible.Stats.query(site, query))  # 同一套查询逻辑
  else
    {:error, %QueryError{message: message}} -> bad_request(conn, message)
  end
end
```

#### (3) 共享链接访问时的特殊处理

**位置**：`lib/plausible_web/controllers/stats_controller.ex:441-538`

```elixir
defp render_shared_link(conn, shared_link) do
  # ... 检查站点锁定、功能可用性等
  
  site_role = get_fallback_site_role(conn)  # 返回 :public
  
  # ... 渲染 stats.html，参数与普通访问几乎相同
  render("stats.html",
    site: shared_link.site,
    site_role: site_role,  # 传递 :public 角色
    shared_link_auth: shared_link.slug,  # 额外传递共享链接标识
    # ... 其他参数
  )
end
```

### 2.4 前端如何区分不同访问方式

前端通过 `site_role` 和 `shared_link_auth` 来区分：

| 场景 | site_role | shared_link_auth | 前端行为 |
|------|-----------|-----------------|---------|
| 团队成员登录 | `:owner/:admin/:editor/:viewer` | `nil` | 显示完整功能 |
| 超级管理员 | `:super_admin` | `nil` | 显示完整功能 + 管理功能 |
| 公开站点 | `:public` | `nil` | 只读模式，隐藏管理功能 |
| 共享链接 | `:public` | 有值（slug） | 只读模式，可能限制 segment |

---

## 三、前后端权限数据来源一致性

### 3.1 服务端权限数据来源

服务端有两个主要的权限数据获取入口：

#### (1) 团队成员角色获取

**来源**：`Teams.Memberships.site_role/2`

**数据来源**：直接查询数据库

```elixir
def site_role(site, user) do
  from(u in Auth.User,
    inner_join: tm in assoc(u, :team_memberships),
    left_join: gm in assoc(tm, :guest_memberships),
    where: tm.team_id == ^site.team_id and tm.user_id == ^user.id,
    select: {tm.role, gm.role}
  )
  |> Repo.one()
end
```

#### (2) 当前团队角色获取

**来源**：`AuthPlug.call/2`

**数据来源**：从 `user.team_memberships` 关联获取（预加载）

```elixir
def call(conn, _opts) do
  case UserAuth.get_user_session(conn) do
    {:ok, user_session} ->
      user = user_session.user
      
      # 从 user.team_memberships 中查找当前团队
      team_membership =
        Enum.find(user.team_memberships, %{}, &(&1.team.identifier == current_team_id))
      
      # 赋值到 conn.assigns
      conn
      |> assign(:current_team, Map.get(team_membership, :team))
      |> assign(:current_team_role, Map.get(team_membership, :role))
      # ...
  end
end
```

### 3.2 前端权限数据来源

前端获取权限数据有两个途径：

#### (1) 页面渲染时注入

**位置**：`StatsController.stats/2` 和 `render_shared_link/2`

渲染 `stats.html` 时传递的参数：

```elixir
render("stats.html",
  site: site,
  site_role: site_role,  # 关键：访问者角色
  # ...
)
```

#### (2) API 响应中的间接信息

API 本身不直接返回权限数据，但：
- 某些端点（如 `referrer_drilldown` 中的 Google Search Console）会检查 `has_editor_access?`
- 功能开关通过 `flags` 参数传递

### 3.3 前后端数据一致性分析

**结论：前后端权限数据来源是同一套数据库，但获取时机和方式不同**

| 维度 | 服务端（AuthorizeSiteAccess） | 服务端（AuthPlug） | 前端 |
|------|------------------------------|-------------------|------|
| 数据来源 | 直接查询 `team_memberships` 表 | 从 `user.team_memberships` 关联获取 | 页面渲染时注入 |
| 角色类型 | `site_role`（站点级别） | `current_team_role`（团队级别） | `site_role`（站点级别） |
| 更新时机 | 每次请求实时查询 | 登录时/切换团队时预加载 | 页面刷新时 |

**潜在不一致场景**：

1. **角色变更后的缓存问题**：
   - `AuthPlug` 使用的是 `user.team_memberships` 关联数据（可能是 Ecto 预加载的）
   - `AuthorizeSiteAccess` 每次请求都会重新查询数据库
   - 如果角色在会话期间变更，`AuthPlug` 的数据可能滞后

2. **团队角色 vs 站点角色**：
   - `current_team_role` 是团队级别的角色
   - `site_role` 是站点级别的角色（考虑 guest membership）
   - 对于 guest 成员，这两个值可能不同

---

## 四、成员角色变更对共享链接的影响

### 4.1 共享链接的独立权限模型

**关键发现**：共享链接是**独立的权限授予机制**，不依赖于创建者的当前角色。

#### (1) 共享链接的数据模型

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
- 没有 `creator_id` 或 `created_by` 字段
- 只关联到 `site_id`
- 权限完全由记录是否存在决定

#### (2) 共享链接的验证逻辑

**位置**：`lib/plausible_web/plugs/authorize_site_access.ex:200-220`

```elixir
defp maybe_get_shared_link(conn, site) do
  slug = conn.path_params["slug"] || conn.params["auth"]

  if valid_path_fragment?(slug) do
    with %Plausible.Site.SharedLink{} = shared_link <-
           Repo.get_by(Plausible.Site.SharedLink, slug: slug, site_id: site.id),
         {%{password_protected?: true}, shared_link} <-
           {%{password_protected?: Plausible.Site.SharedLink.password_protected?(shared_link)},
            shared_link},
         {:ok, shared_link} <-
           PlausibleWeb.StatsController.validate_shared_link_password(conn, shared_link) do
      {:ok, shared_link}
    else
      {%{password_protected?: false}, shared_link} -> {:ok, shared_link}
      {:error, :unauthorized} -> error_not_found(conn)
      nil -> error_not_found(conn)
    end
  else
    {:ok, nil}
  end
end
```

**验证步骤**：
1. 从 URL 获取 `slug`
2. 查询 `shared_links` 表，检查是否存在匹配的记录（`slug` + `site_id`）
3. 如果有密码保护，验证密码 cookie
4. 返回共享链接对象或错误

**关键点**：**没有检查创建者的角色！**

#### (3) 共享链接页面的渲染逻辑

**位置**：`lib/plausible_web/controllers/stats_controller.ex:260-296`

```elixir
def shared_link(conn, %{"domain" => domain, "auth" => auth}) do
  case find_shared_link(domain, auth) do
    {:ok, shared_link} ->
      if Plausible.Site.SharedLink.password_protected?(shared_link) do
        render_password_protected_shared_link(conn, shared_link)
      else
        render_shared_link(conn, shared_link)
      end

    {:error, :not_found} ->
      render_error(conn, 404)
  end
end
```

### 4.2 影响共享链接可用性的因素

以下因素会影响共享链接的可用性，**但不包括创建者的角色变更**：

| 因素 | 是否影响 | 说明 |
|------|---------|------|
| 共享链接记录被删除 | 是 | 管理员可以在设置中撤销共享链接 |
| 站点被锁定 | 是 | 团队订阅过期或欠费 |
| 共享链接功能不可用 | 是 | 需要 Business 计划 |
| 密码被修改 | 是 | 访问者需要重新输入新密码 |
| 创建者角色变更 | **否** | 共享链接独立于创建者权限 |
| 创建者被移除团队 | **否** | 共享链接仍然有效 |
| 站点不再公开 | **否** | 共享链接不受 site.public 影响 |

### 4.3 实际场景分析

#### 场景 1：管理员创建共享链接后被降级为查看者

**结果**：共享链接**仍然可用**

原因：
- 共享链接记录存在于数据库
- 验证逻辑不检查创建者角色
- 即使创建者变成 `:viewer`，链接仍然有效

#### 场景 2：管理员创建共享链接后被移出团队

**结果**：共享链接**仍然可用**

原因：
- 共享链接没有 `creator_id` 字段
- 验证逻辑只检查链接本身是否存在
- 创建者是否在团队中不影响链接有效性

#### 场景 3：成员被降级为 guest，其创建的共享链接

**结果**：共享链接**仍然可用**

原因：
- guest 角色只是限制该成员的访问
- 已创建的共享链接是独立的权限授予

#### 场景 4：管理员撤销共享链接

**结果**：共享链接**立即失效**

原因：
- 管理员在设置中删除共享链接记录
- `maybe_get_shared_link` 查询不到记录，返回 404

---

## 五、总结

### 5.1 权限叠加机制总结

```
访问请求
    ↓
┌─────────────────────────────────────────┐
│  1. 检查是否有团队成员角色（最高优先级）  │
│     - 有 → 使用成员角色                  │
│     - 无 → 继续检查                      │
├─────────────────────────────────────────┤
│  2. 检查是否是超级管理员                  │
│     - 是 → :super_admin                  │
│     - 否 → 继续检查                      │
├─────────────────────────────────────────┤
│  3. 检查站点是否公开                      │
│     - 是 → :public                       │
│     - 否 → 继续检查                      │
├─────────────────────────────────────────┤
│  4. 检查是否有有效的共享链接              │
│     - 有 → :public                       │
│     - 无 → 拒绝访问（404）               │
└─────────────────────────────────────────┘
```

### 5.2 公开报表无登录查询实现总结

**核心原理**：

1. **统一的查询入口**：所有统计查询都通过 `Plausible.Stats` 模块
2. **统一的站点上下文**：无论权限来源，`site` 都赋值到 `conn.assigns`
3. **不同的权限前置**：通过不同的 plug（`AuthorizeSiteAccess`）进行权限验证
4. **前端感知差异**：通过 `site_role` 和 `shared_link_auth` 区分访问模式

**好处**：
- 代码复用：一套查询逻辑支持所有访问模式
- 维护简单：修改查询逻辑只需改一处
- 功能一致：公开访问和登录访问看到相同的数据（除了权限限制的功能）

### 5.3 前后端权限数据一致性总结

| 问题 | 答案 |
|------|------|
| 数据来源是否同一套？ | 是，都来自 `team_memberships` 表 |
| 获取方式是否相同？ | 不同，服务端实时查询，前端通过页面注入 |
| 是否可能不一致？ | 理论上可能（角色变更后页面未刷新） |
| 如何保持一致？ | 页面刷新重新获取，敏感操作服务端二次验证 |

### 5.4 成员角色变更对共享链接影响总结

**核心结论**：成员角色变更**不会影响**已创建的共享链接。

**原因**：
1. 共享链接没有 `creator_id` 字段，不跟踪创建者
2. 验证逻辑只检查链接记录是否存在，不检查创建者权限
3. 共享链接是独立的权限授予，不是创建者权限的代理

**建议**：
- 如果需要撤销某成员创建的所有共享链接，需要：
  1. 识别该成员创建的链接（需要额外的审计日志或添加 creator_id 字段）
  2. 手动删除这些共享链接记录
- 当前实现中，管理员可以看到所有共享链接并随时撤销

---

## 六、关键代码位置索引

| 功能模块 | 文件路径 | 关键行号 |
|---------|---------|---------|
| 权限判定核心逻辑 | `lib/plausible_web/plugs/authorize_site_access.ex` | 78-140 |
| 团队角色获取 | `lib/plausible/teams/memberships.ex` | 80-101 |
| 共享链接验证 | `lib/plausible_web/plugs/authorize_site_access.ex` | 200-220 |
| 共享链接模型 | `lib/plausible/site/shared_link.ex` | 1-58 |
| 登录用户权限填充 | `lib/plausible_web/plugs/auth_plug.ex` | 1-94 |
| Dashboard 页面渲染 | `lib/plausible_web/controllers/stats_controller.ex` | 51-114, 441-538 |
| 统计 API 端点 | `lib/plausible_web/controllers/api/stats_controller.ex` | 40-56 |
| 公开 API 权限验证 | `lib/plausible_web/plugs/authorize_public_api.ex` | 50-131 |
