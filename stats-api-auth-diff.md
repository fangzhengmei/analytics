# Plausible Stats API 两类客户端鉴权机制对比分析

## 概述

Plausible 存在两类客户端访问统计查询层：
1. **第三方 Stats API** - 使用 Bearer Token 鉴权
2. **Dashboard** - 使用 Session Cookie 鉴权

本文档详细分析两类客户端在入口收敛、限速实现、审计区分三个维度的差异，特别关注：
- **所有真实的审计日志写入入口**
- 各入口下的客户端身份判别
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
2. **API Key 查找**: 根据 token 和 context 查找
3. **限速检查** (详见第二章):
   - 小时级限制: `check_api_key_rate_limit/2`
   - 突发限制: `check_api_key_burst_limit/1`
4. **Scope 验证**: 验证 API Key 的权限范围
5. **站点访问验证**: 验证 API Key 对目标站点的访问权限
6. **Conn Assigns 设置**:
   ```elixir
   conn
   |> assign(:current_user, api_key.user)
   |> assign(:current_team, api_key.team)
   ```

#### 控制器层
- **ExternalStatsController** - V1 API
- **ExternalQueryApiController** - V2 API

---

### 1.3 Dashboard (Session 鉴权)

#### 路由配置
```elixir
# 路由位置: lib/plausible_web/router.ex:277-326

scope "/api" do
  pipe_through :internal_stats_api

  scope "/stats", PlausibleWeb.Api do
    post "/:domain/query", StatsController, :query
    get "/:domain/current-visitors", StatsController, :current_visitors
    get "/:domain/sources", StatsController, :sources
    # ... 更多统计端点
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

**链路 2: `AuthPlug` (核心 Session 鉴权 + 审计上下文设置)**
```elixir
# lib/plausible_web/plugs/auth_plug.ex:18-93
def call(conn, _opts) do
  case UserAuth.get_user_session(conn) do
    {:ok, user_session} ->
      user = user_session.user

      # 获取当前团队
      current_team_id =
        conn.params["__team"] || 
        Plug.Conn.get_session(conn, "current_team_id") || 
        user.last_team_identifier

      {current_team, current_team_role} = ...

      # 设置 OpenTelemetry 和 Sentry 上下文
      Plausible.OpenTelemetry.add_user_attributes(user)
      Sentry.Context.set_user_context(%{id: user.id, ...})

      # 关键：设置审计上下文 (仅 EE 版本)
      # 存储在 :logger.process_metadata 中
      on_ee do
        Plausible.Audit.set_context(%{
          current_user: user,
          current_team: current_team
        })
      end

      # 设置 Conn Assigns
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
- 从 URL 路径获取 `domain` 参数
- 查找站点并验证用户角色
- 支持的角色: `[:public, :viewer, :admin, :editor, :super_admin, :owner, :billing]`
- 支持共享链接访问

---

### 1.4 统一查询层收敛

两类客户端最终都调用 **Plausible.Stats** 模块的同一组函数：

| 功能 | 第三方 API | Dashboard | 统一查询层 |
|------|-----------|-----------|------------|
| 实时访客 | `ExternalStatsController` | `Api.StatsController` | `Stats.current_visitors/1` |
| 聚合查询 | `ExternalStatsController` | 内部使用 | `Stats.aggregate/3` |
| 分组查询 | `ExternalStatsController` | `Api.StatsController` | `Stats.breakdown/4` |
| 时间序列 | `ExternalStatsController` | 内部使用 | `Stats.timeseries/3` |
| 查询 API | `ExternalQueryApiController` | `Api.StatsController` | `Stats.query/2` |

---

## 二、限速实现机制

### 2.1 限速策略对比

| 维度 | 第三方 Stats API (Token) | Dashboard (Session) |
|------|--------------------------|---------------------|
| 限速级别 | API Key / Team 级别 | **无显式限速** |
| 限速类型 | 两级限速 (小时级 + 突发) | 依赖授权机制 |
| 实现位置 | `AuthorizePublicAPI` plug | 无专门实现 |

### 2.2 第三方 API 限速详解

#### 限速 Plug 位置
```elixir
# lib/plausible_web/plugs/authorize_public_api.ex:50-64
def call(conn, _opts) do
  with {:ok, token} <- get_bearer_token(conn),
       {:ok, api_key, limit_key, hourly_limit} <- find_api_key(conn, token, context),
       :ok <- check_api_key_rate_limit(limit_key, hourly_limit),      # 1. 小时级限速
       :ok <- check_api_key_burst_limit(limit_key),                    # 2. 突发限速
       {:ok, conn} <- verify_by_scope(conn, api_key, requested_scope) do
    # ...
  end
end
```

#### 底层限速实现

基于 ETS + `:atomics` 实现高性能并发计数：
```elixir
# lib/plausible/rate_limit.ex
defmodule Plausible.RateLimit do
  use GenServer  # 管理 ETS 表和定期清理进程

  @spec check_rate(table, key, scale, limit, increment) :: {:allow, count} | {:deny, limit}
  def check_rate(table \\ __MODULE__, key, scale, limit, increment \\ 1) do
    bucket = div(now(), scale)          # 时间分桶
    full_key = {key, bucket}            # 完整的 ETS 键
    expires_at = (bucket + 1) * scale   # 过期时间

    # 原子计数器操作
    count = case :ets.lookup(table, full_key) do
      [{_, counter, _}] -> :atomics.add_get(counter, 1, increment)
      [] -> # 创建新计数器
        counter = :atomics.new(1, signed: false)
        case :ets.insert_new(table, {full_key, counter, expires_at}) do
          true -> :atomics.add_get(counter, 1, increment)
          false -> # 并发竞争处理
            [{_, counter, _}] = :ets.lookup(table, full_key)
            :atomics.add_get(counter, 1, increment)
        end
    end

    if count <= limit, do: {:allow, count}, else: {:deny, limit}
  end
end
```

---

## 三、审计区分机制

### 3.1 关键认知澄清

在深入分析之前，需要明确几个重要事实：

**事实 1: 统计查询本身不写入审计日志！**
- 审计日志仅用于追踪数据变更和配置变更
- 统计查询是只读操作，不会触发审计日志写入

**事实 2: 存在两类审计写入入口！**
- 第一类：`*_with_audit` 方法（通过 `Audit.Repo`）
- 第二类：直接调用 `Audit.Entry.new() |> persist!()`

**事实 3: 审计上下文存储位置！**
- 审计上下文存储在 **`:logger.process_metadata`** 中
- **不是** 存储在 `conn.assigns` 中
- 两类客户端都设置 `conn.assigns.current_user`，但只有 Dashboard 设置审计上下文

---

### 3.2 审计入口总览

#### 审计写入入口分类

```
┌─────────────────────────────────────────────────────────────────────┐
│                     审计日志写入入口总览                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 第一类入口：*_with_audit 方法 (通过 Audit.Repo)              │   │
│  │                                                              │   │
│  │  调用链：                                                     │   │
│  │  业务代码 → *_with_audit → store_audit → persist!()        │   │
│  │                                                              │   │
│  │  特点：依赖审计上下文 (:logger.process_metadata)             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 第二类入口：直接调用 persist!()                               │   │
│  │                                                              │   │
│  │  调用链：                                                     │   │
│  │  业务代码 → Audit.Entry.new() → include_change() → persist!()│  │
│  │                                                              │   │
│  │  特点：不依赖审计上下文，可能通过 params 传入部分字段        │   │
│  │  示例：SSO login success/failure                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 3.3 第一类入口：`*_with_audit` 方法

#### 入口定义

```elixir
# lib/plausible/audit/repo.ex:41-105
defmacro __using__(_opts) do
  on_ee do
    quote do
      @behaviour Plausible.Audit.Repo

      # 更新操作带审计
      def update_with_audit(changeset, entry_name, params \\ %{}) do
        case update(changeset) do
          {:ok, result} ->
            store_audit(entry_name, result, changeset, params)
            {:ok, result}
          other -> other
        end
      end

      # 插入操作带审计
      def insert_with_audit(changeset, entry_name, params \\ %{}) do
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

#### 实际使用场景

从代码搜索结果看，当前使用 `*_with_audit` 方法的场景：

| 模块 | 方法 | 审计条目名称 |
|------|------|-------------|
| `Plausible.Teams` | `update_with_audit` | 动态 (audit_entry_name) |
| `Plausible.Auth.SSO` | `update_with_audit`, `insert_with_audit`, `delete_with_audit!` | `"sso_user_provisioned"`, `"sso_user_deprovioned"`, `"sso_integration_updated"`, `"sso_policy_updated"`, `"sso_force_mode_changed"`, `"sso_integration_removed"` |
| `Plausible.Auth.SSO.Domains` | `update_with_audit`, `insert_with_audit`, `delete_with_audit!` | `"sso_domain_added"`, `"sso_domain_verification_success"`, `"sso_domain_verification_failure"`, `"sso_domain_verification_cancelled"`, `"sso_domain_removed"` |
| `SettingsController` (测试中) | `update_with_audit`, `insert_with_audit`, `delete_with_audit!` | `"user_update"`, `"user_insert"`, `"user_delete"` |

#### 字段赋值逻辑

```elixir
# extra/lib/plausible/audit/entry.ex:34-51
def changeset(name, params) do
  # 从 :logger.process_metadata 获取审计上下文
  context = get_context()

  # Map.merge 规则：后一个参数覆盖前一个参数的相同键
  params =
    Map.merge(
      %{
        # 从审计上下文获取
        team_id: context[:current_team] && context.current_team.id,
        user_id: context[:current_user] && context.current_user.id,
        actor_type: if(context[:current_user], do: "user", else: "system")
      },
      params  # 调用时传入的参数
    )

  %__MODULE__{name: name}
  |> cast(params, [:entity, :entity_id, :meta, :user_id, :team_id, :actor_type])
  |> validate_required([:name, :entity, :entity_id, :actor_type])
  |> put_change(:datetime, NaiveDateTime.utc_now())
end
```

**审计上下文获取**:
```elixir
# extra/lib/plausible/audit/entry.ex:81-91
defp get_context() do
  case :logger.get_process_metadata() do
    %{:__audit__ => audit_context} -> audit_context  # 存在审计上下文
    %{} -> %{}                                          # 无审计上下文
    :undefined -> %{}
  end
end

def set_context(kv) when is_map(kv) do
  :logger.update_process_metadata(%{:__audit__ => kv})
end
```

#### 两类客户端在此入口下的判别

| 客户端类型 | 审计上下文设置 | 字段落库结果 |
|-----------|---------------|-------------|
| **Dashboard (Session)** | ✅ `AuthPlug` 中调用 `set_context` | `actor_type = "user"`, `user_id = 实际用户 ID`, `team_id = 实际团队 ID` |
| **第三方 API (Token)** | ❌ `AuthorizePublicAPI` 未调用 `set_context` | `actor_type = "system"` (默认), `user_id = nil/0` (默认), `team_id = nil/0` (默认) |

**关键点**:
- 两类客户端都设置 `conn.assigns.current_user`
- 但只有 Dashboard 设置 `:logger.process_metadata[:__audit__]`
- 审计字段依赖 `:logger.process_metadata`，**不依赖** `conn.assigns`

---

### 3.4 第二类入口：直接调用 `persist!()`

#### 入口发现

从代码搜索结果看，存在第二类审计写入入口：**直接调用 `Audit.Entry.new() |> persist!()`**，不经过 `*_with_audit` 方法。

**示例位置**:
- `extra/lib/plausible_web/sso/real_saml_adapter.ex:103-106, 115-120`
- `extra/lib/plausible_web/sso/fake_saml_adapter.ex:50-53`

#### 详细分析：SSO Login 审计

**路由配置** (`lib/plausible_web/router.ex:191-224`):
```elixir
pipeline :sso_saml do
  plug :accepts, ["html"]
  plug PlausibleWeb.Plugs.SecureSSO
  plug PlausibleWeb.Plugs.NoRobots
  plug :fetch_session
  plug :fetch_live_flash
  # 注意：没有 AuthPlug！
end

scope "/sso/saml", PlausibleWeb do
  pipe_through [:sso_saml]
  post "/consume/:integration_id", SSOController, :saml_consume  # SSO 回调入口
end
```

**调用链**:
```
用户访问 SSO IdP → 回调到 /sso/saml/consume/:integration_id
                              │
                              ▼
                    SSOController.saml_consume/2
                              │
                              ▼
                    saml_adapter().consume/2
                              │
                              ▼
              ┌───────────────┴───────────────┐
              │  RealSAMLAdapter.consume/4     │
              │                                │
              │  1. 验证 SAML 响应              │
              │  2. 提取用户属性                │
              │  3. 创建 SSO.Identity           │
              │  4. 直接写入审计日志 ← 关键    │
              │  5. 调用 log_in_user/3          │
              └────────────────────────────────┘
```

**核心代码** (`extra/lib/plausible_web/sso/real_saml_adapter.ex:80-125`):
```elixir
defp consume(conn, integration_id, cookie, saml_response, relay_state) do
  with {:ok, integration} <- SSO.get_integration(integration_id),
       :ok <- validate_authresp(cookie, relay_state),
       {:ok, {root, assertion}} <- SimpleSaml.parse_response(saml_response),
       :ok <- validate_signature(root, assertion, integration),
       {:ok, attributes} <- extract_attributes(assertion) do

    # 创建 SSO Identity
    identity =
      %SSO.Identity{
        id: assertion.name_id,
        integration_id: integration.identifier,
        name: name_from_attributes(attributes),
        email: attributes.email,
        expires_at: expires_at
      }

    # 关键：直接写入审计日志
    "sso_login_success"
    |> Plausible.Audit.Entry.new(identity, %{team_id: integration.team.id})  # 注意：传入 team_id！
    |> Plausible.Audit.Entry.include_change(identity)
    |> Plausible.Audit.Entry.persist!()

    # 审计日志写入后才登录用户
    PlausibleWeb.UserAuth.log_in_user(conn, identity, cookie.return_to)

  else
    {:error, reason} ->
      with {:ok, integration} <- SSO.get_integration(integration_id) do
        "sso_login_failure"
        |> Plausible.Audit.Entry.new(integration, %{team_id: integration.team.id})
        |> Plausible.Audit.Entry.include_change(%{error: inspect(reason)})
        |> Plausible.Audit.Entry.persist!()
      end

      login_error(conn, cookie, "Authentication failed")
  end
end
```

#### 关键发现

**发现 1: `:sso_saml` pipeline 没有 `AuthPlug`！**
- 审计上下文（`:logger.process_metadata[:__audit__]`）**没有设置**
- 在调用 `persist!()` 时，`context` 是 `%{}`

**发现 2: 审计日志在 `log_in_user` 之前写入！**
- `log_in_user` 会创建用户 session，但审计日志已经写入
- 即使 `log_in_user` 之后设置了审计上下文，也不会影响已写入的日志

**发现 3: `team_id` 通过 `params` 传入！**
```elixir
"Audit.Entry.new(identity, %{team_id: integration.team.id})"
```

**发现 4: `Map.merge` 的行为！**
```elixir
# changeset 中的 Map.merge
Map.merge(
  %{
    team_id: context[:current_team] && context.current_team.id,  # nil（无上下文）
    user_id: context[:current_user] && context.current_user.id,  # nil（无上下文）
    actor_type: if(context[:current_user], do: "user", else: "system")  # "system"
  },
  params  # %{team_id: integration.team.id}
)
```

**`Map.merge/2` 规则**：后一个参数覆盖前一个参数的相同键。

**结果**:
- `team_id`: `integration.team.id`（从 `params` 覆盖 `nil`）
- `user_id`: `nil`（`params` 没有这个键，使用 context 中的 `nil`）
- `actor_type`: `"system"`（`params` 没有这个键，使用 context 中的结果）

#### 字段落库结果

| 字段 | 值 | 来源 |
|------|---|------|
| `actor_type` | `"system"` | 默认值（无上下文） |
| `user_id` | `nil` 或 `0` | 默认值（无上下文，params 未传入） |
| `team_id` | `integration.team.id` | 从 `params` 传入，通过 `Map.merge` 覆盖 |

#### 此类入口的判别方式

对于第二类入口（直接调用 `persist!()`）：

**判别要点**:
1. **`actor_type = "system"`** 不代表一定是第三方 API
2. **需要检查 `team_id` 是否有值**：
   - 如果 `team_id` 有值，可能是通过 `params` 传入的
   - 需要结合审计条目名称（`name` 字段）判断

**示例分析**：

| 审计条目名称 | `actor_type` | `team_id` | `user_id` | 可能的客户端类型 |
|-------------|--------------|-----------|-----------|-----------------|
| `"sso_login_success"` | `"system"` | 有值 | `nil/0` | SSO 登录（第二类入口） |
| `"sso_login_failure"` | `"system"` | 有值 | `nil/0` | SSO 登录失败（第二类入口） |
| `"sso_domain_added"` | 取决于上下文 | 取决于上下文 | 取决于上下文 | SSO 配置（第一类入口） |
| `"user_update"` | 取决于上下文 | 取决于上下文 | 取决于上下文 | 用户设置（第一类入口） |

**判别策略**:
1. 首先检查审计条目名称（`name` 字段）
2. `"sso_login_*"` 系列 → 第二类入口，SSO 登录场景
3. 其他名称 → 第一类入口，按 `actor_type` 判断

---

### 3.5 审计入口与客户端判别总结

#### 完整判别流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                    审计条目判别完整流程                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. 检查审计条目名称 (name 字段)                                     │
│                                                                      │
│     ┌─────────────────────────────────────────────────────────────┐ │
│     │ name 以 "sso_login_" 开头？                                   │ │
│     │ (如 "sso_login_success", "sso_login_failure")                │ │
│     └─────────────────────────────────────────────────────────────┘ │
│                      │                        │                      │
│                      │ 是                      │ 否                   │
│                      ▼                        ▼                      │
│     ┌────────────────────────┐    ┌────────────────────────────┐  │
│     │ 第二类入口：SSO 登录   │    │ 第一类入口：*_with_audit  │  │
│     │                        │    │                            │  │
│     │ 判别方式：              │    │ 判别方式：                 │  │
│     │ - name 字段明确标识     │    │ - 检查 actor_type          │  │
│     │ - actor_type = "system"│    │ - "user" → Dashboard       │  │
│     │ - team_id 有值          │    │ - "system" → 第三方 API    │  │
│     │ - user_id = nil/0       │    │                            │  │
│     │                        │    │ 注意：                      │  │
│     │ 注意：                   │    │ 两类客户端都可能设置       │  │
│     │ actor_type = "system"   │    │ conn.assigns.current_user  │  │
│     │ 不代表第三方 API         │    │ 但只有 Dashboard 设置      │  │
│     │ 这是 SSO 登录的特殊情况  │    │ 审计上下文                 │  │
│     └────────────────────────┘    └────────────────────────────┘  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

#### 各入口下的字段落库对比表

| 入口类型 | 审计条目名称示例 | `actor_type` | `team_id` | `user_id` | 说明 |
|---------|-----------------|--------------|-----------|-----------|------|
| **第一类入口** (`*_with_audit`) | `"sso_domain_added"`, `"user_update"` | 取决于审计上下文 | 取决于审计上下文 | 取决于审计上下文 | |
| - Dashboard 调用 | | `"user"` | 实际团队 ID | 实际用户 ID | 有审计上下文 |
| - 第三方 API 调用 | | `"system"` | `nil/0` | `nil/0` | 无审计上下文 |
| **第二类入口** (直接 `persist!()`) | `"sso_login_success"`, `"sso_login_failure"` | | | | |
| - SSO 登录 | | `"system"` | 有值 (从 params) | `nil/0` | 无审计上下文，但 `team_id` 从 params 传入 |

#### 关键澄清

**之前的结论修正**:
1. **`actor_type = "system"` 不唯一代表第三方 API**
   - 也可能是第二类入口（如 SSO 登录）
   - 需要结合审计条目名称判断

2. **`team_id` 有值不代表有审计上下文**
   - 第二类入口可能通过 `params` 传入 `team_id`
   - 此时 `user_id` 仍然是 `nil/0`

3. **审计上下文与 `conn.assigns` 是分开的**
   - 审计上下文存储在 `:logger.process_metadata`
   - `conn.assigns` 中的 `current_user` 不影响审计字段
   - 只有显式调用 `Audit.set_context()` 才会设置审计上下文

---

### 3.6 无审计写入时的客户端判别

由于统计查询本身不写审计日志，需要通过其他方式区分两类客户端。

#### 方式 1: `debug_metadata` (推荐)

所有统计查询控制器都会调用 `debug_metadata(conn)`：

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
```

**通过 `phoenix_controller` 区分**:

| 客户端类型 | `phoenix_controller` 值 |
|-----------|------------------------|
| Dashboard | `"Elixir.PlausibleWeb.Api.StatsController"` |
| 第三方 API (V1) | `"Elixir.PlausibleWeb.Api.ExternalStatsController"` |
| 第三方 API (V2) | `"Elixir.PlausibleWeb.Api.ExternalQueryApiController"` |

#### 方式 2: 请求路径

| 客户端类型 | URL 模式 |
|-----------|---------|
| Dashboard | `/api/stats/:domain/*` (路径包含域名) |
| 第三方 API (V1) | `/api/v1/stats/*` (`site_id` 作为查询参数) |
| 第三方 API (V2) | `/api/v2/query` (固定路径) |

#### 方式 3: Conn Assigns

| Assign Key | Dashboard | 第三方 API |
|------------|-----------|------------|
| `:site_role` | ✅ 有 (`:admin`, `:viewer` 等) | ❌ 无 |
| `:shared_link` | ✅ 可能有 | ❌ 无 |
| `:current_user_session` | ✅ 有 | ❌ 无 |
| `:api_scope` | ❌ 无 | ✅ 有 (如 `"stats:read:*"`) |
| `:api_context` | ❌ 无 | ✅ 有 (如 `:site`) |

#### 方式 4: HTTP 头

| 客户端类型 | Authorization 头 | Cookie |
|-----------|-----------------|--------|
| Dashboard | 无 (或 CSRF token) | 有 (`_plausible_analytics_key`) |
| 第三方 API | `Bearer <token>` | 无 |

---

## 四、完整对比表

### 4.1 鉴权与入口对比

| 维度 | 第三方 Stats API (Token) | Dashboard (Session) |
|------|--------------------------|---------------------|
| **鉴权方式** | Bearer Token | Session Cookie |
| **核心 Plug** | `AuthorizePublicAPI` | `AuthPlug` + `AuthorizeSiteAccess` |
| **路由前缀** | `/api/v1/stats`, `/api/v2` | `/api/stats/:domain` |
| **Pipeline** | `:public_api` | `:internal_stats_api` |
| **控制器** | `ExternalStatsController`, `ExternalQueryApiController` | `Api.StatsController` |

### 4.2 限速对比

| 维度 | 第三方 Stats API (Token) | Dashboard (Session) |
|------|--------------------------|---------------------|
| 小时级限速 | ✅ 有 (按 Team/User) | ❌ 无 |
| 突发限速 | ✅ 有 | ❌ 无 |
| 限速实现位置 | `AuthorizePublicAPI.call/2` | 无 |

### 4.3 审计对比

#### 审计入口分类

| 入口类型 | 调用方式 | 依赖审计上下文 | 示例 |
|---------|---------|---------------|------|
| **第一类** | `*_with_audit` → `store_audit` → `persist!()` | ✅ 依赖 | SSO 配置变更、用户设置变更 |
| **第二类** | 直接 `Audit.Entry.new() |> persist!()` | ❌ 不依赖 | SSO 登录成功/失败 |

#### 第一类入口下的字段落库

| 字段 | Dashboard 调用 | 第三方 API 调用 |
|------|---------------|-----------------|
| `actor_type` | `"user"` | `"system"` (默认) |
| `user_id` | 实际用户 ID | `nil/0` (默认) |
| `team_id` | 实际团队 ID | `nil/0` (默认) |

#### 第二类入口下的字段落库 (SSO Login 示例)

| 字段 | 值 | 来源 |
|------|---|------|
| `actor_type` | `"system"` | 默认值 (无审计上下文) |
| `user_id` | `nil/0` | 默认值 (params 未传入) |
| `team_id` | 有值 | 从 `params` 传入，`Map.merge` 覆盖 |

#### 关键澄清

| 问题 | 答案 |
|------|------|
| `actor_type = "system"` 是否一定是第三方 API？ | **否** - 也可能是第二类入口（如 SSO 登录） |
| `team_id` 有值是否代表有审计上下文？ | **否** - 第二类入口可能通过 `params` 传入 |
| 如何区分 "system" 的两种情况？ | 检查审计条目名称：`"sso_login_*"` → 第二类入口 |
| 审计上下文存储在哪里？ | `:logger.process_metadata`，**不是** `conn.assigns` |

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
| SSO 路由配置 | `lib/plausible_web/router.ex` | 189-225 |
| 统计查询路由 | `lib/plausible_web/router.ex` | 277-358 |

### 5.2 限速相关

| 功能 | 文件路径 | 行号 |
|------|---------|------|
| 限速核心模块 | `lib/plausible/rate_limit.ex` | 全文件 |
| 第三方 API 限速检查 | `lib/plausible_web/plugs/authorize_public_api.ex` | 187-214 |

### 5.3 审计相关

#### 审计入口

| 入口类型 | 文件路径 | 行号 | 说明 |
|---------|---------|------|------|
| 第一类入口定义 | `lib/plausible/audit/repo.ex` | 全文件 | `*_with_audit` 方法定义 |
| 第二类入口 (SSO Login) | `extra/lib/plausible_web/sso/real_saml_adapter.ex` | 103-120 | 直接调用 `persist!()` |
| 第二类入口 (Fake SSO) | `extra/lib/plausible_web/sso/fake_saml_adapter.ex` | 50-53 | 测试用直接调用 |

#### 审计核心

| 功能 | 文件路径 | 行号 |
|------|---------|------|
| 审计条目 Schema | `extra/lib/plausible/audit/entry.ex` | 全文件 |
| 审计上下文获取/设置 | `extra/lib/plausible/audit/entry.ex` | 81-91 |
| 审计入口模块 | `extra/lib/plausible/audit.ex` | 全文件 |
| 审计测试 | `test/plausible/audit_test.exs` | 全文件 |

#### 审计使用示例

| 功能 | 文件路径 | 行号 |
|------|---------|------|
| SSO 领域管理 (第一类) | `extra/lib/plausible/auth/sso/domains.ex` | 20-208 |
| SSO 集成管理 (第一类) | `extra/lib/plausible/auth/sso.ex` | 59-482 |
| 团队策略管理 (第一类) | `lib/plausible/teams.ex` | 368 |
| SSO Controller | `extra/lib/plausible_web/controllers/sso_controller.ex` | 全文件 |

### 5.4 其他辅助

| 功能 | 文件路径 | 行号 |
|------|---------|------|
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

### Q2: `actor_type = "system"` 是否一定是第三方 API？

**A: 否！** 存在两种 `"system"` 的情况：

| 情况 | `actor_type` | `team_id` | `user_id` | 审计条目名称示例 |
|------|--------------|-----------|-----------|-----------------|
| 第三方 API (第一类入口) | `"system"` | `nil/0` | `nil/0` | 不涉及 |
| SSO 登录 (第二类入口) | `"system"` | **有值** | `nil/0` | `"sso_login_success"`, `"sso_login_failure"` |

**判别方法**:
1. 检查审计条目名称是否以 `"sso_login_"` 开头
2. 检查 `team_id` 是否有值（第二类入口通过 `params` 传入）
3. 结合业务场景判断

### Q3: 审计上下文存储在哪里？

**A**: 审计上下文存储在 **`:logger.process_metadata`** 中，**不是** `conn.assigns`。

**关键代码**:
```elixir
# 存储
def set_context(kv) do
  :logger.update_process_metadata(%{:__audit__ => kv})
end

# 获取
defp get_context() do
  case :logger.get_process_metadata() do
    %{:__audit__ => audit_context} -> audit_context
    %{} -> %{}
    :undefined -> %{}
  end
end
```

**重要区分**:
- `conn.assigns.current_user`: 两类客户端都可能设置
- `:logger.process_metadata[:__audit__]`: 只有 Dashboard 经过 `AuthPlug` 后才设置
- 审计字段依赖后者，不依赖前者

### Q4: 第二类入口为什么通过 `params` 传入 `team_id`？

**A**: 因为第二类入口的调用场景中，审计上下文**没有设置**。

**以 SSO Login 为例**:
1. `:sso_saml` pipeline **没有 `AuthPlug`**
2. 审计日志在 `log_in_user` 之前写入
3. 此时用户还没有登录，审计上下文为空
4. 但 `team_id` 可以从 `SSO.Integration` 中获取
5. 所以通过 `params` 传入 `team_id`

**Map.merge 行为**:
```elixir
Map.merge(
  %{team_id: nil, user_id: nil, actor_type: "system"},  # 来自上下文（空）
  %{team_id: integration.team.id}  # 来自 params
)
# 结果: %{team_id: integration.team.id, user_id: nil, actor_type: "system"}
```

### Q5: 如何追踪统计查询的来源？

**A**: 使用以下方式组合：

1. **`debug_metadata`** (首选): 所有统计查询都会传递，包含 `phoenix_controller` 字段
2. **请求日志**: Phoenix 默认会记录请求路径、控制器、动作
3. **自定义中间件**: 可以在 Router 中添加自定义 plug 记录请求来源
4. **APM/可观测性**: 使用 OpenTelemetry (Plausible 已集成) 追踪请求

**关键区分字段**:
- `phoenix_controller`: 最准确，直接标识控制器模块
- `request_path`: 容易获取，模式清晰
- `:api_scope` / `:site_role` 等 Conn Assigns: 辅助判断

### Q6: 如果要给第三方 API 添加审计上下文，应该怎么做？

**A**: 在 `AuthorizePublicAPI` plug 中添加审计上下文设置：

```elixir
# 类似于 AuthPlug 的做法
# lib/plausible_web/plugs/authorize_public_api.ex

def call(conn, _opts) do
  with {:ok, token} <- get_bearer_token(conn),
       {:ok, api_key, limit_key, hourly_limit} <- find_api_key(conn, token, context),
       :ok <- check_api_key_rate_limit(limit_key, hourly_limit),
       :ok <- check_api_key_burst_limit(limit_key),
       {:ok, conn} <- verify_by_scope(conn, api_key, requested_scope) do

    # 添加审计上下文设置
    on_ee do
      Plausible.Audit.set_context(%{
        current_user: api_key.user,
        current_team: api_key.team,
        request_source: :api_token,  # 额外标识来源
        api_key_id: api_key.id        # 可选：记录 API Key ID
      })
    end

    conn
    |> assign(:current_user, api_key.user)
    |> assign(:current_team, api_key.team)
  end
end
```

**注意**:
- 当前 `AuthorizePublicAPI` 只设置 `conn.assigns`，不设置审计上下文
- 添加后，第一类入口的第三方 API 调用会有 `actor_type = "user"`
- 但需要考虑：是否应该区分"人类用户"和"API Key 所属用户"？

### Q7: Plugins API 和 Stats API 有什么区别？

**A**: 这是两种不同的 API：

| 维度 | Plugins API | Stats API |
|------|-------------|-----------|
| 鉴权方式 | Basic Auth (Plugins API Token) | Bearer Token |
| 路由前缀 | `/api/plugins` | `/api/v1/stats`, `/api/v2` |
| 核心 Plug | `AuthorizePluginsAPI` | `AuthorizePublicAPI` |
| 主要用途 | 插件生态、共享链接、Goals 管理 | 统计数据查询 |
| Token 存储 | `plugins_api_tokens` 表 | `api_keys` 表 |

**注意**：本文档主要讨论的是 **Stats API**，不是 Plugins API。

---

## 七、修订历史

| 版本 | 日期 | 修订内容 |
|------|------|---------|
| **v3.0** | 2026-05-05 | **重大修订**：发现第二类审计写入入口（直接 `persist!()`），修正 `actor_type = "system"` 的判别逻辑，添加完整的审计入口分析和判别流程图 |
| v2.0 | 2026-05-05 | 完整修订版，澄清了审计日志写入机制、补充了完整鉴权链路、添加了无审计日志时的判别方式 |
| v1.0 | 2026-05-05 | 初始版本 |

---

## 八、关键发现总结

### 8.1 审计写入入口分类

**第一类入口** (`*_with_audit` 方法):
- 通过 `Audit.Repo` 的 `store_audit` 函数
- 依赖审计上下文 (`:logger.process_metadata`)
- `actor_type` 由审计上下文决定

**第二类入口** (直接 `persist!()`):
- 直接调用 `Audit.Entry.new() |> include_change() |> persist!()`
- **不依赖**审计上下文
- 部分字段可能通过 `params` 传入（如 `team_id`）
- 示例：SSO 登录成功/失败

### 8.2 `actor_type` 判别修正

**之前的错误结论**: `actor_type = "system"` → 第三方 API

**修正后的结论**:

| `actor_type` | 可能的场景 | 辅助判别 |
|--------------|-----------|---------|
| `"user"` | Dashboard 调用第一类入口 | 审计上下文存在 |
| `"system"` | 第三方 API 调用第一类入口 | 无上下文，`team_id = nil/0` |
| `"system"` | 第二类入口（如 SSO 登录） | 审计条目名称以 `"sso_login_"` 开头，`team_id` 有值 |

### 8.3 审计上下文与 `conn.assigns` 的关键区别

| 存储位置 | 设置时机 | 影响审计字段？ |
|---------|---------|--------------|
| `conn.assigns.current_user` | `AuthPlug` 或 `AuthorizePublicAPI` | ❌ 不影响 |
| `:logger.process_metadata[:__audit__]` | **仅** `AuthPlug` 或显式调用 `Audit.set_context()` | ✅ 直接影响 |

**重要**: 两类客户端都可能设置 `conn.assigns.current_user`，但只有 Dashboard 设置审计上下文。

### 8.4 无审计日志时的判别策略

**推荐优先级**:
1. `phoenix_controller` (通过 `debug_metadata`) → 最准确
2. 请求路径 → 容易获取
3. Conn Assigns (`:api_scope` / `:site_role`) → 辅助判断
4. HTTP 头 (Authorization / Cookie) → 特殊场景

**不能用于判别的字段**:
- `user_id` (来自 `conn.assigns`，两类客户端都可能有)
- `team_id` (来自 `conn.assigns`，两类客户端都可能有)
