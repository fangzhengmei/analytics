# ClickHouse 后台写入失败恢复与重试机制分析报告

## 概述

本文档分析了 Plausible Analytics 项目中后台任务向 ClickHouse 写入事件数据时的失败恢复机制和重试策略设计。重点对比了两条核心写入链路：
1. **WriteBuffer 实时写入链路**：事件数据的近实时写入
2. **Oban 后台任务链路**：异步任务（导入、导出、清理等）的调度执行

---

## 一、数据写入架构

### 1.1 两条写入链路概览

```
┌─────────────────────────────────────────────────────────────────────┐
│                         事件数据来源                                  │
├─────────────────────────────────────────────────────────────────────┤
│  实时事件流                    异步任务触发                            │
│  (页面访问、事件)              (导入、导出、清理)                      │
└──────────────────┬──────────────────────────┬───────────────────────┘
                   │                          │
                   ▼                          ▼
┌──────────────────────────────┐   ┌──────────────────────────────────┐
│  链路 A: WriteBuffer 实时写入 │   │     链路 B: Oban 后台任务        │
├──────────────────────────────┤   ├──────────────────────────────────┤
│  • 近实时（秒级）延迟         │   │  • 异步执行，延迟可控             │
│  • 内存缓冲，批量写入         │   │  • PostgreSQL 持久化存储          │
│  • 无重试，无持久化           │   │  • 完善的重试和回退机制           │
│  • 进程崩溃即数据丢失         │   │  • 应用重启不影响任务状态          │
└──────────────────────────────┘   └──────────────────────────────────┘
                   │                          │
                   ▼                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         ClickHouse                                    │
│  events_v2, sessions_v2, ingest_counters, imported_* 等表           │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 WriteBuffer 链路的持久化后端选择

系统提供三种持久化后端，通过 `Plausible.Ingestion.Persistor` 模块统一调度：

| 后端模块 | 用途 | 适用场景 |
|---------|------|---------|
| `Embedded` | 本地 WriteBuffer 写入 | 默认后端，单实例部署 |
| `Remote` | 远程 HTTP 调用写入 | 多实例部署，中心化写入 |
| `EmbeddedWithRelay` | 本地写入 + 异步远程中继 | 混合模式 |

**后端选择逻辑** (`lib/plausible/ingestion/persistor.ex:15-36`)：
- 根据 `backend_percent_enabled` 配置按用户哈希分流
- `Embedded` 作为 fallback 后端

---

## 二、WriteBuffer 实时写入链路

### 2.1 架构设计

`Plausible.Ingestion.WriteBuffer` 是一个 GenServer，负责缓冲事件数据并批量写入 ClickHouse。

**核心配置参数** (`config/runtime.exs:644-650`)：
```elixir
config :plausible, Plausible.IngestRepo,
  flush_interval_ms: ch_flush_interval_ms,  # 默认 5000ms (5秒)
  max_buffer_size: ch_max_buffer_size,       # 默认 100_000 字节 (100KB)
  pool_size: ingest_pool_size
```

### 2.2 刷新触发条件

缓冲区刷新有以下四种触发方式：

1. **缓冲区满** (`lib/plausible/ingestion/write_buffer.ex:51-56`)
   - 当 `buffer_size >= max_buffer_size` 时立即刷新
   - 取消当前定时器，刷新后重建定时器

2. **定时刷新** (`lib/plausible/ingestion/write_buffer.ex:63-67`)
   - 每 `flush_interval_ms` 毫秒自动触发
   - 无论缓冲区是否有数据都会执行

3. **手动刷新** (`lib/plausible/ingestion/write_buffer.ex:70-76`)
   - 调用 `WriteBuffer.flush(server)` 同步刷新
   - 用于优雅关闭等场景

4. **进程终止** (`lib/plausible/ingestion/write_buffer.ex:78-82`)
   - 设置 `Process.flag(:trap_exit, true)`
   - `terminate/2` 回调中执行最终刷新

### 2.3 数据写入实现（可直接证实）

**核心写入代码** (`lib/plausible/ingestion/write_buffer.ex:84-102`)：

```elixir
defp do_flush(state) do
  %{
    buffer: buffer,
    buffer_size: buffer_size,
    insert_opts: insert_opts,
    insert_sql: insert_sql,
    header: header,
    name: name
  } = state

  case buffer do
    [] ->
      nil

    _not_empty ->
      Logger.notice("Flushing #{buffer_size} byte(s) RowBinary from #{name}")
      IngestRepo.query!(insert_sql, [header | buffer], insert_opts)
  end
end
```

### 2.4 关键问题：无重试机制（可直接证实）

**⚠️ 核心问题：WriteBuffer 没有内置重试机制**

在 `do_flush/1` 函数中使用了 `IngestRepo.query!`（带感叹号的版本）：

| 代码证据 | 位置 | 结论 |
|---------|------|------|
| `IngestRepo.query!(...)` | `write_buffer.ex:100` | 使用会抛异常的版本 |
| 无 `try/rescue` 包裹 | `write_buffer.ex:84-102` | 异常直接向上传播 |
| 无重试循环逻辑 | 同上 | 一次失败即终止 |
| 无死信队列逻辑 | 同上 | 失败数据无持久化 |

### 2.5 故障恢复流程（可直接证实）

当 WriteBuffer 写入失败时：

```
1. IngestRepo.query! 抛出异常（代码证实：使用 ! 版本）
         ↓
2. GenServer 进程崩溃（代码证实：无异常捕获）
         ↓
3. Supervisor 检测到进程终止（代码证实：application.ex:212 使用 :one_for_one）
         ↓
4. 使用 :one_for_one 策略重启进程（代码证实）
         ↓
5. 新进程启动，缓冲区为空（代码证实：init/1 中 buffer 初始化为空）
         ↓
6. ❌ 崩溃前缓冲的数据永久丢失
```

**监督树配置证据** (`lib/plausible/application.ex:197-198, 212`)：
```elixir
# WriteBuffer 注册
Supervisor.child_spec(Plausible.Event.WriteBuffer, id: Plausible.Event.WriteBuffer),
Supervisor.child_spec(Plausible.Session.WriteBuffer, id: Plausible.Session.WriteBuffer),

# 监督策略
opts = [strategy: :one_for_one, name: Plausible.Supervisor]
```

**初始化证据** (`lib/plausible/ingestion/write_buffer.ex:20-41`)：
```elixir
def init(opts) do
  buffer = opts[:buffer] || []   # 默认空数组
  # ... 其他配置
  {:ok, %{buffer: buffer, ...}}  # 新进程缓冲区为空
end
```

### 2.6 相关组件的重试策略

#### 2.6.1 Remote 后端的 HTTP 重试（可直接证实）

`Plausible.Ingestion.Persistor.Remote` 有 HTTP 层面的重试：

**配置证据** (`lib/plausible/ingestion/persistor/remote.ex:9, 74-76`)：
```elixir
@max_transient_retries 3

# Req 配置
Req.new(
  # ...
  retry: &handle_transient_error/2,
  max_retries: @max_transient_retries
)
```

**重试条件证据** (`lib/plausible/ingestion/persistor/remote.ex:87-95`)：
```elixir
defp handle_transient_error(_request, %Req.HTTPError{protocol: :http2, reason: :disconnected}), do: true
defp handle_transient_error(_request, %Req.HTTPError{protocol: :http2, reason: :unprocessed}), do: true
defp handle_transient_error(_reqeust, _response), do: false
```

**限制**：
- 仅适用于 Remote 后端（HTTP 调用）
- 仅重试 HTTP/2 特定连接错误
- **不适用**于 Embedded 后端的 ClickHouse 直接写入

#### 2.6.2 Counters 模块的错误处理（可直接证实）

`Plausible.Ingestion.Counters` 有基础的异常捕获：

**代码证据** (`lib/plausible/ingestion/counters.ex:84-95`)：
```elixir
try do
  {_, _} = AsyncInsertRepo.insert_all(Record, records)
catch
  _, thrown ->
    Sentry.capture_message(
      "Caught an error when trying to flush ingest counters.",
      extra: %{
        number_of_records: Enum.count(records),
        error: inspect(thrown)
      }
    )
end
```

**特点**：
- 使用 `try/catch` 捕获异常（代码证实）
- 上报到 Sentry 用于告警（代码证实）
- **但没有重试逻辑**（代码证实：catch 后无重试）
- 数据仍然丢失

---

## 三、Oban 后台任务调度层

### 3.1 Oban 基础配置（可直接证实）

**核心配置证据** (`config/runtime.exs:864-876`)：
```elixir
config :plausible, Oban,
  repo: Plausible.Repo,           # 任务数据存储在 PostgreSQL（代码证实）
  plugins: [
    {Oban.Plugins.Pruner, max_age: thirty_days_in_seconds},
    {Oban.Plugins.Cron, crontab: if(cron_enabled, do: crontab, else: [])},
    {Oban.Plugins.Lifeline, rescue_after: :timer.minutes(120)},  # 配置存在
    {Oban.Plugins.Reindexer, schedule: "0 1 * * *"}
  ],
  queues: queues,
  peer: if(cron_enabled, do: Oban.Peers.Postgres, else: false)
```

**可直接证实的配置项**：
| 配置项 | 证据位置 | 结论 |
|-------|---------|------|
| `repo: Plausible.Repo` | `runtime.exs:865` | 任务数据持久化到 PostgreSQL |
| `Lifeline, rescue_after: 120分钟` | `runtime.exs:871` | 插件配置存在 |

### 3.2 各 Worker 的重试配置（可直接证实）

#### 3.2.1 max_attempts 配置

| Worker | max_attempts | 证据位置 | 结论 |
|--------|-------------|---------|------|
| `ImportAnalytics` | **3** | `import_analytics.ex:11` | 显式设置 3 次 |
| `ExportAnalytics` | **3** | `export_analytics.ex:9` | 显式设置 3 次 |
| `NotifyExportedAnalytics` | **5** | `notify_exported_analytics.ex:6` | 显式设置 5 次 |
| `PurgeCDNCache` | **5** | `purge_cdn_cache.ex:13` | 显式设置 5 次 |
| `SendEmailReport` | **1** | `send_email_report.ex:4` | 显式设置 1 次（不重试） |
| `ClickhouseCleanSites` | **未设置** | `clickhouse_clean_sites.ex:12` | 无显式设置 |
| `LocationsSync` | **未设置** | `locations_sync.ex:5` | 无显式设置 |
| `ScheduleEmailReports` | **未设置** | `schedule_email_reports.ex:3` | 无显式设置 |
| `SendCheckStatsEmails` | **未设置** | `send_check_stats_emails.ex:3` | 无显式设置 |
| `SendSiteSetupEmails` | **未设置** | `send_site_setup_emails.ex:3` | 无显式设置 |
| `SendTrialNotifications` | **未设置** | `send_trial_notifications.ex:6` | 无显式设置 |
| `LockSites` | **未设置** | `lock_sites.ex:3` | 无显式设置 |
| `CheckUsage` | **未设置** | `check_usage.ex:3` | 无显式设置 |
| `CleanInvitations` | **未设置** | `clean_invitations.ex:3` | 无显式设置 |
| `CleanUserSessions` | **未设置** | `clean_user_sessions.ex:3` | 无显式设置 |
| `RotateSalts` | **未设置** | `rotate_salts.ex:3` | 无显式设置 |
| `ExpireDomainChangeTransitions` | **未设置** | `expire_domain_change_transitions.ex:3` | 无显式设置 |
| `TrafficChangeNotifier` | **未设置** | `traffic_change_notifier.ex:3` | 无显式设置 |
| `SetLegacyTimeOnPageCutoff` | **未设置** | `set_legacy_time_on_page_cutoff.ex:3` | 无显式设置 |
| `LocalImportAnalyticsCleaner` | **未设置** | `local_import_analytics_cleaner.ex:6` | 无显式设置 |
| `AcceptTrafficUntilNotification` | **未设置** | `accept_traffic_until_notification.ex:3` | 无显式设置 |
| `NotifyAnnualRenewal` | **未设置** | `notify_annual_renewal.ex:3` | 无显式设置 |

**代码示例证据** (`lib/workers/import_analytics.ex:9-12`)：
```elixir
use Oban.Worker,
  queue: :analytics_imports,
  max_attempts: 3,           # 显式设置 3 次
  unique: [fields: [:args], keys: [:import_id], period: 60]
```

**无显式设置示例** (`lib/workers/clickhouse_clean_sites.ex:12`)：
```elixir
use Oban.Worker, queue: :clickhouse_clean_sites   # 无 max_attempts
```

#### 3.2.2 backoff 策略实现

| Worker | backoff 实现 | 证据位置 | 重试间隔 |
|--------|-------------|---------|---------|
| `ImportAnalytics` | **有实现** | `import_analytics.ex:51-54` | 固定 300 秒（5分钟） |
| `PurgeCDNCache` | **有实现** | `purge_cdn_cache.ex:33-36` | 指数退避（3→6→12→24→48分钟） |
| 其他所有 Worker | **无实现** | - | ⚠️ 待确认 |

**ImportAnalytics backoff 证据** (`lib/workers/import_analytics.ex:51-54`)：
```elixir
@impl Oban.Worker
def backoff(_job) do
  # 5 minutes
  300
end
```

**PurgeCDNCache backoff 证据** (`lib/workers/purge_cdn_cache.ex:33-36`)：
```elixir
@impl Oban.Worker
def backoff(%Oban.Job{attempt: attempt}) do
  # Exponential backoff starting at 3 minutes
  trunc(:math.pow(2, attempt - 1) * 180)
end
```

**PurgeCDNCache backoff 计算（可直接证实）**：

| attempt | 计算 | 实际间隔 |
|---------|------|---------|
| 1 | `2^0 * 180 = 1 * 180` | 180 秒 = 3 分钟 |
| 2 | `2^1 * 180 = 2 * 180` | 360 秒 = 6 分钟 |
| 3 | `2^2 * 180 = 4 * 180` | 720 秒 = 12 分钟 |
| 4 | `2^3 * 180 = 8 * 180` | 1440 秒 = 24 分钟 |
| 5 | `2^4 * 180 = 16 * 180` | 2880 秒 = 48 分钟 |

### 3.3 perform/1 返回值（可直接证实）

各 Worker 的 `perform/1` 返回值：

| Worker | 返回值类型 | 证据位置 |
|--------|-----------|---------|
| `ImportAnalytics` | `:ok` 或 `{:discard, error}` | `import_analytics.ex:33, 46` |
| `PurgeCDNCache` | `{:ok, :success}`, `{:error, reason}`, `{:discard, ...}` | `purge_cdn_cache.ex:57, 64, 68, 40` |
| `ClickhouseCleanSites` | `:ok` | `clickhouse_clean_sites.ex:53` |
| `ExportAnalytics` | `:ok` 或抛出异常 | `export_analytics.ex:61, 65` |

**ImportAnalytics 返回值证据** (`lib/workers/import_analytics.ex:29-47`)：
```elixir
case import_api.run_import(site_import, args) do
  {:ok, site_import} ->
    import_complete(site_import)
    :ok   # 返回 :ok

  {:error, error, error_opts} ->
    # ... 错误处理
    {:discard, error}   # 返回 {:discard, reason}
end
```

**PurgeCDNCache 返回值证据** (`lib/workers/purge_cdn_cache.ex:38-69`)：
```elixir
defp purge_cache(id, pullzone_id, api_key) when is_nil(pullzone_id) or is_nil(api_key) do
  {:discard, "Configuration missing"}   # 返回 {:discard, ...}
end

defp purge_cache(id, pullzone_id, api_key) do
  case Req.post(...) do
    {:ok, %{status: 204}} ->
      {:ok, :success}   # 返回 {:ok, ...}

    {:ok, %{status: status}} ->
      {:error, "Unexpected status: #{status}"}   # 返回 {:error, ...}

    {:error, reason} ->
      {:error, reason}   # 返回 {:error, ...}
  end
end
```

### 3.4 涉及 ClickHouse 写入的后台任务（可直接证实）

| Worker | 功能 | ClickHouse 操作 | max_attempts | backoff |
|--------|------|----------------|-------------|---------|
| `ClickhouseCleanSites` | 清理已删除站点的数据 | `ALTER TABLE ... DELETE` | 未设置 | 无实现 |
| `ImportAnalytics` | 导入历史数据 | 批量 `INSERT` | 3 (显式) | 固定 5 分钟 |
| `ExportAnalytics` | 导出数据 | `SELECT` 查询 | 3 (显式) | 无实现 |
| `LocationsSync` | 同步地理位置数据 | 写入 `locations` 表 | 未设置 | 无实现 |

**ClickhouseCleanSites 示例证据** (`lib/workers/clickhouse_clean_sites.ex:36-54`)：
```elixir
def perform(_job) do
  deleted_sites = get_deleted_sites_with_clickhouse_data()

  if not Enum.empty?(deleted_sites) do
    for table <- @tables_to_clear do
      IngestRepo.query!(
        "ALTER TABLE {$0:Identifier} DELETE WHERE site_id IN {$1:Array(UInt64)}",
        [table, deleted_sites],
        settings: @settings
      )
    end
  end

  :ok
end
```

**注意**：此 Worker 也使用了 `query!`（代码证实），但因为运行在 Oban 中，任务参数存储在 PostgreSQL 中。

### 3.5 待确认的内容（依据不足）

以下内容需要查阅 Oban 官方文档或源码确认，本代码库中无直接证据：

#### 3.5.1 默认 max_attempts

| 问题 | 待确认内容 | 依据不足的原因 |
|-----|-----------|---------------|
| 未设置 `max_attempts` 的 Worker 会重试多少次？ | Oban 默认 `max_attempts` 值 | 代码库中只发现部分 Worker 显式设置，未发现默认值定义 |
| `ClickhouseCleanSites` 实际重试次数是多少？ | 默认值是多少？ | 该 Worker 未显式设置 `max_attempts` |

#### 3.5.2 默认 backoff 策略

| 问题 | 待确认内容 | 依据不足的原因 |
|-----|-----------|---------------|
| 未实现 `backoff/1` 的 Worker 使用什么重试间隔？ | Oban 默认 backoff 策略 | 代码库中只有 `ImportAnalytics` 和 `PurgeCDNCache` 实现了 `backoff/1` |
| `ClickhouseCleanSites` 失败后多久重试？ | 默认间隔是多少？ | 该 Worker 未实现 `backoff/1` |

#### 3.5.3 返回值与任务状态的映射

| 问题 | 待确认内容 | 依据不足的原因 |
|-----|-----------|---------------|
| 返回 `:ok` 后任务状态是什么？ | `:ok` → `completed`? | 代码库中有返回 `:ok` 的示例，但无状态映射证据 |
| 返回 `{:discard, reason}` 后任务状态是什么？ | `{:discard, ...}` → `discarded`? | 代码库中有返回 `{:discard, ...}` 的示例，但无状态映射证据 |
| 返回 `{:error, reason}` 后任务状态是什么？ | `{:error, ...}` → 重试? | 代码库中有返回 `{:error, ...}` 的示例，但无状态映射证据 |
| 抛出异常后任务状态是什么？ | 异常 → 重试? | 代码库中使用了 `query!` 会抛异常，但无异常处理证据 |

#### 3.5.4 Lifeline 插件行为

| 问题 | 待确认内容 | 依据不足的原因 |
|-----|-----------|---------------|
| `rescue_after: 120分钟` 具体做什么？ | 超过 120 分钟的任务如何处理？ | 配置项存在，但具体行为在 Oban 框架内部 |
| "救援孤立任务"的具体含义是什么？ | 状态如何变化？ | 代码库中无相关注释或测试 |

#### 3.5.5 任务状态流转

| 问题 | 待确认内容 | 依据不足的原因 |
|-----|-----------|---------------|
| Oban 任务有哪些状态？ | 状态枚举值 | 代码库中无状态定义 |
| 状态之间如何流转？ | 状态机定义 | 代码库中无状态流转证据 |
| 什么条件下任务会被丢弃？ | `discarded` 状态触发条件 | 代码库中只有 `{:discard, ...}` 返回值，无完整触发条件 |

---

## 四、两条链路对比分析

### 4.1 核心差异对比表

| 维度 | WriteBuffer 实时链路 | Oban 后台任务链路 | 证据来源 |
|-----|---------------------|------------------|---------|
| **数据存储** | 内存缓冲（进程内） | PostgreSQL 持久化 | WriteBuffer: `write_buffer.ex:22`;<br>Oban: `runtime.exs:865` |
| **重试机制** | ❌ 无重试逻辑 | ✅ 有重试机制（max_attempts + backoff） | WriteBuffer: `write_buffer.ex:84-102`;<br>Oban: `import_analytics.ex:11, 51-54` |
| **异常处理** | ❌ 无 try/rescue | ⚠️ 部分 Worker 有，部分依赖框架 | WriteBuffer: `write_buffer.ex:84-102`;<br>Oban: `counters.ex:84-95` |
| **故障恢复** | ❌ 进程崩溃即丢失 | ✅ 应用重启不影响 | WriteBuffer: `write_buffer.ex:22`;<br>Oban: `runtime.exs:865` |
| **数据延迟** | 秒级（5秒刷新间隔） | 可控（异步调度） | WriteBuffer: `runtime.exs:649` |
| **任务状态追踪** | ❌ 无 | ⚠️ 依赖 Oban 框架 | 代码库中无直接状态证据 |

### 4.2 数据持久性对比（可直接证实）

**WriteBuffer 链路证据** (`lib/plausible/ingestion/write_buffer.ex:20-41`)：
```elixir
def init(opts) do
  buffer = opts[:buffer] || []   # 内存数组，无持久化
  # ...
  {:ok, %{buffer: buffer, ...}}
end
```

**Oban 链路证据** (`config/runtime.exs:864-865`)：
```elixir
config :plausible, Oban,
  repo: Plausible.Repo,   # PostgreSQL 持久化
```

### 4.3 失败处理流程对比

#### WriteBuffer 失败流程（可直接证实）

```
事件写入 WriteBuffer
        │
        ▼
┌───────────────┐
│ 内存缓冲累积   │ ←── 代码证实：buffer 是内存数组
│ (5秒/100KB)  │
└───────┬───────┘
        │
        ▼ 定时/定量刷新
┌───────────────┐
│ IngestRepo.   │
│ query!()      │ ←── 代码证实：使用 ! 版本，会抛异常
└───────┬───────┘
        │
   ┌────┴────┐
   │ 成功?    │
   └────┬────┘
        │
   是 ──┴── 否
   │          │
   ▼          ▼
 完成    query! 抛出异常
                │
                ▼
         GenServer 进程崩溃 ←── 代码证实：无 try/rescue
                │
                ▼
         Supervisor 重启进程 ←── 代码证实：:one_for_one 策略
                │
                ▼
         新进程缓冲区为空 ←── 代码证实：init 中 buffer 默认为 []
                │
                ▼
         ❌ 原数据永久丢失
```

#### Oban 失败流程（部分待确认）

```
任务入队 Oban.insert!()
        │
        ▼
┌───────────────┐
│ PostgreSQL    │ ←── 代码证实：repo: Plausible.Repo
│ oban_jobs 表  │ ←── 持久化存储
│ state: avail  │
└───────┬───────┘
        │
        ▼ Worker 拉取执行
┌───────────────┐
│ perform()     │ ←── 代码证实：各 Worker 的 perform 函数
└───────┬───────┘
        │
   ┌────┴────┐
   │ 成功?    │
   └────┬────┘
        │
   是 ──┴── 否
   │          │
   ▼          ▼
completed     │
              ▼
       捕获异常/返回错误
              │
              ▼
    ┌─────────────────┐
    │ attempt <       │ ←── ⚠️ 待确认：异常/错误是否增加 attempt?
    │ max_attempts?   │
    └────────┬────────┘
             │
        是 ──┴── 否
        │          │
        ▼          ▼
   计算 backoff   discarded
        │         (超过重试次数)
        ▼
   更新 scheduled_at
        │
        ▼
   PostgreSQL 持久化
        │
        ▼
   ⏰ 等待下次重试
   
   ✅ 任务参数始终可恢复 ←── 代码证实：存储在 PostgreSQL
```

### 4.4 恢复边界对比

| 链路 | 可恢复场景（代码证实） | 不可恢复场景（代码证实） | 恢复边界 |
|-----|----------------------|------------------------|---------|
| **WriteBuffer** | • 无 | • 进程崩溃<br>• ClickHouse 不可用<br>• 网络分区 | ❌ 无恢复机制 |
| **Oban** | • 任务参数持久化（可手动恢复）<br>• PostgreSQL 不丢则任务不丢 | ⚠️ 待确认：主动 discard?<br>⚠️ 待确认：超过 max_attempts? | PostgreSQL 持久化 |

---

## 五、失败场景分析

### 5.1 WriteBuffer 链路失败场景（可直接证实）

| 场景 | 触发条件 | 当前行为（代码证实） | 数据丢失风险 |
|-----|---------|---------------------|-------------|
| ClickHouse 连接中断 | 网络分区、服务重启 | `query!` 抛异常 → 进程崩溃 | ⚠️ 高 |
| ClickHouse 负载过高 | 写入超时、队列满 | 同上 | ⚠️ 高 |
| 进程优雅关闭 (SIGTERM) | 正常停止 | `terminate/2` 中刷新 | ✅ 低 |
| 进程强制终止 (SIGKILL/OOM) | 异常终止 | 无法执行 `terminate` | ⚠️ 高 |

**证据来源**：
- `query!` 抛异常：`write_buffer.ex:100`
- 无异常捕获：`write_buffer.ex:84-102`
- `terminate/2` 刷新：`write_buffer.ex:78-82`

### 5.2 Oban 链路失败场景（部分待确认）

| 场景 | 触发条件 | 当前行为 | 数据丢失风险 | 证据状态 |
|-----|---------|---------|-------------|---------|
| ClickHouse 连接中断 | 网络分区、服务重启 | ⚠️ 待确认：是否重试？ | ⚠️ 待确认 | 行为待确认 |
| Worker 进程崩溃 | OOM、Bug | ⚠️ 待确认：Lifeline 是否救援？ | ⚠️ 待确认 | 行为待确认 |
| 应用重启 | 部署、维护 | 任务保留在 PostgreSQL | ✅ 低 | 代码证实 |
| 超过最大重试 | `max_attempts` 耗尽 | ⚠️ 待确认：状态变为 discarded? | ⚠️ 需手动恢复 | 行为待确认 |
| PostgreSQL 故障 | 数据库宕机 | 任务无法入队/执行 | ⚠️ 极低（依赖 HA） | 逻辑推断 |

**代码证实的内容**：
- 任务入队 PostgreSQL：`runtime.exs:865`
- 应用重启不影响：PostgreSQL 持久化特性

**待确认的内容**：
- 异常后的重试行为
- Lifeline 插件的具体行为
- 超过 max_attempts 后的状态

### 5.3 数据丢失窗口

**WriteBuffer 链路（代码证实）**：
1. **WriteBuffer 内部缓冲**：约 5 秒或 100KB 数据 (`runtime.exs:649-650`)
2. **无重试机制**：首次失败即丢失 (`write_buffer.ex:84-102`)
3. **进程崩溃后**：新进程无法恢复旧数据 (`write_buffer.ex:22`)

**Oban 链路（部分待确认）**：
1. **执行中状态**：⚠️ 待确认瞬时故障是否可重试恢复
2. **超过重试次数**：⚠️ 待确认状态变化
3. **PostgreSQL 故障**：极端场景下的风险

---

## 六、最终结论

### 6.1 WriteBuffer 实时写入链路

#### 数据可恢复边界（代码证实）

**✅ 唯一可恢复的场景**：
- **进程优雅关闭 (SIGTERM)**：`terminate/2` 回调会执行最终刷新 (`write_buffer.ex:78-82`)

#### 数据丢失边界（代码证实）

**❌ 不可恢复的场景（所有其他情况）**：

| 场景 | 丢失原因 | 证据位置 |
|-----|---------|---------|
| **ClickHouse 不可用** | `query!` 抛异常，无重试 | `write_buffer.ex:84-102` |
| **进程崩溃 (异常)** | 内存缓冲区丢失，新进程无旧数据 | `write_buffer.ex:22` |
| **进程被强制终止 (SIGKILL/OOM)** | 无法执行 `terminate/2` | 逻辑推断 |
| **网络分区** | 同上 | 同上 |

**核心问题根因**：
1. **无持久化**：缓冲区数据仅存在于进程内存 (`write_buffer.ex:22`)
2. **无重试**：`do_flush/1` 使用 `query!` 且无重试循环 (`write_buffer.ex:84-102`)
3. **无死信队列**：失败数据无持久化机制（代码证实：无相关逻辑）

**数据丢失量估算**：
- 默认配置：每 5 秒刷新 或 100KB 缓冲区
- 理论最大丢失：约 5 秒内的所有事件 或 100KB 数据
- 实际丢失：取决于崩溃发生的时间点

### 6.2 Oban 后台任务链路

#### 数据可恢复边界（代码证实 + 逻辑推断）

**✅ 可恢复的场景（代码证实）**：

| 场景 | 可恢复原因 | 证据位置 |
|-----|-----------|---------|
| **应用重启** | 任务参数存储在 PostgreSQL | `runtime.exs:865` |
| **部署/维护** | 同上 | 同上 |
| **Worker 进程崩溃** | ⚠️ 待确认：Lifeline 插件可能救援 | `runtime.exs:871` (配置存在) |

**✅ 可手动恢复的场景**：
- 任务参数始终存在于 `oban_jobs` 表中（代码证实）
- 即使状态为 `discarded`（待确认），参数仍可查询和重跑

#### 数据丢失边界（待确认）

**⚠️ 可能不可恢复的场景（待确认）**：

| 场景 | 丢失原因 | 证据状态 |
|-----|---------|---------|
| **主动 `{:discard, reason}`** | 代码主动放弃重试 | 代码中有此返回值，但状态映射待确认 |
| **超过 `max_attempts`** | 重试次数耗尽 | 配置存在，但状态变化待确认 |
| **PostgreSQL 数据丢失** | 任务存储本身丢失 | 极端场景，依赖数据库 HA |

**关键依赖**：
- Oban 任务的可恢复性 **完全依赖 PostgreSQL 的可靠性**（代码证实：`repo: Plausible.Repo`）
- 只要 PostgreSQL 不丢数据，任务参数就可恢复

### 6.3 两条链路核心差异总结

| 维度 | WriteBuffer 实时链路 | Oban 后台任务链路 |
|-----|---------------------|------------------|
| **设计目标** | 高吞吐量、低延迟 | 高可靠性、异步执行 |
| **数据持久性** | ❌ 仅内存 | ✅ PostgreSQL 持久化 |
| **失败代价** | ⚠️ 数据永久丢失 | ✅ 任务参数可恢复 |
| **恢复机制** | ❌ 无 | ⚠️ 部分依赖 Oban 框架行为 |
| **适用场景** | 可接受少量数据丢失的实时流 | 不可接受数据丢失的异步任务 |

### 6.4 风险评估

| 链路 | 数据丢失风险 | 风险等级 | 关键依据 |
|-----|-------------|---------|---------|
| **WriteBuffer** | **高** | 🔴 高 | 内存缓冲 + 无重试 + 无持久化 |
| **Oban** | **低** | 🟢 低 | PostgreSQL 持久化 + 重试机制 |

### 6.5 建议优先级

#### 针对 WriteBuffer 链路：

1. **P0 - 紧急**：
   - 评估生产环境 ClickHouse 稳定性
   - 确认当前数据丢失风险的接受度

2. **P1 - 高**：
   - 为 `WriteBuffer.do_flush/1` 添加基础重试机制（3-5次，指数退避）
   - 添加 Sentry 告警上报

3. **P2 - 中**：
   - 实现死信队列或持久化缓冲
   - 设计进程启动时的数据恢复逻辑

4. **P3 - 低**：
   - 引入消息队列进行架构升级
   - 启用 `recovery_id` 实现幂等写入

#### 针对 Oban 链路：

1. **P1 - 建议确认**：
   - 查阅 Oban 官方文档，确认默认 `max_attempts` 和 `backoff` 策略
   - 确认返回值与状态的映射关系
   - 确认 Lifeline 插件的具体行为

2. **P2 - 中**：
   - 建立统一的 Oban 监控告警（discarded 任务、重试次数异常）
   - 为 `ClickhouseCleanSites` 等关键 Worker 显式设置 `max_attempts` 和 `backoff`

---

## 七、相关代码文件索引

### 7.1 WriteBuffer 链路相关

| 文件路径 | 功能说明 | 可证实的内容 |
|---------|---------|-------------|
| `lib/plausible/ingestion/write_buffer.ex` | 核心写入缓冲区实现 | `query!` 使用、无重试、无异常捕获、内存缓冲 |
| `lib/plausible/event/write_buffer.ex` | 事件缓冲区封装 | - |
| `lib/plausible/session/write_buffer.ex` | 会话缓冲区封装 | - |
| `lib/plausible/ingestion/persistor.ex` | 持久化后端选择器 | 后端选择逻辑 |
| `lib/plausible/ingestion/persistor/embedded.ex` | 本地写入实现 | - |
| `lib/plausible/ingestion/persistor/remote.ex` | 远程写入实现 | HTTP 重试 3 次、重试条件 |
| `lib/plausible/ingestion/counters.ex` | 计数器刷新 | try/catch、Sentry 上报、无重试 |
| `lib/plausible/ingest_repo.ex` | ClickHouse 写入 Repo | - |
| `lib/plausible/application.ex` | 监督树配置 | `:one_for_one` 策略 |
| `config/runtime.exs` | 运行时配置 | 5秒刷新、100KB 缓冲区 |

### 7.2 Oban 链路相关

| 文件路径 | 功能说明 | 可证实的内容 |
|---------|---------|-------------|
| `config/runtime.exs:864-876` | Oban 核心配置 | PostgreSQL 持久化、Lifeline 配置 |
| `lib/workers/import_analytics.ex` | 数据导入任务 | `max_attempts: 3`、backoff 固定 5 分钟、返回 `:ok`/`{:discard, ...}` |
| `lib/workers/export_analytics.ex` | 数据导出任务 | `max_attempts: 3`、无 backoff 实现 |
| `lib/workers/purge_cdn_cache.ex` | CDN 缓存清理 | `max_attempts: 5`、指数 backoff、返回 `{:ok, ...}`/`{:error, ...}`/`{:discard, ...}` |
| `lib/workers/send_email_report.ex` | 邮件报表 | `max_attempts: 1`（不重试） |
| `lib/workers/clickhouse_clean_sites.ex` | ClickHouse 数据清理 | 无 `max_attempts`、无 `backoff`、使用 `query!` |
| `lib/plausible/application.ex:204` | Oban 监督树启动 | - |

---

## 附录：待确认问题清单

以下问题需要查阅 Oban 官方文档或源码进一步确认：

### A.1 默认行为确认

| 问题 | 建议查阅的文档 |
|-----|---------------|
| Oban 默认 `max_attempts` 值是多少？ | Oban.Worker 文档 |
| Oban 默认 `backoff` 策略是什么？ | Oban.Worker 文档 |
| `perform/1` 返回 `:ok` 后任务状态是什么？ | Oban.Worker 文档 |
| `perform/1` 返回 `{:discard, reason}` 后任务状态是什么？ | Oban.Worker 文档 |
| `perform/1` 返回 `{:error, reason}` 后任务状态是什么？ | Oban.Worker 文档 |
| `perform/1` 抛出异常后任务状态是什么？ | Oban.Worker 文档 |

### A.2 插件行为确认

| 问题 | 建议查阅的文档 |
|-----|---------------|
| `Oban.Plugins.Lifeline` 的具体行为是什么？ | Oban.Plugins.Lifeline 文档 |
| `rescue_after` 配置如何影响任务状态？ | 同上 |
| 超过 `max_attempts` 后任务状态如何变化？ | Oban.Worker 文档 |

### A.3 状态流转确认

| 问题 | 建议查阅的文档 |
|-----|---------------|
| Oban 任务有哪些状态？ | Oban.Job 文档 |
| 状态之间的流转规则是什么？ | Oban 架构文档 |
| `discarded` 状态的任务可以手动重跑吗？ | Oban 操作文档 |
