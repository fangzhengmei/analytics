# 数据保留策略、导入导出流程与历史数据查询兼容协作机制分析报告

## 1. 引言

本报告深入分析了 Plausible Analytics 系统中数据保留策略、导入导出流程和历史数据查询兼容的协作机制。重点关注保留规则的执行、格式兼容性和旧数据访问路径三个核心维度，旨在全面理解系统如何管理数据生命周期并确保历史数据的可访问性。

## 2. 数据保留策略执行机制

### 2.1 策略定义与配置

数据保留策略在系统中通过两个主要层面定义：

**1. 计划级别定义**
- 在 `Plausible.Billing.Plan` 模块中，通过 `data_retention_in_years` 字段定义了每个订阅计划的数据保留年限
- 该字段是可选字段，允许不同计划有不同的数据保留期限

**2. 实现位置**
- 相关代码位于 `lib/plausible/billing/plan.ex:26`
- 作为嵌入式 schema 的一部分，与其他计划属性（如功能列表、页面浏览限制等）一起定义

### 2.2 数据清理机制

系统采用异步清理机制处理数据删除，主要涉及两个核心模块：

**1. 数据清理模块 (`Plausible.Purge`)**

该模块提供了删除导入统计数据的功能，具有以下特点：

- **ClickHouse 异步删除特性**：由于 ClickHouse 的数据删除是异步执行的，系统设计了特殊的处理机制
- **导入表去重机制禁用**：所有导入表都禁用了 MergeTree 的去重机制（通过设置 `replicated_deduplication_window` 为 0），这是为了避免删除后重新导入相同数据被误认为重复的问题
- **多维度删除支持**：
  - 按站点删除所有导入统计数据 (`delete_imported_stats!/1` with `Plausible.Site`)
  - 按特定导入删除数据 (`delete_imported_stats!/1` with `Plausible.Imported.SiteImport`)
  - 支持遗留导入数据的特殊处理（`legacy` 标记）

**2. 站点数据清理 Worker (`Plausible.Workers.ClickhouseCleanSites`)**

这是一个后台任务，用于异步清理已删除站点的 ClickHouse 数据：

- **批量处理**：系统会批量处理数据删除，因为删除单个站点与删除多个站点的开销相同
- **清理范围**：涵盖所有相关表，包括原生数据表（`events_v2`, `sessions_v2`, `ingest_counters`）和所有导入数据表（`imported_*` 系列）
- **同步设置**：在测试环境中使用同步突变（`mutations_sync: 2`），而在生产环境中使用异步处理

### 2.3 导入数据管理规则

导入数据有其特殊的管理规则：

**1. 导入数量限制**
- 系统限制最多同时保留 5 个完整的导入记录 (`@max_complete_imports 5`)
- 此限制在 `lib/plausible/imported.ex:33` 中定义

**2. 导入元数据管理**
- 每个导入都有详细的元数据记录，包括：
  - 导入 ID（用于唯一标识）
  - 开始和结束日期（用于数据范围查询）
  - 状态（待处理、导入中、已完成、失败）
  - 源类型（Google Analytics、CSV 等）
  - 遗留标记（用于区分旧格式导入）

**3. 日期范围管理**
- 系统会自动计算已占用的日期范围，防止重复导入相同时间段的数据
- 提供了查找空闲日期范围的功能，用于新导入的时间窗口选择

## 3. 导入导出流程及格式兼容性

### 3.1 导入流程架构

系统采用灵活的导入架构，基于行为模式设计：

**1. 导入器行为定义 (`Plausible.Imported.Importer`)**

这是所有导入源必须实现的核心行为，定义了以下回调函数：

- **基本信息**：
  - `name/0`：返回导入源名称（原子类型）
  - `label/0`：返回显示友好的源名称
  - `email_template/0`：指定通知使用的邮件模板

- **参数处理**：
  - `parse_args/1`：解析 Oban 作业参数，转换为关键字列表

- **核心功能**：
  - `import_data/2`：执行实际的导入处理，必须是同步过程
  - `before_start/2`：可选回调，在调度导入作业前执行
  - `on_success/2`：可选回调，在导入完成后执行
  - `on_failure/1`：可选回调，在导入永久失败时执行

**2. 导入作业调度**

导入流程通过 Oban 后台作业系统执行：

- **事务性创建**：导入创建过程在数据库事务中执行，确保数据一致性
- **状态管理**：导入状态从 "pending" → "importing" → "completed/failed" 流转
- **通知机制**：通过 Oban.Notifier 发送导入完成、失败等状态通知
- **互斥保障**：确保同一时间只有一个导入在执行

### 3.2 多导入源支持

系统支持多种导入源，通过 `Plausible.Imported.ImportSources` 模块统一管理：

**1. 内置导入源**：
- `Plausible.Imported.GoogleAnalytics4`：Google Analytics 4 导入
- `Plausible.Imported.UniversalAnalytics`：Universal Analytics（旧版 GA）导入
- `Plausible.Imported.NoopImporter`：空操作导入器（用于测试）
- `Plausible.Imported.CSVImporter`：CSV 文件导入

**2. 导入源注册机制**：
- 所有导入源通过 `@sources` 列表注册
- 通过 `@sources_map` 建立名称到模块的映射
- 提供 `by_name/1` 函数根据名称获取导入器模块

### 3.3 CSV 导入实现细节

CSV 导入是最通用的导入方式，具有以下特点：

**1. 存储方式支持**：
- **S3 存储**：使用 ClickHouse 的 `s3` 表函数直接从 S3 读取数据
- **本地存储**：使用 ClickHouse 的 `input` 函数从本地文件读取数据

**2. 文件名格式约定**：
CSV 文件必须遵循特定的命名格式：
- 格式：`{table_name}_{start_date}_{end_date}.csv`
- 示例：`imported_devices_20190101_20210101.csv`
- 支持简化格式：`devices_20190101_20210101.csv`（自动添加 `imported_` 前缀）

**3. 表结构映射**：
系统预定义了各导入表的结构，确保数据正确映射：

```elixir
input_structures = %{
  "imported_browsers" => "date Date, browser String, browser_version String, visitors UInt64, visits UInt64, visit_duration UInt64, bounces UInt32, pageviews UInt64",
  "imported_devices" => "date Date, device String, visitors UInt64, visits UInt64, visit_duration UInt64, bounces UInt32, pageviews UInt64",
  # ... 其他表结构
}
```

**4. 导入过程优化**：
- 分块读取：本地文件使用 512KB 块读取，提高性能
- 日期过滤：导入时自动过滤日期范围外的数据
- 清理任务：本地导入完成后，安排 1 小时后清理本地文件

### 3.4 导出流程与格式兼容性

导出功能与导入功能设计为高度兼容，确保导出的数据可以重新导入系统：

**1. 导出架构**：
- **调度机制**：通过 Oban 作业系统调度导出任务
- **存储选项**：支持 S3 存储和本地存储两种方式
- **通知机制**：导出完成后通过邮件通知用户

**2. 格式兼容性设计**：

导出功能的核心设计理念是"导出即可导入"，具体体现在：

- **统一表格式**：导出数据使用与导入表完全相同的结构（`imported_*` 表格式）
- **查询转换**：`export_queries/2` 函数将原生数据查询转换为导入表格式
- **文件名一致**：导出文件使用与导入文件相同的命名格式

**3. 导出查询构建**：

系统为每种数据类型构建专门的导出查询：

- **访客数据**：从 `sessions_v2` 和 `events_v2` 表聚合数据
- **来源数据**：从 `sessions_v2` 表提取流量来源信息
- **页面数据**：从 `events_v2` 表聚合页面访问数据，包括滚动深度和页面停留时间
- **其他维度**：入口页面、出口页面、自定义事件、地理位置、设备、浏览器、操作系统等

**4. 流式处理**：
- 使用 `stream_archive/3` 函数实现流式数据导出
- 直接从数据库流式读取数据并写入 ZIP 压缩包
- 避免内存中缓存大量数据

### 3.5 数据迁移机制

系统提供了协调式数据迁移框架，用于处理格式版本变更：

**1. 迁移框架 (`Plausible.DataMigration`)**：
- 使用宏定义迁移模块结构
- 支持 SQL 模板文件（`.sql.eex` 格式）
- 提供交互式确认机制，防止误操作
- 支持单查询和多查询执行模式

**2. 迁移执行流程**：
- 从 `priv/data_migrations/{dir}/sql/` 目录读取 SQL 模板
- 支持参数替换（使用 EEx 模板）
- 提供执行确认提示，降低风险
- 记录执行状态和结果

## 4. 历史数据查询兼容实现

### 4.1 查询处理架构

系统设计了统一的查询处理架构，确保原生数据和导入数据的无缝整合：

**1. 查询结构 (`Plausible.Stats.Query`)**：

查询结构体包含专门用于处理导入数据的字段：

- `imports_exist`：布尔值，表示是否存在导入数据
- `imports_in_range`：列表，包含查询时间范围内的导入记录
- `include_imported`：布尔值，表示是否包含导入数据
- `skip_imported_reason`：原子值，说明不包含导入数据的原因

**2. 导入数据决策逻辑**：

系统在 `put_imported_opts/2` 函数中实现了是否包含导入数据的决策逻辑：

- **请求检查**：检查用户是否明确请求包含导入数据（`query.include.imports`）
- **模式支持检查**：验证查询模式是否支持导入数据（如不支持分钟级和小时级时间间隔）
- **存在性检查**：确认是否存在已完成的导入
- **范围检查**：确认是否有导入在查询时间范围内
- **查询支持检查**：验证具体查询是否支持导入数据

**3. 不包含导入数据的原因**：

系统定义了以下可能的原因：
- `:unsupported_interval`：不支持的时间间隔（分钟/小时级）
- `:no_imported_data`：没有导入数据
- `:out_of_range`：导入数据不在查询范围内
- `:unsupported_query`：查询类型不支持导入数据

### 4.2 导入数据查询支持

系统定义了导入数据能够支持的查询类型：

**1. 时间间隔限制**：
- 不支持 `time:minute`（分钟级）时间维度
- 不支持 `time:hour`（小时级）时间维度
- 支持天、周、月等更大时间粒度的查询

**2. 过滤条件支持**：
- 大多数无过滤条件的查询都支持导入数据
- 自定义属性过滤有特殊限制：
  - 仅支持两个自定义属性：`url` 和 `path`
  - 这些属性只能与特定的 `event:name` 过滤器或相应的目标过滤器一起使用

**3. 表决策逻辑**：
- 系统通过 `decide_tables/1` 函数根据查询维度和过滤器决定使用哪些导入表
- 建立了属性到表的映射关系（`@property_to_table_mappings`）

### 4.3 原生数据与导入数据合并机制

这是历史数据查询兼容的核心实现，通过 `Plausible.Stats.Imported` 模块实现：

**1. 合并策略**：

系统根据查询类型采用不同的合并策略：

- **无维度查询**：使用交叉连接（cross join）合并聚合结果
- **单维度查询（目标维度）**：处理自定义事件和页面目标的特殊逻辑
- **多维度查询**：使用全连接（full join）按维度值合并数据

**2. 核心合并函数**：

`merge_imported/3` 是合并逻辑的入口点，执行以下步骤：

1. 检查 `include_imported` 标志，如为 false 则直接返回原生查询
2. 对原生查询应用分页优化（如适用）
3. 构建导入数据查询：
   - 调用 `query_imported/2` 获取基础导入查询
   - 应用 `visitors > 0` 过滤，排除无效数据
   - 按查询维度分组（`group_imported_by/1`）
   - 选择导入指标（`select_imported_metrics/2`）
   - 应用分页优化
4. 执行全连接合并：
   - 使用 `QueryBuilder.build_group_by_join/1` 构建连接条件
   - 选择合并后的维度（`select_joined_dimensions/2`）
   - 选择合并后的指标（`select_joined_metrics/2`）

**3. 过滤建议合并**：

系统还为过滤建议（如国家、地区、城市等）提供专门的合并函数：

- `merge_imported_country_suggestions/3`：合并国家建议
- `merge_imported_region_suggestions/3`：合并地区建议
- `merge_imported_city_suggestions/3`：合并城市建议
- `merge_imported_filter_suggestions/5`：合并通用过滤建议

这些函数确保用户在使用过滤功能时，能够看到来自原生数据和导入数据的完整建议列表。

**4. 分页优化**：

对于高基数维度（如页面路径）的查询，系统应用了特殊的分页优化：

- 问题：直接合并所有原生和导入数据行可能非常耗时（数百万行）
- 解决方案：在合并前分别对两个数据集应用 `LIMIT N * 100` 限制
- 权衡：这种优化是有损的，因为真正的前 N 个值可能来自任一子查询的前 C 项之外
- 适用条件：仅在可以确定性排序的情况下应用（如打开仪表板的标准查询）

### 4.4 导入元数据查询支持

系统提供了丰富的导入元数据查询功能，支持历史数据访问：

**1. 导入存在性检查**：
- `any_completed_imports?/1`：检查站点是否有任何已完成的导入

**2. 日期范围查询**：
- `earliest_import_start_date/1`：获取最早导入的开始日期
- `completed_imports_in_query_range/2`：获取查询时间范围内的导入

**3. 导入标识管理**：
- `complete_import_ids/1`：获取所有已完成导入的 ID 列表
- 特殊处理遗留导入：如有遗留导入，在列表前添加 0 作为特殊标识

**4. 日期范围管理**：
- `get_occupied_date_ranges/1`：获取已被导入占用的日期范围
- `clamp_dates/3`：限制导入日期范围，避免与现有导入重叠
- `find_free_ranges/3`：查找可用的空闲日期范围

## 5. 协作机制分析

### 5.1 数据生命周期管理流程

系统通过一系列协调机制管理数据的完整生命周期：

**1. 数据导入阶段**：
1. 用户选择导入源和时间范围
2. 系统检查日期范围是否与现有导入重叠
3. 创建导入记录（状态：pending）
4. 调度 Oban 导入作业
5. 作业执行（状态：importing）：
   - 从源读取数据
   - 转换为统一的导入表格式
   - 写入 ClickHouse 的 `imported_*` 表
   - 记录 import_id 用于标识
6. 完成或失败状态更新
7. 发送通知邮件

**2. 数据查询阶段**：
1. 用户发起数据查询请求
2. 系统解析查询参数，构建 Query 结构体
3. 检查是否需要包含导入数据：
   - 验证查询时间间隔支持
   - 检查是否存在导入数据
   - 确认导入数据是否在查询范围内
   - 验证查询类型支持
4. 如包含导入数据：
   - 分别构建原生数据查询和导入数据查询
   - 应用分页优化（如适用）
   - 执行合并操作（根据查询类型选择合适的合并策略）
5. 返回合并后的查询结果

**3. 数据清理阶段**：
1. 用户删除导入或站点
2. 系统标记数据为待删除：
   - 对于导入：更新导入记录状态
   - 对于站点：从 PostgreSQL 中删除站点记录
3. 异步清理执行：
   - `ClickhouseCleanSites` worker 定期检查已删除站点
   - 批量执行 ClickHouse 的 `ALTER TABLE ... DELETE` 操作
   - 对于导入数据，使用 `Plausible.Purge` 模块执行删除

### 5.2 组件间交互关系

系统各组件通过明确的接口和依赖关系协同工作：

**1. 导入相关组件**：
- `Plausible.Imported`：上下文模块，提供导入管理的公共 API
- `Plausible.Imported.Importer`：行为定义，规范导入器接口
- `Plausible.Imported.ImportSources`：导入源注册表
- `Plausible.Imported.CSVImporter` 等：具体导入实现
- `Plausible.Workers.ImportAnalytics`：导入作业执行器
- `Plausible.Web.Live.ImportsExportsSettings`：用户界面交互

**2. 查询相关组件**：
- `Plausible.Stats.Query`：查询结构体和基础操作
- `Plausible.Stats.QueryBuilder`：查询构建器
- `Plausible.Stats.Imported`：导入数据查询和合并逻辑
- `Plausible.Stats.Imported.SQL.Expression`：导入数据 SQL 表达式
- `Plausible.Stats.SQL.QueryBuilder`：SQL 查询构建

**3. 数据管理相关组件**：
- `Plausible.Purge`：数据删除逻辑
- `Plausible.Workers.ClickhouseCleanSites`：已删除站点清理
- `Plausible.Billing.Plan`：数据保留策略定义
- `Plausible.Site.Removal`：站点移除逻辑

### 5.3 关键设计决策

系统设计中体现了几个关键的架构决策：

**1. 统一数据格式**：
- 决策：导入数据和导出数据使用相同的表格式（`imported_*`）
- 理由：
  - 简化数据迁移流程：导出的数据可直接重新导入
  - 统一查询逻辑：无需为不同数据源编写不同的查询
  - 降低维护成本：只需维护一套格式定义
- 实现：
  - 导出时将原生数据转换为导入表格式
  - 导入时直接使用导入表格式
  - 查询时统一处理两种数据源

**2. 异步数据删除**：
- 决策：ClickHouse 数据删除采用异步机制
- 理由：
  - ClickHouse 的特性：数据删除是异步操作，通过 mutations 实现
  - 性能考虑：批量删除比单次删除更高效
  - 用户体验：无需等待删除完成即可继续操作
- 实现：
  - 使用 `ALTER TABLE ... DELETE` 语句
  - 通过后台 worker 定期执行清理
  - 测试环境使用同步设置，生产环境使用异步

**3. 导入数据与原生数据分离存储**：
- 决策：导入数据存储在独立的 `imported_*` 表中，与原生数据（`events_v2`、`sessions_v2`）分离
- 理由：
  - 数据来源可追溯：通过 `import_id` 字段明确标识数据来源
  - 灵活管理：可单独删除或管理特定导入的数据
  - 查询性能：避免混合存储导致的查询复杂性
- 实现：
  - 导入数据写入 `imported_*` 系列表
  - 原生数据写入 `events_v2` 和 `sessions_v2` 表
  - 查询时通过合并逻辑整合两种数据源

**4. 查询时合并而非存储时合并**：
- 决策：在查询执行时合并原生数据和导入数据，而非在导入时合并到原生表
- 理由：
  - 数据完整性：保留原始数据的来源信息
  - 灵活性：可随时启用或禁用导入数据的包含
  - 可维护性：导入数据可独立管理（删除、更新等）
- 实现：
  - 查询时分别构建两个数据集的查询
  - 使用 SQL 连接操作合并结果
  - 根据查询类型选择合适的合并策略

### 5.4 格式兼容性保障机制

系统通过多层机制确保数据格式的兼容性：

**1. 严格的表结构定义**：
- 所有导入表都有明确的结构定义（`input_structures` 映射）
- 字段类型和顺序严格定义，确保数据正确解析

**2. 文件名格式验证**：
- 导入前验证文件名是否符合预期格式
- 提供 `valid_filename?/1` 函数进行格式检查
- 支持两种格式变体（带 `imported_` 前缀和不带前缀）

**3. 数据迁移框架**：
- 提供 `Plausible.DataMigration` 框架处理格式变更
- 支持版本化的 SQL 迁移脚本
- 提供回滚和确认机制，降低迁移风险

**4. 遗留数据支持**：
- 通过 `legacy` 标记区分旧格式导入
- 在查询和删除时特殊处理遗留数据
- 遗留导入使用 `import_id = 0` 作为特殊标识

## 6. 技术实现细节

### 6.1 核心数据结构

**1. 导入表结构**：
系统定义了以下导入表，每种表存储特定维度的聚合数据：

- `imported_visitors`：访客级别的聚合数据（按日期）
- `imported_sources`：流量来源数据
- `imported_pages`：页面访问数据
- `imported_entry_pages`：入口页面数据
- `imported_exit_pages`：出口页面数据
- `imported_custom_events`：自定义事件数据
- `imported_locations`：地理位置数据
- `imported_devices`：设备类型数据
- `imported_browsers`：浏览器数据
- `imported_operating_systems`：操作系统数据

**2. 查询维度到表的映射**：
系统建立了查询维度到导入表的映射关系，用于决定查询时使用哪些表：

```elixir
# 示例映射关系（简化）
@property_to_table_mappings = %{
  "visit:source" => "imported_sources",
  "visit:entry_page" => "imported_entry_pages",
  "visit:exit_page" => "imported_exit_pages",
  "event:page" => "imported_pages",
  "event:name" => "imported_custom_events",
  "visit:country" => "imported_locations",
  "visit:region" => "imported_locations",
  "visit:city" => "imported_locations",
  "visit:device" => "imported_devices",
  "visit:browser" => "imported_browsers",
  "visit:browser_version" => "imported_browsers",
  "visit:os" => "imported_operating_systems",
  "visit:os_version" => "imported_operating_systems"
}
```

### 6.2 关键算法与逻辑

**1. 空闲日期范围查找算法**：
系统使用递归算法查找可用的导入日期范围：

```elixir
defp free_ranges(import_range, d, [occupied_range | rest_of_occupied_ranges], result) do
  cond do
    # 已处理完此占用范围，继续下一个
    Date.diff(occupied_range.last, d) <= 0 ->
      free_ranges(import_range, d, rest_of_occupied_ranges, result)
    
    # 当前日期在占用范围内或过于接近，跳过此范围
    in_range?(d, occupied_range) || Date.diff(occupied_range.first, d) < 2 ->
      d = occupied_range.last
      free_ranges(import_range, d, rest_of_occupied_ranges, result)
    
    # 发现空闲范围，记录并继续
    true ->
      free_range = Date.range(d, occupied_range.first)
      result = result ++ [free_range]
      d = occupied_range.last
      free_ranges(import_range, d, rest_of_occupied_ranges, result)
  end
end
```

**2. 导入数据查询构建**：
系统根据查询参数动态构建导入数据查询：

- 基础查询：`query_imported/2` 函数根据站点和查询参数构建基础查询
- 表选择：根据查询维度和过滤器决定使用哪些导入表
- 过滤应用：应用日期范围、导入 ID 等过滤条件
- 聚合处理：按查询维度分组并计算聚合指标

**3. 数据合并逻辑**：
合并逻辑的核心是使用 SQL 的 `full join` 操作：

```elixir
# 简化的合并逻辑示例
from(s in subquery(native_q),
  full_join: i in subquery(imported_q),
  on: ^QueryBuilder.build_group_by_join(query),
  select: %{}
)
|> select_joined_dimensions(query)
|> select_joined_metrics(query)
```

- 维度合并：使用 `if(not empty(?), ?, ?)` 逻辑选择非空维度值
- 指标合并：使用 `? + ?` 逻辑累加两个数据源的指标值
- 排序：按合并后的指标值排序

### 6.3 性能优化策略

**1. 高基数维度分页优化**：
如前所述，对于页面路径等高基数维度，系统采用了有损但高效的分页优化：

- 限制：`LIMIT (pagination.limit + pagination.offset) * 100`
- 理由：减少需要连接的行数，显著提高查询性能
- 权衡：可能丢失一些边缘情况的准确结果，但在实践中提供合理的结果

**2. 预聚合数据存储**：
导入数据以预聚合的形式存储，而非原始事件数据：

- 优势：查询时无需实时聚合，显著提高性能
- 限制：无法进行原始事件级别的查询，只能进行聚合查询
- 适用场景：历史数据通常用于趋势分析和报表，聚合数据已足够

**3. 导入 ID 索引**：
所有导入表都使用 `import_id` 字段作为重要的索引：

- 查询时通过 `import_id` 快速筛选特定导入的数据
- 删除时通过 `import_id` 精准定位需要删除的数据
- 与 `site_id` 组合使用，形成高效的复合索引

## 7. 限制与注意事项

### 7.1 功能限制

**1. 导入数据查询限制**：
- 不支持分钟级和小时级时间粒度的查询
- 自定义属性过滤支持有限（仅 `url` 和 `path`）
- 某些复杂查询类型可能不支持导入数据

**2. 导入数量限制**：
- 最多保留 5 个完整的导入记录
- 同一时间只能执行一个导入作业

**3. 格式限制**：
- CSV 导入必须遵循严格的文件名格式
- 表结构必须与预定义结构完全匹配
- 不支持动态字段或自定义表结构

### 7.2 操作注意事项

**1. 数据删除不可逆**：
- ClickHouse 的数据删除是不可逆操作
- 删除后即使重新导入相同数据，也可能因去重机制问题导致异常（但系统已禁用导入表的去重）

**2. 异步删除延迟**：
- 数据删除不是即时生效的，需要等待后台 worker 执行
- 生产环境中 ClickHouse 的 mutations 是异步的，可能需要更长时间才能完全生效

**3. 导入数据与原生数据的细微差异**：
- 导入数据是预聚合的，可能与原生数据的实时聚合存在细微差异
- 某些高级分析功能可能只支持原生数据

### 7.3 升级与迁移注意事项

**1. 格式变更处理**：
- 使用 `DataMigration` 框架处理表结构变更
- 迁移前应充分测试，确保兼容性
- 考虑数据量，大型迁移可能需要分阶段执行

**2. 遗留数据处理**：
- 系统支持遗留数据，但新功能可能不完全兼容
- 建议在可能的情况下重新导入遗留数据
- 定期评估遗留数据的维护成本

## 8. 总结与建议

### 8.1 核心优势

1. **灵活的导入导出架构**：基于行为模式的设计，支持多种导入源，易于扩展
2. **统一的数据格式**：导入和导出使用相同格式，简化数据迁移流程
3. **查询时合并策略**：保留数据来源的可追溯性，同时提供统一的查询体验
4. **异步数据管理**：利用 ClickHouse 的特性，实现高效的批量数据操作
5. **完善的元数据管理**：详细记录导入信息，支持灵活的导入生命周期管理

### 8.2 潜在改进空间

1. **查询功能扩展**：
   - 支持更多查询类型的导入数据合并
   - 考虑支持更小时间粒度的导入数据查询

2. **性能优化**：
   - 对于超大规模数据，考虑更先进的分页优化策略
   - 探索使用 ClickHouse 的 materialized view 进一步优化导入数据查询

3. **管理功能增强**：
   - 提供导入数据的部分更新功能（当前只能全量删除和重导入）
   - 增强导入数据的监控和报告功能

4. **开发者体验**：
   - 提供更详细的导入格式文档和示例
   - 增强导入前的数据验证和错误提示

### 8.3 最佳实践建议

1. **导入策略**：
   - 合理规划导入时间范围，避免过于碎片化的导入
   - 优先使用较大时间粒度的导入，减少导入数量
   - 定期评估导入数据的实用性，清理不再需要的导入

2. **查询优化**：
   - 对于包含导入数据的查询，尽量使用系统支持的查询模式
   - 避免在高基数维度上进行无限制的查询
   - 合理使用分页参数，利用系统的分页优化

3. **数据管理**：
   - 定期备份重要的导入数据
   - 在删除导入或站点前，确认数据确实不再需要
   - 关注系统版本更新，及时了解数据格式变更

4. **监控与维护**：
   - 监控导入作业的执行状态和性能
   - 定期检查数据清理任务的执行情况
   - 对于大型系统，考虑单独监控 ClickHouse 的 mutations 队列

## 9. 旧数据访问路径深度分析

本节详细分析从查询请求进入到前端提示显示的完整数据访问路径，包括参数解析、导入数据判定、查询构建合并以及 skip reason 的传递机制。

### 9.1 参数解析流程

查询参数解析是整个数据访问路径的起点，由 `Plausible.Stats.ApiQueryParser` 模块负责。

#### 9.1.1 解析入口与流程

**1. 主解析函数 `parse/2`**
- 位置：`lib/plausible/stats/api_query_parser.ex:25-46`
- 流程：
  1. 首先通过 `JSONSchema.validate(params)` 进行 JSON Schema 验证
  2. 依次解析各个参数组件：
     - `parse_input_date_range/1`：解析日期范围
     - `parse_metrics/1`：解析指标列表
     - `parse_filters/1`：解析过滤条件
     - `parse_dimensions/1`：解析维度列表
     - `parse_order_by/1`：解析排序条件
     - `parse_pagination/1`：解析分页参数
     - `parse_include/1`：解析 include 参数（关键！）
  3. 构建 `ParsedQueryParams` 结构体

#### 9.1.2 Include 参数解析（关键路径）

**1. 默认 Include 配置**
```elixir
@default_include %Plausible.Stats.QueryInclude{
  imports: false,
  imports_meta: false,
  time_labels: false,
  total_rows: false,
  trim_relative_date_range: false,
  compare: nil,
  compare_match_day_of_week: false,
  legacy_time_on_page_cutoff: nil
}
```
- 位置：`lib/plausible/stats/api_query_parser.ex:8-17`
- **关键设计**：`imports` 默认值为 `false`，这意味着默认情况下不包含导入数据
- **设计取舍**：
  - **理由1（性能）**：导入数据合并需要额外的查询和连接操作，默认禁用可提高性能
  - **理由2（一致性）**：避免用户意外看到可能已过期的历史数据
  - **理由3（兼容性）**：保持与旧版本行为的一致性

**2. `parse_include/1` 函数**
- 位置：`lib/plausible/stats/api_query_parser.ex:299-326`
- 流程：
  1. 检查 include 参数是否为 map 类型
  2. 遍历 map 中的每个键值对
  3. 验证键是否在 `@allowed_include_keys` 列表中
  4. 将字符串键转换为原子键
  5. 使用 `struct!(@default_include, parsed_include_params)` 合并默认值与用户值

**3. 允许的 Include 键**
```elixir
@allowed_include_keys Enum.map(Map.keys(@default_include), &Atom.to_string/1)
```
- 包括：`imports`、`imports_meta`、`time_labels`、`total_rows` 等

**4. 前端如何请求包含导入数据**
- 前端需要在 API 请求中显式传递 `include: { imports: true }`
- 这通常由 `dashboardState.with_imported` 状态控制

### 9.2 导入数据判定逻辑

参数解析完成后，系统在 `QueryBuilder.build/3` 中构建查询并决定是否包含导入数据。

#### 9.2.1 Query 构建流程

**1. `QueryBuilder.build/3` 函数**
- 位置：`lib/plausible/stats/query_builder.ex:29-67`
- 关键步骤：
  1. 解析段（segments）过滤器
  2. 调用 `do_build/3` 构建基础查询
  3. 应用一系列验证：
     - `validate_order_by/1`：验证排序
     - `validate_custom_props_access/2`：验证自定义属性访问
     - `validate_case_sensitive_filter_modifier/1`：验证大小写敏感修饰符
     - `validate_toplevel_only_filter_dimension/1`：验证仅顶层维度
     - `validate_time_dimension_granularity/1`：验证时间维度粒度
     - `validate_special_metrics_filters/1`：验证特殊指标过滤器
     - `validate_behavioral_filters/1`：验证行为过滤器
     - `validate_filtered_goals_exist/2`：验证过滤的目标存在
     - `validate_revenue_metrics_access/2`：验证收入指标访问
     - `validate_metrics/1`：验证指标
     - `validate_include/1`：验证 include
  4. **关键步骤**：调用 `Query.put_imported_opts(query, site)` 设置导入选项
  5. （企业版）应用采样阈值

#### 9.2.2 导入数据判定核心逻辑

**1. `put_imported_opts/2` 函数**
- 位置：`lib/plausible/stats/query.ex:149-170`
- 这是决定是否包含导入数据的核心函数

**2. 完整判定流程**

```
┌─────────────────────────────────────────────────────────────────┐
│                    put_imported_opts/2 执行流程                   │
├─────────────────────────────────────────────────────────────────┤
│  1. 获取用户请求: requested? = query.include.imports              │
│     └─ 来自 API 参数 include.imports                               │
│                                                                 │
│  2. 检查站点参数: if site && schema_supports_interval?(query)    │
│     ├─ 分支A: 条件不满足 → 不处理导入数据                          │
│     │   └─ 可能原因: 无站点信息 或 时间间隔不支持                   │
│     │                                                             │
│     └─ 分支B: 条件满足 → 继续检查                                  │
│         ├─ 预加载站点的 completed_imports                          │
│         ├─ 设置 imports_exist = any_completed_imports?(site)     │
│         └─ 设置 imports_in_range = get_imports_in_range(site, query) │
│                                                                 │
│  3. 计算 skip_imported_reason = get_skip_imported_reason(query)  │
│     └─ 见下文详细分析                                               │
│                                                                 │
│  4. 最终决定:                                                      │
│     include_imported = requested? && is_nil(skip_imported_reason)│
│     └─ 只有用户请求 且 无跳过原因 才会包含导入数据                   │
└─────────────────────────────────────────────────────────────────┘
```

#### 9.2.3 Skip Reason 判定逻辑详解

**1. `get_skip_imported_reason/1` 函数**
- 位置：`lib/plausible/stats/query.ex:191-210`
- 使用 `cond` 宏按顺序检查条件，**第一个匹配的条件获胜**

**2. 四种 Skip Reason 详解**

| 优先级 | Skip Reason | 触发条件 | 设计含义 |
|--------|-------------|----------|----------|
| 1 (最高) | `:unsupported_interval` | `"time:minute" in query.dimensions or "time:hour" in query.dimensions` | 时间粒度过细，导入数据不支持 |
| 2 | `:no_imported_data` | `not query.imports_exist` | 站点根本没有已完成的导入 |
| 3 | `:out_of_range` | `query.imports_in_range == []` | 有导入但不在查询时间范围内 |
| 4 (最低) | `:unsupported_query` | `not Imported.schema_supports_query?(query)` | 查询类型不支持导入数据 |

**3. 各 Skip Reason 深度分析**

**Reason 1: `:unsupported_interval`（时间间隔不支持）**

- **触发条件**：查询维度包含 `time:minute` 或 `time:hour`
- **位置**：`lib/plausible/stats/query.ex:28-31` 的 `schema_supports_interval?/1` 函数
- **设计取舍**：
  - **为什么不支持**：
    - 导入数据是按天预聚合的（`imported_*` 表使用 `date` 字段）
    - 分钟级和小时级数据在导入时丢失了细粒度信息
    - 无法从日聚合数据反推出小时/分钟级数据
  - **边界情况**：
    - 查询 `time:day`、`time:week`、`time:month` 都支持
    - 但 `time:hour` 和 `time:minute` 直接拒绝
  - **用户提示**：
    ```elixir
    @imports_warnings %{
      unsupported_interval:
        "Imported stats are not included because the time dimension (i.e. the interval) is too short."
    }
    ```
    位置：`lib/plausible/stats/query_result.ex:18-24`

**Reason 2: `:no_imported_data`（无导入数据）**

- **触发条件**：`query.imports_exist` 为 false
- **如何设置 `imports_exist`**：
  ```elixir
  struct!(query,
    imports_exist: Plausible.Imported.any_completed_imports?(site),
    imports_in_range: get_imports_in_range(site, query)
  )
  ```
  位置：`lib/plausible/stats/query.ex:156-159`
- **`any_completed_imports?/1` 实现**：
  ```elixir
  def any_completed_imports?(site) do
    get_completed_imports(site) != []
  end
  ```
  位置：`lib/plausible/imported.ex:54-57`
- **设计含义**：
  - 快速检查是否有任何已完成的导入
  - 避免不必要的后续检查
  - 边界：只检查 `completed` 状态的导入，`pending` 和 `importing` 状态不算

**Reason 3: `:out_of_range`（导入数据超出范围）**

- **触发条件**：`query.imports_in_range == []`
- **如何设置 `imports_in_range`**：
  ```elixir
  defp get_imports_in_range(site, query) do
    in_range = Plausible.Imported.completed_imports_in_query_range(site, query)
    
    in_comparison_range =
      if query.include.compare do
        comparison_query = Comparisons.get_comparison_query(query)
        Plausible.Imported.completed_imports_in_query_range(site, comparison_query)
      else
        []
      end
    
    in_comparison_range ++ in_range
  end
  ```
  位置：`lib/plausible/stats/query.ex:177-189`
- **特殊情况**：实时查询 (`:realtime` 或 `:realtime_30m`) 直接返回空列表
  ```elixir
  defp get_imports_in_range(_site, %__MODULE__{input_date_range: period})
       when period in [:realtime, :realtime_30m] do
    []
  end
  ```
  位置：`lib/plausible/stats/query.ex:172-175`
- **设计取舍**：
  - 实时数据本质上是"现在"的数据，导入数据是历史数据
  - 两者时间范围不可能重叠，所以直接跳过
- **`completed_imports_in_query_range/2` 实现**：
  ```elixir
  def completed_imports_in_query_range(%Site{} = site, %Query{} = query) do
    date_range = Query.date_range(query)
    
    site
    |> get_completed_imports()
    |> Enum.reject(fn site_import ->
      Date.after?(site_import.start_date, date_range.last) or
        Date.before?(site_import.end_date, date_range.first)
    end)
  end
  ```
  位置：`lib/plausible/imported.ex:81-91`
- **判定逻辑**：
  - 导入的 `start_date` 在查询结束日期 **之后** → 超出范围（未来）
  - 导入的 `end_date` 在查询开始日期 **之前** → 超出范围（过去）
  - **否则** → 在范围内

**Reason 4: `:unsupported_query`（查询类型不支持）**

- **触发条件**：`not Imported.schema_supports_query?(query)`
- **位置**：`lib/plausible/stats/imported/imported.ex:24-26` 的 `schema_supports_query?/1` 函数
- **实现**：
  ```elixir
  def schema_supports_query?(query) do
    length(Imported.Base.decide_tables(query)) > 0
  end
  ```
- **核心**：调用 `decide_tables/1` 函数，如果返回空列表则不支持

#### 9.2.4 `decide_tables/1` 深度分析

这是决定查询类型是否支持导入数据的核心函数，位置：`lib/plausible/stats/imported/base.ex:77-208`

**1. 主决策流程**

```elixir
def decide_tables(query) do
  behavioral_filters = dimensions_used_in_filters(query.filters, behavioral_filters: :only)
  
  cond do
    # 条件1: 行为过滤器
    length(behavioral_filters) > 0 ->
      []
    
    # 条件2: 自定义属性查询
    custom_prop_query?(query) ->
      do_decide_custom_prop_table(query)
    
    # 条件3: 普通查询
    true ->
      do_decide_tables(query)
  end
end
```

**2. 条件1：行为过滤器（直接拒绝）**

- **什么是行为过滤器**：`has_done` 和 `has_not_done` 操作符
- **为什么拒绝**：
  - 行为过滤器需要检查用户会话的事件序列
  - 导入数据是预聚合的，丢失了原始事件序列信息
  - 无法从聚合数据判断"用户是否做了 X 然后做了 Y"
- **设计取舍**：
  - 这是一个功能限制，而非性能问题
  - 边界：只要有任何行为过滤器，整个查询都不支持导入数据

**3. 条件2：自定义属性查询（严格限制）**

- **什么是自定义属性查询**：
  - 维度或过滤器包含 `event:props:url` 或 `event:props:path`
  - 检查函数 `custom_prop_query?/1`：
    ```elixir
    defp custom_prop_query?(query) do
      dimensions_used_in_filters(query.filters)
      |> Enum.concat(query.dimensions)
      |> Enum.any?(&(&1 in @imported_custom_props))
    end
    ```
    位置：`lib/plausible/stats/imported/base.ex:93-97`

- **支持的自定义属性**：
  ```elixir
  def imported_custom_props do
    # NOTE: Keep up to date with `Plausible.Props.internal_keys/1`,
    # but _ignore_ unsupported keys. Currently, `search_query` is
    # not supported in imported queries.
    Enum.map(~w(url path), &("event:props:" <> &1))
  end
  ```
  位置：`lib/plausible/imported.ex:46-52`
- **注意**：`search_query` 不支持导入查询

- **`do_decide_custom_prop_table/1` 决策逻辑**：
  位置：`lib/plausible/stats/imported/base.ex:99-150`
  
  **必须同时满足两个条件**：
  
  **条件A：必须的事件/目标名称过滤器**
  ```elixir
  has_required_event_or_goal_name_filter? =
    query.filters
    |> Enum.flat_map(fn
      [:is, "event:name", event_names | _rest] -> event_names
      [:is, "event:goal", goal_names | _rest] -> goal_names
      _ -> []
    end)
    |> Enum.any?(fn event_or_goal_name ->
      event_or_goal_name in Plausible.Event.SystemEvents.special_events_for_prop_key(prop_key)
    end)
  ```
  - 必须有 `event:name` 或 `event:goal` 过滤器
  - 过滤值必须是"特殊事件"（与特定 prop 相关的系统事件）
  
  **条件B：无不受支持的过滤器**
  ```elixir
  has_unsupported_filters? =
    query.filters
    |> dimensions_used_in_filters()
    |> Enum.any?(&(&1 not in [property, "event:name", "event:goal"]))
  ```
  - 除了自定义属性本身、`event:name`、`event:goal` 之外，不能有其他过滤器

  **设计取舍**：
  - **为什么限制**：
    - 导入数据的自定义属性存储在 `imported_custom_events` 表的 `link_url` 和 `path` 字段
    - 这些字段只与特定事件类型相关
    - 无法支持任意组合的过滤
  - **边界**：
    - 只能是单维度查询（或无维度）
    - 维度只能是 `event:goal`、`event:name` 或时间维度

**4. 条件3：普通查询决策逻辑**

位置：`lib/plausible/stats/imported/base.ex:152-208`

**子情况A：无过滤器、无维度**
```elixir
defp do_decide_tables(%Query{filters: [], dimensions: []}), do: ["imported_visitors"]
```
- 使用 `imported_visitors` 表
- 这是最简单的汇总查询

**子情况B：无过滤器、单维度 `event:goal`**
```elixir
defp do_decide_tables(%Query{filters: [], dimensions: ["event:goal"]}) do
  ["imported_pages", "imported_custom_events"]
end
```
- 目标维度需要同时查询页面目标和事件目标
- 使用两个表：`imported_pages`（页面目标）和 `imported_custom_events`（事件目标）

**子情况C：有过滤器、单维度 `event:goal`**
位置：`lib/plausible/stats/imported/base.ex:158-180`

决策逻辑：
```elixir
filter_dimensions = dimensions_used_in_filters(query.filters)
filter_goals = query.preloaded_goals.matching_toplevel_filters

any_event_goals? = Enum.any?(filter_goals, fn goal -> Plausible.Goal.type(goal) == :event end)
any_pageview_goals? = Enum.any?(filter_goals, fn goal -> Plausible.Goal.type(goal) == :page end)

any_event_name_filters? = "event:name" in filter_dimensions or any_event_goals?
any_page_filters? = "event:page" in filter_dimensions or any_pageview_goals?

any_other_filters? = Enum.any?(filter_dimensions, &(&1 not in ["event:page", "event:name", "event:goal"]))

cond do
  any_other_filters? -> []  # 其他过滤器 → 不支持
  any_event_name_filters? and not any_page_filters? -> ["imported_custom_events"]
  any_page_filters? and not any_event_name_filters? -> ["imported_pages"]
  true -> []  # 混合或无 → 不支持
end
```

**设计取舍**：
- 页面目标和事件目标在导入数据中存储在不同的表
- 无法同时查询两种类型的目标（除非没有过滤器）
- 如果有其他类型的过滤器（如 `visit:source`），直接不支持

**子情况D：其他普通查询**
位置：`lib/plausible/stats/imported/base.ex:182-208`

决策逻辑：
```elixir
table_candidates =
  dimensions_used_in_filters(query.filters)
  |> Enum.concat(query.dimensions)
  |> Enum.reject(&(&1 in @queriable_time_dimensions or &1 == "event:goal"))
  |> Enum.flat_map(fn
    "visit:screen" -> ["visit:device"]  # screen 映射到 device
    dimension -> [dimension]
  end)
  |> Enum.map(&@property_to_table_mappings[&1])

filter_goal_table_candidates =
  query.preloaded_goals.matching_toplevel_filters
  |> Enum.map(&Plausible.Goal.type/1)
  |> Enum.map(fn
    :event -> "imported_custom_events"
    :page -> "imported_pages"
    :scroll -> nil
  end)

case Enum.uniq(table_candidates ++ filter_goal_table_candidates) do
  [] -> ["imported_visitors"]  # 无明确维度 → 使用 visitors 表
  [nil] -> []  # 只有滚动目标 → 不支持
  [candidate] -> [candidate]  # 单一表 → 支持
  _ -> []  # 多个表 → 不支持（无法跨表 JOIN 聚合数据）
end
```

**关键设计限制**：
- **单一表限制**：只能从一个导入表查询
- **为什么**：
  - 导入数据是按维度分离存储的（每个维度一个表）
  - 表之间没有通用的 JOIN 键（除了 date 和 site_id）
  - 跨维度聚合会导致数据重复计算
- **边界情况**：
  - `visit:screen` 映射到 `visit:device`，使用 `imported_devices` 表
  - 滚动目标（`:scroll`）不支持导入数据

### 9.3 查询构建与合并路径

当 `include_imported` 为 true 时，系统需要构建并合并原生数据和导入数据的查询。

#### 9.3.1 SQL 查询构建入口

**1. `SQL.QueryBuilder.build/2` 函数**
- 位置：`lib/plausible/stats/sql/query_builder.ex:17-28`
- 流程：
  1. 调用 `QueryOptimizer.split/1` 分割查询（可能生成多个子查询）
  2. 对每个子查询调用 `build_table_query/3`
  3. 调用 `join_query_results/2` 合并子查询结果
  4. 应用排序、分页、总行数选择

**2. `build_table_query/3` 函数**
- 位置：`lib/plausible/stats/sql/query_builder.ex:34-72`
- 这是实际构建单表查询的地方

**3. 事件表查询构建（`:events` 类型）**
位置：`lib/plausible/stats/sql/query_builder.ex:34-53`

```elixir
defp build_table_query(:events, site, events_query) do
  q =
    from(
      e in "events_v2",
      where: ^SQL.WhereBuilder.build(:events, events_query),
      where: ^SQL.WhereBuilder.derived_name_filter(events_query),
      select: ^select_event_metrics(events_query)
    )

  on_ee do
    q = Plausible.Stats.Sampling.add_query_hint(q, events_query)
  end

  q
  |> join_sessions_if_needed(events_query)
  |> build_group_by(:events, events_query)
  |> merge_imported(site, events_query)  # 关键：合并导入数据
  |> SQL.SpecialMetrics.add(site, events_query)
  |> TimeOnPage.merge_legacy_time_on_page(events_query)
end
```

**4. 会话表查询构建（`:sessions` 类型）**
位置：`lib/plausible/stats/sql/query_builder.ex:55-72`

类似事件表，也会调用 `merge_imported/3`

#### 9.3.2 `merge_imported/3` 合并逻辑详解

位置：`lib/plausible/stats/imported/imported.ex:218-311`

**1. 入口检查**
```elixir
def merge_imported(q, _, %Query{include_imported: false}), do: q
```
- 如果 `include_imported` 为 false，直接返回原生查询
- 这是一个快速退出路径

**2. 无维度查询（聚合查询）**
位置：`lib/plausible/stats/imported/imported.ex:220-235`

```elixir
def merge_imported(q, site, %Query{dimensions: []} = query) do
  q = paginate_optimization(q, query)

  imported_q =
    site
    |> Imported.Base.query_imported(query)
    |> select_imported_metrics(query)
    |> paginate_optimization(query)

  from(
    s in subquery(q),
    cross_join: i in subquery(imported_q),
    select: %{}
  )
  |> select_joined_metrics(query)
end
```

**设计要点**：
- 使用 `cross_join`（笛卡尔积）
- 因为两个查询都是单一行的聚合结果
- 指标通过 `select_joined_metrics/2` 相加

**3. 单维度 `event:goal` 查询（特殊处理）**
位置：`lib/plausible/stats/imported/imported.ex:237-287`

```elixir
def merge_imported(q, site, %Query{dimensions: ["event:goal"]} = query) do
  goal_join_data = Plausible.Stats.Goals.goal_join_data(query)

  Imported.Base.decide_tables(query)
  |> Enum.map(fn
    "imported_custom_events" ->
      # 处理自定义事件目标
      Imported.Base.query_imported("imported_custom_events", site, query)
      |> where([i], i.visitors > 0)
      |> select_merge_as([i], %{
        dim0: fragment("indexOf(?, ?)", type(^goal_join_data.event_names_imports, {:array, :string}), i.name)
      })
      |> select_imported_metrics(query)
      |> group_by([], selected_as(:dim0))
      |> where([], selected_as(:dim0) != 0)

    "imported_pages" ->
      # 处理页面目标
      Imported.Base.query_imported("imported_pages", site, query)
      |> where([i], i.visitors > 0)
      |> where(...)  # 复杂的页面正则匹配逻辑
      |> join(:inner, [_i], index in fragment("indices"), hints: "ARRAY", on: true)
      |> group_by([_i, index], index)
      |> select_merge_as([_i, index], %{dim0: fragment("CAST(?, 'UInt64')", index)})
      |> select_imported_metrics(query)
  end)
  |> Enum.reduce(q, fn imports_q, q ->
    naive_dimension_join(q, imports_q, query)
  end)
end
```

**设计要点**：
- 目标查询需要同时处理页面目标和事件目标
- 使用 `indexOf` 函数匹配目标名称到索引
- 页面目标使用正则表达式匹配
- 通过 `naive_dimension_join/3` 简单按维度值连接

**4. 普通维度查询**
位置：`lib/plausible/stats/imported/imported.ex:289-311`

```elixir
def merge_imported(q, site, query) do
  if schema_supports_query?(query) do
    q = paginate_optimization(q, query)

    imported_q =
      site
      |> Imported.Base.query_imported(query)
      |> where([i], i.visitors > 0)
      |> group_imported_by(query)
      |> select_imported_metrics(query)
      |> paginate_optimization(query)

    from(s in subquery(q),
      full_join: i in subquery(imported_q),
      on: ^QueryBuilder.build_group_by_join(query),
      select: %{}
    )
    |> select_joined_dimensions(query)
    |> select_joined_metrics(query)
  else
    q  # 不支持，返回原生查询
  end
end
```

**设计要点**：
- 使用 `full_join`（全连接）
- 连接条件由 `build_group_by_join/1` 构建（按所有维度值匹配）
- 维度和指标分别通过专门函数选择

#### 9.3.3 基础导入查询构建

**1. `query_imported/2` 函数**
位置：`lib/plausible/stats/imported/base.ex:56-75`

```elixir
def query_imported(site, query) do
  [table] = decide_tables(query)
  query_imported(table, site, query)
end

def query_imported(table, site, query) do
  import_ids = Imported.complete_import_ids(site)
  # Assumption: dates in imported table are in user-local timezone.
  %{first: date_from, last: date_to} = Query.date_range(query)

  from(i in table,
    where: i.site_id == ^site.id,
    where: i.import_id in ^import_ids,
    where: i.date >= ^date_from,
    where: i.date <= ^date_to,
    where: ^Plausible.Stats.Imported.SQL.WhereBuilder.build(query),
    select: %{}
  )
end
```

**关键过滤条件**：
1. `site_id`：站点匹配
2. `import_id`：只查询已完成的导入（包括遗留导入的 `import_id = 0`）
3. `date` 范围：查询时间范围内的数据
4. 额外的 WHERE 条件：由 `WhereBuilder.build/1` 生成

**2. `complete_import_ids/1` 函数**
位置：`lib/plausible/imported.ex:67-79`

```elixir
def complete_import_ids(site) do
  imports = get_completed_imports(site)
  has_legacy? = Enum.any?(imports, fn %{legacy: legacy?} -> legacy? end)
  ids = Enum.map(imports, fn %{id: id} -> id end)

  # account for legacy imports as well
  if has_legacy? do
    [0 | ids]
  else
    ids
  end
end
```

**设计要点**：
- 遗留导入（`legacy: true`）使用特殊的 `import_id = 0`
- 这是为了兼容旧版本系统中没有 `import_id` 字段的导入数据

#### 9.3.4 维度和指标选择

**1. 导入数据分组**
位置：`lib/plausible/stats/imported/sql/expression.ex:187-194`

```elixir
def group_imported_by(q, query) do
  Enum.reduce(query.dimensions, q, fn dimension, q ->
    q
    |> select_group_fields(dimension, shortname(query, dimension), query)
    |> filter_group_values(dimension)
    |> group_by([], selected_as(^shortname(query, dimension)))
  end)
end
```

**2. 维度字段选择**
位置：`lib/plausible/stats/imported/sql/expression.ex:196-304`

不同维度有不同的处理逻辑：
- `visit:source/referrer`：空值替换为 `"Direct / None"`
- `event:page`：直接使用 `page` 字段
- `visit:device/browser/channel`：空值替换为 `"(not set)"`
- UTM 维度：过滤空值
- 地理位置：过滤无效值（`"ZZ"` 国家、空地区、0 城市）
- 时间维度：
  - `time:month`：`toStartOfMonth(date)`
  - `time:week`：`weekstart_not_before` 函数
  - `time:day/hour`：直接使用 `date` 字段

**3. 指标选择**
位置：`lib/plausible/stats/imported/sql/expression.ex:18-186`

**设计要点**：
- 不同的表可能有不同的字段名
- 例如：
  - `imported_exit_pages` 的访问量使用 `exits` 字段
  - `imported_entry_pages` 的访问量使用 `entrances` 字段
  - 普通表使用 `visits` 字段
- 指标通常使用 `sum()` 聚合

**4. 合并维度和指标**
位置：`lib/plausible/stats/imported/sql/expression.ex:306-466`

**维度合并**：
```elixir
defp select_joined_dimension(q, _dimension, key) do
  select_merge_as(q, [s, i], %{
    key => fragment("if(empty(?), ?, ?)", field(s, ^key), field(i, ^key), field(s, ^key))
  })
end
```
- 优先使用原生数据的维度值（如果非空）
- 否则使用导入数据的维度值

**指标合并**：
```elixir
defp joined_metric(:visitors, _query) do
  wrap_alias([s, i], %{visitors: s.visitors + i.visitors})
end
```
- 简单相加两个数据源的指标值
- 比率指标（bounce_rate、visit_duration 等）需要复杂的加权计算

#### 9.3.5 分页优化（高基数维度）

位置：`lib/plausible/stats/imported/imported.ex:331-372`

**1. 优化条件**：
```elixir
defp paginate_optimization(q, query) do
  if is_map(query.pagination) and can_order_by?(query) do
    n = (query.pagination.limit + query.pagination.offset) * 100
    
    q
    |> QueryBuilder.build_order_by(query)
    |> limit(^n)
  else
    q
  end
end
```

**2. 无法优化的指标**：
```elixir
@cannot_optimize_metrics [
  :exit_rate,
  :scroll_depth,
  :percentage,
  :bounce_rate,
  :conversion_rate,
  :group_conversion_rate,
  :time_on_page
]
```

**设计取舍**：
- **为什么优化**：
  - 高基数维度（如页面路径）可能有数百万条记录
  - FULL JOIN 两个大表非常慢
- **如何优化**：
  - 分别对两个数据集应用 `LIMIT N * 100`
  - 只合并前 N 项
- **权衡**：
  - 这是有损优化
  - 真正的前 N 项可能来自任一子查询的前 C 项之外
  - 但在实践中，前几项通常在两个数据源中都排名靠前

### 9.4 Skip Reason 到前端的传递路径

当 `skip_imported_reason` 被设置后，它需要通过多层传递最终到达前端并显示给用户。

#### 9.4.1 查询结果元数据构建

**1. `QueryResult.from/1` 函数**
- 位置：`lib/plausible/stats/query_result.ex:47-55`
- 这是构建 API 响应的入口

**2. `meta/1` 函数**
位置：`lib/plausible/stats/query_result.ex:57-71`

```elixir
defp meta(%QueryRunner{} = runner) do
  %{}
  |> add_imports_meta(runner.main_query)  # 关键：添加导入元数据
  |> add_metric_warnings_meta(runner.main_query)
  |> add_empty_metrics_meta(runner.main_query)
  |> add_time_labels_meta(runner)
  # ... 其他元数据
  |> Enum.sort_by(&elem(&1, 0))
end
```

**3. `add_imports_meta/2` 函数**
位置：`lib/plausible/stats/query_result.ex:73-85`

```elixir
defp add_imports_meta(meta, %Query{include: include} = query) do
  if include.imports or include.imports_meta do
    %{
      imports_included: query.include_imported,
      imports_skip_reason: query.skip_imported_reason,
      imports_warning: @imports_warnings[query.skip_imported_reason]
    }
    |> Map.reject(fn {_key, value} -> is_nil(value) end)
    |> Map.merge(meta)
  else
    meta
  end
end
```

**设计要点**：
- 只有当 `include.imports` 或 `include.imports_meta` 为 true 时才添加这些元数据
- `imports_warning` 是预定义的用户友好消息
- 使用 `Map.reject` 过滤掉 `nil` 值

**4. 预定义警告消息**
位置：`lib/plausible/stats/query_result.ex:18-24`

```elixir
@imports_warnings %{
  unsupported_query:
    "Imported stats are not included in the results because query parameters are not supported. " <>
      "For more information, see: https://plausible.io/docs/stats-api#filtering-imported-stats",
  unsupported_interval:
    "Imported stats are not included because the time dimension (i.e. the interval) is too short."
}
```

**注意**：
- `:no_imported_data` 和 `:out_of_range` 没有预定义警告消息
- 这是设计决定：这些是"正常"情况，不需要警告用户
- 只有 `:unsupported_query` 和 `:unsupported_interval` 需要显示警告

#### 9.4.2 API Controller 层传递

**1. 统一查询 API（`query/2`）**
位置：`lib/plausible_web/controllers/api/stats_controller.ex:40-57`

```elixir
def query(conn, params) do
  site = conn.assigns.site
  now = conn.private[:now]

  with {:ok, %ParsedQueryParams{} = params} <- Dashboard.QueryParser.parse(params, now: now),
       {:ok, %Query{} = query} <- QueryBuilder.build(site, params, debug_metadata(conn)) do
    query =
      if query.include.time_labels do
        Query.set_include(query, :time_label_result_indices, true)
      else
        query
      end

    json(conn, Plausible.Stats.query(site, query))  # 直接返回 QueryResult
  else
    {:error, %QueryError{message: message}} -> bad_request(conn, message)
  end
end
```

- 统一 API 直接返回 `QueryResult`，包含完整的 `meta` 字段

**2. 细分统计 API（如 `sources/2`）**
位置：`lib/plausible_web/controllers/api/stats_controller.ex:59-97`

```elixir
def sources(conn, params) do
  # ... 构建查询 ...
  
  %{results: results, meta: meta} = Stats.breakdown(site, query, metrics, pagination)
  
  # ... 转换结果 ...
  
  if params["csv"] do
    # ... CSV 导出 ...
  else
    json(conn, %{
      results: res,
      meta: Stats.Breakdown.formatted_date_ranges(query),
      skip_imported_reason: meta[:imports_skip_reason]  # 显式提取
    })
  end
end
```

**设计要点**：
- 细分 API 不返回完整的 `meta` 对象
- 只显式提取 `meta[:imports_skip_reason]` 字段
- 这是为了保持 API 响应格式的兼容性
- **边界**：细分 API 不传递 `imports_warning`，只传递 `skip_imported_reason`

**3. 所有细分 API 的一致性**
在 `stats_controller.ex` 中，以下函数都有相同的模式：
- `sources/2`（第 94 行）
- `channels/2`（第 134 行）
- `pages/2`（第 344 行）
- `entry_pages/2`（第 379 行）
- `exit_pages/2`（第 414 行）
- `countries/2`（第 449 行）
- `regions/2`（第 484 行）
- `cities/2`（第 519 行）
- `devices/2`（第 652 行）
- `browsers/2`（第 695 行）
- `operating_systems/2`（第 744 行）
- `custom_events/2`（第 857 行）
- `conversions/2`（第 908 行）
- `props/2`（第 946 行）
- `goals/2`（第 993 行）
- `utm_mediums/2`（第 1031 行）
- `utm_sources/2`（第 1078 行）
- `utm_campaigns/2`（第 1116 行）
- `utm_contents/2`（第 1154 行）
- `utm_terms/2`（第 1232 行）

**模式**：
```elixir
json(conn, %{
  results: ...,
  meta: ...,
  skip_imported_reason: meta[:imports_skip_reason]
})
```

#### 9.4.3 外部 API Controller 传递

**1. `external_stats_controller.ex` 中的处理**
位置：`lib/plausible_web/controllers/api/external_stats_controller.ex:264-381`

```elixir
# 处理 imports_warning
case meta[:imports_warning] do
  nil -> conn
  warning -> put_resp_header(conn, "x-warning", warning)
end

# 处理 imports_skip_reason
case meta[:imports_skip_reason] do
  nil -> conn
  reason -> put_resp_header(conn, "x-imports-skip-reason", Atom.to_string(reason))
end
```

**设计要点**：
- 外部 API 使用 HTTP 响应头传递元数据
- `x-warning` 头包含用户友好的警告消息
- `x-imports-skip-reason` 头包含机器可读的原因代码
- 这是外部 API 与内部 API 的主要区别

#### 9.4.4 前端接收与处理

**1. API 响应解析**
以 `sources/index.js` 为例，位置：`assets/js/dashboard/stats/sources/index.js:184-248`

```javascript
const afterFetchData = useCallback((apiResponse) => {
  setLoading(false)
  if (apiResponse) {
    setSkipImportedReason(apiResponse.skip_imported_reason)  // 提取 skip reason
    if (apiResponse.results && apiResponse.results.length > 0) {
      setMoreLinkState(MoreLinkState.READY)
    } else {
      setMoreLinkState(MoreLinkState.HIDDEN)
    }
  } else {
    setLoading(false)
    setMoreLinkState(MoreLinkState.HIDDEN)
  }
}, [])
```

**2. 状态管理**
```javascript
const [skipImportedReason, setSkipImportedReason] = useState(null)
```

**3. 警告组件显示**
位置：`assets/js/dashboard/stats/imported-query-unsupported-warning.js`

```javascript
export default function ImportedQueryUnsupportedWarning({
  loading,
  skipImportedReason,
  altCondition,
  message
}) {
  const { dashboardState } = useDashboardStateContext()
  const portalRef = useRef(null)
  const tooltipMessage =
    message || 'Imported data is excluded due to applied filters'
  
  const show =
    dashboardState &&
    dashboardState.with_imported &&  # 用户请求了导入数据
    skipImportedReason === 'unsupported_query' &&  # 原因是 unsupported_query
    dashboardState.period !== 'realtime'  # 不是实时查询

  // ... 渲染逻辑
}
```

**设计要点**：
- **只显示 `unsupported_query` 原因**：
  - `:no_imported_data`：用户没有导入数据，不需要警告
  - `:out_of_range`：用户选择的时间范围内没有导入数据，正常情况
  - `:unsupported_interval`：时间粒度过细，前端通常有其他提示
  - `:unsupported_query`：查询参数不支持，需要提示用户
- **需要用户请求了导入数据**：`dashboardState.with_imported` 必须为 true
- **不是实时查询**：实时查询本来就不支持导入数据

**4. 渲染逻辑**
```javascript
if (show || altCondition) {
  return (
    <FadeIn show={!loading} className="h-4.5">
      <Tooltip info={tooltipMessage} containerRef={portalRef}>
        <ExclamationCircleIcon className="mb-1 size-4.5 text-gray-500 dark:text-gray-400" />
      </Tooltip>
    </FadeIn>
  )
} else {
  return null
}
```

- 显示一个感叹号图标
- 鼠标悬停时显示工具提示消息
- 默认消息：`"Imported data is excluded due to applied filters"`

### 9.5 关键分支设计取舍与边界情况总结

#### 9.5.1 设计取舍总览

| 决策点 | 选择 | 理由 | 权衡 |
|--------|------|------|------|
| 默认 `imports: false` | 禁用 | 性能、一致性、兼容性 | 用户需要显式请求 |
| 预聚合存储 | 按维度分表 | 查询性能 | 无法跨维度 JOIN |
| 单一表限制 | 只查一个表 | 数据正确性 | 限制查询灵活性 |
| 分页优化 | LIMIT N*100 | 性能 | 有损，可能丢失边缘项 |
| 行为过滤器 | 不支持 | 数据特性 | 功能限制 |
| 自定义属性 | 严格限制 | 数据存储方式 | 功能限制 |
| skip reason 显示 | 只显示 unsupported_query | 用户体验 | 其他原因无提示 |

#### 9.5.2 边界情况详解

**1. 实时查询边界**
```elixir
defp get_imports_in_range(_site, %__MODULE__{input_date_range: period})
     when period in [:realtime, :realtime_30m] do
  []
end
```
- 实时查询永远不会包含导入数据
- 这是设计决定：实时数据是"现在"的，导入数据是历史的

**2. 遗留导入边界**
```elixir
if has_legacy? do
  [0 | ids]
else
  ids
end
```
- 遗留导入使用 `import_id = 0`
- 这是为了兼容旧版本系统
- 边界：在 `delete_imported_stats!/2` 中也有特殊处理

**3. 目标查询边界**
- 无过滤器 + `event:goal` 维度 → 支持（两个表）
- 有过滤器 + `event:goal` 维度 → 只能是纯页面或纯事件目标
- 混合目标类型 → 不支持

**4. 时间维度边界**
- `time:day/week/month` → 支持
- `time:hour/minute` → 不支持
- 边界：`schema_supports_interval?/1` 是第一个检查条件

**5. 分页优化边界**
- 有分页参数 + 可排序 → 应用优化
- 无可排序指标 → 不应用优化
- 比率指标（bounce_rate 等）→ 不应用优化

#### 9.5.3 完整数据流图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         旧数据访问完整数据流                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. 用户操作                                                                 │
│     └─ 前端设置 dashboardState.with_imported = true                         │
│                                                                             │
│  2. API 请求                                                                 │
│     └─ 传递 include: { imports: true } 参数                                 │
│                                                                             │
│  3. 参数解析 (ApiQueryParser)                                               │
│     ├─ parse_include/1 → 验证并转换参数                                    │
│     └─ @default_include 中 imports: false 被覆盖为 true                    │
│                                                                             │
│  4. 查询构建 (QueryBuilder)                                                 │
│     ├─ do_build/3 → 构建基础 Query 结构体                                   │
│     └─ put_imported_opts/2 → 关键决策点                                    │
│         ├─ 检查 schema_supports_interval? → time:hour/minute?              │
│         ├─ 检查 imports_exist → 有已完成导入？                              │
│         ├─ 检查 imports_in_range → 导入在查询范围内？                        │
│         ├─ 检查 schema_supports_query? → decide_tables 返回空？             │
│         └─ 设置 skip_imported_reason 和 include_imported                    │
│                                                                             │
│  5. SQL 构建 (SQL.QueryBuilder)                                             │
│     ├─ build_table_query → 构建原生数据查询                                 │
│     └─ merge_imported/3 → 条件合并                                          │
│         ├─ include_imported == false → 直接返回                              │
│         └─ include_imported == true → FULL JOIN 两个子查询                   │
│             ├─ 构建 imported_q (imported_* 表)                              │
│             ├─ 应用 paginate_optimization (如有)                            │
│             └─ select_joined_dimensions + select_joined_metrics            │
│                                                                             │
│  6. 结果构建 (QueryResult)                                                   │
│     └─ add_imports_meta/2 → 添加到 meta 字段                                │
│         ├─ imports_included: true/false                                     │
│         ├─ imports_skip_reason: reason | nil                                │
│         └─ imports_warning: message | nil                                    │
│                                                                             │
│  7. API 响应 (StatsController)                                              │
│     ├─ 统一 API → 直接返回完整 meta                                          │
│     └─ 细分 API → 只提取 skip_imported_reason                               │
│                                                                             │
│  8. 前端处理                                                                 │
│     ├─ afterFetchData → 保存 skipImportedReason 状态                        │
│     └─ ImportedQueryUnsupportedWarning → 条件显示                           │
│         ├─ 检查 with_imported == true                                       │
│         ├─ 检查 skipImportedReason == 'unsupported_query'                   │
│         └─ 显示感叹号图标 + 工具提示                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 10. 兼容分流深度分析：Legacy 入口、Compare 场景与提示一致性

本节深入分析系统中存在的三条兼容分流路径：Legacy 入口与当前入口的参数差异、Compare 场景下导入范围合并的边界情况、以及 Skip Reason 在不同统计面板提示不一致的原因。

### 10.1 Legacy 入口与当前入口的分流差异

系统中存在两条独立的查询入口路径，分别服务于不同的 API 端点和客户端。

#### 10.1.1 两条入口路径概述

| 入口类型 | 入口函数 | 使用场景 | 参数来源 |
|----------|----------|----------|----------|
| **Legacy 入口** | `Query.from/3` → `Legacy.QueryBuilder.from/4` | 细分 API（sources、channels、pages 等）、外部 Stats API v1 | URL 查询参数（旧格式） |
| **当前入口** | `Query.parse_and_build/3` → `Dashboard.QueryParser.parse/2` → `QueryBuilder.build/3` | 统一 `query/2` API | JSON Body 参数（新格式） |

#### 10.1.2 Legacy 入口参数解析

**入口位置**：`lib/plausible/stats/query.ex:74-82`

```elixir
def from(site, params, opts \\ []) do
  Legacy.QueryBuilder.from(
    site,
    params,
    Keyword.get(opts, :debug_metadata, %{}),
    Keyword.get(opts, :now)
  )
end
```

**使用 Legacy 入口的 API 端点**（`lib/plausible_web/controllers/api/stats_controller.ex`）：

| 端点 | 行号 | 调用方式 |
|------|------|----------|
| `sources/2` | 第 62 行 | `Query.from(site, params, ...)` |
| `channels/2` | 第 102 行 | `Query.from(site, params, ...)` |
| `pages/2` | 第 352 行 | `Query.from(site, params, ...)` |
| `entry_pages/2` | 第 387 行 | `Query.from(site, params, ...)` |
| `exit_pages/2` | 第 422 行 | `Query.from(site, params, ...)` |
| `countries/2` | 第 457 行 | `Query.from(site, params, ...)` |
| `regions/2` | 第 492 行 | `Query.from(site, params, ...)` |
| `cities/2` | 第 527 行 | `Query.from(site, params, ...)` |
| `devices/2` | 第 583 行 | `Query.from(site, params, ...)` |
| `browsers/2` | 第 616 行 | `Query.from(site, params, ...)` |
| `operating_systems/2` | 第 660 行 | `Query.from(site, params, ...)` |
| `custom_events/2` | 第 752 行 | `Query.from(site, params, ...)` |
| `conversions/2` | 第 819 行 | `Query.from(site, params, ...)` |
| `props/2` | 第 865 行 | `Query.from(site, params, ...)` |
| `goals/2` | 第 916 行 | `Query.from(site, params, ...)` |
| `utm_mediums/2` | 第 954 行 | `Query.from(site, params, ...)` |
| `utm_sources/2` | 第 1001 行 | `Query.from(site, params, ...)` |
| `utm_campaigns/2` | 第 1039 行 | `Query.from(site, params, ...)` |
| `utm_contents/2` | 第 1086 行 | `Query.from(site, params, ...)` |
| `utm_terms/2` | 第 1134 行 | `Query.from(site, params, ...)` |

**外部 Stats API v1** 也使用 Legacy 入口（`lib/plausible_web/controllers/api/external_stats_controller.ex`）：
- 第 19、39、257 行均调用 `Query.from(site, params, ...)`

#### 10.1.3 当前入口参数解析

**入口位置**：`lib/plausible/stats/query.ex:50-59`

```elixir
def parse_and_build(
      %Plausible.Site{domain: domain} = site,
      %{"site_id" => domain} = params,
      opts \\ []
    ) do
  with {:ok, %ParsedQueryParams{} = parsed_query_params} <-
         ApiQueryParser.parse(params, opts) do
    QueryBuilder.build(site, parsed_query_params, Keyword.get(opts, :debug_metadata, %{}))
  end
end
```

**使用当前入口的 API 端点**：
- `stats_controller.ex` 中的 `query/2`（第 40-57 行）：
  ```elixir
  def query(conn, params) do
    with {:ok, %ParsedQueryParams{} = params} <- Dashboard.QueryParser.parse(params, now: now),
         {:ok, %Query{} = query} <- QueryBuilder.build(site, params, debug_metadata(conn)) do
      json(conn, Plausible.Stats.query(site, query))
    end
  end
  ```

#### 10.1.4 关键参数差异：`with_imported` vs `include.imports`

这是两条入口最关键的差异：

**Legacy 入口**（`lib/plausible/stats/legacy/legacy_query_builder.ex:248-256`）：

```elixir
defp put_include(query, params) do
  include = parse_include(params["include"])

  query
  |> struct!(include: include)
  |> Query.set_include(:compare, parse_include_compare(params))
  |> Query.set_include(:compare_match_day_of_week, params["match_day_of_week"] == "true")
  |> Query.set_include(:imports, params["with_imported"] == "true")  # 关键：使用 with_imported 参数
end
```

- **参数名**：`with_imported`（字符串类型）
- **值比较**：`params["with_imported"] == "true"`
- **来源**：URL 查询参数，如 `?with_imported=true`

**当前入口**（`lib/plausible/stats/dashboard/query_parser.ex:76-93`）：

```elixir
defp parse_include(params) do
  with {:ok, compare} <- parse_include_compare(params["include"]) do
    {:ok,
     %QueryInclude{
       imports: params["include"]["imports"] == true,  # 关键：使用 include.imports
       imports_meta: params["include"]["imports_meta"] == true,
       compare: compare,
       compare_match_day_of_week: params["include"]["compare_match_day_of_week"] == true,
       time_labels: params["include"]["time_labels"] == true,
       # ...
     }}
  end
end
```

- **参数名**：`include.imports`（嵌套在 `include` 对象中）
- **值比较**：`params["include"]["imports"] == true`（布尔值）
- **来源**：JSON Body，如 `{ "include": { "imports": true } }`

#### 10.1.5 设计取舍与影响

| 维度 | Legacy 入口 | 当前入口 | 设计取舍 |
|------|-------------|----------|----------|
| **参数格式** | URL 查询字符串（扁平） | JSON Body（结构化） | 向后兼容 vs 现代 API 设计 |
| **布尔值表示** | 字符串 `"true"` | 布尔值 `true` | URL 参数限制 vs 类型安全 |
| **使用场景** | 细分 API、外部 API v1 | 统一 `query` API | 渐进式迁移策略 |
| **前端调用** | 旧版 Dashboard 组件 | 新版 Dashboard 组件 | 双轨制运行 |

**潜在风险**：
1. **参数名称混淆**：开发者可能混淆 `with_imported` 和 `include.imports`
2. **类型不匹配**：Legacy 入口使用字符串比较，当前入口使用布尔值比较
3. **测试覆盖**：两条路径需要分别测试，增加维护成本

**设计意图**：
- 保持向后兼容性：现有 API 客户端无需修改
- 渐进式迁移：新功能使用当前入口，旧功能保持 Legacy 入口
- 统一最终逻辑：两条入口最终都调用 `Query.put_imported_opts/2` 进行相同的导入数据判定

### 10.2 Compare 场景下导入范围合并的边界

当用户启用"同期对比"功能时，系统需要同时考虑主查询时间范围和比较查询时间范围内的导入数据。

#### 10.2.1 导入范围合并逻辑

**核心实现**：`lib/plausible/stats/query.ex:177-189`

```elixir
defp get_imports_in_range(site, query) do
  in_range = Plausible.Imported.completed_imports_in_query_range(site, query)
  
  in_comparison_range =
    if query.include.compare do
      comparison_query = Comparisons.get_comparison_query(query)
      Plausible.Imported.completed_imports_in_query_range(site, comparison_query)
    else
      []
    end
  
  in_comparison_range ++ in_range  # 合并两个范围的导入
end
```

**流程拆解**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Compare 场景导入范围合并流程                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. 获取主查询范围内的导入                                                    │
│     └─ in_range = completed_imports_in_query_range(site, query)            │
│                                                                             │
│  2. 检查是否启用比较功能                                                      │
│     └─ if query.include.compare do ...                                      │
│                                                                             │
│  3. 构建比较查询                                                             │
│     └─ comparison_query = Comparisons.get_comparison_query(query)          │
│         ├─ 复制原查询的大部分属性                                            │
│         └─ 替换 utc_time_range 为比较时间范围                                │
│                                                                             │
│  4. 获取比较查询范围内的导入                                                  │
│     └─ in_comparison_range = completed_imports_in_query_range(...)          │
│                                                                             │
│  5. 合并两个范围的导入                                                        │
│     └─ in_comparison_range ++ in_range                                      │
│         └─ 使用列表拼接操作符 ++，保持顺序                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 10.2.2 比较时间范围计算

**核心函数**：`lib/plausible/stats/comparisons.ex:50-72`

```elixir
def get_comparison_utc_time_range(%Stats.Query{} = source_query) do
  datetime_range =
    case source_query.include.compare do
      {:datetime_range, from, to} ->
        DateTimeRange.new!(from, to)
      
      _ ->
        if use_datetime_for_comparison?(source_query) do
          get_comparison_datetime_range(source_query)  # 24h 或当天的精确时间
        else
          comparison_date_range = get_comparison_date_range(source_query)
          DateTimeRange.new!(
            comparison_date_range.first,
            comparison_date_range.last,
            source_query.timezone
          )
        end
    end
  
  DateTimeRange.to_timezone(datetime_range, "Etc/UTC")
end
```

**比较模式**：

| 比较模式 | 参数值 | 计算方式 |
|----------|--------|----------|
| **上一周期** | `:previous_period` | 按天数向前偏移相同天数 |
| **同比去年** | `:year_over_year` | 向前偏移 1 年 |
| **自定义范围** | `{:date_range, from, to}` | 使用用户指定的日期范围 |

#### 10.2.3 四种边界情况分析

**情况 1：主查询有导入，比较查询无导入**

```
主查询范围：2024-01-01 ~ 2024-01-31
比较范围：  2023-12-01 ~ 2023-12-31

导入数据：
  导入 A：2024-01-01 ~ 2024-01-15 ✓（主查询范围内）
  导入 B：2023-11-01 ~ 2023-11-30 ✗（比较范围外）

合并结果：[导入 A]
skip_imported_reason：不会触发（因为有导入在范围内）
```

**情况 2：主查询无导入，比较查询有导入**

```
主查询范围：2024-01-01 ~ 2024-01-31
比较范围：  2023-12-01 ~ 2023-12-31

导入数据：
  导入 A：2023-12-10 ~ 2023-12-20 ✓（比较范围内）
  导入 B：2024-02-01 ~ 2024-02-28 ✗（主查询范围外）

合并结果：[导入 A]
skip_imported_reason：不会触发
```

**情况 3：两者都有导入**

```
主查询范围：2024-01-01 ~ 2024-01-31
比较范围：  2023-12-01 ~ 2023-12-31

导入数据：
  导入 A：2024-01-01 ~ 2024-01-15 ✓（主查询范围内）
  导入 B：2023-12-10 ~ 2023-12-20 ✓（比较范围内）

合并结果：[导入 B, 导入 A]（++ 操作符保持顺序）
skip_imported_reason：不会触发
```

**情况 4：两者都无导入**

```
主查询范围：2024-01-01 ~ 2024-01-31
比较范围：  2023-12-01 ~ 2023-12-31

导入数据：
  导入 A：2022-01-01 ~ 2022-12-31 ✗（两个范围都不包含）

合并结果：[]
skip_imported_reason：`:out_of_range`
```

#### 10.2.4 特殊边界：实时查询

**代码位置**：`lib/plausible/stats/query.ex:172-175`

```elixir
defp get_imports_in_range(_site, %__MODULE__{input_date_range: period})
     when period in [:realtime, :realtime_30m] do
  []  # 实时查询直接返回空列表
end
```

**设计含义**：
- 实时查询（`realtime` 和 `realtime_30m`）永远不会包含导入数据
- 这是语义上的合理限制：实时数据是"现在"的，导入数据是历史的
- 即使启用 Compare 功能，实时查询的比较范围也不会包含导入数据

#### 10.2.5 设计取舍与影响

| 设计决策 | 选择 | 理由 | 权衡 |
|----------|------|------|------|
| **合并策略** | `in_comparison_range ++ in_range` | 确保两个范围的导入都被考虑 | 可能导入数据跨两个范围，但不会重复计算 |
| **优先级** | 比较范围在前，主范围在后 | 列表顺序不影响 `imports_in_range == []` 判断 | 顺序对去重无影响（使用 `++` 而非 `MapSet.union`） |
| **去重策略** | 无显式去重 | `completed_imports_in_query_range` 已返回唯一导入 | 如果导入同时在两个范围，会出现重复？ |
| **实时查询** | 直接返回 `[]` | 语义清晰：实时 ≠ 历史 | 即使 Compare 也不例外 |

**潜在问题分析**：

```elixir
# 假设存在一个导入同时覆盖主范围和比较范围
导入 C：2023-12-15 ~ 2024-01-15

主查询范围：2024-01-01 ~ 2024-01-31 → 包含导入 C
比较范围：  2023-12-01 ~ 2023-12-31 → 包含导入 C

# 合并结果：[导入 C, 导入 C]（重复！）
```

**实际影响**：
- `imports_in_range` 用于两个地方：
  1. `:out_of_range` 判断：`query.imports_in_range == []` → 重复不影响（列表非空）
  2. `complete_import_ids/1`：已通过 `Enum.uniq()` 去重（见下文）

**去重保障**：`lib/plausible/imported.ex:67-79`

```elixir
def complete_import_ids(site) do
  imports = get_completed_imports(site)
  has_legacy? = Enum.any?(imports, fn %{legacy: legacy?} -> legacy? end)
  ids = Enum.map(imports, fn %{id: id} -> id end)  # 从 SiteImport 记录获取，已去重
  
  if has_legacy? do
    [0 | ids]
  else
    ids
  end
end
```

- `complete_import_ids/1` 从 `SiteImport` 记录获取 ID，天然去重
- `imports_in_range` 中的重复只影响 `:out_of_range` 判断（非空即有效）
- 实际 SQL 查询使用 `import_id in ^import_ids`，不会重复

### 10.3 Skip Reason 在不同统计面板提示不一致的原因

前端系统中，不同统计面板对 `skip_imported_reason` 的显示逻辑存在差异，导致用户体验不一致。

#### 10.3.1 核心显示逻辑

**警告组件**：`assets/js/dashboard/stats/imported-query-unsupported-warning.js`

```javascript
const show =
  dashboardState &&
  dashboardState.with_imported &&              // 条件1：用户请求了导入数据
  skipImportedReason === 'unsupported_query' && // 条件2：原因必须是 unsupported_query
  dashboardState.period !== 'realtime'          // 条件3：不是实时查询
```

**核心规则**：
1. 只有当 `skipImportedReason === 'unsupported_query'` 时才显示警告
2. 其他原因（`:no_imported_data`、`:out_of_range`、`:unsupported_interval`）都不显示

#### 10.3.2 不同面板的实现差异

让我分析五个主要统计面板的实现：

**面板 1：Sources（来源面板）**
位置：`assets/js/dashboard/stats/sources/index.js`

```javascript
// 第 184 行：状态初始化
const [skipImportedReason, setSkipImportedReason] = useState(null)

// 第 235-248 行：数据获取后处理
const afterFetchData = useCallback((apiResponse) => {
  setLoading(false)
  if (apiResponse) {
    setSkipImportedReason(apiResponse.skip_imported_reason)  // 正确提取
    // ...
  }
}, [])

// 第 367-370 行：渲染警告
<ImportedQueryUnsupportedWarning
  loading={loading}
  skipImportedReason={skipImportedReason}
/>
```

**面板 2：Pages（页面面板）**
位置：`assets/js/dashboard/stats/pages/index.js`

```javascript
// 第 159 行：状态初始化
const [skipImportedReason, setSkipImportedReason] = useState(null)

// 第 167-175 行：数据获取后处理
function afterFetchData(apiResponse) {
  setLoading(false)
  setSkipImportedReason(apiResponse.skip_imported_reason)  // 正确提取
  // ...
}

// 第 239-242 行：渲染警告
<ImportedQueryUnsupportedWarning
  loading={loading}
  skipImportedReason={skipImportedReason}
/>
```

**面板 3：Devices（设备面板）**
位置：`assets/js/dashboard/stats/devices/index.js`

```javascript
// 第 431 行：状态初始化
const [skipImportedReason, setSkipImportedReason] = useState(null)

// 第 439-447 行：数据获取后处理
function afterFetchData(apiResponse) {
  setLoading(false)
  setSkipImportedReason(apiResponse.skip_imported_reason)  // 正确提取
  // ...
}

// 第 524-527 行：渲染警告
<ImportedQueryUnsupportedWarning
  loading={loading}
  skipImportedReason={skipImportedReason}
/>
```

**面板 4：Locations（地理位置面板）**
位置：`assets/js/dashboard/stats/locations/index.js`

```javascript
// 第 154 行：状态初始化
skipImportedReason: null,

// 第 202-215 行：数据获取后处理
afterFetchData(apiResponse) {
  // ...
  this.setState({
    loading: false,
    moreLinkState: newMoreLinkState,
    skipImportedReason: apiResponse.skip_imported_reason  // 正确提取
  })
}

// 第 294-297 行：渲染警告
<ImportedQueryUnsupportedWarning
  loading={this.state.loading}
  skipImportedReason={this.state.skipImportedReason}
/>
```

**面板 5：Behaviours（行为面板）**
位置：`assets/js/dashboard/stats/behaviours/index.js`

```javascript
// 第 145 行：状态初始化
const [skipImportedReason, setSkipImportedReason] = useState(null)

// 第 256-264 行：数据获取后处理
function afterFetchData(apiResponse) {
  setLoading(false)
  setSkipImportedReason(apiResponse.skip_imported_reason)  // 正确提取
  // ...
}

// 第 440-464 行：特殊渲染逻辑
function renderImportedQueryUnsupportedWarning() {
  if (mode === Mode.CONVERSIONS) {
    return (
      <ImportedQueryUnsupportedWarning
        loading={loading}
        skipImportedReason={skipImportedReason}
      />
    )
  } else if (mode === Mode.PROPS) {
    return (
      <ImportedQueryUnsupportedWarning
        loading={loading}
        skipImportedReason={skipImportedReason}
        message="Imported data is unavailable in this view"  // 自定义消息
      />
    )
  } else {
    // FUNNELS 或 EXPLORATION 模式
    return (
      <ImportedQueryUnsupportedWarning
        altCondition={importedDataInView}  // 关键：使用 altCondition
        message="Imported data is unavailable in this view"
      />
    )
  }
}
```

#### 10.3.3 Behaviours 面板的特殊逻辑

这是导致提示不一致的核心原因。让我深入分析 `altCondition` 的作用：

**警告组件中的 `altCondition`**：`assets/js/dashboard/stats/imported-query-unsupported-warning.js:29`

```javascript
if (show || altCondition) {  // 逻辑或！
  return (
    <FadeIn show={!loading} className="h-4.5">
      <Tooltip info={tooltipMessage} containerRef={portalRef}>
        <ExclamationCircleIcon className="mb-1 size-4.5 text-gray-500 dark:text-gray-400" />
      </Tooltip>
    </FadeIn>
  )
} else {
  return null
}
```

**关键逻辑**：
- 显示条件是 `show || altCondition`（逻辑或）
- 即使 `show` 为 false，只要 `altCondition` 为 true，也会显示警告

**`importedDataInView` 的来源**：

在 `behaviours/index.js:561-591`：

```javascript
function BehavioursOuter({ importedDataInView }) {
  // ...
  return enabledModes.length && mode ? (
    <Behaviours
      importedDataInView={importedDataInView}  // 从父组件传入
      mode={mode}
      setMode={setMode}
    />
  ) : null
}

export default function BehavioursWrapped({ importedDataInView }) {
  return (
    <ModesContextProvider>
      <BehavioursOuter importedDataInView={importedDataInView} />
    </ModesContextProvider>
  )
}
```

**`importedDataInView` 的含义**：
- 由更上层的 Dashboard 组件传入
- 表示"视图中是否存在导入数据"（通过其他方式判断，而非 `skipImportedReason`）
- 在 FUNNELS 和 EXPLORATION 模式下，即使 `skipImportedReason` 不是 `'unsupported_query'`，只要 `importedDataInView` 为 true，就会显示警告

#### 10.3.4 四种 Skip Reason 的显示行为对比

| Skip Reason | 普通面板（Sources/Pages/Devices/Locations） | Behaviours - Conversions 模式 | Behaviours - Props 模式 | Behaviours - Funnels/Exploration 模式 |
|-------------|-----------------------------------------------|--------------------------------|--------------------------|----------------------------------------|
| `:unsupported_query` | 显示 ✓ | 显示 ✓ | 显示 ✓（自定义消息） | 显示 ✓（`altCondition` 或 `show`） |
| `:no_imported_data` | 不显示 ✗ | 不显示 ✗ | 不显示 ✗ | 可能显示（取决于 `importedDataInView`） |
| `:out_of_range` | 不显示 ✗ | 不显示 ✗ | 不显示 ✗ | 可能显示（取决于 `importedDataInView`） |
| `:unsupported_interval` | 不显示 ✗ | 不显示 ✗ | 不显示 ✗ | 可能显示（取决于 `importedDataInView`） |

#### 10.3.5 设计取舍与影响

**为什么只显示 `:unsupported_query`**？

`lib/plausible/stats/query_result.ex:18-24` 中的预定义警告消息：

```elixir
@imports_warnings %{
  unsupported_query:
    "Imported stats are not included in the results because query parameters are not supported. " <>
      "For more information, see: https://plausible.io/docs/stats-api#filtering-imported-stats",
  unsupported_interval:
    "Imported stats are not included because the time dimension (i.e. the interval) is too short."
}
```

**设计意图**：
1. **`:unsupported_query`**：用户可以调整查询参数来解决（如移除某些过滤器），需要提示
2. **`:unsupported_interval`**：虽然有消息，但前端组件不显示（可能有其他时间粒度提示）
3. **`:no_imported_data`**：用户没有导入数据，是"正常"情况，无需警告
4. **`:out_of_range`**：用户选择的时间范围不包含导入数据，也是"正常"情况

**Behaviours 面板的特殊考虑**：
- FUNNELS 和 EXPLORATION 是企业版高级功能
- 这些功能本质上不支持导入数据（需要原始事件序列）
- 使用 `altCondition` 可以在用户有导入数据时持续提示"导入数据在此视图不可用"
- 这是一种"功能告知"而非"错误提示"

**潜在问题**：
1. **用户困惑**：不同面板显示逻辑不一致，用户可能不理解为什么某些面板显示警告而其他不显示
2. **调试困难**：`importedDataInView` 的来源不明确，可能与 `skipImportedReason` 不同步
3. **消息不一致**：普通面板使用默认消息 `"Imported data is excluded due to applied filters"`，而 Behaviours 面板使用 `"Imported data is unavailable in this view"`

### 10.4 兼容分流总结与影响评估

#### 10.4.1 三条分流路径汇总

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        旧数据访问兼容分流全景                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. 参数解析分流                                                            │
│     ├─ Legacy 入口 (Query.from/3)                                          │
│     │   ├─ 参数：with_imported (字符串 "true")                              │
│     │   ├─ 使用者：细分 API、外部 API v1                                    │
│     │   └─ 特点：向后兼容，但参数格式不统一                                  │
│     │                                                                        │
│     └─ 当前入口 (Query.parse_and_build/3)                                   │
│         ├─ 参数：include.imports (布尔值 true)                             │
│         ├─ 使用者：统一 query API                                           │
│         └─ 特点：结构化 JSON，类型安全                                      │
│                                                                             │
│  2. Compare 场景分流                                                        │
│     ├─ 主查询范围：in_range                                                  │
│     ├─ 比较查询范围：in_comparison_range                                    │
│     ├─ 合并：in_comparison_range ++ in_range                                │
│     └─ 边界：                                                                │
│         ├─ 实时查询：直接返回 []                                            │
│         ├─ 跨范围导入：可能重复，但实际不影响                               │
│         └─ 两者都无：:out_of_range                                         │
│                                                                             │
│  3. 前端提示分流                                                            │
│     ├─ 普通面板 (Sources/Pages/Devices/Locations)                          │
│     │   └─ 只显示：skipImportedReason === 'unsupported_query'              │
│     │                                                                        │
│     └─ Behaviours 面板                                                      │
│         ├─ Conversions/Props：同普通面板                                   │
│         └─ Funnels/Exploration：show || altCondition                       │
│             └─ altCondition = importedDataInView                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 10.4.2 关键设计取舍复盘

| 决策点 | 选择 | 优势 | 劣势 |
|--------|------|------|------|
| **双入口并存** | Legacy + 当前 | 向后兼容，渐进迁移 | 维护成本高，测试复杂 |
| **参数格式差异** | `with_imported` vs `include.imports` | 符合各自 API 风格 | 开发者混淆，类型不匹配 |
| **Compare 合并策略** | 列表拼接 `++` | 实现简单，不影响判断 | 可能重复（但实际无害） |
| **Skip Reason 显示** | 只显示 `unsupported_query` | 用户体验简洁 | 某些情况缺乏反馈 |
| **Behaviours 特殊逻辑** | `altCondition` | 高级功能明确告知 | 与其他面板不一致 |

#### 10.4.3 潜在风险与改进建议

**风险 1：参数名称混淆**
- 场景：开发者同时使用细分 API 和统一 API，可能混淆 `with_imported` 和 `include.imports`
- 建议：添加参数别名支持，或在文档中明确区分

**风险 2：Compare 场景边界不清**
- 场景：用户启用 Compare 但比较范围内没有导入数据，可能困惑为什么某些数据不合并
- 建议：在 `skip_imported_reason` 中增加 Compare 相关的细分原因，或在前端明确告知比较范围

**风险 3：提示不一致导致用户困惑**
- 场景：用户在普通面板看不到提示，但在 Behaviours 面板看到，可能误解为 Bug
- 建议：统一显示逻辑，或为每种 `skip_imported_reason` 提供明确的用户反馈

**风险 4：Legacy 入口技术债务**
- 场景：`Legacy.QueryBuilder` 已标记 `@deprecated`，但仍被广泛使用
- 建议：制定迁移计划，逐步将细分 API 迁移到当前入口

## 11. 结论

Plausible Analytics 系统设计了一套完善的数据保留策略、导入导出流程和历史数据查询兼容机制，通过以下核心原则实现了高效的数据生命周期管理：

1. **分离存储，统一查询**：导入数据与原生数据分离存储，但在查询时透明合并
2. **格式一致，双向兼容**：导入和导出使用相同的表结构，确保数据可自由迁移
3. **异步操作，批量处理**：利用 ClickHouse 的特性，实现高效的大规模数据操作
4. **元数据驱动，灵活管理**：通过详细的元数据记录，支持导入的全生命周期管理
5. **精细判定，透明反馈**：通过多层 skip reason 机制，精确判定不包含导入数据的原因，并通过 API 和前端反馈给用户

这套机制不仅满足了当前的业务需求，也为未来的功能扩展和性能优化奠定了坚实的基础。通过遵循最佳实践和持续改进，系统可以更好地支持用户的数据分析需求。

**新增的关键洞察**：
- **默认禁用导入数据**是性能和一致性的平衡选择
- **单一表限制**是数据正确性的保障，但也限制了查询灵活性
- **skip reason 的四层判定**（时间间隔 → 存在性 → 范围 → 查询类型）是精心设计的优先级
- **前端只显示 `unsupported_query`** 是用户体验的精细化设计：其他原因要么是"正常"情况，要么有其他提示机制
