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
2. 浏览首页（其他事件）→ **匹配中断！**
3. 添加购物车（步骤 2）
4. 查看帮助（其他事件）
5. 完成购买（步骤 3）

**严格顺序判定结果**：用户 A 只到达了步骤 1，因为：
- 步骤 1 之后立即出现了非预期事件"浏览首页"
- 匹配过程在步骤 1 之后停止
- 后续的步骤 2 和步骤 3 不再被考虑

**实际业务影响**：

| 场景 | 非严格顺序 | 严格顺序 | 适用业务场景 |
|------|-----------|---------|-------------|
| 电商购买流程 | 用户可以在浏览产品后继续浏览其他页面再回来购买 | 必须严格按照"浏览→加购→购买"连续执行 | 非严格顺序更适合大多数电商场景 |
| 表单多步提交 | 不适用（表单通常有严格顺序） | 必须严格按照步骤 1→步骤 2→步骤 3 连续填写 | 严格顺序适合引导式流程 |
| 游戏关卡 | 可以返回玩之前的关卡 | 必须按关卡顺序连续通过 | 视游戏设计而定 |

**关键代码对比**：

两种模式的唯一区别在于查询构造时是否传递 `'strict_order'` 参数：

```elixir
# 非严格顺序
dynamic(
  [q],
  fragment("windowFunnel(?)(timestamp, ?)", @funnel_window_duration, ^window_funnel_steps)
)

# 严格顺序
dynamic(
  [q],
  fragment(
    "windowFunnel(?, 'strict_order')(timestamp, ?)",
    @funnel_window_duration,
    ^window_funnel_steps
  )
)
```

这个看似微小的差异，在 ClickHouse 内部会导致完全不同的匹配算法。

#### 3.4.4 时间窗口对结果的影响

**窗口大小定义**：

```elixir
@funnel_window_duration 86_400
```

Plausible 固定使用 86400 秒（1 天）作为时间窗口。这个参数对分析结果有重大影响。

**窗口工作原理**：

1. **窗口启动**：当用户触发第一个步骤的事件时，窗口开始计时
2. **窗口持续**：窗口持续 86400 秒（从第一个匹配事件的时间戳开始）
3. **窗口内匹配**：只有在窗口内发生的后续步骤才会被计入
4. **多窗口处理**：如果用户在不同时间段有多个匹配序列，选择最长的那个

**示例场景**：

假设有一个 2 步 Funnel：浏览产品 → 完成购买

**情况 1：所有步骤在窗口内**
- 第 1 天 10:00：用户浏览产品（步骤 1，窗口启动）
- 第 1 天 15:00：用户完成购买（步骤 2，在窗口内）
- **结果**：用户到达步骤 2，转化率计入

**情况 2：步骤 2 在窗口外**
- 第 1 天 10:00：用户浏览产品（步骤 1，窗口启动）
- 第 2 天 12:00：用户完成购买（步骤 2，已超过 24 小时窗口）
- **结果**：用户只到达步骤 1，步骤 2 不计入

**情况 3：多次访问，选择最长链**
- 第 1 天 10:00：浏览产品（步骤 1，窗口 A 启动）
- 第 1 天 11:00：浏览产品（步骤 1，窗口 B 启动）
- 第 1 天 12:00：完成购买（步骤 2，在窗口 B 内）
- **结果**：选择最长的链（窗口 B：步骤 1→步骤 2），用户到达步骤 2

**窗口大小的业务含义**：

1 天的窗口大小意味着：
- **转化时效性**：系统认为用户应该在 1 天内完成转化
- **短期行为分析**：适合分析短期的用户旅程
- **跨天行为丢失**：如果用户在第 1 天浏览产品，第 2 天购买，这个转化不会被计入

**潜在问题与优化方向**：

当前固定窗口的局限性：
1. **业务适配性差**：不同业务有不同的转化周期
   - 电商购买：可能需要几小时到几天
   - B2B 转化：可能需要几周甚至几个月
   - 游戏内转化：可能只需要几分钟

2. **无法对比分析**：用户无法看到不同时间窗口下的转化差异

3. **与查询时间范围的关系**：
   - Funnel 分析有两个时间概念：
     1. **查询时间范围**：用户选择的分析时间段（如过去 30 天）
     2. **转化窗口**：固定 1 天
   - 这两个概念容易混淆，但完全不同

**建议的优化方向**：
- 允许用户自定义时间窗口（如 1 小时、1 天、7 天、30 天）
- 在 UI 中明确区分"分析时间范围"和"转化时间窗口"
- 提供不同窗口大小的对比分析功能

### 3.5 结果计算与状态回填

#### 3.5.1 原始数据处理

ClickHouse 返回的原始数据格式是 `{step_index, visitor_count}`，其中：
- `step_index`：用户到达的最深步骤（从 0 开始）
- `visitor_count`：到达该步骤的用户数量

**问题**：这种格式只告诉我们"有多少用户到达了步骤 N"，但没有告诉我们"有多少用户到达了步骤 M（M ≤ N）"。

例如，如果 ClickHouse 返回：
- {0, 100}：100 个用户未进入漏斗
- {1, 200}：200 个用户到达了步骤 1（但未到步骤 2）
- {2, 150}：150 个用户到达了步骤 2（但未到步骤 3）
- {3, 50}：50 个用户到达了步骤 3

这意味着：
- 到达步骤 1 的总用户数 = 200 + 150 + 50 = 400
- 到达步骤 2 的总用户数 = 150 + 50 = 200
- 到达步骤 3 的总用户数 = 50

**状态回填算法**：

```elixir
defp backfill_steps(funnel_result, funnel) do
  funnel_result = Enum.into(funnel_result, %{})
  max_step = Enum.max_by(funnel.steps, & &1.step_order).step_order

  funnel
  |> Map.fetch!(:steps)
  |> Enum.reduce({nil, nil, []}, fn step, {total_visitors, visitors_at_previous, acc} ->
    # 核心逻辑：累加当前步骤及后续所有步骤的用户数
    visitors_at_step =
      step.step_order..max_step
      |> Enum.map(&Map.get(funnel_result, &1, 0))
      |> Enum.sum()

    # 累计当前用户数，用于下一次迭代
    current_visitors = visitors_at_step

    # 第一个步骤的用户数作为总基数
    total_visitors = total_visitors || current_visitors

    # 计算流失：上一步用户数 - 当前步骤用户数
    dropoff = if visitors_at_previous, do: visitors_at_previous - current_visitors, else: 0

    # 计算各种转化率指标
    dropoff_percentage = percentage(dropoff, visitors_at_previous)
    conversion_rate = percentage(current_visitors, total_visitors)
    conversion_rate_step = percentage(current_visitors, visitors_at_previous)

    # 构建步骤结果
    step = %{
      dropoff: dropoff,
      dropoff_percentage: dropoff_percentage,
      conversion_rate: conversion_rate,
      conversion_rate_step: conversion_rate_step,
      visitors: visitors_at_step,
      label: to_string(step.goal)
    }

    {total_visitors, current_visitors, [step | acc]}
  end)
  |> elem(2)
  |> Enum.reverse()
end
```

#### 3.5.2 指标计算详解

**五个核心指标**：

1. **visitors（到达该步骤的访客数）**
   - 含义：所有到达或超过该步骤的用户总数
   - 计算：`sum(用户数 for 步骤 N to 最大步骤)`
   - 示例：步骤 1 的 visitors = 到达步骤 1 的用户 + 到达步骤 2 的用户 + 到达步骤 3 的用户

2. **dropoff（从上个步骤流失的访客数）**
   - 含义：从上一个步骤流失的用户数量
   - 计算：`上一步骤 visitors - 当前步骤 visitors`
   - 示例：如果步骤 1 有 400 人，步骤 2 有 200 人，则 dropoff = 200

3. **dropoff_percentage（流失率百分比）**
   - 含义：流失用户占上一步骤用户的比例
   - 计算：`dropoff / visitors_at_previous * 100%`
   - 示例：200 人流失 / 400 人上一步 = 50% 流失率

4. **conversion_rate（总转化率）**
   - 含义：从第一个步骤到当前步骤的累计转化率
   - 计算：`current_visitors / total_visitors * 100%`
   - 示例：步骤 3 有 50 人 / 步骤 1 有 400 人 = 12.5% 总转化率

5. **conversion_rate_step（步转化率）**
   - 含义：从上一个步骤到当前步骤的单步转化率
   - 计算：`current_visitors / visitors_at_previous * 100%`
   - 示例：步骤 2 有 200 人 / 步骤 1 有 400 人 = 50% 步转化率

**指标关系图**：

```
步骤 1 (400 人)
├── 流失：0 人（第一步无流失）
├── 总转化率：100%
└── 步转化率：100%
    ↓
步骤 2 (200 人)
├── 流失：200 人 (400-200)
├── 流失率：50% (200/400)
├── 总转化率：50% (200/400)
└── 步转化率：50% (200/400)
    ↓
步骤 3 (50 人)
├── 流失：150 人 (200-50)
├── 流失率：75% (150/200)
├── 总转化率：12.5% (50/400)
└── 步转化率：25% (50/200)
```

### 3.6 API 层

#### 3.6.1 API 控制器 (PlausibleWeb.Plugins.API.Controllers.Funnels)

**文件位置**：`extra/lib/plausible_web/plugins/API/controllers/funnels.ex`

提供 RESTful API 接口：

1. **创建 Funnel**：`POST /api/funnels`
   - 支持获取或创建（Get or Create）模式
   - 验证请求体参数
   - 检查权限

2. **获取 Funnel 列表**：`GET /api/funnels`
   - 支持分页（limit, after, before 参数）

3. **获取单个 Funnel**：`GET /api/funnels/:id`
   - 根据 ID 获取 Funnel 详情

#### 3.6.2 API Schema (PlausibleWeb.Plugins.API.Schemas.Funnel)

**文件位置**：`extra/lib/plausible_web/plugins/API/schemas/funnel.ex`

定义 API 的数据结构和验证规则：
- Funnel 对象包含 `id`, `name`, `steps` 字段
- Steps 是 Goal 对象数组，最少 2 个，最多 8 个
- 提供示例数据用于文档生成

### 3.7 前端展示层

#### 3.7.1 Funnel 配置表单 (PlausibleWeb.Live.FunnelSettings.Form)

**文件位置**：`extra/lib/plausible_web/live/funnel_settings/form.ex`

这是一个 Phoenix LiveComponent，用于创建和编辑 Funnel。

**核心状态管理**：

```elixir
# 关键状态字段
socket.assigns = %{
  goals: goals,                    # 站点可用的所有 Goals
  site: site,                      # 当前站点
  evaluation_result: nil,          # 实时预览结果
  form: form,                      # 表单数据
  funnel: funnel,                  # 正在编辑的 Funnel（如为编辑模式）
  strict_order?: false,            # 是否严格顺序
  funnel_modified?: false,         # Funnel 是否已修改
  selections_made: %{},            # 已选择的步骤映射："step-1" => %Goal{}
  step_ids: [1, 2]                 # 步骤 ID 列表
}
```

**动态步骤管理流程**：

1. **初始化**：
   - 加载站点所有 Goals
   - 最少初始化 2 个步骤

2. **添加步骤**：
   ```elixir
   def handle_event("add-step", _value, socket) do
     step_ids = socket.assigns.step_ids
     socket = assign(socket, funnel_modified?: true)
     
     if length(step_ids) < Funnel.max_steps() do
       first_free_idx = find_sequence_break(step_ids)
       new_ids = step_ids ++ [first_free_idx]
       {:noreply, assign(socket, step_ids: new_ids)}
     else
       {:noreply, socket}
     end
   end
   ```

3. **删除步骤**：
   - 从 `step_ids` 中移除
   - 从 `selections_made` 中清除对应选择
   - 触发重新评估

4. **选择 Goal**：
   - 更新 `selections_made` 映射
   - 发送 `:evaluate_funnel` 消息触发实时预览

**实时预览机制**：

```elixir
defp evaluate_funnel(
       %{
         assigns: %{
           site: site,
           selections_made: selections_made,
           strict_order?: strict_order?
         }
       } = socket
     ) do
  with {:ok, {definition, query}} <-
         build_ephemeral_funnel(site, selections_made, strict_order?: strict_order?),
       {:ok, funnel} <- Plausible.Stats.funnel(site, query, definition) do
    assign(socket, evaluation_result: funnel)
  else
    _ ->
      socket
  end
end
```

**临时 Funnel 构建**：
- 使用 `ephemeral_definition` 创建不保存到数据库的 Funnel
- 使用上个月的数据进行预览计算
- 当配置变化时自动重新计算

#### 3.7.2 Funnel 图表组件 (Funnel React 组件)

**文件位置**：`assets/js/dashboard/extra/funnel.js`

**核心渲染逻辑**：

**数据获取流程**：
1. 从上下文获取站点信息和仪表板状态
2. 根据 Funnel 名称查找对应的 Funnel ID
3. 调用 API 获取分析结果
4. 响应状态变化（如时间范围改变）

**图表配置**：

```javascript
const config = {
  plugins: [ChartDataLabels],
  type: 'bar',
  data: data,
  options: {
    responsive: true,
    barThickness: calcBarThickness,
    plugins: {
      legend: { display: false },
      tooltip: {
        enabled: false,
        mode: 'index',
        intersect: true,
        position: 'average',
        external: FunnelTooltip(palette, funnel)  // 自定义工具提示
      },
      datalabels: {
        formatter: formatDataLabel,  // 显示转化率
        anchor: 'end',
        align: 'end',
        offset: calcOffset,
        backgroundColor: palette.dataLabelBackground,
        color: palette.dataLabelTextColor,
        borderRadius: 4,
        clip: true,
        font: {
          size: 12,
          weight: 'normal',
          lineHeight: 1.6,
          family: fontFamily
        },
        textAlign: 'center',
        padding: { top: 8, bottom: 8, right: 8, left: 8 }
      }
    },
    scales: {
      y: { display: false },
      x: {
        position: 'bottom',
        display: true,
        border: { display: false },
        grid: { drawBorder: false, display: false },
        ticks: {
          padding: 8,
          font: { weight: 'bold', family: fontFamily, size: 14 },
          color: palette.stepNameLegendColor
        }
      }
    }
  }
}
```

**数据结构**：

```javascript
const data = {
  labels: funnel.steps.map((step) => step.label),
  datasets: [
    {
      label: 'Visitors',
      data: stepData,  // 每个步骤的访客数
      backgroundColor: gradient,
      hoverBackgroundColor: gradient,
      borderRadius: 4,
      stack: 'Stack 0'
    },
    {
      label: 'Dropoff',
      data: dropOffData,  // 每个步骤的流失数
      backgroundColor: createDiagonalPattern(
        palette.dropoffBackground,
        palette.dropoffStripes
      ),
      hoverBackgroundColor: palette.dropoffBackground,
      borderRadius: 4,
      stack: 'Stack 0'
    }
  ]
}
```

**响应式设计**：

```javascript
// 检测屏幕尺寸
useEffect(() => {
  const mediaQuery = window.matchMedia('(max-width: 768px)')
  setSmallScreen(mediaQuery.matches)
  const handleScreenChange = (e) => {
    setSmallScreen(e.matches)
  }
  mediaQuery.addEventListener('change', handleScreenChange)
  return () => {
    mediaQuery.removeEventListener('change', handleScreenChange)
  }
}, [])

// 条件渲染
{isSmallScreen && (
  <div className="mt-4">{renderBars(funnel, theme)}</div>
)}
{!isSmallScreen && (
  <canvas className="" id="funnel" ref={canvasRef}></canvas>
)}
```

**小屏幕渲染**：
- 使用简单的条形图列表
- 显示步骤名称、访客数
- 不显示详细的流失率图表

#### 3.7.3 视觉设计细节

**渐变背景**：
```javascript
var gradient = ctx.createLinearGradient(900, 0, 900, 900)
gradient.addColorStop(1, palette.dropoffBackground)
gradient.addColorStop(0, palette.visitorsBackground)
```

**斜线图案（流失率）**：
```javascript
const createDiagonalPattern = (color1, color2) => {
  let shape = document.createElement('canvas')
  shape.width = 10
  shape.height = 10
  let c = shape.getContext('2d')

  c.fillStyle = color1
  c.strokeStyle = color2
  c.fillRect(0, 0, shape.width, shape.height)

