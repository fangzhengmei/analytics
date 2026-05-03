# Bot 过滤、隐私保护与访客归因协同分析报告

## 1. 整体架构概览

Plausible Analytics 的数据处理流水线采用了分层过滤 + 隐私优先 + 智能归因的设计模式。三个核心模块在 `lib/plausible/ingestion/event.ex` 中通过 Pipeline 机制协同工作。

**请求处理入口**：
- 原始请求在 `lib/plausible/ingestion/request.ex` 中构建 `%Request{}` 结构体
- 包含：remote_ip、user_agent、referrer、query_params、ip_classification 等核心元数据

**处理流水线**（`event.ex:130-151`）：
```
[
  drop_verification_agent,     # 安装验证代理过滤
  drop_datacenter_ip,           # 数据中心IP过滤
  drop_threat_ip,               # 威胁IP过滤
  drop_shield_rule_hostname,    # 主机名白名单检查
  drop_shield_rule_page,        # 页面黑名单检查
  drop_shield_rule_ip,          # IP黑名单检查
  put_geolocation,              # 地理位置解析
  drop_shield_rule_country,     # 国家黑名单检查
  put_user_agent,               # UA解析 + Bot检测 (关键)
  put_basic_info,               # 基本信息填充
  put_source_info,              # 来源归因 (关键)
  maybe_infer_medium,           # 媒介推断
  put_props, put_revenue,       # 扩展属性
  put_salts, put_user_id,       # 隐私保护 (关键)
  validate_clickhouse_event,    # 数据验证
  register_session              # 持久化入库
]
```

---

## 2. Bot 过滤机制

### 2.1 多层过滤策略

系统采用 **四层 Bot/垃圾流量过滤** 机制：

| 过滤层级 | 实现位置 | 检测方式 | 丢弃原因 |
|---------|---------|---------|---------|
| L1: 验证代理 | `event.ex:196-210` | UA 匹配预设值 | `:verification_agent` |
| L2: IP 信誉 | `event.ex:212-230` | `x-plausible-ip-type` 头 | `:dc_ip`, `:threat_ip` |
| L3: 规则引擎 | `event.ex:232-360` | Shields 规则 | `:site_ip_blocklist`, `:site_country_blocklist` 等 |
| L4: UA 解析 | `event.ex:256-276` | UAInspector Bot 检测 | `:bot` |

### 2.2 核心 Bot 检测逻辑

**关键实现** (`lib/plausible/ingestion/event.ex:256-276`):

```elixir
defp put_user_agent(%__MODULE__{} = event, _context) do
  case parse_user_agent(event.request) do
    # 无头浏览器检测 - Headless Chrome
    {:ok, %UAInspector.Result{client: %UAInspector.Result.Client{name: "Headless Chrome"}}} ->
      drop(event, :bot)

    # Bot UA 检测 - 匹配 bot.bots.yml 正则
    {:ok, %UAInspector.Result.Bot{}} ->
      drop(event, :bot)

    # 正常用户 UA - 提取设备信息
    {:ok, %UAInspector.Result{} = user_agent} ->
      update_session_attrs(event, %{
        operating_system: os_name(user_agent),
        browser: browser_name(user_agent),
        screen_size: screen_size(user_agent)
        # ...
      })

    _any -> event  # 解析失败不丢弃，保持事件
  end
end
```

### 2.3 Bot 检测数据源

**Bot 规则库** (`priv/ua_inspector/bot.bots.yml`):

包含超过 500+ Bot 规则，分类如下：

| 分类 | 示例 | 匹配方式 |
|-----|------|---------|
| 搜索引擎爬虫 | `Googlebot`, `BingBot`, `Baidu Spider` | 正则匹配 |
| 社交媒体爬虫 | `Twitterbot`, `Facebook Crawler`, `LinkedInBot` | 正则匹配 |
| 网站监控 | `UptimeRobot`, `Pingdom`, `StatusCake` | 正则匹配 |
| 安全扫描器 | `Nmap`, `Arachni`, `Nikto` | 正则匹配 |
| AI 数据采集 | `Google-Extended`, `GoogleAgent-Mariner` | 正则匹配 |
| 验证工具 | `W3C_Validator`, `SSL Labs` | 正则匹配 |

### 2.4 请求信息边界 - Bot 过滤

**入站信息**：
- `user_agent` - 来自 HTTP `User-Agent` 头 (`request.ex:356-363`)
- `ip_classification` - 来自 `x-plausible-ip-type` 头 (`request.ex:347-354`)
  - `dc_ip` = 数据中心 IP
  - `threat_ip` = 威胁 IP
  - `anonymous_vpn_ip` = 匿名 VPN

**边界决策点**：
1. **UA 解析超时保护** (`event.ex:444-470`):
   ```elixir
   @parse_user_agent_timeout 200
   # 超时后返回 {:error, :timeout}，但不丢弃事件
   ```
   - 设计决策：宁可保留可能的 Bot，也不误杀真实用户

2. **数据中心 IP 直接丢弃** (`event.ex:212-220`):
   ```elixir
   defp drop_datacenter_ip(event, _context) do
     case event.request.ip_classification do
       "dc_ip" -> drop(event, :dc_ip)
       _any -> event
     end
   end
   ```
   - 这是一个**强过滤**，数据中心流量一律不入库

---

## 3. 隐私保护机制

### 3.1 核心设计原则

Plausible 采用 **"隐私-by-design"** 架构，核心保护机制包括：
1. 无 Cookie 追踪
2. 盐值轮换的用户哈希
3. 敏感 IP 模糊化
4. 定期数据清理

### 3.2 用户 ID 生成机制

**关键实现** (`event.ex:553-567`):

```elixir
defp generate_user_id(request, domain, hostname, salt) do
  cond do
    is_nil(salt) -> nil
    is_nil(domain) -> nil
    true ->
      user_agent = request.user_agent || ""
      root_domain = get_root_domain(hostname)

      # 核心哈希公式
      SipHash.hash!(salt, user_agent <> request.remote_ip <> domain <> root_domain)
  end
end
```

**哈希输入因子**：
| 因子 | 来源 | 作用 |
|-----|------|-----|
| `salt` | 数据库 + ETS 缓存 | 定期轮换，防止跨期关联 |
| `user_agent` | HTTP 头 | 设备指纹组件 |
| `remote_ip` | 客户端 IP | 网络指纹组件 |
| `domain` | 事件目标域 | 站点隔离 |
| `root_domain` | 公共后缀解析 | 子域聚合 |

### 3.3 盐值轮换机制

**实现模块** (`lib/plausible/session/salts.ex`):

```elixir
defmodule Plausible.Session.Salts do
  # 盐值刷新间隔（默认 90 秒）
  @interval :timer.seconds(90)

  def refresh(name, now) do
    # 只保留最近 2 个盐值
    salts = Repo.all(from s in "salts", select: s.salt, order_by: [desc: s.id], limit: 2)
    # ...
  end

  defp clean_old_salts(now) do
    # 48 小时前的盐值彻底删除
    h48_ago = DateTime.shift(now, hour: -48)
    Repo.delete_all(from s in "salts", where: s.inserted_at < ^h48_ago)
  end

  defp generate_and_persist_new_salt(now) do
    # 使用加密安全随机字节
    salt = :crypto.strong_rand_bytes(16)
    Repo.insert_all("salts", [%{salt: salt, inserted_at: now}])
    salt
  end
end
```

**盐值生命周期**：
```
┌─────────────────────────────────────────────────────────────┐
│  时间轴                                                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  T=0      ┌─────────────┐                                    │
│           │  Salt #1    │  ← current                         │
│           │ (current)   │                                    │
│           └─────────────┘                                    │
│                                                              │
│  T=90s    ┌─────────────┐  ┌─────────────┐                 │
│           │  Salt #1    │  │  Salt #2    │  ← current     │
│           │ (previous)  │  │ (current)   │                 │
│           └─────────────┘  └─────────────┘                 │
│                                                              │
│  T=48h+   盐值被 DELETE，无法回溯                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 3.4 双盐值设计与用户跨期识别

**关键代码** (`event.ex:414-433`):

```elixir
defp register_session(%__MODULE__{} = event, context) do
  # 使用当前盐值生成的 user_id 已存在于 clickhouse_event_attrs
  # 这里使用 previous 盐值生成另一个 user_id 用于会话查找

  previous_user_id =
    generate_user_id(
      event.request,
      event.domain,
      event.clickhouse_event.hostname,
      event.salts.previous  # 注意：使用 previous 盐值！
    )

  # 持久化时同时尝试 current 和 previous user_id
  Plausible.Ingestion.Persistor.persist_event(event, previous_user_id, persistor_opts)
end
```

**设计意图**：
- **current 盐值**：用于生成最终存入数据库的 `user_id`
- **previous 盐值**：用于在盐值轮换后，查找"昨天的"同一个用户
- 目的：在盐值轮换的 90 秒窗口内，保持会话的连续性

### 3.5 IP 分类与隐私处理

**处理流程** (`event.ex:324-333`, `event.ex:212-230`):

```
                  ┌──────────────────────┐
                  │  x-plausible-ip-type │
                  └──────────┬───────────┘
                             │
            ┌────────────────┼────────────────┐
            ▼                ▼                ▼
      ┌──────────┐    ┌──────────┐    ┌──────────────┐
      │  dc_ip   │    │ threat_ip│    │anonymous_vpn │
      │数据中心  │    │  威胁IP  │    │    IP        │
      └────┬─────┘    └────┬─────┘    └──────┬───────┘
           │               │                  │
           ▼               ▼                  ▼
      ┌─────────────────────────────────┐    │
      │         DROP (丢弃)              │    │
      │  drop_reason: :dc_ip/:threat_ip │    │
      └─────────────────────────────────┘    │
                                              ▼
                                   ┌──────────────────┐
                                   │  country_code    │
                                   │     = "A1"       │
                                   │  (匿名网络标识)  │
                                   └──────────────────┘
```

**代码实现** (`event.ex:324-333`):

```elixir
defp put_geolocation(%__MODULE__{} = event, _context) do
  case event.request.ip_classification do
    "anonymous_vpn_ip" ->
      # 匿名 VPN：不使用真实地理位置，统一标记为 A1
      update_session_attrs(event, %{country_code: "A1"})

    _any ->
      # 正常 IP：使用 MaxMind GeoIP 数据库解析
      result = Plausible.Ingestion.Geolocation.lookup(event.request.remote_ip) || %{}
      update_session_attrs(event, result)
  end
end
```

### 3.6 请求信息边界 - 隐私保护

**入站信息**：
| 信息 | 来源 | 处理方式 |
|-----|------|---------|
| `remote_ip` | HTTP 连接 (经过代理头处理) | 用于哈希生成，不直接存储 |
| `user_agent` | HTTP 头 | 用于哈希生成，解析后只存储规范化的 browser/os |
| `x-plausible-ip-type` | 反向代理注入 | 决定 IP 分类处理策略 |

**存储边界**：
- ❌ **不存储**：原始 IP、完整 User-Agent 字符串
- ✅ **存储**：哈希后的 `user_id` (UInt64)、规范化的设备信息、模糊化的地理位置
- ⏰ **自动清理**：盐值 48 小时后删除，无法逆向关联

**准确性取舍**：
| 取舍点 | 设计选择 | 隐私收益 | 准确性损失 |
|-------|---------|---------|-----------|
| 盐值 48 小时轮换 | 定期删除 | 无法跨 48 小时追踪用户 | 长期用户行为分析受限 |
| 双盐值设计 | 90 秒 overlap | 平滑过渡 | 实现复杂度增加 |
| 匿名 VPN 标记为 A1 | 统一国家码 | 不暴露真实地理位置 | 损失精确流量来源分析 |
| 数据中心 IP 丢弃 | 直接过滤 | 减少非人类流量污染 | 可能误杀企业代理用户 |

---

## 4. 访客归因机制

### 4.1 归因层级与优先级

**核心模块** (`lib/plausible/ingestion/source.ex`):

归因遵循 **"标记优先于自然来源"** 的原则：

```
                    ┌─────────────────────────────┐
                    │      归因决策流程           │
                    └──────────────┬──────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              ▼                    ▼                    ▼
       ┌─────────────┐      ┌─────────────┐     ┌─────────────┐
       │  utm_source │      │   source    │     │     ref     │
       │  优先级 1   │      │  优先级 2   │     │  优先级 3   │
       └──────┬──────┘      └──────┬──────┘     └──────┬──────┘
              └────────────────────┼────────────────────┘
                                   ▼
                         ┌─────────────────┐
                         │  tagged_source  │
                         │  (有值则使用)   │
                         └────────┬────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │  tagged_source 有值吗？   │
                    └─────────────┬─────────────┘
                                  │
           ┌──────────────────────┴──────────────────────┐
           ▼                                             ▼
    ┌─────────────┐                              ┌─────────────────┐
    │  使用标记   │                              │  检查 Referer   │
    │  (不考虑Ref)│                              │   头有效性      │
    └─────────────┘                              └────────┬────────┘
                                                          │
                                              ┌───────────┴───────────┐
                                              │ 是有效的外部引用吗？  │
                                              └───────────┬───────────┘
                                                          │
                                    ┌─────────────────────┴─────────────────────┐
                                    ▼                                           ▼
                             ┌─────────────┐                            ┌─────────────┐
                             │  解析来源   │                            │  无来源     │
                             │ RefInspector│                            │ source=nil  │
                             └─────────────┘                            └─────────────┘
```

### 4.2 来源解析实现

**关键代码** (`source.ex:66-80`):

```elixir
def resolve(request) do
  # 步骤 1: 检查 URL 标记参数
  tagged_source =
    request.query_params["utm_source"] ||
      request.query_params["source"] ||
      request.query_params["ref"]

  source =
    cond do
      tagged_source -> tagged_source  # 标记优先
      has_valid_referral?(request) -> parse(request.referrer)  # 自然来源
      true -> nil  # 直接访问
    end

  # 步骤 2: 标准化来源名称 (case-insensitive 映射)
  find_mapping(source)
end
```

**有效引用检查** (`source.ex:110-124`):

```elixir
defp has_valid_referral?(%Request{referrer: referrer, uri: uri}) do
  referrer_uri = URI.parse(referrer)

  valid_scheme? = referrer_uri.scheme in ["http", "https", "android-app"]
  valid_host? = !is_nil(referrer_uri.host) && byte_size(referrer_uri.host) > 0

  # 内部跳转不算来源
  internal? =
    Request.sanitize_hostname(referrer_uri.host) == Request.sanitize_hostname(uri.host)

  # localhost 不算来源
  local? = referrer_uri.host == "localhost"

  valid_scheme? and valid_host? and not internal? and not local?
end
```

### 4.3 来源标准化与映射

**映射机制** (`source.ex:13-43`):

系统维护三层来源映射：

1. **RefInspector 内置来源** (`priv/ref_inspector/referers.yml`)
   - 例如：`google.co.uk` → `"Google"`，`facebook.com` → `"Facebook"`

2. **自定义来源** (`priv/custom_sources.json`)
   - 扩展 RefInspector 未覆盖的来源
   - 包含常见的 UTM 简写映射

3. **付费来源识别** (`source.ex:13-16`):

```elixir
@paid_sources Map.keys(@custom_sources)
              |> Enum.filter(&String.ends_with?(&1, ["ads", "ad"]))
              |> then(&["adwords" | &1])
              |> MapSet.new()

# 付费来源包括：adwords, google-ads, bing-ads, facebook-ads 等
```

### 4.4 点击 ID 与媒介推断

**点击 ID 支持** (`event.ex:435-442`):

```elixir
@click_id_params ["gclid", "gbraid", "wbraid", "msclkid", "fbclid", "twclid"]

defp get_click_id_param(query_params) do
  @click_id_params
  |> Enum.find(fn param_name -> Map.has_key?(query_params, param_name) end)
end
```

| 点击 ID | 广告平台 | 用途 |
|---------|---------|-----|
| `gclid` | Google Ads | 广告点击追踪 |
| `gbraid` | Google Ads | 应用到网络转化 |
| `wbraid` | Google Ads | 网络到应用转化 |
| `msclkid` | Microsoft/Bing Ads | 微软广告追踪 |
| `fbclid` | Meta (Facebook/Instagram) | Meta 广告 |
| `twclid` | Twitter/X Ads | X 平台广告 |

**媒介自动推断** (`event.ex:312-322`):

```elixir
defp maybe_infer_medium(%__MODULE__{} = event, _context) do
  inferred_medium =
    case event.clickhouse_session_attrs do
      # 显式 utm_medium 优先
      %{utm_medium: medium} when is_binary(medium) -> medium

      # Google + gclid = CPC 推断
      %{utm_medium: nil, referrer_source: "Google", click_id_param: "gclid"} -> "(gclid)"

      # Bing + msclkid = CPC 推断
      %{utm_medium: nil, referrer_source: "Bing", click_id_param: "msclkid"} -> "(msclkid)"

      _ -> nil
    end

  update_session_attrs(event, %{utm_medium: inferred_medium})
end
```

### 4.5 请求信息边界 - 访客归因

**入站信息**：
| 信息 | 来源 | 处理方式 |
|-----|------|---------|
| `query_params` | URL query string | 提取 UTM 参数、点击 ID |
| `referrer` | HTTP `Referer` 头 | 解析来源域名，匹配已知来源 |
| `uri` | 请求 URL | 用于检测内部跳转 |

**存储边界** (`ClickhouseEventV2`  Schema):

```elixir
# 存储的归因字段
field :referrer, :string              # 完整引用路径 (如 google.com/search?q=...)
field :referrer_source, :string       # 标准化来源名 (如 "Google", "Facebook")
field :click_id_param, Ch, type: "LowCardinality(String)"  # 如 "gclid"
field :utm_medium, :string            # 如 "cpc", "organic", "(gclid)"
field :utm_source, :string            # 如 "google", "newsletter"
field :utm_campaign, :string          # 广告活动名
field :utm_content, :string           # 广告内容
field :utm_term, :string              # 搜索关键词
```

**准确性取舍**：

| 取舍点 | 设计选择 | 准确性收益 | 潜在问题 |
|-------|---------|-----------|---------|
| UTM 优先于 Referer | 标记优先 | 广告投放追踪准确 | 可能忽略自然来源变化 |
| 规范化来源名称 | `google.co.uk` → `"Google"` | 跨域聚合统计准确 | 损失具体搜索引擎子域信息 |
| 点击 ID 推断媒介 | `gclid` → `"(gclid)"` | 无 UTM 时也能识别 CPC | 依赖平台点击 ID 存在 |
| 排除内部跳转 | 同域 Referer 不算来源 | 避免内部导航误判 | 单页应用可能需要特殊处理 |

---

## 5. 入库判断与 Pipeline 协同

### 5.1 完整处理流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          事件处理 Pipeline 完整流程                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────┐                                                           │
│  │  HTTP Request│                                                           │
│  └──────┬───────┘                                                           │
│         ▼                                                                   │
│  ┌────────────────────────────────────────────────────────────────────┐   │
│  │ lib/plausible/ingestion/request.ex: build()                        │   │
│  │ - 解析请求体 JSON                                                    │   │
│  │ - 提取 remote_ip, user_agent, referrer                              │   │
│  │ - 读取 x-plausible-ip-type 头 → ip_classification                   │   │
│  │ - 验证必填字段                                                        │   │
│  └───────────────────────────────┬────────────────────────────────────┘   │
│                                  ▼                                            │
│  ┌────────────────────────────────────────────────────────────────────┐   │
│  │ lib/plausible/ingestion/event.ex: build_and_buffer()               │   │
│  │                                                                      │   │
│  │  前置检查:                                                           │   │
│  │  ┌──────────────────────────────────────────────────────────────┐  │   │
│  │  │ 1. spam_referrer?() - 检查 Referer 是否在垃圾列表              │  │   │
│  │  │    是 → drop_all(:spam_referrer)                               │  │   │
│  │  └──────────────────────────────────────────────────────────────┘  │   │
│  │                                                                      │   │
│  │  逐个 domain 处理:                                                   │   │
│  │  ┌──────────────────────────────────────────────────────────────┐  │   │
│  │  │ GateKeeper.check(domain)                                       │  │   │
│  │  │ - {:allow, site} → 继续 Pipeline                                │  │   │
│  │  │ - {:deny, reason} → drop(:policy_reason)                       │  │   │
│  │  └───────────────────────────┬──────────────────────────────────┘  │   │
│  │                              ▼                                        │   │
│  │  ┌──────────────────────────────────────────────────────────────┐  │   │
│  │  │ Pipeline 执行 (Enum.reduce_while - 遇到 drop 则 halt)         │  │   │
│  │  │                                                              │  │   │
│  │  │ 【第一层：基础设施过滤】                                       │  │   │
│  │  │ 1. drop_verification_agent()                                  │  │   │
│  │  │    - 匹配内部验证 UA → drop(:verification_agent)              │  │   │
│  │  │                                                              │  │   │
│  │  │ 2. drop_datacenter_ip()                                       │  │   │
│  │  │    - ip_classification == "dc_ip" → drop(:dc_ip)              │  │   │
│  │  │                                                              │  │   │
│  │  │ 3. drop_threat_ip()                                           │  │   │
│  │  │    - ip_classification == "threat_ip" → drop(:threat_ip)     │  │   │
│  │  │                                                              │  │   │
│  │  └───────────────────────────┬──────────────────────────────────┘  │   │
│  │                              ▼                                        │   │
│  │  ┌──────────────────────────────────────────────────────────────┐  │   │
│  │  │ 【第二层：Shields 规则引擎】                                   │  │   │
│  │  │ 4. drop_shield_rule_hostname()                                 │  │   │
│  │  │    - 检查 hostname 白名单 → 不在列表则 drop                     │  │   │
│  │  │                                                              │  │   │
│  │  │ 5. drop_shield_rule_page()                                     │  │   │
│  │  │    - 检查 pathname 黑名单 → 匹配则 drop                         │  │   │
│  │  │                                                              │  │   │
│  │  │ 6. drop_shield_rule_ip()                                       │  │   │
│  │  │    - 检查 IP 黑名单 → 匹配则 drop                               │  │   │
│  │  │                                                              │  │   │
│  │  └───────────────────────────┬──────────────────────────────────┘  │   │
│  │                              ▼                                        │   │
│  │  ┌──────────────────────────────────────────────────────────────┐  │   │
│  │  │ 【第三层：地理位置 + Bot 检测】                                │  │   │
│  │  │ 7. put_geolocation()                                           │  │   │
│  │  │    - anonymous_vpn_ip → country_code = "A1"                   │  │   │
│  │  │    - 其他 → GeoIP lookup                                        │  │   │
│  │  │                                                              │  │   │
│  │  │ 8. drop_shield_rule_country()                                  │  │   │
│  │  │    - 检查国家黑名单 → 匹配则 drop                               │  │   │
│  │  │                                                              │  │   │
│  │  │ 9. put_user_agent()  ⭐ 关键 Bot 检测点                      │  │   │
│  │  │    - Headless Chrome → drop(:bot)                              │  │   │
│  │  │    - UAInspector.Result.Bot{} → drop(:bot)                     │  │   │
│  │  │    - 正常 UA → 提取 browser/os/screen_size                      │  │   │
│  │  │                                                              │  │   │
│  │  └───────────────────────────┬──────────────────────────────────┘  │   │
│  │                              ▼                                        │   │
│  │  ┌──────────────────────────────────────────────────────────────┐  │   │
│  │  │ 【第四层：属性填充 + 归因】                                    │  │   │
│  │  │ 10. put_basic_info()                                           │  │   │
│  │  │     - domain, site_id, timestamp, name, pathname 等            │  │   │
│  │  │                                                              │  │   │
│  │  │ 11. put_source_info()  ⭐ 归因核心                            │  │   │
│  │  │     - Source.resolve() → referrer_source                       │  │   │
│  │  │     - Source.format_referrer() → referrer                      │  │   │
│  │  │     - 提取所有 UTM 参数 + click_id_param                        │  │   │
│  │  │                                                              │  │   │
│  │  │ 12. maybe_infer_medium()                                       │  │   │
│  │  │     - gclid + Google → "(gclid)"                               │  │   │
│  │  │     - msclkid + Bing → "(msclkid)"                              │  │   │
│  │  │                                                              │  │   │
│  │  │ 13. put_props(), put_revenue()                                 │  │   │
│  │  │     - 自定义事件属性、收入数据                                   │  │   │
│  │  │                                                              │  │   │
│  │  └───────────────────────────┬──────────────────────────────────┘  │   │
│  │                              ▼                                        │   │
│  │  ┌──────────────────────────────────────────────────────────────┐  │   │
│  │  │ 【第五层：隐私保护 + 持久化】                                  │  │   │
│  │  │ 14. put_salts()                                                │  │   │
│  │  │     - 从 ETS 获取 {current, previous} 盐值                     │  │   │
│  │  │                                                              │  │   │
│  │  │ 15. put_user_id()  ⭐ 隐私核心                                │  │   │
│  │  │     - SipHash(current_salt, ua + ip + domain + root_domain)   │  │   │
│  │  │                                                              │  │   │
│  │  │ 16. validate_clickhouse_event()                               │  │   │
│  │  │     - Ecto Changeset 验证必填字段                              │  │   │
│  │  │                                                              │  │   │
│  │  │ 17. register_session()                                        │  │   │
│  │  │     - 使用 previous_salt 生成 previous_user_id                 │  │   │
│  │  │     - 调用 Persistor.persist_event() 入库                      │  │   │
│  │  │                                                              │  │   │
│  │  └───────────────────────────┬──────────────────────────────────┘  │   │
│  │                              ▼                                        │   │
│  │  ┌──────────────────────────────────────────────────────────────┐  │   │
│  │  │ 【输出】                                                       │  │   │
│  │  │ - 通过所有步骤 → emit_telemetry(:buffered)                     │  │   │
│  │  │ - 被 drop → emit_telemetry(:dropped, reason)                   │  │   │
│  │  └──────────────────────────────────────────────────────────────┘  │   │
│  │                                                                      │   │
│  └───────────────────────────────┬────────────────────────────────────┘   │
│                                  ▼                                            │
│  ┌────────────────────────────────────────────────────────────────────┐   │
│  │ 分类返回: {dropped: [...], buffered: [...]}                         │   │
│  │ - dropped 事件记录 drop_reason 用于监控                              │   │
│  │ - buffered 事件写入 ClickHouse                                        │   │
│  └────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 丢弃原因枚举

**所有可能的丢弃原因** (`event.ex:24-40`):

```elixir
@type drop_reason() ::
        :bot                      # Bot UA 或 Headless Chrome
        | :spam_referrer          # 垃圾引用域名
        | GateKeeper.policy()     # 站点访问策略拒绝
        | :invalid                 # 数据验证失败
        | :dc_ip                   # 数据中心 IP
        | :threat_ip              # 威胁 IP
        | :site_ip_blocklist       # 站点 IP 黑名单
        | :site_country_blocklist  # 站点国家黑名单
        | :site_page_blocklist     # 站点页面黑名单
        | :site_hostname_allowlist # 主机名不在白名单
        | :verification_agent      # 安装验证代理
        | :lock_timeout            # 锁超时
        | :no_session_for_engagement  # 无对应会话
        | :persist_timeout         # 持久化超时
        | :persist_error           # 持久化错误
        | :persist_decode_error    # 持久化解码错误
```

### 5.3 三层协同的决策边界

| 阶段 | 模块 | 决策依据 | 信息边界 |
|-----|------|---------|---------|
| **过滤** | Bot/IP 检测 | User-Agent 模式、IP 信誉、Shields 规则 | 原始请求元数据，不修改 |
| **归因** | Source 解析 | UTM 参数、Referer 头、已知来源映射 | 规范化后的数据，可能推断补充 |
| **隐私** | User ID 生成 | 盐值 + 指纹因子 (UA + IP + Domain) | 不可逆哈希，无原始信息存储 |

### 5.4 关键决策点的准确性取舍分析

#### 决策点 1: Bot 检测的假阳性/假阴性

**当前设计**：
- ✅ 强过滤：数据中心 IP (`dc_ip`) 直接丢弃
- ⚠️ 弱过滤：UA 匹配失败不丢弃，超时也不丢弃
- ❌ 无法检测：自定义 UA 字符串、真实用户被误判为 Bot

**取舍分析**：

```
                    ┌──────────────────────────────────┐
                    │        准确性 - 隐私 权衡        │
                    └──────────────────────────────────┘
                                   │
         ┌─────────────────────────┼─────────────────────────┐
         ▼                         ▼                         ▼
   ┌───────────┐           ┌───────────┐            ┌───────────┐
   │  误判为   │           │   理想    │            │  漏判为   │
   │   Bot     │           │   状态    │            │  正常用户 │
   └─────┬─────┘           └─────┬─────┘            └─────┬─────┘
         │                       │                       │
         ▼                       ▼                       ▼
   ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
   │ 数据准确性下降  │    │  平衡状态       │    │  数据被污染     │
   │ 真实用户数据丢失│    │                 │    │ Bot 数据混入统计│
   └─────────────────┘    └─────────────────┘    └─────────────────┘
         ▲                                              ▲
         │                                              │
    ┌────┴────┐                                    ┌────┴────┐
    │ dc_ip   │                                    │ UA 伪造 │
    │ 丢弃    │                                    │  绕过   │
    └─────────┘                                    └─────────┘
```

**代码中的证据** (`event.ex:444-470`):

```elixir
# UA 解析超时处理 - 超时不丢弃，只是不解析设备信息
case Task.yield(task, @parse_user_agent_timeout) || Task.shutdown(task) do
  {:ok, result} -> {:ok, result}
  nil ->
    emit_telemetry_ua_parse_timeout()
    {:error, :timeout}  # 注意：返回 error 但不 drop 事件！
end
```

#### 决策点 2: 盐值轮换与用户追踪连续性

**当前设计**：
- 盐值每 90 秒刷新（可配置）
- 保留 2 个盐值 (`current` + `previous`)
- 48 小时后删除所有旧盐值

**取舍分析**：

| 维度 | 48 小时轮换 | 如果不轮换（固定盐值） |
|-----|------------|----------------------|
| **隐私保护** | ✅ 无法跨天追踪同一用户 | ❌ 永久可关联用户行为 |
| **会话连续性** | ⚠️ 需双盐值处理 | ✅ 完美连续 |
| **实现复杂度** | 较高（双盐值、会话查找） | 简单 |
| **合规性** | 符合 GDPR "被遗忘权" | 有合规风险 |

#### 决策点 3: 归因中的 UTM 优先原则

**当前设计**：
```elixir
tagged_source =
  request.query_params["utm_source"] ||
    request.query_params["source"] ||
    request.query_params["ref"]

source =
  cond do
    tagged_source -> tagged_source  # 只要有标记，就用标记
    has_valid_referral?(request) -> parse(request.referrer)
    true -> nil
  end
```

**取舍分析**：

| 场景 | UTM 优先结果 | 准确性影响 |
|-----|------------|-----------|
| 用户点击广告 → 直接访问 | 使用广告来源 | ✅ 正确（最后非直接触点归因） |
| 用户点击广告 → 点击自然搜索 | **仍使用广告来源** | ⚠️ 可能掩盖自然流量效果 |
| 营销人员手动加 UTM | 按 UTM 统计 | ✅ 符合营销预期 |
| 分享链接带 UTM | 按 UTM 统计 | ⚠️ 可能不是真实来源 |

---

## 6. 模块交互与数据流

### 6.1 核心模块依赖关系

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          模块依赖关系图                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                        接入层 (Plug/Controller)                    │  │
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐ │  │
│  │  │ NoRobots    │    │ RemoteIP    │    │ StatsController     │ │  │
│  │  │ (响应头)    │    │ (IP 提取)   │    │ (API 入口)          │ │  │
│  │  └─────────────┘    └─────────────┘    └──────────┬──────────┘ │  │
│  └─────────────────────────────────────────────────────┼────────────┘  │
│                                                        ▼                │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                        摄入层 (Ingestion)                           │  │
│  │                                                                      │  │
│  │   ┌──────────────┐      ┌──────────────┐      ┌──────────────┐  │  │
│  │   │   Request    │─────▶│    Event     │─────▶│  Persistor   │  │  │
│  │   │   (构建)     │      │  (Pipeline)  │      │  (持久化)    │  │  │
│  │   └──────────────┘      └──────┬───────┘      └──────┬───────┘  │  │
│  │                                │                     │           │  │
│  │   ┌────────────────────────────┼─────────────────────┘           │  │
│  │   ▼                            ▼                                  │  │
│  │ ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐│  │
│  │ │  Source  │  │Geolocation│ │UAInspector│ │ Session.Salts     ││  │
│  │ │ (归因)   │  │ (地理)   │  │ (Bot检测) │ │ (盐值管理)        ││  │
│  │ └──────────┘  └──────────┘  └──────────┘  └──────────────────┘│  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                        │                │
│                                                        ▼                │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                        存储层 (Data Layer)                          │  │
│  │  ┌──────────────┐         ┌─────────────────────────────────┐    │  │
│  │  │ClickhouseEvent│         │           PostgreSQL            │    │  │
│  │  │  (事件数据)   │         │  ┌─────────┐  ┌─────────────┐ │    │  │
│  │  │              │         │  │  salts  │  │ site_shields│ │    │  │
│  │  │  - user_id   │         │  │(盐值表) │  │ (规则配置)  │ │    │  │
│  │  │  - session_id│         │  └─────────┘  └─────────────┘ │    │  │
│  │  │  - 归因字段  │         └─────────────────────────────────┘    │  │
│  │  └──────────────┘                                                │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 关键数据流路径

**路径 1: 正常用户事件 (从请求到入库)**

```
HTTP Request
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 1. Request.build()                                                    │
│    - 提取 remote_ip = "203.0.113.42"                                 │
│    - 提取 user_agent = "Mozilla/5.0 (Windows NT 10.0; Win64; x64)..."│
│    - 提取 referrer = "https://www.google.com/search?q=plausible"     │
│    - 提取 query_params = %{"utm_source" => "google", "gclid" => "abc123"}│
│    - ip_classification = nil (未被标记)                               │
└───────────────────────────────────┬─────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 2. Event.build_and_buffer() - Pipeline 执行                          │
│                                                                      │
│    2.1 spam_referrer?("google.com") → false                          │
│    2.2 GateKeeper.check("example.com") → {:allow, site}              │
│                                                                      │
│    2.3 Pipeline 步骤:                                                 │
│        - drop_verification_agent() → 通过 (UA 不匹配)                 │
│        - drop_datacenter_ip() → 通过 (ip_classification != dc_ip)    │
│        - drop_threat_ip() → 通过                                      │
│        - Shields 规则检查 → 全部通过                                  │
│                                                                      │
│        - put_geolocation() → GeoIP  lookup → %{country_code: "US"}  │
│        - put_user_agent() → UAInspector 解析:                         │
│            → %UAInspector.Result{client: %{name: "Chrome"}, ...}    │
│            → 不是 Bot，更新 session_attrs                              │
│                                                                      │
│        - put_source_info() → 归因核心:                                 │
│            → utm_source = "google" (有值，优先)                       │
│            → referrer_source = Source.resolve() → "Google"           │
│            → click_id_param = "gclid"                                 │
│            → utm_campaign = nil (未设置)                              │
│                                                                      │
│        - maybe_infer_medium() → utm_medium = nil → 检查:             │
│            → referrer_source = "Google" + click_id_param = "gclid"   │
│            → 推断 utm_medium = "(gclid)"                              │
│                                                                      │
│        - put_salts() → 从 ETS 获取:                                   │
│            → %{current: <<salt_bytes_1>>, previous: <<salt_bytes_2>>}│
│                                                                      │
│        - put_user_id() → 计算:                                        │
│            → SipHash(current_salt, "Mozilla/5.0..." <> "203.0.113.42"│
│            →                      <> "example.com" <> "example.com")  │
│            → user_id = 123456789012345 (UInt64)                     │
│                                                                      │
│        - validate_clickhouse_event() → 通过                           │
│                                                                      │
│        - register_session() →                                         │
│            → previous_user_id = SipHash(previous_salt, ...)          │
│            → 调用 Persistor.persist_event()                           │
│            → {:ok, event} → emit_telemetry_buffered()                │
└───────────────────────────────────┬─────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 3. 数据入库 (ClickHouse)                                               │
│    存储字段:                                                           │
│    - name: "pageview"                                                 │
│    - site_id: 1                                                        │
│    - user_id: 123456789012345  ← 哈希值，不可逆                     │
│    - timestamp: ~N[2026-05-03 10:00:00]                              │
│    - referrer: "google.com/search?q=plausible"                        │
│    - referrer_source: "Google"                                         │
│    - utm_source: "google"                                              │
│    - utm_medium: "(gclid)"  ← 推断的媒介                              │
│    - click_id_param: "gclid"                                           │
│    - country_code: "US"                                                 │
│    - browser: "Chrome"                                                  │
│    - operating_system: "Windows"                                        │
│    - screen_size: "Desktop"                                             │
└─────────────────────────────────────────────────────────────────────┘
```

**路径 2: Bot 请求 (被丢弃)**

```
HTTP Request (Bot)
     │
     │ user_agent = "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)"
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 1. Request.build()                                                    │
│    - ip_classification = 可能是 "dc_ip" 或 nil                       │
│    - user_agent = "Googlebot/2.1..."                                 │
└───────────────────────────────────┬─────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 2. Event Pipeline - 可能在两个点被丢弃                                │
│                                                                      │
│    场景 A: 数据中心 IP + Bot UA                                        │
│    ┌──────────────────────────────────────────────────────────────┐ │
│    │ - drop_datacenter_ip() → ip_classification == "dc_ip"        │ │
│    │   → drop(event, :dc_ip) ← 在 L2 就被丢弃，不执行后续步骤       │ │
│    └──────────────────────────────────────────────────────────────┘ │
│                                                                      │
│    场景 B: 普通 IP 但 Bot UA                                          │
│    ┌──────────────────────────────────────────────────────────────┐ │
│    │ - 前置步骤全部通过                                              │ │
│    │ - put_user_agent() → UAInspector.parse():                      │ │
│    │     → %UAInspector.Result.Bot{name: "Googlebot"}              │ │
│    │     → drop(event, :bot) ← 在这里被丢弃                         │ │
│    └──────────────────────────────────────────────────────────────┘ │
│                                                                      │
│    场景 C: Headless Chrome                                            │
│    ┌──────────────────────────────────────────────────────────────┐ │
│    │ - put_user_agent() → UAInspector.Result:                       │ │
│    │     → client.name = "Headless Chrome"                          │ │
│    │     → drop(event, :bot) ← 无头浏览器也被视为 Bot               │ │
│    └──────────────────────────────────────────────────────────────┘ │
│                                                                      │
│   所有场景最终:                                                        │
│   - emit_telemetry_dropped(event, reason)                            │
│   - 事件加入 dropped 列表，不入库                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**路径 3: 匿名 VPN 用户**

```
HTTP Request (通过 VPN)
     │
     │ x-plausible-ip-type = "anonymous_vpn_ip" (反向代理注入)
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 1. Request.build()                                                    │
│    - ip_classification = "anonymous_vpn_ip"                          │
└───────────────────────────────────┬─────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 2. Event Pipeline                                                     │
│                                                                      │
│    - drop_datacenter_ip() → 不是 "dc_ip"，通过                       │
│    - drop_threat_ip() → 不是 "threat_ip"，通过                       │
│    ... 其他 Shields 检查 ...                                         │
│                                                                      │
│    - put_geolocation() → 关键分支:                                   │
│      ┌──────────────────────────────────────────────────────────┐   │
│      │ case event.request.ip_classification do                    │   │
│      │   "anonymous_vpn_ip" →                                     │   │
│      │     update_session_attrs(event, %{country_code: "A1"})   │   │
│      │     ↑ 不进行 GeoIP lookup，直接设置为 "A1"                 │   │
│      │ end                                                         │   │
│      └──────────────────────────────────────────────────────────┘   │
│                                                                      │
│    - 后续步骤正常执行:                                                │
│      → put_user_agent() (如果不是 Bot)                               │
│      → put_source_info() (归因正常工作)                              │
│      → put_user_id() (仍使用 IP 进行哈希，但 IP 是 VPN 出口)         │
│                                                                      │
│    - 最终入库，但是:                                                  │
│      - country_code = "A1" (匿名网络，不是真实国家)                  │
│      - 没有 city_geoname_id、subdivision_code 等精确地理信息         │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. 总结与设计洞察

### 7.1 三大模块的协同原则

| 模块 | 核心目标 | 与其他模块的交互 | 设计哲学 |
|-----|---------|-----------------|---------|
| **Bot 过滤** | 确保数据纯净性 | 在 Pipeline 早期执行，减少无效计算 | **宁可漏判，不可误杀** (除了 dc_ip) |
| **隐私保护** | 合规 + 用户信任 | 最后阶段生成 user_id，使用盐值隔离 | **隐私-by-design，不可逆设计** |
| **访客归因** | 营销洞察准确性 | 依赖原始 query_params 和 referrer | **标记优先，推断补充** |

### 7.2 关键设计决策的权衡

#### 决策 1: 数据中心 IP 直接丢弃 vs 保留分析

**代码证据** (`event.ex:212-220`):
```elixir
defp drop_datacenter_ip(%__MODULE__{} = event, _context) do
  case event.request.ip_classification do
    "dc_ip" -> drop(event, :dc_ip)  # 无条件丢弃
    _any -> event
  end
end
```

**权衡分析**：

| 维度 | 直接丢弃 | 保留但标记 |
|-----|---------|-----------|
| **数据质量** | ✅ 避免 Bot 污染 | ⚠️ 需额外过滤逻辑 |
| **分析灵活性** | ❌ 无法分析云服务流量 | ✅ 可选择是否包含 |
| **性能** | ✅ 早期丢弃，节省资源 | ⚠️ 需处理到入库 |
| **误判风险** | ❌ 可能丢弃真实用户 (企业代理) | ✅ 可事后调整 |

**洞察**：这是一个 **"运营效率优先"** 的决策。数据中心 IP 中 Bot 比例极高，但也包含：
- 通过 AWS/GCP 代理访问的真实用户
- 企业 VPN 出口（可能被标记为 dc_ip）

#### 决策 2: 双盐值设计 vs 单盐值

**代码证据** (`event.ex:414-424`, `salts.ex:28-48`):

```elixir
# 持久化时使用 previous_user_id 进行会话查找
previous_user_id =
  generate_user_id(
    event.request,
    event.domain,
    event.clickhouse_event.hostname,
    event.salts.previous  # 注意：previous 盐值
  )
```

**权衡分析**：

| 维度 | 双盐值 (current + previous) | 单盐值 |
|-----|---------------------------|--------|
| **会话连续性** | ✅ 盐值轮换时无缝过渡 | ❌ 轮换时刻会话断裂 |
| **实现复杂度** | ⚠️ 较高 (需处理两个 user_id) | ✅ 简单 |
| **隐私窗口** | ⚠️ 略微延长 (90s overlap) | ✅ 精确 48 小时 |
| **数据一致性** | ⚠️ 需处理两种查找路径 | ✅ 单一逻辑 |

**洞察**：这是一个 **"用户体验优先"** 的决策。90 秒的 overlap 窗口：
- 避免了盐值轮换时刻的用户"突然消失"
- 保持了会话指标的连续性
- 隐私成本极低（仅延长 90 秒）

#### 决策 3: UTM 优先于 Referer vs 末次触点归因

**代码证据** (`source.ex:66-80`):

```elixir
source =
  cond do
    tagged_source -> tagged_source  # 只要有 UTM，就忽略 Referer
    has_valid_referral?(request) -> parse(request.referrer)
    true -> nil
  end
```

**权衡分析**：

| 场景 | UTM 优先结果 | 末次触点 (Referer) 结果 |
|-----|------------|----------------------|
| 广告 → 直接 | 广告来源 | 直接访问 |
| 广告 → 搜索 → 网站 | 广告来源 | 搜索引擎 |
| 搜索 → 广告 → 网站 | 广告来源 | 广告来源 |
| 分享链接带 UTM | UTM 指定来源 | 分享者来源 |

**洞察**：这是一个 **"营销人员视角"** 的决策。
- 营销人员希望知道 **"是哪个活动带来了用户"**
- 而不是 **"用户最后从哪里来"**
- 这符合广告平台的归因模型（Google Ads、Meta 等都使用 first/last campaign touch）

### 7.3 潜在改进点

基于代码分析，以下是可能的优化方向：

1. **Bot 检测增强**
   - 当前仅依赖 UA 和 IP 分类
   - 可考虑添加：行为分析 (页面停留时间、滚动深度)、JS 指纹检测

2. **盐值策略可配置**
   - 当前硬编码 48 小时清理、90 秒刷新
   - 可让用户根据合规需求调整

3. **归因模型选择**
   - 当前硬编码 UTM 优先
   - 可支持多种归因模型：末次触点、首次触点、线性等

4. **dc_ip 处理策略**
   - 当前无条件丢弃
   - 可改为：可配置规则、仅丢弃高风险 DC 等

---

## 附录：关键文件索引

| 文件路径 | 职责 | 关键行号 |
|---------|------|---------|
| `lib/plausible/ingestion/event.ex` | Pipeline 核心 | 130-151 (pipeline), 256-276 (bot检测), 553-567 (user_id) |
| `lib/plausible/ingestion/request.ex` | 请求构建 | 76-124 (build), 347-363 (ip_classification) |
| `lib/plausible/ingestion/source.ex` | 归因逻辑 | 66-80 (resolve), 110-124 (valid_referral) |
| `lib/plausible/session/salts.ex` | 盐值管理 | 28-48 (refresh), 80-91 (cleanup) |
| `lib/plausible/ingestion/geolocation.ex` | 地理处理 | 4-25 (lookup) |
| `lib/plausible/clickhouse_event_v2.ex` | 数据 Schema | 8-53 (schema) |
| `priv/ua_inspector/bot.bots.yml` | Bot 规则库 | 全文 |
