# 订阅与账单状态传播分析报告 (R2)

**重点**: 账单事件触发后的截止日期更新链路、无有效订阅时的真实行为、短暂不一致窗口分析

---

## 1. 账单事件触发后的更新链路

### 1.1 入口：Paddle Webhook 处理

**文件**: `lib/plausible_web/controllers/api/paddle_controller.ex:8-26`

```elixir
def webhook(conn, %{"alert_name" => "subscription_created"} = params) do
  Plausible.Billing.subscription_created(params) |> webhook_response(conn, params)
end

def webhook(conn, %{"alert_name" => "subscription_updated"} = params) do
  Plausible.Billing.subscription_updated(params) |> webhook_response(conn, params)
end

def webhook(conn, %{"alert_name" => "subscription_cancelled"} = params) do
  Plausible.Billing.subscription_cancelled(params) |> webhook_response(conn, params)
end

def webhook(conn, %{"alert_name" => "subscription_payment_succeeded"} = params) do
  Plausible.Billing.subscription_payment_succeeded(params) |> webhook_response(conn, params)
end
```

### 1.2 关键处理函数：`after_subscription_update/1`

**文件**: `lib/plausible/billing/billing.ex:219-238`

这是订阅/账单状态变化后的核心处理链路：

```elixir
defp after_subscription_update(subscription) do
  team =
    Teams.Team
    |> Repo.get!(subscription.team_id)
    |> Teams.with_subscription()
    |> Repo.preload(:owners)

  team
    |> Plausible.Teams.update_accept_traffic_until()      # 1. 更新截止日期
    |> Plausible.Teams.remove_grace_period()              # 2. 移除宽限期
    |> Plausible.Teams.maybe_reset_next_upgrade_override() # 3. 重置升级覆盖
    |> tap(&Plausible.Billing.SiteLocker.update_for/1)    # 4. 更新锁定状态
    |> maybe_adjust_api_key_limits()                       # 5. 调整API限制
end
```

### 1.3 各事件的更新行为

| 事件类型 | 是否调用 `after_subscription_update` | `accept_traffic_until` 更新 | 锁定状态更新 |
|----------|--------------------------------------|------------------------------|--------------|
| `subscription_created` | ✅ 是 (`billing.ex:57`) | ✅ 更新 | ✅ 检查并更新 |
| `subscription_updated` | ✅ 是 (`billing.ex:84`) | ✅ 更新 | ✅ 检查并更新 |
| `subscription_cancelled` | ❌ **否** (`billing.ex:88-110`) | ❌ **不更新** | ❌ **不检查** |
| `subscription_payment_succeeded` | ❌ **否** (单独处理) | ✅ 单独更新 (`billing.ex:131`) | ❌ **不检查** |

### 1.4 `accept_traffic_until` 计算逻辑

**文件**: `lib/plausible/teams.ex:279-306` (EE版本)

```elixir
def accept_traffic_until(team) do
  team = with_subscription(team)

  cond do
    on_trial?(team) ->
      Date.shift(team.trial_expiry_date,
        day: Teams.Team.trial_accept_traffic_until_offset_days()  # 14天
      )

    team.subscription && team.subscription.paddle_plan_id == "free_10k" ->
      @accept_traffic_until_free  # 2135-01-01 (永久)

    team.subscription && team.subscription.next_bill_date ->
      Date.shift(team.subscription.next_bill_date,
        day: Teams.Team.subscription_accept_traffic_until_offset_days()  # 30天
      )

    true ->
      raise "This user is neither on trial or has a valid subscription. Manual intervention required."
  end
end
```

**计算规则汇总**:

| 条件 | `accept_traffic_until` 值 |
|------|---------------------------|
| 试用期内 | `trial_expiry_date + 14天` |
| Free 10k 计划 | `2135-01-01` (永久) |
| 有效订阅且有 `next_bill_date` | `next_bill_date + 30天` |
| 其他情况 | **抛出异常** |

### 1.5 更新链路流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Paddle Webhook 事件                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐              │
│  │ subscription │    │ subscription │    │ subscription_    │              │
│  │  _created    │    │  _updated    │    │ payment_succeeded│              │
│  └──────┬───────┘    └──────┬───────┘    └────────┬─────────┘              │
│         │                   │                     │                         │
│         ▼                   ▼                     │                         │
│  ┌─────────────────────────────────────┐          │                         │
│  │   after_subscription_update/1       │          │                         │
│  └─────────────────┬───────────────────┘          │                         │
│                    │                              │                         │
│     ┌──────────────┼──────────────┐               │                         │
│     ▼              ▼              ▼               │                         │
│ ┌────────┐  ┌────────────┐  ┌────────────┐       │                         │
│ │update  │  │ remove_    │  │SiteLocker  │       │                         │
│ │_accept │  │ grace_     │  │.update_for │       │                         │
│ │_traffic │  │ period    │  │            │       │                         │
│ └────┬───┘  └────────────┘  └─────┬──────┘       │                         │
│      │                             │               │                         │
│      ▼                             ▼               │                         │
│ ┌─────────────┐            ┌────────────────┐      │                         │
│ │ 更新数据库  │            │ 检查并设置     │      │                         │
│ │teams.      │            │teams.locked    │      │                         │
│ │accept_     │            │                │      │                         │
│ │traffic_    │            └────────────────┘      │                         │
│ │until       │                                      │                         │
│ └─────────────┘                                      │                         │
│                                                      │                         │
│  subscription_payment_succeeded 单独处理:          │                         │
│  ┌──────────────────────────────────────────────────────────────────────┐    │
│  │ 1. 更新 subscription.next_bill_date / last_bill_date                  │    │
│  │ 2. 单独调用 Teams.update_accept_traffic_until(team)                    │    │
│  │ 3. ❌ 不调用 SiteLocker.update_for                                      │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘

⚠️  关键发现: subscription_cancelled 不调用 after_subscription_update
    → accept_traffic_until 不会自动更新
    → 锁定状态不会重新检查
```

---

## 2. 无有效订阅时的真实行为

### 2.1 订阅"活跃"状态定义

**文件**: `lib/plausible/billing/subscriptions.ex:7-16`

```elixir
def active?(%Subscription{status: Subscription.Status.active()}), do: true
def active?(%Subscription{status: Subscription.Status.past_due()}), do: true

def active?(%Subscription{status: Subscription.Status.deleted()} = subscription) do
  not is_nil(subscription.next_bill_date) and
    not Date.before?(subscription.next_bill_date, Date.utc_today())
end

def active?(%Subscription{}), do: false
def active?(nil), do: false
```

**关键理解**:
- `status = "deleted"` 的订阅，只要 `next_bill_date >= today`，仍被认为是"活跃"的
- 这是因为用户已支付到 `next_bill_date`，服务应该持续到该日期

### 2.2 无有效订阅的场景分析

#### 场景A: 订阅已取消但 next_bill_date 仍在未来

**状态**:
- `subscription.status = "deleted"`
- `next_bill_date` > today (例如: 还有15天到下一个账单日)
- 数据库中 `accept_traffic_until = next_bill_date + 30天`

**行为**:
1. `Subscriptions.active?(subscription)` → `true` (因为 `next_bill_date >= today`)
2. 如果调用 `update_accept_traffic_until`:
   - 条件 `team.subscription && team.subscription.next_bill_date` 为 true
   - 新的 `accept_traffic_until = next_bill_date + 30天` (仍在未来)
3. `GateKeeper` 使用数据库中存储的值 → 仍在未来 → **接受流量**

#### 场景B: 订阅已取消且 next_bill_date 已过

**状态**:
- `subscription.status = "deleted"`
- `next_bill_date` < today
- 数据库中 `accept_traffic_until` = 过去的日期

**行为**:
1. `Subscriptions.active?(subscription)` → `false`
2. `GateKeeper` 检查:
   - 数据库中 `accept_traffic_until` < today
   - 返回 `:payment_required` → **拒绝流量**

#### 场景C: 无订阅记录 (subscription 为 nil)

**状态**:
- `team.subscription = nil`

**行为分析**:

1. **`accept_traffic_until/1` 实时计算** (如果被调用):
   ```elixir
   cond do
     on_trial?(team) -> trial_expiry_date + 14天  # 可能匹配
     team.subscription && ... -> false            # 不匹配
     true -> raise "..."                           # 抛出异常
   end
   ```
   - 如果在试用期内: 正常计算
   - 如果不在试用期且无订阅: **抛出异常**

2. **`GateKeeper` 实际行为**:
   - `GateKeeper` 使用的是**数据库中存储的** `teams.accept_traffic_until` 值
   - 不是实时调用 `accept_traffic_until/1` 计算
   - 所以即使无订阅，只要数据库中存储的值仍在未来，就会接受流量

### 2.3 关键发现：subscription_cancelled 不更新截止日期

**文件**: `lib/plausible/billing/billing.ex:88-110`

```elixir
defp handle_subscription_cancelled(params) do
  subscription =
    Subscription
    |> Repo.get_by(paddle_subscription_id: params["subscription_id"])
    |> Repo.preload(team: [:owners, :billing_members])

  if subscription do
    changeset =
      Subscription.changeset(subscription, %{
        status: params["status"]  # 只更新 status
      })

    updated = Repo.update!(changeset)  # 更新 subscription.status = "deleted"

    # 发送取消邮件
    for recipient <- subscription.team.owners ++ subscription.team.billing_members do
      recipient
        |> PlausibleWeb.Email.cancellation_email()
        |> Plausible.Mailer.send()
    end

    updated  # ❌ 没有调用 after_subscription_update
  end
end
```

**影响**:
1. `accept_traffic_until` 保持原值 (`next_bill_date + 30天`)
2. `locked` 状态不会重新检查
3. 宽限期不会被移除

**这是设计还是缺陷?**

分析:
- 设计意图可能是: 用户已支付到 `next_bill_date`，服务应该持续到 `next_bill_date + 30天`
- 但问题是: 没有后台任务在 `next_bill_date` 过后更新 `accept_traffic_until`

---

## 3. 缓存刷新机制与延迟窗口

### 3.1 Site.Cache 刷新调度

**文件**: `lib/plausible/application.ex:63-76`

```elixir
warmed_cache(Plausible.Site.Cache,
  warmers: [
    refresh_all:
      {Plausible.Site.Cache.All,
       interval: :timer.minutes(15) + Enum.random(1..:timer.seconds(10))},  # 15分钟
    refresh_updated_recently:
      {Plausible.Site.Cache.RecentlyUpdated, interval: :timer.seconds(30)}      # 30秒
  ]
)
```

### 3.2 refresh_updated_recently 的关键限制

**文件**: `lib/plausible/cache.ex:133-146`

```elixir
def refresh_updated_recently(opts) do
  recently_updated_query =
    from([s, ...] in base_db_query(),
      order_by: [asc: s.updated_at],
      where: s.updated_at > ago(^15, "minute")  # ⚠️ 只检查 site.updated_at
    )

  refresh(
    :updated_recently,
    recently_updated_query,
    Keyword.put(opts, :delete_stale_items?, false)
  )
end
```

### 3.3 Site.Cache 的查询结构

**文件**: `lib/plausible/site/cache.ex:49-59`

```elixir
def base_db_query() do
  from s in Site.regular(),
    left_join: rg in assoc(s, :revenue_goals),
    inner_join: team in assoc(s, :team),  # join team 表
    select: {
      s.domain,
      s.domain_changed_from,
      %{struct(s, ^@cached_schema_fields) | from_cache?: true}
    },
    preload: [revenue_goals: rg, team: team]  # preload team 数据
end
```

**`@cached_schema_fields`** (`lib/plausible/site/cache.ex:28-34`):

```elixir
@cached_schema_fields ~w(
  id
  domain
  domain_changed_from
  ingest_rate_limit_scale_seconds
  ingest_rate_limit_threshold
)a
```

⚠️ **重要发现**:
- `@cached_schema_fields` 只包含 `site` 表的字段
- 但查询通过 `preload: [team: team]` 同时加载了 `team` 数据
- 所以缓存的 `%Site{}` 结构体包含 `team` 关联数据

### 3.4 关键问题：team 更新不会触发快速缓存刷新

**场景**:
1. `update_accept_traffic_until/1` 更新 `teams.accept_traffic_until`
2. 这会更新 `team.updated_at` (如果 Ecto 自动更新)
3. 但 `site.updated_at` **不会** 自动更新

**结果**:
- `refresh_updated_recently` (每30秒) 的 where 条件 `s.updated_at > ago(15, "minute")` 不会匹配
- 缓存直到 `refresh_all` (每15分钟) 才会刷新

### 3.5 延迟窗口汇总

| 刷新类型 | 间隔 | 触发条件 | team 更新后是否刷新 |
|----------|------|----------|---------------------|
| `refresh_updated_recently` | 30秒 | `site.updated_at` 在15分钟内 | ❌ **否** |
| `refresh_all` | 15分钟 | 无条件 | ✅ 是 |

**实际延迟**:
- **最小延迟**: 0 (如果 `site.updated_at` 恰好也更新了，或者刚巧遇到 `refresh_all`)
- **最大延迟**: **15分钟** (从 team 更新到下一次 `refresh_all`)
- **典型延迟**: ~7.5分钟 (平均等待时间)

---

## 4. 短暂不一致窗口详细分析

### 4.1 不一致窗口汇总表

| 窗口类型 | 最小持续时间 | 最大持续时间 | 触发条件 | 影响 |
|----------|-------------|-------------|----------|------|
| **缓存延迟窗口** | 0 | 15分钟 | `team.accept_traffic_until` 更新后 | 采集层使用旧的截止日期 |
| **订阅取消后宽限窗口** | 0 | 30+天 | `subscription_cancelled` 不更新截止日期 | 应拒绝的流量继续被接受 |
| **支付成功后锁定状态窗口** | 0 | 到下一次 LockSites | `subscription_payment_succeeded` 不调用 SiteLocker | 应解锁的仍显示锁定 |
| **数据库 vs 缓存不一致** | 0 | 15分钟 | 任何数据库更新后 | 数据层和采集层状态不同步 |

### 4.2 窗口1: 缓存延迟窗口

**时间线示例**:

```
时间轴 (T=0 为账单事件触发时间)

T=0: 数据库更新
     ├── teams.accept_traffic_until = 过去的日期 (应拒绝)
     └── 缓存中仍为未来的日期 (会接受)

T=30秒: refresh_updated_recently 运行
        └── 只检查 site.updated_at，不刷新
            缓存仍为旧值

T=1分钟: 同上，缓存仍为旧值
...
T=15分钟: refresh_all 运行
          └── 缓存刷新为新值
              现在开始正确拒绝流量

不一致窗口: 0 ~ 15分钟
```

**影响**:
- 应被拒绝的流量继续被接受
- 或者应被接受的流量被错误拒绝

### 4.3 窗口2: 订阅取消后宽限窗口

**时间线示例**:

```
场景: 用户在 2026-04-01 取消订阅
      next_bill_date = 2026-04-15 (已支付到这天)

T=0 (2026-04-01): subscription_cancelled 事件
                   ├── subscription.status = "deleted"
                   ├── accept_traffic_until 保持原值: 2026-04-15 + 30天 = 2026-05-15
                   └── ❌ 没有计划在 2026-04-15 后更新这个值

T=2026-04-16: next_bill_date 已过
              ├── Subscriptions.active?(subscription) = false
              └── 但 accept_traffic_until = 2026-05-15 (仍在未来)
              → GateKeeper 仍接受流量

T=2026-05-16: accept_traffic_until 已过
              → GateKeeper 开始拒绝流量

不一致窗口: 从 subscription_cancelled 到 accept_traffic_until 过期
            最长可达: (next_bill_date - 取消日期) + 30天
            极端情况: 约60天
```

**关键问题**:
- 设计意图可能是"已支付的服务应该继续"
- 但 `accept_traffic_until = next_bill_date + 30天` 意味着在 `next_bill_date` 后还有30天的宽限
- 这30天是"未支付"的

### 4.4 窗口3: 支付成功后锁定状态窗口

**文件**: `lib/plausible/billing/billing.ex:112-135`

```elixir
defp handle_subscription_payment_succeeded(params) do
  subscription = Repo.get_by(Subscription, paddle_subscription_id: params["subscription_id"])

  if subscription do
    # 1. 从 Paddle API 获取最新信息
    {:ok, api_subscription} = paddle_api().get_subscription(subscription.paddle_subscription_id)

    # 2. 更新 subscription 表
    subscription =
      subscription
        |> Subscription.changeset(%{
          next_bill_amount: amount,
          next_bill_date: api_subscription["next_payment"]["date"],
          last_bill_date: api_subscription["last_payment"]["date"]
        })
        |> Repo.update!()
        |> Repo.preload(:team)

    # 3. 更新 accept_traffic_until
    Plausible.Teams.update_accept_traffic_until(subscription.team)

    # 4. ❌ 没有调用:
    #    - remove_grace_period
    #    - SiteLocker.update_for
  end
end
```

**对比 `after_subscription_update`**:
```elixir
defp after_subscription_update(subscription) do
  team = ...
  team
    |> Plausible.Teams.update_accept_traffic_until()   # ✅ 有
    |> Plausible.Teams.remove_grace_period()           # ❌ 没有
    |> Plausible.Teams.maybe_reset_next_upgrade_override()  # ❌ 没有
    |> tap(&Plausible.Billing.SiteLocker.update_for/1)  # ❌ 没有
    |> maybe_adjust_api_key_limits()                     # ❌ 没有
end
```

**影响场景**:

用户因为超量被锁定 (`locked = true`, `grace_period` 活跃)，然后升级订阅并支付成功:

1. `subscription_payment_succeeded` 触发
2. `accept_traffic_until` 更新为新的未来日期
3. ❌ `grace_period` 没有被移除
4. ❌ `locked` 状态没有被检查/重置
5. ❌ 用户看到面板仍被锁定

**用户需要等到**:
- 下一次 `LockSites` 任务运行
- 或者手动刷新

### 4.5 窗口4: 数据库 vs 缓存不一致

**数据流**:

```
┌─────────────┐        ┌─────────────┐        ┌─────────────┐
│  数据库     │        │    缓存     │        │ GateKeeper  │
│             │        │             │        │   (采集层)   │
│ teams.      │        │             │        │             │
│ accept_     │───────▶│  preload    │───────▶│ 使用缓存中  │
│ traffic_    │  定时  │  的 team    │  实时  │ 的 team 值  │
│ until       │  刷新  │             │        │             │
└─────────────┘        └─────────────┘        └─────────────┘
       △                      │
       │                      │
       └──────────────────────┘
         refresh_all (15分钟)
         refresh_updated_recently (30秒，但只看 site.updated_at)
```

**不一致的可能性**:

| 数据库状态 | 缓存状态 | 实际行为 |
|-----------|---------|---------|
| 未来日期 | 过去日期 | 应接受但被拒绝 |
| 过去日期 | 未来日期 | 应拒绝但被接受 |
| 未来日期 | 未来日期 | 一致 ✓ |
| 过去日期 | 过去日期 | 一致 ✓ |

---

## 5. 状态传播完整链路图

### 5.1 从账单事件到采集层

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                        账单事件 → 采集层状态传播                                │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌──────────────────┐                                                         │
│  │  Paddle Webhook  │                                                         │
│  └────────┬─────────┘                                                         │
│           │                                                                    │
│           ▼                                                                    │
│  ┌──────────────────────┐                                                     │
│  │ Billing.subscription │                                                     │
│  │   _created/updated   │                                                     │
│  └──────────┬───────────┘                                                     │
│             │                                                                  │
│             ▼                                                                  │
│  ┌──────────────────────────┐    ┌──────────────────────────────────────┐  │
│  │ after_subscription_update│    │  subscription_cancelled (单独处理)  │  │
│  └──────────┬───────────────┘    └──────────────────┬───────────────────┘  │
│             │                                         │                       │
│             ▼                                         ▼                       │
│  ┌──────────────────────┐              ┌─────────────────────────────┐     │
│  │1. update_accept_     │              │ 只更新:                      │     │
│  │   traffic_until      │              │  subscription.status         │     │
│  │   (数据库)            │              │                             │     │
│  └──────────┬───────────┘              │ ❌ 不更新:                  │     │
│             │                           │    accept_traffic_until     │     │
│             ▼                           │    locked 状态               │     │
│  ┌──────────────────────┐              └─────────────────────────────┘     │
│  │2. SiteLocker.        │                                                 │
│  │   update_for         │                                                 │
│  │   (检查并设置 locked) │                                                 │
│  └──────────────────────┘                                                 │
│                                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                           缓存层                                       │  │
│  ├──────────────────────────────────────────────────────────────────────┤  │
│  │                                                                      │  │
│  │   数据库更新后，缓存刷新取决于:                                       │  │
│  │   ├── refresh_updated_recently (30秒)                               │  │
│  │   │   └── 只检查 site.updated_at                                     │  │
│  │   │   └── ❌ team 更新不会触发                                       │  │
│  │   │                                                                  │  │
│  │   └── refresh_all (15分钟)                                          │  │
│  │       └── ✅ 无条件刷新所有                                           │  │
│  │                                                                      │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                          GateKeeper (采集层)                          │  │
│  ├──────────────────────────────────────────────────────────────────────┤  │
│  │                                                                      │  │
│  │   检查逻辑 (lib/plausible/site/gate_keeper.ex:45-59):              │  │
│  │                                                                      │  │
│  │   with %Site{team: %{accept_traffic_until: atu}} = site <-         │  │
│  │        Cache.get(domain, ...),                                      │  │
│  │        true <- Plausible.Sites.regular?(site) do                    │  │
│  │                                                                      │  │
│  │     if not is_nil(atu) and Date.after?(Date.utc_today(), atu) do  │  │
│  │       :payment_required   ❌ 拒绝流量                                │  │
│  │     else                                                             │  │
│  │       check_rate_limit(site, opts)  ✅ 接受流量                     │  │
│  │     end                                                              │  │
│  │   end                                                                │  │
│  │                                                                      │  │
│  │   ⚠️ 关键点:                                                         │  │
│  │   - 使用的是 Cache 中 preload 的 team.accept_traffic_until          │  │
│  │   - 不是实时查询数据库                                               │  │
│  │   - 不是调用 accept_traffic_until/1 函数计算                        │  │
│  │                                                                      │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                               │
└────────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 从账单事件到展示层

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                        账单事件 → 展示层状态传播                                │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  展示层有两个独立的控制字段:                                                   │
│  ├── teams.locked          - 控制面板是否可访问                               │
│  └── teams.grace_period    - 控制通知显示和倒计时                             │
│                                                                               │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  locked 状态更新                                                          │ │
│  ├─────────────────────────────────────────────────────────────────────────┤ │
│  │                                                                         │ │
│  │  触发点:                                                                │ │
│  │  1. after_subscription_update → SiteLocker.update_for/1              │ │
│  │  2. LockSites 后台任务 (定期运行)                                       │ │
│  │                                                                         │ │
│  │  SiteLocker.update_for 逻辑 (lib/plausible/billing/site_locker.ex):  │ │
│  │                                                                         │ │
│  │  case Teams.Billing.check_needs_to_upgrade(team) do                   │ │
│  │    {:needs_to_upgrade, :grace_period_ended} ->                        │ │
│  │      set_lock_status_for(team, true)   ✅ 设置 locked=true            │ │
│  │      Teams.end_grace_period(team)       ✅ 设置 grace_period.is_over  │ │
│  │                                                                         │ │
│  │    {:needs_to_upgrade, reason} ->                                      │ │
│  │      if owned_sites_count > 0 do                                       │ │
│  │        set_lock_status_for(team, true)  ✅ 设置 locked=true           │ │
│  │      end                                                                │ │
│  │                                                                         │ │
│  │    :no_upgrade_needed ->                                                │ │
│  │      set_lock_status_for(team, false)   ✅ 设置 locked=false          │ │
│  │  end                                                                    │ │
│  │                                                                         │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  展示层检查逻辑                                                          │ │
│  ├─────────────────────────────────────────────────────────────────────────┤ │
│  │                                                                         │ │
│  │  StatsController (lib/plausible_web/controllers/stats_controller.ex): │ │
│  │                                                                         │ │
│  │  can_see_stats? = not Teams.locked?(site.team) or site_role == :super│ │
│  │                                                                         │ │
│  │  cond do                                                                │ │
│  │    can_see_stats? -> render("stats.html", ...)    ✅ 正常显示         │ │
│  │    Teams.locked?(site.team) -> render("site_locked.html", ...) ❌ 锁定│ │
│  │  end                                                                    │ │
│  │                                                                         │ │
│  │  布局层通知 (lib/plausible_web/templates/layout/_notice.html.heex):   │ │
│  │                                                                         │ │
│  │  <Notice.dashboard_locked :if={Plausible.Teams.GracePeriod.expired?} │ │
│  │                                                                         │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ⚠️ 关键点:                                                                   │
│  - 展示层直接查询数据库 (teams.locked, teams.grace_period)                 │
│  - 不经过 Site.Cache                                                         │
│  - 所以数据库更新后，展示层能实时看到变化                                     │
│  - 但采集层仍受缓存延迟影响                                                   │
│                                                                               │
└────────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. 关键发现汇总

### 6.1 最严重的问题

| 问题 | 严重程度 | 影响 | 代码位置 |
|------|----------|------|----------|
| `subscription_cancelled` 不更新 `accept_traffic_until` | 高 | 应拒绝的流量继续被接受最长可达60天 | `billing.ex:88-110` |
| `refresh_updated_recently` 不检查 `team.updated_at` | 高 | 任何 team 字段更新后，缓存最多延迟15分钟才刷新 | `cache.ex:133-146` |
| `subscription_payment_succeeded` 不调用 `SiteLocker` | 中 | 支付成功后面板可能仍显示锁定 | `billing.ex:112-135` |

### 6.2 状态不一致矩阵

| 事件类型 | 数据库更新 | 缓存刷新 | 展示层 | 采集层 | 一致性 |
|----------|-----------|---------|--------|--------|--------|
| `subscription_created` | ✅ | ⚠️ 延迟 | ✅ 实时 | ⚠️ 延迟 | 部分 |
| `subscription_updated` | ✅ | ⚠️ 延迟 | ✅ 实时 | ⚠️ 延迟 | 部分 |
| `subscription_cancelled` | ❌ 部分 | ⚠️ 延迟 | ✅ 实时 | ❌ 不同步 | **不一致** |
| `subscription_payment_succeeded` | ⚠️ 部分 | ⚠️ 延迟 | ⚠️ 可能过时 | ⚠️ 延迟 | 部分 |

### 6.3 建议改进

1. **`subscription_cancelled` 处理**:
   - 应该调用 `after_subscription_update` 或至少更新 `accept_traffic_until`
   - 或者确保有后台任务在 `next_bill_date` 过后更新截止日期

2. **缓存刷新机制**:
   - `refresh_updated_recently` 应该同时检查 `team.updated_at`
   - 或者在更新 `team` 相关字段时，也更新关联 `site.updated_at`

3. **`subscription_payment_succeeded` 处理**:
   - 应该调用完整的 `after_subscription_update` 流程
   - 至少应该调用 `remove_grace_period` 和 `SiteLocker.update_for`

---

## 7. 代码位置速查

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| Webhook 入口 | `lib/plausible_web/controllers/api/paddle_controller.ex` | 8-26 |
| after_subscription_update | `lib/plausible/billing/billing.ex` | 219-238 |
| handle_subscription_cancelled | `lib/plausible/billing/billing.ex` | 88-110 |
| handle_subscription_payment_succeeded | `lib/plausible/billing/billing.ex` | 112-135 |
| accept_traffic_until 计算 | `lib/plausible/teams.ex` | 279-306 |
| SiteLocker.update_for | `lib/plausible/billing/site_locker.ex` | 14-49 |
| GateKeeper 检查逻辑 | `lib/plausible/site/gate_keeper.ex` | 45-59 |
| Site.Cache 刷新调度 | `lib/plausible/application.ex` | 63-76 |
| refresh_updated_recently | `lib/plausible/cache.ex` | 133-146 |
| Subscription.active? | `lib/plausible/billing/subscriptions.ex` | 7-16 |
| 展示层 locked 检查 | `lib/plausible_web/controllers/stats_controller.ex` | 51-114 |
