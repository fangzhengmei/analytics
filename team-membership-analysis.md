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

## 3. 团队邀请机制

### 3.1 邀请流程
1. **发起邀请**
   - 团队邀请：`Teams.Invitations.invite/4` 针对 `Team` 调用
   - 站点邀请：`Teams.Invitations.invite/4` 针对 `Site` 调用
   - 角色为 `:owner` 时自动转换为站点转移

2. **权限验证** (`check_invitation_permissions/4`)
   - 团队邀请：只有 `:owner` 可以邀请 `:owner`，`:owner` 和 `:admin` 可邀请其他角色
   - 站点邀请：只有团队成员中的 `:owner` 和 `:admin` 可以发起

3. **配额检查** (`check_team_member_limit/3`)
   - 根据订阅计划限制团队成员数量
   - `:owner` 角色不受此限制

4. **重复成员验证** (`ensure_new_membership/3`)
   - 防止邀请已存在的成员
   - 访客可以升级为团队成员

### 3.2 特殊邀请类型
1. **团队邀请** (`accept_team_invitation/2`)
   - 接受后创建完整的团队会员
   - 非访客角色会清理所有访客会员

2. **访客邀请** (`accept_guest_invitation/2`)
   - 接受后创建 `:guest` 角色的团队会员 + 访客会员
   - 保留特定站点的访问权限

## 4. 角色分配与变更

### 4.1 角色权限矩阵
- **团队角色**:
  - `:owner`: 最高权限，可以转让所有权、管理所有成员
  - `:admin`: 可以管理非所有者成员、修改自己的角色（只能降级）
  - `:editor/viewer/billing`: 基础访问权限

- **站点角色**:
  - 通过 `team_member` 或 `guest_member` 区分类型
  - 访客会员有独立的站点级别角色

### 4.2 角色变更规则
1. **团队成员角色变更** (`Memberships.UpdateRole.update/4`)
   - 检查目标角色有效性（排除 `:guest`）
   - 验证操作者权限
   - 确保团队至少有一个所有者
   - 处理 SSO 和双因素认证约束

2. **访客角色变更** (`Memberships.update_role/5`)
   - 仅适用于站点层面
   - 只能将访客升级或降级为 `:editor` 或 `:viewer`

### 4.3 权限检查函数
```elixir
# 团队角色检查
Teams.Memberships.team_role(team, user)

# 站点角色检查（区分团队成员和访客）
Teams.Memberships.site_role(site, user)
# 返回格式: {:ok, {:team_member | :guest_member, role}} 或 {:error, :not_a_member}
```

## 5. 站点转移机制

### 5.1 转移类型
1. **内部转移** (`Transfer.change_team/3`)
   - 将站点从一个团队移动到另一个团队
   - 操作者必须是源团队所有者，同时是目标团队的 `:owner` 或 `:admin`

2. **所有权转移**（通过邀请机制）
   - 邀请特定邮箱为 `:owner` 时触发
   - 创建 `SiteTransfer` 记录等待接受

### 5.2 转移验证流程
1. **基本验证**
   - 检查是否转移到自身团队
   - 验证目标团队权限
   - 检查订阅计划（EE 版本）

2. **配额验证** (`ensure_can_take_ownership/3`)
   - 站点数量限制
   - 团队成员数量限制
   - 页面浏览量限制

### 5.3 转移执行 (`transfer_site_ownership/4`)
1. **事务管理**：所有操作在数据库事务中执行
2. **站点归属更新**：修改 `site.team_id`
3. **访客邀请迁移**：
   - 在目标团队重新创建访客邀请
   - 删除源团队的旧邀请
4. **访客会员迁移**：
   - 在目标团队创建新的访客会员
   - 删除源团队的旧会员
5. **原所有者处理**：
   - 在目标团队添加原所有者为访客
   - 授予 `:editor` 角色
6. **计费调整**（EE 版本）：
   - 更新源团队和目标团队的站点锁定状态
   - 重置合并视图（如果启用）

## 6. 邀请接受与拒绝

### 6.1 统一接受入口 (`Invitations.Accept.accept/3`)
- 根据邀请类型路由到不同处理逻辑：
  - `SiteTransfer` → `Teams.Sites.Transfer.accept/4`
  - `Invitation` → `Teams.Invitations.accept_team_invitation/2`
  - `GuestInvitation` → `Teams.Invitations.accept_guest_invitation/2`

### 6.2 接受过程中的关键逻辑
1. **事务一致性**：所有接受操作在数据库事务中执行
2. **角色保护**：不会降级现有成员的角色
3. **多团队处理**：用户拥有多个团队时需要明确指定目标
4. **无成员转移选项** (`accept_transfer_no_members/3`)
   - 不迁移访客和原所有者
   - 可绕过成员配额限制

### 6.3 拒绝机制 (`Invitations.Reject.reject/2`)
- 清理邀请记录
- 发送拒绝通知邮件（可选）

## 7. 协作关系图

```
┌─────────────────────────────────────────────────────────────┐
│                    团队/站点邀请发起                         │
│                        (Owner/Admin)                        │
└──────────────────────┬──────────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
   团队邀请        站点访客邀请     站点所有者邀请
   (Team)         (Site Guest)     (Site Owner)
        │              │              │
        │              │              ▼
        │              │       站点转移记录
        │              │       (SiteTransfer)
        ▼              ▼              │
   团队邀请记录    访客邀请记录         │
   (Invitation)   (GuestInvitation)   │
        │              │              │
        └──────────────┼──────────────┘
                       ▼
              ┌──────────────────┐
              │   邀请接受入口    │
              │ Invitations.Accept │
              └────────┬─────────┘
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
   团队会员创建    访客会员创建    站点所有权转移
   (Membership)  (GuestMembership) (Teams.Sites.Transfer)
                       │
                       ▼
              ┌──────────────────┐
              │   角色管理服务    │
              │Memberships.UpdateRole │
              └──────────────────┘
```

## 8. 关键代码位置

### 8.1 核心服务模块
- `lib/plausible/teams/invitations.ex` - 邀请和站点转移的核心逻辑
- `lib/plausible/teams/memberships.ex` - 成员关系查询和基础操作
- `lib/plausible/teams/memberships/update_role.ex` - 角色变更服务
- `lib/plausible/teams/sites/transfer.ex` - 站点转移服务
- `lib/plausible/teams/invitations/accept.ex` - 邀请接受统一入口

### 8.2 控制器层
- `lib/plausible_web/controllers/site/membership_controller.ex` - 站点层面的成员管理
- `lib/plausible_web/controllers/invitation_controller.ex` - 邀请接受和拒绝

### 8.3 数据模型
- `lib/plausible/teams/team.ex` - 团队模型
- `lib/plausible/teams/membership.ex` - 成员模型
- `lib/plausible/teams/invitation.ex` - 邀请模型
- `lib/plausible/teams/site_transfer.ex` - 站点转移模型

## 9. 设计亮点

1. **统一的邀请系统**：通过 `Invitations.find_for_user/2` 统一查找所有类型的待处理邀请
2. **事务一致性**：所有状态变更操作都在数据库事务中执行，确保数据一致性
3. **角色保护机制**：接受邀请时不会降级现有成员的角色，保护用户权限
4. **灵活的转移策略**：支持完整转移和无成员转移两种模式
5. **权限链检查**：从团队层面到站点层面的多层权限验证
6. **配额感知**：所有操作都会检查订阅计划的配额限制

## 10. 测试覆盖情况

测试文件位置：
- `test/plausible/teams/invitations/accept_test.exs` - 邀请接受测试
- `test/plausible/teams/sites/transfer_test.exs` - 站点转移测试
- `test/plausible/teams/memberships/update_role_test.exs` - 角色变更测试
- `test/plausible_web/controllers/invitation_controller_test.exs` - 控制器层测试

关键测试场景：
- 各种角色的邀请接受
- 站点所有权转移（含配额限制）
- 团队之间的站点移动
- 访客升级为团队成员
- 多团队用户的转移选择
- 无成员转移模式
