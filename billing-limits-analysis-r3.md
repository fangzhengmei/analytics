# 订阅与账单状态传播分析报告 (R3)

**重点**: 支付成功后锁定状态恢复时机、取消订阅后的策略边界、按定时任务节拍量化不一致窗口、修正结论

---

## 1. 定时任务调度配置

### 1.1 核心定时任务时间表 (UTC)

**文件**: `config/runtime.exs:811-826`

| 任务名称 | Cron 表达式 | 执行时间 | 主要职责 |
|----------|-------------|----------|----------|
| `LockSites` | `"0 0 * * *"` | 每天 00:00 | 检查并设置 `teams.locked` 状态 |
| `AcceptTrafficUntil` | `"0 8 * * *"` | 每天 08:00 | 发送截止日期临近通知（**不更新截止日期**） |
| `CheckUsage` | `"0 14 * * *"` | 每天 14:00 | 检查页面浏览量，启动/移除宽限期 |
| `SendTrialNotifications` | `"0 12 * * *"` | 每天 12:00 | 发送试用相关通知 |

### 1.2 任务调度时序图

```
一天的时间轴 (UTC):

00:00  LockSites ──────────────────▶ 检查锁定状态
           │
           ▼
08:00  AcceptTrafficUntil ─────────▶ 发送通知邮件
           │
           ▼
12:00  SendTrialNotifications ─────▶ 试用通知
           │
           ▼
14:00  CheckUsage ─────────────────▶ 检查页面浏览量
           │
           ▼
次日 00:00  LockSites ─────────────▶ 再次检查锁定状态
```

---

## 2. 支付成功后锁定状态恢复的触发时机

### 2.1 `subscription_payment_succeeded` 处理流程

**文件**: `lib/plausible/billing/billing.ex:112-135`

```elixir
defp handle_subscription_payment_succeeded(params) do
  subscription = Repo.get_by(Subscription, paddle_subscription_id: params["subscription_id"])

  if subscription do
    # 1. 从 Paddle API 获取最新信息
    {:ok, api_subscription} = paddle_api().get_subscription(subscription.paddle_subscription_id)

    # 2. 计算金额
    amount =
      Money.new(api_subscription["next_payment"]["currency"], api_subscription["next_payment"]["amount"])

    # 3. 更新 subscription 表
    subscription =
      subscription
        |> Subscription.changeset(%{
          next_bill_amount: amount,
          next_bill_date: api_subscription["next_payment"]["date"],
          last_bill_date: api_subscription["last_payment"]["date"]
        })
        |> Repo.update!()
        |> Repo.preload(:team)

    # 4. ✅ 更新 accept_traffic_until
    Plausible.Teams.update_accept_traffic_until(subscription.team)

    # 5. ❌ 不调用:
    #    - remove_grace_period
    #    - SiteLocker.update_for
  end
end
```

### 2.2 对比 `after_subscription_update`

**文件**: `lib/plausible/billing/billing.ex:219-238`

```elixir
defp after_subscription_update(subscription) do
  team = ... |> Repo.preload(:owners)

  team
    |> Plausible.Teams.update_accept_traffic_until()      # ✅ 有
    |> Plausible.Teams.remove_grace_period()              # ❌ 支付成功没有
    |> Plausible.Teams.maybe_reset_next_upgrade_override() # ❌ 支付成功没有
    |> tap(&Plausible.Billing.SiteLocker.update_for/1)    # ❌ 支付成功没有
    |> maybe_adjust_api_key_limits()                       # ❌ 支付成功没有
end
```

### 2.3 支付成功后锁定状态恢复的完整时序

**场景**: 用户因超量被锁定（`locked = true`, `grace_period.is_over = true`），然后升级订阅并支付成功。

```
时间轴 (假设支付发生在 10:00 UTC):

T=10:00  支付成功，Paddle 发送 subscription_payment_succeeded webhook
           │
           ▼
T=10:00  handle_subscription_payment_succeeded 执行:
           ├── ✅ subscription.next_bill_date 更新为未来日期
           ├── ✅ team.accept_traffic_until 更新为 next_bill_date + 30天
           ├── ❌ grace_period 未移除
           ├── ❌ locked 状态未重置
           └── ❌ SiteLocker.update_for 未调用
           │
           ▼
T=10:00 ~ T=次日 00:00  状态不一致:
           ├── 展示层: locked = true → 面板仍显示锁定
           ├── 采集层: accept_traffic_until 已更新 → 接受流量
           └── 数据库: locked = true, grace_period.is_over = true
           │
           ▼
T=次日 00:00  LockSites 任务运行:
           │
           ├── 调用 SiteLocker.update_for(team)
           │       │
           │       ▼
           │    ┌─────────────────────────────────────────────────────┐
           │    │ check_needs_to_upgrade 判断逻辑:                    │
           │    │                                                      │
           │    │ cond do                                              │
           │    │   Plausible.Teams.on_trial?(team) ->               │
           │    │     :no_upgrade_needed                              │
           │    │                                                      │
           │    │   not Subscriptions.active?(subscription) ->        │
           │    │     {:needs_to_upgrade, :no_active_trial_or_sub}   │
           │    │                                                      │
           │    │   Teams.GracePeriod.expired?(team) ->               │
           │    │     revise_pageview_usage(team, usage_mod)          │
           │    │                                                      │
           │    │   true ->                                            │
           │    │     :no_upgrade_needed                              │
           │    │ end                                                  │
           │    └─────────────────────────────────────────────────────┘
           │       │
           │       ▼
           │    关键点分析:
           │    ├── subscription.status 可能仍是 "deleted" (如果是升级)
           │    ├── 但 next_bill_date 已更新为未来日期
           │    └── Subscriptions.active?(subscription) 判断:
           │
           │    ┌─────────────────────────────────────────────────────┐
           │    │ subscriptions.ex:7-16                               │
           │    │                                                      │
           │    │ def active?(%Subscription{status: "deleted"} = s) do│
           │    │   not is_nil(s.next_bill_date) and                  │
           │    │     not Date.before?(s.next_bill_date, today)       │
           │    │ end                                                  │
           │    │                                                      │
           │    │ → 如果 next_bill_date >= today，返回 true           │
           │    └─────────────────────────────────────────────────────┘
           │
           ▼
    分支判断:
    ├── 如果 subscription 被认为是 "active":
    │       ├── 检查 GracePeriod.expired?(team)
    │       │      ├── 如果 grace_period.is_over = true (之前被锁定)
    │       │      │      ├── expired?() = true
    │       │      │      ├── 调用 revise_pageview_usage
    │       │      │      ├── 如果现在使用量在限额内:
    │       │      │      │      ├── remove_grace_period
    │       │      │      │      └── :no_upgrade_needed
    │       │      │      └── 如果仍超量:
    │       │      │             └── {:needs_to_upgrade, :grace_period_ended}
    │       │      │
    │       └── 如果 grace_period 已被移除:
    │              └── :no_upgrade_needed
    │
    └── 如果 subscription 被认为是 "not active":
            └── {:needs_to_upgrade, :no_active_trial_or_subscription}
           │
           ▼
T=次日 00:00+  LockSites 执行完成:
           ├── 如果 :no_upgrade_needed → set_lock_status_for(team, false)
           └── 如果 {:needs_to_upgrade, ...} → set_lock_status_for(team, true)
```

### 2.4 关键发现：支付成功后的不一致窗口

**不一致窗口持续时间**:

| 场景 | 支付时间 | 下一次 LockSites | 不一致窗口 |
|------|----------|-----------------|-----------|
| 最糟情况 | 00:01 UTC | 次日 00:00 | **约 23小时59分钟** |
| 典型情况 | 10:00 UTC | 次日 00:00 | **约 14小时** |
| 最好情况 | 23:59 UTC | 次日 00:00 | **约 1分钟** |

**不一致期间的状态**:

| 层面 | 状态 | 影响 |
|------|------|------|
| **数据库** | `locked = true`, `grace_period.is_over = true` | - |
| **展示层** | `locked = true` → 显示锁定页面 | 用户无法查看数据 |
| **采集层** | `accept_traffic_until` 已更新为未来日期 | 数据继续被采集 |

**用户体验影响**:
- 用户刚刚升级并支付成功
- 但面板仍显示"已锁定"
- 需要等待最多 **24小时** 才能恢复正常访问

---

## 3. 取消订阅后的策略边界

### 3.1 订阅"活跃"状态的定义

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
- `status = "deleted"` 只表示**订阅已被取消**，不表示**立即失效**
- 用户已支付到 `next_bill_date`，服务应该持续到该日期
- 所以 `status = "deleted"` 且 `next_bill_date >= today` → 仍被认为是"活跃"的

### 3.2 测试用例验证

**文件**: `test/workers/lock_sites_test.exs:58-80`

```elixir
test "does not lock user who cancelled subscription but it hasn't expired yet" do
  user = new_user() |> subscribe_to_growth_plan(status: Subscription.Status.deleted())
  site = new_site(owner: user)

  LockSites.perform(nil)

  refute Repo.reload!(site.team).locked  # 不会锁定
end

test "locks user who cancelled subscription and the cancelled subscription has expired" do
  user =
    new_user(trial_expiry_date: Date.utc_today() |> Date.shift(day: -1))
    |> subscribe_to_growth_plan(
      status: Subscription.Status.deleted(),
      next_bill_date: Date.utc_today() |> Date.shift(day: -1)  # next_bill_date 已过
    )

  site = new_site(owner: user)

  LockSites.perform(nil)

  assert Repo.reload!(site.team).locked  # 会锁定
end
```

### 3.3 取消订阅后的策略边界表

| 条件 | 展示层 (locked) | 采集层 (accept_traffic_until) |
|------|-----------------|-------------------------------|
| `status = "deleted"` 且 `next_bill_date >= today` | ❌ 不锁定 | ✅ 接受流量 (值为 `next_bill_date + 30天`) |
| `status = "deleted"` 且 `next_bill_date < today` | ✅ 锁定 (下次 LockSites 后) | ✅ 接受流量直到 `next_bill_date + 30天` |
| `status = "deleted"` 且 `next_bill_date + 30天 < today` | ✅ 已锁定 | ❌ 拒绝流量 |

### 3.4 关键发现：双截止日期策略

系统实际使用**两个独立的截止日期**：

| 截止日期 | 用途 | 值 |
|----------|------|-----|
| **展示层截止** (`next_bill_date`) | 控制面板是否锁定 | 账单日期 |
| **采集层截止** (`accept_traffic_until`) | 控制是否接受流量 | `next_bill_date + 30天` |

**设计意图分析**:
- 展示层在 `next_bill_date` 后锁定：提醒用户已过付费期
- 采集层在 `next_bill_date + 30天` 后停止：给用户30天宽限导出数据

**但存在的问题**:
1. `subscription_cancelled` webhook 不更新 `accept_traffic_until`
2. 即使 `next_bill_date` 已过，采集层仍继续接受流量
3. 直到 `accept_traffic_until` 自然过期

### 3.5 取消订阅后的时序示例

**场景**: 用户在 2026-04-01 取消订阅，`next_bill_date = 2026-04-15`

```
时间轴:

2026-04-01  subscription_cancelled webhook:
           ├── subscription.status = "deleted"
           ├── ❌ accept_traffic_until 保持原值: 2026-04-15 + 30天 = 2026-05-15
           └── ❌ 不调用 SiteLocker.update_for
           │
           ▼
2026-04-15  next_bill_date 已过:
           ├── Subscriptions.active?(subscription) = false
           └── 但 LockSites 只在 00:00 UTC 运行
           │
           ▼
2026-04-16 00:00  LockSites 运行:
           ├── check_needs_to_upgrade
           │       ├── on_trial?() = false
           │       ├── Subscriptions.active?() = false
           │       └── {:needs_to_upgrade, :no_active_trial_or_subscription}
           │
           └── set_lock_status_for(team, true) → locked = true
           │
           ▼
2026-04-16 ~ 2026-05-15  状态不一致期间:
           ├── 展示层: locked = true → 显示锁定页面
           ├── 采集层: accept_traffic_until = 2026-05-15 (仍在未来)
           │       └── 继续接受流量约 29天
           │
           └── 不一致窗口: 约 29天
           │
           ▼
2026-05-15  accept_traffic_until 过期:
           ├── 缓存刷新后 (0 ~ 15分钟延迟)
           └── 采集层开始拒绝流量
           │
           ▼
2026-05-16 00:00  LockSites 再次运行:
           ├── 状态与之前相同: locked = true
           └── 无变化
```

**不一致窗口量化**:

| 阶段 | 持续时间 | 不一致类型 |
|------|----------|-----------|
| 取消到 `next_bill_date` | 约 14天 (示例中) | 无不一致 (都正常) |
| `next_bill_date` 到 `accept_traffic_until` | **约 30天** | 展示层锁定，采集层仍接受流量 |

---

## 4. CheckUsage 与 LockSites 的协作流程

### 4.1 CheckUsage 的触发条件

**文件**: `test/workers/check_usage_test.exs:508-580`

```elixir
describe "timing" do
  test "checks usage one day after the last_bill_date" do
    # 只有在 last_bill_date 后一天才检查
  end

  test "does not check exactly one month after last_bill_date" do
    # 账单日当天不检查
  end

  test "for yearly subscriptions, checks usage multiple months + one day after" do
    # 年付订阅也检查
  end
end
```

**关键发现**:
- `CheckUsage` 不是每天检查所有用户
- 只检查 `last_bill_date` 后一天的用户
- 这意味着使用量检查不是实时的，而是按账单日周期性的

### 4.2 完整的超限 → 锁定流程

```
时间轴 (假设用户在 2026-04-15 账单日后持续超量):

T=2026-04-14  last_bill_date (上一个账单日)
           │
           ▼
T=2026-04-15  账单日当天
           └── CheckUsage 不检查 ("does not check exactly one month after")
           │
           ▼
T=2026-04-16 00:00  LockSites 运行:
           ├── 检查是否需要锁定
           └── 宽限期还没启动 → 不锁定
           │
           ▼
T=2026-04-16 14:00  CheckUsage 运行:
           ├── 检查 last_bill_date 后一天的用户
           ├── 如果连续两个周期超量
           │       ├── 发送超限邮件
           │       └── 启动宽限期: grace_period.end_date = 2026-04-16 + 7 = 2026-04-23
           │
           └── 如果没有连续两个周期超量
                   └── 不启动宽限期
           │
           ▼
T=2026-04-17 00:00  LockSites 运行:
           ├── check_needs_to_upgrade
           │       ├── subscription.active?() = true
           │       ├── GracePeriod.expired?(team) = false (end_date=2026-04-23)
           │       └── :no_upgrade_needed
           │
           └── set_lock_status_for(team, false) → 不锁定
           │
           ▼
... (宽限期期间: 2026-04-16 ~ 2026-04-23) ...
           │
           ▼
T=2026-04-23  宽限期 end_date = today
           │
           ├── GracePeriod.active?() 判断:
           │       Date.diff(end_date, today) >= 0 → true
           │
           └── 当天仍算"活跃"
           │
           ▼
T=2026-04-24 00:00  LockSites 运行:
           ├── GracePeriod.expired?(team) = true
           │
           ├── revise_pageview_usage:
           │       ├── 如果仍超量 → {:needs_to_upgrade, :grace_period_ended}
           │       └── 如果在限额内 → remove_grace_period, :no_upgrade_needed
           │
           └── 如果仍超量:
                   ├── set_lock_status_for(team, true)
                   ├── end_grace_period → grace_period.is_over = true
                   └── 发送锁定邮件
           │
           ▼
T=2026-04-24  锁定生效:
           ├── 展示层: locked = true → 显示锁定页面
           └── 采集层: accept_traffic_until 仍是未来日期 → 继续接受流量
```

### 4.3 从超限检测到实际锁定的时间线

| 阶段 | 持续时间 | 说明 |
|------|----------|------|
| 超限发生到 CheckUsage 检测 | **0 ~ 30天** | 取决于账单日 |
| 宽限期 | **7天** | 固定 |
| 宽限期结束到 LockSites 运行 | **0 ~ 24小时** | 取决于宽限期结束时间 |
| **总计 (最糟情况)** | **约 61天** | 从超限到实际锁定 |

---

## 5. 缓存刷新对不一致窗口的影响

### 5.1 缓存刷新调度

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

### 5.2 refresh_updated_recently 的关键限制

**文件**: `lib/plausible/cache.ex:133-146`

```elixir
def refresh_updated_recently(opts) do
  recently_updated_query =
    from([s, ...] in base_db_query(),
      order_by: [asc: s.updated_at],
      where: s.updated_at > ago(^15, "minute")  # ⚠️ 只检查 site.updated_at
    )
  ...
end
```

### 5.3 缓存刷新对不一致窗口的叠加影响

**场景**: `accept_traffic_until` 数据库更新后

```
时间轴:

T=0  数据库更新:
           ├── team.accept_traffic_until = 新值
           ├── team.updated_at = now()
           └── ❌ site.updated_at 不更新
           │
           ▼
T=30秒  refresh_updated_recently 运行:
           ├── 只检查 site.updated_at > ago(15, "minute")
           └── site 没有变化 → 不刷新缓存
           │
           ▼
T=1分钟, 2分钟, ..., 14分钟  同上，缓存仍为旧值
           │
           ▼
T=15分钟  refresh_all 运行:
           └── 缓存刷新为新值
```

**缓存延迟窗口量化**:

| 情况 | 延迟 |
|------|------|
| 如果 `site.updated_at` 恰好也更新了 | 30秒内 |
| 典型情况 (只有 `team` 更新) | **0 ~ 15分钟** |

---

## 6. 不一致窗口汇总与量化

### 6.1 各场景的不一致窗口

| 场景 | 主要延迟源 | 最小窗口 | 最大窗口 | 典型窗口 |
|------|-----------|----------|----------|----------|
| **支付成功后** | LockSites 每日运行 + 缓存 | 1分钟 + 0 | **24小时 + 15分钟** | 14小时 + 7.5分钟 |
| **取消订阅后** | 双截止日期差异 + LockSites | 0 | **30天 + 15分钟** | 30天 |
| **超限检测到锁定** | CheckUsage 周期 + 宽限期 + LockSites | 7天 + 0 | **61天 + 15分钟** | 38天 |
| **订阅状态变化** | 缓存刷新 | 30秒 | **15分钟** | 7.5分钟 |

### 6.2 状态不一致矩阵

| 事件 | 展示层 (locked) | 采集层 (accept_traffic_until) | 一致性 |
|------|-----------------|-------------------------------|--------|
| 支付成功 (立即) | 旧值 (可能锁定) | 已更新 | **不一致** |
| 支付成功 (次日 LockSites 后) | 已更新 | 已更新 | 一致 ✓ |
| 取消订阅 (next_bill_date 前) | 不锁定 | 接受流量 | 一致 ✓ |
| 取消订阅 (next_bill_date 后, LockSites 前) | 不锁定 | 接受流量 | 一致 ✓ |
| 取消订阅 (next_bill_date 后, LockSites 后) | **锁定** | 接受流量 | **不一致** |
| 取消订阅 (accept_traffic_until 后) | 锁定 | **拒绝流量** | 一致 ✓ |
| 超限检测后 | 不锁定 | 接受流量 | 一致 ✓ |
| 宽限期期间 | 不锁定 | 接受流量 | 一致 ✓ |
| 宽限期结束后, LockSites 前 | 不锁定 | 接受流量 | 一致 ✓ |
| 宽限期结束后, LockSites 后 | **锁定** | 接受流量 | **不一致** |

### 6.3 最严重的不一致场景

**场景A: 取消订阅后的 30天窗口**

```
用户视角:
┌─────────────────────────────────────────────────────────────────────────┐
│  我已经取消订阅了，为什么还在收集我的数据？                                │
│                                                                           │
│  时间点:                                                                  │
│  T=0: 取消订阅，收到取消确认邮件                                           │
│  T=15天后: 面板显示"已锁定"，无法查看数据                                  │
│  T=45天后: 数据终于停止采集                                                │
│                                                                           │
│  问题: 面板锁定后还继续采集了30天！                                        │
└─────────────────────────────────────────────────────────────────────────┘
```

**场景B: 支付成功后的 24小时窗口**

```
用户视角:
┌─────────────────────────────────────────────────────────────────────────┐
│  我刚刚支付升级了，为什么面板还是锁定的？                                  │
│                                                                           │
│  时间点:                                                                  │
│  T=0: 支付成功，收到支付确认邮件                                           │
│  T=1小时: 刷新页面，仍显示"已锁定"                                        │
│  T=5小时: 联系客服，客服说"系统可能有延迟"                                 │
│  T=次日: 面板终于恢复正常                                                  │
│                                                                           │
│  问题: 支付成功后需要等待最多24小时才能恢复访问！                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 7. 修正结论与改进建议

### 7.1 关键修正结论

#### 结论1: `subscription_payment_succeeded` 处理不完整

**当前行为**:
- ✅ 更新 `subscription.next_bill_date`
- ✅ 更新 `team.accept_traffic_until`
- ❌ **不调用** `remove_grace_period`
- ❌ **不调用** `SiteLocker.update_for`

**问题**:
- 支付成功后，`locked` 状态要等到**下一次 LockSites (00:00 UTC)** 才会重置
- 最多延迟 **24小时**

**建议修正**:
```elixir
defp handle_subscription_payment_succeeded(params) do
  subscription = ... |> Repo.update!() |> Repo.preload(:team)

  # 完整的后续处理，与 after_subscription_update 一致
  subscription.team
    |> Plausible.Teams.update_accept_traffic_until()
    |> Plausible.Teams.remove_grace_period()  # 新增
    |> Plausible.Teams.maybe_reset_next_upgrade_override()  # 新增
    |> tap(&Plausible.Billing.SiteLocker.update_for/1)  # 新增
    |> maybe_adjust_api_key_limits()  # 新增
end
```

#### 结论2: `subscription_cancelled` 不更新 `accept_traffic_until` 是设计选择，但缺少文档

**当前行为**:
- ✅ 更新 `subscription.status = "deleted"`
- ❌ **不更新** `accept_traffic_until`
- ❌ **不调用** `SiteLocker.update_for`

**分析**:
- 设计意图可能是：已支付的服务应该持续到 `next_bill_date + 30天`
- 但 `next_bill_date` 后 `LockSites` 会锁定面板
- 这造成了 **30天的不一致窗口**

**建议**:
1. 如果是设计选择，需要明确文档说明
2. 如果是 bug，应该在 `subscription_cancelled` 中调用完整的后续处理

#### 结论3: 定时任务调度导致的延迟是系统性问题

**当前调度**:
- `CheckUsage`: 每天 14:00 UTC
- `LockSites`: 每天 00:00 UTC
- `AcceptTrafficUntil` (通知): 每天 08:00 UTC

**问题**:
- 事件发生后，需要等待到下一个任务执行时间
- 没有"即时触发"机制

**建议**:
1. 关键操作后（支付成功、订阅创建/更新）应该**即时调用** `SiteLocker.update_for`
2. 考虑增加任务的执行频率，或使用事件驱动的方式

#### 结论4: 缓存刷新的 `site.updated_at` 限制是 bug

**当前行为**:
- `refresh_updated_recently` 只检查 `site.updated_at`
- 当 `team.accept_traffic_until` 更新时，`site.updated_at` 不更新
- 导致缓存延迟 **15分钟**

**建议修正**:
```elixir
def refresh_updated_recently(opts) do
  recently_updated_query =
    from([s, team in assoc(s, :team)] in base_db_query(),
      order_by: [asc: s.updated_at],
      where: s.updated_at > ago(^15, "minute") or
             team.updated_at > ago(^15, "minute")  # 新增: 同时检查 team.updated_at
    )
  ...
end
```

### 7.2 改进优先级

| 优先级 | 问题 | 影响 | 建议方案 |
|--------|------|------|----------|
| **P0 (立即修复)** | `subscription_payment_succeeded` 不调用 `SiteLocker.update_for` | 支付成功后最多延迟24小时才能访问 | 即时调用完整的后续处理 |
| **P0 (立即修复)** | 缓存刷新不检查 `team.updated_at` | 所有 `team` 字段更新后缓存延迟15分钟 | 修改 `refresh_updated_recently` 查询 |
| **P1 (高优先级)** | `subscription_cancelled` 后续处理不完整 | 取消订阅后30天内采集层和展示层不一致 | 确认设计意图，选择文档或修复 |
| **P2 (中优先级)** | 定时任务每日运行一次，延迟较大 | 所有状态传播都有延迟 | 关键操作后即时触发 |

### 7.3 完整的事件处理流程修正建议

```
理想的事件处理流程:

Paddle Webhook 事件
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  所有订阅相关事件都调用完整的后续处理:                        │
│                                                             │
│  team                                                       │
│    |> update_accept_traffic_until()                        │
│    |> remove_grace_period()     ← 新增到 payment_succeeded │
│    |> reset_next_upgrade_override()  ← 新增                │
│    |> tap(&SiteLocker.update_for/1)   ← 新增到 payment_succeeded │
│    |> adjust_api_key_limits()        ← 新增到 payment_succeeded │
│                                                             │
│  例外: subscription_cancelled                              │
│        - 如果设计意图是"已支付服务应持续"，则保持现状       │
│        - 但需要确保 accept_traffic_until 的值正确          │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  缓存刷新改进:                                               │
│                                                             │
│  refresh_updated_recently 查询条件:                         │
│    site.updated_at > ago(15, "minute") OR                  │
│    team.updated_at > ago(15, "minute")  ← 新增            │
│                                                             │
│  这样 team 字段更新后 30秒内缓存就能刷新                    │
└─────────────────────────────────────────────────────────────┘
```

---

## 8. 代码位置速查

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| 定时任务调度配置 | `config/runtime.exs` | 811-826 |
| `handle_subscription_payment_succeeded` | `lib/plausible/billing/billing.ex` | 112-135 |
| `after_subscription_update` | `lib/plausible/billing/billing.ex` | 219-238 |
| `subscription_cancelled` 测试 | `test/workers/lock_sites_test.exs` | 58-80 |
| `Subscription.active?` | `lib/plausible/billing/subscriptions.ex` | 7-16 |
| `SiteLocker.update_for` | `lib/plausible/billing/site_locker.ex` | 14-49 |
| `check_needs_to_upgrade` | `lib/plausible/teams/billing.ex` | 125-149 |
| 缓存刷新调度 | `lib/plausible/application.ex` | 63-76 |
| `refresh_updated_recently` | `lib/plausible/cache.ex` | 133-146 |
| `CheckUsage` 时机测试 | `test/workers/check_usage_test.exs` | 508-580 |
