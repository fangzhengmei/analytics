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

### 2.3 数据写入实现

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

### 2.4 关键问题：无重试机制

**⚠️ 核心问题：WriteBuffer 没有内置重试机制**

在 `do_flush/1` 函数中使用了 `IngestRepo.query!`（带感叹号的版本）：

| 问题 | 影响 |
|-----|------|
| 使用 `query!` 而非 `query` | 失败时抛出异常，不返回错误 tuple |
| 无 try/rescue 包裹 | 异常直接向上传播 |
| 无重试循环 | 一次失败即终止 |
| 无死信队列 | 失败数据永久丢失 |

### 2.5 故障恢复流程

当 WriteBuffer 写入失败时：

```
1. IngestRepo.query! 抛出异常
         ↓
2. GenServer 进程崩溃
         ↓
3. Supervisor 检测到进程终止
         ↓
4. 使用 :one_for_one 策略重启进程
         ↓
5. 新进程启动，缓冲区为空
         ↓
6. ❌ 崩溃前缓冲的数据永久丢失
```

### 2.6 相关组件的重试策略

#### 2.6.1 Remote 后端的 HTTP 重试

`Plausible.Ingestion.Persistor.Remote` 有 HTTP 层面的重试：

**配置** (`lib/plausible/ingestion/persistor/remote.ex:9`)：
```elixir
@max_transient_retries 3
```

**重试条件** (`lib/plausible/ingestion/persistor/remote.ex:87-95`)：
```elixir
defp handle_transient_error(_request, %Req.HTTPError{protocol: :http2, reason: :disconnected}), do: true
defp handle_transient_error(_request, %Req.HTTPError{protocol: :http2, reason: :unprocessed}), do: true
defp handle_transient_error(_reqeust, _response), do: false
```

**限制**：
- 仅适用于 Remote 后端（HTTP 调用）
- 仅重试 HTTP/2 特定连接错误
- **不适用**于 Embedded 后端的 ClickHouse 直接写入

#### 2.6.2 Counters 模块的错误处理

`Plausible.Ingestion.Counters` 有基础的异常捕获：

**代码** (`lib/plausible/ingestion/counters.ex:84-95`)：
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
- 使用 `try/catch` 捕获异常
- 上报到 Sentry 用于告警
- **但没有重试，数据仍然丢失**

---

## 三、Oban 后台任务调度层

### 3.1 Oban 基础架构

Oban 是一个基于 PostgreSQL 的可靠作业处理库，用于处理异步后台任务。

**核心配置** (`config/runtime.exs:864-876`)：
```elixir
config :plausible, Oban,
  repo: Plausible.Repo,           # 任务数据存储在 PostgreSQL
  plugins: [
    # 保留 30 天历史
    {Oban.Plugins.Pruner, max_age: sixty_days_in_seconds},
    # 定时任务调度
    {Oban.Plugins.Cron, crontab: if(cron_enabled, do: crontab, else: [])},
    # 救援孤立任务（执行超过 2 小时）
    {Oban.Plugins.Lifeline, rescue_after: :timer.minutes(120)},
    # 每日凌晨 1 点重建索引
    {Oban.Plugins.Reindexer, schedule: "0 1 * * *"}
  ],
  queues: queues,
  peer: if(cron_enabled, do: Oban.Peers.Postgres, else: false)
```

### 3.2 任务生命周期与状态流转

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌───────────┐
│  available │ ──▶ │ scheduled │ ──▶ │ executing │ ──▶ │ completed │
│  (可执行)   │     │ (已调度)   │     │ (执行中)   │     │ (已完成)   │
└──────────┘     └──────────┘     └──────┬───┘     └───────────┘
                                            │
                              执行失败       │ 超过 max_attempts
                              重试          │
                                            ▼
                                       ┌──────────┐
                                       │ discarded │
                                       │ (已丢弃)  │
                                       └──────────┘
```

**状态说明**：

| 状态 | 说明 | 数据可恢复 |
|-----|------|-----------|
| `available` | 任务已入队，等待执行 | ✅ 是，存储在 PostgreSQL |
| `scheduled` | 任务已调度，在指定时间执行 | ✅ 是 |
| `executing` | 任务正在执行 | ⚠️ 进行中，失败可重试 |
| `completed` | 任务执行成功 | ❌ 已完成 |
| `discarded` | 任务失败且超过最大重试次数 | ❌ 已丢弃，需手动恢复 |
| `cancelled` | 任务被取消 | ❌ 已取消 |

### 3.3 重试机制详解

#### 3.3.1 最大重试次数 (max_attempts)

每个 Worker 可以自定义最大重试次数：

| Worker | max_attempts | 说明 |
|--------|-------------|------|
| `ImportAnalytics` | 3 | 数据导入，失败后可手动重跑 |
| `ExportAnalytics` | 3 | 数据导出 |
| `NotifyExportedAnalytics` | 5 | 导出结果邮件通知 |
| `PurgeCDNCache` | 5 | CDN 缓存清理 |
| `SendEmailReport` | 1 | **不重试**，邮件报表发送 |
| `ClickhouseCleanSites` | **默认 20** | 未显式设置，使用 Oban 默认值 |

**配置示例** (`lib/workers/import_analytics.ex:9-12`)：
```elixir
use Oban.Worker,
  queue: :analytics_imports,
  max_attempts: 3,
  unique: [fields: [:args], keys: [:import_id], period: 60]
```

#### 3.3.2 回退策略 (backoff)

Oban 支持自定义回退策略，控制重试间隔。

**默认策略**（Oban 内置）：
```elixir
# 指数退避：attempt + 3 秒
# 第1次重试: 1 + 3 = 4秒
# 第2次重试: 2 + 3 = 5秒
# 第3次重试: 3 + 3 = 6秒
# ...
```

**自定义策略示例 1 - 固定间隔** (`lib/workers/import_analytics.ex:51-54`)：
```elixir
@impl Oban.Worker
def backoff(_job) do
  # 固定 5 分钟 (300秒) 间隔
  300
end
```

**自定义策略示例 2 - 指数退避** (`lib/workers/purge_cdn_cache.ex:33-36`)：
```elixir
@impl Oban.Worker
def backoff(%Oban.Job{attempt: attempt}) do
  # 指数退避，起始 3 分钟
  # attempt=1: 2^0 * 180 = 180秒 = 3分钟
  # attempt=2: 2^1 * 180 = 360秒 = 6分钟
  # attempt=3: 2^2 * 180 = 720秒 = 12分钟
  # attempt=4: 2^3 * 180 = 1440秒 = 24分钟
  # attempt=5: 2^4 * 180 = 2880秒 = 48分钟
  trunc(:math.pow(2, attempt - 1) * 180)
end
```

### 3.4 任务终止条件

任务终止（不再重试）的条件：

| 条件 | 触发方式 | 结果 |
|-----|---------|------|
| 显式成功 | `perform/1` 返回 `:ok` 或 `{:ok, result}` | 状态变为 `completed` |
| 显式放弃 | `perform/1` 返回 `{:discard, reason}` | 状态变为 `discarded`，**不重试** |
| 超过重试次数 | `attempt >= max_attempts` | 状态变为 `discarded` |
| 显式取消 | 调用 `Oban.cancel_job/1` | 状态变为 `cancelled` |

**代码示例** (`lib/workers/import_analytics.ex:29-47`)：
```elixir
@impl Oban.Worker
def perform(%Oban.Job{args: %{"import_id" => import_id} = args}) do
  case import_api.run_import(site_import, args) do
    {:ok, site_import} ->
      import_complete(site_import)
      :ok   # ✅ 成功完成

    {:error, error, error_opts} ->
      # 业务错误处理
      import_fail(site_import, error_opts)
      {:discard, error}   # ❌ 主动放弃，不再重试
  end
end
```

### 3.5 Oban 插件机制

| 插件 | 功能 | 配置 |
|-----|------|------|
| `Oban.Plugins.Pruner` | 自动清理已完成的任务记录 | `max_age: 30天` |
| `Oban.Plugins.Cron` | 定时任务调度（类似 crontab） | 可配置周期性任务 |
| `Oban.Plugins.Lifeline` | 救援孤立任务 | `rescue_after: 120分钟` |
| `Oban.Plugins.Reindexer` | 定期重建数据库索引 | `schedule: "0 1 * * *"` |

**Lifeline 插件说明**：
- 检测执行时间超过 120 分钟的"孤立"任务
- 将状态从 `executing` 改回 `available`
- 允许其他 worker 重新执行
- 适用于：进程崩溃、网络分区导致任务卡住

### 3.6 涉及 ClickHouse 写入的后台任务

| Worker | 功能 | ClickHouse 操作 | 重试次数 |
|--------|------|----------------|---------|
| `ClickhouseCleanSites` | 清理已删除站点的数据 | `ALTER TABLE ... DELETE` | 默认 20 |
| `ImportAnalytics` | 导入历史数据 | 批量 `INSERT` | 3 |
| `ExportAnalytics` | 导出数据 | `SELECT` 查询 | 3 |
| `LocationsSync` | 同步地理位置数据 | 写入 `locations` 表 | 默认 20 |

**ClickhouseCleanSites 示例** (`lib/workers/clickhouse_clean_sites.ex:36-54`)：
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

**注意**：此 Worker 也使用了 `query!`，但因为运行在 Oban 中：
- 失败时任务状态变为 `available` 并按 backoff 重试
- 重试次数耗尽后变为 `discarded`
- 任务参数（deleted_sites）存储在 PostgreSQL 中，**可恢复**

---

## 四、两条链路对比分析

### 4.1 核心差异对比表

| 维度 | WriteBuffer 实时链路 | Oban 后台任务链路 |
|-----|---------------------|------------------|
| **数据存储** | 内存缓冲（进程内） | PostgreSQL 持久化 |
| **重试机制** | ❌ 无 | ✅ 完善（max_attempts + backoff） |
| **故障恢复** | ❌ 进程崩溃即丢失 | ✅ 应用重启不影响 |
| **数据延迟** | 秒级（5秒刷新间隔） | 可控（异步调度） |
| **适用场景** | 高吞吐量实时事件 | 异步批量操作 |
| **失败可恢复** | ❌ 不可恢复 | ✅ 可恢复（手动/自动） |
| **任务状态追踪** | ❌ 无 | ✅ 完整状态管理 |

### 4.2 详细对比分析

#### 4.2.1 数据持久性

**WriteBuffer 链路**：
```
┌─────────────────────────────────────────────────────────────┐
│                      进程内存空间                             │
│  ┌──────────────┐                                            │
│  │ WriteBuffer  │  ←── 数据仅存在于内存                       │
│  │  (GenServer) │                                            │
│  └──────┬───────┘                                            │
│         │ 进程崩溃 (SIGKILL/OOM)                             │
│         ▼                                                      │
│   ⚠️ 数据永久丢失 ❌                                           │
└─────────────────────────────────────────────────────────────┘
```

**Oban 链路**：
```
┌─────────────────────────────────────────────────────────────┐
│                      PostgreSQL                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                  oban_jobs 表                        │    │
│  │  • args (任务参数)         ←── 持久化存储            │    │
│  │  • state (状态)                                     │    │
│  │  • attempt (重试次数)                               │    │
│  │  • max_attempts (最大重试)                          │    │
│  └───────────────────┬─────────────────────────────────┘    │
│                      │                                        │
│         ┌────────────┼────────────┐                          │
│         ▼            ▼            ▼                          │
│   应用重启      Worker崩溃    网络分区                        │
│         │            │            │                          │
│         └────────────┼────────────┘                          │
│                      ▼                                        │
│              ✅ 任务自动恢复执行                              │
└─────────────────────────────────────────────────────────────┘
```

#### 4.2.2 失败处理流程

**WriteBuffer 失败流程**：

```
事件写入 WriteBuffer
        │
        ▼
┌───────────────┐
│ 内存缓冲累积   │
│ (5秒/100KB)  │
└───────┬───────┘
        │
        ▼ 定时/定量刷新
┌───────────────┐
│ IngestRepo.   │
│ query!()      │ ──▶ ClickHouse
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
         GenServer 进程崩溃
                │
                ▼
         Supervisor 重启进程
                │
                ▼
         新进程缓冲区为空
                │
                ▼
         ⚠️ 原数据永久丢失 ❌
```

**Oban 失败流程**：

```
任务入队 Oban.insert!()
        │
        ▼
┌───────────────┐
│ PostgreSQL    │
│ oban_jobs 表  │ ←── 持久化
│ state: avail  │
└───────┬───────┘
        │
        ▼ Worker 拉取执行
┌───────────────┐
│ perform()     │ ──▶ ClickHouse 操作
└───────┬───────┘
        │
   ┌────┴────┐
   │ 成功?    │
   └────┬────┘
        │
   是 ──┴── 否
   │          │
   ▼          │
completed     │
              ▼
       捕获异常/返回错误
              │
              ▼
    ┌─────────────────┐
    │ attempt <       │
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
   
   ✅ 任务参数始终可恢复
```

#### 4.2.3 恢复边界

| 链路 | 可恢复场景 | 不可恢复场景 | 恢复边界 |
|-----|-----------|-------------|---------|
| **WriteBuffer** | 无 | • 进程崩溃<br>• ClickHouse 不可用<br>• 网络分区 | ❌ 无恢复机制 |
| **Oban** | • Worker 进程崩溃<br>• 应用重启<br>• 瞬时网络错误<br>• ClickHouse 瞬时不可用 | • 业务逻辑错误（主动 discard）<br>• PostgreSQL 数据丢失 | PostgreSQL 持久化 |

### 4.3 数据丢失风险分析

#### WriteBuffer 链路风险点

| 风险场景 | 概率 | 数据丢失量 | 缓解措施 |
|---------|------|-----------|---------|
| ClickHouse 连接中断 | 中 | 约 5 秒 / 100KB 数据 | 当前：无 |
| 进程 OOM 被杀 | 中 | 约 5 秒数据 | 当前：无 |
| 优雅关闭 (SIGTERM) | 低 | ✅ 无丢失 | `terminate/2` 刷新 |
| 强制终止 (SIGKILL) | 低 | 约 5 秒数据 | 当前：无 |

#### Oban 链路风险点

| 风险场景 | 概率 | 数据丢失量 | 缓解措施 |
|---------|------|-----------|---------|
| ClickHouse 连接中断 | 中 | ✅ 无丢失 | 自动重试 |
| Worker 进程崩溃 | 中 | ✅ 无丢失 | Lifeline 救援 |
| 应用重启 | 低 | ✅ 无丢失 | PostgreSQL 持久化 |
| PostgreSQL 数据丢失 | 极低 | ❌ 任务丢失 | 数据库备份 |
| 超过 max_attempts | 中 | ❌ 任务丢弃 | 手动重跑 / 监控告警 |

---

## 五、监督与恢复策略

### 5.1 监督树结构

**WriteBuffer 在监督树中的位置** (`lib/plausible/application.ex:197-198`)：

```elixir
Supervisor.child_spec(Plausible.Event.WriteBuffer, id: Plausible.Event.WriteBuffer),
Supervisor.child_spec(Plausible.Session.WriteBuffer, id: Plausible.Session.WriteBuffer),
```

**顶层监督策略** (`lib/plausible/application.ex:212`)：
```elixir
opts = [strategy: :one_for_one, name: Plausible.Supervisor]
```

**WriteBuffer 默认 child_spec**：
GenServer 默认的 child_spec 配置：
- `restart: :permanent`（永久进程，崩溃后重启）
- `shutdown: 5000`（优雅关闭超时 5 秒）

这意味着：
- WriteBuffer 崩溃后会被 Supervisor 自动重启
- **但重启后的新进程没有旧数据**

### 5.2 其他相关机制

#### 5.2.1 Session.Transfer 机制

**⚠️ 注意：Session.Transfer 不是 ClickHouse 写入恢复机制**

`Plausible.Session.Transfer` (`lib/plausible/session/transfer.ex`) 的设计目的：
- 跨部署实例的会话缓存迁移
- 优雅重启时保持会话状态
- 通过 Unix Domain Socket 传输

**适用场景**：滚动升级、蓝绿部署时的会话迁移
**不适用场景**：ClickHouse 写入失败的数据恢复

#### 5.2.2 recovery_id 字段

数据库 schema 中有 `recovery_id` 字段：

**迁移文件** (`priv/ingest_repo/migrations/20251106111224_add_recovery_id_to_events_sessions.exs`)：
```elixir
ALTER TABLE sessions_v2 ADD COLUMN recovery_id UInt64
ALTER TABLE events_v2 ADD COLUMN recovery_id UInt64
```

**当前状态**：
- 字段仅在企业版中添加
- **应用层代码中未使用此字段**
- 可能是为未来恢复机制预留的设计

---

## 六、失败场景分析

### 6.1 WriteBuffer 链路失败场景

| 场景 | 触发条件 | 当前行为 | 数据丢失风险 |
|-----|---------|---------|-------------|
| ClickHouse 连接中断 | 网络分区、服务重启 | query! 抛出异常，进程崩溃 | ⚠️ 高 |
| ClickHouse 负载过高 | 写入超时、队列满 | 同上 | ⚠️ 高 |
| 数据格式错误 | 罕见的 schema 不兼容 | 同上 | 低（可重复） |
| 进程优雅关闭 | SIGTERM 信号 | terminate/2 中刷新 | ✅ 低 |
| 进程强制终止 | SIGKILL、OOM | 无法执行 terminate | ⚠️ 高 |

### 6.2 Oban 链路失败场景

| 场景 | 触发条件 | 当前行为 | 数据丢失风险 |
|-----|---------|---------|-------------|
| ClickHouse 连接中断 | 网络分区、服务重启 | 按 backoff 自动重试 | ✅ 无 |
| Worker 进程崩溃 | OOM、Bug | Lifeline 救援，任务继续 | ✅ 无 |
| 应用重启 | 部署、维护 | 任务保留在 PostgreSQL | ✅ 无 |
| 超过最大重试 | max_attempts 耗尽 | 状态变为 discarded | ⚠️ 需手动恢复 |
| PostgreSQL 故障 | 数据库宕机 | 任务无法入队/执行 | ⚠️ 极低（依赖 HA）|

### 6.3 数据丢失窗口

**WriteBuffer 链路**：
1. **WriteBuffer 内部缓冲**：约 5 秒或 100KB 数据
2. **无重试机制**：首次失败即丢失
3. **进程崩溃后**：新进程无法恢复旧数据

**Oban 链路**：
1. **执行中状态**：瞬时故障可重试恢复
2. **超过重试次数**：需手动检查 discarded 任务
3. **PostgreSQL 故障**：极端场景下的风险

---

## 七、改进建议

### 7.1 WriteBuffer 链路改进

#### 7.1.1 短期改进（低风险）

**添加基础重试机制**：

修改 `do_flush/1` 添加重试：

```elixir
defp do_flush(state, retries_left \\ 3) do
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
      try do
        Logger.notice("Flushing #{buffer_size} byte(s) RowBinary from #{name}")
        IngestRepo.query!(insert_sql, [header | buffer], insert_opts)
      rescue
        e in DBConnection.ConnectionError ->
          if retries_left > 0 do
            backoff_seconds = :math.pow(2, 3 - retries_left) |> trunc()
            Logger.warning(
              "Flush to #{name} failed, retrying in #{backoff_seconds}s... " <>
              "#{retries_left} attempts left. Error: #{inspect(e)}"
            )
            :timer.sleep(:timer.seconds(backoff_seconds))
            do_flush(state, retries_left - 1)
          else
            # 重试耗尽，写入死信队列
            Logger.error("Flush to #{name} failed after all retries: #{inspect(e)}")
            write_to_dead_letter_queue(name, buffer, e)
            # 继续抛出异常让进程重启，避免阻塞
            reraise e, __STACKTRACE__
          end
      end
  end
end
```

#### 7.1.2 中期改进（中等风险）

**实现死信队列 (DLQ)**：

```
写入失败 → 记录到本地文件/PostgreSQL → 后台定期重试 → 成功后清理
```

设计要点：
1. **存储**：使用 PostgreSQL 表或本地文件
2. **结构**：包含缓冲区数据、时间戳、错误信息、重试次数
3. **重试**：独立的后台任务定期处理 DLQ
4. **监控**：DLQ 大小作为关键指标告警

**持久化缓冲区**：
- 将缓冲区数据定期写入磁盘（如 DETS 或追加文件）
- 进程启动时从磁盘恢复未刷新的数据
- 类似 Oban 的思路，但更轻量

#### 7.1.3 长期改进（架构级）

**引入消息队列**：

```
事件接收 → Kafka/RabbitMQ → 消费者 → WriteBuffer → ClickHouse
                    ↓
              至少一次交付保证
```

优势：
- 消息队列本身具备持久化和重试机制
- 消费者进程崩溃不影响消息
- 支持消费者组横向扩展

**启用 recovery_id**：
- 实现基于 `recovery_id` 的幂等写入
- 支持重复数据的去重和恢复
- 与消息队列结合实现精确一次语义

### 7.2 Oban 链路改进

Oban 链路已经相对完善，但仍有可优化点：

#### 7.2.1 监控与告警

当前部分 Worker 已有 Sentry 上报，建议：
1. **统一监控**：所有 discarded 任务自动触发告警
2. **指标采集**：
   - `oban_jobs_count` 按状态、queue 统计
   - `oban_job_duration` 执行时长
   - `oban_retries_count` 重试次数分布

#### 7.2.2 死信任务处理

对于 `discarded` 状态的任务：
1. **定期审计**：每日检查 discarded 任务
2. **手动重跑**：提供管理界面或脚本
3. **自动清理**：超过一定时间（如 7 天）后清理

#### 7.2.3 ClickHouse 操作优化

对于涉及 ClickHouse 的 Worker：
1. **使用 `query` 替代 `query!`**：更优雅的错误处理
2. **事务性操作**：对于多步 ClickHouse 操作，考虑幂等设计
3. **批量操作拆分**：大批次拆分为小批次，降低失败影响范围

---

## 八、相关代码文件索引

### 8.1 WriteBuffer 链路相关

| 文件路径 | 功能说明 |
|---------|---------|
| `lib/plausible/ingestion/write_buffer.ex` | 核心写入缓冲区实现 |
| `lib/plausible/event/write_buffer.ex` | 事件缓冲区封装 |
| `lib/plausible/session/write_buffer.ex` | 会话缓冲区封装 |
| `lib/plausible/ingestion/persistor.ex` | 持久化后端选择器 |
| `lib/plausible/ingestion/persistor/embedded.ex` | 本地写入实现 |
| `lib/plausible/ingestion/persistor/remote.ex` | 远程写入实现（含 HTTP 重试） |
| `lib/plausible/ingestion/counters.ex` | 计数器刷新（含异常捕获） |
| `lib/plausible/ingest_repo.ex` | ClickHouse 写入 Repo |
| `lib/plausible/application.ex` | 监督树配置 |
| `config/runtime.exs` | 运行时配置（缓冲区参数） |

### 8.2 Oban 链路相关

| 文件路径 | 功能说明 |
|---------|---------|
| `config/runtime.exs:864-876` | Oban 核心配置 |
| `lib/workers/clickhouse_clean_sites.ex` | ClickHouse 数据清理任务 |
| `lib/workers/import_analytics.ex` | 数据导入任务（含自定义 backoff） |
| `lib/workers/export_analytics.ex` | 数据导出任务 |
| `lib/workers/purge_cdn_cache.ex` | CDN 缓存清理（指数 backoff 示例） |
| `lib/workers/send_email_report.ex` | 邮件报表（不重试示例） |
| `lib/plausible/application.ex:204` | Oban 监督树启动 |

---

## 九、总结

### 9.1 当前状态评估

| 维度 | WriteBuffer 实时链路 | Oban 后台任务链路 |
|-----|---------------------|------------------|
| **重试机制** | ❌ 无重试；Remote 仅 HTTP 层重试 | ✅ 完善（max_attempts + 可定制 backoff） |
| **故障恢复** | ❌ 进程崩溃后数据丢失；仅 Supervisor 重启进程 | ✅ 应用重启不影响；Lifeline 救援孤立任务 |
| **死信队列** | ❌ 未实现 | ⚠️ 有 discarded 状态，需手动处理 |
| **持久化缓冲** | ❌ 仅内存缓冲，无磁盘备份 | ✅ PostgreSQL 完整持久化 |
| **监控告警** | ⚠️ Counters 有 Sentry 上报；WriteBuffer 无 | ⚠️ 部分 Worker 有上报；需统一监控 |
| **数据可恢复** | ❌ 不可恢复 | ✅ 可恢复（除非主动 discard） |

### 9.2 核心风险

**WriteBuffer 链路**：
当前设计在 ClickHouse 不可用时**存在数据丢失风险**，因为：
1. `query!` 抛出异常导致进程崩溃
2. 内存缓冲区在进程重启后清空
3. 无重试机制从瞬时故障中恢复

**Oban 链路**：
相对完善，但需注意：
1. 主动 `{:discard, reason}` 的任务不再重试
2. 超过 `max_attempts` 的任务需手动干预
3. 依赖 PostgreSQL 的高可用

### 9.3 建议优先级

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

1. **P1 - 高**：
   - 建立统一的 Oban 监控告警（discarded 任务、重试次数异常）
   - 制定 discarded 任务的处理流程

2. **P2 - 中**：
   - 统一 Worker 的错误处理模式
   - 考虑为关键任务增加 max_attempts 或实现指数 backoff

### 9.4 架构建议

对于高可靠场景，建议：

1. **实时事件**：
   - 短期：增强 WriteBuffer 重试 + 死信队列
   - 长期：引入 Kafka/Pulsar 等消息队列

2. **异步任务**：
   - 继续使用 Oban，已足够可靠
   - 增强监控和 discarded 任务处理流程

3. **统一恢复机制**：
   - 考虑将 `recovery_id` 字段实际投入使用
   - 设计跨链路的统一数据恢复策略
