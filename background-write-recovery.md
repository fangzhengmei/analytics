# ClickHouse 后台写入失败恢复与重试机制分析报告

## 概述

本文档分析了 Plausible Analytics 项目中后台任务向 ClickHouse 写入事件数据时的失败恢复机制和重试策略设计。

---

## 一、数据写入架构

### 1.1 整体写入流程

```
事件接收 → Persistor 选择后端 → WriteBuffer 缓冲 → 定期/定量刷新 → ClickHouse
```

### 1.2 持久化后端选择

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

## 二、WriteBuffer 核心机制

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

---

## 三、重试机制分析

### 3.1 当前实现的问题

**⚠️ 关键问题：WriteBuffer 没有内置重试机制**

在 `do_flush/1` 函数中使用了 `IngestRepo.query!`（带感叹号的版本）：

```elixir
IngestRepo.query!(insert_sql, [header | buffer], insert_opts)
```

这意味着：

| 问题 | 影响 |
|-----|------|
| 使用 `query!` 而非 `query` | 失败时抛出异常，不返回错误 tuple |
| 无 try/rescue 包裹 | 异常直接向上传播 |
| 无重试循环 | 一次失败即终止 |
| 无死信队列 | 失败数据永久丢失 |

### 3.2 各组件的重试策略

#### 3.2.1 Remote 后端的 HTTP 重试

`Plausible.Ingestion.Persistor.Remote` 有 HTTP 层面的重试：

**配置** (`lib/plausible/ingestion/persistor/remote.ex:9`)：
```elixir
@max_transient_retries 3
```

**重试条件** (`lib/plausible/ingestion/persistor/remote.ex:87-95`)：
```elixir
defp handle_transient_error(_request, %Req.HTTPError{protocol: :http2, reason: :disconnected}) do
  true
end

defp handle_transient_error(_request, %Req.HTTPError{protocol: :http2, reason: :unprocessed}) do
  true
end

defp handle_transient_error(_reqeust, _response), do: false
```

**限制**：
- 仅适用于 Remote 后端（HTTP 调用）
- 仅重试 HTTP/2 特定连接错误
- **不适用**于 Embedded 后端的 ClickHouse 直接写入

#### 3.2.2 Counters 模块的错误处理

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

## 四、监督与恢复策略

### 4.1 监督树结构

**WriteBuffer 在监督树中的位置** (`lib/plausible/application.ex:197-198`)：

```elixir
Supervisor.child_spec(Plausible.Event.WriteBuffer, id: Plausible.Event.WriteBuffer),
Supervisor.child_spec(Plausible.Session.WriteBuffer, id: Plausible.Session.WriteBuffer),
```

**顶层监督策略** (`lib/plausible/application.ex:212`)：
```elixir
opts = [strategy: :one_for_one, name: Plausible.Supervisor]
```

### 4.2 故障恢复流程

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

### 4.3 Session.Transfer 机制

**⚠️ 注意：Session.Transfer 不是 ClickHouse 写入恢复机制**

`Plausible.Session.Transfer` (`lib/plausible/session/transfer.ex`) 的设计目的：
- 跨部署实例的会话缓存迁移
- 优雅重启时保持会话状态
- 通过 Unix Domain Socket 传输

**适用场景**：滚动升级、蓝绿部署时的会话迁移
**不适用场景**：ClickHouse 写入失败的数据恢复

### 4.4 recovery_id 字段

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

## 五、失败场景分析

### 5.1 可能的失败场景

| 场景 | 触发条件 | 当前行为 | 数据丢失风险 |
|-----|---------|---------|-------------|
| ClickHouse 连接中断 | 网络分区、服务重启 | query! 抛出异常，进程崩溃 | ⚠️ 高 |
| ClickHouse 负载过高 | 写入超时、队列满 | 同上 | ⚠️ 高 |
| 数据格式错误 | 罕见的 schema 不兼容 | 同上 | 低（可重复） |
| 进程优雅关闭 | SIGTERM 信号 | terminate/2 中刷新 | ✅ 低 |
| 进程强制终止 | SIGKILL、OOM | 无法执行 terminate | ⚠️ 高 |

### 5.2 数据丢失窗口

在当前设计下，数据可能在以下时间窗口丢失：

1. **WriteBuffer 内部缓冲**：约 5 秒或 100KB 数据
2. **无重试机制**：首次失败即丢失
3. **进程崩溃后**：新进程无法恢复旧数据

---

## 六、改进建议

### 6.1 短期改进（低风险）

#### 6.1.1 添加基础重试机制

修改 `do_flush/1` 添加重试：

```elixir
defp do_flush(state, retries_left \\ 3) do
  # ... 提取 state ...
  
  case buffer do
    [] -> nil
    _not_empty ->
      try do
        IngestRepo.query!(insert_sql, [header | buffer], insert_opts)
      rescue
        e in DBConnection.ConnectionError ->
          if retries_left > 0 do
            Logger.warning("Flush failed, retrying... #{retries_left} attempts left")
            :timer.sleep(:timer.seconds(1))
            do_flush(state, retries_left - 1)
          else
            # 重试耗尽，考虑死信队列
            Logger.error("Flush failed after all retries: #{inspect(e)}")
            # TODO: 写入死信队列
            reraise e, __STACKTRACE__
          end
      end
  end
end
```

#### 6.1.2 使用 `query` 替代 `query!`

使用返回 `{:ok, result}` 或 `{:error, reason}` 的版本，便于错误处理。

### 6.2 中期改进（中等风险）

#### 6.2.1 实现死信队列 (DLQ)

```
写入失败 → 记录到本地文件/PostgreSQL → 后台定期重试 → 成功后清理
```

#### 6.2.2 持久化缓冲区

- 将缓冲区数据定期写入磁盘
- 进程启动时从磁盘恢复
- 可使用 DETS 或简单的文件追加

### 6.3 长期改进（架构级）

#### 6.3.1 引入消息队列

- 使用 Kafka / RabbitMQ 作为事件缓冲
- 至少一次交付保证
- 消费者端实现重试和死信队列

#### 6.3.2 启用 recovery_id

- 实现基于 `recovery_id` 的幂等写入
- 支持重复数据的去重和恢复

---

## 七、相关代码文件索引

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

---

## 八、总结

### 当前状态

| 维度 | 评估 |
|-----|------|
| 重试机制 | ❌ WriteBuffer 无重试；Remote 仅 HTTP 层重试 |
| 故障恢复 | ❌ 进程崩溃后数据丢失；仅 Supervisor 重启进程 |
| 死信队列 | ❌ 未实现 |
| 持久化缓冲 | ❌ 仅内存缓冲，无磁盘备份 |
| 监控告警 | ⚠️ Counters 有 Sentry 上报；WriteBuffer 无 |

### 核心风险

当前设计在 ClickHouse 不可用时**存在数据丢失风险**，因为：
1. `query!` 抛出异常导致进程崩溃
2. 内存缓冲区在进程重启后清空
3. 无重试机制从瞬时故障中恢复

### 建议优先级

1. **P0 - 紧急**：评估生产环境 ClickHouse 稳定性，确认风险接受度
2. **P1 - 高**：为 WriteBuffer 添加基础重试机制（3-5次，指数退避）
3. **P2 - 中**：实现死信队列或持久化缓冲
4. **P3 - 低**：引入消息队列进行架构升级
