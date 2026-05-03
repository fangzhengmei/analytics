# 邮件报表链路深度分析报告（第三轮）

## 1. 概述

本报告是对邮件报表系统的第三轮深度分析，重点关注三个核心问题：
1. **退订链接邮箱参数特殊字符处理**：分析特殊字符场景下的误退订风险
2. **GET请求执行写操作的触发风险**：分析CSRF、爬虫预加载等安全风险
3. **调度入队后团队锁定状态变化**：分析调度与发送之间的权限边界一致性

## 2. 退订链接邮箱参数特殊字符处理分析

### 2.1 退订链接构建代码

**构建位置**
- **位置**: `lib/workers/send_email_report.ex:64-67`
- **实现**:
  ```elixir
  defp unsubscribe_link(site, email, interval) do
    PlausibleWeb.Endpoint.url() <>
      "/sites/#{URI.encode_www_form(site.domain)}/#{interval}-report/unsubscribe?email=#{email}"
  end
  ```

### 2.2 关键发现：编码不一致

**对比分析**

| 参数 | 编码处理 | 代码位置 |
|------|----------|----------|
| `site.domain` | ✅ 使用 `URI.encode_www_form()` | `URI.encode_www_form(site.domain)` |
| `email` | ❌ **未编码** | 直接拼接 `#{email}` |

**代码证据**
```elixir
"/sites/#{URI.encode_www_form(site.domain)}/#{interval}-report/unsubscribe?email=#{email}"
#         ↑ 已编码                                      ↑ 未编码！
```

### 2.3 特殊字符风险场景分析

**场景1: 邮箱包含 `+` 符号**

**问题邮箱示例**
```
user+tag@example.com
```

**构建的URL（未编码）**
```
https://plausible.example.com/sites/example.com/weekly-report/unsubscribe?email=user+tag@example.com
```

**问题分析**
- `+` 符号在 URL 查询参数中会被解析为**空格**
- 实际解析结果: `user tag@example.com`（注意空格）
- 与原始邮箱 `user+tag@example.com` 不匹配
- **可能导致误退订或退订失败**

**场景2: 邮箱包含 `&` 符号**

**问题邮箱示例**
```
user&admin@example.com
```

**构建的URL（未编码）**
```
https://plausible.example.com/sites/example.com/weekly-report/unsubscribe?email=user&admin@example.com
```

**问题分析**
- `&` 符号会被解析为**参数分隔符**
- 实际解析结果:
  - `email` = `user`
  - `admin@example.com` = 被解析为新的参数名（无值）
- **邮箱被截断为 `user`**
- **可能导致完全错误的退订操作**

**场景3: 邮箱包含 `=` 符号**

**问题邮箱示例**
```
user=test@example.com
```

**构建的URL（未编码）**
```
https://plausible.example.com/sites/example.com/weekly-report/unsubscribe?email=user=test@example.com
```

**问题分析**
- `=` 符号会被解析为**参数值分隔符**
- 实际解析结果可能混乱
- 取决于具体的 URL 解析器实现

**场景4: 邮箱包含 `?` 符号**

**问题邮箱示例**
```
user?tag@example.com
```

**构建的URL（未编码）**
```
https://plausible.example.com/sites/example.com/weekly-report/unsubscribe?email=user?tag@example.com
```

**问题分析**
- `?` 符号会被解析为**查询字符串开始**
- 后续字符可能被错误解析

**场景5: 邮箱包含中文或特殊字符**

**问题邮箱示例**
```
用户@example.com
测试+标签@example.com
```

**问题分析**
- 非 ASCII 字符需要 URL 编码
- 未编码可能导致:
  - URL 解析错误
  - 乱码
  - 退订失败或误退订

### 2.4 退订时的参数处理

**控制器代码**
- **位置**: `lib/plausible_web/controllers/unsubscribe_controller.ex:6-14`
- **实现**:
  ```elixir
  def weekly_report(conn, %{"domain" => domain, "email" => email}) do
    site = Repo.get_by(Plausible.Site, domain: domain)
    weekly_report = site && Repo.get_by(WeeklyReport, site_id: site.id)

    if weekly_report do
      weekly_report
      |> WeeklyReport.remove_recipient(email)
      |> Repo.update!()
    end
    # ...
  end
  ```

**关键问题**
- 控制器直接使用从 URL 解析出的 `email` 参数
- 没有进行 URL 解码的反向操作
- 如果原始邮箱是 `user+tag@example.com`：
  1. 构建 URL 时：`user+tag@example.com`（未编码）
  2. URL 解析时：`+` 被解析为空格 → `user tag@example.com`
  3. 退订时：尝试移除 `user tag@example.com`
  4. 实际收件人列表中是 `user+tag@example.com`
  5. **结果：退订失败！**

### 2.5 收件人移除逻辑

**WeeklyReport.remove_recipient/2**
- **位置**: `lib/plausible/site/weekly_report.ex:24-27`
- **实现**:
  ```elixir
  def remove_recipient(report, recipient) do
    report
    |> change(recipients: List.delete(report.recipients, recipient))
  end
  ```

**MonthlyReport.remove_recipient/2**
- **位置**: `lib/plausible/site/monthly_report.ex:24-27`
- **实现**: 与周报完全相同

**问题分析**
- 使用 `List.delete/2` 进行**精确匹配**
- 如果邮箱因为编码问题被解析为不同的字符串：
  - `List.delete(["user+tag@example.com"], "user tag@example.com")` → **不删除任何元素**
  - **退订操作静默失败**
  - 用户点击退订但仍然会收到后续邮件

### 2.6 特殊字符处理总结

| 特殊字符 | URL 中的含义 | 未编码的影响 | 风险等级 |
|----------|--------------|--------------|----------|
| `+` | 空格 | 邮箱被解析为包含空格 | 🔴 高 |
| `&` | 参数分隔符 | 邮箱被截断 | 🔴 高 |
| `=` | 参数值分隔符 | 解析混乱 | 🟡 中 |
| `?` | 查询字符串开始 | URL 结构破坏 | 🟡 中 |
| `%` | 编码转义字符 | 解析错误 | 🟡 中 |
| 非 ASCII | 需要编码 | 乱码、解析失败 | 🟡 中 |
| `#` | 片段标识符 | 后续内容被忽略 | 🔴 高 |
| `/` | 路径分隔符 | 路径解析错误 | 🟡 中 |

**关键发现**
1. **编码不一致**: `site.domain` 编码了，但 `email` 没有编码
2. **精确匹配**: 退订使用 `List.delete/2` 精确匹配
3. **静默失败**: 编码问题导致退订失败时，用户看到"退订成功"但实际未成功

**潜在改进建议**
1. **统一编码**: 对 `email` 参数也使用 `URI.encode_www_form()`
2. **或使用签名**: 生成退订链接时使用签名，而不是直接传递邮箱
3. **添加验证**: 退订成功后发送确认邮件

## 3. GET请求执行写操作的触发风险分析

### 3.1 路由配置分析

**退订路由**
- **位置**: `lib/plausible_web/router.ex:725-726`
- **实现**:
  ```elixir
  get "/sites/:domain/weekly-report/unsubscribe", UnsubscribeController, :weekly_report
  get "/sites/:domain/monthly-report/unsubscribe", UnsubscribeController, :monthly_report
  ```

**所属 Scope**
- **位置**: `lib/plausible_web/router.ex:568-569`
- **实现**:
  ```elixir
  scope "/", PlausibleWeb do
    pipe_through [:browser, :csrf]
    # ... 退订路由在这个 scope 内
  end
  ```

### 3.2 CSRF 保护分析

**CSRF Pipeline**
- **位置**: `lib/plausible_web/router.ex:54-56`
- **实现**:
  ```elixir
  pipeline :csrf do
    plug :protect_from_forgery
  end
  ```

**关键问题**
- `Plug.CSRFProtection` 只对**非 GET 请求**生效
- **GET、HEAD、OPTIONS 请求默认不受 CSRF 保护**
- 退订路由使用 `GET` 方法，**CSRF 保护不生效**

**证据**: Plug.CSRFProtection 文档说明
> 此 Plug 会自动检查除 GET、HEAD 和 OPTIONS 之外的所有请求的 CSRF 令牌。

### 3.3 触发风险场景分析

**场景1: CSRF 攻击**

**攻击原理**
1. 攻击者创建恶意网站
2. 在页面中嵌入:
   ```html
   <img src="https://plausible.example.com/sites/example.com/weekly-report/unsubscribe?email=victim@example.com" />
   ```
3. 当受害者访问攻击者的网站时
4. 浏览器自动加载图片
5. **退订请求被触发**

**影响**
- 攻击者可以任意退订任意邮箱
- 无需受害者交互（只需访问页面）
- **完全匿名的攻击**

**场景2: 邮件客户端/浏览器预加载**

**问题描述**
- 某些邮件客户端会预加载邮件中的链接
- 某些浏览器/安全软件会预检查链接
- 为了"安全"或"加速"而提前请求

**影响**
- 用户打开邮件时
- 邮件客户端自动请求退订链接
- **用户还没点击，退订就已经执行**

**场景3: 爬虫/搜索引擎索引**

**问题描述**
- 搜索引擎爬虫可能发现退订链接
- 虽然 `robots.txt` 可能排除，但不能保证
- 某些爬虫可能不遵守 `robots.txt`

**影响**
- 爬虫访问退订链接
- **无意退订**

**场景4: 浏览器扩展/安全软件**

**问题描述**
- 某些浏览器扩展会检查页面链接
- 安全软件可能"检测"链接
- 可能主动请求链接以验证安全性

**影响**
- 扩展/软件请求退订链接
- **无意退订**

### 3.4 对比：其他写操作的 HTTP 方法

**其他报告操作的路由**
- **位置**: `lib/plausible_web/router.ex:686-703`
- **实现**:
  ```elixir
  post "/sites/:domain/weekly-report/enable", SiteController, :enable_weekly_report
  post "/sites/:domain/weekly-report/disable", SiteController, :disable_weekly_report
  post "/sites/:domain/weekly-report/recipients", SiteController, :add_weekly_report_recipient

  delete "/sites/:domain/weekly-report/recipients/:recipient",
         SiteController,
         :remove_weekly_report_recipient
  ```

**对比分析**

| 操作 | HTTP 方法 | CSRF 保护 | 风险 |
|------|-----------|-----------|------|
| 启用周报 | `POST` | ✅ 保护 | 低 |
| 禁用周报 | `POST` | ✅ 保护 | 低 |
| 添加收件人 | `POST` | ✅ 保护 | 低 |
| 移除收件人（管理界面） | `DELETE` | ✅ 保护 | 低 |
| **退订（邮件链接）** | `GET` | ❌ **无保护** | **高** |

**关键发现**
- 管理界面的写操作都使用 `POST` 或 `DELETE`
- 都受 CSRF 保护
- **只有退订操作使用 `GET`，无 CSRF 保护**

### 3.5 控制器执行写操作

**退订控制器代码**
- **位置**: `lib/plausible_web/controllers/unsubscribe_controller.ex:10-14`
- **实现**:
  ```elixir
  if weekly_report do
    weekly_report
    |> WeeklyReport.remove_recipient(email)
    |> Repo.update!()  # 写操作！
  end
  ```

**问题分析**
- `GET` 请求应该是**安全的**（不修改服务器状态）
- `GET` 请求应该是**幂等的**（多次请求结果相同）
- 但这里的 `GET` 请求执行了 `Repo.update!()`
- **违反了 REST 设计原则**

### 3.6 GET 请求写操作风险总结

| 风险类型 | 触发方式 | 影响程度 | 可能性 |
|----------|----------|----------|--------|
| CSRF 攻击 | 恶意网站嵌入图片 | 🔴 高 | 🟡 中 |
| 邮件客户端预加载 | 自动预加载链接 | 🔴 高 | 🟡 中 |
| 爬虫索引 | 搜索引擎爬虫 | 🟡 中 | 🟢 低 |
| 浏览器扩展 | 安全软件检查 | 🟡 中 | 🟡 中 |
| 无意点击 | 用户误操作 | 🟡 中 | 🟡 中 |

**REST 原则违反**
- ❌ `GET` 请求执行写操作
- ❌ 非安全的 `GET` 请求
- ❌ 非幂等的 `GET` 请求

**潜在改进建议**
1. **使用 POST 方法**: 退订应该使用 POST 或 DELETE
2. **添加确认页面**: 点击链接后显示确认页面，用户再次确认才执行退订
3. **使用一次性令牌**: 退订链接包含一次性签名令牌，验证后才能执行
4. **限制访问来源**: 检查 Referer 头（但不完全可靠）

## 4. 调度入队后团队锁定状态变化的权限边界分析

### 4.1 调度与发送的时间线

**完整流程**
```
┌─────────────────────────────────────────────────────────────────┐
│                        时间线分析                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  调度时间点                      发送时间点                      │
│       │                              │                          │
│       ▼                              ▼                          │
│  ┌─────────┐    时间窗口      ┌─────────┐                     │
│  │  调度   │────────────────▶│  发送   │                     │
│  │ Schedule│    (可能很长)    │  Send   │                     │
│  └─────────┘                  └─────────┘                     │
│       │                              │                          │
│       │  检查: not t.locked          │  检查: ???              │
│       │                              │                          │
│       ▼                              ▼                          │
│  团队状态: 未锁定              团队状态: ??? (可能已锁定)       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 调度时的权限检查

**调度代码**
- **位置**: `lib/workers/schedule_email_reports.ex:26-39` (周报)
- **实现**:
  ```elixir
  sites =
    Repo.all(
      from s in Plausible.Site,
        inner_join: t in assoc(s, :team),
        join: wr in Plausible.Site.WeeklyReport,
        on: wr.site_id == s.id,
        left_join: job in subquery(weekly_jobs),
        on:
          fragment("(? -> 'site_id')::int", job.args) == s.id and
            job.state not in ["completed", "discarded"],
        where: is_nil(job),
        where: not t.locked,  # 🔴 关键检查：团队未锁定
        preload: [weekly_report: wr]
    )
  ```

**月报调度**
- **位置**: `lib/workers/schedule_email_reports.ex:67-80`
- **实现**: 与周报完全相同，也包含 `where: not t.locked`

**调度时的检查**
| 检查项 | 代码位置 | 说明 |
|--------|----------|------|
| 团队锁定 | `where: not t.locked` | 只调度未锁定的团队 |
| 报告配置存在 | `join: wr in WeeklyReport` | 必须有周报配置 |
| 无待处理任务 | `left_join: job ... where: is_nil(job)` | 避免重复调度 |

### 4.3 发送时的权限检查

**发送代码**
- **位置**: `lib/workers/send_email_report.ex:17-28`
- **实现**:
  ```elixir
  site =
    from(s in Plausible.Site,
      where: s.id == ^site_id,
      inner_join: r in assoc(s, ^report_type),
      inner_join: t in assoc(s, :team),  # 只是 join，没有检查 locked！
      preload: [{^report_type, r}, {:team, t}]
    )
    |> Repo.one()

  with %Plausible.Site{} <- site,
       %{} = report <- Map.fetch!(site, report_type),
       true <- ok_to_send?(site) do  # 检查 ok_to_send?
    # 发送邮件...
  end
  ```

**ok_to_send? 函数**
- **位置**: `lib/workers/send_email_report.ex:201-209`
- **实现**:
  ```elixir
  on_ee do
    defp ok_to_send?(site) do
      Plausible.Sites.regular?(site) or
        (Plausible.Sites.consolidated?(site) and
           Plausible.ConsolidatedView.ok_to_display?(site.team))
    end
  else
    defp ok_to_send?(_site), do: always(true)
  end
  ```

**关键对比分析**

| 检查项 | 调度时 (Schedule) | 发送时 (Send) | 一致性 |
|--------|-------------------|---------------|--------|
| 团队锁定 (`team.locked`) | ✅ 检查 `where: not t.locked` | ❌ **未检查** | ❌ 不一致 |
| 报告配置存在 | ✅ `join: WeeklyReport` | ✅ `inner_join: r in assoc(s, ^report_type)` | ✅ 一致 |
| 站点类型 (regular/consolidated) | ❌ 未检查 | ✅ `ok_to_send?` 检查 | ❌ 不一致 |
| 合并视图权限 | ❌ 未检查 | ✅ `ConsolidatedView.ok_to_display?` | ❌ 不一致 |

### 4.4 时间窗口分析

**周报时间窗口**

调度时间计算
- **位置**: `lib/workers/schedule_email_reports.ex:51-56`
- **实现**:
  ```elixir
  def monday_9am(timezone) do
    DateTime.now!(timezone)
    |> DateTime.shift(week: 1)           # 下周
    |> Plausible.Times.beginning_of_week() # 周一
    |> DateTime.shift(hour: 9)              # 早上9点
  end
  ```

**时间窗口**
- **最小窗口**: 约 1 秒（如果调度时正好是周一 8:59:59）
- **最大窗口**: **约 7 天**（如果调度时是周二早上）

**月报时间窗口**

调度时间计算
- **位置**: `lib/workers/schedule_email_reports.ex:92-97`
- **实现**:
  ```elixir
  def first_of_month_9am(timezone) do
    DateTime.now!(timezone)
    |> DateTime.shift(month: 1)              # 下月
    |> Plausible.Times.beginning_of_month()  # 1号
    |> DateTime.shift(hour: 9)                 # 早上9点
  end
  ```

**时间窗口**
- **最小窗口**: 约 1 秒
- **最大窗口**: **约 30-31 天**（如果调度时是 2 号早上）

### 4.5 问题场景分析

**场景: 调度后团队被锁定**

**时间线**
```
Day 1 (周一):
  - 调度任务执行
  - 检查团队状态: 未锁定
  - 创建发送任务， scheduled_at: 下周一 9:00

Day 3 (周三):
  - 团队因欠费/违规被锁定
  - team.locked = true

Day 8 (下周一 9:00):
  - 发送任务执行
  - ❌ 没有检查 team.locked
  - ✅ 检查 ok_to_send? (站点类型)
  - 📧 邮件仍然发送！
```

**问题分析**

调度时的假设
```elixir
# 调度时检查
where: not t.locked  # 假设：发送时团队仍然未锁定
```

发送时的实际
```elixir
# 发送时没有检查
# 团队可能已经锁定！
```

**权限边界破坏**

| 检查项 | 调度时 | 发送时 | 实际结果 |
|--------|--------|--------|----------|
| 团队锁定 | 未锁定 | 已锁定 | 邮件仍然发送 |
| 预期行为 | 不发送 | 不发送 | ❌ 实际发送 |

### 4.6 ok_to_send? 的检查内容

**企业版检查**
- **位置**: `lib/workers/send_email_report.ex:201-206`
- **实现**:
  ```elixir
  defp ok_to_send?(site) do
    Plausible.Sites.regular?(site) or
      (Plausible.Sites.consolidated?(site) and
         Plausible.ConsolidatedView.ok_to_display?(site.team))
  end
  ```

**检查内容**
1. `Sites.regular?(site)` - 站点是否是常规站点
2. `Sites.consolidated?(site)` - 站点是否是合并视图
3. `ConsolidatedView.ok_to_display?(site.team)` - 合并视图的团队是否有权限

**关键缺失**
- ❌ **没有检查 `team.locked`**
- 团队锁定是更高优先级的权限控制

### 4.7 团队锁定的影响

**团队锁定的含义**
- 团队锁定通常意味着:
  - 订阅过期
  - 违规操作
  - 主动暂停
  - 其他原因

**预期行为**
- 锁定的团队不应该接收任何邮件
- 这是调度时检查 `not t.locked` 的初衷

**实际行为**
- 调度后锁定的团队**仍然**会收到邮件
- 权限边界在调度和发送之间被破坏

### 4.8 权限边界一致性总结

| 维度 | 调度时 | 发送时 | 一致性 | 风险 |
|------|--------|--------|--------|------|
| 团队锁定 | ✅ 检查 | ❌ 未检查 | ❌ 不一致 | 🔴 高 |
| 站点类型 | ❌ 未检查 | ✅ 检查 | ❌ 不一致 | 🟡 中 |
| 报告配置 | ✅ 检查 | ✅ 检查 | ✅ 一致 | 低 |
| 时间窗口 | - | - | - | 周报 7 天，月报 30 天 |

**关键发现**
1. **团队锁定检查不一致**: 调度时检查，发送时不检查
2. **时间窗口很长**: 周报最多 7 天，月报最多 30 天
3. **权限边界可能被绕过**: 调度后锁定的团队仍然会收到邮件
4. **其他检查也不一致**: 站点类型检查只在发送时执行

**潜在改进建议**
1. **发送时再次检查团队锁定**: 在 `ok_to_send?` 中添加 `not team.locked` 检查
2. **或统一检查时机**: 考虑只在发送时检查所有权限
3. **添加任务取消机制**: 团队锁定时，取消所有待执行的发送任务
4. **缩短时间窗口**: 考虑更频繁的调度，减少状态变化的影响

## 5. 三轮分析对比总结

### 5.1 问题演进

| 轮次 | 重点分析 | 关键发现 |
|------|----------|----------|
| 第一轮 | 权限边界、数据查询、发送调度、失败处理 | 基础架构理解 |
| 第二轮 | 退订入口、成员/非成员差异、失败一致性 | 发现退订无保护、部分失败无感知 |
| 第三轮 | 邮箱编码、GET风险、调度/发送权限差 | **发现编码不一致、CSRF风险、权限边界破坏** |

### 5.2 高优先级问题汇总

| 问题 | 位置 | 风险等级 | 建议优先级 |
|------|------|----------|------------|
| 退订链接邮箱未编码 | `send_email_report.ex:66` | 🔴 高 | P0 |
| 退订使用 GET 方法 | `router.ex:725-726` | 🔴 高 | P0 |
| 发送时未检查团队锁定 | `send_email_report.ex:201-209` | 🔴 高 | P0 |
| 部分邮件失败无感知 | `send_email_report.ex:36-50` | 🔴 高 | P1 |
| 退订入口无身份验证 | `unsubscribe_controller.ex` | 🟡 中 | P1 |

### 5.3 权限边界时间线

```
┌──────────────────────────────────────────────────────────────────────┐
│                      完整权限边界时间线                                │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  调度时间点                    发送时间点                             │
│       │                            │                                 │
│       ▼                            ▼                                 │
│  ┌─────────┐   时间窗口      ┌─────────┐                           │
│  │ Schedule│────────────────▶│  Send   │                           │
│  └─────────┘  (7-30天)      └─────────┘                           │
│       │                            │                                 │
│       │  检查:                      │  检查:                         │
│       │    ✅ not t.locked          │    ❌ 未检查 team.locked      │
│       │    ✅ 报告配置存在          │    ✅ ok_to_send? (站点类型)  │
│       │    ❌ 站点类型              │    ✅ 报告配置存在            │
│       │                            │                                 │
│       │  风险:                      │  风险:                          │
│       │    无（检查时状态正确）     │    🔴 团队可能已锁定           │
│       │                            │    🔴 权限边界被绕过           │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

## 6. 关键文件索引（第三轮补充）

| 功能模块 | 文件路径 | 主要职责 | 问题 |
|---------|---------|---------|------|
| 退订链接构建 | `lib/workers/send_email_report.ex:64-67` | 生成退订 URL | 邮箱未编码 |
| 退订控制器 | `lib/plausible_web/controllers/unsubscribe_controller.ex` | 处理退订请求 | GET 执行写操作 |
| 退订路由 | `lib/plausible_web/router.ex:725-726` | 路由配置 | 使用 GET 方法 |
| 调度团队锁定检查 | `lib/workers/schedule_email_reports.ex:37,78` | 调度时检查 | 仅调度时检查 |
| 发送权限检查 | `lib/workers/send_email_report.ex:201-209` | 发送时检查 | 未检查团队锁定 |
| 收件人移除 | `lib/plausible/site/weekly_report.ex:24-27` | 移除收件人 | 精确匹配 |

## 7. 建议修复方案汇总

### 7.1 退订链接编码问题

**问题**: 邮箱参数未进行 URL 编码

**修复方案**
```elixir
# 修改前
defp unsubscribe_link(site, email, interval) do
  PlausibleWeb.Endpoint.url() <>
    "/sites/#{URI.encode_www_form(site.domain)}/#{interval}-report/unsubscribe?email=#{email}"
end

# 修改后
defp unsubscribe_link(site, email, interval) do
  PlausibleWeb.Endpoint.url() <>
    "/sites/#{URI.encode_www_form(site.domain)}/#{interval}-report/unsubscribe?email=#{URI.encode_www_form(email)}"
end
```

### 7.2 GET 请求写操作风险

**问题**: GET 请求执行写操作，无 CSRF 保护

**修复方案选项**

**选项A: 添加确认页面（推荐）**
- 点击退订链接显示确认页面
- 用户点击"确认退订"按钮才执行写操作
- 确认按钮使用 POST 方法，受 CSRF 保护

**选项B: 使用一次性令牌**
- 生成退订链接时添加签名令牌
- 令牌包含: 邮箱、过期时间、签名
- 验证令牌有效性后才执行退订

**选项C: 改为 POST 方法**
- 路由改为 `post`
- 邮件中使用表单或 JavaScript 提交
- 但邮件客户端可能不支持 JavaScript

### 7.3 权限边界一致性

**问题**: 调度后团队锁定，邮件仍然发送

**修复方案**

**方案A: 发送时再次检查团队锁定（推荐）**
```elixir
# 修改 ok_to_send? 函数
defp ok_to_send?(site) do
  # 添加团队锁定检查
  not site.team.locked and (
    Plausible.Sites.regular?(site) or
      (Plausible.Sites.consolidated?(site) and
         Plausible.ConsolidatedView.ok_to_display?(site.team))
  )
end
```

**方案B: 团队锁定时取消待处理任务**
- 团队锁定时，查询所有待执行的发送任务
- 取消或标记为 discarded

**方案C: 统一检查时机**
- 调度时不检查权限
- 只在发送时检查所有权限
- 更简单、更一致

### 7.4 修复优先级建议

| 优先级 | 问题 | 修复难度 | 影响范围 |
|--------|------|----------|----------|
| P0 | 邮箱参数编码 | 极低 | 所有包含特殊字符的邮箱 |
| P0 | 团队锁定一致性 | 低 | 所有锁定的团队 |
| P0 | GET 请求写操作 | 中 | 所有退订功能 |
| P1 | 部分失败无感知 | 中 | 大量收件人时 |
| P1 | 退订无身份验证 | 中 | 所有退订功能 |
