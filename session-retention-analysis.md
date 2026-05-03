# 访客会话与留存指标计算口径分析报告

## 目录
1. [会话边界定义](#会话边界定义)
2. [跨时间段统计方式](#跨时间段统计方式)
3. [不同维度下的口径一致性](#不同维度下的口径一致性)
4. [代码参考位置](#代码参考位置)

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

## 3. 不同维度下的口径一致性

### 3.1 维度分类体系

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

### 3.2 指标表归属规则

**指标分类**（`lib/plausible/stats/table_decider.ex:146-177`）：

| 指标 | 归属表 | 计算方式 |
|------|--------|---------|
| `visitors` | `:either` | 去重计数 `uniq(user_id)` |
| `visits` | `:either` | 去重计数 `uniq(session_id)` |
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

### 3.3 跨维度查询兼容性

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

### 3.4 自动维度过滤

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

### 3.5 表选择决策逻辑

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

### 3.6 指标计算一致性

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

---

## 4. 代码参考位置

### 4.1 会话边界相关

| 功能 | 文件路径 | 行号 |
|------|---------|------|
| 会话超时配置 | `config/config.exs` | 57-59 |
| 会话查找逻辑 | `lib/plausible/session/cache_store.ex` | 69-81 |
| 会话更新逻辑 | `lib/plausible/session/cache_store.ex` | 95-123 |
| 新会话创建 | `lib/plausible/session/cache_store.ex` | 125-161 |
| 会话ID生成 | `lib/plausible/clickhouse_session_v2.ex` | 80-82 |
| 用户ID生成 | `lib/plausible/ingestion/event.ex` | 553-567 |
| 盐值获取 | `lib/plausible/session/salts.ex` | - |
| 会话平衡器 | `lib/plausible/session/balancer.ex` | - |

### 4.2 时间统计相关

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

### 4.3 维度一致性相关

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

### 4.4 数据模型

| 模型 | 文件路径 | 说明 |
|------|---------|------|
| 会话Schema | `lib/plausible/clickhouse_session_v2.ex` | sessions_v2 表结构 |
| 事件处理 | `lib/plausible/ingestion/event.ex` | 事件处理管道 |
| 持久化器 | `lib/plausible/ingestion/persistor.ex` | 会话持久化入口 |
| 写入缓冲区 | `lib/plausible/session/write_buffer.ex` | 批量写入缓冲 |

---

## 附录：关键数据流图

### 会话生命周期
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

### 多表查询决策
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

---

## 修订历史

| 版本 | 日期 | 修订内容 |
|------|------|---------|
| 1.0 | 2024-01 | 初始版本，基于代码库分析 |

---

**报告生成依据**：
- 代码库版本：当前工作目录 `g:\fangzheng\solo-dogfeeding\code\21076-analytics`
- 分析日期：2026-05-03
- 分析范围：Elixir 源代码，不含测试文件（除非特别引用）