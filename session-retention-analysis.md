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

### 2.2 时间维度粒度

**支持的时间维度**（`lib/plausible/stats/sql/expression.ex:52-162`）：

| 时间维度 | 粒度 | 适用场景 | 特殊处理 |
|---------|------|---------|---------|
| `time:minute` | 分钟 | 实时监控 | 仅支持≤30小时范围 |
| `time:hour` | 小时 | 日内分析 | 支持涂抹机制 |
| `time:day` | 天 | 日常报告 | 按站点时区截断 |
| `time:week` | 周 | 周度趋势 | 周起始日期对齐 |
| `time:month` | 月 | 月度报告 | 月初对齐 |

**会话时间涂抹（Smearing）机制**（`lib/plausible/stats/table_decider.ex:123-141`）：

**问题背景**：
- 一个会话可能跨越多个时间边界（如从 14:55 到 15:10）
- 简单按 `session.start` 或 `session.timestamp` 分组会导致统计偏差

**解决方案**（`lib/plausible/stats/table_decider.ex:127-137`）：

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

使用 ClickHouse 的 `timeSlots` 函数将会话时间切分为15分钟槽位：
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

**时区注意事项**：
- ClickHouse 的 `timeSlots` 基于 Unix 时间戳，**无时区感知**
- 对于非整小时偏移的时区（如 Asia/Katmandu, GMT+5:45），使用15分钟槽位
- 后续通过 `toStartOfHour` 合并为小时级统计

### 2.3 部分时间桶处理

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

### 2.4 时间标签生成

**各维度的时间标签**（`lib/plausible/stats/time.ex:48-121`）：

| 维度 | 标签格式 | 示例 |
|------|---------|------|
| `time:minute` | `YYYY-MM-DD HH:MM:00` | `2024-01-15 14:30:00` |
| `time:hour` | `YYYY-MM-DD HH:00:00` | `2024-01-15 14:00:00` |
| `time:day` | `YYYY-MM-DD` | `2024-01-15` |
| `time:week` | `YYYY-MM-DD`（周一） | `2024-01-15` |
| `time:month` | `YYYY-MM-DD`（月初） | `2024-01-01` |

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

### 3.2 盐值轮换机制

**双盐值设计**（`lib/plausible/session/cache_store.ex:24-25`）：

系统维护两个盐值以确保平滑过渡：

```elixir
found_session =
  find_session(event, event.user_id) || find_session(event, prev_user_id)
```

**盐值获取逻辑**（`lib/plausible/ingestion/event.ex:383-396`）：

```elixir
defp put_salts(%__MODULE__{} = event, _context) do
  %{event | salts: Plausible.Session.Salts.fetch()}
end

defp put_user_id(%__MODULE__{} = event, _context) do
  update_event_attrs(event, %{
    user_id:
      generate_user_id(
        event.request,
        event.domain,
        event.clickhouse_event_attrs.hostname,
        event.salts.current  # 使用当前盐值
      )
  })
end
```

**盐值轮换影响**：

| 场景 | 行为 | 结果 |
|------|------|------|
| 同一天内 | 使用 `current` 盐值 | 同一访客生成相同 `user_id` |
| 盐值切换日 | 先用 `current` 查找，再用 `previous` | 尽可能保持会话连续性 |
| 次日 | 新 `current` 盐值 | **同一访客生成不同 `user_id`** |

### 3.3 新老访客区分的实际状态

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
| **无持久化存储** | 每次查询需要全表扫描历史数据，性能极低 |
| **IP/UA变化** | 网络环境变化导致 `user_id` 变化，误判为新访客 |

### 3.4 访客统计的实际口径

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

### 3.5 实时访客统计

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

### 4.2 基于现有数据结构的潜在留存计算

如需实现留存分析，可基于现有数据进行**二次计算**。以下是理论上的实现方式：

#### 4.2.1 同期群（Cohort）分析框架

**同期群定义**：按首次访问时间分组

```
同期群 = {site_id, first_visit_date}
```

**但存在的问题**：
- 无 `first_visit_date` 字段
- `user_id` 每日变化，无法长期追踪

#### 4.2.2 单日新老访客区分（理论实现）

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

#### 4.2.3 留存率计算（理论实现）

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

### 4.3 盐值轮换对留存分析的根本性影响

**核心问题说明**：

```
场景：同一访客连续3天访问站点

第1天（盐值 S1）：
  user_id = SipHash(S1, UA + IP + ...) = U1
  活动记录：events_v2 中存储 U1

第2天（盐值 S2）：
  user_id = SipHash(S2, UA + IP + ...) = U2  (≠ U1)
  活动记录：events_v2 中存储 U2
  ← 系统会尝试用 S1 查找会话，但新事件用 S2 生成

第3天（盐值 S3）：
  user_id = SipHash(S3, UA + IP + ...) = U3  (≠ U1, ≠ U2)
  活动记录：events_v2 中存储 U3

留存分析（查询第1天访客在第3天是否回访）：
  第1天 user_id 集合：{U1}
  第3天 user_id 集合：{U3}
  交集：{}（空集）
  结果：0% 留存率 ← 严重低估！
```

**实际影响矩阵**：

| 分析类型 | 可行性 | 准确性 | 说明 |
|---------|--------|--------|------|
| 单日访客统计 | ✅ 完全支持 | ✅ 准确 | `uniq(user_id)` 单日内稳定 |
| 日内会话分析 | ✅ 完全支持 | ✅ 准确 | 30分钟超时规则 |
| 跨天新老区分 | ⚠️ 需二次计算 | ❌ 不准确 | 盐值轮换导致 `user_id` 变化 |
| N日留存率 | ❌ 无法准确计算 | ❌ 无意义 | 跨天 `user_id` 无关联性 |
| 同期群分析 | ❌ 无法准确计算 | ❌ 无意义 | 无法追踪长期访客 |

### 4.4 访客统计的有效口径

**当前系统可准确计算的指标**：

| 指标 | 计算方式 | 时间范围 | 准确性 |
|------|---------|---------|--------|
| **实时访客** | 过去N分钟 `uniq(user_id)` | 分钟级 | ✅ 准确 |
| **日访客** | 当天 `uniq(user_id)` | 自然日 | ✅ 准确 |
| **日会话** | 当天 `uniq(session_id)` 或 `sum(sign)` | 自然日 | ✅ 准确 |
| **页面浏览** | `countIf(name='pageview')` | 任意 | ✅ 准确 |
| **跳出率** | `sum(is_bounce * sign) / sum(sign)` | 任意 | ✅ 准确 |
| **平均会话时长** | `sum(duration * sign) / sum(sign)` | 任意 | ✅ 准确 |

**需要谨慎使用的指标**：

| 指标 | 问题 | 建议 |
|------|------|------|
| 跨天访客趋势 | 盐值轮换可能导致波动 | 仅作趋势参考，不作精确对比 |
| 周/月访客统计 | 同上 | 理解数据局限性 |
| 任何形式的"新老访客" | 无持久化标记 | 不建议使用，或需自定义实现 |

### 4.5 设计初衷与权衡

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

### 5.3 跨维度查询兼容性

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

### 5.4 自动维度过滤

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

### 5.5 表选择决策逻辑

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

### 5.6 指标计算一致性

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

### 5.7 访客指标在不同维度下的一致性

**`visitors` 指标的多源计算**：

`visitors` 指标可从两个表计算，但口径一致：

| 数据源 | 计算方式 | 适用场景 |
|--------|---------|---------|
| `events_v2` | `uniq(user_id)` | 有事件维度/过滤时 |
| `sessions_v2` | `uniq(user_id)` | 有会话维度/过滤时 |
| `sessions_v2`（涂抹） | 跨时间槽位去重 | `time:minute`/`time:hour` 维度时 |

**一致性保证**：

1. **同一时间窗口**：
   - 两个表的 `user_id` 来自同一事件处理管道
   - 去重逻辑相同（`uniq()`）

2. **涂抹机制的特殊处理**：
   - 仅在 `time:minute` 或 `time:hour` 维度触发
   - 目的是更准确地统计跨时间边界的会话
   - 不会改变 `user_id` 的基础定义

3. **盐值轮换的全局影响**：
   - 所有维度、所有表都使用相同的盐值生成 `user_id`
   - 同一天内，不同维度下的 `visitors` 统计具有一致性
   - 跨天时，所有维度都会受到盐值轮换的相同影响

**不一致场景**：

| 场景 | 问题 | 说明 |
|------|------|------|
| 跨天对比 | 盐值轮换 | 同一访客有不同 `user_id`，导致访客数"虚增" |
| 涂抹 vs 非涂抹 | 时间边界处理 | 小时/分钟维度的统计方式不同，但更准确 |
| 采样数据 | 采样因子 | 需使用 `scale_sample` 确保一致性 |

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
| 实时访客统计 | `lib/plausible/stats/current_visitors.ex` | 6-19 |
| 访客指标定义 | `lib/plausible/stats/metrics.ex` | 12-26 |
| 事件表访客计数 | `lib/plausible/stats/sql/expression.ex` | 302-306 |
| 会话表访客计数 | `lib/plausible/stats/sql/expression.ex` | 462-466 |
| 历史迁移（参考） | `priv/repo/migrations/20181214201821_add_new_visitor_to_pageviews.exs` | - |

### 6.3 时间统计相关

| 功能 | 文件路径 | 行号 |
|------|---------|------|
| UTC边界计算 | `lib/plausible/stats/time.ex` | 8-28 |
| 时间维度检测 | `lib/plausible/stats/time.ex` | 38-43 |
| 时间标签生成 | `lib/plausible/stats/time.ex` | 48-121 |
| 部分桶识别 | `lib/plausible/stats/time.ex` | 123-153 |
| 时间范围构建 | `lib/plausible/stats/query_builder.ex` | 88-150 |
| 时区转换 | `lib/plausible/stats/query_builder.ex` | 165-169 |
| timeSlots宏 | `lib/plausible/stats/sql/expression.ex` | 30-50 |
| 小时维度涂抹 | `lib/plausible/stats/sql/expression.ex` | 117-133 |

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

### 附录C：多表查询决策

```
┌─────────────────────────────────────────────────────────────────┐
│                         查询分析                                  │
│  指标: visitors, bounce_rate                                     │
│  维度: time:hour, visit:country                                  │
│  过滤: event:page = '/home'                                      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────────┐
              │         组件分类                   │
┌─────────────┴─────────────┬─────────────────────┴───────────┐
│          事件侧           │            会话侧                  │
│  • event:page 过滤器      │  • bounce_rate 指标              │
│  • (visitors 可从任意表)  │  • visit:country 维度            │
│                          │  • time:hour (涂抹触发)           │
└──────────────────────────┴───────────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────────┐
              │         执行两个查询               │
┌─────────────┴─────────────┬─────────────────────┴───────────┐
│     events_v2 查询        │      sessions_v2 查询            │
│  • visitors (uniq)        │  • visitors (涂抹)               │
│  • pageviews 过滤应用      │  • bounce_rate                   │
│                          │  • time:hour 维度                  │
└──────────────────────────┴───────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │   结果合并返回   │
                    └─────────────────┘
```

### 附录D：指标口径总览

```
┌─────────────────────────────────────────────────────────────────────┐
│                        指标口径总览                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐│
│  │   会话指标      │    │   事件指标      │    │   访客指标      ││
│  │  (sessions_v2) │    │  (events_v2)   │    │   (双表均可)    ││
│  └─────────────────┘    └─────────────────┘    └─────────────────┘│
│  │                  │    │                  │    │                  ││
│  │ • bounce_rate    │    │ • pageviews      │    │ • visitors     ││
│  │ • visit_duration │    │ • events         │    │ • visits       ││
│  │ • views_per_visit│    │ • time_on_page   │    │                ││
│  │ • exit_rate      │    │ • scroll_depth   │    │  ⚠️ 限制:      ││
│  │                  │    │                  │    │  • 盐值每日轮换 ││
│  │ ✅ 跨天稳定      │    │ ✅ 跨天稳定      │    │  • 无新老区分   ││
│  │ ✅ 基于 sign 聚合│    │ ✅ 简单计数      │    │  • 无留存追踪   ││
│  └──────────────────┘    └──────────────────┘    └──────────────────┘│
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 修订历史

| 版本 | 日期 | 修订内容 |
|------|------|---------|
| 1.0 | 2024-01 | 初始版本，基于代码库分析 |
| 2.0 | 2024-01 | 补充访客识别、新老区分、留存指标分析；明确盐值轮换的影响 |

---

**报告生成依据**：
- 代码库版本：当前工作目录 `g:\fangzheng\solo-dogfeeding\code\21076-analytics`
- 分析日期：2026-05-03
- 分析范围：Elixir 源代码，含测试文件引用

**关键发现总结**：

1. ✅ **会话边界清晰**：30分钟无活动超时，基于内存缓存 + ClickHouse CollapsingMergeTree
2. ✅ **时间统计完善**：支持多粒度时间维度，小时/分钟级有涂抹机制处理跨边界会话
3. ⚠️ **访客识别有局限**：`user_id` 基于盐值每日轮换，设计目标是隐私保护而非长期追踪
4. ❌ **无内置新老访客区分**：v2 模型无 `new_visitor` 字段，需二次计算且不准确
5. ❌ **无内置留存指标**：代码库无 `retention`/`cohort` 相关实现，且盐值轮换使跨天追踪无意义
6. ✅ **维度一致性良好**：事件/会话维度分类清晰，有严格的兼容性检查和表选择策略