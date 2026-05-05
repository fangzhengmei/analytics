# Plausible Stats API 两类客户端鉴权机制对比分析

## 概述

Plausible 存在两类客户端访问统计查询层：
1. **第三方 Stats API** - 使用 Bearer Token 鉴权
2. **Dashboard** - 使用 Session Cookie 鉴权

本文档详细分析两类客户端在入口收敛、限速实现、审计区分三个维度的差异，特别关注：
- 真正会写审计日志的入口
- 两类客户端在审计日志中的字段落库差异
- 无审计写入时的客户端身份判别方式
- Dashboard 统计查询的完整鉴权链路

---

## 一、入口收敛机制

### 1.1 架构概览

```
                    ┌─────────────────────────────────────────┐
                    │           Plausible.Stats               │
                    │  (统一查询层 - query/breakdown/aggregate) │
                    └─────────────────────────────────────────┘
                              ▲                  ▲
                              │                  │
                    ┌─────────┴──────┐   ┌─────┴──────────┐
                    │  Api.Stats      │   │ ExternalStats   │
                    │  Controller    │   │ Controller      │
                    │  (Dashboard)   │   │ (第三方API)    │
                    └────────────────┘   └─────────────────┘
                              ▲                  ▲
                              │                  │
                    ┌─────────┴──────┐   ┌─────┴──────────┐
                    │internal_stats_ │   │  public_api +   │
                    │     api        │   │AuthorizePublicAPI│
                    │  (Session)     │   │   (Token)       │
                    └────────────────┘   └─────────────────┘
```

### 1.2 第三方 Stats API (Token 鉴权)

#### 路由配置
```elixir
# 路由位置: lib/plausible_web/router.ex:337-358

# V1 API
scope "/api/v1/stats", PlausibleWeb.Api,
  assigns: %{api_scope: "stats:read:*", api_context: :site} do
  pipe_through [:public_api, PlausibleWeb.Plugs.AuthorizePublicAPI]

  get "/realtime/visitors", ExternalStatsController, :realtime_visitors
  get "/aggregate", ExternalStatsController, :aggregate
  get "/breakdown", ExternalStatsController, :breakdown
  get "/timeseries", ExternalStatsController, :timeseries
end

# V2 API (Query API)
scope "/api/v2", PlausibleWeb.Api,
  private: %{allow_consolidated_views: true},
  assigns: %{api_scope: "stats:read:*", api_context: :site} do
  pipe_through [:public_api, PlausibleWeb.Plugs.AuthorizePublicAPI]
  post "/query", ExternalQueryApiController, :query
end
```

#### Pipeline 组成
- `:public_api` - 仅设置 `accepts: ["json"]`
- `PlausibleWeb.Plugs.AuthorizePublicAPI` - 核心鉴权逻辑

#### 鉴权流程
1. **Token 提取**: 从 `Authorization: Bearer <token>` 头提取
   ```elixir
   # lib/plausible_web/plugs/authorize_public_api.ex:175-185
   defp get_bearer_token(conn) do
     authorization_header =
       conn
       |> Plug.Conn.get_req_header("authorization")
       |> List.first()
   
     case authorization_header do
       "Bearer " <> token -> {:ok, String.trim(token)}
       _ -> {:error, :missing_api_key}
     end
   end
   ```

2. **API Key 查找**: 根据 token 和 context 查找
   - `:site` context: 通过 `Auth.find_api_key_for_team_of_site(token, site_id)`
   - 其他 context: 通过 `Auth.find_api_key(token)`

3. **限速检查** (详见第二章):
   - 小时级限制: `check_api_key_rate_limit/2`
   - 突发限制: `check_api_key_burst_limit/1`

4. **Scope 验证**: 验证 API Key 的权限范围
   - 隐式 scopes: `["stats:read:*", "sites:read:*"]`
   - 前缀匹配机制: `some:*` 可匹配 `some:scope:*`

5. **站点访问验证**: 验证 API Key 对目标站点的访问权限

6. **Conn Assigns 设置**:
   ```elixir
   # lib/plausible_web/plugs/authorize_public_api.ex:60-61
   conn
   |> assign(:current_user, api_key.user)
   |> assign(:current_team, api_key.team)
   ```

#### 控制器层
- **ExternalStatsController** (`lib/plausible_web/controllers/api/external_stats_controller.ex`)
  - `realtime_visitors/2` → `Plausible.Stats.current_visitors/1`
  - `aggregate/2` → `Plausible.Stats.aggregate/3`
  - `breakdown/2` → `Plausible.Stats.breakdown/4`
  - `timeseries/2` → `Plausible.Stats.timeseries/3`

- **ExternalQueryApiController** (`lib/plausible_web/controllers/api/external_query_api_controller.ex`)
  - `query/2` → `Plausible.Stats.query/2`

---

### 1.3 Dashboard (Session 鉴权)

#### 路由配置
```elixir
# 路由位置: lib/plausible_web/router.ex:277-326

scope "/api" do
  pipe_through :internal_stats_api

  scope "/stats", PlausibleWeb.Api do
    # EE 特性
    on_ee do
      get "/:domain/funnels/:id", StatsController, :funnel
      post "/:domain/exploration/next", StatsController, :exploration_next
      # ...
    end

    scope private: %{allow_consolidated_views: true} do
      post "/:domain/query", StatsController, :query
      get "/:domain/current-visitors", StatsController, :current_visitors
      get "/:domain/sources", StatsController, :sources
      get "/:domain/channels", StatsController, :channels
      # ... 更多统计端点
    end
  end
end
```

#### 完整鉴权链路

**Pipeline 定义** (`lib/plausible_web/router.ex:72-78`):
```elixir
pipeline :internal_stats_api do
  plug :accepts, ["json"]                    # 1. 仅接受 JSON
  plug :fetch_session                        # 2. 从 Cookie 获取 session
  plug PlausibleWeb.AuthPlug                 # 3. Session 鉴权 + 设置审计上下文
  plug PlausibleWeb.Plugs.AuthorizeSiteAccess # 4. 站点访问授权
  plug PlausibleWeb.Plugs.NoRobots           # 5. 设置 X-Robots-Tag
end
```

**链路 1: `fetch_session`**
- 从 Cookie (`_plausible_analytics_key`) 中解析 session ID
- 从 ETS 或数据库获取 session 数据

**链路 2: `AuthPlug` (核心 Session 鉴权)**
```elixir
# lib/plausible_web/plugs/auth_plug.ex:18-93
def call(conn, _opts) do
  case UserAuth.get_user_session(conn) do
    {:ok, user_session} ->
      user = user_session.user

      # 2.1 获取当前团队
      current_team_id =
        conn.params["__team"] || 
        Plug.Conn.get_session(conn, "current_team_id") || 
        user.last_team_identifier

      {current_team, current_team_role} = ...

      # 2.2 设置 OpenTelemetry 和 Sentry 上下文
      Plausible.OpenTelemetry.add_user_attributes(user)
      Sentry.Context.set_user_context(%{id: user.id, ...})

      # 2.3 关键：设置审计上下文 (仅 EE 版本)
      on_ee do
        Plausible.Audit.set_context(%{
          current_user: user,
          current_team: current_team
        })
      end

      # 2.4 设置 Conn Assigns
      conn
      |> assign(:current_user, user)
      |> assign(:current_user_session, user_session)
      |> assign(:current_team, current_team || my_team)
      |> assign(:current_team_role, current_team_role)
      # ...

    {:error, :session_expired, user_session} ->
      assign(conn, :expired_session, user_session)

    _ ->
      conn
  end
end
```

**链路 3: `AuthorizeSiteAccess` (站点访问授权)**
```elixir
# lib/plausible_web/plugs/authorize_site_access.ex:78-140
def call(conn, {allowed_roles, site_param}) do
  current_user = conn.assigns[:current_user]

  with {:ok, domain} <- get_domain(conn, site_param),  # 3.1 从路径获取 domain
       {:ok, %{site: site, role: membership_role, member_type: member_type}} <-
         get_site_with_role(conn, current_user, domain),  # 3.2 查找站点
       :ok <- ensure_consolidated_view_access(conn, site),
       {:ok, shared_link} <- maybe_get_shared_link(conn, site) do

    # 3.3 确定用户角色
    role =
      cond do
        membership_role -> membership_role
        Plausible.Auth.is_super_admin?(current_user) -> :super_admin
        site.public -> :public
        shared_link -> :public
        true -> nil
      end

    if role in allowed_roles do
      # 3.4 设置站点相关 Assigns
      site = Repo.preload(site, [:owners, :completed_imports, ...])
      
      conn
      |> merge_assigns(site: site, site_role: role, shared_link: shared_link)
      |> set_current_team(site.team)  # 切换当前团队
    else
      error_not_found(conn)
    end
  end
end
```

**支持的角色** (`lib/plausible_web/plugs/authorize_site_access.ex:43`):
```elixir
@all_roles [:public, :viewer, :admin, :editor, :super_admin, :owner, :billing]
```

**链路 4: 控制器层**
- **Api.StatsController** (`lib/plausible_web/controllers/api/stats_controller.ex`)
  - `query/2` → `Plausible.Stats.query/2`
  - `sources/2` → `Plausible.Stats.breakdown/4`
  - 以及各种细分端点 (countries, cities, browsers 等)

---

### 1.4 统一查询层收敛

两类客户端最终都调用 **Plausible.Stats** 模块的同一组函数：

| 功能 | 第三方 API (ExternalStatsController) | Dashboard (Api.StatsController) | 统一查询层 |
|------|--------------------------------------|----------------------------------|------------|
| 实时访客 | `realtime_visitors/2` | `current_visitors/2` | `Stats.current_visitors/1` |
| 聚合查询 | `aggregate/2` | 各细分方法内部 | `Stats.aggregate/3` |
| 分组查询 | `breakdown/2` | `sources/2`, `pages/2` 等 | `Stats.breakdown/4` |
| 时间序列 | `timeseries/2` | 内部使用 | `Stats.timeseries/3` |
| 查询 API | `query/2` (V2) | `query/2` | `Stats.query/2` |

**Plausible.Stats 模块定义**:
```elixir
# lib/plausible/stats.ex:1-47
defmodule Plausible.Stats do
  def query(site, query), do: QueryRunner.run(site, query)
  def breakdown(site, query, metrics, pagination), do: Breakdown.breakdown(...)
  def aggregate(site, query, metrics), do: Aggregate.aggregate(...)
  def timeseries(site, query, metrics), do: Timeseries.timeseries(...)
  def current_visitors(site, duration \\ ...), do: CurrentVisitors.current_visitors(...)
  # ...
end
```

---

## 二、限速实现机制

### 2.1 限速策略对比

| 维度 | 第三方 Stats API (Token) | Dashboard (Session) |
|------|--------------------------|---------------------|
| 限速级别 | API Key / Team 级别 | **无显式限速** |
| 限速类型 | 两级限速 (小时级 + 突发) | 依赖授权机制 |
| 实现位置 | `AuthorizePublicAPI` plug | 无专门实现 |
| 可配置性 | 按 Team 配置 | 不可配置 |

### 2.2 第三方 API 限速详解

#### 限速 Plug 位置
```elixir
# lib/plausible_web/plugs/authorize_public_api.ex:50-64
def call(conn, _opts) do
  requested_scope = Map.fetch!(conn.assigns, :api_scope)
  context = conn.assigns[:api_context]

  with {:ok, token} <- get_bearer_token(conn),
       {:ok, api_key, limit_key, hourly_limit} <- find_api_key(conn, token, context),
       :ok <- check_api_key_rate_limit(limit_key, hourly_limit),      # 1. 小时级限速
       :ok <- check_api_key_burst_limit(limit_key),                    # 2. 突发限速
       {:ok, conn} <- verify_by_scope(conn, api_key, requested_scope) do
    # ...
  end
end
```

#### 第一级：小时级请求限制
```elixir
# lib/plausible_web/plugs/authorize_public_api.ex:187-196
defp check_api_key_rate_limit(limit_key, hourly_limit) do
  case RateLimit.check_rate(limit_key, to_timeout(hour: 1), hourly_limit) do
    {:allow, _} ->
      :ok

    {:deny, _} ->
      {:error, :rate_limit,
       "Too many API requests. The limit is #{hourly_limit} per hour. Please contact us to request more capacity."}
  end
end
```

**参数来源**:
- `limit_key`: 
  - 新系统: `Auth.ApiKey.limit_key(team)` (基于 Team)
  - 旧系统: `Auth.ApiKey.legacy_limit_key(user)` (基于 User)
- `hourly_limit`:
  - 新系统: `team.hourly_api_request_limit` (从数据库读取)
  - 旧系统: `Auth.ApiKey.legacy_hourly_request_limit()` (硬编码默认值)

#### 第二级：突发请求限制
```elixir
# lib/plausible_web/plugs/authorize_public_api.ex:198-214
defp check_api_key_burst_limit(limit_key) do
  burst_period_seconds = Auth.ApiKey.burst_period_seconds()
  burst_request_limit = Auth.ApiKey.burst_request_limit()

  case RateLimit.check_rate(
         limit_key,
         to_timeout(second: burst_period_seconds),
         burst_request_limit
       ) do
    {:allow, _} ->
      :ok

    {:deny, _} ->
      {:error, :rate_limit,
       "Too many API requests in a short period of time. The limit is #{burst_request_limit} per #{burst_period_seconds} seconds. Please throttle your requests."}
  end
end
```

**参数来源**:
- `burst_period_seconds`: 从配置获取 (默认较短的时间窗口)
- `burst_request_limit`: 从配置获取 (突发请求数限制)

#### 底层限速实现
```elixir
# lib/plausible/rate_limit.ex:1-98
defmodule Plausible.RateLimit do
  @moduledoc """
  Thin wrapper around `:ets.update_counter/4` and a
  clean-up process to act as a rate limiter.
  """

  # 使用 GenServer 管理 ETS 表和清理进程
  use GenServer

  # 核心限速检查函数
  @spec check_rate(:ets.table(), key, scale, limit, increment) :: {:allow, count} | {:deny, limit}
  def check_rate(table \\ __MODULE__, key, scale, limit, increment \\ 1) do
    bucket = div(now(), scale)          # 时间分桶
    full_key = {key, bucket}            # 完整的 ETS 键
    expires_at = (bucket + 1) * scale   # 过期时间

    # 原子计数器操作
    count =
      case :ets.lookup(table, full_key) do
        [{_, counter, _expires_at}] ->
          :atomics.add_get(counter, 1, increment)

        [] ->
          counter = :atomics.new(1, signed: false)
          case :ets.insert_new(table, {full_key, counter, expires_at}) do
            true -> :atomics.add_get(counter, 1, increment)
            false -> 
              [{_, counter, _}] = :ets.lookup(table, full_key)
              :atomics.add_get(counter, 1, increment)
          end
      end

    if count <= limit, do: {:allow, count}, else: {:deny, limit}
  end

  # 定期清理过期的计数器
  defp clean(table) do
    ms = [{{{:_, :_}, :_, :"$1"}, [], [{:<, :"$1", {:const, now()}}]}]
    :ets.select_delete(table, ms)
  end
end
```

**技术要点**:
1. **分桶机制**: 使用时间分桶 (`bucket = div(now(), scale)`)，避免滑动窗口的复杂度
2. **原子计数器**: 使用 `:atomics` 实现高效的并发安全计数
3. **ETS 表**: 支持高并发读写 (`:public`, `:read_concurrency`, `:write_concurrency`)
4. **定期清理**: GenServer 定期删除过期的分桶数据

---

### 2.3 Dashboard 无显式限速的原因

1. **授权作为天然限制**: Dashboard 需要有效的用户会话和站点权限，天然过滤非法访问
2. **用户交互模式**: Dashboard 是用户交互界面，请求频率自然受限于用户操作
3. **资源保护**: 通过其他机制保护 (如数据库连接池、查询超时等)

---

## 三、审计区分机制

### 3.1 关键认知澄清

在深入分析之前，需要明确一个重要事实：

**统计查询本身不会写入审计日志！**

审计日志仅在执行 `*_with_audit` 系列方法时才会写入，这些方法主要用于：
- 用户数据变更 (insert/update/delete)
- 配置变更
- SSO 相关操作

统计查询是只读操作，不会触发审计日志写入。

---

### 3.2 真正会写审计日志的入口

#### 审计写入机制

```elixir
# lib/plausible/audit/repo.ex:41-105
defmacro __using__(_opts) do
  on_ee do
    quote do
      @behaviour Plausible.Audit.Repo

      # 更新操作带审计
      def update_with_audit(%Ecto.Changeset{} = changeset, entry_name, params \\ %{}) do
        case update(changeset) do
          {:ok, result} ->
            store_audit(entry_name, result, changeset, params)
            {:ok, result}
          other -> other
        end
      end

      # 插入操作带审计
      def insert_with_audit(%Ecto.Changeset{} = changeset, entry_name, params \\ %{}) do
        case insert(changeset) do
          {:ok, result} ->
            store_audit(entry_name, result, params)
            {:ok, result}
          other -> other
        end
      end

      # 删除操作带审计
      def delete_with_audit!(resource, entry_name, params \\ %{}) do
        result = delete!(resource)
        store_audit(entry_name, resource, params)
        result
      end

      # 核心审计存储逻辑
      defp store_audit(entry_name, result, changeset, params) do
        entry_name
        |> Plausible.Audit.Entry.new(result, params)      # 1. 创建审计条目
        |> Plausible.Audit.Entry.include_change(changeset)  # 2. 包含变更内容
        |> Plausible.Audit.Entry.persist!()               # 3. 写入数据库
      end

      defp store_audit(entry_name, result, params) do
        entry_name
        |> Plausible.Audit.Entry.new(result, params)
        |> Plausible.Audit.Entry.include_change(result)
        |> Plausible.Audit.Entry.persist!()
      end
    end
  end
end
```

#### 实际写入审计日志的操作

从代码搜索结果看，当前使用 `*_with_audit` 方法的场景：

| 模块 | 操作 | 审计条目名称 |
|------|------|-------------|
| `Plausible.Teams` | 团队策略更新 | 动态 (audit_entry_name) |
| `Plausible.Auth.SSO` | SSO 用户预配 | `"sso_user_provisioned"` |
| `Plausible.Auth.SSO` | SSO 用户取消预配 | `"sso_user_deprovioned"` |
| `Plausible.Auth.SSO` | SSO 集成更新 | `"sso_integration_updated"` |
| `Plausible.Auth.SSO` | SSO 策略更新 | `"sso_policy_updated"` |
| `Plausible.Auth.SSO` | SSO 强制模式变更 | `"sso_force_mode_changed"` |
| `Plausible.Auth.SSO` | SSO 集成移除 | `"sso_integration_removed"` |
| `Plausible.Auth.SSO.Domains` | SSO 域名添加 | `"sso_domain_added"` |
| `Plausible.Auth.SSO.Domains` | SSO 域名验证成功 | `"sso_domain_verification_success"` |
| `Plausible.Auth.SSO.Domains` | SSO 域名验证失败 | `"sso_domain_verification_failure"` |
| `Plausible.Auth.SSO.Domains` | SSO 域名验证取消 | `"sso_domain_verification_cancelled"` |
| `Plausible.Auth.SSO.Domains` | SSO 域名移除 | `"sso_domain_removed"` |
| `Plausible.Auth.SSO.RealSamlAdapter` | SSO Identity 创建 | 动态 |
| `SettingsController` | 用户更新 | `"user_update"` |
| `SettingsController` | 用户创建 | `"user_insert"` |
| `SettingsController` | 用户删除 | `"user_delete"` |

**注意**：统计查询控制器 (`Api.StatsController`, `ExternalStatsController`, `ExternalQueryApiController`) 中**没有任何** `*_with_audit` 调用。

---

### 3.3 审计上下文设置机制

审计上下文决定了写入的审计条目中的 `actor_type`、`user_id`、`team_id` 字段。

#### 审计上下文存储位置

```elixir
# extra/lib/plausible/audit/entry.ex:81-91
defp get_context() do
  case :logger.get_process_metadata() do
    %{:__audit__ => audit_context} -> audit_context
    %{} -> %{}
    :undefined -> %{}
  end
end

def set_context(kv) when is_map(kv) do
  :logger.update_process_metadata(%{:__audit__ => kv})
end
```

**关键点**：审计上下文存储在**当前进程的元数据** (`:logger.process_metadata`) 中，而不是 `conn.assigns` 中。

#### Dashboard 请求的审计上下文设置

```elixir
# lib/plausible_web/plugs/auth_plug.ex:70-75 (仅 EE 版本)
on_ee do
  Plausible.Audit.set_context(%{
    current_user: user,
    current_team: current_team
  })
end
```

**设置时机**：在 `AuthPlug.call/2` 中，当 session 验证成功后设置。

#### LiveView 的审计上下文设置

```elixir
# extra/lib/plausible/audit/live_context.ex:1-22
defmodule Plausible.Audit.LiveContext do
  defmacro __using__(_) do
    quote do
      on_mount Plausible.Audit.LiveContext
    end
  end

  def on_mount(:default, _params, _session, socket) do
    if Phoenix.LiveView.connected?(socket) do
      Plausible.Audit.set_context(%{
        current_user: socket.assigns[:current_user],
        current_team: socket.assigns[:current_team]
      })
    end

    {:cont, socket}
  end
end
```

#### 第三方 API 请求的审计上下文设置

**关键发现**：`AuthorizePublicAPI` plug **没有设置审计上下文**！

```elixir
# lib/plausible_web/plugs/authorize_public_api.ex:60-61
# 虽然设置了 conn.assigns，但没有设置审计上下文
conn
|> assign(:current_user, api_key.user)    # 这是 Plug.Conn 的 assigns
|> assign(:current_team, api_key.team)     # 不是 :logger.process_metadata
```

**差异总结**：
| 上下文类型 | Dashboard (Session) | 第三方 API (Token) |
|-----------|---------------------|---------------------|
| `conn.assigns.current_user` | ✅ 有 | ✅ 有 |
| `conn.assigns.current_team` | ✅ 有 | ✅ 有 |
| `:logger.process_metadata[:__audit__]` | ✅ 有 (通过 `AuthPlug`) | ❌ 无 |

---

### 3.4 两类客户端在审计日志中的字段落库

当有审计日志写入时，字段值如下：

#### 审计条目 Schema

```elixir
# extra/lib/plausible/audit/entry.ex:20-32
schema "audit_entries" do
  field :name, :string
  field :entity, :string
  field :entity_id, :string
  field :meta, :map
  field :change, :map, default: %{}
  field :user_id, :integer, default: 0      # 默认值 0
  field :team_id, :integer, default: 0      # 默认值 0
  field :datetime, :naive_datetime_usec
  field :actor_type, Ecto.Enum, default: :system, values: [:system, :user]  # 默认 :system
end
```

#### 字段赋值逻辑

```elixir
# extra/lib/plausible/audit/entry.ex:34-51
def changeset(name, params) do
  context = get_context()  # 从 :logger.process_metadata 获取

  params =
    Map.merge(
      %{
        team_id: context[:current_team] && context.current_team.id,
        user_id: context[:current_user] && context.current_user.id,
        actor_type: if(context[:current_user], do: "user", else: "system")  # 关键区分点
      },
      params
    )

  %__MODULE__{name: name}
  |> cast(params, [:entity, :entity_id, :meta, :user_id, :team_id, :actor_type])
  |> validate_required([:name, :entity, :entity_id, :actor_type])
  |> put_change(:datetime, NaiveDateTime.utc_now())
end
```

#### 两类客户端的字段对比

| 字段 | Dashboard (Session 鉴权) | 第三方 API (Token 鉴权) |
|------|--------------------------|-------------------------|
| `actor_type` | `"user"` (因为 `context[:current_user]` 存在) | `"system"` (默认值，因为 context 为空) |
| `user_id` | 实际用户 ID (从 `context[:current_user].id` 获取) | `nil` 或 `0` (默认值) |
| `team_id` | 实际团队 ID (从 `context[:current_team].id` 获取) | `nil` 或 `0` (默认值) |
| `datetime` | 当前时间 | 当前时间 |

**测试验证** (`test/plausible/audit_test.exs:149-156`):
```elixir
test "changeset/2 with missing context" do
  cs = Entry.changeset("test", %{entity: "E", entity_id: "1"})
  assert is_nil(cs.changes.user_id)    # 没有变更
  assert is_nil(cs.changes.team_id)    # 没有变更
  assert cs.data.user_id == 0           # 使用默认值 0
  assert cs.data.team_id == 0           # 使用默认值 0
  assert cs.data.actor_type == :system  # 使用默认值 :system
end
```

---

### 3.5 无审计写入时的客户端身份判别

由于统计查询本身不写入审计日志，需要通过其他方式区分两类客户端。

#### 方式 1: `debug_metadata` (推荐)

两类控制器都会调用 `debug_metadata(conn)` 来收集请求元数据，传递给查询层用于调试和追踪。

```elixir
# lib/plausible_web/controllers/helpers.ex:24-51
def debug_metadata(conn) do
  %{
    request_method: conn.method,
    request_path: conn.request_path,
    params: conn.params,
    phoenix_controller: conn.private.phoenix_controller |> to_string(),  # 关键区分点
    phoenix_action: conn.private.phoenix_action |> to_string(),
    site_id: conn.assigns.site.id,
    site_domain: conn.assigns.site.domain,
    user_id: get_user_id(conn, conn.assigns),
    team_id: get_team_id(conn, conn.assigns)
  }
end

defp get_user_id(_conn, %{current_user: user}), do: user.id

defp get_user_id(conn, _assigns) do
  case PlausibleWeb.UserAuth.get_user_session(conn) do
    {:ok, user_session} -> user_session.user_id
    _ -> nil
  end
end

defp get_team_id(_conn, %{current_team: %Plausible.Teams.Team{id: id}}), do: id
defp get_team_id(_conn, %{site_team: %Plausible.Teams.Team{id: id}}), do: id
defp get_team_id(_conn, %{api_context: :site, site: %Plausible.Site{team_id: id}}), do: id
defp get_team_id(_conn, _assigns), do: nil
```

**通过 `phoenix_controller` 区分**：

| 客户端类型 | `phoenix_controller` 值 |
|-----------|------------------------|
| Dashboard | `"Elixir.PlausibleWeb.Api.StatsController"` |
| 第三方 API (V1) | `"Elixir.PlausibleWeb.Api.ExternalStatsController"` |
| 第三方 API (V2) | `"Elixir.PlausibleWeb.Api.ExternalQueryApiController"` |

**注意**：
- `user_id` 对两类客户端都有值：
  - Dashboard: 从 session 获取
  - 第三方 API: 从 `conn.assigns.current_user` 获取 (`AuthorizePublicAPI` 设置)
- 不能仅通过 `user_id` 区分，需要结合 `phoenix_controller` 或 `request_path`

#### 方式 2: 请求路径

| 客户端类型 | URL 模式 | 示例 |
|-----------|---------|------|
| Dashboard | `/api/stats/:domain/*` | `/api/stats/example.com/sources` |
| 第三方 API (V1) | `/api/v1/stats/*` | `/api/v1/stats/aggregate?site_id=example.com` |
| 第三方 API (V2) | `/api/v2/query` | `/api/v2/query` |

**路径特征**：
- Dashboard: 路径中包含 `:domain` 参数 (如 `example.com`)
- 第三方 API V1: 路径前缀 `/api/v1/stats`，`site_id` 作为查询参数
- 第三方 API V2: 固定路径 `/api/v2/query`

#### 方式 3: Pipeline 和 Conn Assigns

**Pipeline 差异**：

| Pipeline | 包含的 Plug | 客户端类型 |
|----------|-------------|-----------|
| `:internal_stats_api` | `fetch_session`, `AuthPlug`, `AuthorizeSiteAccess` | Dashboard |
| `:public_api` | 仅 `accepts: ["json"]` | 第三方 API |

**Conn Assigns 差异**：

| Assign Key | Dashboard | 第三方 API |
|------------|-----------|------------|
| `:site_role` | ✅ 有 (角色信息: `:admin`, `:viewer` 等) | ❌ 无 |
| `:shared_link` | ✅ 可能有 (共享链接对象) | ❌ 无 |
| `:site_team` | ✅ 可能有 | ❌ 无 |
| `:current_user_session` | ✅ 有 (session 对象) | ❌ 无 |
| `:api_scope` | ❌ 无 | ✅ 有 (如 `"stats:read:*"`) |
| `:api_context` | ❌ 无 | ✅ 有 (如 `:site`) |

**`AuthorizeSiteAccess` 设置的 Assigns** (`lib/plausible_web/plugs/authorize_site_access.ex:121`):
```elixir
conn = merge_assigns(conn, site: site, site_role: role, shared_link: shared_link)
```

#### 方式 4: Authorization 头

| 客户端类型 | Authorization 头 | Cookie |
|-----------|-----------------|--------|
| Dashboard | 无 (或 CSRF token) | 有 (session cookie: `_plausible_analytics_key`) |
| 第三方 API | `Bearer <token>` | 无 |

**第三方 API Token 提取** (`lib/plausible_web/plugs/authorize_public_api.ex:175-185`):
```elixir
defp get_bearer_token(conn) do
  authorization_header =
    conn
    |> Plug.Conn.get_req_header("authorization")
    |> List.first()

  case authorization_header do
    "Bearer " <> token -> {:ok, String.trim(token)}
    _ -> {:error, :missing_api_key}
  end
end
```

#### 方式 5: 限速检查

第三方 API 请求会经过显式的限速检查，Dashboard 请求不会。

**限速检查位置** (`lib/plausible_web/plugs/authorize_public_api.ex:56-57`):
```elixir
:ok <- check_api_key_rate_limit(limit_key, hourly_limit),
:ok <- check_api_key_burst_limit(limit_key),
```

如果在请求处理过程中观察到 `RateLimit.check_rate/4` 调用，说明是第三方 API 请求。

---

### 3.6 判别策略总结

**推荐的判别优先级**：

1. **首选**: `phoenix_controller` (通过 `debug_metadata` 或 `conn.private.phoenix_controller`)
   - 最准确，直接标识控制器模块

2. **次选**: 请求路径
   - 容易获取，模式清晰
   - 注意: Dashboard 路径包含域名，第三方 API 使用查询参数

3. **辅助**: Conn Assigns
   - `:site_role` 存在 → Dashboard
   - `:api_scope` 存在 → 第三方 API

4. **特殊场景**: Authorization 头
   - 存在 `Bearer` token → 第三方 API
   - 注意: 某些内部 API 也可能使用 token

**注意事项**：
- 不能仅通过 `user_id` 区分，因为两类请求都会设置 `conn.assigns.current_user`
- 审计上下文 (`:logger.process_metadata`) 仅在有审计日志写入时有用
- 统计查询本身不写审计日志，所以 `actor_type` 字段无法用于统计查询的判别

---

## 四、完整对比表

### 4.1 鉴权与入口对比

| 维度 | 第三方 Stats API (Token) | Dashboard (Session) |
|------|--------------------------|---------------------|
| **鉴权方式** | Bearer Token (Authorization 头) | Session Cookie |
| **核心 Plug** | `AuthorizePublicAPI` | `AuthPlug` + `AuthorizeSiteAccess` |
| **路由前缀** | `/api/v1/stats`, `/api/v2` | `/api/stats/:domain` |
| **Pipeline** | `:public_api` | `:internal_stats_api` |
| **控制器** | `ExternalStatsController`, `ExternalQueryApiController` | `Api.StatsController` |
| **统一查询层** | ✅ `Plausible.Stats.*` | ✅ `Plausible.Stats.*` |

### 4.2 限速对比

| 维度 | 第三方 Stats API (Token) | Dashboard (Session) |
|------|--------------------------|---------------------|
| 小时级限速 | ✅ 有 (按 Team/User) | ❌ 无 |
| 突发限速 | ✅ 有 | ❌ 无 |
| 限速实现位置 | `AuthorizePublicAPI.call/2` | 无 |
| 限速存储 | ETS + `:atomics` | 无 |

### 4.3 审计对比

| 维度 | 第三方 Stats API (Token) | Dashboard (Session) |
|------|--------------------------|---------------------|
| **统计查询写审计日志** | ❌ 否 | ❌ 否 |
| **审计上下文设置** | ❌ 无 | ✅ `AuthPlug` 中设置 |
| **上下文存储位置** | - | `:logger.process_metadata` |
| **有审计日志时的字段** | | |
| `actor_type` | `"system"` (默认) | `"user"` |
| `user_id` | `nil` 或 `0` | 实际用户 ID |
| `team_id` | `nil` 或 `0` | 实际团队 ID |

### 4.4 无审计日志时的判别方式

| 判别方式 | 第三方 API 特征 | Dashboard 特征 |
|----------|-----------------|---------------|
| `phoenix_controller` | `ExternalStatsController`, `ExternalQueryApiController` | `Api.StatsController` |
| 请求路径 | `/api/v1/stats/*`, `/api/v2/query` | `/api/stats/:domain/*` |
| `:api_scope` assign | ✅ 有 | ❌ 无 |
| `:site_role` assign | ❌ 无 | ✅ 有 |
| Authorization 头 | `Bearer <token>` | 无 |
| Cookie | 无 | `_plausible_analytics_key` |
| 限速检查 | ✅ 经过 `RateLimit.check_rate` | ❌ 无 |

---

## 五、关键代码位置索引

### 5.1 鉴权相关

| 功能 | 文件路径 | 行号 |
|------|---------|------|
| Token 鉴权 Plug | `lib/plausible_web/plugs/authorize_public_api.ex` | 全文件 |
| Session 鉴权 Plug | `lib/plausible_web/plugs/auth_plug.ex` | 全文件 |
| 站点授权 Plug | `lib/plausible_web/plugs/authorize_site_access.ex` | 全文件 |
| 路由配置 | `lib/plausible_web/router.ex` | 72-358 |

### 5.2 限速相关

| 功能 | 文件路径 | 行号 |
|------|---------|------|
| 限速核心模块 | `lib/plausible/rate_limit.ex` | 全文件 |
| 第三方 API 限速检查 | `lib/plausible_web/plugs/authorize_public_api.ex` | 187-214 |

### 5.3 审计相关

| 功能 | 文件路径 | 行号 |
|------|---------|------|
| 审计入口模块 | `extra/lib/plausible/audit.ex` | 全文件 |
| 审计条目 Schema | `extra/lib/plausible/audit/entry.ex` | 全文件 |
| 审计 Repo 封装 | `lib/plausible/audit/repo.ex` | 全文件 |
| LiveView 审计上下文 | `extra/lib/plausible/audit/live_context.ex` | 全文件 |
| 审计测试 | `test/plausible/audit_test.exs` | 全文件 |

### 5.4 控制器与查询层

| 功能 | 文件路径 | 行号 |
|------|---------|------|
| 第三方 API 控制器 | `lib/plausible_web/controllers/api/external_stats_controller.ex` | 全文件 |
| 第三方 API V2 控制器 | `lib/plausible_web/controllers/api/external_query_api_controller.ex` | 全文件 |
| Dashboard API 控制器 | `lib/plausible_web/controllers/api/stats_controller.ex` | 全文件 |
| 调试元数据 | `lib/plausible_web/controllers/helpers.ex` | 24-51 |
| 统一查询层 | `lib/plausible/stats.ex` | 全文件 |

---

## 六、常见问题解答

### Q1: 为什么统计查询不写审计日志？

**A**: 审计日志设计用于追踪**数据变更**和**配置变更**，而不是**数据读取**。原因：
1. **性能考虑**: 统计查询是高频操作，写入审计日志会带来显著性能开销
2. **存储考虑**: 大量的只读查询会快速填满审计日志表
3. **实际需求**: 用户通常更关心"谁改了什么"，而不是"谁查了什么"
4. **替代方案**: 通过 `debug_metadata`、日志分析、APM 等方式追踪查询

### Q2: 如何追踪统计查询的来源？

**A**: 使用以下方式组合：

1. **`debug_metadata`**: 所有统计查询都会传递这个元数据，包含 `phoenix_controller` 字段
2. **请求日志**: Phoenix 默认会记录请求路径、控制器、动作
3. **自定义中间件**: 可以在 Router 中添加自定义 plug 记录请求来源
4. **APM/可观测性**: 使用 OpenTelemetry (Plausible 已集成) 追踪请求

### Q3: 第三方 API 和 Dashboard 的 `user_id` 有什么区别？

**A**: 表面上看，两类请求都会设置 `conn.assigns.current_user`，但本质不同：

| 维度 | 第三方 API | Dashboard |
|------|-----------|-----------|
| 用户来源 | API Key 所属用户 | 登录用户 |
| 认证方式 | Bearer Token | Session Cookie |
| 审计上下文 | ❌ 无 | ✅ 有 |
| 典型场景 | 自动化脚本、第三方集成 | 用户在浏览器中操作 |

### Q4: 如果要给统计查询添加审计，应该怎么做？

**A**: 需要修改以下部分：

1. **第三方 API**: 在 `AuthorizePublicAPI` 中添加审计上下文设置
   ```elixir
   # 类似于 AuthPlug 的做法
   Plausible.Audit.set_context(%{
     current_user: api_key.user,
     current_team: api_key.team,
     request_source: :api_token  # 额外标识来源
   })
   ```

2. **查询层**: 在 `Plausible.Stats.*` 函数中添加审计日志写入
   - 注意: 需要考虑性能影响
   - 建议: 抽样审计或仅审计特定高价值查询

3. **字段扩展**: 在审计条目中添加更多查询相关字段
   - `query_type`: `aggregate`, `breakdown`, `timeseries` 等
   - `metrics`: 查询的指标
   - `dimensions`: 查询的维度
   - `date_range`: 日期范围

### Q5: Plugins API 和 Stats API 有什么区别？

**A**: 这是两种不同的 API：

| 维度 | Plugins API | Stats API |
|------|-------------|-----------|
| 鉴权方式 | Basic Auth (Plugins API Token) | Bearer Token |
| 路由前缀 | `/api/plugins` | `/api/v1/stats`, `/api/v2` |
| 核心 Plug | `AuthorizePluginsAPI` | `AuthorizePublicAPI` |
| 主要用途 | 插件生态、共享链接、Goals 管理 | 统计数据查询 |
| Token 存储 | `plugins_api_tokens` 表 | `api_keys` 表 |

**注意**：本文档主要讨论的是 **Stats API** (`/api/v1/stats`, `/api/v2`)，而不是 Plugins API (`/api/plugins`)。

---

## 七、修订历史

| 版本 | 日期 | 修订内容 |
|------|------|---------|
| v2.0 | 2026-05-05 | 完整修订版，澄清了审计日志写入机制、补充了完整鉴权链路、添加了无审计日志时的判别方式 |
| v1.0 | 2026-05-05 | 初始版本 |
