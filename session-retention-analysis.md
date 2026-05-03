# 访客会话与留存指标计算口径分析报告

## 目录
1. [会话边界定义](#会话边界定义)
2. [跨时间段统计方式](#跨时间段统计方式)
3. [访客识别与新老区分](#访客识别与新老区分)
4. [留存指标计算逻辑](#留存指标计算逻辑)
5. [不同维度下的口径一致性](#不同维度下的口径一致性)
6. [代码参考位置](#代码参考位置)

---

## 1. 会话边界定义

### 1.1 会话超时规则

**核心配置**（`config/config.exs:57-59`）：
```elixir
config :plausible,
  # 30分钟（毫秒）
  session_timeout: 1000 * 60 * 30,
  session_length_minutes: 30
```

**会话查找逻辑**（`lib/plausible/session/cache_store.ex:69-81`）：

会话边界判定采用**30分钟无活动超时**机制：
- 当新事件到达时，首先在内存缓存中查找 `{site_id, user_id}` 对应的会话
- 检查当前事件时间戳与会话最后更新时间戳的差值：
  - 差值 **≤ 30分钟**：视为同一会话的延续
  - 差值 **> 30分钟**：视为新会话开始

```elixir
defp find_session(event, user_id) do
  from_cache = Plausible.Cache.Adapter.get(:sessions, {event.site_id, user_id})
  
  case from_cache do
    nil ->
      nil
    session ->
      if NaiveDateTime.diff(event.timestamp, session.timestamp, :minute) <= 30 do
        session
      end
  end
end
```

### 1.2 会话ID生成规则

**会话ID唯一性**（`lib/plausible/clickhouse_session_v2.ex:80-82`）：

会话ID采用**加密强随机数**生成：
- 使用 `:crypto.strong_rand_bytes(8)` 生成8字节随机数
- 转换为无符号64位整数（UInt64）
- 每个新会话获得唯一的 `session_id`

```elixir
def random_uint64() do
  :crypto.strong_rand_bytes(8) |> :binary.decode_unsigned()
end
```

### 1.3 用户ID生成规则

**用户识别机制**（`lib/plausible/ingestion/event.ex:553-567`）：

用户ID基于以下因素**确定性生成**：
- `user_agent`：用户代理字符串
- `remote_ip`：客户端IP地址
- `domain`：站点域名
- `root_domain`：根域名（通过PublicSuffix解析）
- `salt`：每日轮换的盐值

```elixir
defp generate_user_id(request, domain, hostname, salt) do
  cond do
    is_nil(salt) -> nil
    is_nil(domain) -> nil
    true ->
      user_agent = request.user_agent || ""
      root_domain = get_root_domain(hostname)
      SipHash.hash!(salt, user_agent <> request.remote_ip <> domain <> root_domain)
  end
end
```

**盐值轮换机制**：
- 支持 `current` 和 `previous` 两个盐值
- 盐值轮换时，会尝试用新旧两个盐值查找会话（`lib/plausible/session/cache_store.ex:25`）
- 确保盐值切换期间会话的连续性

### 1.4 会话生命周期管理

**新会话创建**（`lib/plausible/session/cache_store.ex:125-161`）：

新会话初始化时设置以下核心字段：
| 字段 | 初始值 | 说明 |
|------|--------|------|
| `sign` | 1 | ClickHouse MergeTree 签名（1表示新增） |
| `session_id` | 随机UInt64 | 唯一会话标识 |
| `user_id` | event.user_id | 用户标识（使用当前盐值生成） |
| `start` | 事件时间 | 会话开始时间 |
| `timestamp` | 事件时间 | 最后活动时间 |
| `duration` | 0 | 会话时长（秒） |
| `pageviews` | 1（pageview事件）或0 | 页面浏览数 |
| `events` | 1 | 事件计数 |
| `is_bounce` | true（pageview）或 !interactive? | 是否为跳出 |

**会话更新逻辑**（`lib/plausible/session/cache_store.ex:95-123`）：

会话更新时的关键计算：
1. **时间戳更新**：`timestamp` 设置为当前事件时间
2. **事件计数**：`events + 1`
3. **页面浏览**：如果是 `pageview` 事件：
   - `pageviews + 1`
   - 更新 `exit_page` 和 `exit_page_hostname`
   - 如果 `entry_page` 为空，设置为当前页面
4. **跳出判定**：
   ```elixir
   is_bounce = if(session.is_bounce,
     do: not (pageviews >= 2 or (event.interactive? and not pageview?)),
     else: session.is_bounce
   )
   ```
   - 初始为 `true`
   - 当页面浏览数 ≥ 2 时，取消跳出标记
   - 当存在交互式非页面浏览事件时，取消跳出标记
5. **会话时长**：
   ```elixir
   duration: NaiveDateTime.diff(event.timestamp, session.start) |> abs
   ```
   - 从会话开始到当前事件的秒数

### 1.5 会话持久化机制

**内存缓存 + 写入缓冲区**（`lib/plausible/session/cache_store.ex:55-65`）：

会话采用**版本化更新**模式：
- 当会话更新时，写入两条记录：
  1. 旧版本记录，`sign = -1`（表示删除/取消）
  2. 新版本记录，`sign = 1`（表示新增）
- ClickHouse 的 CollapsingMergeTree 引擎会在后台合并这些版本

```elixir
defp handle_event(event, found_session, session_attributes, buffer_insert) do
  if found_session do
    updated_session = update_session(found_session, event)
    buffer_insert.([%{found_session | sign: -1}, %{updated_session | sign: 1}])
    update_session_cache(updated_session)
  else
    new_session = new_session_from_event(event, session_attributes)
    buffer_insert.([new_session])
    update_session_cache(new_session)
  end
end
```

**Engagement事件特殊处理**（`lib/plausible/session/cache_store.ex:44-53`）：

- `engagement` 事件是用于延长会话活性的心跳事件
- 不会创建新会话
- 仅刷新缓存中的会话时间戳
- 如果找不到会话，返回 `:no_session_for_engagement`

---

## 2. 跨时间段统计方式

### 2.1 时区处理机制

**UTC存储 + 时区转换**（`lib/plausible/stats/query_builder.ex:165-169`）：

时间范围处理流程：
1. **查询构建**：将站点时区的时间范围转换为 UTC
2. **数据存储**：所有时间戳以 UTC 存储在 ClickHouse
3. **结果展示**：将 UTC 结果转换回站点时区

```elixir
utc_time_range =
  input_date_range
  |> build_datetime_range(site, relative_date, now)
  |> DateTimeRange.to_timezone("Etc/UTC")
```

**时间窗口起止边界定义**（`lib/plausible/stats/datetime_range.ex:23-58`）：

| 输入类型 | 起始边界 | 结束边界 | 时区归属 |
|---------|---------|---------|----------|
| `Date` 范围 | `00:00:00 | `23:59:59` | 站点时区 |
| `DateTime` 范围 | 原值（秒截断） | 原值（秒截断） | 输入时区 |

```elixir
def new!(%Date{} = first, last, timezone) do
  first =
    case DateTime.new(first, ~T[00:00:00], timezone) do
      {:ok, datetime} -> datetime
      {:gap, _just_before, just_after} -> just_after
      {:ambiguous, _first_datetime, second_datetime} -> second_datetime
    end
  ...
end

def new!(%Date{} = last, timezone) do
  last =
    case DateTime.new(last, ~T[23:59:59], timezone) do
      {:ok, datetime} -> datetime
      {:gap, just_before, _just_after} -> just_before
      {:ambiguous, first_datetime, _second_datetime} -> first_datetime
    end
  ...
end
```

**UTC边界计算**（`lib/plausible/stats/time.ex:8-28`）：

```elixir
def utc_boundaries(%Query{
  utc_time_range: time_range,
  site_native_stats_start_at: native_stats_start_at
}) do
  first =
    time_range.first
    |> DateTime.to_naive()
    |> beginning_of_time(native_stats_start_at)

  last = DateTime.to_naive(time_range.last)
  {first, last}
end
```

### 2.2 自动时间分桶口径

**自动粒度选择规则**（`lib/plausible/stats/query_optimizer.ex:95-102`）：

系统根据查询时间范围自动选择时间分桶粒度：

| 时间范围 | 自动选择维度 | 分桶规则 |
|---------|-------------|----------|
| ≤ 48 小时 | `time:hour` | 按小时截断（`toStartOfHour`） |
| ≤ 40 天 | `time:day` | 按天截断（`toDate`） |
| ≤ 52 周 | `time:week` | 按周截断（周一为起始） |
| > 52 周 | `time:month` | 按月截断（月初为起始） |

```elixir
defp resolve_time_dimension(first, last) do
  cond do
    DateTime.diff(last, first, :hour) <= 48 -> "time:hour"
    DateTime.diff(last, first, :day) <= 40 -> "time:day"
    Plausible.Times.diff(last, first, :week) <= 52 -> "time:week"
    true -> "time:month"
  end
end
```

**各时间维度的分桶实现**（`lib/plausible/stats/sql/expression.ex:52-162`）：

| 时间维度 | SQL 分桶函数 | 时区处理 | 说明 |
|---------|-------------|---------|------|
| `time:minute` | `toStartOfMinute(toTimeZone(timestamp, timezone))` | 站点时区 | 实时监控粒度 |
| `time:hour` | `toStartOfHour(toTimeZone(timestamp, timezone))` | 站点时区 | 日内分析，支持涂抹 |
| `time:day` | `toDate(toTimeZone(timestamp, timezone))` | 站点时区 | 日常报告 |
| `time:week` | `weekstart_not_before(..., date_range.first)` | 站点时区 | 周起始对齐查询起始日期 |
| `time:month` | `toStartOfMonth(toTimeZone(timestamp, timezone))` | 站点时区 | 月初对齐 |

**周起始对齐逻辑**：

周维度有特殊的对齐逻辑，确保周桶与查询起始日期对齐：

```elixir
def select_dimension(q, key, "time:week", _table, query) do
  date_range = Query.date_range(query)

  select_merge_as(q, [t], %{
    key =>
      weekstart_not_before(
        to_timezone(t.timestamp, ^query.timezone),
        ^date_range.first
      )
  })
end
```

### 2.3 会话时间涂抹（Smearing）机制

**问题背景**：
- 一个会话可能跨越多个时间边界（如从 14:55 到 15:10）
- 简单按 `session.start` 或 `session.timestamp` 分组会导致统计偏差

**涂抹触发条件**（`lib/plausible/stats/table_decider.ex:127-137`）：

```elixir
@smearable_metrics [:visitors, :visits]
defp smear_session_metrics({:sessions, metrics} = value, query) do
  if ("time:minute" in query.dimensions or "time:hour" in query.dimensions) and
       not filtering_on_dimension?(query, "event:goal") do
    {smearable_metrics, session_metrics} = Enum.split_with(metrics, &(&1 in @smearable_metrics))
    
    [
      {:sessions, session_metrics},
      {:sessions_smeared, smearable_metrics}
    ]
  else
    [value]
  end
end
```

**涂抹触发条件**：
- 使用 `time:minute` 或 `time:hour` 维度
- 未过滤 `event:goal`
- 仅针对 `visitors` 和 `visits` 指标

**SQL实现**（`lib/plausible/stats/sql/expression.ex:117-133`）：

使用 ClickHouse 的 `timeSlots` 函数将会话时间切分为槽位：

```elixir
defmacrop time_slots(query, period_in_seconds, first, last) do
  quote do
    fragment(
      """
      timeSlots(
        toTimeZone(greatest(?, ?), ?),
        toUInt32(timeDiff(greatest(?, ?), least(?, ?))),
        toUInt32(?)
      )
      """,
      s.start,
      ^unquote(first),
      ^unquote(query).timezone,
      s.start,
      ^unquote(first),
      s.timestamp,
      ^unquote(last),
      ^unquote(period_in_seconds)
    )
  end
end
```

**涂抹边界处理**：

| 参数 | 计算方式 | 说明 |
|------|---------|------|
| 开始时间 | `greatest(session.start, query.first)` | 取会话开始和查询开始的较大值 |
| 结束时间 | `least(session.timestamp, query.last)` | 取会话结束和查询结束的较小值 |
| 槽位间隔 | 15分钟（hour）或 60秒（minute） | 支持非整小时时区 |

**时区注意事项**：
- ClickHouse 的 `timeSlots` 基于 Unix 时间戳，**无时区感知**
- 对于非整小时偏移的时区（如 Asia/Katmandu, GMT+5:45），使用15分钟槽位
- 后续通过 `toStartOfHour` 合并为小时级统计

### 2.4 部分时间桶处理

**实时数据的部分桶标记**（`lib/plausible/stats/time.ex:123-153`）：

当查询包含当前时间点时，最后一个时间桶可能是不完整的：
- 识别第一个桶是否被范围起始截断
- 识别最后一个桶是否被当前时间截断
- 返回这些部分桶的标签列表

```elixir
def partial_time_labels(time_labels, query) do
  time_dimension = time_dimension(query)
  range_start = to_naive_in_tz!(query.utc_time_range.first, query.timezone)
  range_end = to_naive_in_tz!(query.utc_time_range.last, query.timezone)
  now = to_naive_in_tz!(query.now, query.timezone)
  
  cutoff = if NaiveDateTime.before?(now, range_end), do: now, else: range_end
  
  # 检查首尾桶是否完整
  first_partial? = ...
  last_partial? = ...
  
  [if(first_partial?, do: first_bucket), if(last_partial?, do: last_bucket)]
  |> Enum.uniq()
  |> Enum.reject(&is_nil/1)
end
```

### 2.5 时间标签生成

**各维度的时间标签**（`lib/plausible/stats/time.ex:48-121`）：

| 维度 | 标签格式 | 示例 | 时区归属 |
|------|---------|------|----------|
| `time:minute` | `YYYY-MM-DD HH:MM:00` | `2024-01-15 14:30:00` | 站点时区 |
| `time:hour` | `YYYY-MM-DD HH:00:00` | `2024-01-15 14:00:00` | 站点时区 |
| `time:day` | `YYYY-MM-DD` | `2024-01-15` | 站点时区 |
| `time:week` | `YYYY-MM-DD`（周一） | `2024-01-15` | 站点时区，对齐查询起始 |
| `time:month` | `YYYY-MM-DD`（月初） | `2024-01-01` | 站点时区 |

---

## 3. 访客识别与新老区分

### 3.1 访客识别核心机制

**确定性哈希生成**（`lib/plausible/ingestion/event.ex:553-567`）：

访客识别采用**无Cookie、隐私优先**的设计：

```elixir
defp generate_user_id(request, domain, hostname, salt) do
  cond do
    is_nil(salt) -> nil
    is_nil(domain) -> nil
    true ->
      user_agent = request.user_agent || ""
      root_domain = get_root_domain(hostname)
      SipHash.hash!(salt, user_agent <> request.remote_ip <> domain <> root_domain)
  end
end
```

**输入因子分析**：

| 因子 | 说明 | 稳定性 |
|------|------|--------|
| `user_agent` | 浏览器用户代理字符串 | 中等（浏览器更新可能变化） |
| `remote_ip` | 客户端IP地址 | 低（动态IP、VPN切换） |
| `domain` | 请求域名 | 高 |
| `root_domain` | 根域名 | 高 |
| `salt` | 每日轮换盐值 | **每日变化** |

### 3.2 盐值管理机制

**双盐值设计**（`lib/plausible/session/salts.ex`）：

系统维护两个盐值以确保平滑过渡：

```elixir
def refresh(name, now) do
  salts = Repo.all(from s in "salts", select: s.salt, order_by: [desc: s.id], limit: 2)

  state =
    case salts do
      [current, prev] ->
        %{previous: prev, current: current}

      [current] ->
        %{previous: nil, current: current}

      [] ->
        new = generate_and_persist_new_salt(now)
        %{previous: nil, current: new}
    end
  ...
end

defp clean_old_salts(now) do
  h48_ago = DateTime.shift(now, hour: -48)
  Repo.delete_all(from s in "salts", where: s.inserted_at < ^h48_ago)
end
```

**盐值生命周期**：

| 状态 | 持续时间 | 说明 |
|------|---------|------|
| `current` | 约1天 | 用于生成新事件的 `user_id` |
| `previous` | 约1天 | 用于盐值切换时的会话查找 |
| 清理 | 超过48小时 | 从数据库删除 |

### 3.3 跨盐值会话衔接机制

**双盐值会话查找**（`lib/plausible/session/cache_store.ex:24-25`）：

这是理解跨盐值会话衔接的核心逻辑：

```elixir
found_session =
  find_session(event, event.user_id) || find_session(event, prev_user_id)
```

**事件处理中的盐值生成**（`lib/plausible/ingestion/event.ex:414-425`）：

```elixir
defp register_session(%__MODULE__{} = event, context) do
  ...
  previous_user_id =
    generate_user_id(
      event.request,
      event.domain,
      event.clickhouse_event.hostname,
      event.salts.previous  # 使用前一个盐值
    )

  case Plausible.Ingestion.Persistor.persist_event(event, previous_user_id, persistor_opts) do
    ...
  end
end
```

**跨盐值会话衔接的实际行为**：

假设盐值从 S1 轮换到 S2，同一访客持续活动：

| 时间点 | 盐值状态 | 事件 `user_id` | 会话 `user_id` | 行为 |
|--------|----------|----------------|----------------|------|
| T1（S1期间） | `current=S1`, `previous=nil` | U1（S1生成） | U1（S1生成） | 创建新会话，存储 U1 |
| T2（盐值刚轮换后） | `current=S2`, `previous=S1` | U2（S2生成） | U1（保持不变） | 先找 U2 失败，再找 U1 成功，会话延续，`session.user_id 保持 U1 |
| T3（S2期间） | `current=S2`, `previous=S1` | U2（S2生成） | U1（保持不变） | 同一会话延续，事件用 U2，会话用 U1 |

**关键发现：事件表与会话表的 `user_id` 不一致**：

当会话跨盐值延续时：

| 数据表 | `user_id` 来源 | 存储值 |
|---------|----------------|--------|
| `events_v2` | 每次事件生成时的 `current` 盐值 | U2（S2生成） |
| `sessions_v2` | 会话**创建时**的盐值 | U1（S1生成） |

**这导致 `visitors` 指标在不同查询路径下可能不一致！**

### 3.4 新老访客区分的实际状态

**重要发现：v2 模型无显式新访客标记**

通过代码分析，当前 `sessions_v2` 和 `events_v2` 表**没有存储**新老访客标识字段：

1. **历史迁移痕迹**（`priv/repo/migrations/20181214201821_add_new_visitor_to_pageviews.exs`）：
   - 早期版本的 `pageviews` 表曾有 `new_visitor` 字段
   - 但在 v2 数据模型中已不再使用

2. **当前Schema**（`lib/plausible/clickhouse_session_v2.ex` 和 `lib/plausible/clickhouse_event_v2.ex`）：
   - 无 `new_visitor` 或 `visitor_type` 字段
   - 无 `first_visit_timestamp` 或类似字段

**基于现有数据的新老访客判定方式**：

如需区分新老访客，需要**动态计算**：

```elixir
# 伪代码：基于历史存在性判定
def is_new_visitor(site_id, user_id, event_timestamp) do
  # 查询该 user_id 在 event_timestamp 之前是否有任何活动
  has_history = ClickhouseRepo.exists?(
    from e in "events_v2",
      where: e.site_id == ^site_id,
      where: e.user_id == ^user_id,
      where: e.timestamp < ^event_timestamp,
      limit: 1
  )
  not has_history
end
```

**但存在重大限制**：

| 限制因素 | 影响 |
|---------|------|
| **盐值每日轮换** | 同一访客跨天会有不同 `user_id`，无法追踪长期历史 |
| **跨盐值会话 `user_id` 不一致** | 事件表和会话表的 `user_id` 可能不同，影响判定 |
| **无持久化存储** | 每次查询需要全表扫描历史数据，性能极低 |
| **IP/UA变化** | 网络环境变化导致 `user_id` 变化，误判为新访客 |

### 3.5 访客统计的实际口径

**当前实现的访客统计**（`lib/plausible/stats/sql/expression.ex:302-312`）：

```elixir
# 事件表访客计数
def event_metric(:visitors, _query) do
  wrap_alias([e], %{
    visitors: scale_sample(fragment("uniq(?)", e.user_id))
  })
end

# 事件表会话计数
def event_metric(:visits, _query) do
  wrap_alias([e], %{
    visits: scale_sample(fragment("uniq(?)", e.session_id))
  })
end
```

**会话表访客计数**（`lib/plausible/stats/sql/expression.ex:462-466`）：

```elixir
def session_metric(:visitors, _query) do
  wrap_alias([s], %{
    visitors: scale_sample(fragment("uniq(?)", s.user_id))
  })
end
```

**口径定义**：

| 指标 | 计算方式 | 时间范围 |
|------|---------|---------|
| `visitors` | `uniq(user_id)` | 查询时间窗口内的去重用户 |
| `visits` | `uniq(session_id)` 或 `sum(sign)` | 查询时间窗口内的去重会话 |

**关键理解**：
- 这些指标**仅统计查询时间窗口内的活动**
- **不区分**新访客和老访客
- `uniq(user_id)` 只是去重计数，不是"独立访客"在传统意义上的新老区分

### 3.6 实时访客统计

**当前访客计算**（`lib/plausible/stats/current_visitors.ex:6-19`）：

```elixir
def current_visitors(site, duration \\ Duration.new!(minute: -5)) do
  first_datetime =
    NaiveDateTime.utc_now()
    |> NaiveDateTime.shift(duration)
    |> NaiveDateTime.truncate(:second)

  ClickhouseRepo.one(
    from e in "events_v2",
      where: ^Plausible.Sites.site_id_query_filter(site),
      where: e.timestamp >= ^first_datetime,
      where: e.name != "engagement",
      select: uniq(e.user_id)
  )
end
```

**口径特点**：
- 默认时间窗口：**过去5分钟**
- 排除 `engagement` 事件（仅统计实质活动）
- 使用 `uniq(user_id)` 去重计数

---

## 4. 留存指标计算逻辑

### 4.1 代码库中留存功能的状态

**重要发现：无内置留存指标计算**

通过全面代码分析，当前实现**未包含**传统意义上的留存指标：

1. **代码搜索结果**：
   - 搜索 `retention` 仅返回与数据保留策略（data retention）相关的代码
   - 搜索 `cohort` 无相关结果
   - 搜索 `new_visitor` / `returning` 无业务逻辑实现

2. **可用指标列表**（`lib/plausible/stats/metrics.ex:12-26`）：

```elixir
@all_metrics [
  :visitors,
  :visits,
  :pageviews,
  :exit_rate,
  :views_per_visit,
  :bounce_rate,
  :visit_duration,
  :events,
  :conversion_rate,
  :group_conversion_rate,
  :time_on_page,
  :percentage,
  :scroll_depth
] ++ @revenue_metrics
```

**无以下指标**：
- `new_visitors` / `returning_visitors`
- `retention_rate`
- `cohort_*` 类指标

### 4.2 留存窗口定义（理论框架）

**留存分析的核心概念**：

| 概念 | 定义 | 时区归属 |
|------|------|----------|
| **基准日（Cohort Date） | 用户首次访问的日期 | 站点时区 |
| **N日窗口** | 从基准日开始的N天内 | 站点时区 |
| **回访判定** | 用户在N日内是否有活动 | 站点时区 |

**时间窗口边界（理论）：

```
基准日（Day 0）：
  起始：`toDate(first_visit_timestamp) in 站点时区
  结束：同日期 23:59:59 站点站点 站点 站点站点 站点

N日留存窗口（Day N）：
  起始：基准日 + N天 00:00:00 站点时区
  结束：基准日 + N天 23:59:59 站点站点 站点站点 站点站点 站点站点

示例：3日留存率
  基准日：2024-01-01 00:00:00 到 2024-01-01 23:59:59 站点时区
  回访窗口：2024-01-04 00:00:00 到 2024-01-04 23:59:59 站点站点站点站点站点站点
```

### 4.3 基于现有数据结构的潜在留存计算

如需实现留存分析，可基于现有数据进行**二次计算**。以下是理论上的实现方式：

#### 4.3.1 同期群（Cohort）分析框架

**同期群定义**：按首次访问时间分组

```
同期群 = {site_id, first_visit_date}
```

**但存在的问题**：
- 无 `first_visit_date` 字段
- `user_id` 每日变化，无法长期追踪
- 跨盐值会话 `user_id` 不一致

#### 4.3.2 单日新老访客区分（理论实现）

```elixir
# 伪代码：单日新老访客统计
def daily_new_returning_visitors(site_id, date) do
  # 1. 获取当天所有 user_id
  today_user_ids = ClickhouseRepo.all(
    from e in "events_v2",
      where: e.site_id == ^site_id,
      where: fragment("toDate(?)", e.timestamp) == ^date,
      select: distinct(e.user_id)
  )
  
  # 2. 检查每个 user_id 在此日期前是否有活动
  # 注意：这是 N+1 查询，性能极差
  {new_count, returning_count} =
    today_user_ids
    |> Enum.split_with(fn user_id ->
      not ClickhouseRepo.exists?(
        from e in "events_v2",
          where: e.site_id == ^site_id,
          where: e.user_id == ^user_id,
          where: fragment("toDate(?)", e.timestamp) < ^date,
          limit: 1
      )
    end)
  
  %{new: length(new_count), returning: length(returning_count)}
end
```

#### 4.3.3 留存率计算（理论实现）

```elixir
# 伪代码：N日留存率
# 问题：由于 user_id 每日变化，这实际上无法准确计算
def retention_rate(site_id, cohort_date, days_later) do
  # 基准日访客
  baseline_visitors = MapSet.new(
    ClickhouseRepo.all(
      from e in "events_v2",
        where: e.site_id == ^site_id,
        where: fragment("toDate(?)", e.timestamp) == ^cohort_date,
        select: distinct(e.user_id)
    )
  )
  
  # N日后访客
  later_date = Date.add(cohort_date, days_later)
  later_visitors = MapSet.new(
    ClickhouseRepo.all(
      from e in "events_v2",
        where: e.site_id == ^site_id,
        where: fragment("toDate(?)", e.timestamp) == ^later_date,
        select: distinct(e.user_id)
    )
  )
  
  # 交集（实际上由于盐值轮换，交集很可能为空）
  retained = MapSet.intersection(baseline_visitors, later_visitors)
  
  if MapSet.size(baseline_visitors) > 0 do
    MapSet.size(retained) / MapSet.size(baseline_visitors)
  else
    0.0
  end
end
```

### 4.4 盐值轮换对留存分析的根本性影响

**核心问题说明**：

```
场景：同一访客连续3天访问站点，且会话持续活动（30分钟内）

第1天（盐值 S1，创建会话）：
  事件 user_id = SipHash(S1, UA + IP + ...) = U1
  会话 user_id = U1（创建时 S1）
  活动记录：events_v2 存储 U1，sessions_v2 存储 U1

盐值轮换 S1 → S2，会话通过双盐值查找延续

第2天（盐值 S2，会话延续）：
  事件 user_id = SipHash(S2, UA + IP + ...) = U2（≠ U1）
  会话 user_id = U1（保持不变）
  活动记录：events_v2 存储 U2，sessions_v2 存储 U1
  ← 两个表的 user_id 不一致！

第3天（盐值 S3）：
  事件 user_id = SipHash(S3, UA + IP + ...) = U3（≠ U1, ≠ U2）
  会话 user_id = 取决于是否跨盐值延续情况
  活动记录：events_v2 存储 U3，sessions_v2 存储 会话创建时的盐值

留存分析（查询第1天访客在第3天是否回访）：
  第1天 events_v2 user_id 集合：{U1}
  第1天 sessions_v2 user_id 集合：{U1}
  第3天 events_v2 user_id 集合：{U3}
  第3天 sessions_v2 user_id 集合：取决于会话是否延续
  交集：{}（空集）或部分交集
  结果：0% 留存率 ← 严重低估！
```

**实际影响矩阵**：

| 分析类型 | 可行性 | 准确性 | 说明 |
|---------|--------|--------|------|
| 单日访客统计（同盐值） | ✅ 完全支持 | ✅ 准确 | `uniq(user_id)` 单日内稳定 |
| 日内会话分析 | ✅ 完全支持 | ✅ 准确 | 30分钟超时规则 |
| 盐值切换日访客统计 | ⚠️ 有条件支持 | ⚠️ 可能不一致 | 跨盐值会话两表 user_id 不一致 |
| 跨天新老区分 | ⚠️ 需二次计算 | ❌ 不准确 | 盐值轮换导致 `user_id` 变化 |
| N日留存率 | ❌ 无法准确计算 | ❌ 无意义 | 跨天 `user_id` 无关联性 |
| 同期群分析 | ❌ 无法准确计算 | ❌ 无意义 | 无法追踪长期访客 |

### 4.5 访客统计的有效口径

**当前系统可准确计算的指标**：

| 指标 | 计算方式 | 时间范围 | 准确性 |
|------|---------|---------|--------|
| **实时访客** | 过去N分钟 `uniq(user_id)` | 分钟级 | ✅ 准确 |
| **日访客（events_v2）** | 当天 `uniq(user_id)` 事件表 | 自然日 | ✅ 准确 |
| **日访客（sessions_v2）** | 当天 `uniq(user_id)` 会话表 | 自然日 | ⚠️ 盐值切换日可能不一致 |
| **日会话** | 当天 `uniq(session_id)` 或 `sum(sign)` | 自然日 | ✅ 准确 |
| **页面浏览** | `countIf(name='pageview')` | 任意 | ✅ 准确 |
| **跳出率** | `sum(is_bounce * sign) / sum(sign)` | 任意 | ✅ 准确 |
| **平均会话时长** | `sum(duration * sign) / sum(sign)` | 任意 | ✅ 准确 |

**需要谨慎使用的指标**：

| 指标 | 问题 | 建议 |
|------|------|------|
| 盐值切换日访客趋势 | 两表 `user_id` 可能不一致 | 理解数据局限性，以事件表为准 |
| 跨天访客趋势 | 盐值轮换可能导致波动 | 仅作趋势参考，不作精确对比 |
| 周/月访客统计 | 同上 | 理解数据局限性 |
| 任何形式的"新老访客" | 无持久化标记 | 不建议使用，或需自定义实现 |

### 4.6 设计初衷与权衡

**隐私优先设计**：

盐值每日轮换是**隐私保护的刻意设计**：
- 防止长期追踪用户（符合 GDPR、CCPA 等法规）
- `user_id` 只是临时标识符，不是永久用户标识
- 系统设计目标是**聚合统计**，而非个体追踪

**数据模型设计原则**：

```
核心目标：
├── 实时性：快速处理大量事件
├── 聚合性：高效计算聚合指标（访客数、会话数等）
├── 隐私性：避免长期用户追踪
└── 简单性：减少状态管理复杂度

非核心目标（当前版本）：
├── 长期用户追踪
├── 新老访客精确区分
├── 留存率/同期群分析
└── 用户级别的漏斗分析
```

---

## 5. 不同维度下的口径一致性

### 5.1 维度分类体系

**事件维度 vs 会话维度**（`lib/plausible/stats/table_decider.ex:179-187`）：

| 维度前缀 | 数据来源 | 说明 |
|---------|---------|------|
| `event:*` | `events_v2` 表 | 基于单个事件的属性 |
| `visit:*` | `sessions_v2` 表 | 基于会话聚合的属性 |

**维度分类规则**：
```elixir
defp dimension_partitioner(_, "event:" <> _), do: :event
defp dimension_partitioner(_, "visit:entry_page"), do: :session
defp dimension_partitioner(_, "visit:entry_page_hostname"), do: :session
defp dimension_partitioner(_, "visit:exit_page"), do: :session
defp dimension_partitioner(_, "visit:exit_page_hostname"), do: :session
defp dimension_partitioner(_, "visit:" <> _), do: :either
defp dimension_partitioner(_, _), do: :either
```

**特殊说明**：
- `visit:entry_page`、`visit:exit_page` 等**必须**从会话表查询
- 其他 `visit:*` 维度可从任一表获取（存在时）

### 5.2 指标表归属规则

**指标分类**（`lib/plausible/stats/table_decider.ex:146-177`）：

| 指标 | 归属表 | 计算方式 |
|------|--------|---------|
| `visitors` | `:either` | 去重计数 `uniq(user_id)` |
| `visits` | `:either` | 去重计数 `uniq(session_id)` 或 `sum(sign)` |
| `pageviews` | `:event` | 条件计数 `countIf(name='pageview')` |
| `events` | `:event` | 条件计数 `countIf(name!='engagement')` |
| `bounce_rate` | `:session` | 聚合计算 `sum(is_bounce * sign) / sum(sign)` |
| `visit_duration` | `:session` | 平均计算 `sum(duration * sign) / sum(sign)` |
| `views_per_visit` | `:session` | 平均计算 `sum(sign * pageviews) / sum(sign)` |
| `exit_rate` | `:session` | 基于退出页的特殊计算 |
| `time_on_page` | `:event` | 基于 `engagement_time` 计算 |
| `scroll_depth` | `:event` | 滚动深度指标 |
| `conversion_rate` | `:either` | 转化率（目标相关） |
| `group_conversion_rate` | `:either` | 组转化率 |

**特殊规则 - 时间维度优先选择**：
```elixir
defp metric_partitioner(query, metric) when metric in [:visitors, :visits] do
  if "time:minute" in query.dimensions and not filtering_on_dimension?(query, "event:goal") do
    :session  # 优先使用会话表，支持涂抹
  else
    :either
  end
end
```

### 5.3 visitors 指标的多统计路径分析

**`visitors` 指标的三种统计路径**：

| 统计路径 | 触发条件 | 计算方式 | `user_id` 来源 |
|---------|----------|-------------|
| **路径A：事件表 | 有 `event:*` 维度/过滤 | `uniq(e.user_id)` | 每次事件的 `current` 盐值 |
| **路径B：会话表** | 有 `visit:*` 维度/过滤 | `uniq(s.user_id)` | 会话**创建时**的盐值 |
| **路径C：会话表涂抹** | `time:minute`/`time:hour` 维度 | `uniq(s.user_id)` 跨时间槽 | 会话创建时的盐值 |

**三种路径的详细对比**：

#### 路径A：事件表统计

**触发场景**：
- 使用 `event:page`、`event:hostname`、`event:props:*` 等维度
- 过滤条件涉及 `event:*` 维度

**SQL实现**（`lib/plausible/stats/sql/expression.ex:302-306`）：
```elixir
def event_metric(:visitors, _query) do
  wrap_alias([e], %{
    visitors: scale_sample(fragment("uniq(?)", e.user_id))
  })
end
```

**`user_id` 来源**：
- 每个事件写入时的 `current` 盐值生成
- 跨盐值会话的事件会使用不同盐值

#### 路径B：会话表统计

**触发场景**：
- 使用 `visit:entry_page`、`visit:exit_page` 等必须从会话表查询的维度
- 仅会话指标（如 `bounce_rate`）

**SQL实现**（`lib/plausible/stats/sql/expression.ex:462-466`）：
```elixir
def session_metric(:visitors, _query) do
  wrap_alias([s], %{
    visitors: scale_sample(fragment("uniq(?)", s.user_id))
  })
end
```

**`user_id` 来源**：
- 会话**创建时**的盐值
- 跨盐值延续的会话保持创建时的 `user_id`

#### 路径C：会话表涂抹统计

**触发场景**：
- 使用 `time:minute` 或 `time:hour` 维度
- 未过滤 `event:goal`

**特殊处理**：
- 使用 `timeSlots` 函数展开会话时间
- 每个时间槽位都计入 `uniq`
- 但 `user_id` 仍是会话创建时的值

**三种路径的一致性分析

**同一天内（无盐值轮换）**：

| 场景 | 路径A（事件表） | 路径B（会话表） | 一致性 |
|------|---------------|-----------------|--------|
| 同一访客同一盐值） | U1（U1（U1 | U1 | ✅ 一致 |
| 同一会话 | U1 | U1 | ✅ 一致 |

**盐值切换日（有跨盐值会话）**：

| 场景 | 路径A（事件表） | 路径B（会话表） | 一致性 |
|------|----------------|-----------------|--------|
| 会话创建于S1，事件在S2 | U2（S2生成） | U1（S1生成） | ❌ **不一致** |
| 新会话创建于S2 | U2（S2生成） | U2（S2生成） | ✅ 一致 |

**跨天对比（盐值轮换）**：

| 场景 | 路径A（事件表） | 路径B（会话表） | 说明 |
|------|----------------|-----------------|------|
| 同一访客跨天 | 不同 `user_id` | 不同 `user_id` | ✅ 一致（都无法追踪） |

### 5.4 跨维度查询兼容性

**兼容性验证**（`lib/plausible/stats/table_decider.ex:44-77`）：

查询构建时验证指标与维度的兼容性：

```elixir
def validate_no_metrics_dimensions_conflict(query) do
  %{event: event_only_metrics, session: session_only_metrics} =
    partition(query.metrics, query, &metric_partitioner/2)
  
  %{event: event_only_dimensions, session: session_only_dimensions} =
    partition(query.dimensions, query, &dimension_partitioner/2)
  
  cond do
    # event:page 是特殊情况，由 QueryOptimizer 处理
    event_only_dimensions == ["event:page"] -> :ok
    
    # 会话指标 + 事件维度 = 不兼容
    not empty?(session_only_metrics) and not empty?(event_only_dimensions) ->
      {:error, %QueryError{message: "Session metric(s) #{i(session_only_metrics)} cannot be queried along with event dimension(s) #{i(event_only_dimensions)}"}}
    
    # 事件指标 + 会话维度 = 不兼容（除 revenue）
    not empty?(conflicting_event_metrics) and not empty?(session_only_dimensions) ->
      {:error, %QueryError{message: "Event metric(s) #{i(conflicting_event_metrics)} cannot be queried along with session dimension(s) #{i(session_only_dimensions)}"}}
    
    true -> :ok
  end
end
```

**不兼容组合示例**：
| 指标类型 | 维度类型 | 结果 |
|---------|---------|------|
| 会话指标（`bounce_rate`） | 事件维度（`event:page`） | ❌ 错误 |
| 事件指标（`scroll_depth`） | 会话维度（`visit:country`） | ❌ 错误 |
| 通用指标（`visitors`） | 任一维度 | ✅ 允许 |

### 5.5 自动维度过滤

**空值过滤规则**（`lib/plausible/stats/breakdown.ex:141-160`）：

在 Breakdown 查询中，某些维度自动排除空值：

```elixir
@filter_dimensions_not %{
  "visit:city" => [0],
  "visit:country" => ["\0\0", "ZZ"],
  "visit:region" => [""],
  "visit:utm_medium" => [""],
  "visit:utm_source" => [""],
  "visit:utm_campaign" => [""],
  "visit:utm_content" => [""],
  "visit:utm_term" => [""],
  "visit:entry_page" => [""],
  "visit:exit_page" => [""]
}

defp dimension_filters(dimension) when dimension in @extra_filter_dimensions do
  [[:is_not, dimension, Map.get(@filter_dimensions_not, dimension)]]
end
```

**过滤目的**：
- 确保维度值有意义，排除"未设置"状态
- 提高跨维度报告的可比性
- 避免空值主导统计结果

### 5.6 表选择决策逻辑

**查询分区策略**（`lib/plausible/stats/table_decider.ex:87-121`）：

根据指标、过滤器、维度的组合，决定查询哪些表：

```elixir
def partition_metrics(requested_metrics, query) do
  metrics = partition(requested_metrics, query, &metric_partitioner/2)
  filters = query.filters |> dimensions_used_in_filters() |> partition(query, &dimension_partitioner/2)
  dimensions = partition(query.dimensions, query, &dimension_partitioner/2)
  
  cond do
    # 仅会话表
    empty?(metrics.event) && empty?(filters.event) && empty?(dimensions.event) ->
      [sessions: metrics.session ++ metrics.either ++ metrics.sample_percent]
    
    # 仅事件表
    empty?(metrics.session) && empty?(filters.session) && empty?(dimensions.session) ->
      [events: metrics.event ++ metrics.either ++ metrics.sample_percent]
    
    # 混合场景
    true ->
      [
        events: metrics.event ++ metrics.either ++ metrics.sample_percent,
        sessions: metrics.session ++ metrics.sample_percent
      ]
  end
  |> Enum.flat_map(&smear_session_metrics(&1, query))
  |> Enum.reject(fn {_table_type, metrics} -> empty?(metrics) end)
end
```

**决策流程图**：
```
┌─────────────────────────────────────────────────────────────┐
│                    分析查询组件                               │
│  metrics (指标) + filters (过滤器) + dimensions (维度)      │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │   是否只涉及会话侧？          │
              │  (无 event 指标/过滤/维度)    │
              └───────────────────────────────┘
                    │               │
                   是               否
                    │               │
                    ▼               ▼
            ┌──────────┐    ┌───────────────────────┐
            │ 单表查询  │    │ 是否只涉及事件侧？    │
            │ sessions │    │ (无 session 指标/过滤/维度) │
            └──────────┘    └───────────────────────┘
                                    │           │
                                   是           否
                                    │           │
                                    ▼           ▼
                            ┌──────────┐  ┌─────────────────┐
                            │ 单表查询  │  │   多表查询      │
                            │  events  │  │ events+sessions │
                            └──────────┘  └─────────────────┘
```

### 5.7 指标计算一致性

**核心指标的SQL实现**（`lib/plausible/stats/sql/expression.ex`）：

#### 事件表指标（`event_metric/2`）

```elixir
# 页面浏览数
def event_metric(:pageviews, _query) do
  wrap_alias([e], %{
    pageviews: scale_sample(fragment("countIf(? = 'pageview')", e.name))
  })
end

# 事件数（排除 engagement）
def event_metric(:events, _query) do
  wrap_alias([e], %{
    events: scale_sample(fragment("countIf(? != 'engagement')", e.name))
  })
end

# 访客数（去重）
def event_metric(:visitors, _query) do
  wrap_alias([e], %{
    visitors: scale_sample(fragment("uniq(?)", e.user_id))
  })
end

# 会话数（去重）
def event_metric(:visits, _query) do
  wrap_alias([e], %{
    visits: scale_sample(fragment("uniq(?)", e.session_id))
  })
end
```

#### 会话表指标（`session_metric/2`）

```elixir
# 跳出率（带历史数据兼容处理）
def session_metric(:bounce_rate, query) do
  wrap_alias([], %{
    bounce_rate:
      fragment(
        """
        toUInt32(greatest(ifNotFinite(round(sumIf(is_bounce * sign, ?) / sumIf(sign, ?) * 100), 0), 0))
        """,
        ^condition,
        ^condition
      ),
    __internal_visits: fragment("toUInt32(greatest(sum(sign), 0))")
  })
end

# 平均会话时长
def session_metric(:visit_duration, _query) do
  wrap_alias([], %{
    visit_duration:
      fragment("toUInt32(greatest(ifNotFinite(round(sum(duration * sign) / sum(sign)), 0), 0))"),
    __internal_visits: fragment("toUInt32(greatest(sum(sign), 0))")
  })
end

# 每次访问浏览页数
def session_metric(:views_per_visit, _query) do
  wrap_alias([s], %{
    views_per_visit:
      fragment(
        "greatest(ifNotFinite(round(sum(? * ?) / sum(?), 2), 0), 0)",
        s.sign,
        s.pageviews,
        s.sign
      ),
    __internal_visits: fragment("toUInt32(greatest(sum(sign), 0))")
  })
end
```

**关键设计要点**：
1. **CollapsingMergeTree 支持**：所有聚合使用 `sign` 字段
   - `sum(field * sign)`：正确计算版本化数据
   - `greatest(..., 0)`：防止历史数据导致的负值
2. **采样支持**：使用 `scale_sample` 函数处理采样数据
3. **除零保护**：使用 `ifNotFinite` 处理除零情况
4. **类型转换**：使用 `toUInt32` 确保整数结果

### 5.8 一致性与差异场景汇总

**visitors 指标一致性场景**：

| 场景 | 路径A（事件表） | 路径B（会话表） | 一致性 | 原因 |
|------|----------------|-----------------|--------|------|
| 同一天内，无盐值轮换 | ✅ 准确 | ✅ 准确 | ✅ 一致 | 同一盐值生成 |
| 盐值切换日，新会话 | ✅ 准确 | ✅ 准确 | ✅ 一致 | 新盐值生成 |
| 盐值切换日，跨盐值会话 | U2（新盐值） | U1（旧盐值） | ❌ **不一致** | 事件用新盐值，会话用旧盐值 |
| 跨天对比（不同盐值） | 无法追踪 | 无法追踪 | ✅ 一致（都无意义） | 盐值轮换导致 user_id 变化 |

**不同维度组合下的统计路径选择**：

| 指标 | 维度 | 过滤器 | 统计路径 | 说明 |
|------|------|--------|----------|------|
| `visitors` | `time:day` | 无 | 会话表（路径B） | 无事件维度，选择会话表 |
| `visitors` | `event:page` | 无 | 事件表（路径A） | 事件维度，选择事件表 |
| `visitors` | `time:hour` | 无 | 会话表涂抹（路径C） | 小时维度触发涂抹 |
| `visitors` | `visit:country` | `event:page = '/home' | 双表查询（路径A+B） | 事件过滤+会话维度，分别查询 |

**双表查询的结果合并**：

当查询涉及事件侧和会话侧的组合时：

```
查询示例：
  指标: visitors
  维度: visit:country
  过滤: event:page = '/home'

执行流程：
1. 事件侧过滤（事件侧：`event:page` 过滤器）
   → 事件侧查询：`events_v2`，过滤 `pathname = '/home'`，计算 `uniq(user_id)`

2. 会话侧维度（会话侧：`visit:country` 维度）
   → 会话侧查询：`sessions_v2`，按 `country` 分组，计算 `uniq(user_id)`

3. 结果合并
   → 注意：两表的 `user_id` 在盐值切换日可能不一致！
```

**时间维度的特殊处理**：

| 时间维度 | 涂抹触发 | 统计路径 | 说明 |
|---------|---------|----------|------|
| `time:minute` | ✅ 触发 | 会话表涂抹（路径C） | 15分钟槽位展开 |
| `time:hour` | ✅ 触发 | 会话表涂抹（路径C） | 15分钟槽位后合并小时 |
| `time:day` | ❌ 不触发 | 会话表（路径B） | 按天截断 |
| `time:week` | ❌ 不触发 | 会话表（路径B） | 按周截断对齐查询起始 |
| `time:month` | ❌ 不触发 | 会话表（路径B） | 按月截断 |

---

## 6. 代码参考位置

### 6.1 会话边界相关

| 功能 | 文件路径 | 行号 |
|------|---------|------|
| 会话超时配置 | `config/config.exs` | 57-59 |
| 会话查找逻辑 | `lib/plausible/session/cache_store.ex` | 69-81 |
| 会话更新逻辑 | `lib/plausible/session/cache_store.ex` | 95-123 |
| 新会话创建 | `lib/plausible/session/cache_store.ex` | 125-161 |
| 会话ID生成 | `lib/plausible/clickhouse_session_v2.ex` | 80-82 |
| 用户ID生成 | `lib/plausible/ingestion/event.ex` | 553-567 |
| 双盐值会话查找 | `lib/plausible/session/cache_store.ex` | 24-25 |
| 盐值获取 | `lib/plausible/session/salts.ex` | - |
| 会话平衡器 | `lib/plausible/session/balancer.ex` | - |

### 6.2 访客识别相关

| 功能 | 文件路径 | 行号 |
|------|---------|------|
| 用户ID生成 | `lib/plausible/ingestion/event.ex` | 553-567 |
| 盐值获取 | `lib/plausible/ingestion/event.ex` | 383-396 |
| 双盐值会话查找 | `lib/plausible/session/cache_store.ex` | 24-25 |
| 前一个盐值 user_id 生成 | `lib/plausible/ingestion/event.ex` | 414-425 |
| 盐值管理 | `lib/plausible/session/salts.ex` | - |
| 实时访客统计 | `lib/plausible/stats/current_visitors.ex` | 6-19 |
| 访客指标定义 | `lib/plausible/stats/metrics.ex` | 12-26 |
| 事件表访客计数 | `lib/plausible/stats/sql/expression.ex` | 302-306 |
| 会话表访客计数 | `lib/plausible/stats/sql/expression.ex` | 462-466 |
| 历史迁移（参考） | `priv/repo/migrations/20181214201821_add_new_visitor_to_pageviews.exs` | - |

### 6.3 时间统计相关

| 功能 | 文件路径 | 行号 |
|------|---------|------|
| 时间范围构建 | `lib/plausible/stats/datetime_range.ex` | 23-58 |
| UTC边界计算 | `lib/plausible/stats/time.ex` | 8-28 |
| 自动时间维度选择 | `lib/plausible/stats/query_optimizer.ex` | 95-102 |
| 时间维度检测 | `lib/plausible/stats/time.ex` | 38-43 |
| 时间标签生成 | `lib/plausible/stats/time.ex` | 48-121 |
| 部分桶识别 | `lib/plausible/stats/time.ex` | 123-153 |
| 时间范围构建 | `lib/plausible/stats/query_builder.ex` | 88-150 |
| 时区转换 | `lib/plausible/stats/query_builder.ex` | 165-169 |
| timeSlots宏 | `lib/plausible/stats/sql/expression.ex` | 30-50 |
| 小时维度涂抹 | `lib/plausible/stats/sql/expression.ex` | 117-133 |
| 周维度对齐 | `lib/plausible/stats/sql/expression.ex` | 72-95 |

### 6.4 维度一致性相关

| 功能 | 文件路径 | 行号 |
|------|---------|------|
| 维度分类器 | `lib/plausible/stats/table_decider.ex` | 179-187 |
| 指标分类器 | `lib/plausible/stats/table_decider.ex` | 146-177 |
| 兼容性验证 | `lib/plausible/stats/table_decider.ex` | 44-77 |
| 表分区策略 | `lib/plausible/stats/table_decider.ex` | 87-121 |
| 涂抹机制 | `lib/plausible/stats/table_decider.ex` | 123-141 |
| 自动过滤维度 | `lib/plausible/stats/breakdown.ex` | 141-160 |
| 事件表指标 | `lib/plausible/stats/sql/expression.ex` | 290-411 |
| 会话表指标 | `lib/plausible/stats/sql/expression.ex` | 413-498 |
| 查询优化器 | `lib/plausible/stats/query_optimizer.ex` | - |

### 6.5 数据模型

| 模型 | 文件路径 | 说明 |
|------|---------|------|
| 会话Schema | `lib/plausible/clickhouse_session_v2.ex` | sessions_v2 表结构 |
| 事件Schema | `lib/plausible/clickhouse_event_v2.ex` | events_v2 表结构 |
| 事件处理 | `lib/plausible/ingestion/event.ex` | 事件处理管道 |
| 持久化器 | `lib/plausible/ingestion/persistor.ex` | 会话持久化入口 |
| 写入缓冲区 | `lib/plausible/session/write_buffer.ex` | 批量写入缓冲 |

---

## 附录：关键数据流图

### 附录A：会话生命周期

```
┌──────────────┐     ┌─────────────────┐     ┌────────────────┐
│  新事件到达   │────▶│  查找现有会话   │────▶│  30分钟超时？  │
└──────────────┘     └─────────────────┘     └────────────────┘
                                                  │         │
                                                 是         否
                                                  │         │
                                                  ▼         ▼
                                          ┌──────────┐ ┌──────────┐
                                          │ 创建新会话 │ │ 更新现有会话 │
                                          │ sign = 1 │ │ sign=-1,+1│
                                          └──────────┘ └──────────┘
                                               │              │
                                               ▼              ▼
                                          ┌────────────────────────┐
                                          │  写入 ClickHouse       │
                                          │  CollapsingMergeTree   │
                                          └────────────────────────┘
```

### 附录B：访客识别与盐值轮换

```
┌─────────────────────────────────────────────────────────────────┐
│                        访客识别流程                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────────────┐
              │     生成输入因子                       │
              │  UA + IP + Domain + RootDomain       │
              └───────────────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────────────┐
              │     结合每日盐值                       │
              │  SipHash(Salt, 输入因子) = user_id   │
              └───────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
    ┌───────────────────┐           ┌───────────────────┐
    │    同一天内        │           │    跨天后          │
    │                   │           │                   │
    │  Salt 保持不变    │           │  Salt 轮换(S1→S2) │
    │  user_id 稳定     │           │  user_id 变化     │
    │  可准确去重统计   │           │  无法追踪同一访客 │
    └───────────────────┘           └───────────────────┘
```

### 附录C：跨盐值会话衔接

```
┌─────────────────────────────────────────────────────────────────┐
│                    跨盐值会话衔接流程                               │
└─────────────────────────────────────────────────────────────────┘

时间线：
─────────────────────────────────────────────────────────────────►
        T1                          T2                          T3
        │                           │                           │
        ▼                           ▼                           ▼
┌───────────────┐         ┌─────────────────┐         ┌───────────────┐
│  盐值 S1     │         │  盐值 S2        │         │  盐值 S3        │
│  期间        │         │  期间             │         │  期间             │
└───────────────┘         └─────────────────┘         └───────────────┘
        │                           │                           │
        ▼                           ▼                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  同一访客持续活动（30分钟内无超时）                              │
│                                                                 │
│  T1 (S1期间)：                                                  │
│    事件 user_id = U1 (S1生成)                                    │
│    会话 user_id = U1 (创建时)                                   │
│    events_v2: U1                                               │
│    sessions_v2: U1                                              │
│                                                                 │
│  T2 (S2期间，会话延续)：                                        │
│    事件 user_id = U2 (S2生成)  ← 新盐值                         │
│    会话 user_id = U1 (保持不变)  ← 旧盐值