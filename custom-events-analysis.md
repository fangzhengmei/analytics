# 自定义事件、属性和 Revenue 指标查询链路分析报告

## 1. 概述

本文档详细分析了 Plausible Analytics 中自定义事件、自定义属性和 Revenue 指标的完整查询链路，包括采集校验、存储映射、报表聚合和扩展成本四个核心环节。

---

## 2. 采集校验环节

### 2.1 前端数据采集

**核心文件**: `tracker/src/track.js`

#### 2.1.1 事件采集流程

前端通过 `track()` 函数采集事件数据，支持以下参数：

| 参数 | 说明 | 对应字段 |
|------|------|----------|
| `eventName` | 事件名称（如 'pageview', 'signup'） | `n` / `name` |
| `options.u` / `options.url` | 自定义 URL | `u` |
| `options.props` | 自定义属性 | `p` / `props` |
| `options.revenue` | Revenue 数据 | `$` / `revenue` |
| `options.interactive` | 是否为交互事件 | `i` |

#### 2.1.2 关键校验点

```javascript
// 本地环境过滤
if (/^localhost$|^127(\.[0-9]+){0,2}\.[0-9]+$|^\[::1?\]$/.test(location.hostname)) {
  return onIgnoredEvent(eventName, options, 'localhost')
}

// 爬虫/自动化工具过滤
if ((window._phantom || window.__nightmare || window.navigator.webdriver || window.Cypress) && !window.__plausible) {
  return onIgnoredEvent(eventName, options)
}

// localStorage 忽略标记
if (window.localStorage.plausible_ignore === 'true') {
  return onIgnoredEvent(eventName, options, 'localStorage flag')
}
```

### 2.2 后端请求校验

**核心文件**: `lib/plausible/ingestion/request.ex`

#### 2.2.1 请求构建流程

```elixir
def build(%Plug.Conn{} = conn, now \\ NaiveDateTime.utc_now()) do
  changeset =
    %__MODULE__{}
    |> Changeset.change()
    |> Changeset.put_change(:timestamp, NaiveDateTime.truncate(now, :second))

  case parse_body(conn) do
    {:ok, request_body, conn} ->
      request =
        changeset
        |> put_ip_classification(conn)
        |> put_remote_ip(conn)
        |> put_uri(request_body)
        |> put_hostname()
        |> put_user_agent(conn)
        |> put_request_params(request_body)
        |> put_referrer(request_body)
        |> put_pathname()
        |> put_props(request_body)        # 自定义属性处理
        |> put_engagement_fields(request_body)
        |> put_query_params()
        |> put_revenue_source(request_body) # Revenue 处理
        |> put_interactive(request_body)
        |> put_tracker_script_version(request_body)
        |> map_domains(request_body)
        |> Changeset.validate_required([:event_name, :hostname, :pathname, :timestamp])
        |> Changeset.validate_length(:event_name, max: 120)  # 事件名称长度限制
        |> Changeset.apply_action(nil)
      ...
  end
end
```

#### 2.2.2 自定义属性校验

```elixir
@max_props 30  # 每个事件最多 30 个属性
@max_prop_key_length Plausible.Props.max_prop_key_length()  # 300 字节
@max_prop_value_length Plausible.Props.max_prop_value_length()  # 2000 字节

defp put_props(changeset, %{} = request_body) do
  props =
    (request_body["m"] || request_body["meta"] || request_body["p"] || request_body["props"])
    |> Plausible.Helpers.JSON.decode_or_fallback()
    |> Enum.reduce([], &filter_bad_props/2)  # 过滤无效属性
    |> maybe_set_props_path_to_pathname(changeset)
    |> Enum.take(@max_props)  # 限制数量
    |> Map.new()

  changeset
  |> Changeset.put_change(:props, props)
  |> validate_props()  # 校验键值长度
end

defp filter_bad_props({k, v}, acc) do
  cond do
    # 过滤列表和映射类型
    Enum.any?([k, v], &(is_list(&1) or is_map(&1))) -> acc
    # 过滤空字符串
    Enum.any?([k, v], &(String.trim_leading(to_string(&1)) == "")) -> acc
    true -> [{to_string(k), to_string(v)} | acc]
  end
end
```

#### 2.2.3 Revenue 数据校验

**核心文件**: `extra/lib/plausible/ingestion/request/revenue.ex`

```elixir
def put_revenue_source(%Ecto.Changeset{} = changeset, %{} = request_body) do
  with revenue_source <- request_body["revenue"] || request_body["$"],
       %{"amount" => _, "currency" => _} = revenue_source <-
         Plausible.Helpers.JSON.decode_or_fallback(revenue_source) do
    parse_revenue_source(changeset, revenue_source)
  else
    _any -> changeset
  end
end

@valid_currencies Plausible.Goal.Revenue.valid_currencies()
defp parse_revenue_source(changeset, %{"amount" => amount, "currency" => currency}) do
  with true <- currency in @valid_currencies,  # 校验货币有效性
       {%Decimal{} = amount, _rest} <- parse_decimal(amount),  # 解析金额
       %Money{} = amount <- Money.new(currency, amount) do
    Ecto.Changeset.put_change(changeset, :revenue_source, amount)
  else
    _any -> changeset
  end
end
```

### 2.3 校验规则汇总

| 数据类型 | 校验规则 | 限制值 |
|----------|----------|--------|
| 事件名称 | 必填、长度限制 | 最大 120 字节 |
| 自定义属性数量 | 每个事件最多 | 30 个 |
| 属性键长度 | 字节限制 | 最大 300 字节 |
| 属性值长度 | 字节限制 | 最大 2000 字节 |
| 属性类型 | 仅允许简单类型 | 不允许列表/映射 |
| Revenue 货币 | 必须为有效货币 | ISO 4217 代码 |
| Revenue 金额 | 必须可解析为 Decimal | 支持整数/浮点数/字符串 |

---

## 3. 存储映射环节

### 3.1 事件处理管道

**核心文件**: `lib/plausible/ingestion/event.ex`

#### 3.1.1 处理流程

```elixir
defp pipeline() do
  [
    drop_verification_agent: &drop_verification_agent/2,
    drop_datacenter_ip: &drop_datacenter_ip/2,
    drop_threat_ip: &drop_threat_ip/2,
    drop_shield_rule_hostname: &drop_shield_rule_hostname/2,
    drop_shield_rule_page: &drop_shield_rule_page/2,
    drop_shield_rule_ip: &drop_shield_rule_ip/2,
    put_geolocation: &put_geolocation/2,
    drop_shield_rule_country: &drop_shield_rule_country/2,
    put_user_agent: &put_user_agent/2,
    put_basic_info: &put_basic_info/2,
    put_source_info: &put_source_info/2,
    maybe_infer_medium: &maybe_infer_medium/2,
    put_props: &put_props/2,              # 自定义属性映射
    put_revenue: &put_revenue/2,          # Revenue 映射
    put_salts: &put_salts/2,
    put_user_id: &put_user_id/2,
    validate_clickhouse_event: &validate_clickhouse_event/2,
    register_session: &register_session/2
  ]
end
```

#### 3.1.2 自定义属性存储映射

```elixir
defp put_props(%__MODULE__{request: %{props: %{} = props}} = event, _context) do
  # 确保键值顺序一致
  {keys, values} = Enum.unzip(props)

  update_event_attrs(event, %{
    "meta.key": keys,
    "meta.value": values
  })
end
```

**存储设计**: 自定义属性采用**两个并行数组**存储，而非键值对结构，这是 ClickHouse 列式存储的最佳实践。

#### 3.1.3 Revenue 数据存储映射

**核心文件**: `extra/lib/plausible/ingestion/event/revenue.ex`

```elixir
def get_revenue_attrs(event) do
  case event.request.revenue_source do
    %Money{} = revenue ->
      %{
        revenue_source_amount: revenue.amount,
        revenue_source_currency: revenue.currency,
        revenue_reporting_amount: revenue.amount,  # TODO: 汇率转换
        revenue_reporting_currency: revenue.currency
      }
    _ ->
      %{}
  end
end
```

### 3.2 ClickHouse 数据模型

**核心文件**: `lib/plausible/clickhouse_event_v2.ex`

#### 3.2.1 表结构定义

```elixir
schema "events_v2" do
  # 基础字段
  field :name, Ch, type: "LowCardinality(String)"  # 事件名称
  field :site_id, Ch, type: "UInt64"
  field :timestamp, :naive_datetime

  # 自定义属性 - 并行数组设计
  field :"meta.key", {:array, :string}
  field :"meta.value", {:array, :string}

  # Revenue 字段 - 双货币设计
  field :revenue_source_amount, Ch, type: "Nullable(Decimal64(3))"
  field :revenue_source_currency, Ch, type: "FixedString(3)"
  field :revenue_reporting_amount, Ch, type: "Nullable(Decimal64(3))"
  field :revenue_reporting_currency, Ch, type: "FixedString(3)"

  # 会话属性（反规范化）
  field :referrer, :string
  field :referrer_source, :string
  field :country_code, Ch, type: "FixedString(2)"
  field :screen_size, Ch, type: "LowCardinality(String)"
  field :operating_system, Ch, type: "LowCardinality(String)"
  field :browser, Ch, type: "LowCardinality(String)"
  # ... 更多字段
end
```

#### 3.2.2 存储设计要点

| 设计决策 | 说明 | 优势 |
|----------|------|------|
| 并行数组存储属性 | `meta.key` 和 `meta.value` 两个数组 | 列式存储友好，查询高效 |
| LowCardinality 类型 | 高基数重复字段使用此类型 | 减少存储，提高压缩率 |
| 双货币 Revenue 设计 | source + reporting | 支持原始货币和报表货币 |
| Decimal64(3) | 金额使用定点数 | 避免浮点精度问题 |
| 会话属性反规范化 | 会话字段直接存储在事件表 | 避免 JOIN，查询性能高 |

---

## 4. 报表聚合环节

### 4.1 目标过滤机制

**核心文件**: `lib/plausible/stats/goals.ex`

#### 4.1.1 目标预加载

```elixir
def preload_needed_goals(site, dimensions, filters) do
  if Enum.member?(dimensions, "event:goal") or
       Filters.filtering_on_dimension?(filters, "event:goal") do
    site = Plausible.Repo.preload(site, :team)
    props_available? = Plausible.Billing.Feature.Props.check_availability(site.team) == :ok
    goals = Plausible.Goals.for_site(site, include_goals_with_custom_props?: props_available?)

    %{
      matching_toplevel_filters: goals_matching_toplevel_filters(goals, filters),
      all: goals
    }
  else
    %{all: [], matching_toplevel_filters: []}
  end
end
```

#### 4.1.2 目标条件构建

```elixir
def goal_condition(goal, imported? \\ false) do
  type = Plausible.Goal.type(goal)
  goal_condition(type, goal, imported?)
end

defp goal_condition(:event, goal, _) do
  name_condition = dynamic([e], e.name == ^goal.event_name)

  if Plausible.Goal.has_custom_props?(goal) do
    custom_props_condition = build_custom_props_condition(goal.custom_props)
    dynamic([e], ^name_condition and ^custom_props_condition)
  else
    name_condition
  end
end
```

#### 4.1.3 自定义属性过滤条件

```elixir
defp build_custom_props_condition(custom_props) do
  Enum.reduce(custom_props, true, fn {prop_key, prop_value}, acc ->
    condition =
      dynamic(
        [e],
        fragment(
          "?[indexOf(?, ?)] = ?",
          field(e, :"meta.value"),  # 值数组
          field(e, :"meta.key"),    # 键数组
          ^prop_key,                 # 目标键
          ^prop_value                # 目标值
        )
      )

    dynamic([e], ^acc and ^condition)
  end)
end
```

**查询原理**: 使用 `indexOf(meta.key, prop_key)` 找到键在数组中的索引，然后用该索引从 `meta.value` 数组中获取对应的值进行比较。

### 4.2 Revenue 指标聚合

**核心文件**: `extra/lib/plausible/stats/goal/revenue.ex`

#### 4.2.1 支持的 Revenue 指标

```elixir
@revenue_metrics [:average_revenue, :total_revenue]

def revenue_metrics(), do: @revenue_metrics
```

#### 4.2.2 货币处理逻辑

```elixir
def preload(site, preloaded_goals, metrics, dimensions) do
  cond do
    not requested?(metrics) -> {nil, %{}}
    not available?(site) -> {:revenue_goals_unavailable, %{}}
    true -> preload(preloaded_goals.matching_toplevel_filters, dimensions)
  end
end

defp preload(goals, dimensions) do
  goal_currency_map =
    goals
    |> Map.new(fn goal -> {Plausible.Goal.display_name(goal), goal.currency} end)
    |> Map.reject(fn {_goal, currency} -> is_nil(currency) end)

  currencies = goal_currency_map |> Map.values() |> Enum.uniq()
  goal_dimension? = "event:goal" in dimensions

  case {currencies, goal_dimension?} do
    {[currency], false} -> {nil, %{default: currency}}  # 单一货币
    {[], _} -> {:no_revenue_goals_matching, %{}}
    {_, true} -> {nil, goal_currency_map}  # 按目标分组，支持多货币
    _ -> {:no_single_revenue_currency, %{}}  # 多货币但无分组，无法计算
  end
end
```

#### 4.2.3 Revenue 格式化

```elixir
def format_revenue_metric(value, currency) do
  if currency do
    money = Money.new!(value || 0, currency)

    %{
      short: Money.to_string!(money, format: :short, fractional_digits: 1),
      long: Money.to_string!(money),
      value: Decimal.to_float(money.amount),
      currency: currency
    }
  else
    value
  end
end
```

### 4.3 自定义属性查询

**核心文件**: `lib/plausible/stats/custom_props.ex`

#### 4.3.1 获取属性名列表

```elixir
def fetch_prop_names(site, query) do
  case Filters.get_toplevel_filter(query, "event:props:") do
    [_op, "event:props:" <> key | _rest] ->
      [key]  # 已指定具体属性

    _ ->
      from(e in base_event_query(query),
        join: meta in fragment("meta"),  # 数组展开
        hints: "ARRAY",
        on: true,
        select: meta.key,
        distinct: true
      )
      |> maybe_allowed_props_only(site)  # 权限过滤
      |> ClickhouseRepo.all()
  end
end

def maybe_allowed_props_only(q, site) do
  case Plausible.Props.allowed_for(site) do
    :all -> q
    allowed_props -> from [..., m] in q, where: m.key in ^allowed_props
  end
end
```

### 4.4 聚合查询流程

```
用户查询
    │
    ▼
┌─────────────────┐
│ Query.parse()   │ 解析查询参数
│  - 指标解析      │
│  - 维度解析      │
│  - 过滤器解析    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ QueryRunner.run()│ 执行查询
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ QueryBuilder    │ 构建 SQL
│  - 选择表        │
│  - 构建 WHERE    │
│  - 构建 GROUP BY │
│  - 聚合函数      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ ClickHouse      │ 执行查询
│  - 数组索引查询  │
│  - 聚合计算      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 结果格式化      │
│  - Revenue 格式化│
│  - 百分比计算    │
└────────┬────────┘
         │
         ▼
      返回结果
```

---

## 5. 扩展成本分析

### 5.1 资源限制

**核心文件**: `lib/plausible/props.ex`

#### 5.1.1 配置限制

```elixir
@max_props 300          # 每个站点最多 300 个可配置属性
@max_prop_key_length 300    # 属性键最大长度
@max_prop_value_length 2000  # 属性值最大长度
```

#### 5.1.2 运行时限制

| 限制项 | 值 | 说明 |
|--------|-----|------|
| 单事件属性数量 | 30 | `put_props` 中 `Enum.take(@max_props)` |
| 站点可配置属性 | 300 | `allowed_event_props` 列表最大长度 |
| 属性键长度 | 300 字节 | 校验时检查 |
| 属性值长度 | 2000 字节 | 校验时检查 |
| 事件名称长度 | 120 字节 | Changeset 校验 |

### 5.2 计费特性

#### 5.2.1 自定义属性特性

```elixir
def allowed_for(site, opts \\ []) do
  site = Plausible.Repo.preload(site, :team)
  internal_keys = Plausible.Props.internal_keys()
  props_enabled? = Plausible.Billing.Feature.Props.check_availability(site.team) == :ok
  bypass_setup? = Keyword.get(opts, :bypass_setup?)

  cond do
    props_enabled? && is_nil(site.allowed_event_props) -> :all
    props_enabled? && bypass_setup? -> :all
    props_enabled? -> site.allowed_event_props ++ internal_keys
    true -> internal_keys  # 仅内部属性
  end
end
```

#### 5.2.2 Revenue Goals 特性

```elixir
def available?(site) do
  site = Plausible.Repo.preload(site, :team)
  Plausible.Billing.Feature.RevenueGoals.check_availability(site.team) == :ok
end
```

#### 5.2.3 特性依赖矩阵

| 功能 | 所需特性 | 内部属性访问 |
|------|----------|-------------|
| 页面浏览事件 | 无 | 总是可用 |
| 自定义事件追踪 | 无 | 总是可用 |
| 自定义属性查询 | `Props` | 仅 `url`, `path`, `search_query` |
| Revenue Goals | `RevenueGoals` | 不可用 |

### 5.3 性能成本

#### 5.3.1 存储成本

- **自定义属性**: 每个属性存储为两个字符串（键+值），使用数组压缩
- **Revenue**: 4 个字段（2 个 Decimal64 + 2 个 FixedString(3)）≈ 16 字节/事件
- **LowCardinality 字段**: 重复值高的字段使用字典编码，显著减少存储

#### 5.3.2 查询成本

| 查询类型 | 复杂度 | 说明 |
|----------|--------|------|
| 按事件名称过滤 | O(log n) | 有索引，高效 |
| 按单个属性过滤 | O(n) | 需扫描数组，无直接索引 |
| 按多个属性过滤 | O(n * k) | k 为属性数量 |
| 按属性分组 | O(n) | 需展开数组 |
| Revenue 聚合 | O(n) | 需扫描相关事件 |

#### 5.3.3 索引策略

当前设计中，`meta.key` 和 `meta.value` 数组**没有直接索引**，属性过滤依赖：

1. **分区裁剪**: 按 `site_id` + `timestamp` 分区
2. **前置过滤**: 先按事件名称、时间范围等过滤，减少扫描行数
3. **向量化执行**: ClickHouse 的向量化处理缓解数组扫描开销

### 5.4 扩展建议

#### 5.4.1 高基数属性优化

对于高基数属性（如 `user_id`、`order_id`），建议：

1. **避免存储为自定义属性**: 考虑使用专门字段或外部系统
2. **控制属性值长度**: 长值会增加存储和扫描成本
3. **限制单事件属性数量**: 过多属性会增加查询复杂度

#### 5.4.2 常见查询模式优化

| 场景 | 建议 |
|------|------|
| 频繁按某个属性过滤 | 考虑添加专门字段或使用 Materialized View |
| 多属性组合查询 | 评估是否需要更结构化的存储 |
| 高吞吐量事件 | 评估采样策略或数据保留策略 |
| Revenue 多货币场景 | 确保所有目标使用统一货币，或按 `event:goal` 分组 |

---

## 6. 关键代码位置汇总

| 功能模块 | 文件路径 | 关键函数/模块 |
|----------|----------|--------------|
| 前端采集 | `tracker/src/track.js` | `track()` |
| 请求解析 | `lib/plausible/ingestion/request.ex` | `build/2` |
| 属性校验 | `lib/plausible/ingestion/request.ex` | `put_props/2`, `validate_props/1` |
| Revenue 校验 | `extra/lib/plausible/ingestion/request/revenue.ex` | `put_revenue_source/2` |
| 事件处理管道 | `lib/plausible/ingestion/event.ex` | `pipeline/0` |
| 属性存储映射 | `lib/plausible/ingestion/event.ex` | `put_props/2` |
| Revenue 存储映射 | `extra/lib/plausible/ingestion/event/revenue.ex` | `get_revenue_attrs/1` |
| 数据模型 | `lib/plausible/clickhouse_event_v2.ex` | `schema "events_v2"` |
| 目标过滤 | `lib/plausible/stats/goals.ex` | `add_filter/3`, `goal_condition/2` |
| Revenue 聚合 | `extra/lib/plausible/stats/goal/revenue.ex` | `preload/4`, `format_revenue_metric/3` |
| 属性查询 | `lib/plausible/stats/custom_props.ex` | `fetch_prop_names/2` |
| 配置管理 | `lib/plausible/props.ex` | `allowed_for/2` |
| 聚合查询 | `lib/plausible/stats/aggregate.ex` | `aggregate/3` |

---

## 7. 总结

### 7.1 架构优势

1. **灵活的采集层**: 支持多种属性传递方式（`props`, `meta`, `p`）
2. **严格的校验机制**: 多层校验确保数据质量
3. **ClickHouse 优化设计**: 并行数组、LowCardinality 等设计适应列式存储
4. **特性控制**: 基于计费特性的精细访问控制

### 7.2 潜在风险

1. **属性查询性能**: 数组索引查询无直接索引，高基数场景可能较慢
2. **多货币限制**: 非单一货币且不按目标分组时无法计算 Revenue 聚合
3. **配置复杂度**: `allowed_event_props` 增加了运维复杂度

### 7.3 扩展建议

1. **监控属性基数**: 定期检查高基数属性，评估是否需要优化
2. **查询模式分析**: 识别高频属性查询，考虑 Materialized View
3. **容量规划**: 根据事件量和属性数量预估存储增长
4. **货币策略**: 建议使用统一报表货币简化聚合逻辑

---

*报告生成时间: 2026-05-03*
*基于代码版本: 当前工作目录*
