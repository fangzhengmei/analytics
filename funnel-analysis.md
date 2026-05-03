# Funnel 分析实现思路深度分析

## 1. 概述

Funnel 分析是一种用于追踪用户从初始步骤到最终转化的整个旅程的分析方法。在 Plausible 系统中，Funnel 分析允许用户定义多个步骤（基于 Goals），然后分析用户在这些步骤之间的转化和流失情况。

## 2. 架构概览

Funnel 分析系统主要由以下几个模块组成：

1. **数据模型层**：定义 Funnel 及其步骤的数据结构
2. **业务逻辑层**：处理 Funnel 的 CRUD 操作
3. **统计分析层**：构造查询并计算 Funnel 结果
4. **API 层**：提供对外的 RESTful API
5. **前端展示层**：渲染 Funnel 图表和交互界面

## 3. 核心模块详解

### 3.1 数据模型层

#### 3.1.1 Funnel Schema (Plausible.Funnel)

**文件位置**：`extra/lib/plausible/funnel.ex`

Funnel 是整个分析的核心对象，其数据结构如下：

```elixir
schema "funnels" do
  field :name, :string
  field :strict_order, :boolean, default: false
  belongs_to :site, Plausible.Site

  has_many :steps, Step,
    preload_order: [
      asc: :step_order
    ],
    on_replace: :delete

  has_many :goals, through: [:steps, :goal]
  timestamps()
end
```

**关键字段说明**：
- `name`：Funnel 的名称，用于标识和展示
- `strict_order`：布尔值，指示是否严格按照步骤顺序执行
- `steps`：Funnel 的步骤列表，通过 `has_many` 关联
- `step_order`：步骤的顺序，1 表示第一个步骤

**约束条件**：
- 最少 2 个步骤，最多 8 个步骤（`@min_steps` 和 `@max_steps` 常量定义）
- 同一站点下的 Funnel 名称必须唯一

#### 3.1.2 Step Schema

虽然代码中没有直接展示 Step 模块的完整实现，但从上下文可以推断出其结构：
- 每个 Step 关联一个 Goal（自定义事件或页面访问）
- 包含 `step_order` 字段用于排序

### 3.2 业务逻辑层

#### 3.2.1 Funnels 模块 (Plausible.Funnels)

**文件位置**：`extra/lib/plausible/funnels.ex`

该模块提供了 Funnel 的 CRUD 操作和管理功能：

**主要功能**：
1. **创建 Funnel**：`create/4` 函数
   - 验证步骤数量（2-8 个）
   - 检查计费权限（Funnels 功能是否可用）
   - 插入数据库

2. **更新 Funnel**：`update/4` 函数
   - 类似创建逻辑，但针对现有 Funnel

3. **查询 Funnel**：
   - `get/2`：根据 ID 或名称获取单个 Funnel
   - `list/1`：获取站点下的所有 Funnel 列表
   - `with_goals_query/1`：预加载关联的 Goals

4. **临时定义**：`ephemeral_definition/4`
   - 不保存到数据库，用于表单预览和实时计算

**关键实现**：
- 使用 Ecto Changeset 进行数据验证
- 支持 `strict_order` 选项
- 与计费系统集成，检查功能可用性

### 3.3 统计分析层 (核心)

#### 3.3.1 Stats.Funnel 模块 (Plausible.Stats.Funnel)

**文件位置**：`extra/lib/plausible/stats/funnel.ex`

这是 Funnel 分析的核心模块，负责构造查询和计算结果。

**主要流程**：

1. **入口函数**：`funnel/3`
   - 接受站点、查询参数和 Funnel ID 或定义
   - 预加载 Goals 并构造基础查询
   - 执行查询并处理结果

2. **查询构造**：`funnel_query/2` 和 `select_funnel/2`
   - 构造 ClickHouse 查询，使用 `windowFunnel` 函数
   - 支持 `strict_order` 模式

3. **结果处理**：`backfill_steps/2`
   - 处理 ClickHouse 返回的原始数据
   - 计算转化率、流失率等指标

#### 3.3.2 真实漏斗查询接口的门禁条件

在执行 `windowFunnel` 分析之前，系统会应用多层门禁条件来过滤事件数据。这些门禁条件对于结果的准确性至关重要。

**完整的查询执行流程**：

```elixir
def funnel(_site, query, %Funnel{} = funnel) do
  goals = Enum.map(funnel.steps, & &1.goal)

  funnel_data =
    query
    |> Query.set(preloaded_goals: %{all: [], matching_toplevel_filters: goals})  # 第1层：设置预加载 Goals
    |> Base.base_event_query()                                                  # 第2层：基础事件查询
    |> funnel_query(funnel)                                                     # 第3层：Funnel 特定查询
    |> ClickhouseRepo.all(query: query)
  ...
end
```

**第1层：预加载 Goals 设置**

```elixir
Query.set(preloaded_goals: %{all: [], matching_toplevel_filters: goals})
```

这一步将 Funnel 定义的所有 Goals 传递给查询上下文，用于后续的事件名称过滤。

**第2层：基础事件查询门禁**

`Base.base_event_query/1` 函数应用了核心的门禁条件：

```elixir
def base_event_query(query) do
  events_q = query_events(query)

  if TableDecider.events_join_sessions?(query) do
    sessions_q =
      from(
        s in query_sessions(query),
        select: %{session_id: s.session_id},
        where: s.sign == 1,
        group_by: s.session_id
      )

    from(
      e in events_q,
      join: sq in subquery(sessions_q),
      on: e.session_id == sq.session_id
    )
  else
    events_q
  end
end
```

**`query_events/1` 中的门禁条件**：

```elixir
defp query_events(query) do
  q =
    from(e in "events_v2",
      where: ^SQL.WhereBuilder.build(:events, query),      # 门禁A：基础过滤
      where: ^SQL.WhereBuilder.derived_name_filter(query)  # 门禁B：事件名称过滤
    )

  on_ee do
    q = Plausible.Stats.Sampling.add_query_hint(q, query)  # 门禁C：采样提示
  end

  q
end
```

**详细的门禁条件解析**：

**门禁A：基础过滤 (WhereBuilder.build/2)**

```elixir
def build(table, query) do
  base_condition = filter_site_time_range(table, query)

  query.filters
  |> Enum.map(&add_filter(table, query, &1))
  |> Enum.reduce(base_condition, fn condition, acc -> dynamic([], ^acc and ^condition) end)
end

defp filter_site_time_range(table, query) do
  dynamic([], ^filter_site_id(query) and ^filter_time_range(table, query))
end
```

**门禁A1：Site ID 过滤**

```elixir
defp filter_site_id(query) do
  case query.consolidated_site_ids do
    nil ->
      dynamic([x], x.site_id == ^query.site_id)  # 单站点：精确匹配

    [_ | _] = ids ->
      dynamic([x], fragment("? in ?", x.site_id, ^ids))  # 多站点：IN 匹配
  end
end
```

**作用**：确保只查询当前站点（或授权的多站点）的事件数据。

**对结果准确性的影响**：
- **防止跨站点数据污染**：如果没有这个门禁，一个站点的 Funnel 分析可能会包含其他站点的用户行为
- **数据隔离**：确保多租户环境下的数据安全和准确性

**门禁A2：时间范围过滤**

```elixir
defp filter_time_range(:events, query) do
  {first_datetime, last_datetime} = utc_boundaries(query)
  dynamic([e], e.timestamp >= ^first_datetime and e.timestamp <= ^last_datetime)
end
```

**作用**：限制事件数据在用户选择的时间范围内。

**对结果准确性的影响**：
- **时间一致性**：确保分析的是用户指定时间段内的行为
- **避免历史数据干扰**：旧数据不会影响当前分析结果

**门禁B：事件名称过滤 (derived_name_filter/1)**

```elixir
def derived_name_filter(query) do
  cond do
    Plausible.Stats.Filters.filtering_on_dimension?(query.filters, "event:goal") ->
      true  # 已有 Goal 过滤器，不额外限制

    query.preloaded_goals.matching_toplevel_filters != [] ->
      names = goal_event_names(query.preloaded_goals.matching_toplevel_filters)
      dynamic([e], e.name in ^names)  # 限制为 Funnel 步骤涉及的事件类型

    :time_on_page not in query.metrics and :scroll_depth not in query.metrics ->
      dynamic([e], e.name != "engagement")  # 排除 engagement 事件（除非明确需要）

    true ->
      true
  end
end

defp goal_event_names(goals) do
  goals
  |> Enum.map(fn goal ->
    case Plausible.Goal.type(goal) do
      :event -> goal.event_name   # 自定义事件：使用事件名
      :page -> "pageview"          # 页面访问：固定为 "pageview"
      :scroll -> "engagement"      # 滚动深度：固定为 "engagement"
    end
  end)
  |> Enum.uniq()
end
```

**作用**：根据 Funnel 定义的 Goals，只查询相关类型的事件。

**对结果准确性的影响**：
- **减少无关事件干扰**：如果 Funnel 步骤都是页面访问，那么自定义事件（如 "Purchase"）不会被纳入分析
- **性能优化**：减少需要扫描的数据量
- **防止误判**：特别是在严格顺序模式下，无关事件可能导致匹配中断

**门禁C：额外的 Site ID 验证**

在 `funnel_query/2` 中还有一层额外的 Site ID 验证：

```elixir
defp funnel_query(query, funnel_definition) do
  q_events =
    from(e in query,
      select: %{user_id: e.user_id, _sample_factor: fragment("any(_sample_factor)")},
      where: e.site_id == ^funnel_definition.site_id,  # 额外的 Site ID 验证
      group_by: e.user_id,
      order_by: [desc: fragment("step")]
    )
    |> select_funnel(funnel_definition)
  ...
end
```

**作用**：双重验证，确保只查询 Funnel 所属站点的数据。

**对结果准确性的影响**：
- **防御性编程**：即使前面的查询上下文被错误修改，这层验证也能防止跨站点数据
- **数据一致性**：确保 Funnel 定义和查询数据属于同一站点

**门禁条件汇总表**：

| 门禁条件 | 实现位置 | 过滤逻辑 | 对结果准确性的影响 |
|---------|---------|---------|-------------------|
| Site ID 过滤 | `filter_site_id/1` | `site_id == query.site_id` | 防止跨站点数据污染，确保数据隔离 |
| 时间范围过滤 | `filter_time_range/2` | `timestamp >= first_datetime AND timestamp <= last_datetime` | 确保分析的是指定时间段内的行为 |
| 事件名称过滤 | `derived_name_filter/1` | `name IN [goal_event_names]` | 减少无关事件干扰，防止严格顺序模式下的误判 |
| 额外 Site ID 验证 | `funnel_query/2` | `site_id == funnel_definition.site_id` | 双重验证，防御性编程 |

**门禁条件缺失的风险**：

1. **没有 Site ID 过滤**：
   - 风险：一个站点的 Funnel 分析可能包含其他站点的用户数据
   - 示例：站点 A 的"浏览产品→购买"Funnel 可能错误地包含站点 B 的购买事件

2. **没有时间范围过滤**：
   - 风险：分析结果可能包含历史数据，导致转化率虚高或虚低
   - 示例：用户选择"过去7天"分析，但查询返回了过去30天的数据

3. **没有事件名称过滤**：
   - 风险：在严格顺序模式下，无关事件可能导致匹配中断
   - 示例：Funnel 步骤是"页面A→页面B"，但用户在页面A之后触发了"Purchase"事件（不是步骤的一部分），在严格顺序模式下会被误判为中断

4. **没有双重 Site ID 验证**：
   - 风险：如果查询上下文被错误修改，可能导致数据泄露
   - 示例：查询参数的 `site_id` 被篡改为其他站点的 ID

#### 3.3.3 严格顺序判定前的事件过滤前提

在理解 `strict_order` 模式的行为之前，必须明确：**`windowFunnel` 函数只在已经经过门禁条件过滤的事件序列中进行匹配**。

**完整的事件处理流程**：

```
原始事件数据
    ↓
【门禁条件过滤】
├── Site ID 匹配
├── 时间范围匹配
└── 事件名称匹配（基于 Funnel 步骤的 Goals）
    ↓
【过滤后的事件序列】
    ↓
【windowFunnel 匹配】
├── 按 user_id 分组
├── 按 timestamp 排序
└── 应用 strict_order 或普通模式匹配
    ↓
【匹配结果】
```

**关键理解**：严格顺序模式的"中间事件"概念，**只针对已经经过过滤的事件**，而不是原始数据库中的所有事件。

**事件过滤前提的详细分析**：

**第一步：事件名称过滤的具体实现**

从 `goal_condition/3` 函数可以看到不同类型 Goal 的匹配条件：

```elixir
def goal_condition(goal, imported? \\ false) do
  type = Plausible.Goal.type(goal)
  goal_condition(type, goal, imported?)
end

# 自定义事件 Goal
defp goal_condition(:event, goal, _) do
  name_condition = dynamic([e], e.name == ^goal.event_name)
  
  if Plausible.Goal.has_custom_props?(goal) do
    custom_props_condition = build_custom_props_condition(goal.custom_props)
    dynamic([e], ^name_condition and ^custom_props_condition)
  else
    name_condition
  end
end

# 页面访问 Goal
defp goal_condition(:page, goal, false = _imported?) do
  name_condition = dynamic([e], e.name == "pageview")
  pathname_condition = page_path_condition(goal.page_path, _imported? = false)
  base_condition = dynamic([e], ^pathname_condition and ^name_condition)
  
  if Plausible.Goal.has_custom_props?(goal) do
    custom_props_condition = build_custom_props_condition(goal.custom_props)
    dynamic([e], ^base_condition and ^custom_props_condition)
  else
    base_condition
  end
end

# 滚动深度 Goal
defp goal_condition(:scroll, goal, false = _imported?) do
  pathname_condition = page_path_condition(goal.page_path, _imported? = false)
  name_condition = dynamic([e], e.name == "engagement")
  
  scroll_condition =
    dynamic([e], e.scroll_depth <= 100 and e.scroll_depth >= ^goal.scroll_threshold)
  
  base_condition = dynamic([e], ^pathname_condition and ^name_condition and ^scroll_condition)
  
  if Plausible.Goal.has_custom_props?(goal) do
    custom_props_condition = build_custom_props_condition(goal.custom_props)
    dynamic([e], ^base_condition and ^custom_props_condition)
  else
    base_condition
  end
end
```

**这意味着**：

1. **自定义事件 Goal**：只匹配 `name == goal.event_name` 的事件（可能还需要自定义属性匹配）
2. **页面访问 Goal**：只匹配 `name == "pageview"` 且路径匹配的事件
3. **滚动深度 Goal**：只匹配 `name == "engagement"` 且滚动深度达标的事件

**第二步：过滤后事件序列的构成**

假设我们有一个 Funnel：
- 步骤 1：Goal A（页面访问 `/product`）
- 步骤 2：Goal B（自定义事件 `AddToCart`）

**门禁条件过滤后的事件序列只包含**：
- `name == "pageview"` 且 `pathname == "/product"` 的事件（步骤 1 的匹配事件）
- `name == "AddToCart"` 的事件（步骤 2 的匹配事件）

**不会包含**：
- 其他页面的 pageview 事件
- 其他自定义事件（如 `Purchase`、`Signup`）
- engagement 事件

**第三步：严格顺序模式在过滤后序列中的行为**

现在让我们重新理解严格顺序模式的"中间事件"概念。

**示例场景**：

Funnel 定义（2 步，严格顺序模式）：
- 步骤 1：浏览产品页面（Goal：页面访问 `/product`）
- 步骤 2：添加购物车（Goal：自定义事件 `AddToCart`）

**用户实际行为序列（原始数据库）**：
1. 10:00 - 访问首页（pageview, `/`）
2. 10:01 - 访问产品页（pageview, `/product`）→ **步骤 1 匹配**
3. 10:02 - 访问帮助页（pageview, `/help`）
4. 10:03 - 点击购买按钮（自定义事件 `PurchaseClick`）
5. 10:04 - 添加购物车（自定义事件 `AddToCart`）→ **步骤 2 匹配**

**经过门禁条件过滤后的事件序列**：
只保留：
1. 10:01 - 访问产品页（pageview, `/product`）→ **步骤 1 匹配**
2. 10:04 - 添加购物车（自定义事件 `AddToCart`）→ **步骤 2 匹配**

过滤掉了：
- 访问首页（不是步骤 1 的目标路径）
- 访问帮助页（不是任何步骤的 Goal）
- 点击购买按钮（不是步骤 2 的事件名）

**严格顺序模式的判定**：

在**过滤后的事件序列**中：
- 事件 1（步骤 1）之后直接是事件 2（步骤 2）
- **没有其他事件**！

**结果**：用户被判定为到达步骤 2。

**关键洞察**：

**在非严格顺序模式下结果相同，但原因不同**：

| 模式 | 过滤前序列 | 过滤后序列 | 判定结果 | 原因 |
|------|-----------|-----------|---------|------|
| 非严格顺序 | 5 个事件 | 2 个事件 | 步骤 2 | 中间事件被忽略 |
| 严格顺序 | 5 个事件 | 2 个事件 | 步骤 2 | 过滤后没有中间事件 |

**但如果 Funnel 包含多个同类型的 Goal**：

让我们看一个更复杂的例子：

Funnel 定义（3 步，严格顺序模式）：
- 步骤 1：访问产品页 A（Goal：页面访问 `/product/A`）
- 步骤 2：访问产品页 B（Goal：页面访问 `/product/B`）
- 步骤 3：完成购买（Goal：自定义事件 `Purchase`）

**用户实际行为序列**：
1. 10:00 - 访问产品页 A（pageview, `/product/A`）→ **步骤 1 匹配**
2. 10:01 - 访问产品页 C（pageview, `/product/C`）→ **不是任何步骤的 Goal**
3. 10:02 - 访问产品页 B（pageview, `/product/B`）→ **步骤 2 匹配**
4. 10:03 - 完成购买（自定义事件 `Purchase`）→ **步骤 3 匹配**

**门禁条件过滤后的事件序列**：

注意：所有 3 个步骤都是 Goal，且：
- 步骤 1 和 2 都是"页面访问"类型，事件名都是 `"pageview"`
- 步骤 3 是"自定义事件"类型，事件名是 `"Purchase"`

**事件名称过滤**：
```elixir
goal_event_names([步骤1, 步骤2, 步骤3]) = ["pageview", "Purchase"]
```

**过滤后的事件序列**：
1. 10:00 - 访问产品页 A（pageview, `/product/A`）→ **步骤 1 匹配**
2. 10:01 - 访问产品页 C（pageview, `/product/C`）→ **事件名匹配，但不是任何步骤的路径**
3. 10:02 - 访问产品页 B（pageview, `/product/B`）→ **步骤 2 匹配**
4. 10:03 - 完成购买（自定义事件 `Purchase`）→ **步骤 3 匹配**

**关键点**：访问产品页 C 的事件**没有被过滤掉**！因为：
- 事件名是 `"pageview"`，在 `goal_event_names` 列表中
- 门禁条件的事件名称过滤只看 `e.name`，不看具体的路径匹配

**现在来看严格顺序模式的判定**：

在 `windowFunnel` 函数内部：
- 它会遍历**过滤后的事件序列**（4 个事件）
- 但只有在**步骤条件匹配**时才会推进步骤计数

让我们详细分析：

**事件序列（过滤后）**：
1. 10:00 - pageview, `/product/A`
2. 10:01 - pageview, `/product/C`
3. 10:02 - pageview, `/product/B`
4. 10:03 - Purchase

**严格顺序模式的匹配过程**：

1. **处理事件 1**（pageview, `/product/A`）：
   - 检查步骤 1 条件：`pathname == "/product/A"` ✓
   - 步骤计数增加到 1
   - 等待下一个步骤条件匹配

2. **处理事件 2**（pageview, `/product/C`）：
   - 检查当前期望的步骤 2 条件：`pathname == "/product/B"` ✗
   - **这是一个"中间事件"**！（事件在过滤后的序列中，但不匹配当前期望的步骤条件）
   - **严格顺序模式下，匹配中断**！
   - 步骤计数保持为 1（不再推进）

3. **处理事件 3**（pageview, `/product/B`）：
   - 匹配已经中断，不再检查
   - 步骤计数仍为 1

4. **处理事件 4**（Purchase）：
   - 匹配已经中断，不再检查
   - 步骤计数仍为 1

**最终结果**：用户只到达步骤 1！

**对比非严格顺序模式**：

在非严格顺序模式下：
- 事件 2 不会导致匹配中断
- 事件 3 会匹配步骤 2
- 事件 4 会匹配步骤 3
- **最终结果**：用户到达步骤 3

**两种模式的本质差异**：

| 模式 | 对"中间事件"的定义 | 处理方式 |
|------|-------------------|---------|
| 非严格顺序 | 只关心步骤条件的时间顺序 | 中间事件被忽略，继续匹配后续步骤 |
| 严格顺序 | 任何不匹配当前期望步骤的事件 | 一旦出现，匹配立即中断 |

**这里的"中间事件"是指**：
在过滤后的事件序列中，出现在步骤 N 和步骤 N+1 之间的、**不匹配步骤 N+1 条件**的事件。

**对结果准确性的影响**：

**严格顺序模式可能导致的"误判"场景**：

**场景 1：同类型 Goal 之间的其他同类事件**

如上面的例子，用户在产品页 A 和产品页 B 之间访问了产品页 C。

- **非严格顺序**：正确判定为到达步骤 3
- **严格顺序**：错误地判定为只到达步骤 1

**问题**：产品页 C 的访问是一个合理的用户行为，不应该被视为"中断"。

**场景 2：页面访问 Goal 之间的页面浏览**

Funnel：浏览产品页 → 查看购物车 → 完成购买

用户行为：
1. 浏览产品页 A（步骤 1）
2. 浏览产品页 B（同类型，不是步骤）
3. 查看购物车（步骤 2）
4. 完成购买（步骤 3）

- **非严格顺序**：正确判定为到达步骤 3
- **严格顺序**：错误地判定为只到达步骤 1

**场景 3：自定义事件 Goal 之间的其他自定义事件**

Funnel：添加购物车 → 开始结账 → 完成购买

用户行为：
1. 添加购物车（步骤 1）
2. 应用优惠券（自定义事件，不是步骤）
3. 开始结账（步骤 2）
4. 完成购买（步骤 3）

- **非严格顺序**：正确判定为到达步骤 3
- **严格顺序**：错误地判定为只到达步骤 1

**严格顺序模式的正确使用场景**：

**场景：多步骤表单提交**

Funnel：表单步骤 1 → 表单步骤 2 → 表单步骤 3

用户行为：
1. 提交表单步骤 1（步骤 1）
2. 提交表单步骤 2（步骤 2）
3. 提交表单步骤 3（步骤 3）

- **两种模式**：都正确判定为到达步骤 3

**但如果用户回退**：
1. 提交表单步骤 1（步骤 1）
2. 回退到表单步骤 1（重新提交，步骤 1）
3. 提交表单步骤 2（步骤 2）

- **非严格顺序**：到达步骤 2
- **严格顺序**：到达步骤 2（步骤 1 的重复匹配是否会中断？取决于具体实现）

**严格顺序模式的设计意图**：

根据 ClickHouse 源代码注释：
> "When the 'strict_order' is set, it doesn't allow interventions of other events. In the case of 'A->B->D->C', it stops finding 'A->B->C' at the 'D' and the max event level is 2."

**设计意图**：确保事件序列**完全按照指定顺序连续发生**，没有任何"干扰"事件。

**但这与大多数业务场景的期望不符**：
- 用户在购买过程中浏览其他产品是正常行为
- 用户在结账过程中应用优惠券是正常行为
- 这些不应该被视为"中断"

**对结果准确性的影响总结**：

| 因素 | 非严格顺序模式 | 严格顺序模式 |
|------|--------------|-------------|
| 同类型 Goal 之间的同类事件 | 正确匹配 | 可能误判为中断 |
| 不同类型 Goal 之间的其他事件 | 正确匹配 | 正确（这些事件已被门禁过滤） |
| 业务期望的用户行为 | 符合 | 可能不符合 |
| 适用场景 | 大多数业务场景 | 极少数严格引导式流程 |

**关键结论**：

1. **门禁条件过滤是严格顺序判定的前提**：只有经过过滤的事件才会进入 `windowFunnel` 匹配
2. **事件名称过滤的粒度较粗**：只看 `e.name`，不看具体的路径或自定义属性
3. **严格顺序模式的"中间事件"概念容易被误解**：它指的是过滤后序列中不匹配当前步骤条件的事件
4. **严格顺序模式可能导致误判**：特别是在同类型 Goal 之间有其他同类事件时
5. **非严格顺序模式更符合大多数业务场景**：用户在步骤之间的其他正常行为不应该被视为"中断"

### 3.4 步骤归因深度解析

步骤归因是 Funnel 分析的核心，决定了用户行为如何被映射到 Funnel 步骤中。本节将深入解析其工作原理。

#### 3.4.1 ClickHouse windowFunnel 函数工作原理

系统完全依赖 ClickHouse 的 `windowFunnel` 函数来实现多步分析。该函数的工作原理如下：

**基本算法**：
1. **窗口启动**：搜索触发条件链中第一个条件的数据，将事件计数器设置为 1。这是滑动窗口开始的时刻。
2. **连续匹配**：如果链中的事件在窗口内连续发生，则计数器增加。如果事件序列被打断，则计数器不增加。
3. **最长链选择**：如果数据在不同完成点有多个事件链，函数只会输出最长链的长度。

**核心参数**：
- **窗口大小**（window）：分析的时间范围，单位由 timestamp 决定
- **时间戳列**（timestamp）：用于确定事件顺序
- **条件列表**（conditions）：每个步骤的匹配条件

#### 3.4.2 步骤匹配机制详解

**条件构建过程**：
```elixir
window_funnel_steps =
  Enum.reduce(funnel_definition.steps, nil, fn step, acc ->
    goal_condition = Plausible.Stats.Goals.goal_condition(step.goal)
    
    if acc do
      dynamic([q], fragment("?, ?", ^acc, ^goal_condition))
    else
      dynamic([q], fragment("?", ^goal_condition))
    end
  end)
```

这段代码将 Funnel 的多个步骤条件组合成一个逗号分隔的列表，传递给 `windowFunnel` 函数。

**匹配逻辑**：
1. **按用户分组**：首先按 `user_id` 对事件进行分组
2. **按时间排序**：在每个用户的事件序列中，按时间戳排序
3. **滑动窗口匹配**：
   - 从第一个匹配步骤 1 的事件开始，启动时间窗口
   - 在窗口内搜索后续步骤的匹配事件
   - 记录用户在窗口内到达的最深步骤

**原始查询结果格式**：
ClickHouse 返回的是 `{step_index, visitor_count}` 元组，其中：
- `step_index`：用户到达的最深步骤（从 0 开始，0 表示未进入漏斗）
- `visitor_count`：到达该步骤的用户数量

#### 3.4.3 严格顺序 vs 非严格顺序的判定差异

这是 Funnel 分析中最关键的配置之一，直接影响用户行为的判定结果。

**非严格顺序模式（默认）**：

```elixir
fragment("windowFunnel(?)(timestamp, ?)", @funnel_window_duration, ^window_funnel_steps)
```

**行为特征**：
- **允许中间事件**：在步骤之间可以有其他事件发生
- **只关注顺序**：只关心步骤 A 是否在步骤 B 之前发生，不关心中间是否有其他事件
- **回溯匹配**：可以在窗口内回溯，找到符合条件的事件序列

**示例场景**：
假设有一个 3 步 Funnel：浏览产品 → 添加购物车 → 完成购买

用户 A 的事件序列：
1. 浏览产品（步骤 1）
2. 浏览首页（其他事件）
3. 添加购物车（步骤 2）
4. 查看帮助（其他事件）
5. 完成购买（步骤 3）

**非严格顺序判定结果**：用户 A 到达了步骤 3，因为：
- 步骤 1 在步骤 2 之前 ✓
- 步骤 2 在步骤 3 之前 ✓
- 中间的其他事件被忽略

**严格顺序模式**：

```elixir
fragment(
  "windowFunnel(?, 'strict_order')(timestamp, ?)",
  @funnel_window_duration,
  ^window_funnel_steps
)
```

**行为特征**：
- **禁止中间事件**：步骤之间不能有其他事件
- **严格连续性**：事件必须严格按照步骤顺序连续发生
- **中断即停止**：如果出现非预期事件，匹配过程立即停止

**核心实现原理**：
根据 ClickHouse 源代码注释，`strict_order` 模式的实现逻辑是：
> "When the 'strict_order' is set, it doesn't allow interventions of other events. In the case of 'A->B->D->C', it stops finding 'A->B->C' at the 'D' and the max event level is 2."

**示例场景**：
同样的用户 A 事件序列：
1. 浏览产品（步骤 1）
2. 浏览首页（其他事件）→