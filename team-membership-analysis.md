# Plausible Analytics 团队成员关系协作分析

## 1. 整体架构概述

Plausible Analytics 的团队成员管理系统采用多层次协作架构，主要围绕三个核心操作：**团队邀请**、**成员角色分配**和**站点转移**。这些操作通过精细的权限控制、事务管理和状态流转机制协同工作。

## 2. 核心数据模型

### 2.1 团队模型 (`Plausible.Teams.Team`)
- 团队是资源管理的基本单位
- 包含 `policy` 嵌入式策略（用于 SSO、双因素认证等企业级功能）
- 与站点、成员、邀请等实体建立关联

```elixir
# 关键关联关系
has_many :sites, Plausible.Site
has_many :team_memberships, Plausible.Teams.Membership
has_many :team_invitations, Plausible.Teams.Invitation
has_one :subscription, Plausible.Billing.Subscription
```

### 2.2 成员模型 (`Plausible.Teams.Membership`)
- 定义了六种角色：`:guest, :viewer, :editor, :admin, :owner, :billing`
- 通过 `is_autocreated` 标记区分自动创建的个人团队成员
- 支持访客会员关联 (`guest_memberships`)

### 2.3 邀请模型
- **团队邀请** (`Teams.Invitation`)：邀请加入整个团队
- **访客邀请** (`Teams.GuestInvitation`)：仅邀请访问特定站点
- **站点转移** (`Teams.SiteTransfer`)：转移站点所有权

## 3. 权限控制体系

### 3.1 发起权限（Who can do what）

| 操作 | 发起者角色 | 说明 |
|------|-----------|------|
| **团队邀请** | 团队 `:owner` 或 `:admin` | `:owner` 可邀请任何角色，`:admin` 可邀请除 `:owner` 外的角色 |
| **站点邀请（访客）** | 团队 `:owner` 或 `:admin` | 通过站点角色判断（必须是 `team_member` 类型） |
| **站点所有权转移** | 团队 `:owner` 或 `:admin` | 通过 `team_member` 类型的 `:owner` 或 `:admin` |
| **站点在团队间移动** | 源团队 `:owner` + 目标团队 `:owner` 或 `:admin` | 双重权限校验 |
| **角色变更（团队）** | 团队 `:owner` 或 `:admin` | 有复杂的权限授予规则 |
| **角色变更（访客）** | 团队 `:owner` 或 `:admin` | 仅在站点层面 |

### 3.2 接受权限

| 操作 | 接受者要求 | 说明 |
|------|-----------|------|
| 团队邀请/访客邀请 | 邮箱匹配 | 无需额外权限 |
| 站点所有权转移 | 目标团队 `:owner` 或 `:admin` | `check_can_transfer_site(new_team, new_owner)` |

## 4. 端到端协作时序

```
发起方（Owner/Admin）                          系统                                接受方
     │                                            │                                    │
     │ 1. 发起邀请/转移请求                         │                                    │
     │───────────────────────────────────────────>│                                    │
     │                                            │                                    │
     │  [权限校验点 1 - 发起权限]                   │                                    │
     │                                            │                                    │
     │  [权限校验点 2 - 配额检查]                   │                                    │
     │                                            │                                    │
     │  [权限校验点 3 - 重复成员检查]               │                                    │
     │                                            │                                    │
     │  [事务点 1 - 创建邀请/转移记录]               │                                    │
     │<───────────────────────────────────────────│                                    │
     │                                            │                                    │
     │                                            │ 2. 发送邀请邮件                    │
     │                                            │───────────────────────────────────>│
     │                                            │                                    │
     │                                            │                                    │
     │                                            │ 3. 接受邀请/转移请求                 │
     │                                            │<───────────────────────────────────│
     │                                            │                                    │
     │                                            │ [权限校验点 4 - 邀请有效性]      │
     │                                            │                                    │
     │                                            │ [权限校验点 5 - 接受者目标权限]      │
     │                                            │                                    │
     │                                            │ [权限校验点 6 - 配额检查（EE）]  │
     │                                            │                                    │
     │                                            │ [事务点 2 - 执行变更 + 回滚点]     │
     │                                            │                                    │
     │                                            │                                    │
     │ 4. 发送接受通知邮件                        │                                    │
     │<───────────────────────────────────────────│                                    │
     │                                            │                                    │
```

## 5. 团队邀请机制

### 5.1 邀请流程（带校验点）

#### 发起阶段校验链

```
发起方 (Team/Admin)
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│ [校验点 1] check_invitation_permissions(team, inviter,  │
│              invitation_role, opts)                         │
│  - Team邀请:                                            │
│    :owner -> 可邀请任何角色（包括:owner）               │
│    :admin -> 可邀请除:owner外的角色                      │
│  - Site邀请:                                             │
│    必须是 team_member 类型的 :owner 或 :admin             │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│ [校验点 2] check_team_member_limit(team, role, email)   │
│  - :owner 角色不受限制                                  │
│  - 其他角色检查订阅配额                                │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│ [校验点 3] ensure_new_membership(site_or_team, invitee,  │
│              role)                                        │
│  - 防止邀请已存在成员                                  │
│  - 访客可升级为团队成员                                │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│ [事务点 1] 创建邀请记录                                │
│  - Team邀请: Teams.Invitation                        │
│  - Site访客邀请: Teams.Invitation + GuestInvitation  │
│  - Site所有权转移: Teams.SiteTransfer                 │
│  ○ 回滚点: 事务回滚，记录不创建                        │
└─────────────────────────────────────────────────────────────┘
```

#### 接受阶段校验链

```
接受方
   │
   ▼
┌─────────────────────────────────────────────────────────────┐
│ [校验点 4] 邀请有效性检查                           │
│  - invitation_id/transfer_id 匹配                     │
│  - 邮箱匹配                                           │
│  - 邀请未过期/未接受                                    │
│  ○ 失败: {:error, :invitation_not_found}          │
└─────────────────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────────────────┐
│ [校验点 5] 目标权限检查（仅站点转移）              │
│  - check_can_transfer_site(new_team, new_owner)     │
│  - 接受者必须是目标团队的 :owner 或 :admin        │
│  ○ 失败: {:error, :permission_denied}                 │
└─────────────────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────────────────┐
│ [校验点 6] 目标配额检查（仅EE版本）                  │
│  - ensure_can_take_ownership(site, new_team, opts)   │
│  - 检查订阅计划有效性                                │
│  - 检查站点数量、成员数量、页面浏览量                │
│  ○ 失败: {:error, :no_plan} 或 {:error, {:over_plan_limits, _}} │
└─────────────────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────────────────┐
│ [事务点 2] 执行变更（关键业务逻辑）                  │
│  所有操作在数据库事务中执行                          │
│  - 创建/更新成员关系                                │
│  - 清理旧邀请/会员                                 │
│  - 发送通知邮件                                    │
│  ○ 回滚点: 全部回滚，状态不变                      │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 特殊邀请类型

#### 团队邀请 (`accept_team_invitation/2`)

**校验链:**
1. 检查用户类型（SSO用户不可接受团队邀请）
2. 创建团队会员（`create_team_membership`)
3. 非访客角色清理访客会员
4. 事务回滚点：任何步骤失败全部回滚

```elixir
# 代码位置: lib/plausible/teams/invitations.ex:290-299
def accept_team_invitation(team_invitation, user) do
  team_invitation = Repo.preload(team_invitation, [:team, :inviter])
  now = NaiveDateTime.utc_now(:second)

  if Plausible.Users.type(user) == :sso do
    {:error, :permission_denied}  # ○ 校验失败点
  else
    do_accept(team_invitation, user, now, guest_invitations: [])
  end
end
```

#### 访客邀请 (`accept_guest_invitation/2`)

**校验链:**
1. 创建 `:guest` 角色的团队会员
2. 创建访客会员
3. 清理对应的访客邀请
4. 事务回滚点

#### 站点所有权转移（通过邀请机制）

**发起校验（InviteToSite.invite/4）:**

```
inviter (发起者)
    │
    ▼
check_invitation_permissions(site, inviter, :owner, opts)
    │
    ├── 检查 site_role(site, inviter)
    │       └── 必须是 {:ok, {:team_member, role}} 且 role in [:owner, :admin]
    │
    ├── check_team_member_limit(team, :owner, email)
    │       └── :owner 角色不受配额限制
    │
    └── ensure_new_membership(site, invitee, :owner)
            └── 允许邀请已存在的站点成员（可升级）
```

**关键测试验证（invite_to_site_test.exs:137-159）:**

```elixir
# ✅ 通过: owner 发起所有权转移
test "sends ownership transfer email when invitation role is owner" do
  inviter = new_user()
  site = new_site(owner: inviter)  # inviter 是 team_member:owner
  assert {:ok, %Teams.SiteTransfer{}} =
           InviteToSite.invite(site, inviter, "vini@plausible.test", :owner)
end

# ✅ 通过: admin 也可以发起所有权转移
test "admin can initiate ownership transfer too" do
  inviter = new_user()
  site = new_site()
  add_member(site.team, user: inviter, role: :admin)  # inviter 是 team_member:admin
  assert {:ok, %Teams.SiteTransfer{}} =
           InviteToSite.invite(site, inviter, "vini@plausible.test", :owner)
end

# ❌ 失败: 访客 editor 不能发起所有权转移
test "only allows owners and admins to transfer ownership" do
  inviter = new_user()
  site = new_site()
  add_guest(site, user: inviter, role: :editor)  # inviter 是 guest_member:editor
  assert {:error, :permission_denied} =
           InviteToSite.invite(site, inviter, "vini@plausible.test", :owner)
end
```

**接受校验（Sites.Transfer.accept/4）:**

```
new_owner (接受者)
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│ 1. maybe_get_team(new_owner, team)              │
│    - 自动获取/创建接受者的团队                   │
│    - 多团队需显式指定                           │
│    ○ 失败: {:error, :multiple_teams}           │
└─────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│ 2. ensure_transfer_valid(site.team, new_team, :owner)  │
│    - 不能转移到自己的团队                               │
│    ○ 失败: {:error, :transfer_to_self}                 │
└─────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│ 3. check_can_transfer_site(new_team, new_owner)    │
│    - 接受者必须是目标团队的 :owner 或 :admin      │
│    ○ 失败: {:error, :permission_denied}               │
└─────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│ 4. ensure_can_take_ownership(site, new_team, opts) │
│    - EE版本检查订阅计划和配额                      │
│    ○ 失败: {:error, :no_plan}                  │
│    ○ 失败: {:error, {:over_plan_limits, _}}    │
└─────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│ 5. accept_site_transfer(site_transfer, new_team)      │
│    - 事务中执行转移                                 │
│    - 更新 site.team_id                              │
│    - 迁移访客邀请/会员                               │
│    - 原所有者降级为访客 editor                       │
│    ○ 回滚点: 全部回滚                               │
└─────────────────────────────────────────────────────────┘
```

## 6. 站点转移机制（两种模式）

### 6.1 模式一：所有权转移（通过邀请）

**适用场景:** 将站点转移给外部用户或其他团队成员

**完整流程:**

```
发起方（Team Owner/Admin）                    系统                        接受方（目标团队 Owner/Admin）
     │                                              │                                    │
     │ 1. 邀请目标邮箱为 :owner                      │                                    │
     │─────────────────────────────────────────────>│                                    │
     │                                              │                                    │
     │ ┌─────────────────────────────────────────┐ │                                    │
     │ │ 发起校验:                                │ │                                    │
     │ │ - team_member 类型的 :owner 或 :admin   │ │                                    │
     │ │ - 不检查成员配额（:owner 特殊）          │ │                                    │
     │ └─────────────────────────────────────────┘ │                                    │
     │                                              │                                    │
     │                                              │ 创建 SiteTransfer 记录            │
     │                                              │                                    │
     │                                              │ 发送转移请求邮件                   │
     │                                              │─────────────────────────────────>│
     │                                              │                                    │
     │                                              │                                    │
     │                                              │ 2. 接受转移                       │
     │                                              │<─────────────────────────────────│
     │                                              │                                    │
     │                                              │ ┌───────────────────────────────┐  │
     │                                              │ │ 接受校验:                   │  │
     │                                              │ │ - 目标团队权限             │  │
     │                                              │ │ - 订阅计划（EE）             │  │
     │                                              │ │ - 配额检查（EE）             │  │
     │                                              │ └───────────────────────────────┘  │
     │                                              │                                    │
     │                                              │ ┌───────────────────────────────┐  │
     │                                              │ │ 事务执行:                     │  │
     │                                              │ │ 1. site.team_id = new_team   │  │
     │                                              │ │ 2. 迁移访客邀请              │  │
     │                                              │ │ 3. 迁移访客会员              │  │
     │                                              │ │ 4. 原所有者 → 访客 editor    │  │
     │                                              │ │ 5. 清理 SiteTransfer        │  │
     │                                              │ │ ○ 全部事务，失败全部回滚     │  │
     │                                              │ └───────────────────────────────┘  │
     │                                              │                                    │
     │ 3. 发送接受通知                             │                                    │
     │<─────────────────────────────────────────────│                                    │
     │                                              │                                    │
```

### 6.2 模式二：团队间移动（change_team）

**适用场景:** 站点在同一个用户的不同团队间移动

**发起者要求:**
- 源团队：必须是 `:owner`（因为需要有站点的所有权
- 目标团队：必须是 `:owner` 或 `:admin`

**完整流程（Sites.Transfer.change_team/3）:**

```
user（源团队 Owner）                              系统
     │                                              │
     │ 1. change_team(site, user, new_team)      │
     │─────────────────────────────────────────────>│
     │                                              │
     │ ┌──────────────────────────────────────────┐     │
     │ │ 发起校验（隐含在业务逻辑中）        │     │
     │ │ - 源团队: 必须拥有该站点           │     │
     │ │ - 目标团队: check_can_transfer_site │     │
     │ │   检查 user 在 new_team 的角色      │     │
     │ │   必须是 :owner 或 :admin        │     │
     │ └──────────────────────────────────────────┘     │
     │                                              │
     │                                              │ ┌──────────────────────────────────┐
     │                                              │ │ transfer_ownership 执行:            │
     │                                              │ │ 1. 确保不是转移到自身团队        │
     │                                              │ │ 2. 检查目标团队权限              │
     │                                              │ │ 3. 检查目标配额（EE）           │
     │                                              │ │ 4. 事务中执行 transfer_site    │
     │                                              │ │    - 站点归属变更             │
     │                                              │ │    - 访客迁移                   │
     │                                              │ │    - 原所有者降级               │
     │                                              │ │ ○ 事务回滚点                   │
     │                                              │ └──────────────────────────────────┘
     │                                              │
     │ 2. 发送团队变更通知                         │
     │<─────────────────────────────────────────────│
     │                                              │
```

**关键测试验证（transfer_test.exs:11-83）:**

```elixir
# ✅ 通过: 源团队 owner + 目标团队 owner
test "changes the team if owner in both teams (CE)" do
  user = new_user()
  site = new_site(owner: user)  # user 是源团队 owner

  another = new_user()
  new_site(owner: another)
  team2 = team_of(another)
  add_member(team2, user: user, role: :owner)  # user 也是目标团队 owner

  assert :ok = Transfer.change_team(site, user, team2)
end

# ✅ 通过: 源团队 owner + 目标团队 admin
test "changes the team if admin in second team" do
  user = new_user()
  site = new_site(owner: user)  # user 是源团队 owner

  another = new_user()
  subscribe_to_growth_plan(another)
  new_site(owner: another)
  team2 = team_of(another)
  add_member(team2, user: user, role: :admin)  # user 是目标团队 admin

  assert :ok = Transfer.change_team(site, user, team2)
end

# ❌ 失败: 源团队 owner + 目标团队 viewer/editor/billing/guest
for role <- Plausible.Teams.Membership.roles() -- [:admin, :owner] do
  test "refuses to change the team if #{role} in second team" do
    # ...
    assert {:error, :permission_denied} = Transfer.change_team(site, user, team2)
  end
end
```

## 7. 角色分配与变更

### 7.1 角色权限矩阵

| 角色 | 团队级别权限 | 站点级别权限 |
|------|-------------|--------------|
| `:owner` | 最高权限，可转让所有权，管理所有成员 | 所有权限 |
| `:admin` | 管理非所有者成员，可降级自己 | 所有权限 |
| `:editor` | 可添加站点 | 可编辑站点配置 |
| `:viewer` | 只读 | 只读 |
| `:billing` | 管理账单 | 无 |
| `:guest` | 仅通过访客会员访问特定站点 | 按访客会员角色 |

### 7.2 团队角色变更规则（Memberships.UpdateRole.update/4）

**校验链:**

```
current_user（操作者）
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│ [校验点 1] check_valid_role(new_role)               │
│  - 目标角色不能是 :guest                            │
│  ○ 失败: {:error, :invalid_role}                 │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│ [校验点 2] 检查目标成员是否存在                  │
│  - get_team_membership(team, user_id)           │
│  ○ 失败: 异常（由上层处理）                     │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│ [校验点 3] 检查操作者在团队中的角色              │
│  - team_role(team, current_user)                │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│ [校验点 4] 角色授予权限检查                     │
│  分两种情况：                                  │
│  1. 授予给自己 (granting_to_self? = true)     │
│     - :owner 可降级为 :admin/:editor/:viewer/:billing │
│     - :admin 可降级为 :editor/:viewer/:billing │
│     ○ 失败: {:error, :permission_denied}    │
│                                                     │
│  2. 授予给他人 (granting_to_self? = false) │
│     - :owner 可授予任何角色                      │
│     - :admin 有复杂规则（不能提升到 :admin     │
│       不能授予 :owner                               │
│     ○ 失败: {:error, :permission_denied}            │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│ [校验点 5] 检查所有者降级保护                      │
│  - 如果目标成员是 :owner 且新角色不是 :owner       │
│  - 必须确保团队至少有一个 :owner                │
│  ○ 失败: {:error, :only_one_owner}              │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│ [校验点 6] SSO/2FA 约束（EE版本）         │
│  - 提升为 :owner 需检查 2FA                  │
│  ○ 失败: {:error, :disabled_2fa}                 │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│ [事务点] 执行角色变更                              │
│  - 发送升级通知邮件（访客 → 团队成员）          │
│  - 清理访客会员（非 :guest 角色）           │
└─────────────────────────────────────────────────────────┘
```

**角色授予规则（can_grant_role_to_other?）:**

```elixir
# Owner 授予他人:
- 可授予任何角色（包括 :owner）

# Admin 授予他人:
- 可授予 :admin → :admin/:editor/:viewer/:billing
- 可授予 :editor → :admin/:editor/:viewer/:billing
- 可授予 :viewer → :admin/:editor/:viewer/:billing
- 可授予 :billing → :billing
- 不可授予 :owner
- 其他角色（:editor/:viewer/:billing/:guest:
- 不可授予任何角色
```

### 7.3 访客角色变更（Memberships.update_role/5）

**校验链:**

```
current_user（操作者）
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│ [校验点 1] 获取目标访客会员                     │
│  - get_guest_membership(site.id, user_id)        │
│  ○ 失败: {:error, :no_guest}               │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│ [校验点 2] 角色授予权限检查                     │
│  - 不能修改自己的角色                          │
│  - :owner 可授予 :editor/:viewer             │
│  - :admin 可授予 :editor/:viewer             │
│  ○ 失败: {:error, :not_allowed}              │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│ [执行点] 更新访客会员角色                        │
│  - 仅更新 guest_membership.role                 │
└─────────────────────────────────────────────────────────┘
```

## 8. 事务与回滚点汇总

### 8.1 事务边界

| 操作 | 事务范围 | 回滚点 |
|------|---------|--------|
| 发起邀请/转移 | `Repo.transaction(fn -> ... end) | 创建邀请记录前的所有校验失败 |
| 接受团队邀请 | `Repo.transaction(fn -> ... end) | 创建会员、清理邀请的任何步骤失败 |
| 接受访客邀请 | `Repo.transaction(fn -> ... end) | 创建访客会员失败 |
| 接受站点转移 | `Repo.transaction(fn -> ... end) | 站点归属变更、访客迁移、原所有者处理的任何步骤失败 |
| 团队间移动站点 | `Repo.transaction(fn -> ... end) | 同上 |

### 8.2 关键回滚场景

```
场景 1: 发起邀请时配额不足
─────────────────────────────
发起者: Team Admin
目标: 邀请新成员
校验: check_team_member_limit 失败
结果: 事务回滚，无记录创建

场景 2: 接受转移时目标团队权限不足
─────────────────────────────
接受者: 目标团队 Viewer
校验: check_can_transfer_site 失败
结果: 接受失败，站点归属不变

场景 3: 转移过程中某一步失败
─────────────────────────────
执行: transfer_site_ownership 中
步骤: 迁移访客会员时数据库错误
结果: 全部回滚，站点归属、访客关系、原所有者状态全部恢复

场景 4: 角色变更时权限不足
─────────────────────────────
操作者: Team Editor
目标: 提升他人为 Admin
校验: check_can_grant_role 失败
结果: 角色不变
```

## 9. 完整端到端协作图

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                         端到端协作时序（以站点所有权转移为例）                          │
└─────────────────────────────────────────────────────────────────────────────────────────┘

时间轴:
  t0        t1          t2           t3           t4           t5           t6
  │         │           │            │            │            │            │
  │ 发起方  │ 系统       │ 系统       │ 接受方  │ 系统       │ 系统       │ 发起方
  │ (Team    │ (校验)  │ (创建记录) │ (接受)   │ (校验)   │ (执行)    │ (通知)
  │ Admin)   │            │            │            │            │            │
  │         │            │            │            │            │            │
  │  1. 邀请 │            │            │            │            │            │            │
  │ 目标邮箱   │            │            │            │            │            │            │
  │ 为:owner  │            │            │            │            │            │            │
  │─────────>│            │            │            │            │            │
  │         │            │            │            │            │            │            │            │
  │         │ [校验点1] │            │            │            │            │
  │         │ - 发起权限    │            │            │            │            │            │
  │         │ check_invitation_permissions │            │            │            │            │
  │         │ 必须是 team_member:owner/admin │            │            │            │            │
  │         │            │            │            │            │            │
  │         │ [校验点2] │            │            │            │            │            │
  │         │ - 配额检查  │            │            │            │            │            │
  │         │ :owner 不受限 │            │            │            │            │            │
  │         │            │            │            │            │            │            │
  │         │ [校验点3] │            │            │            │            │            │
  │         │ - 重复成员    │            │            │            │            │            │
  │         │            │            │            │            │            │            │
  │         │ [事务点1]  │            │            │            │            │            │
  │         │            │            │            │            │            │            │
  │         │            │ 创建       │            │            │            │            │
  │         │            │ SiteTransfer│            │            │            │            │
  │         │            │ 记录       │            │            │            │            │
  │<────────│            │            │            │            │            │
  │         │            │            │            │            │            │            │
  │         │            │ 发送       │            │            │            │            │
  │         │            │ 转移请求   │            │            │            │            │
  │         │            │ 邮件       │───────────>│            │            │            │
  │         │            │            │            │            │            │            │
  │         │            │            │            │            │            │            │
  │         │            │            │ 2. 接受   │            │            │            │
  │         │            │            │ 转移      │            │            │            │
  │         │            │            │───────────>│            │            │
  │         │            │            │            │            │            │            │
  │         │            │            │            │ [校验点4]  │            │            │
  │         │            │            │            │ - 邀请有效性│            │            │
  │         │            │            │            │ ID/邮箱匹配 │            │            │
  │         │            │            │            │            │            │            │
  │         │            │            │            │ [校验点5]  │            │            │
  │         │            │            │            │ - 目标团队  │            │            │
  │         │            │            │            │ 权限检查    │            │            │
  │         │            │            │            │ check_can_ │            │            │
  │         │            │            │            │ transfer_   │            │            │
  │         │            │            │            │ site      │            │            │
  │         │            │            │            │ 必须是     │            │            │
  │         │            │            │            │ :owner/admin │            │            │
  │         │            │            │            │            │            │            │
  │         │            │            │            │ [校验点6]  │            │            │
  │         │            │            │            │ - 配额检查│            │            │
  │         │            │            │            │ 站点/成员/  │            │            │
  │         │            │            │            │ PV 浏览量 │            │            │
  │         │            │            │            │            │            │            │
  │         │            │            │            │ [事务点2]  │            │            │
  │         │            │            │            │            │            │            │
  │         │            │            │            │            │ 1. site.  │            │            │
  │         │            │            │            │ team_id   │            │            │
  │         │            │            │            │            │ = new_team│            │            │
  │         │            │            │            │            │            │            │
  │         │            │            │            │            │ 2. 迁移  │            │            │
  │         │            │            │            │ 访客邀请  │            │            │
  │         │            │            │            │            │            │            │
  │         │            │            │            │            │ 3. 迁移  │            │            │
  │         │            │            │            │            │ 访客会员  │            │            │
  │         │            │            │            │            │            │            │
  │         │            │            │            │            │ 4. 原所有  │            │
  │         │            │            │            │            │ 者降级为  │            │
  │         │            │            │            │            │ 访客     │            │            │
  │         │            │            │            │            │ editor   │            │
  │         │            │            │            │            │            │            │
  │         │            │            │            │            │ ○ 任何   │            │            │
  │         │            │            │            │            │ 步骤失败  │            │
  │         │            │            │            │            │ 全部回滚  │            │
  │         │            │            │            │            │            │            │
  │         │            │            │            │            │ 5. 清理  │            │            │
  │         │            │            │            │            │ SiteTrans- │            │
  │         │            │            │            │            │ fer 记录   │            │            │
  │         │            │            │            │            │            │            │
  │         │            │            │            │<───────────│            │            │
  │         │            │            │            │            │            │            │
  │         │            │            │            │            │ 发送       │            │
  │         │            │            │            │            │            │ 接受通知 │            │
  │<─────────────────────────────────────────────────────────────────────│            │
  │         │            │            │            │            │            │            │
```

## 10. 关键代码位置

### 10.1 核心服务模块

- `lib/plausible/teams/invitations.ex` - 邀请和站点转移的核心逻辑
  - `check_invitation_permissions/4` - 发起权限校验（第587-621行）
  - `check_can_transfer_site/2` - 目标团队权限校验（第576-584行）
  - `ensure_can_take_ownership/3` - 配额校验（第523-548行）
  - `transfer_site_ownership/4` - 转移执行（第401-453行）

- `lib/plausible/teams/memberships.ex` - 成员关系查询和基础操作

- `lib/plausible/teams/memberships/update_role.ex` - 角色变更服务
  - `check_can_grant_role/4` - 角色授予权限校验（第92-106行）
  - `can_grant_role_to_other?/3` - 授予规则（第117-131行）

- `lib/plausible/teams/sites/transfer.ex` - 站点转移服务
  - `accept/4` - 转移接受（第29-46行）
  - `change_team/3` - 团队间移动（第20-27行）

- `lib/plausible/teams/invitations/accept.ex` - 邀请接受统一入口

### 10.2 控制器层

- `lib/plausible_web/controllers/site/membership_controller.ex` - 站点层面的成员管理
  - 插件级权限：`plug AuthorizeSiteAccess, [:owner, :admin]`（第20行）

- `lib/plausible_web/controllers/invitation_controller.ex` - 邀请接受和拒绝

### 10.3 数据模型

- `lib/plausible/teams/team.ex` - 团队模型
- `lib/plausible/teams/membership.ex` - 成员模型
- `lib/plausible/teams/invitation.ex` - 邀请模型
- `lib/plausible/teams/site_transfer.ex` - 站点转移模型

## 11. 设计亮点与权限链总结

### 11.1 统一的邀请系统
- 通过 `Invitations.find_for_user/2` 统一查找所有类型的待处理邀请
- 统一的接受入口 `Invitations.Accept.accept/3` 根据类型路由

### 11.2 双重权限校验
**发起方校验链：
1. 插件级权限（控制器层）
2. 服务级权限（`check_invitation_permissions`）
3. 配额检查
4. 重复成员检查

**接受方校验链：
1. 邀请有效性
2. 目标团队权限（`check_can_transfer_site`）
3. 目标配额检查（`ensure_can_take_ownership`）
4. 事务执行

### 11.3 事务一致性
- 所有状态变更操作都在数据库事务中执行
- 任何步骤失败全部回滚
- 确保数据一致性

### 11.4 角色保护机制
- 接受邀请时不会降级现有成员的角色
- `create_team_membership` 中的冲突处理：`CASE WHEN tm.role = 'guest' THEN new_role ELSE tm.role END

### 11.5 灵活的转移策略
- 完整转移：迁移所有访客和原所有者
- 无成员转移：`skip_site_members_transfer?: true
- 团队间移动：`change_team` 直接移动

## 12. 测试覆盖情况

测试文件位置：
- `test/plausible/teams/invitations/invite_to_site_test.exs` - 站点邀请测试（含所有权转移发起权限）
- `test/plausible/teams/invitations/accept_test.exs` - 邀请接受测试
- `test/plausible/teams/sites/transfer_test.exs` - 站点转移测试
- `test/plausible/teams/memberships/update_role_test.exs` - 角色变更测试
- `test/plausible_web/controllers/invitation_controller_test.exs` - 控制器层测试

关键测试场景：
- 各种角色的邀请发起权限
- 站点所有权转移（含配额限制）
- 团队之间的站点移动
- 访客升级为团队成员
- 多团队用户的转移选择
- 无成员转移模式
- 角色授予规则验证
