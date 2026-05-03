# 订阅、账单和站点额度限制分析报告

## 1. 系统架构概览

### 1.1 核心模块

| 模块 | 文件路径 | 主要职责 |
|------|----------|----------|
| SiteLocker | `lib/plausible/billing/site_locker.ex` | 管理团队锁定状态 |
| Teams.Billing | `lib/plausible/teams/billing.ex` | 团队账单逻辑、限额检查 |
| Quota | `lib/plausible/billing/qouta/quota.ex` | 配额计算、超限判断 |
| GateKeeper | `lib/plausible/site/gate_keeper.ex` | 数据采集入口控制 |
| CheckUsage | `lib/workers/check_usage.ex` | 定期使用量检查任务 |
| LockSites | `lib/workers/lock_sites.ex` | 定期锁定检查任务 |
| GracePeriod | `lib/plausible/teams/grace_period.ex` | 宽限期管理 |

### 1.2 关键数据模型

**Team 表关键字段** (`lib/plausible/teams/team.ex`):
- `locked`: boolean - 团队是否被锁定
- `grace_period`: embedded schema - 宽限期信息
- `accept_traffic_until`: date - 接受流量截止日期

**GracePeriod 嵌入式结构**:
- `end_date`: Date - 宽限期结束日期
- `is_over`: boolean - 宽限期是否已结束
- `manual_lock`: boolean - 是否为手动锁定（企业客户）

---

## 2. 状态传播流程

### 2.1 完整状态转换图

```
订阅状态正常
    ↓
[CheckUsage 任务] - 每日运行，检查页面浏览量
    ↓
连续两个周期超过 限制+10%裕量？
    ↓ 是
[启动7天宽限期] - grace_period.end_date = today + 7
    ↓
[LockSites 任务] - 定期运行，检查是否需要锁定
    ↓
宽限期已过 且 仍超限？
    ↓ 是
[设置 locked=true] - 团队锁定
    ↓
[影响采集层和展示层]
```

### 2.2 详细流程分析

#### 阶段1: 使用量检查 (`CheckUsage`)

**触发条件**: 每日运行，检查订阅状态为 `active`/`past_due`/`deleted` 的团队

**关键逻辑** (`lib/workers/check_usage.ex:149-158`):

```elixir
def check_pageview_usage_two_cycles(subscriber, usage_mod) do
  usage = usage_mod.monthly_pageview_usage(subscriber)
  limit = Teams.Billing.monthly_pageview_limit(subscriber.subscription)

  if Quota.exceeds_last_two_usage_cycles?(usage, limit) do
    {:over_limit, usage}
  else
    {:below_limit, usage}
  end
end
```

**超限判断条件**:
- **连续两个计费周期**（last_cycle + penultimate_cycle）都超过限制
- 使用 **限制 + 10%裕量** 作为判断阈值

**裕量计算** (`lib/plausible/billing/qouta/limits.ex:9-12`):

```elixir
@pageview_allowance_margin 0.1

def pageview_limit_with_margin(limit, margin \\ nil) do
  margin = if margin, do: margin, else: @pageview_allowance_margin
  ceil(limit * (1 + margin))
end
```

#### 阶段2: 宽限期启动

当检测到超限时，执行以下操作 (`lib/workers/check_usage.ex:108-124`):

1. 发送超限邮件通知
2. 启动7天宽限期 (`Plausible.Teams.start_grace_period/1`)

**宽限期结构** (`lib/plausible/teams/grace_period.ex:30-42`):

```elixir
def start_changeset(%Teams.Team{} = team) do
  grace_period = %__MODULE__{
    end_date: Date.shift(Date.utc_today(), day: 7),
    is_over: false,
    manual_lock: false
  }

  Ecto.Changeset.change(team, grace_period: grace_period)
end
```

#### 阶段3: 锁定检查 (`LockSites`)

**触发条件**: 定期运行，检查所有团队

**关键逻辑** (`lib/plausible/billing/site_locker.ex:14-49`):

```elixir
def update_for(team, opts \\ []) do
  team = Teams.with_subscription(team)

  case Teams.Billing.check_needs_to_upgrade(team, usage_mod) do
    {:needs_to_upgrade, :grace_period_ended} ->
      set_lock_status_for(team, true)  # 设置 locked=true

      if team.grace_period.is_over != true do
        Teams.end_grace_period(team)  # 标记宽限期已结束
        send_grace_period_end_email(team, send_email?)
        {:locked, :grace_period_ended_now}
      else
        {:locked, :grace_period_ended_already}
      end

    {:needs_to_upgrade, reason} ->
      if Teams.owned_sites_count(team) > 0 do
        set_lock_status_for(team, true)
        {:locked, reason}
      else
        set_lock_status_for(team, false)
        :unlocked
      end

    :no_upgrade_needed ->
      set_lock_status_for(team, false)
      :unlocked
  end
end
```

**check_needs_to_upgrade 逻辑** (`lib/plausible/teams/billing.ex:125-149`):

```elixir
def check_needs_to_upgrade(team, usage_mod) do
  team = Teams.with_subscription(team)

  cond do
    Plausible.Teams.on_trial?(team) ->
      :no_upgrade_needed

    not Subscriptions.active?(team.subscription) ->
      {:needs_to_upgrade, :no_active_trial_or_subscription}

    Teams.GracePeriod.expired?(team) ->
      revise_pageview_usage(team, usage_mod)  # 再次检查使用量

    true ->
      :no_upgrade_needed
  end
end
```

---

## 3. 限制判断逻辑

### 3.1 页面浏览量限制

#### 双阈值机制

系统采用**双阈值机制**，不同场景使用不同阈值：

| 场景 | 阈值 | 用途 |
|------|------|------|
| 警告通知 | 基础限制（如10k） | 提醒用户接近/超过限制 |
| 强制执行 | 限制 + 10%裕量（如11k） | 启动宽限期、锁定团队 |

#### 超限判断函数

**连续两个周期超限** (`lib/plausible/billing/qouta/quota.ex:140-145`):

```elixir
def exceeds_last_two_usage_cycles?(cycles_usage, allowed_volume) do
  exceeded = exceeded_cycles(cycles_usage, allowed_volume)
  :penultimate_cycle in exceeded && :last_cycle in exceeded
end
```

**超限周期计算** (`lib/plausible/billing/qouta/quota.ex:147-163`):

```elixir
def exceeded_cycles(cycles_usage, allowed_volume, opts \\ []) do
  limit =
    if Keyword.get(opts, :with_margin, true) do
      Limits.pageview_limit_with_margin(allowed_volume)  # 默认使用裕量
    else
      allowed_volume
    end

  Enum.reduce(cycles_usage, [], fn {cycle, %{total: total}}, exceeded_cycles ->
    if below_limit?(total, limit) do
      exceeded_cycles
    else
      exceeded_cycles ++ [cycle]
    end
  end)
end
```

#### 通知类型优先级

根据使用情况和限制，系统显示不同级别的通知 (`lib/plausible/billing/qouta/quota.ex:226-270`):

```
优先级从高到低:
1. trial_ended - 试用结束
2. dashboard_locked - 面板已锁定
3. manual_lock_grace_period_active - 手动锁定宽限期
4. grace_period_active - 宽限期进行中（有倒计时）
5. traffic_exceeded_sustained - 连续两个周期超限
6. traffic_exceeded_current_cycle - 当前周期超限
7. traffic_exceeded_last_cycle - 上一周期超限
8. pageview_approaching_limit - 接近限制（90%）
9. site_and_team_member_limit_reached - 站点和成员都达限
10. site_limit_reached - 站点数量达限
11. team_member_limit_reached - 团队成员达限
```

### 3.2 站点数量限制

**获取站点限制** (`lib/plausible/teams/billing.ex:191-217`):

```elixir
on_ee do
  @site_limit_for_trials 10

  def site_limit(team) do
    if grandfathered_team?(team) do
      :unlimited  # 2021-05-05之前创建的团队无限制
    else
      get_site_limit_from_plan(team)
    end
  end

  defp get_site_limit_from_plan(team) do
    team = Teams.with_subscription(team)

    case Plans.get_subscription_plan(team.subscription) do
      %{site_limit: site_limit} -> site_limit
      :free_10k -> 50
      nil -> @site_limit_for_trials
    end
  end
else
  def site_limit(_team), do: :unlimited  # CE版本无限制
end
```

**检查是否可添加新站点** (`lib/plausible/teams/billing.ex:167-188`):

```elixir
def ensure_can_add_new_site(team) do
  team = Teams.with_subscription(team)

  case Plans.get_subscription_plan(team.subscription) do
    %EnterprisePlan{} ->
      :ok  # 企业计划总是允许添加站点

    _ ->
      usage = site_usage(team)
      limit = site_limit(team)

      if Quota.below_limit?(usage, limit) do
        :ok
      else
        {:error, {:over_limit, limit}}
      end
  end
end
```

### 3.3 团队成员限制

**获取成员限制** (`lib/plausible/teams/billing.ex:229-255`):

```elixir
on_ee do
  @team_member_limit_for_trials 10

  def team_member_limit(team) do
    team = Teams.with_subscription(team)

    case Plans.get_subscription_plan(team.subscription) do
      %{team_member_limit: limit} -> limit
      :free_10k -> :unlimited
      nil -> @team_member_limit_for_trials
    end
  end

  def solo?(team) do
    team_member_limit(team) == 0  # 限制为0表示只能单人使用
  end
else
  def team_member_limit(_team), do: :unlimited
  def solo?(_team), do: false
end
```

### 3.4 流量接受截止日期 (`accept_traffic_until`)

**计算逻辑** (`lib/plausible/teams.ex:279-304`):

```elixir
def accept_traffic_until(team) do
  cond do
    Teams.on_trial?(team) ->
      Date.add(
        team.trial_expiry_date,
        day: Teams.Team.trial_accept_traffic_until_offset_days()  # 14天
      )

    is_nil(team.subscription) ->
      @accept_traffic_until_free  # 2135-01-01（永久）

    Subscriptions.active?(team.subscription) ->
      Date.add(
        team.subscription.next_bill_date,
        day: Teams.Team.subscription_accept_traffic_until_offset_days()  # 30天
      )

    true ->
      @accept_traffic_until_free
  end
end
```

---

## 4. 对采集层的影响

### 4.1 采集入口控制

**GateKeeper 检查逻辑** (`lib/plausible/site/gate_keeper.ex:32-78`):

```elixir
def check(domain, opts \\ []) when is_binary(domain) do
  case policy(domain, opts) do
    {:allow, site} -> {:allow, site}
    other -> {:deny, other}
  end
end

defp policy(domain, opts) do
  with %Site{team: %{accept_traffic_until: accept_traffic_until}} = site <-
         Cache.get(domain, Keyword.get(opts, :cache_opts, [])),
       true <- Plausible.Sites.regular?(site) do
    if not is_nil(accept_traffic_until) and
         Date.after?(Date.utc_today(), accept_traffic_until) do
      :payment_required  # 关键：超过截止日期则拒绝
    else
      check_rate_limit(site, opts)
    end
  else
    _ ->
      @policy_for_non_existing_sites  # :not_found
  end
end
```

### 4.2 采集事件处理

**事件构建和缓冲** (`lib/plausible/ingestion/event.ex:55-80`):

```elixir
def build_and_buffer(%Request{domains: domains} = request, context \\ []) do
  processed_events =
    if spam_referrer?(request) do
      for domain <- domains, do: drop(new(domain, request), :spam_referrer)
    else
      Enum.reduce(domains, [], fn domain, acc ->
        case GateKeeper.check(domain) do  # 关键：检查是否允许
          {:allow, site} ->
            processed =
              domain
              |> new(site, request)
              |> process_unless_dropped(pipeline(), context)

            [processed | acc]

          {:deny, reason} ->
            [drop(new(domain, request), reason) | acc]  # 拒绝则丢弃
        end
      end)
    end

  {dropped, buffered} = Enum.split_with(processed_events, & &1.dropped?)
  {:ok, %{dropped: dropped, buffered: buffered}}
end
```

### 4.3 可能的丢弃原因

```elixir
@type drop_reason() ::
        :bot
        | :spam_referrer
        | GateKeeper.policy()  # :not_found, :block, :throttle, :payment_required
        | :invalid
        | :dc_ip
        | :threat_ip
        | :site_ip_blocklist
        | :site_country_blocklist
        | :site_page_blocklist
        | :site_hostname_allowlist
        | :verification_agent
        | :lock_timeout
        | :no_session_for_engagement
        | :persist_timeout
        | :persist_error
        | :persist_decode_error
```

---

## 5. 对展示层的影响

### 5.1 控制面板访问控制

**StatsController 检查逻辑** (`lib/plausible_web/controllers/stats_controller.ex:51-114`):

```elixir
def stats(%{assigns: %{site: site}} = conn, _params) do
  site = Plausible.Repo.preload(site, :owners)
  site_role = conn.assigns[:site_role]
  can_see_stats? = not Teams.locked?(site.team) or site_role == :super_admin

  cond do
    (stats_start_date && can_see_stats?) || (can_see_stats? && skip_to_dashboard?) ->
      # 正常显示统计数据
      render(conn, "stats.html", ...)

    !stats_start_date && can_see_stats? ->
      # 重定向到验证页面
      redirect(conn, to: Routes.site_path(conn, :verification, site.domain))

    Teams.locked?(site.team) ->
      # 显示锁定页面
      render(conn, "site_locked.html", site: site, ...)
  end
end
```

### 5.2 超级管理员豁免

**关键条件**:
```elixir
can_see_stats? = not Teams.locked?(site.team) or site_role == :super_admin
```

超级管理员即使在团队锁定状态下也能查看统计数据，但会显示警告横幅：

**模板中的警告** (`lib/plausible_web/templates/stats/stats.html.heex:4-12`):

```heex
<%= if Plausible.Teams.locked?(@site.team) do %>
  <div class="w-full px-4 py-4 text-sm font-bold text-center text-yellow-800 bg-yellow-100 rounded-sm transition" ...>
    <p>This dashboard is actually locked. You are viewing it with super-admin access</p>
  </div>
<% end %>
```

### 5.3 布局层通知

**全局锁定通知** (`lib/plausible_web/templates/layout/_notice.html.heex:11`):

```heex
<Notice.dashboard_locked :if={Plausible.Teams.GracePeriod.expired?(@current_team)} />
```

**Dashboard Locked 通知组件** (`lib/plausible_web/components/billing/notice.ex:47-64`):

```heex
def dashboard_locked(assigns) do
  ~H"""
  <aside class="container">
    <.notice title={Plausible.Billing.dashboard_locked_notice_title()} ...>
      Since you've outgrown your current subscription tier, it's time to upgrade to match your growing usage.
      <.link href={Routes.billing_path(PlausibleWeb.Endpoint, :choose_plan)} ...>
        Upgrade now &rarr;
      </.link>
    </.notice>
  </aside>
  """
end
```

---

## 6. 短暂不一致风险分析

### 6.1 风险点汇总

| 风险点 | 位置 | 影响 | 严重程度 |
|--------|------|------|----------|
| 站点缓存延迟 | Site.Cache | 采集层使用旧状态 | 中 |
| 后台任务调度延迟 | CheckUsage/LockSites | 状态更新不及时 | 中 |
| 双状态字段不同步 | locked vs accept_traffic_until | 采集和展示判断不一致 | 高 |
| 宽限期过渡期间 | grace_period 状态变化 | 边界条件处理 | 中 |
| 计费周期切换 | last_cycle / penultimate_cycle | 使用量计算跳变 | 低 |

### 6.2 详细风险分析

#### 风险1: 站点缓存延迟

**问题描述**:
`GateKeeper` 使用 `Site.Cache` 获取站点信息，包括 `team.accept_traffic_until`。如果缓存未及时刷新，可能导致：

- 应该被拒绝的流量继续被接受（缓存了旧的未来日期）
- 应该被接受的流量被错误拒绝（缓存了旧的过期日期）

**相关代码** (`lib/plausible/site/cache.ex:28-34`):

```elixir
@cached_schema_fields ~w(
  id
  domain
  domain_changed_from
  ingest_rate_limit_scale_seconds
  ingest_rate_limit_threshold
)a
```

⚠️ **注意**: 缓存字段列表中**不包含** `accept_traffic_until`，但 `base_db_query` 中 preload 了 `team`，所以实际缓存的数据包含团队信息。

**缓存刷新机制**:
- 缓存是定期刷新的（具体间隔取决于 `Plausible.Cache` 实现）
- 没有主动失效机制

**风险场景**:
1. 团队 `accept_traffic_until` 从未来日期变为过去日期
2. 缓存尚未刷新
3. 采集层仍使用旧的未来日期，继续接受流量
4. 直到缓存刷新后才开始拒绝

#### 风险2: 后台任务调度延迟

**问题描述**:
`CheckUsage` 和 `LockSites` 是周期性后台任务，不是实时执行的。

**任务调度**:
- `CheckUsage`: 每日运行一次（通常在账单日期附近）
- `LockSites`: 定期运行（具体间隔取决于 Oban 配置）

**风险场景**:
1. 团队在早上8点连续两个周期超限
2. `CheckUsage` 任务在凌晨2点已经运行过
3. 宽限期要等到**第二天凌晨**才会启动
4. 存在最多24小时的延迟窗口

**更复杂的场景**:
- 宽限期结束日期是今天
- `LockSites` 任务在凌晨运行时宽限期还没结束
- 宽限期在中午12点结束（按日期计算）
- 锁定要等到**第二天**才会执行

#### 风险3: 双状态字段不同步

**问题描述**:
系统使用两个独立的字段控制不同层面：

| 字段 | 控制层面 | 更新时机 |
|------|----------|----------|
| `accept_traffic_until` | 采集层 | 订阅状态变化时更新 |
| `locked` | 展示层 | `LockSites` 任务运行时更新 |

**潜在不一致场景**:

**场景A: 锁定但仍接受流量**
- `locked = true`（展示层已锁定）
- `accept_traffic_until` 仍是未来日期
- **结果**: 用户无法查看面板，但数据仍在采集

**场景B: 未锁定但已拒绝流量**
- `locked = false`（展示层正常）
- `accept_traffic_until` 已是过去日期
- **结果**: 用户可以查看面板，但新数据不再采集

**相关代码位置**:
- 采集层判断: `gate_keeper.ex:45-59` - 使用 `accept_traffic_until`
- 展示层判断: `stats_controller.ex:110-112` - 使用 `locked?`

#### 风险4: 宽限期过渡期间

**问题描述**:
宽限期有三个相关状态：
1. `grace_period.end_date` - 结束日期
2. `grace_period.is_over` - 是否已结束标记
3. `locked` - 团队锁定标记

**状态转换流程**:
1. 宽限期进行中: `end_date > today`, `is_over = false`, `locked = false`
2. 宽限期结束当天: `end_date == today` → `active?()` 返回 `true`（当天仍算活跃）
3. 宽限期结束后: `end_date < today` → `active?()` 返回 `false`
4. `LockSites` 任务运行后: `is_over = true`, `locked = true`

**关键逻辑** (`lib/plausible/teams/grace_period.ex:82-92`):

```elixir
def active?(%{grace_period: %__MODULE__{end_date: %Date{} = end_date}}) do
  Date.diff(end_date, Date.utc_today()) >= 0  # 当天仍算活跃
end

def active?(%{grace_period: %__MODULE__{manual_lock: true}}) do
  true  # 手动锁定永远"活跃"
end
```

**风险点**:
- `Date.diff(end_date, today) >= 0` 意味着结束日期**当天**仍算活跃
- 如果 `LockSites` 在宽限期结束日期的凌晨运行，`active?()` 仍返回 `true`
- 锁定可能被推迟一天

#### 风险5: 计费周期切换

**问题描述**:
页面浏览量检查使用 `last_cycle` 和 `penultimate_cycle` 两个周期。

**相关代码** (`lib/plausible/teams/billing.ex:364-377`):

```elixir
def monthly_pageview_usage(team, site_ids) do
  team = Teams.with_subscription(team)
  active_subscription? = Subscriptions.active?(team.subscription)

  if active_subscription? and team.subscription.last_bill_date != nil do
    [:current_cycle, :last_cycle, :penultimate_cycle]  # 三个周期
    |> Task.async_stream(fn cycle ->
      {cycle, usage_cycle(team, cycle, site_ids)}
    end)
    |> Enum.into(%{}, fn {:ok, cycle_usage} -> cycle_usage end)
  else
    %{last_30_days: usage_cycle(team, :last_30_days, site_ids)}
  end
end
```

**超限判断** (`lib/plausible/billing/qouta/quota.ex:140-145`):

```elixir
def exceeds_last_two_usage_cycles?(cycles_usage, allowed_volume) do
  exceeded = exceeded_cycles(cycles_usage, allowed_volume)
  :penultimate_cycle in exceeded && :last_cycle in exceeded  # 两个都要超限
end
```

**风险场景**:
在计费周期切换当天：
- 原本 `last_cycle` 和 `penultimate_cycle` 都超限
- 切换后，新的 `last_cycle` 是之前的 `current_cycle`（可能未超限）
- 新的 `penultimate_cycle` 是之前的 `last_cycle`
- **结果**: 可能突然从"连续两个周期超限"变为"只有一个周期超限"

**示例**:
```
账单日期: 每月15号

状态 (14号检查):
- penultimate_cycle: 12/15 - 1/14 (超限: 12k vs 10k限制)
- last_cycle: 1/15 - 2/14 (超限: 11k vs 10k限制)
→ 连续两个周期超限，宽限期进行中

状态 (16号检查):
- penultimate_cycle: 1/15 - 2/14 (超限: 11k vs 10k限制)
- last_cycle: 2/15 - 3/14 (当前周期刚开始: 1k vs 10k限制)
→ 只有一个周期超限，宽限期可能被移除
```

---

## 7. 企业客户特殊处理

### 7.1 企业计划特性

**站点数量限制检查豁免** (`lib/plausible/teams/billing.ex:167-188`):

```elixir
def ensure_can_add_new_site(team) do
  case Plans.get_subscription_plan(team.subscription) do
    %EnterprisePlan{} ->
      :ok  # 总是允许添加站点

    _ ->
      # 检查站点数量限制
  end
end
```

**文档说明** (`lib/plausible/teams/billing.ex:162-166`):

```elixir
@doc """
Enterprise plans are always allowed to add more sites (even when
over limit) to avoid service disruption. Their usage is checked
in a background job instead (see `check_usage.ex`).
"""
```

### 7.2 手动锁定宽限期

**企业客户超限时** (`lib/workers/check_usage.ex:126-147`):

```elixir
def check_enterprise_subscriber(subscriber, usage_mod) do
  pageview_usage = check_pageview_usage_two_cycles(subscriber, usage_mod)
  site_usage = check_site_usage_for_enterprise(subscriber)

  case {pageview_usage, site_usage} do
    {{:below_limit, _}, {:below_limit, _}} ->
      nil

    {{_, pageview_usage}, {_, {site_usage, site_allowance}}} ->
      # 发送内部通知邮件
      Plausible.Teams.start_manual_lock_grace_period(subscriber)  # 手动锁定
  end
end
```

**手动锁定特性** (`lib/plausible/teams/grace_period.ex:44-58`):

```elixir
def start_manual_lock_changeset(%Teams.Team{} = team) do
  grace_period = %__MODULE__{
    end_date: nil,  # 没有结束日期
    is_over: false,
    manual_lock: true  # 标记为手动锁定
  }

  Ecto.Changeset.change(team, grace_period: grace_period)
end
```

**手动锁定活跃判断** (`lib/plausible/teams/grace_period.ex:88-92`):

```elixir
def active?(%{grace_period: %__MODULE__{manual_lock: true}}) do
  true  # 永远返回 true
end
```

### 7.3 企业客户风险点

1. **无结束日期的宽限期**: 手动锁定没有 `end_date`，`active?()` 永远返回 `true`
2. **依赖手动操作**: 需要 CRM 手动解除或确认
3. **状态传播**: `LockSites` 如何处理手动锁定？

**查看 `SiteLocker` 逻辑** (`lib/plausible/billing/site_locker.ex:22-48`):

```elixir
case Teams.Billing.check_needs_to_upgrade(team, usage_mod) do
  {:needs_to_upgrade, :grace_period_ended} ->
    # 锁定逻辑

  {:needs_to_upgrade, reason} ->
    # 其他原因锁定

  :no_upgrade_needed ->
    set_lock_status_for(team, false)  # 解锁
end
```

**`check_needs_to_upgrade` 中的宽限期判断** (`lib/plausible/teams/billing.ex:143-149`):

```elixir
Teams.GracePeriod.expired?(team) ->
  revise_pageview_usage(team, usage_mod)
```

**`expired?` 逻辑** (`lib/plausible/teams/grace_period.ex:106-113`):

```elixir
def expired?(team) do
  if team && team.grace_period, do: !active?(team), else: false
end
```

对于手动锁定:
- `active?()` → `true`
- `expired?()` → `false`
- `check_needs_to_upgrade` → `:no_upgrade_needed`
- `locked` → 被设置为 `false`

**风险**: 企业客户的手动锁定宽限期可能导致 `locked` 状态被错误地设置为 `false`。

---

## 8. 建议和改进

### 8.1 立即关注的问题

1. **双状态字段一致性**
   - 建议统一使用单一数据源控制采集和展示
   - 或确保两个字段的更新在同一事务中

2. **缓存刷新机制**
   - 建议在 `accept_traffic_until` 更新时主动失效缓存
   - 或缩短缓存刷新间隔

3. **企业客户手动锁定**
   - 建议审查 `LockSites` 对手动锁定的处理逻辑
   - 考虑添加额外的锁定条件

### 8.2 监控建议

1. **关键指标监控**:
   - `locked` 状态变化次数
   - `accept_traffic_until` 过期的团队数量
   - 被 `GateKeeper` 拒绝的流量（`payment_required` 原因）

2. **一致性检查**:
   - 定期检查 `locked` 和 `accept_traffic_until` 的一致性
   - 检查宽限期状态与锁定状态的对应关系

### 8.3 测试建议

1. **边界条件测试**:
   - 计费周期切换当天的超限判断
   - 宽限期结束日期的判断
   - 缓存刷新前后的状态变化

2. **竞态条件测试**:
   - 后台任务运行时的订阅状态变化
   - 并发更新锁定状态

---

## 9. 关键代码位置速查

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| 采集层流量控制 | `lib/plausible/site/gate_keeper.ex` | 45-59 |
| 展示层访问控制 | `lib/plausible_web/controllers/stats_controller.ex` | 51-114 |
| 使用量超限判断 | `lib/plausible/billing/qouta/quota.ex` | 140-145 |
| 页面浏览量裕量 | `lib/plausible/billing/qouta/limits.ex` | 9-12 |
| 宽限期启动 | `lib/workers/check_usage.ex` | 108-124 |
| 团队锁定 | `lib/plausible/billing/site_locker.ex` | 14-49 |
| 宽限期活跃判断 | `lib/plausible/teams/grace_period.ex` | 82-92 |
| accept_traffic_until计算 | `lib/plausible/teams.ex` | 279-304 |
