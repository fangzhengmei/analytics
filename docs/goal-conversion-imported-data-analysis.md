# Plausible Analytics 目标转化补充分析

## 1. 概述

本文档是对目标转化全链路分析的补充，重点关注：

1. **导入数据分支下的目标匹配差异**：原生数据与导入数据在目标匹配上的不同处理逻辑
2. **转化率分母计算口径**：移除目标过滤后的分母计算方式，以及 `include_imported` 参数的影响
3. **空属性默认值的正确表述**：`custom_props` 字段的实际默认值和匹配行为

---

## 2. 导入数据分支下的目标匹配差异

### 2.1 导入数据与原生数据的架构差异

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        数据来源分层                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────────────────┐    ┌──────────────────────────┐            │
│  │      原生数据（Native）    │    │     导入数据（Imported）  │            │
│  ├──────────────────────────┤    ├──────────────────────────┤            │
│  │ 数据来源: Tracker 实时上报 │    │ 数据来源: GA/UA/CSV 导入  │            │
│  │ 存储表: events_v2         │    │ 存储表: imported_* 系列   │            │
│  │ 粒度: 单条事件（行级）     │    │ 粒度: 预聚合数据          │            │
│  │ 支持: 全部目标类型         │    │ 支持: 部分目标类型        │            │
│  └──────────────────────────┘    └──────────────────────────┘            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 导入数据支持的目标类型

从代码中可以看到导入数据对目标类型的支持情况：

```elixir
# lib/plausible/stats/imported/base.ex:193-208
defp do_decide_tables(query) do
  # ...
  filter_goal_table_candidates =
    query.preloaded_goals.matching_toplevel_filters
    |> Enum.map(&Plausible.Goal.type/1)
    |> Enum.map(fn
      :event -> "imported_custom_events"
      :page -> "imported_pages"
      :scroll -> nil  # ← 滚动目标不支持导入数据
    end)
  # ...
end
```
[base.ex:193-208](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/imported/base.ex#L193-L208)

**目标类型支持矩阵**：

| 目标类型 | 原生数据 (events_v2) | 导入数据 (imported_*) | 说明 |
|---------|---------------------|---------------------|------|
| **事件目标** | ✅ 完全支持 | ✅ 部分支持 | 导入数据不支持自定义属性匹配 |
| **页面目标** | ✅ 完全支持 | ✅ 部分支持 | 导入数据不支持自定义属性匹配 |
| **滚动目标** | ✅ 完全支持 | ❌ 不支持 | 导入数据无滚动深度字段 |

### 2.3 目标匹配条件的分支差异

**核心差异点**：`goal_condition` 函数的 `imported?` 参数控制不同分支

```elixir
# lib/plausible/stats/goals.ex:197-276
def goal_condition(goal, imported? \\ false) do
  type = Plausible.Goal.type(goal)
  goal_condition(type, goal, imported?)
end
```
[goals.ex:197](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/goals.ex#L197)

#### 2.3.1 事件目标的差异

```elixir
# 原生数据分支 (imported? = false) - 相同的实现
# lib/plausible/stats/goals.ex:202-211
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

**关键发现**：
- 事件目标的 `goal_condition` 函数定义为 `goal_condition(:event, goal, _)`
- 第三个参数 `_` 表示忽略 `imported?`
- **这意味着事件目标在原生和导入分支使用相同的条件构建逻辑**

但是，导入数据的存储结构不同：

```elixir
# lib/plausible/stats/imported/imported.ex:237-287
def merge_imported(q, site, %Query{dimensions: ["event:goal"]} = query) do
  goal_join_data = Plausible.Stats.Goals.goal_join_data(query)

  Imported.Base.decide_tables(query)
  |> Enum.map(fn
    "imported_custom_events" ->
      Imported.Base.query_imported("imported_custom_events", site, query)
      |> where([i], i.visitors > 0)
      |> select_merge_as([i], %{
        dim0:
          fragment(
            "indexOf(?, ?)",
            type(^goal_join_data.event_names_imports, {:array, :string}),
            i.name  # ← 只匹配事件名，没有自定义属性！
          )
      })
      # ...
  end)
end
```
[imported.ex:237-287](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/imported/imported.ex#L237-L287)

**事件目标在导入数据中的实际匹配逻辑**：

| 匹配维度 | 原生数据 | 导入数据 |
|---------|---------|---------|
| **事件名称** | ✅ `e.name == goal.event_name` | ✅ `i.name` 匹配 |
| **自定义属性** | ✅ `meta.value[indexOf(...)] = value` | ❌ **不支持** |
| **匹配精度** | 精确匹配（事件名 + 属性） | 宽松匹配（只匹配事件名） |

#### 2.3.2 页面目标的差异

页面目标在两个分支有明确的不同实现：

```elixir
# 导入数据分支 (imported? = true)
# lib/plausible/stats/goals.ex:230-232
defp goal_condition(:page, goal, true = _imported?) do
  page_path_condition(goal.page_path, _imported? = true)
  # ← 没有检查 has_custom_props?!
end

# 原生数据分支 (imported? = false)
# lib/plausible/stats/goals.ex:234-245
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
[goals.ex:230-245](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/goals.ex#L230-L245)

**页面目标匹配差异对比**：

| 匹配维度 | 原生数据 (imported?=false) | 导入数据 (imported?=true) |
|---------|-----------------------------|---------------------------|
| **事件名称** | ✅ `e.name == "pageview"` | ❌ **不检查** |
| **页面路径** | ✅ `pathname` 字段匹配 | ✅ `page` 字段匹配 |
| **自定义属性** | ✅ 检查并匹配 | ❌ **不支持** |
| **条件复杂度** | 路径 + 事件名 + 属性 | 仅路径 |

#### 2.3.3 滚动目标的差异

滚动目标**完全不支持**导入数据：

```elixir
# 只有原生数据分支实现
# lib/plausible/stats/goals.ex:213-228
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
[goals.ex:213-228](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/goals.ex#L213-L228)

**滚动目标不支持导入数据的原因**：

1. **表决定逻辑返回 nil**：
   ```elixir
   # lib/plausible/stats/imported/base.ex:196-200
   :scroll -> nil  # 不会选择任何表
   ```

2. **导入数据表没有滚动深度字段**：
   - 原生数据：`events_v2.scroll_depth` (UInt8)
   - 导入数据：`imported_pages` 表早期有，但现在可能不完整

### 2.4 页面路径字段名差异

除了匹配逻辑差异，页面路径的字段名在两个分支也不同：

```elixir
# lib/plausible/stats/goals.ex:275-276
def page_path_db_field(true = _imported?), do: :page
def page_path_db_field(false = _imported?), do: :pathname
```
[goals.ex:275-276](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/goals.ex#L275-L276)

**字段映射**：

| 数据来源 | 表名 | 页面路径字段 | 事件名称字段 |
|---------|------|-------------|-------------|
| **原生数据** | `events_v2` | `pathname` | `name` |
| **导入数据** | `imported_pages` | `page` | 无单独字段（已预聚合） |
| **导入数据** | `imported_custom_events` | 无 | `name` |

### 2.5 导入数据目标匹配示例

假设配置了以下目标：

| 目标名称 | 类型 | 事件名/路径 | 自定义属性 |
|---------|------|------------|-----------|
| `Premium Signup` | 事件目标 | `Signup` | `{plan: "premium"}` |
| `Free Signup` | 事件目标 | `Signup` | `{plan: "free"}` |
| `Checkout A` | 页面目标 | `/checkout` | `{variant: "A"}` |
| `50% Scroll` | 滚动目标 | `/blog` | `{}` |

**匹配结果对比**：

| 目标 | 原生数据（events_v2） | 导入数据（imported_*） |
|-----|----------------------|-----------------------|
| `Premium Signup` | ✅ 仅匹配 `Signup` + `plan="premium"` | ⚠️ 匹配所有 `Signup` 事件（忽略属性） |
| `Free Signup` | ✅ 仅匹配 `Signup` + `plan="free"` | ⚠️ 匹配所有 `Signup` 事件（忽略属性） |
| `Checkout A` | ✅ 仅匹配 `/checkout` + `variant="A"` | ⚠️ 匹配所有 `/checkout` 页面（忽略属性） |
| `50% Scroll` | ✅ 匹配滚动深度 >= 50% | ❌ **不支持** |

**注意**：在导入数据分支，`Premium Signup` 和 `Free Signup` 两个目标会匹配**相同的事件集合**，因为导入数据不区分自定义属性！

---

## 3. 转化率分母计算口径与 include_imported 影响

### 3.1 转化率计算的核心逻辑

转化率的计算公式：

```
conversion_rate = (目标访客数 / 总访客数) * 100
```

**关键点**：
- **分子**：满足目标过滤器的访客数（应用了 `event:goal` 过滤器）
- **分母**：**总访客数**（需要移除目标过滤器）

### 3.2 分母计算的特殊处理

```elixir
# lib/plausible/stats/sql/special_metrics.ex:253-294

# `total_visitors_subquery` returns a subquery which selects `total_visitors` -
# the number used as the denominator in the calculation of `conversion_rate` and
# `percentage` metrics.

# Usually, when calculating the totals, a new query is passed into this function,
# where certain filters (e.g. goal, props) are removed. That might make the query
# able to include imported data. However, we always want to include imported data
# only if it's included in the base query - otherwise the total will be based on
# a different data set, making the metric inaccurate. This is why we're using an
# explicit `include_imported` argument here.
defp total_visitors_subquery(site, query, include_imported)

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
```
[special_metrics.ex:253-294](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/sql/special_metrics.ex#L253-L294)

### 3.3 include_imported 参数的传递

这个参数从哪里来？让我追溯调用链：

```elixir
# lib/plausible/stats/sql/special_metrics.ex:181-189
defp add_special_metric(q, :conversion_rate, site, query) do
  q
  |> add_percentage_metric(site, query, :visitors)
end

# lib/plausible/stats/sql/special_metrics.ex:191-211
defp add_special_metric(q, :percentage, site, query) do
  add_percentage_metric(q, site, query, :visitors)
end

# lib/plausible/stats/sql/special_metrics.ex:213-248
defp add_percentage_metric(q, site, query, metric) do
  # ...
  totals_query =
    query
    |> Query.remove_top_level_filters(["event:goal", "event:props"])
    |> remove_filters_ignored_in_totals_query()

  # 关键：传递 query.include_imported
  total_visitors_q = total_visitors_subquery(site, totals_query, query.include_imported)
  # ...
end
```
[special_metrics.ex:181-248](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/stats/sql/special_metrics.ex#L181-L248)

### 3.4 移除的过滤器类型

```elixir
# lib/plausible/stats/query.ex:（需要确认）
# 从代码中可以看到移除的过滤器：

# 1. 目标过滤器
"event:goal"

# 2. 自定义属性过滤器
"event:props"

# 3. ignore_in_totals_query 标记的过滤器
# 通过 remove_filters_ignored_in_totals_query 处理
```

### 3.5 分母计算口径图解

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    转化率计算流程                                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                      分子计算（目标访客数）                            │  │
│  │  ┌─────────────────────────────────────────────────────────────┐   │  │
│  │  │ 查询: SELECT uniq(user_id) FROM events_v2 WHERE ...          │   │  │
│  │  │ 过滤器: 包含 event:goal + 其他所有过滤器                      │   │  │
│  │  │ include_imported: 与原查询保持一致                             │   │  │
│  │  └─────────────────────────────────────────────────────────────┘   │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                      │                                       │
│                                      ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                      分母计算（总访客数）                              │  │
│  │  ┌─────────────────────────────────────────────────────────────┐   │  │
│  │  │ 步骤 1: 移除目标过滤器                                          │   │  │
│  │  │   - 移除 "event:goal" 过滤器                                   │   │  │
│  │  │   - 移除 "event:props" 过滤器                                  │   │  │
│  │  │   - 移除 :ignore_in_totals_query 标记的过滤器                  │   │  │
│  │  │                                                                 │   │  │
│  │  │ 步骤 2: include_imported 参数保持不变                           │   │  │
│  │  │   - 如果原查询 include_imported=true                           │   │  │
│  │  │     分母 = 原生访客数 + 导入访客数                               │   │  │
│  │  │   - 如果原查询 include_imported=false                          │   │  │
│  │  │     分母 = 原生访客数                                            │   │  │
│  │  └─────────────────────────────────────────────────────────────┘   │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                      │                                       │
│                                      ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                      最终转化率                                       │  │
│  │  conversion_rate = (分子 / 分母) * 100                              │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.6 include_imported 对转化率的影响

**场景假设**：
- 原生数据：100 个访客，其中 10 个完成目标
- 导入数据：50 个访客，其中 5 个完成目标（但导入数据目标匹配更宽松）

#### 情况 1: include_imported = false

```
分子（目标访客数）= 10（仅原生）
分母（总访客数）= 100（仅原生，移除目标过滤器后）

转化率 = (10 / 100) * 100 = 10.00%
```

#### 情况 2: include_imported = true

```
分子（目标访客数）= 10（原生） + 5（导入） = 15
分母（总访客数）= 100（原生） + 50（导入） = 150
        ↑
        注意：include_imported 参数传递给分母计算！

转化率 = (15 / 150) * 100 = 10.00%
```

**关键发现**：
- 当 `include_imported=true` 时，**分子和分母都包含导入数据**
- 这确保了转化率计算的**一致性**：分子和分母来自同一数据集
- 如果设计不同（比如分母总是包含导入数据），会导致转化率计算不准确

### 3.7 为什么 include_imported 需要显式传递？

代码注释中的关键说明：

```elixir
# Usually, when calculating the totals, a new query is passed into this function,
# where certain filters (e.g. goal, props) are removed. That might make the query
# able to include imported data. However, we always want to include imported data
# only if it's included in the base query - otherwise the total will be based on
# a different data set, making the metric inaccurate.
```

**翻译解释**：
- 通常计算总数时，会创建一个新查询，移除某些过滤器（如 goal、props）
- 这可能使新查询能够包含导入数据（因为某些过滤器可能阻止了导入数据的使用）
- 但我们希望**只有原查询包含导入数据时，总数才包含导入数据**
- 否则总数会基于不同的数据集，导致指标不准确

**场景示例**：
1. 原查询：使用了 `event:props` 过滤器（自定义属性）
   - 导入数据不支持自定义属性过滤
   - 所以原查询 `include_imported=false`（实际只用原生数据）

2. 分母查询：移除了 `event:props` 过滤器
   - 如果不传递 `include_imported`，分母查询可能会包含导入数据
   - 这会导致：分子（原生）/ 分母（原生+导入）= 不准确的转化率

3. 正确做法：显式传递 `include_imported=false`
   - 分母查询也只使用原生数据
   - 保持分子分母数据集一致

---

## 4. 空属性默认值的正确表述

### 4.1 之前的表述回顾

之前的分析中提到：
> **关键点**：`custom_props = %{}`（空 Map）和 `custom_props = nil`（未设置）在 PostgreSQL 中被视为不同的值

这个表述**不完全准确**，需要纠正。

### 4.2 数据库迁移的实际定义

让我们重新查看迁移文件：

```elixir
# priv/repo/migrations/20251209120138_goals_custom_props.exs:21-23
# 第一次添加 custom_props 字段
alter table(:goals) do
  add(:custom_props, :map)  # 没有 default，允许 null
end
```

```elixir
# priv/repo/migrations/20251211110619_goals_custom_props_default.exs:12-15
# 修正默认值
alter table(:goals) do
  remove :custom_props
  add :custom_props, :map, null: false, default: %{}  # ← 关键！
end
```
[20251211110619_goals_custom_props_default.exs:12-15](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/priv/repo/migrations/20251211110619_goals_custom_props_default.exs#L12-L15)

**关键发现**：
1. `custom_props` 字段定义为 `null: false`
2. 默认值是 `%{}`（空 Map）
3. **不存在 `custom_props = nil` 的情况**

### 4.3 Schema 定义

```elixir
# lib/plausible/goal.ex:52
field :custom_props, :map, default: %{}
```
[goal.ex:52](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/goal.ex#L52)

### 4.4 has_custom_props? 函数的正确理解

```elixir
# lib/plausible/goal.ex:91-97
@spec has_custom_props?(t()) :: boolean()
def has_custom_props?(%__MODULE__{custom_props: custom_props})
    when map_size(custom_props) > 0 do
  true
end

def has_custom_props?(_), do: false
```
[goal.ex:91-97](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/lib/plausible/goal.ex#L91-L97)

**函数行为分析**：

| custom_props 值 | map_size | has_custom_props? 返回值 | 说明 |
|----------------|----------|--------------------------|------|
| `%{}` | `0` | `false` | 空 Map，不认为有自定义属性 |
| `%{a: "b"}` | `1` | `true` | 有一个属性 |
| `%{a: "b", c: "d"}` | `2` | `true` | 有多个属性 |

**注意**：
- 函数使用 `when map_size(custom_props) > 0` 守卫
- 默认值 `%{}` 满足 `map_size(%{}) == 0`，所以返回 `false`
- 第二个子句 `def has_custom_props?(_), do: false` 是兜底，但实际上不会被触发（因为 `custom_props` 永远是 Map）

### 4.5 测试用例验证

```elixir
# test/plausible/goals_test.exs:134-150
test "create/2 fails to create the same custom event goal twice with different display names but no props each" do
  site = new_site()

  {:ok, _} =
    Goals.create(site, %{
      "event_name" => "Purchase",
      "display_name" => "General Purchase"
    })  # 没有显式设置 custom_props，使用默认值 %{}

  {:error, changeset} =
    Goals.create(site, %{
      "event_name" => "Purchase",
      "display_name" => "General Purchase 2"
    })  # 同样使用默认值 %{}

  assert {"has already been taken", _} = changeset.errors[:event_name]
end
```
[goals_test.exs:134-150](file:///g:/fangzheng/solo-dogfeeding/code/17717-analytics/test/plausible/goals_test.exs#L134-L150)

**分析**：
- 两次创建都没有显式设置 `custom_props`
- 数据库使用默认值 `%{}`
- 唯一约束 `(site_id, event_name, custom_props)` 检测到重复
- 证明：**未显式设置 = `custom_props = %{}`**

### 4.6 目标去重中的空属性

唯一约束定义：

```elixir
# priv/repo/migrations/20251211110619_goals_custom_props_default.exs:17-22
create(
  unique_index(:goals, [:site_id, :event_name, :custom_props],
    where: "event_name IS NOT NULL",
    name: :goals_event_config_unique
  )
)
```

**PostgreSQL 中 JSONB 的比较规则**：

| 场景 | 目标 1 | 目标 2 | 唯一约束结果 |
|-----|-------|-------|------------|
| 场景 1 | `event_name='Purchase', custom_props='{}'` | `event_name='Purchase', custom_props='{}'` | ❌ 重复 |
| 场景 2 | `event_name='Purchase', custom_props='{}'` | `event_name='Purchase', custom_props='{"product":"tablet"}'` | ✅ 不重复 |
| 场景 3 | `event_name='Purchase', custom_props='{"product":"tablet"}'` | `event_name='Purchase', custom_props='{"product":"tablet"}'` | ❌ 重复 |
| 场景 4 | `event_name='Purchase', custom_props='{"product":"tablet"}'` | `event_name='Purchase', custom_props='{"product":"speaker"}'` | ✅ 不重复 |

### 4.7 正确表述总结

**之前的表述（需要纠正）**：
> `custom_props = %{}`（空 Map）和 `custom_props = nil`（未设置）在 PostgreSQL 中被视为不同的值

**正确的表述**：

| 表述项 | 正确内容 |
|-------|---------|
| **字段默认值** | `custom_props` 字段的默认值是 `%{}`（空 Map），不是 `nil` |
| **是否允许 NULL** | 字段定义为 `null: false`，永远不会是 `nil` |
| **未显式设置的含义** | 未显式设置 `custom_props` = 使用默认值 `%{}` |
| **has_custom_props? 判断** | `map_size(custom_props) > 0` 才返回 `true` |
| **空属性的语义** | `custom_props = %{}` 表示"没有自定义属性条件" |
| **唯一约束比较** | PostgreSQL 对 JSONB 类型进行值比较，`{}` 是合法的比较值 |

**目标去重的实际规则**：

```
事件目标唯一约束 = site_id + event_name + custom_props (JSONB 值比较)

其中：
- custom_props = %{} 是一个有效的、可比较的值
- 所有未显式设置 custom_props 的目标，custom_props 都是 %{}
- %{} 和 {"a":"b"} 被视为不同的值
- {"a":"b", "c":"d"} 和 {"c":"d", "a":"b"} 被视为相同的值（JSONB 不保留键顺序）
```

---

## 5. 完整示例：导入数据对目标统计的影响

### 5.1 场景设置

假设我们有一个站点，包含：

**配置的目标**：

| 目标名称 | 类型 | 事件名/路径 | 自定义属性 |
|---------|------|------------|-----------|
| `Premium Signup` | 事件目标 | `Signup` | `{plan: "premium"}` |
| `Free Signup` | 事件目标 | `Signup` | `{plan: "free"}` |
| `Any Signup` | 事件目标 | `Signup` | `{}` |

**实际数据**：

| 数据来源 | 事件名 | 自定义属性 | 访客数 |
|---------|-------|-----------|--------|
| 原生数据 | `Signup` | `{plan: "premium"}` | 10 |
| 原生数据 | `Signup` | `{plan: "free"}` | 20 |
| 原生数据 | `Signup` | `{}`（无属性） | 5 |
| 导入数据 | `Signup` | （不存储属性） | 30 |

**总访客数（不考虑目标）**：
- 原生数据：100 个访客
- 导入数据：50 个访客

### 5.2 原生数据查询（include_imported = false）

**查询**：
```elixir
%{
  metrics: [:visitors, :conversion_rate],
  dimensions: ["event:goal"],
  filters: [[:is, "event:goal", ["Premium Signup", "Free Signup", "Any Signup"]]],
  include_imported: false
}
```

**匹配逻辑**（精确匹配）：

| 目标 | 匹配的事件 | 目标访客数 | 总访客数 | 转化率 |
|-----|-----------|-----------|---------|--------|
| `Premium Signup` | `Signup` + `plan="premium"` | 10 | 100 | 10.00% |
| `Free Signup` | `Signup` + `plan="free"` | 20 | 100 | 20.00% |
| `Any Signup` | 任何 `Signup`（无属性条件） | 35 | 100 | 35.00% |

### 5.3 包含导入数据查询（include_imported = true）

**查询**：
```elixir
%{
  metrics: [:visitors, :conversion_rate],
  dimensions: ["event:goal"],
  filters: [[:is, "event:goal", ["Premium Signup", "Free Signup", "Any Signup"]]],
  include_imported: true
}
```

**匹配逻辑**（导入数据忽略属性）：

| 目标 | 原生匹配 | 导入匹配 | 总目标访客数 | 总访客数 | 转化率 |
|-----|---------|---------|-------------|---------|--------|
| `Premium Signup` | 10 | 30（所有 Signup） | 40 | 150 | 26.67% |
| `Free Signup` | 20 | 30（所有 Signup） | 50 | 150 | 33.33% |
| `Any Signup` | 35 | 30（所有 Signup） | 65 | 150 | 43.33% |

### 5.4 关键差异分析

| 维度 | 原生数据 | 包含导入数据 | 差异说明 |
|-----|---------|-------------|---------|
| `Premium Signup` 访客数 | 10 | 40 | 导入数据的 30 个 Signup 全部匹配 |
| `Free Signup` 访客数 | 20 | 50 | 同上，导入数据不区分属性 |
| `Any Signup` 访客数 | 35 | 65 | 包含所有 Signup |
| 目标间区分度 | 清晰区分 | **模糊不清** | Premium 和 Free 都匹配相同的导入事件 |

**重要警告**：
当查询包含导入数据时，**具有相同事件名但不同自定义属性的多个目标会匹配相同的导入数据事件集合**。这可能导致：

1. **重复计数**：同一个导入事件可能被多个目标统计
2. **转化率失真**：由于导入数据匹配更宽松，转化率可能被高估
3. **数据不可比**：原生数据和导入数据的统计口径不一致

---

## 6. 关键修正与补充总结

### 6.1 本次分析的核心修正

| 主题 | 之前的理解 | 修正后的正确理解 |
|-----|-----------|-----------------|
| **空属性默认值** | `custom_props` 可能是 `nil` 或 `%{}`，两者不同 | `custom_props` 永远是 Map，默认值是 `%{}`，不允许 `nil` |
| **导入数据目标匹配** | 未深入分析 | 导入数据不支持自定义属性匹配，页面目标不检查事件名，滚动目标完全不支持 |
| **include_imported 传递** | 未明确 | 计算转化率分母时，显式传递原查询的 `include_imported` 值，确保分子分母数据集一致 |

### 6.2 目标匹配完整规则矩阵

| 目标类型 | 原生数据支持 | 导入数据支持 | 匹配条件差异 |
|---------|-------------|-------------|-------------|
| **事件目标** | ✅ 完全支持 | ✅ 部分支持 | 原生：事件名 + 自定义属性；导入：仅事件名 |
| **页面目标** | ✅ 完全支持 | ✅ 部分支持 | 原生：路径 + "pageview" 事件名 + 自定义属性；导入：仅路径 |
| **滚动目标** | ✅ 完全支持 | ❌ 不支持 | 导入数据无滚动深度字段 |

### 6.3 转化率计算口径

```
当查询 include_imported = true 时：
┌─────────────────────────────────────────────────────────────────────┐
│  分子（目标访客数）= 原生目标访客数 + 导入目标访客数（宽松匹配）       │
│  分母（总访客数）  = 原生总访客数（移除目标过滤器）                    │
│                   + 导入总访客数（移除目标过滤器）                    │
│  转化率          = (分子 / 分母) * 100                               │
│                                                                      │
│  关键：include_imported 参数同时影响分子和分母，保持数据一致性         │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.4 空属性语义

```
目标的 custom_props 字段：
┌─────────────────────────────────────────────────────────────────────┐
│  默认值: %{} (空 Map)                                                │
│  允许 NULL: 否 (null: false)                                         │
│                                                                      │
│  has_custom_props? 判断逻辑:                                          │
│  - custom_props = %{}          → map_size = 0 → has_custom_props? = false │
│  - custom_props = %{a: "b"}   → map_size = 1 → has_custom_props? = true  │
│                                                                      │
│  目标匹配时:                                                          │
│  - has_custom_props? = false → 只匹配事件名/路径，不检查属性         │
│  - has_custom_props? = true  → 匹配事件名/路径 + 所有定义的属性      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. 相关文件索引

| 文件路径 | 功能描述 |
|---------|---------|
| `lib/plausible/stats/imported/imported.ex` | 导入数据查询合并逻辑 |
| `lib/plausible/stats/imported/base.ex` | 导入数据表选择逻辑 |
| `lib/plausible/stats/goals.ex` | 目标条件构建（含 imported? 分支） |
| `lib/plausible/stats/sql/special_metrics.ex` | 转化率计算、分母计算逻辑 |
| `lib/plausible/goal.ex` | 目标数据模型、has_custom_props? 函数 |
| `priv/repo/migrations/20251211110619_goals_custom_props_default.exs` | custom_props 默认值迁移 |
| `test/plausible/goals_test.exs` | 目标去重测试用例 |
