# Plausible Analytics 维度解析与展示分析报告

## 目录
1. [概述](#1-概述)
2. [Geo 定位维度](#2-geo-定位维度)
3. [设备类型维度](#3-设备类型维度)
4. [流量来源维度](#4-流量来源维度)
5. [数据存储模型](#5-数据存储模型)
6. [查询与展示流程](#6-查询与展示流程)
7. [关键文件索引](#7-关键文件索引)

---

## 1. 概述

Plausible Analytics 采用完整的数据管道来处理三个核心维度数据：

**完整处理流程：**
```
Tracker Script → HTTP Request → Ingestion Pipeline → ClickHouse Storage 
    → SQL Query Builder → API Response → React Dashboard
```

三个维度的解析均在 **数据摄入阶段（Ingestion）** 完成，然后以规范化的字段存储到 ClickHouse 中，查询时直接使用已解析的字段进行聚合。

---

## 2. Geo 定位维度

### 2.1 地理位置解析机制

**核心模块：** `lib/plausible/ingestion/geolocation.ex`

#### 地理位置数据源
使用 `locus` 库加载 MaxMind 或 DB-IP 的 MMDB 数据库文件：
- `lib/plausible/geo.ex:44-74` 提供数据库加载 API
- 支持两种模式：
  1. **本地文件模式**：通过 `:path` 参数指定 `.mmdb` 文件
  2. **MaxMind 授权模式**：通过 `:license_key` + `:edition` 从 MaxMind 下载

#### 地理数据解析流程

```
IP地址 → Plausible.Geo.lookup(ip) → MMDB查询 → 结构化地理信息
```

**关键处理步骤：** `lib/plausible/ingestion/geolocation.ex:4-52`

```elixir
def lookup(ip_address) do
  case Plausible.Geo.lookup(ip_address) do
    %{} = entry ->
      %{
        country_code: entry |> get_in(["country", "iso_code"]) |> ignore_unknown_country(),
        subdivision1_code: subdivision1_code(country_code, entry),  # 一级行政区划
        subdivision2_code: subdivision2_code(country_code, entry),  # 二级行政区划
        city_geoname_id: city_geoname_id  # 城市 Geoname ID
      }
    nil -> nil
  end
end
```

#### 国家代码过滤规则
特殊国家/地区代码会被过滤掉：
- `ZZ` - Worldwide（全球）
- `XX` - Disputed territory（争议地区）
- `T1` - Tor exit node（Tor 出口节点）

#### 匿名 VPN IP 处理
`lib/plausible/ingestion/event.ex:324-333`

```elixir
defp put_geolocation(%__MODULE__{} = event, _context) do
  case event.request.ip_classification do
    "anonymous_vpn_ip" ->
      update_session_attrs(event, %{country_code: "A1"})  # 匿名代理标记为 A1
    _any ->
      result = Plausible.Ingestion.Geolocation.lookup(event.request.remote_ip) || %{}
      update_session_attrs(event, result)
  end
end
```

**IP 分类来源**：从 HTTP Header `x-plausible-ip-type` 获取，由边缘层（如 Cloudflare）注入。

### 2.2 地理名称映射

地理代码（`country_code`, `subdivision_code`, `city_geoname_id`）在查询时通过 **ClickHouse Dictionary** 映射为可读名称。

**位置数据表：** `lib/plausible/clickhouse_location_data.ex`

```elixir
schema "location_data" do
  field :type, Ch, type: "LowCardinality(String)"  # country / region / city
  field :id, :string                               # 代码/ID
  field :name, :string                             # 显示名称
end
```

该表通过 `location_data_dictionary` 字典在 ALIAS 列中被间接访问：
- `sessions_v2.country` → ALIAS 映射到国家名称
- `sessions_v2.region` → ALIAS 映射到地区名称
- `sessions_v2.city` → ALIAS 映射到城市名称

### 2.3 Geo 数据展示

#### 前端组件
`assets/js/dashboard/stats/locations/index.js`

**展示模式：**
1. **地图视图（Map）**：使用 `CountriesMap` 组件展示世界地图
2. **国家列表（Countries）**：列表展示，附带国旗
3. **地区列表（Regions）**：州/省级别
4. **城市列表（Cities）**：城市级别

#### 后端 API
`lib/plausible_web/controllers/api/stats_controller.ex:751-912`

**接口说明：**

| 接口 | 维度 | 数据转换 |
|------|------|----------|
| `GET /countries` | `visit:country` | country_code → name + flag + alpha_3 |
| `GET /regions` | `visit:region` | subdivision_code → name + country_flag |
| `GET /cities` | `visit:city` | geoname_id → name + country_flag |

**国家信息获取：** 使用 `Location` 库（基于 `:location` hex 包）获取国家的详细信息：
- 国家名称（多语言支持）
- 国旗 emoji
- Alpha-2 / Alpha-3 代码

### 2.4 Geo 数据屏蔽规则
`lib/plausible/ingestion/event.ex:335-360`

支持基于国家代码的屏蔽（Shield 功能）：
```elixir
defp drop_shield_rule_country(event, _context) do
  if Plausible.Shields.country_blocked?(domain, country_code) do
    drop(event, :site_country_blocklist)
  else
    event
  end
end
```

---

## 3. 设备类型维度

### 3.1 User Agent 解析机制

**核心模块：** `lib/plausible/ingestion/event.ex:256-551`

使用 `UAInspector` 库进行 User Agent 解析，配合异步任务和超时保护。

#### UA 解析流程
```
User-Agent Header → UAInspector.parse() → {os, browser, device}
```

**关键实现：** `lib/plausible/ingestion/event.ex:444-470`

```elixir
@parse_user_agent_timeout 200  # 200ms 超时保护

defp parse_user_agent_safe(user_agent) do
  task = Task.Supervisor.async_nolink(
    Plausible.UserAgentParseTaskSupervisor,
    fn -> UAInspector.parse(user_agent) end
  )

  case Task.yield(task, @parse_user_agent_timeout) || Task.shutdown(task) do
    {:ok, result} -> {:ok, result}
    nil -> 
      emit_telemetry_ua_parse_timeout()
      {:error, :timeout}
  end
end
```

**缓存机制：** 解析结果会缓存在 `:user_agents` Cache 中，避免重复解析。

#### Bot 检测与过滤
`lib/plausible/ingestion/event.ex:256-276`

以下 User Agent 会被标记为 bot 并丢弃：
1. `UAInspector.Result.Bot{}` - 明确的机器人
2. `Headless Chrome` - 无头浏览器

#### 设备字段解析

**操作系统（Operating System）：** `lib/plausible/ingestion/event.ex:526-531`
```elixir
defp os_name(ua) do
  case ua.os do
    :unknown -> ""
    os -> os.name
  end
end
```

**浏览器（Browser）：** `lib/plausible/ingestion/event.ex:472-488`

移动端浏览器会被规范化：
| 原始名称 | 规范化后 |
|---------|---------|
| Mobile Safari | Safari |
| Chrome Mobile | Chrome |
| Chrome Mobile iOS | Chrome |
| Firefox Mobile | Firefox |
| Chrome Webview / mobile app | Mobile App |

**屏幕尺寸/设备类型（Screen Size）：** `lib/plausible/ingestion/event.ex:490-516`

基于 `UAInspector.Result.Device.type` 分类：

```elixir
@mobile_types ["smartphone", "feature phone", "portable media player", 
               "phablet", "wearable", "camera"]
@tablet_types ["car browser", "tablet"]
@desktop_types ["tv", "console", "desktop"]

defp screen_size(ua) do
  case ua.device do
    %Device{type: t} when t in @mobile_types -> "Mobile"
    %Device{type: t} when t in @tablet_types -> "Tablet"
    %Device{type: t} when t in @desktop_types -> "Desktop"
    _ -> nil
  end
end
```

**版本号处理：** `lib/plausible/ingestion/event.ex:540-551`
```elixir
defp major_minor(version) do
  version
  |> String.split(".")
  |> Enum.take(2)      # 只取主版本.次版本
  |> Enum.join(".")
end
```

### 3.2 设备数据展示

#### 前端组件
`assets/js/dashboard/stats/devices/index.js`

**三个 Tab 视图：**

1. **浏览器（Browsers）**
   - 维度：`visit:browser`
   - 显示浏览器图标（Chrome, Safari, Firefox, Edge 等）
   - 点击可下钻到版本明细

2. **操作系统（Operating Systems）**
   - 维度：`visit:os`
   - 显示 OS 图标（Windows, macOS, iOS, Android 等）
   - 点击可下钻到版本明细

3. **设备类型（Devices）**
   - 维度：`visit:device`（即 screen_size）
   - 显示 SVG 图标：Mobile / Tablet / Desktop

#### 后端 API
`lib/plausible_web/controllers/api/stats_controller.ex:915-1121`

| 接口 | 维度 | 说明 |
|------|------|------|
| `GET /browsers` | `visit:browser` | 浏览器列表 |
| `GET /browser-versions` | `visit:browser_version` | 版本明细（同时返回 browser 分组） |
| `GET /operating-systems` | `visit:os` | 操作系统列表 |
| `GET /operating-system-versions` | `visit:os_version` | 系统版本明细 |
| `GET /screen-sizes` | `visit:device` | 设备类型（Mobile/Tablet/Desktop） |

**版本查询特殊处理：** `lib/plausible/stats/breakdown.ex:135-139`
```elixir
def transform_dimensions("visit:browser_version"),
  do: ["visit:browser", "visit:browser_version"]

def transform_dimensions("visit:os_version"), 
  do: ["visit:os", "visit:os_version"]
```
查询版本时会同时查询主名称，用于前端显示组合名称。

---

## 4. 流量来源维度

### 4.1 来源解析机制

**核心模块：** 
- `lib/plausible/ingestion/source.ex` - Source 解析
- `lib/plausible/ingestion/acquisition.ex` - Channel 推断

#### 来源解析优先级
`lib/plausible/ingestion/source.ex:66-80`

```elixir
def resolve(request) do
  tagged_source =
    request.query_params["utm_source"] ||
      request.query_params["source"] ||
      request.query_params["ref"]

  source =
    cond do
      tagged_source -> tagged_source                           # 1. UTM 参数优先
      has_valid_referral?(request) -> parse(request.referrer)  # 2. Referer 头
      true -> nil
    end

  find_mapping(source)  # 3. 规范化映射
end
```

#### Referer 解析
`lib/plausible/ingestion/source.ex:82-100`

使用 `RefInspector` 库解析 Referer：
```elixir
def parse(ref) do
  case RefInspector.parse(ref).source do
    :unknown ->
      uri = URI.parse(String.trim(ref))
      format_referrer_host(uri)   # 未知来源用域名
    source ->
      source                      # 已知来源（如 "Google"）
  end
end
```

**来源规范化映射：**
- 内置 `RefInspector` 数据库（`priv/ref_inspector/referers.yml`）
- 自定义扩展 `priv/custom_sources.json`
- 支持大小写不敏感匹配

**自定义来源示例：**
| 缩写/变体 | 规范化后 |
|----------|---------|
| ig | Instagram |
| adwords | Google |
| yt-ads | YouTube |

#### 有效 Referer 检查
`lib/plausible/ingestion/source.ex:110-124`

必须满足以下条件才认为是有效引用：
1. Scheme 为 `http` / `https` / `android-app`
2. Host 不为空
3. **不是内部域名**（与当前站点 hostname 对比）
4. **不是 localhost**

#### 流量获取渠道推断
`lib/plausible/ingestion/acquisition.ex:50-201`

基于 Source + UTM 参数推断 Acquisition Channel：

**渠道分类规则（优先级从高到低）：**

| 渠道 | 条件 |
|------|------|
| Cross-network | utm_campaign 包含 "cross-network" |
| Paid Shopping | 购物来源 + 付费 medium |
| Paid Search | 搜索来源 + 付费 medium/utm_source 或 gclid/msclkid |
| Paid Social | 社交来源 + 付费 medium/utm_source |
| Paid Video | 视频来源 + 付费 medium/utm_source |
| Display | utm_medium 为 display/banner 等 |
| Organic Shopping | 购物来源 |
| Organic Social | 社交来源或 utm_medium 包含 social |
| Organic Video | 视频来源或 utm_medium 包含 video |
| Organic Search | 搜索来源 |
| Email | 邮件来源或 utm 参数包含 email 关键词 |
| Affiliates | utm_medium == "affiliate" |
| Audio | utm_medium == "audio" |
| SMS | utm_source/utm_medium == "sms" |
| Mobile Push Notifications | utm_medium 包含 push/mobile/notification |
| Referral | utm_medium 为 referral/app/link 或有 source |
| Direct | 其他所有情况 |

**付费 medium 匹配正则：**
```elixir
~r/^(.*cp.*|ppc|retargeting|paid.*)$/
```

**自定义来源类别扩展：**
`lib/plausible/ingestion/acquisition.ex:20-40`

```elixir
@custom_source_categories [
  {"hacker news", "SOURCE_CATEGORY_SOCIAL"},
  {"yahoo!", "SOURCE_CATEGORY_SEARCH"},
  {"gmail", "SOURCE_CATEGORY_EMAIL"},
  {"telegram", "SOURCE_CATEGORY_SOCIAL"},
  {"slack", "SOURCE_CATEGORY_SOCIAL"},
  {"producthunt", "SOURCE_CATEGORY_SOCIAL"},
  {"github", "SOURCE_CATEGORY_SOCIAL"},
  {"perplexity", "SOURCE_CATEGORY_SEARCH"},     # AI 搜索
  {"chatgpt.com", "SOURCE_CATEGORY_SEARCH"},    # AI 搜索
  {"brave", "SOURCE_CATEGORY_SEARCH"},
  {"discord", "SOURCE_CATEGORY_SOCIAL"},
  # ...
]
```

**基础数据源：** `priv/ga4-source-categories.csv`（来自 Google Analytics 4 官方分类）

### 4.2 流量来源数据展示

#### 前端组件
`assets/js/dashboard/stats/sources/index.js`

**四个视图模式：**

1. **Channels（渠道）**
   - 维度：`visit:channel`
   - 展示：Organic Search, Direct, Paid Search, Social, Referral, Email 等

2. **Sources（来源）**
   - 维度：`visit:source`
   - 展示来源网站图标（favicon）
   - 点击 Google 来源会显示搜索关键词（需配置 Google Search Console）
   - 点击其他来源下钻到具体 referrer 页面

3. **UTM Campaigns（广告系列）**
   - 下拉菜单包含：utm_medium, utm_source, utm_campaign, utm_content, utm_term

#### 后端 API
`lib/plausible_web/controllers/api/stats_controller.ex:59-612`

| 接口 | 维度 | 说明 |
|------|------|------|
| `GET /sources` | `visit:source` | 来源列表 |
| `GET /channels` | `visit:channel` | 渠道列表 |
| `GET /utm_mediums` | `visit:utm_medium` | UTM 媒介 |
| `GET /utm_sources` | `visit:utm_source` | UTM 来源 |
| `GET /utm_campaigns` | `visit:utm_campaign` | UTM 广告系列 |
| `GET /utm_contents` | `visit:utm_content` | UTM 内容 |
| `GET /utm_terms` | `visit:utm_term` | UTM 关键词 |
| `GET /referrers` | `visit:referrer` | 具体引用页面 |

**Google 搜索关键词特殊处理：**
`lib/plausible_web/controllers/api/stats_controller.ex:526-578`

当 source 为 "Google" 时，调用 Google Search Console API 获取搜索关键词，而非使用 Plausible 自身数据。

**Click ID 自动推断：** `lib/plausible/ingestion/event.ex:312-322`

```elixir
defp maybe_infer_medium(%__MODULE__{} = event, _context) do
  inferred_medium =
    case event.clickhouse_session_attrs do
      %{utm_medium: medium} when is_binary(medium) -> medium
      %{utm_medium: nil, referrer_source: "Google", click_id_param: "gclid"} -> "(gclid)"
      %{utm_medium: nil, referrer_source: "Bing", click_id_param: "msclkid"} -> "(msclkid)"
      _ -> nil
    end
  update_session_attrs(event, %{utm_medium: inferred_medium})
end
```

支持的 Click ID 参数：`gclid`, `gbraid`, `wbraid`, `msclkid`, `fbclid`, `twclid`

---

## 5. 数据存储模型

### 5.1 Session 表结构
`lib/plausible/clickhouse_session_v2.ex`

核心维度字段：

```elixir
# Geo 定位
field :country_code, Ch, type: "LowCardinality(FixedString(2))"
field :subdivision1_code, Ch, type: "LowCardinality(String)"
field :subdivision2_code, Ch, type: "LowCardinality(String)"
field :city_geoname_id, Ch, type: "UInt32"

# 设备类型
field :screen_size, Ch, type: "LowCardinality(String)"       # Mobile/Tablet/Desktop
field :operating_system, Ch, type: "LowCardinality(String)"
field :operating_system_version, Ch, type: "LowCardinality(String)"
field :browser, Ch, type: "LowCardinality(String)"
field :browser_version, Ch, type: "LowCardinality(String)"

# 流量来源
field :referrer_source, :string                            # 规范化来源名
field :referrer, :string                                    # 具体引用 URL
field :click_id_param, Ch, type: "LowCardinality(String)"  # gclid/msclkid 等
field :utm_medium, :string
field :utm_source, :string
field :utm_campaign, :string
field :utm_content, :string
field :utm_term, :string
field :acquisition_channel, Ch, type: "LowCardinality(String)", writable: :never
```

**注意：** `acquisition_channel` 是 `writable: :never` 的 ALIAS 列，在查询时根据其他字段动态计算。

### 5.2 维度名称映射

`lib/plausible/stats/sql/expression.ex:164-246`

查询时的维度选择：

| 查询维度 | 对应字段 |
|---------|---------|
| `visit:country` | `t.country`（ALIAS → 名称） |
| `visit:region` | `t.region`（ALIAS → 名称） |
| `visit:city` | `t.city`（ALIAS → 名称） |
| `visit:device` | `t.device`（即 screen_size） |
| `visit:os` | `t.os` |
| `visit:os_version` | `t.os_version` |
| `visit:browser` | `t.browser` |
| `visit:browser_version` | `t.browser_version` |
| `visit:source` | `t.source`（即 referrer_source） |
| `visit:channel` | `t.acquisition_channel` |
| `visit:referrer` | `t.referrer` |
| `visit:utm_*` | `t.utm_*` |

空值处理：
- Source: `"Direct / None"`
- Channel: `"Direct"`
- 其他: `"(not set)"`

---

## 6. 查询与展示流程

### 6.1 完整数据流

```
前端请求
    ↓
PlausibleWeb.Api.StatsController
    ↓  (选择 property 参数如 "visit:country")
Plausible.Stats.breakdown/4
    ↓
Plausible.Stats.Breakdown.breakdown/5
    ↓  (构建 Query 结构体)
Plausible.Stats.QueryRunner.run/2
    ↓
Plausible.Stats.SQL.QueryBuilder.build/2
    ↓  (生成 ClickHouse SQL)
ClickHouse 查询
    ↓  (按维度 GROUP BY + 指标聚合)
查询结果
    ↓  (API 层附加名称/国旗等展示信息)
前端 React 组件渲染
```

### 6.2 数据摄入流水线

`lib/plausible/ingestion/event.ex:130-151`

```elixir
defp pipeline() do
  [
    drop_verification_agent: &drop_verification_agent/2,
    drop_datacenter_ip: &drop_datacenter_ip/2,
    drop_threat_ip: &drop_threat_ip/2,
    drop_shield_rule_hostname: &drop_shield_rule_hostname/2,
    drop_shield_rule_page: &drop_shield_rule_page/2,
    drop_shield_rule_ip: &drop_shield_rule_ip/2,
    put_geolocation: &put_geolocation/2,           # ← Geo 定位
    drop_shield_rule_country: &drop_shield_rule_country/2,
    put_user_agent: &put_user_agent/2,             # ← 设备类型
    put_basic_info: &put_basic_info/2,
    put_source_info: &put_source_info/2,           # ← 流量来源
    maybe_infer_medium: &maybe_infer_medium/2,     # ← 推断 medium
    put_props: &put_props/2,
    put_revenue: &put_revenue/2,
    put_salts: &put_salts/2,
    put_user_id: &put_user_id/2,
    validate_clickhouse_event: &validate_clickhouse_event/2,
    register_session: &register_session/2
  ]
end
```

---

## 7. 关键文件索引

| 文件路径 | 职责 |
|---------|------|
| `lib/plausible/ingestion/geolocation.ex` | Geo 定位解析 |
| `lib/plausible/geo.ex` | Geo 数据库加载 API |
| `lib/plausible/ingestion/event.ex` | 设备类型 + 来源解析主入口 |
| `lib/plausible/ingestion/source.ex` | 流量来源（Source）解析 |
| `lib/plausible/ingestion/acquisition.ex` | 流量渠道（Channel）推断 |
| `lib/plausible/ingestion/request.ex` | HTTP 请求构建 |
| `lib/plausible/clickhouse_session_v2.ex` | Session 存储模型 |
| `lib/plausible/stats/sql/expression.ex` | SQL 维度/指标表达式 |
| `lib/plausible/stats/sql/query_builder.ex` | SQL 查询构建 |
| `lib/plausible/stats/breakdown.ex` | Breakdown 查询封装 |
| `lib/plausible_web/controllers/api/stats_controller.ex` | 统计 API 端点 |
| `assets/js/dashboard/stats/locations/index.js` | 地理位置前端展示 |
| `assets/js/dashboard/stats/devices/index.js` | 设备类型前端展示 |
| `assets/js/dashboard/stats/sources/index.js` | 流量来源前端展示 |
| `priv/ref_inspector/referers.yml` | Referer 解析规则库 |
| `priv/custom_sources.json` | 自定义来源映射 |
| `priv/ga4-source-categories.csv` | GA4 来源类别基准 |

---

## 8. 总结

三个维度的设计模式高度一致：

1. **摄入时解析**：在数据进入 ClickHouse 之前完成所有解析工作
2. **规范化存储**：使用代码/ID 存储，而非原始字符串
3. **查询时映射**：通过 Dictionary/ALIAS 或应用层代码映射为展示名称
4. **分层展示**：前端使用 Tab 切换不同粒度的视图（国家→地区→城市，浏览器→版本等）

关键技术选择：
- **地理位置**：MaxMind/DB-IP MMDB + locus 库
- **设备识别**：UAInspector 库 + 200ms 超时保护 + 缓存
- **来源识别**：RefInspector 库 + 自定义规则 + UTM 参数
- **渠道推断**：基于 GA4 分类 + Plausible 自定义扩展

