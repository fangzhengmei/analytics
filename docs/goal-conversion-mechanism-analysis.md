# Plausible Analytics 目标转化功能实现机制分析

## 1. 概述

本文档深入分析了 Plausible Analytics 中目标转化（Goal Conversion）功能的实现机制，涵盖以下三个核心方面：

1. **目标的定义方式与数据结构**
2. **过滤查询的跨层协作机制**
3. **转化率计算与统计结果组装环节**

---

## 2. 目标的定义方式与数据结构

### 2.1 核心数据模型

目标的核心定义位于 `lib/plausible/goal.ex`，使用 Ecto Schema 定义数据库模型。

#### 2.1.1 Schema 定义

```elixir
schema "goals" do
  field :event_name, :string        # 事件目标的事件名称
  field :page_path, :string         # 页面目标/滚动目标的页面路径
  field :scroll_threshold, :integer, default: -1  # 滚动目标的阈值（百分比）
  field :display_name, :string      # 显示名称
  field :currency, Ecto.Enum, ...   # 货币类型（仅企业版）
  field :custom_props, :map, default: %{}  # 自定义属性
  
  belongs_to :site, Plausible.Site  # 所属站点
  many_to_many :funnels, Plausible.Funnel, ...  # 漏斗关联（仅企业版）
  
  timestamps()
end
```
[goal.ex:8-27](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/goal.ex#L8-L27)

#### 2.1.2 三种目标类型

目标类型通过 `type/1` 函数动态判断，基于字段的存在性：

| 目标类型 | 判断条件 | 说明 |
|---------|---------|------|
| `:event` | `is_binary(goal.event_name)` | 事件目标，跟踪特定事件 |
| `:scroll` | `is_binary(goal.page_path)` 且 `scroll_threshold > -1` | 滚动目标，跟踪页面滚动深度 |
| `:page` | `is_binary(goal.page_path)` 且 `scroll_threshold == -1` | 页面访问目标，跟踪特定页面访问 |

```elixir
@spec type(t()) :: :event | :scroll | :page
def type(goal) do
  cond do
    is_binary(goal.event_name) -> :event
    is_binary(goal.page_path) && goal.scroll_threshold > -1 -> :scroll
    is_binary(goal.page_path) -> :page
  end
end
```
[goal.ex:83-89](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/goal.ex#L83-L89)

#### 2.1.3 自定义属性支持

目标支持最多 3 个自定义属性用于更精细的过滤：

```elixir
@max_custom_props_per_goal 3

defp validate_custom_props(:custom_props, custom_props) when is_map(custom_props) do
  cond do
    map_size(custom_props) > @max_custom_props_per_goal ->
      [custom_props: "use at most #{@max_custom_props_per_goal} properties per goal"]
    # ... 更多验证
  end
end
```
[goal.ex:198-225](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/goal.ex#L198-L225)

#### 2.1.4 数据库约束

目标定义包含以下唯一约束：

1. **display_name 唯一**：同一站点内显示名称唯一
2. **event_name 唯一**：事件目标的事件名称唯一
3. **page_path + scroll_threshold 唯一**：页面/滚动目标的组合唯一

```elixir
|> unique_constraint(:display_name, name: :goals_display_name_unique)
|> unique_constraint(:event_name, name: :goals_event_config_unique)
|> unique_constraint([:page_path, :scroll_threshold],
  name: :goals_pageview_config_unique
)
```
[goal.ex:58-62](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/goal.ex#L58-L62)

---

## 3. 过滤查询的跨层协作机制

### 3.1 整体架构

目标过滤查询涉及多个层级的协作：

```
┌─────────────────────────────────────────────────────────────┐
│                    QueryBuilder (参数层)                      │
│  - 解析查询参数                                               │
│  - 预加载目标数据                                             │
│  - 构建 Query 结构体                                          │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                    SQL.QueryBuilder (查询层)                  │
│  - 构建 SQL 查询                                              │
│  - 处理 GROUP BY 目标维度                                     │
│  - 合并特殊指标（如转化率）                                   │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                    SQL.WhereBuilder (条件层)                  │
│  - 解析过滤器                                                 │
│  - 调用 Goals.add_filter 处理目标过滤                        │
│  - 构建 Ecto dynamic 条件表达式                               │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                    Stats.Goals (目标匹配层)                   │
│  - 预加载目标数据                                             │
│  - 匹配过滤条件与目标                                         │
│  - 构建目标匹配的 SQL 条件                                    │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 目标预加载机制

在查询构建的早期阶段，系统会根据需要预加载目标数据：

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
[goals.ex:14-34](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/goals.ex#L14-L34)

**预加载触发条件**：
- 维度包含 `"event:goal"`
- 过滤器包含 `"event:goal"` 维度

### 3.3 目标过滤的特殊处理

与其他维度不同，`event:goal` 过滤器有特殊的处理流程：

#### 3.3.1 过滤器分发

在 `WhereBuilder` 中，`event:goal` 过滤器被特殊处理：

```elixir
defp add_filter(:events, query, [_, "event:goal" | _rest] = filter) do
  Plausible.Stats.Goals.add_filter(query, filter)
end
```
[where_builder.ex:160-162](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/sql/where_builder.ex#L160-L162)

#### 3.3.2 目标过滤的工作原理

目标过滤的核心逻辑在 `Goals.add_filter/3`：

```elixir
def add_filter(query, [operation, "event:goal", clauses | _] = filter, opts \\ [])
    when operation in [:is, :contains] do
  imported? = Keyword.get(opts, :imported?, false)

  Enum.reduce(clauses, false, fn clause, dynamic_statement ->
    condition =
      query.preloaded_goals.all
      |> filter_preloaded(filter, clause)
      |> build_condition(imported?)

    dynamic([e], ^condition or ^dynamic_statement)
  end)
end
```
[goals.ex:52-64](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/goals.ex#L52-L64)

**关键步骤**：
1. 从预加载的目标列表中筛选匹配的目标
2. 为每个匹配的目标构建 SQL 条件
3. 使用 `OR` 连接多个条件

#### 3.3.3 目标匹配逻辑

目标通过名称匹配过滤条件：

```elixir
defp matches?(goal, [operation | _rest] = filter, clause) do
  goal_name =
    goal
    |> Plausible.Goal.display_name()
    |> mod(filter)

  clause = mod(clause, filter)

  case operation do
    :is ->
      goal_name == clause

    :contains ->
      String.contains?(goal_name, clause)
  end
end
```
[goals.ex:163-178](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/goals.ex#L163-L178)

### 3.4 目标条件构建

根据目标类型的不同，构建不同的 SQL 条件：

#### 3.4.1 事件目标条件

```elixir
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
[goals.ex:202-211](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/goals.ex#L202-L211)

**条件**：`e.name == event_name` [AND 自定义属性条件]

#### 3.4.2 页面目标条件

```elixir
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
```
[goals.ex:234-245](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/goals.ex#L234-L245)

**条件**：`e.name == "pageview"` AND `pathname == page_path` [AND 自定义属性条件]

#### 3.4.3 滚动目标条件

```elixir
defp goal_condition(:scroll, goal, false = _imported?) do
  pathname_condition = page_path_condition(goal.page_path, _imported? = false)
  name_condition = dynamic([e], e.name == "engagement")

  scroll_condition =
    dynamic([e], e.scroll_depth <= 100 and e.scroll_depth >= ^goal.scroll_threshold)

  base_condition = dynamic([e], ^pathname_condition and ^name_condition and ^scroll_condition)
  # ... 自定义属性处理
end
```
[goals.ex:213-228](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/goals.ex#L213-L228)

**条件**：`e.name == "engagement"` AND `pathname == page_path` AND `scroll_depth >= threshold`

#### 3.4.4 自定义属性条件

```elixir
defp build_custom_props_condition(custom_props) do
  Enum.reduce(custom_props, true, fn {prop_key, prop_value}, acc ->
    condition =
      dynamic(
        [e],
        fragment(
          "?[indexOf(?, ?)] = ?",
          field(e, :"meta.value"),
          field(e, :"meta.key"),
          ^prop_key,
          ^prop_value
        )
      )

    dynamic([e], ^acc and ^condition)
  end)
end
```
[goals.ex:257-273](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/goals.ex#L257-L273)

### 3.5 按目标维度分组

当查询维度包含 `"event:goal"` 时，需要特殊的 GROUP BY 处理：

```elixir
defp dimension_group_by(q, :events, query, "event:goal" = dimension) do
  goal_join_data = Plausible.Stats.Goals.goal_join_data(query)

  if Enum.all?(goal_join_data.custom_props_keys, &Enum.empty?/1) do
    from(e in q,
      join: goal in Expression.event_goal_join_no_props(goal_join_data),
      hints: "ARRAY",
      on: true,
      select_merge: %{
        ^shortname(query, dimension) => fragment("?", goal)
      },
      group_by: goal
    )
  else
    from(e in q,
      join: goal in Expression.event_goal_join(goal_join_data),
      hints: "ARRAY",
      on: true,
      select_merge: %{
        ^shortname(query, dimension) => fragment("?", goal)
      },
      group_by: goal
    )
  end
end
```
[sql/query_builder.ex:173-197](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/sql/query_builder.ex#L173-L197)

**目标分组数据结构**：

```elixir
@type goal_join_data() :: %{
        indices: [non_neg_integer()],
        types: [String.t()],
        event_names_imports: [String.t()],
        event_names_by_type: [String.t()],
        page_regexes: [String.t()],
        scroll_thresholds: [non_neg_integer()],
        custom_props_keys: [[String.t()]],
        custom_props_values: [[String.t()]]
      }
```
[goals.ex:66-75](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/goals.ex#L66-L75)

---

## 4. 转化率计算与统计结果组装

### 4.1 特殊指标模块

转化率作为"特殊指标"在 `SpecialMetrics` 模块中处理：

```elixir
@special_metrics [
  :percentage,
  :conversion_rate,
  :group_conversion_rate,
  :scroll_depth,
  :exit_rate
]
```
[special_metrics.ex:14-20](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/sql/special_metrics.ex#L14-L20)

### 4.2 转化率计算逻辑

#### 4.2.1 总体转化率 (`:conversion_rate`)

```elixir
defp add_special_metric(q, :conversion_rate, site, query) do
  total_query =
    query
    |> Query.remove_top_level_filters(["event:goal", "event:props"])
    |> remove_filters_ignored_in_totals_query()
    |> Query.set(
      dimensions: [],
      include_imported: query.include_imported,
      preloaded_goals: Map.put(query.preloaded_goals, :matching_toplevel_filters, []),
      pagination: nil
    )

  q
    |> select_merge_as(
      [],
      total_visitors_subquery(site, total_query, query.include_imported)
    )
    |> select_merge_as([e], %{
      conversion_rate:
        fragment(
          "if(? > 0, round(? / ? * 100, 2), 0)",
          selected_as(:total_visitors),
          selected_as(:visitors),
          selected_as(:total_visitors)
        )
    })
end
```
[special_metrics.ex:58-84](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/sql/special_metrics.ex#L58-L84)

**计算公式**：
```
conversion_rate = (visitors_with_goal / total_visitors) * 100
```

**关键步骤**：
1. **构建总访客查询**：移除 `event:goal` 和 `event:props` 过滤器
2. **计算总访客数**：作为分母
3. **计算转化率**：使用 SQL 片段 `round(visitors / total_visitors * 100, 2)`

#### 4.2.2 分组转化率 (`:group_conversion_rate`)

当查询包含维度（如按页面、来源等分组）时，使用分组转化率：

```elixir
defp add_special_metric(q, :group_conversion_rate, site, query) do
  group_totals_query =
    query
      |> Query.remove_top_level_filters(["event:goal", "event:props"])
      |> remove_filters_ignored_in_totals_query()
      |> Query.set(
        metrics: [:visitors],
        order_by: [],
        include_imported: query.include_imported,
        preloaded_goals: Map.put(query.preloaded_goals, :matching_toplevel_filters, []),
        pagination: nil
      )

  from(e in subquery(q),
    left_join: c in subquery(SQL.QueryBuilder.build(group_totals_query, site)),
    on: ^SQL.QueryBuilder.build_group_by_join(query)
  )
    |> select_merge_as([e, c], %{
      total_visitors: c.visitors,
      group_conversion_rate:
        fragment(
          "if(? > 0, round(? / ? * 100, 2), 0)",
          c.visitors,
          e.visitors,
          c.visitors
        )
    })
    # ...
end
```
[special_metrics.ex:97-126](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/sql/special_metrics.ex#L97-L126)

**关键差异**：
- 使用 `LEFT JOIN` 连接分组的总访客数
- 按维度匹配计算各组的转化率

### 4.3 总访客子查询

总访客数的计算考虑了导入数据：

```elixir
defp total_visitors_subquery(site, query, true = _include_imported) do
  wrap_alias([], %{
    total_visitors:
      subquery(total_visitors(query)) +
        subquery(Plausible.Stats.Imported.total_imported_visitors(site, query))
  })
end

defp total_visitors_subquery(_site, query, false = _include_imported) do
  wrap_alias([], %{
    total_visitors: subquery(total_visitors(query))
  })
end

defp total_visitors(query) do
  Base.base_event_query(query)
    |> select([e],
      total_visitors: scale_sample(fragment("uniq(?)", e.user_id))
    )
end
```
[special_metrics.ex:263-294](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/sql/special_metrics.ex#L263-L294)

### 4.4 查询执行流程

完整的查询执行流程如下：

```
┌────────────────────────────────────────────────────────────────┐
│                     QueryRunner.run/2                           │
│  入口函数，协调整个查询执行过程                                   │
└────────────────────────────┬───────────────────────────────────┘
                             │
┌────────────────────────────▼───────────────────────────────────┐
│              execute_main_query/1                                │
│  - 调用 SQL.QueryBuilder.build/2 构建查询                        │
│  - 执行 ClickHouse 查询                                          │
│  - 转换查询结果                                                   │
└────────────────────────────┬───────────────────────────────────┘
                             │
┌────────────────────────────▼───────────────────────────────────┐
│              SQL.QueryBuilder.build/2                            │
│  - 优化查询（拆分表查询）                                         │
│  - 为每个表类型构建查询                                           │
│  - 合并结果                                                       │
│  - 添加排序和分页                                                 │
└────────────────────────────┬───────────────────────────────────┘
                             │
┌────────────────────────────▼───────────────────────────────────┐
│              build_table_query/3                                 │
│  - 构建基础查询（WHERE 条件）                                     │
│  - 连接会话表（如需要）                                           │
│  - 构建 GROUP BY                                                 │
│  - 合并导入数据                                                   │
│  - 添加特殊指标（转化率等）                                       │
└────────────────────────────┬───────────────────────────────────┘
                             │
┌────────────────────────────▼───────────────────────────────────┐
│              SpecialMetrics.add/3                                │
│  - 检查是否需要转化率等特殊指标                                   │
│  - 调用 add_special_metric/4 添加到查询                          │
└─────────────────────────────────────────────────────────────────┘
```

### 4.5 结果组装流程

#### 4.5.1 从 ClickHouse 结果转换

```elixir
defp build_from_ch(ch_results, query) do
  ch_results
    |> Enum.map(fn entry ->
      dimension_labels = Enum.map(query.dimensions, &dimension_label(&1, entry, query))

      %{
        dimensions: dimension_labels,
        metrics: Enum.map(query.metrics, &get_metric(entry, &1, dimension_labels, query))
      }
    end)
end
```
[query_runner.ex:152-162](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/query_runner.ex#L152-L162)

#### 4.5.2 目标维度标签转换

当维度是 `event:goal` 时，需要将目标索引转换为目标显示名称：

```elixir
defp dimension_label("event:goal", entry, query) do
  get_dimension_goal(entry, query)
    |> Plausible.Goal.display_name()
end

defp get_dimension_goal(entry, query) do
  goal_index = Map.get(entry, Util.shortname(query, "event:goal"))

  query.preloaded_goals.matching_toplevel_filters
    |> Enum.at(goal_index - 1)
end
```
[query_runner.ex:164-215](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/query_runner.ex#L164-L215)

#### 4.5.3 最终结果结构

`QueryResult` 结构体封装最终的查询结果：

```elixir
defstruct results: [],
          comparison_results: nil,
          meta: %{},
          query: nil
```
[query_result.ex:13-16](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/query_result.ex#L13-L16)

**结果格式示例**：
```json
{
  "results": [
    {
      "dimensions": ["Visit /checkout"],
      "metrics": [150, 10.5]  // visitors, conversion_rate
    }
  ],
  "meta": {
    "imports_included": true
  },
  "query": {
    "metrics": ["visitors", "conversion_rate"],
    "dimensions": ["event:goal"]
  }
}
```

---

## 5. 完整流程图

### 5.1 目标转化查询完整流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           用户查询请求                                      │
│  {metrics: ["visitors", "conversion_rate"], dimensions: ["event:goal"]} │
└─────────────────────────────────────┬───────────────────────────────────┘
                                      │
┌─────────────────────────────────────▼───────────────────────────────────┐
│                      QueryBuilder.build/3 (参数层)                        │
│  1. 检测到 "event:goal" 维度/过滤器                                        │
│  2. 调用 Goals.preload_needed_goals/3 预加载目标                          │
│  3. 构建 Query 结构体，包含 preloaded_goals                                │
└─────────────────────────────────────┬───────────────────────────────────┘
                                      │
┌─────────────────────────────────────▼───────────────────────────────────┐
│                      QueryRunner.run/2 (执行层)                           │
│  1. 优化查询                                                               │
│  2. 调用 SQL.QueryBuilder.build/2 构建 SQL 查询                           │
│  3. 执行 ClickHouse 查询                                                   │
│  4. 转换结果为标准格式                                                      │
└─────────────────────────────────────┬───────────────────────────────────┘
                                      │
┌─────────────────────────────────────▼───────────────────────────────────┐
│                  SQL.QueryBuilder.build/2 (SQL 层)                        │
│  1. 构建 events_v2 表查询                                                  │
│  2. 添加 WHERE 条件（调用 WhereBuilder）                                    │
│  3. 构建 GROUP BY（目标维度特殊处理）                                       │
│  4. 调用 SpecialMetrics.add 添加转化率                                      │
└─────────────────────────────────────┬───────────────────────────────────┘
                                      │
┌─────────────────────────────────────▼───────────────────────────────────┐
│                  SQL.WhereBuilder.build/2 (条件层)                         │
│  1. 遍历过滤器                                                             │
│  2. 对 "event:goal" 过滤器调用 Goals.add_filter                            │
│  3. 构建 Ecto dynamic 条件表达式                                           │
└─────────────────────────────────────┬───────────────────────────────────┘
                                      │
┌─────────────────────────────────────▼───────────────────────────────────┐
│                  Goals.add_filter/3 (目标匹配层)                           │
│  1. 从 preloaded_goals.all 筛选匹配的目标                                  │
│  2. 为每个目标调用 goal_condition/2 构建 SQL 条件                          │
│  3. 使用 OR 连接多个条件                                                   │
└─────────────────────────────────────┬───────────────────────────────────┘
                                      │
┌─────────────────────────────────────▼───────────────────────────────────┐
│                  SpecialMetrics.add/3 (转化率计算层)                       │
│  1. 检测 conversion_rate 在 metrics 中                                      │
│  2. 构建 total_query（移除目标过滤器）                                       │
│  3. 计算 total_visitors（作为分母）                                         │
│  4. 使用 SQL 片段计算转化率: visitors / total_visitors * 100              │
└─────────────────────────────────────┬───────────────────────────────────┘
                                      │
┌─────────────────────────────────────▼───────────────────────────────────┐
│                  QueryResult.from/1 (结果组装层)                           │
│  1. 从 QueryRunner 提取结果                                                 │
│  2. 构建 meta 信息（导入状态、警告等）                                       │
│  3. 构建 query 信息（原始查询参数）                                         │
│  4. 返回 JSON 可序列化的结果                                                │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## 6. 关键设计要点

### 6.1 目标过滤的特殊性

与其他维度过滤不同，`event:goal` 过滤有以下特点：

1. **预加载机制**：目标数据必须预加载到查询中
2. **名称匹配**：过滤条件通过目标的 `display_name` 匹配
3. **条件构建**：匹配的目标转换为实际的事件/页面/滚动条件
4. **多条件 OR**：多个目标条件使用 OR 连接

### 6.2 转化率计算的核心思想

转化率计算的核心是 **"对比"**：

- **分子**：满足目标条件的访客数
- **分母**：相同时间范围内的总访客数（移除目标过滤器）

这确保了转化率反映的是"在所有访客中，有多少比例完成了目标"。

### 6.3 自定义属性的支持

目标支持自定义属性，这使得：

1. **更精细的目标定义**：同一事件可以根据属性值定义不同目标
2. **灵活的过滤**：自定义属性条件与目标类型条件使用 AND 连接
3. **性能考虑**：自定义属性使用 ClickHouse 的数组索引操作

---

## 7. 相关文件索引

| 文件路径 | 功能描述 |
|---------|---------|
| `lib/plausible/goal.ex` | 目标数据模型定义、类型判断、验证 |
| `lib/plausible/stats/goals.ex` | 目标过滤、条件构建、分组数据准备 |
| `lib/plausible/stats/filters/filters.ex` | 过滤器解析、维度检测 |
| `lib/plausible/stats/sql/where_builder.ex` | SQL WHERE 条件构建、过滤器分发 |
| `lib/plausible/stats/sql/special_metrics.ex` | 转化率等特殊指标计算 |
| `lib/plausible/stats/sql/query_builder.ex` | SQL 查询构建、目标维度 GROUP BY |
| `lib/plausible/stats/query_runner.ex` | 查询执行、结果转换 |
| `lib/plausible/stats/query_result.ex` | 最终结果组装、JSON 序列化 |
| `lib/plausible/stats/query_builder.ex` | Query 结构体构建、目标预加载 |

---

## 8. 总结

Plausible Analytics 的目标转化功能通过以下核心机制实现：

1. **灵活的目标定义**：支持事件目标、页面目标、滚动目标三种类型，每种类型有不同的匹配条件。

2. **分层的过滤协作**：从参数解析到 SQL 构建，涉及多个模块的协作，其中 `event:goal` 过滤器有特殊的处理流程。

3. **独立的转化率计算**：转化率作为特殊指标，通过构建"移除目标过滤器的总访客查询"作为分母，与当前查询结果进行对比计算。

4. **高效的查询优化**：使用预加载、条件下推、ClickHouse 特定的 SQL 片段等技术确保查询性能。

这种设计既保证了功能的灵活性（支持多种目标类型、自定义属性），又保证了查询性能（利用 ClickHouse 的列式存储和数组操作）。
