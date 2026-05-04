# Plausible Stats API 两类客户端鉴权机制对比分析

## 概述

Plausible 存在两类客户端访问统计查询层：
1. **第三方 Stats API** - 使用 Bearer Token 鉴权
2. **Dashboard** - 使用 Session Cookie 鉴权

本文档详细分析两类客户端在入口收敛、限速实现和审计区分三个维度的差异。

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

3. **Scope 验证**: 验证 API Key 的权限范围
   - 隐式 scopes: `["stats:read:*", "sites:read:*"]`
   - 前缀匹配机制: `some:*` 可匹配 `some:scope:*`

4. **站点访问验证**: 验证 API Key 对目标站点的访问权限

5. **Conn Assigns 设置**:
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

#### Pipeline 组成
```elixir
# lib/plausible_web/router.ex:72-78
pipeline :internal_stats_api do
  plug :accepts, ["json"]
  plug :fetch_session
  plug PlausibleWeb.AuthPlug
  plug PlausibleWeb.Plugs.AuthorizeSiteAccess
  plug PlausibleWeb.Plugs.NoRobots
end
```

#### 鉴权流程

1. **AuthPlug (Session 鉴权)**
   - 从 session 中获取用户会话: `UserAuth.get_user_session(conn)`
   - 加载用户信息、团队信息、团队角色
   - 设置 OpenTelemetry 和 Sentry 上下文
   - **关键**: 设置审计上下文 (仅 EE 版本)
     ```elixir
     # lib/plausible_web/plugs/auth_plug.ex:70-75
     on_ee do
       Plausible.Audit.set_context(%{
         current_user: user,
         current_team: current_team
       })
     end
     ```

2. **AuthorizeSiteAccess (站点访问授权)**
   - 从 URL 路径获取 `domain` 参数
   - 查找站点: `Repo.get_by(Plausible.Site, domain: domain)`
   - 验证用户角色: `Teams.Memberships.site_role(site, current_user)`
   - 支持的角色: `[:public, :viewer, :admin, :editor, :super_admin, :owner, :billing]`
   - 支持共享链接 (Shared Links) 访问

3. **Conn Assigns 设置**:
   - `:site` - 当前站点对象
   - `:site_role` - 用户在该站点的角色
   - `:shared_link` - 共享链接对象 (如果使用共享链接访问)
   - `:current_user` - 当前用户
   - `:current_team` - 当前团队

#### 控制器层
- **Api.StatsController** (`lib/plausible_web/controllers/api/stats_controller.ex`)
  - `query/2` → `Plausible.Stats.query/2`
  - `sources/2` → `Plausible.Stats.breakdown/4`
  - `aggregate/2` → `Plausible.Stats.aggregate/3`
  - `timeseries/2` → `Plausible.Stats.timeseries/3`
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

### 3.1 审计上下文设置对比

| 维度 | 第三方 Stats API (Token) | Dashboard (Session) |
|------|--------------------------|---------------------|
| 上下文设置 | **无显式审计上下文** | 有 (通过 `AuthPlug`) |
| 触发位置 | - | `AuthPlug.call/2` |
| 存储位置 | - | 进程元数据 (`:logger.process_metadata`) |
| 影响 | `actor_type = :system` | `actor_type = :user` |

### 3.2 Dashboard 审计上下文设置

```elixir
# lib/plausible_web/plugs/auth_plug.ex:70-75 (仅 EE 版本)
on_ee do
  Plausible.Audit.set_context(%{
    current_user: user,
    current_team: current_team
  })
end
```

**`set_context` 实现**:
```elixir
# extra/lib/plausible/audit/entry.ex:89-91
def set_context(kv) when is_map(kv) do
  :logger.update_process_metadata(%{:__audit__ => kv})
end
```

**上下文获取**:
```elixir
# extra/lib/plausible/audit/entry.ex:81-87
defp get_context() do
  case :logger.get_process_metadata() do
    %{:__audit__ => audit_context} -> audit_context
    %{} -> %{}
    :undefined -> %{}
  end
end
```

### 3.3 审计条目创建时的区分逻辑

```elixir
# extra/lib/plausible/audit/entry.ex:34-51
def changeset(name, params) do
  context = get_context()  # 从进程元数据获取

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

### 3.4 两类请求的审计特征

#### Dashboard 请求 (Session 鉴权)
**审计条目特征**:
- `actor_type = "user"` (因为 `context[:current_user]` 存在)
- `user_id` = 当前登录用户的 ID
- `team_id` = 当前团队的 ID

**触发条件**:
- 请求经过 `:browser` 或 `:api` pipeline
- `AuthPlug` 设置了审计上下文

#### 第三方 API 请求 (Token 鉴权)
**审计条目特征**:
- `actor_type = "system"` (因为 `context[:current_user]` 为 `nil`)
- `user_id` = `nil` 或默认值 `0`
- `team_id` = `nil` 或默认值 `0`

**触发条件**:
- 请求经过 `:public_api` pipeline
- `AuthorizePublicAPI` **没有调用** `Audit.set_context/1`
- 尽管设置了 `conn.assigns.current_user`，但这**不会**影响审计上下文

### 3.5 额外区分方式

除了审计上下文，还可以通过以下方式区分两类请求：

#### 1. 请求路径
| 客户端类型 | URL 模式 | 示例 |
|-----------|---------|------|
| Dashboard | `/api/stats/:domain/*` | `/api/stats/example.com/sources` |
| 第三方 API | `/api/v1/stats/*` | `/api/v1/stats/aggregate?site_id=example.com` |
| 第三方 API (V2) | `/api/v2/query` | `/api/v2/query` |

#### 2. Pipeline 标识
```elixir
# Dashboard
pipeline :internal_stats_api do
  plug :fetch_session          # 有 session
  plug PlausibleWeb.AuthPlug   # 有 session 鉴权
  plug PlausibleWeb.Plugs.AuthorizeSiteAccess  # 有站点授权
end

# 第三方 API
pipeline :public_api do
  plug :accepts, ["json"]  # 仅接受 JSON，无 session
end
# 额外通过 AuthorizePublicAPI plug 处理
```

#### 3. Conn Assigns 差异
| Assign Key | Dashboard | 第三方 API |
|------------|-----------|------------|
| `:site_role` | ✅ 有 (角色信息) | ❌ 无 |
| `:shared_link` | ✅ 可能有 (共享链接) | ❌ 无 |
| `:site_team` | ✅ 可能有 | ❌ 无 |
| `:api_scope` | ❌ 无 | ✅ 有 (如 `"stats:read:*"`) |
| `:api_context` | ❌ 无 | ✅ 有 (如 `:site`) |

---

## 四、完整对比表

| 维度 | 第三方 Stats API (Token) | Dashboard (Session) |
|------|--------------------------|---------------------|
| **鉴权方式** | Bearer Token (Authorization 头) | Session Cookie |
| **核心 Plug** | `AuthorizePublicAPI` | `AuthPlug` + `AuthorizeSiteAccess` |
| **路由前缀** | `/api/v1/stats`, `/api/v2` | `/api/stats/:domain` |
| **Pipeline** | `:public_api` | `:internal_stats_api` |
| **控制器** | `ExternalStatsController` | `Api.StatsController` |
| **统一查询层** | ✅ `Plausible.Stats.*` | ✅ `Plausible.Stats.*` |
| **小时级限速** | ✅ 有 (按 Team/User) | ❌ 无 |
| **突发限速** | ✅ 有 | ❌ 无 |
| **限速实现位置** | `AuthorizePublicAPI.call/2` | 无 |
| **审计上下文** | ❌ 未设置 | ✅ `AuthPlug` 中设置 |
| `actor_type` | `"system"` | `"user"` |
| `user_id` | 空/0 | 实际用户 ID |
| `team_id` | 空/0 | 实际团队 ID |

---

## 五、关键代码位置索引

| 功能 | 文件路径 | 行号 |
|------|---------|------|
| Token 鉴权 Plug | `lib/plausible_web/plugs/authorize_public_api.ex` | 全文件 |
| Session 鉴权 Plug | `lib/plausible_web/plugs/auth_plug.ex` | 全文件 |
| 站点授权 Plug | `lib/plausible_web/plugs/authorize_site_access.ex` | 全文件 |
| 限速核心模块 | `lib/plausible/rate_limit.ex` | 全文件 |
| 审计入口模块 | `extra/lib/plausible/audit.ex` | 全文件 |
| 审计条目 Schema | `extra/lib/plausible/audit/entry.ex` | 全文件 |
| 路由配置 | `lib/plausible_web/router.ex` | 72-358 |
| 第三方 API 控制器 | `lib/plausible_web/controllers/api/external_stats_controller.ex` | 全文件 |
| Dashboard API 控制器 | `lib/plausible_web/controllers/api/stats_controller.ex` | 全文件 |
| 统一查询层 | `lib/plausible/stats.ex` | 全文件 |
