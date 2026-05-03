# Funnel 分析实现思路分析

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

**核心实现细节**：

#### 3.3.2 多步查询构造

系统使用 ClickHouse 的 `windowFunnel` 函数来实现多步分析：

```elixir
# 普通模式
fragment("windowFunnel(?)(timestamp, ?)", @funnel_window_duration, ^window_funnel_steps)

# 严格顺序模式
fragment(
  "windowFunnel(?, 'strict_order')(timestamp, ?)",
  @funnel_window_duration,
  ^window_funnel_steps
)
```

**关键参数**：
- `@funnel_window_duration`：86400 秒（1 天），表示分析的时间窗口
- `window_funnel_steps`：动态构建的步骤条件列表
- `strict_order`：是否严格按照步骤顺序执行

#### 3.3.3 步骤归因

步骤归因是 Funnel 分析的核心，系统通过以下方式实现：

1. **条件构建**：
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

2. **查询执行**：
   - 按用户 ID 分组
   - 计算每个用户到达的最深步骤
   - 按步骤计数分组

3. **结果回填**：
   - ClickHouse 只返回每个用户到达的最深步骤
   - 需要累加计算每个步骤的总访客数
   - 计算转化率和流失率

#### 3.3.4 结果计算与展示

`backfill_steps/2` 函数处理原始查询结果，计算各种指标：

```elixir
# 计算当前步骤的访客数（包括后续步骤的用户）
visitors_at_step =
  step.step_order..max_step
  |> Enum.map(&Map.get(funnel_result, &1, 0))
  |> Enum.sum()

# 计算流失率和转化率
dropoff = if visitors_at_previous, do: visitors_at_previous - current_visitors, else: 0
dropoff_percentage = percentage(dropoff, visitors_at_previous)
conversion_rate = percentage(current_visitors, total_visitors)
conversion_rate_step = percentage(current_visitors, visitors_at_previous)
```

**结果结构**：
每个步骤包含以下信息：
- `visitors`：到达该步骤的访客数
- `dropoff`：从上个步骤流失的访客数
- `dropoff_percentage`：流失率百分比
- `conversion_rate`：从第一个步骤到当前步骤的总转化率
- `conversion_rate_step`：从上一个步骤到当前步骤的步转化率
- `label`：步骤的名称（Goal 名称）

### 3.4 API 层

#### 3.4.1 API 控制器 (PlausibleWeb.Plugins.API.Controllers.Funnels)

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

#### 3.4.2 API Schema (PlausibleWeb.Plugins.API.Schemas.Funnel)

**文件位置**：`extra/lib/plausible_web/plugins/API/schemas/funnel.ex`

定义 API 的数据结构和验证规则：
- Funnel 对象包含 `id`, `name`, `steps` 字段
- Steps 是 Goal 对象数组，最少 2 个，最多 8 个
- 提供示例数据用于文档生成

### 3.5 前端展示层

#### 3.5.1 Funnel 配置表单 (PlausibleWeb.Live.FunnelSettings.Form)

**文件位置**：`extra/lib/plausible_web/live/funnel_settings/form.ex`

这是一个 Phoenix LiveComponent，用于创建和编辑 Funnel：

**主要功能**：
1. **动态步骤管理**：
   - 支持添加/删除步骤（2-8 个）
   - 使用 ComboBox 组件选择 Goal
   - 已选择的 Goal 不能再次选择

2. **实时预览**：
   - 选择步骤后自动计算预览结果
   - 显示上个月的转化率
   - 显示每个步骤的访客数和流失率

3. **严格顺序切换**：
   - 提供开关控制 `strict_order` 选项
   - 切换时自动重新计算预览

**关键实现**：
- 使用 `selections_made` 映射跟踪已选择的步骤
- 使用 `ephemeral_definition` 创建临时 Funnel 用于预览
- 当选择变化或严格顺序切换时，发送 `:evaluate_funnel` 消息触发重新计算

#### 3.5.2 Funnel 图表组件 (Funnel React 组件)

**文件位置**：`assets/js/dashboard/extra/funnel.js`

这是一个 React 组件，用于在仪表板上展示 Funnel 分析结果：

**主要功能**：
1. **数据获取**：
   - 根据 Funnel 名称从 API 获取数据
   - 响应仪表板状态变化（如时间范围）

2. **图表渲染**：
   - 使用 Chart.js 渲染柱状图
   - 每个步骤显示两个堆叠的柱子：
     - 蓝色渐变：到达该步骤的访客
     - 斜线图案：从上个步骤流失的访客
   - 显示转化率标签

3. **响应式设计**：
   - 大屏幕：完整的 Chart.js 图表
   - 小屏幕：简化的条形图列表

4. **交互功能**：
   - 悬停显示详细信息（通过 FunnelTooltip）
   - 支持深色/浅色主题

**关键实现**：
- 使用 `useEffect` 钩子处理数据获取和图表初始化
- 使用 `useRef` 管理 Chart.js 实例
- 实现 `getPalette` 函数根据主题返回颜色配置
- 使用 `createDiagonalPattern` 创建流失率的斜线图案

#### 3.5.3 Funnel 工具提示 (FunnelTooltip)

**文件位置**：`assets/js/dashboard/extra/funnel-tooltip.js`

为 Funnel 图表提供自定义工具提示：
- 显示步骤的详细指标
- 根据鼠标位置动态定位
- 支持深色/浅色主题

## 4. 复杂度来源分析

### 4.1 查询复杂度

1. **ClickHouse `windowFunnel` 函数**：
   - 这是一个相对高级的分析函数，理解其工作原理需要一定学习成本
   - 不同模式（普通/严格顺序）的行为差异需要仔细处理

2. **动态查询构建**：
   - 需要根据 Funnel 定义动态构建查询条件
   - 处理不同类型的 Goal（页面访问、自定义事件、收入等）

3. **结果回填**：
   - ClickHouse 返回的是每个用户到达的最深步骤
   - 需要累加计算每个步骤的总访客数
   - 计算多种转化率指标（总转化率、步转化率、流失率）

### 4.2 数据模型复杂度

1. **多表关联**：
   - Funnel → Step → Goal 的三级关联
   - 需要正确处理预加载和关联查询

2. **动态步骤**：
   - 步骤数量不固定（2-8 个）
   - 需要动态管理步骤顺序和关联

3. **约束验证**：
   - 步骤数量限制
   - 名称唯一性
   - 与计费系统的集成

### 4.3 前端复杂度

1. **动态表单**：
   - LiveComponent 中管理动态步骤
   - 实时预览计算
   - 步骤选择的状态管理

2. **图表渲染**：
   - Chart.js 的配置和自定义
   - 响应式布局
   - 主题适配（深色/浅色）

3. **交互体验**：
   - 工具提示的定位和样式
   - 数据加载状态管理
   - 错误处理

### 4.4 业务逻辑复杂度

1. **严格顺序 vs 宽松顺序**：
   - 理解两种模式的业务含义
   - 在查询和结果处理中正确实现

2. **时间窗口**：
   - 固定的 1 天时间窗口
   - 理解其对分析结果的影响

3. **计费集成**：
   - Funnels 功能是高级功能
   - 需要检查用户权限和套餐

## 5. 关键流程梳理

### 5.1 Funnel 创建流程

1. **用户打开表单**：
   - 加载站点的所有 Goals
   - 初始化最少 2 个步骤的表单

2. **用户配置 Funnel**：
   - 输入名称
   - 选择步骤（每个步骤对应一个 Goal）
   - 选择是否严格顺序
   - 实时预览计算结果

3. **保存 Funnel**：
   - 验证表单数据
   - 检查计费权限
   - 保存到数据库

### 5.2 Funnel 分析流程

1. **请求发起**：
   - 用户在仪表板选择 Funnel
   - 前端根据 Funnel 名称获取 ID
   - 调用 API 获取分析结果

2. **查询执行**：
   - 后端根据 Funnel 定义构造 ClickHouse 查询
   - 使用 `windowFunnel` 函数执行多步分析
   - 按步骤分组计数

3. **结果计算**：
   - 回填步骤数据，累加后续步骤的访客
   - 计算转化率、流失率等指标
   - 构建返回给前端的结果结构

4. **前端展示**：
   - 解析返回的数据
   - 根据屏幕尺寸选择合适的渲染方式
   - 渲染图表或条形图
   - 提供交互功能

## 6. 技术亮点与设计思路

### 6.1 技术亮点

1. **利用 ClickHouse 原生功能**：
   - 直接使用 `windowFunnel` 函数，避免复杂的自定义实现
   - 利用 ClickHouse 的高性能分析能力

2. **模块化设计**：
   - 清晰的模块划分：数据模型、业务逻辑、统计分析、API、前端
   - 每个模块职责单一，易于维护和测试

3. **实时预览**：
   - 表单配置时实时计算预览结果
   - 提高用户体验，减少保存后的调整

4. **响应式前端**：
   - 自适应不同屏幕尺寸
   - 提供最佳的视觉体验

### 6.2 设计思路

1. **数据驱动**：
   - Funnel 定义完全存储在数据库中
   - 查询和计算完全基于这些定义

2. **渐进式复杂度**：
   - 核心功能（Funnel 定义和基本分析）相对简单
   - 高级功能（严格顺序、实时预览、图表渲染）逐步增加复杂度

3. **关注点分离**：
   - 后端专注于数据查询和计算
   - 前端专注于用户交互和可视化
   - API 层作为中间层，提供清晰的接口

4. **性能优化**：
   - 使用 ClickHouse 作为分析引擎，适合大数据量
   - 前端懒加载，只在需要时获取数据
   - 图表组件只在可见时渲染

## 7. 潜在优化方向

1. **可配置的时间窗口**：
   - 目前时间窗口固定为 1 天
   - 可以考虑让用户自定义时间窗口

2. **更丰富的分析维度**：
   - 目前只按步骤分析
   - 可以考虑增加按时间、按来源等维度的细分

3. **性能优化**：
   - 对于大数据量的 Funnel 分析，可以考虑缓存结果
   - 优化查询构造，减少不必要的计算

4. **用户体验**：
   - 提供更多的图表类型和定制选项
   - 增强交互功能，如步骤详情查看、对比分析等

## 8. 总结

Plausible 的 Funnel 分析系统是一个设计良好、功能完整的用户旅程分析工具。它通过以下核心技术实现了复杂的多步分析：

1. **数据模型**：灵活的 Funnel-Step-Goal 关联结构
2. **查询引擎**：利用 ClickHouse 的 `windowFunnel` 函数
3. **结果计算**：智能的结果回填和指标计算
4. **用户界面**：直观的配置表单和美观的图表展示

系统的复杂度主要来源于：
- 高级数据库函数的使用和理解
- 动态查询构建和结果处理
- 响应式前端和交互体验
- 与现有系统（计费、Goal 管理）的集成

整体设计遵循了模块化、关注点分离的原则，使得系统既功能强大，又易于维护和扩展。
