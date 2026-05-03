# 邮件报表与定时任务服务团队收件人分析报告

## 1. 概述

本文档分析 Plausible Analytics 系统中邮件报表功能如何服务团队收件人，重点关注权限边界、数据查询、发送调度和失败处理四个核心方面。

## 2. 权限边界分析

### 2.1 团队级权限控制

**团队锁定检查**
- **位置**: `lib/workers/schedule_email_reports.ex:37` 和 `lib/workers/schedule_email_reports.ex:78`
- **实现**: `where: not t.locked`
- **说明**: 只调度未被锁定的团队的邮件报表，确保被锁定的团队不会收到任何邮件

**站点与团队关联**
- **位置**: `lib/workers/schedule_email_reports.ex:29` 和 `lib/workers/schedule_email_reports.ex:70`
- **实现**: `inner_join: t in assoc(s, :team)`
- **说明**: 站点必须属于一个团队，报表是基于站点的，但权限受团队控制

### 2.2 发送权限检查

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
- **说明**:
  - 企业版(EE): 站点必须是常规站点，或者是合并视图且团队有权限显示
  - 社区版(CE): 始终允许发送

### 2.3 收件人权限管理

**报告收件人列表**
- **位置**: `lib/plausible/site/weekly_report.ex:6` 和 `lib/plausible/site/monthly_report.ex:6`
- **实现**: `field :recipients, {:array, :string}`
- **说明**: 每个站点的周报和月报都有独立的收件人列表，存储为邮箱字符串数组

**站点成员识别**
- **位置**: `lib/workers/send_email_report.ex:59-62`
- **实现**:
  ```elixir
  defp site_member?(site, email) do
    user = Plausible.Auth.find_user_by(email: email)
    user && Plausible.Teams.Memberships.site_member?(site, user)
  end
  ```
- **说明**: 检查收件人是否是站点成员，用于在邮件中显示不同的内容（如退订链接）

**团队成员关系检查**
- **位置**: `lib/plausible/teams/memberships.ex:103-109`
- **实现**:
  ```elixir
  @spec site_member?(Plausible.Site.t(), Auth.User.t() | nil) :: boolean()
  def site_member?(site, user) do
    case site_role(site, user) do
      {:ok, _} -> true
      _ -> false
    end
  end
  ```
- **说明**: 基于站点角色判断用户是否是站点成员

**站点角色计算**
- **位置**: `lib/plausible/teams/memberships.ex:80-101`
- **实现**:
  ```elixir
  @spec site_role(Plausible.Site.t(), Auth.User.t() | nil) ::
          {:ok, {:team_member | :guest_member, Teams.Membership.role()}} | {:error, :not_a_member}

  def site_role(_site, nil), do: {:error, :not_a_member}

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
- **说明**:
  - 区分团队成员(team_member)和访客成员(guest_member)
  - 访客成员只能访问特定站点，而团队成员可以访问团队下所有站点
  - 角色包括: owner, admin, editor, viewer, guest

## 3. 数据查询分析

### 3.1 日期范围计算

**时区感知的日期范围**
- **位置**: `lib/workers/send_email_report.ex:167-199`
- **实现**:
  - 周报: 上周周一到周日
    ```elixir
    defp date_range(site, @weekly) do
      first =
        site.timezone
        |> DateTime.now!()
        |> Date.shift(day: -7)
        |> Date.beginning_of_week()

      last =
        site.timezone
        |> DateTime.now!()
        |> DateTime.to_date()
        |> Date.shift(day: -7)
        |> Date.end_of_week()

      Date.range(first, last)
    end
    ```
  - 月报: 上个月1号到最后一天
    ```elixir
    defp date_range(site, @monthly) do
      first =
        site.timezone
        |> DateTime.now!()
        |> Date.shift(month: -1)
        |> Date.beginning_of_month()

      last =
        site.timezone
        |> DateTime.now!()
        |> DateTime.shift(month: -1)
        |> DateTime.to_date()
        |> Date.end_of_month()

      Date.range(first, last)
    end
    ```
- **说明**: 日期范围计算基于站点时区，确保不同时区的用户收到正确时间范围的报告

### 3.2 统计数据查询

**基础查询参数**
- **位置**: `lib/workers/send_email_report.ex:73-77`
- **实现**:
  ```elixir
  shared_params = %ParsedQueryParams{
    metrics: [:visitors],
    input_date_range: {:date_range, date_range.first, date_range.last},
    pagination: %{limit: 5, offset: 0}
  }
  ```
- **说明**: 所有查询共享相同的基础参数，包括指标、日期范围和分页

**聚合统计查询**
- **位置**: `lib/workers/send_email_report.ex:90-116`
- **实现**:
  ```elixir
  defp stats_aggregates(site, %ParsedQueryParams{} = shared_params) do
    query =
      QueryBuilder.build!(
        site,
        struct!(shared_params,
          metrics: [:pageviews, :visitors, :bounce_rate],
          include: %QueryInclude{compare: :previous_period}
        )
      )

    %QueryResult{
      results: [
        %{
          metrics: [pageviews, visitors, bounce_rate],
          comparison: %{
            change: [pageviews_change, visitors_change, bounce_rate_change]
          }
        }
      ]
    } = Plausible.Stats.query(site, query)

    %{
      pageviews: %{value: pageviews, change: pageviews_change},
      visitors: %{value: visitors, change: visitors_change},
      bounce_rate: %{value: bounce_rate, change: bounce_rate_change}
    }
  end
  ```
- **说明**:
  - 查询页面浏览量(pageviews)、访客数(visitors)、跳出率(bounce_rate)
  - 包含与前一时期的比较数据
  - 使用 `QueryBuilder` 构建查询，通过 `Plausible.Stats.query` 执行

**热门页面查询**
- **位置**: `lib/workers/send_email_report.ex:118-130`
- **实现**:
  ```elixir
  defp pages(site, %ParsedQueryParams{} = shared_params) do
    query = QueryBuilder.build!(site, struct!(shared_params, dimensions: ["event:page"]))

    site
    |> Plausible.Stats.query(query)
    |> Map.fetch!(:results)
    |> Enum.map(fn %{metrics: [visitors], dimensions: [page]} ->
      %{
        page: page,
        visitors: visitors
      }
    end)
  end
  ```
- **说明**: 查询按页面维度分组的访客数，用于显示热门页面

**流量来源查询**
- **位置**: `lib/workers/send_email_report.ex:132-151`
- **实现**:
  ```elixir
  defp sources(site, %ParsedQueryParams{} = shared_params) do
    query =
      QueryBuilder.build!(
        site,
        struct!(shared_params,
          dimensions: ["visit:source"],
          filters: [[:is_not, "visit:source", ["Direct / None"]]]
        )
      )

    site
    |> Plausible.Stats.query(query)
    |> Map.fetch!(:results)
    |> Enum.map(fn %{metrics: [visitors], dimensions: [source]} ->
      %{
        source: source,
        visitors: visitors
      }
    end)
  end
  ```
- **说明**: 查询按来源维度分组的访客数，排除直接流量(Direct / None)

**目标转化查询**
- **位置**: `lib/workers/send_email_report.ex:153-165`
- **实现**:
  ```elixir
  defp goals(site, %ParsedQueryParams{} = shared_params) do
    query = QueryBuilder.build!(site, struct!(shared_params, dimensions: ["event:goal"]))

    site
    |> Plausible.Stats.query(query)
    |> Map.fetch!(:results)
    |> Enum.map(fn %{metrics: [visitors], dimensions: [goal_name]} ->
      %{
        goal: goal_name,
        visitors: visitors
      }
    end)
  end
  ```
- **说明**: 查询按目标维度分组的访客数，用于显示目标转化情况

### 3.3 查询执行流程

**查询构建与执行**
- **位置**: `lib/workers/send_email_report.ex:72-88`
- **实现**:
  ```elixir
  defp stats(site, date_range) do
    shared_params = %ParsedQueryParams{
      metrics: [:visitors],
      input_date_range: {:date_range, date_range.first, date_range.last},
      pagination: %{limit: 5, offset: 0}
    }

    stats = stats_aggregates(site, shared_params)
    pages = pages(site, shared_params)
    sources = sources(site, shared_params)
    goals = goals(site, shared_params)

    stats
    |> Map.put(:pages, pages)
    |> Map.put(:sources, sources)
    |> Map.put(:goals, goals)
  end
  ```
- **说明**:
  1. 构建共享查询参数
  2. 依次执行聚合统计、页面、来源、目标四个查询
  3. 合并所有查询结果返回

## 4. 发送调度分析

### 4.1 调度入口

**调度工作器**
- **位置**: `lib/workers/schedule_email_reports.ex:1-97`
- **实现**:
  ```elixir
  defmodule Plausible.Workers.ScheduleEmailReports do
    use Plausible.Repo
    use Oban.Worker, queue: :schedule_email_reports
    alias Plausible.Workers.SendEmailReport
    require Logger

    @impl Oban.Worker
    @doc """
    Email reports should be sent on Monday at 9am according to the timezone
    of a site.
    """
    def perform(_job) do
      schedule_weekly_emails()
      schedule_monthly_emails()
    end
  end
  ```
- **说明**:
  - 使用 Oban 作为任务调度框架
  - 队列名称: `schedule_email_reports`
  - 执行时同时调度周报和月报

### 4.2 周报调度

**周报调度逻辑**
- **位置**: `lib/workers/schedule_email_reports.ex:17-49`
- **实现**:
  ```elixir
  defp schedule_weekly_emails() do
    weekly_jobs =
      from(
        j in Oban.Job,
        where:
          j.worker == "Plausible.Workers.SendEmailReport" and
            fragment("(? ->> 'interval')", j.args) == "weekly"
      )

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
          where: not t.locked,
          preload: [weekly_report: wr]
      )

    for site <- sites do
      SendEmailReport.new(%{site_id: site.id, interval: "weekly"},
        scheduled_at: monday_9am(site.timezone)
      )
      |> Oban.insert!()
    end

    :ok
  end
  ```
- **说明**:
  1. 查询所有已存在的周报发送任务
  2. 查找满足以下条件的站点:
     - 有周报配置(join weekly_reports)
     - 没有未完成的周报任务(left join + is_nil(job))
     - 团队未被锁定(not t.locked)
  3. 为每个符合条件的站点创建新的发送任务

**周报调度时间计算**
- **位置**: `lib/workers/schedule_email_reports.ex:51-56`
- **实现**:
  ```elixir
  def monday_9am(timezone) do
    DateTime.now!(timezone)
    |> DateTime.shift(week: 1)
    |> Plausible.Times.beginning_of_week()
    |> DateTime.shift(hour: 9)
  end
  ```
- **说明**: 计算下周一早上9点，基于站点时区

### 4.3 月报调度

**月报调度逻辑**
- **位置**: `lib/workers/schedule_email_reports.ex:58-90`
- **实现**:
  ```elixir
  defp schedule_monthly_emails() do
    monthly_jobs =
      from(
        j in Oban.Job,
        where:
          j.worker == "Plausible.Workers.SendEmailReport" and
            fragment("(? ->> 'interval')", j.args) == "monthly"
      )

    sites =
      Repo.all(
        from s in Plausible.Site,
          inner_join: t in assoc(s, :team),
          join: mr in Plausible.Site.MonthlyReport,
          on: mr.site_id == s.id,
          left_join: job in subquery(monthly_jobs),
          on:
            fragment("(? -> 'site_id')::int", job.args) == s.id and
              job.state not in ["completed", "discarded"],
          where: is_nil(job),
          where: not t.locked,
          preload: [monthly_report: mr]
      )

    for site <- sites do
      SendEmailReport.new(%{site_id: site.id, interval: "monthly"},
        scheduled_at: first_of_month_9am(site.timezone)
      )
      |> Oban.insert!()
    end

    :ok
  end
  ```
- **说明**: 与周报调度逻辑类似，但使用月报配置和不同的调度时间

**月报调度时间计算**
- **位置**: `lib/workers/schedule_email_reports.ex:92-97`
- **实现**:
  ```elixir
  def first_of_month_9am(timezone) do
    DateTime.now!(timezone)
    |> DateTime.shift(month: 1)
    |> Plausible.Times.beginning_of_month()
    |> DateTime.shift(hour: 9)
  end
  ```
- **说明**: 计算下个月1号早上9点，基于站点时区

### 4.4 发送任务执行

**发送工作器**
- **位置**: `lib/workers/send_email_report.ex:1-54`
- **实现**:
  ```elixir
  defmodule Plausible.Workers.SendEmailReport do
    use Plausible
    use Plausible.Repo
    use Oban.Worker, queue: :send_email_reports, max_attempts: 1

    alias Plausible.Stats.{QueryResult, QueryBuilder, ParsedQueryParams, QueryInclude}

    import Ecto.Query, only: [from: 2]

    @weekly "weekly"
    @monthly "monthly"

    def perform(%Oban.Job{args: %{"interval" => interval, "site_id" => site_id}})
        when interval in [@weekly, @monthly] do
      report_type = report_type(interval)

      site =
        from(s in Plausible.Site,
          where: s.id == ^site_id,
          inner_join: r in assoc(s, ^report_type),
          inner_join: t in assoc(s, :team),
          preload: [{^report_type, r}, {:team, t}]
        )
        |> Repo.one()

      with %Plausible.Site{} <- site,
           %{} = report <- Map.fetch!(site, report_type),
           true <- ok_to_send?(site) do
        date_range = date_range(site, interval)
        report_name = report_name(interval, date_range.first)
        date_label = Calendar.strftime(date_range.last, "%-d %b %Y")
        stats = stats(site, date_range)

        report
        |> Map.fetch!(:recipients)
        |> Enum.each(fn email ->
          assigns = %{
            site: site,
            report_name: report_name,
            date_label: date_label,
            unsubscribe_link: unsubscribe_link(site, email, interval),
            site_member?: site_member?(site, email),
            interval: interval,
            stats: stats
          }

          email
          |> PlausibleWeb.Email.stats_report(assigns)
          |> Plausible.Mailer.send()
        end)
      else
        _ -> :discard
      end
    end
  end
  ```
- **说明**:
  - 队列名称: `send_email_reports`
  - 最大尝试次数: `max_attempts: 1` (只尝试一次)
  - 执行流程:
    1. 加载站点、报告配置和团队信息
    2. 检查发送权限
    3. 计算日期范围和查询统计数据
    4. 遍历收件人列表，发送邮件

### 4.5 邮件构建与发送

**邮件构建**
- **位置**: `lib/plausible_web/email.ex:137-143`
- **实现**:
  ```elixir
  def stats_report(email, assigns) do
    base_email(%{layout: nil})
    |> to(email)
    |> tag("#{assigns.interval}-report")
    |> subject("#{assigns.report_name} report for #{assigns.site.domain}")
    |> html_body(PlausibleWeb.MJML.StatsReport.render(assigns))
  end
  ```
- **说明**:
  - 使用基础邮件模板
  - 设置收件人、标签、主题
  - 使用 MJML 渲染 HTML 内容

**退订链接生成**
- **位置**: `lib/workers/send_email_report.ex:64-67`
- **实现**:
  ```elixir
  defp unsubscribe_link(site, email, interval) do
    PlausibleWeb.Endpoint.url() <>
      "/sites/#{URI.encode_www_form(site.domain)}/#{interval}-report/unsubscribe?email=#{email}"
  end
  ```
- **说明**: 为每个收件人生成个性化的退订链接

## 5. 失败处理分析

### 5.1 发送任务失败处理

**单次尝试策略**
- **位置**: `lib/workers/send_email_report.ex:4`
- **实现**: `use Oban.Worker, queue: :send_email_reports, max_attempts: 1`
- **说明**: 发送任务只尝试一次，失败后不会重试

**条件失败处理**
- **位置**: `lib/workers/send_email_report.ex:26-53`
- **实现**:
  ```elixir
  with %Plausible.Site{} <- site,
       %{} = report <- Map.fetch!(site, report_type),
       true <- ok_to_send?(site) do
    # 发送逻辑...
  else
    _ -> :discard
  end
  ```
- **说明**: 使用 `with` 语句处理条件失败，任何条件不满足都返回 `:discard`，任务被丢弃

### 5.2 调度任务失败处理

**Oban 错误报告器**
- **位置**: `lib/oban_error_reporter.ex:1-78`
- **实现**:
  ```elixir
  defmodule ObanErrorReporter do
    use Plausible
    require Logger

    def handle_event(name, measurements, metadata, _) do
      # handling telemetry event in a try/catch block
      # to avoid handler detachment in the case of an error
      # see https://hexdocs.pm/telemetry/telemetry.html#attach/4
      try do
        handle_event(name, measurements, metadata)
      catch
        kind, reason ->
          message = Exception.format(kind, reason, __STACKTRACE__)
          Logger.error(message)
      end
    end

    defp handle_event([:oban, :job, :exception], measure, %{job: job} = meta) do
      extra =
        job
        |> Map.take([:id, :args, :meta, :queue, :worker])
        |> Map.merge(measure)

      on_job_exception(job)
      capture_error(meta, extra)
    end

    # ... 其他事件处理
  end
  ```
- **说明**:
  - 监听 Oban 的 telemetry 事件
  - 捕获任务异常并记录日志
  - 执行特定任务的异常处理逻辑

**异常捕获与日志**
- **位置**: `lib/oban_error_reporter.ex:67-77`
- **实现**:
  ```elixir
  # Logs the error and sends it to Sentry
  defp capture_error(meta, extra) do
    Logger.error(
      # this message is ignored by Sentry
      "Background job (#{inspect(extra)}) failed:\n\n  " <>
        Exception.format(:error, meta.reason, meta.stacktrace),
      # Sentry report is built entirely from crash_reason
      crash_reason: {meta.reason, meta.stacktrace},
      sentry: %{extra: extra}
    )
  end
  ```
- **说明**:
  - 记录错误日志
  - 发送错误到 Sentry 进行监控
  - 包含任务详细信息作为额外上下文

### 5.3 特定任务异常处理

**导入任务异常处理**
- **位置**: `lib/oban_error_reporter.ex:38-65`
- **实现**:
  ```elixir
  defp on_job_exception(%Oban.Job{
         queue: "analytics_imports",
         args: %{"import_id" => import_id},
         state: "executing",
         attempt: attempt,
         max_attempts: max_attempts
       })
       when attempt >= max_attempts do
    site_import = Plausible.Repo.get(Plausible.Imported.SiteImport, import_id)

    if site_import do
      Plausible.Workers.ImportAnalytics.import_fail(site_import, [])
    end
  end

  defp on_job_exception(%Oban.Job{
         queue: "analytics_imports",
         args: %{"import_id" => import_id},
         state: "executing"
       }) do
    site_import = Plausible.Repo.get(Plausible.Imported.SiteImport, import_id)

    if site_import do
      Plausible.Workers.ImportAnalytics.import_fail_transient(site_import)
    end
  end

  defp on_job_exception(_job), do: :ignore
  ```
- **说明**:
  - 只有导入任务有特殊的异常处理逻辑
  - 邮件报表任务没有特殊的异常处理，异常会被 `capture_error` 捕获并记录
  - 最后一次尝试失败: 标记导入失败
  - 还有重试机会: 标记为瞬时失败

### 5.4 邮件发送失败处理

**邮件发送异常**
- **位置**: `lib/workers/send_email_report.ex:47-50`
- **实现**:
  ```elixir
  email
  |> PlausibleWeb.Email.stats_report(assigns)
  |> Plausible.Mailer.send()
  ```
- **说明**:
  - 邮件发送在 `Enum.each` 中执行
  - 如果某个邮件发送失败，会抛出异常
  - 整个任务失败，但已发送的邮件不会回滚
  - 由于 `max_attempts: 1`，失败的任务不会重试

## 6. 关键文件索引

| 功能模块 | 文件路径 | 主要职责 |
|---------|---------|---------|
| 调度工作器 | `lib/workers/schedule_email_reports.ex` | 调度周报和月报发送任务 |
| 发送工作器 | `lib/workers/send_email_report.ex` | 执行邮件发送，包括数据查询和邮件发送 |
| 错误报告器 | `lib/oban_error_reporter.ex` | 处理 Oban 任务异常，记录日志和发送到 Sentry |
| 周报配置 | `lib/plausible/site/weekly_report.ex` | 定义周报配置的数据结构 |
| 月报配置 | `lib/plausible/site/monthly_report.ex` | 定义月报配置的数据结构 |
| 邮件构建 | `lib/plausible_web/email.ex` | 构建各类邮件，包括统计报告邮件 |
| 成员关系 | `lib/plausible/teams/memberships.ex` | 管理团队成员关系和权限检查 |
| 团队策略 | `lib/plausible/teams/policy.ex` | 定义团队级别的策略配置 |
| 用户认证 | `lib/plausible/auth/auth.ex` | 用户查找和认证相关功能 |

## 7. 总结

### 7.1 权限边界
- **团队级控制**: 通过 `team.locked` 字段控制整个团队是否能接收邮件
- **站点级控制**: 通过 `ok_to_send?` 函数检查站点是否可以发送报表
- **收件人列表**: 每个站点有独立的收件人列表，存储为邮箱字符串数组
- **成员识别**: 通过 `site_member?` 函数区分站点成员和外部收件人

### 7.2 数据查询
- **时区感知**: 所有日期计算都基于站点时区
- **多维度查询**: 包括聚合统计、页面、来源、目标四个维度
- **比较数据**: 包含与前一时期的比较数据
- **查询构建**: 使用 `QueryBuilder` 构建查询，通过 `Plausible.Stats.query` 执行

### 7.3 发送调度
- **Oban 框架**: 使用 Oban 作为任务调度和执行框架
- **两级调度**: 调度工作器(`ScheduleEmailReports`)负责创建发送任务，发送工作器(`SendEmailReport`)负责实际发送
- **时区调度**: 调度时间基于站点时区计算，确保在正确的时间发送
- **防重复调度**: 通过检查已存在的任务状态避免重复调度

### 7.4 失败处理
- **单次尝试**: 发送任务只尝试一次，失败后丢弃
- **条件失败**: 使用 `with` 语句处理条件失败，不满足条件的任务被丢弃
- **异常捕获**: 通过 `ObanErrorReporter` 捕获任务异常，记录日志并发送到 Sentry
- **无重试机制**: 邮件报表任务没有特殊的异常处理和重试机制，依赖于日志和监控

### 7.5 潜在改进点
1. **发送失败重试**: 当前 `max_attempts: 1`，可以考虑增加重试次数或实现指数退避策略
2. **部分失败处理**: 当前 `Enum.each` 中一个邮件失败会导致整个任务失败，可以考虑使用 `Enum.reduce` 或 `Task.async_stream` 实现容错
3. **发送状态追踪**: 可以考虑记录每个收件人的发送状态，便于问题排查
4. **限流机制**: 对于大量收件人的情况，可以考虑添加限流机制，避免邮件服务商限制
