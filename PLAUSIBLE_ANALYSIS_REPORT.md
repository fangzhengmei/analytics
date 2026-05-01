# Plausible Analytics 核心机制分析报告

## 目录
1. [机器人流量过滤机制](#1-机器人流量过滤机制)
2. [用户隐私保护机制](#2-用户隐私保护机制)
3. [事件归因边界处理](#3-事件归因边界处理)

---

## 1. 机器人流量过滤机制

### 1.1 多层过滤架构

Plausible Analytics 采用多层防御机制过滤机器人流量，从不同维度识别和排除非人类访问。

#### 过滤层次结构图

```
┌─────────────────────────────────────────────────────────────┐
│                    事件处理管道 (Pipeline)                     │
├─────────────────────────────────────────────────────────────┤
│  1. drop_verification_agent  ← 过滤安装验证代理               │
│  2. drop_datacenter_ip       ← 过滤数据中心 IP                │
│  3. drop_threat_ip           ← 过滤威胁 IP                    │
│  4. drop_shield_rule_hostname ← 主机名白名单过滤               │
│  5. drop_shield_rule_page    ← 页面路径屏蔽                   │
│  6. drop_shield_rule_ip      ← IP 地址屏蔽                    │
│  7. put_geolocation          ← 地理位置解析                    │
│  8. drop_shield_rule_country ← 国家/地区屏蔽                  │
│  9. put_user_agent           ← User-Agent 解析 + 机器人检测   │
│  ...                                                          │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 机器人识别判定规则

#### 1.2.1 User-Agent 检测 (核心机制)

**位置**: `lib/plausible/ingestion/event.ex:256-276`

```elixir
defp put_user_agent(%__MODULE__{} = event, _context) do
  case parse_user_agent(event.request) do
    {:ok, %UAInspector.Result{client: %UAInspector.Result.Client{name: "Headless Chrome"}}} ->
      drop(event, :bot)

    {:ok, %UAInspector.Result.Bot{}} ->
      drop(event, :bot)

    {:ok, %UAInspector.Result{} = user_agent} ->
      update_session_attrs(event, %{
        operating_system: os_name(user_agent),
        operating_system_version: os_version(user_agent),
        browser: browser_name(user_agent),
        browser_version: browser_version(user_agent),
        screen_size: screen_size(user_agent)
      })

    _any ->
      event
  end
end
```

**判定规则**:
1. **Headless Chrome 检测**: 专门检测 `Headless Chrome`，这是自动化测试和爬虫常用的浏览器模式
2. **UAInspector Bot 类型**: 任何被解析为 `%UAInspector.Result.Bot{}` 的请求都会被丢弃
3. **超时保护**: User-Agent 解析设置 200ms 超时，防止解析卡住

#### 1.2.2 机器人规则库

**位置**: `priv/ua_inspector/bot.bots.yml`

基于 [Device Detector (Matomo)](https://matomo.org) 的机器人分类体系，包含以下类型：

| 类别 | 说明 | 示例 |
|------|------|------|
| `Search bot` | 搜索引擎爬虫 | Googlebot, Bingbot, Baiduspider |
| `Crawler` | 通用爬虫 | AhrefsBot, Amazonbot |
| `AI Search Crawler` | AI 搜索爬虫 | Applebot, Amazonbot |
| `AI Data Scraper` | AI 数据抓取 | Applebot-Extended |
| `Site Monitor` | 站点监控 | 360 Monitoring, UptimeRobot |
| `Feed Fetcher` | RSS 抓取 | WireReaderBot |
| `Social Media Agent` | 社交媒体代理 | AddThis.com, FacebookExternalHit |
| `Service Agent` | 服务代理 | Cloudflare-Healthchecks, Apache |
| `Benchmark` | 基准测试工具 | ApacheBench |

**规则示例**:
```yaml
- regex: 'AhrefsBot'
  name: 'aHrefs Bot'
  category: 'Crawler'
  url: 'https://ahrefs.com/robot'

- regex: 'HeadlessChrome'
  name: 'Headless Chrome'
  category: 'Browser'  # 但在代码中被特殊处理为 bot
```

#### 1.2.3 IP 分类过滤

**位置**: `lib/plausible/ingestion/event.ex:212-230`

```elixir
defp drop_datacenter_ip(%__MODULE__{} = event, _context) do
  case event.request.ip_classification do
    "dc_ip" ->
      drop(event, :dc_ip)
    _any ->
      event
  end
end

defp drop_threat_ip(%__MODULE__{} = event, _context) do
  case event.request.ip_classification do
    "threat_ip" ->
      drop(event, :threat_ip)
    _any ->
      event
  end
end
```

**IP 分类类型**:
- `dc_ip`: 数据中心 IP（云服务商、主机托管）
- `threat_ip`: 威胁 IP（已知恶意 IP）
- `anonymous_vpn_ip`: 匿名 VPN/代理（特殊处理）

**来源**: 通过请求头 `x-plausible-ip-type` 传入，通常由反向代理或 CDN 进行 IP 信誉评估。

#### 1.2.4 Shields 用户自定义屏蔽

**位置**: `lib/plausible/shields.ex`

提供四种用户可配置的屏蔽规则：

##### IP 地址屏蔽
```elixir
def ip_blocked?(domain, address) when is_binary(domain) and is_binary(address) do
  case Shield.IPRuleCache.get({domain, address}) do
    %Shield.IPRule{action: :deny} -> true
    _ -> false
  end
end
```
- 最多 30 条规则
- 支持单个 IP 或 CIDR 范围

##### 国家/地区屏蔽
```elixir
def country_blocked?(domain, country_code) do
  case Shield.CountryRuleCache.get({domain, String.upcase(country_code)}) do
    %Shield.CountryRule{action: :deny} -> true
    _ -> false
  end
end
```
- 最多 30 条规则
- 基于 ISO 3166-1 alpha-2 国家代码

##### 页面路径屏蔽
```elixir
def page_blocked?(domain, pathname) do
  page_rules = Shield.PageRuleCache.get(domain)
  if page_rules do
    page_rules
    |> List.wrap()
    |> Enum.find_value(false, fn rule ->
      rule.action == :deny and Regex.match?(rule.page_path_pattern, pathname)
    end)
  else
    false
  end
end
```
- 最多 30 条规则
- 支持通配符模式（如 `/admin/**`, `/test/*/page`）

##### 主机名白名单
```elixir
def hostname_allowed?(domain, hostname) do
  hostname_rules = Shield.HostnameRuleCache.get(domain)
  if hostname_rules do
    hostname_rules
    |> List.wrap()
    |> Enum.find_value(false, fn rule ->
      rule.action == :allow and Regex.match?(rule.hostname_pattern, hostname)
    end)
  else
    true  # 默认允许所有
  end
end
```
- 最多 10 条规则
- 白名单模式：配置后只有匹配的主机名才被允许

#### 1.2.5 垃圾推荐过滤

**位置**: `lib/plausible/ingestion/event.ex:581-587`

```elixir
defp spam_referrer?(%Request{referrer: referrer}) when is_binary(referrer) do
  URI.parse(referrer).host
  |> Request.sanitize_hostname()
  |> ReferrerBlocklist.is_spammer?()
end
```

- 使用独立的 `referrer_blocklist` 库
- 过滤已知的垃圾推荐域名
- 在管道最早期检查，避免无效处理

#### 1.2.6 安装验证代理过滤

**位置**: `lib/plausible/ingestion/event.ex:199-206`

```elixir
on_ee do
  @verification_user_agent Plausible.InstallationSupport.user_agent()

  defp drop_verification_agent(%__MODULE__{} = event, _context) do
    case event.request.user_agent do
      @verification_user_agent ->
        drop(event, :verification_agent)
      _ ->
        event
    end
  end
end
```

- 仅企业版功能
- 过滤 Plausible 官方的安装验证请求

### 1.3 丢弃原因汇总

**位置**: `lib/plausible/ingestion/event.ex:24-41`

```elixir
@type drop_reason() ::
        :bot                    # 机器人
        | :spam_referrer        # 垃圾推荐
        | GateKeeper.policy()   # 站点访问控制
        | :invalid              # 无效数据
        | :dc_ip                # 数据中心 IP
        | :threat_ip            # 威胁 IP
        | :site_ip_blocklist    # 站点 IP 屏蔽
        | :site_country_blocklist # 国家屏蔽
        | :site_page_blocklist    # 页面屏蔽
        | :site_hostname_allowlist # 主机名不匹配
        | :verification_agent      # 验证代理
        | :lock_timeout            # 锁超时
        | :no_session_for_engagement # 无会话的互动事件
        | :persist_timeout         # 持久化超时
        | :persist_error           # 持久化错误
        | :persist_decode_error    # 持久化解码错误
```

---

## 2. 用户隐私保护机制

### 2.1 隐私设计原则

Plausible Analytics 以"隐私友好"为核心设计原则，通过以下方式保护用户隐私：
1. **无 Cookie 追踪**: 不使用任何持久化 Cookie
2. **数据最小化**: 仅收集必要的最小数据
3. **单向哈希**: 用户标识符无法反向还原
4. **定期重置**: 盐值轮换防止长期追踪
5. **匿名化处理**: 敏感信息在存储前处理

### 2.2 User ID 生成机制

#### 2.2.1 生成算法

**位置**: `lib/plausible/ingestion/event.ex:553-579`

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

**输入因子**:
| 因子 | 说明 | 来源 |
|------|------|------|
| `user_agent` | 用户代理字符串 | HTTP Header |
| `remote_ip` | 客户端 IP 地址 | 连接信息 |
| `domain` | 站点域名 | 请求参数 |
| `root_domain` | 根域名 | 从 hostname 解析 |
| `salt` | 每日轮换的盐值 | 数据库 + ETS 缓存 |

**算法特性**:
- **单向哈希**: SipHash 是密钥哈希函数，无法从输出反推输入
- **确定性**: 相同输入 + 相同 salt 产生相同 ID
- **碰撞抵抗**: 64 位输出，碰撞概率极低

#### 2.2.2 盐值轮换机制

**位置**: `lib/plausible/session/salts.ex`

```elixir
defmodule Plausible.Session.Salts do
  use GenServer
  use Plausible.Repo

  @impl true
  def init(opts) do
    name = opts[:name] || __MODULE__
    now = opts[:now] || DateTime.utc_now()
    clean_old_salts(now)

    ^name = :ets.new(name, [
      :named_table, :set, :protected, {:read_concurrency, true}
    ])

    refresh(name, now)
    {:ok, name}
  end

  def rotate(name \\ __MODULE__, now \\ DateTime.utc_now()) do
    GenServer.call(name, {:rotate, now})
  end

  defp generate_and_persist_new_salt(now) do
    salt = :crypto.strong_rand_bytes(16)
    Repo.insert_all("salts", [%{salt: salt, inserted_at: now}])
    salt
  end

  defp clean_old_salts(now) do
    h48_ago = DateTime.shift(now, hour: -48)
    Repo.delete_all(from s in "salts", where: s.inserted_at < ^h48_ago)
  end
end
```

**轮换策略**:

```
┌────────────────────────────────────────────────────────────────┐
│                        盐值生命周期                               │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  第 1 天 (Day 1)                                               │
│  ┌─────────────┐                                               │
│  │  Salt A     │  ← current (用于生成新 User ID)              │
│  │  (current)  │                                               │
│  └─────────────┘                                               │
│                                                                │
│  第 2 天 (Day 2) - 轮换后                                       │
│  ┌─────────────┐  ┌─────────────┐                             │
│  │  Salt B     │  │  Salt A     │                             │
│  │  (current)  │  │  (previous) │                             │
│  └─────────────┘  └─────────────┘                             │
│        ↑                  ↑                                    │
│   新会话使用           旧会话兼容                               │
│                                                                │
│  第 3 天 (Day 3) - 清理后                                       │
│  ┌─────────────┐  ┌─────────────┐                             │
│  │  Salt C     │  │  Salt B     │                             │
│  │  (current)  │  │  (previous) │                             │
│  └─────────────┘  └─────────────┘                             │
│                     ↑                                          │
│              Salt A 已被删除 (超过 48 小时)                    │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

**关键特性**:
1. **每日轮换**: 通过 Oban Worker `RotateSalts` 在 UTC 0 点执行（`config/runtime.exs:792`）
2. **双盐值保留**: 始终保留 `current` 和 `previous` 两个盐值
3. **48 小时清理**: 超过 48 小时的盐值从数据库删除
4. **强随机**: 使用 `:crypto.strong_rand_bytes(16)` 生成 128 位随机数
5. **高可用**: ETS 缓存 + 读并发优化

#### 2.2.3 盐值轮换对会话的影响

**位置**: `lib/plausible/session/cache_store.ex:24-27`

```elixir
found_session =
  find_session(event, event.user_id) || find_session(event, prev_user_id)
```

**会话查找顺序**:
1. 首先使用 `current` 盐值生成的 `user_id` 查找
2. 如果找不到，再使用 `previous` 盐值生成的 `prev_user_id` 查找

**效果**:
- 盐值轮换当天，旧会话仍可被识别（平滑过渡）
- 48 小时后，旧盐值被删除，无法再关联旧会话

### 2.3 无 Cookie 追踪架构

#### 2.3.1 传统 Cookie 追踪 vs Plausible

```
┌────────────────────────────────────────────────────────────────┐
│  传统 Google Analytics 方式                                      │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  第 1 次访问:                                                   │
│  ┌─────────────┐     Set Cookie: _ga=GA1.1.123456789.12345  │
│  │   Browser   │ ←──────────────────────────────────────────  │
│  └─────────────┘                                               │
│                                                                │
│  第 2 次访问 (数月后):                                          │
│  ┌─────────────┐     Cookie: _ga=GA1.1.123456789.12345     │
│  │   Browser   │ ──────────────────────────────────────────→ │
│  └─────────────┘                                               │
│                                                                │
│  问题: Cookie 可持久保存数年，实现跨天、跨月追踪                │
│                                                                │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│  Plausible 方式 (无 Cookie)                                      │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  第 1 次访问 (Day 1, Salt = A):                                │
│  User_ID = SipHash(A, UA + IP + Domain) = 0xAAA111           │
│                                                                │
│  第 2 次访问 (Day 2, Salt = B, IP/UA 相同):                   │
│  User_ID = SipHash(B, UA + IP + Domain) = 0xBBB222           │
│                    ↑                                            │
│              不同的 ID！无法关联为同一用户                       │
│                                                                │
│  优势: 无法跨天长期追踪同一用户                                  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 2.4 数据最小化实践

#### 2.4.1 请求验证和截断

**位置**: `lib/plausible/ingestion/request.ex`

```elixir
@max_url_size 2_000
@max_props 30

defp put_props(changeset, %{} = request_body) do
  props =
    (request_body["m"] || request_body["meta"] || ...)
    |> Plausible.Helpers.JSON.decode_or_fallback()
    |> Enum.reduce([], &filter_bad_props/2)
    |> Enum.take(@max_props)  # 最多 30 个属性
    |> Map.new()
  # ...
end

defp put_referrer(changeset, %{} = request_body) do
  referrer = request_body["r"] || request_body["referrer"]
  if is_binary(referrer) do
    referrer = String.slice(referrer, 0..(@max_url_size - 1))  # 截断
    Changeset.put_change(changeset, :referrer, referrer)
  # ...
end
```

**限制措施**:
| 限制项 | 最大值 | 说明 |
|--------|--------|------|
| URL 长度 | 2000 字符 | 防止超长 URL 注入 |
| 事件名长度 | 120 字符 | 限制事件名称大小 |
| 自定义属性 | 30 个 | 限制数据收集范围 |
| 属性键长度 | 配置项 | 防止超大键名 |
| 属性值长度 | 配置项 | 防止超大值 |

#### 2.4.2 敏感数据过滤

```elixir
defp filter_bad_props({k, v}, acc) do
  cond do
    Enum.any?([k, v], &(is_list(&1) or is_map(&1))) -> acc  # 拒绝嵌套结构
    Enum.any?([k, v], &(String.trim_leading(to_string(&1)) == "")) -> acc  # 拒绝空值
    true -> [{to_string(k), to_string(v)} | acc]
  end
end
```

### 2.5 地理位置处理

#### 2.5.1 匿名代理处理

**位置**: `lib/plausible/ingestion/event.ex:324-333`

```elixir
defp put_geolocation(%__MODULE__{} = event, _context) do
  case event.request.ip_classification do
    "anonymous_vpn_ip" ->
      update_session_attrs(event, %{country_code: "A1"})
    _any ->
      result = Plausible.Ingestion.Geolocation.lookup(event.request.remote_ip) || %{}
      update_session_attrs(event, result)
  end
end
```

**特殊处理**:
- `anonymous_vpn_ip` (匿名 VPN/代理) 的国家代码统一设为 `"A1"`
- 不进行精确地理位置解析
- 保护 VPN/Tor 用户的隐私

### 2.6 会话超时机制

**位置**: `lib/plausible/session/cache_store.ex:77-80`

```elixir
defp find_session(event, user_id) do
  from_cache = Plausible.Cache.Adapter.get(:sessions, {event.site_id, user_id})
  case from_cache do
    nil -> nil
    session ->
      if NaiveDateTime.diff(event.timestamp, session.timestamp, :minute) <= 30 do
        session
      end
  end
end
```

**超时规则**:
- 会话超时时间: **30 分钟**
- 从最后一个事件的时间戳开始计算
- 超时后创建新会话，不关联历史数据

---

## 3. 事件归因边界处理

### 3.1 归因处理流程

```
┌─────────────────────────────────────────────────────────────────┐
│                      事件归因处理流程                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1. 请求构建 (Request.build)                                      │
│     ├── 解析 URL、query params                                    │
│     ├── 提取 referrer                                             │
│     └── 验证必填字段                                               │
│                                                                   │
│  2. 来源解析 (put_source_info)                                    │
│     ├── 检查 UTM 参数 (utm_source, utm_medium 等)                │
│     ├── 解析 Referrer 头                                          │
│     └── 使用 RefInspector 分类来源                                 │
│                                                                   │
│  3. 媒介推断 (maybe_infer_medium)                                 │
│     ├── gclid → Google 付费搜索                                   │
│     ├── msclkid → Bing 付费搜索                                   │
│     └── 付费来源检测                                               │
│                                                                   │
│  4. 渠道分类 (Acquisition.get_channel)                            │
│     ├── 基于来源 + UTM 参数判断营销渠道                            │
│     └── 区分 Organic vs Paid                                      │
│                                                                   │
│  5. 会话归属 (Session.CacheStore)                                 │
│     ├── 30 分钟内 → 同一会话                                      │
│     ├── 新会话 → 记录首次来源                                      │
│     └── 已有会话 → 保持原始来源不变                                │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 来源解析逻辑

#### 3.2.1 来源优先级

**位置**: `lib/plausible/ingestion/source.ex:66-80`

```elixir
def resolve(request) do
  tagged_source =
    request.query_params["utm_source"] ||
      request.query_params["source"] ||
      request.query_params["ref"]

  source =
    cond do
      tagged_source -> tagged_source
      has_valid_referral?(request) -> parse(request.referrer)
      true -> nil
    end

  find_mapping(source)
end
```

**优先级顺序**:
```
1. utm_source  (最高优先级)
      ↓
2. source
      ↓
3. ref
      ↓
4. Referer HTTP Header
      ↓
5. nil (Direct)
```

#### 3.2.2 Referrer 验证规则

**位置**: `lib/plausible/ingestion/source.ex:110-124`

```elixir
defp has_valid_referral?(%Request{referrer: nil}), do: false

defp has_valid_referral?(%Request{referrer: referrer, uri: uri}) do
  referrer_uri = URI.parse(referrer)

  valid_scheme? = referrer_uri.scheme in ["http", "https", "android-app"]
  valid_host? = !is_nil(referrer_uri.host) && byte_size(referrer_uri.host) > 0

  internal? =
    Request.sanitize_hostname(referrer_uri.host) == Request.sanitize_hostname(uri.host)

  local? = referrer_uri.host == "localhost"

  valid_scheme? and valid_host? and not internal? and not local?
end
```

**有效 Referrer 条件**:
| 条件 | 说明 |
|------|------|
| `valid_scheme?` | 必须是 http、https 或 android-app |
| `valid_host?` | 必须有主机名 |
| `not internal?` | 不能是同一站点的内部跳转 |
| `not local?` | 不能是 localhost |

#### 3.2.3 来源规范化

**位置**: `lib/plausible/ingestion/source.ex:8-43`

```elixir
@external_resource "priv/custom_sources.json"
@custom_sources Application.app_dir(:plausible, "priv/custom_sources.json")
                |> File.read!()
                |> Jason.decode!()

@paid_sources Map.keys(@custom_sources)
              |> Enum.filter(&String.ends_with?(&1, ["ads", "ad"]))
              |> then(&["adwords" | &1])
              |> MapSet.new()
```

**规范化示例**:
- `google.com`、`google.co.uk`、`google.de` → 统一为 `"Google"`
- `ig` → `"Instagram"`
- `adwords` → `"Google"`
- `yt-ads` → 标记为付费来源

### 3.3 渠道分类逻辑

#### 3.3.1 分类引擎

**位置**: `lib/plausible/ingestion/acquisition.ex`

```elixir
def get_channel(source, utm_medium, utm_campaign, utm_source, click_id_param) do
  get_channel_lowered(
    String.downcase(source || ""),
    String.downcase(utm_medium || ""),
    String.downcase(utm_campaign || ""),
    String.downcase(utm_source || ""),
    click_id_param
  )
end

defp get_channel_lowered(source, utm_medium, utm_campaign, utm_source, click_id_param) do
  cond do
    cross_network?(utm_campaign) -> "Cross-network"
    paid_shopping?(source, utm_campaign, utm_medium) -> "Paid Shopping"
    paid_search?(source, utm_medium, utm_source, click_id_param) -> "Paid Search"
    paid_social?(source, utm_medium, utm_source) -> "Paid Social"
    paid_video?(source, utm_medium, utm_source) -> "Paid Video"
    display?(utm_medium) -> "Display"
    paid_other?(utm_medium) -> "Paid Other"
    organic_shopping?(source, utm_campaign) -> "Organic Shopping"
    organic_social?(source, utm_medium) -> "Organic Social"
    organic_video?(source, utm_medium) -> "Organic Video"
    search_source?(source) -> "Organic Search"
    email?(source, utm_source, utm_medium) -> "Email"
    affiliates?(utm_medium) -> "Affiliates"
    audio?(utm_medium) -> "Audio"
    sms?(utm_source, utm_medium) -> "SMS"
    mobile_push_notifications?(source, utm_medium) -> "Mobile Push Notifications"
    referral?(source, utm_medium) -> "Referral"
    true -> "Direct"
  end
end
```

#### 3.3.2 付费搜索检测

```elixir
defp paid_search?(source, utm_medium, utm_source, click_id_param) do
  (search_source?(source) and paid_medium?(utm_medium)) or
    (search_source?(source) and paid_source?(utm_source)) or
    (source == "google" and click_id_param == "gclid") or
    (source == "bing" and click_id_param == "msclkid")
end
```

**付费搜索判定条件**:
1. 是搜索引擎来源 + `utm_medium` 是付费模式
2. 是搜索引擎来源 + `utm_source` 是付费来源
3. Google 来源 + 有 `gclid` 参数
4. Bing 来源 + 有 `msclkid` 参数

#### 3.3.3 点击 ID 参数支持

**位置**: `lib/plausible/ingestion/event.ex:435-442`

```elixir
@click_id_params ["gclid", "gbraid", "wbraid", "msclkid", "fbclid", "twclid"]

defp get_click_id_param(query_params) do
  @click_id_params
  |> Enum.find(fn param_name -> Map.has_key?(query_params, param_name) end)
end
```

| 参数 | 平台 | 说明 |
|------|------|------|
| `gclid` | Google Ads | Google Click Identifier |
| `gbraid` | Google Ads | 应用到网页转化 |
| `wbraid` | Google Ads | 网页到应用转化 |
| `msclkid` | Microsoft Ads | Bing Click ID |
| `fbclid` | Facebook/Meta | Facebook Click ID |
| `twclid` | Twitter/X | Twitter Click ID |

#### 3.3.4 付费模式检测

```elixir
defp paid_medium?(utm_medium) do
  Regex.match?(~r/^(.*cp.*|ppc|retargeting|paid.*)$/, utm_medium)
end

defp paid_source?(utm_source) do
  Plausible.Ingestion.Source.paid_source?(utm_source)
end
```

**付费 medium 模式**:
- `cp.*` (cpc, cpm, cpp 等)
- `ppc` (按点击付费)
- `retargeting` (再营销)
- `paid.*` (paid, paid-search 等)

### 3.4 自定义来源分类

**位置**: `lib/plausible/ingestion/acquisition.ex:20-46`

```elixir
@custom_source_categories [
  {"hacker news", "SOURCE_CATEGORY_SOCIAL"},
  {"yahoo!", "SOURCE_CATEGORY_SEARCH"},
  {"gmail", "SOURCE_CATEGORY_EMAIL"},
  {"telegram", "SOURCE_CATEGORY_SOCIAL"},
  {"slack", "SOURCE_CATEGORY_SOCIAL"},
  {"producthunt", "SOURCE_CATEGORY_SOCIAL"},
  {"github", "SOURCE_CATEGORY_SOCIAL"},
  {"steamcommunity.com", "SOURCE_CATEGORY_SOCIAL"},
  {"statics.teams.cdn.office.net", "SOURCE_CATEGORY_SOCIAL"},
  {"vkontakte", "SOURCE_CATEGORY_SOCIAL"},
  {"threads", "SOURCE_CATEGORY_SOCIAL"},
  {"ecosia", "SOURCE_CATEGORY_SEARCH"},
  {"perplexity", "SOURCE_CATEGORY_SEARCH"},
  {"brave", "SOURCE_CATEGORY_SEARCH"},
  {"chatgpt.com", "SOURCE_CATEGORY_SEARCH"},  # AI 工具视为搜索引擎
  {"temu.com", "SOURCE_CATEGORY_SHOPPING"},
  {"discord", "SOURCE_CATEGORY_SOCIAL"},
  {"sogou", "SOURCE_CATEGORY_SEARCH"},
  {"microsoft teams", "SOURCE_CATEGORY_SOCIAL"}
]
```

**关键自定义规则**:
- **AI 工具 (ChatGPT, Perplexity)**: 归类为搜索引擎 (`SOURCE_CATEGORY_SEARCH`)
- **IM 工具 (Telegram, Discord, Slack, Teams)**: 归类为社交
- **邮件服务 (Gmail)**: 归类为邮件渠道
- **开发社区 (GitHub, Product Hunt, Hacker News)**: 归类为社交

### 3.5 会话归属逻辑

#### 3.5.1 会话创建与更新

**位置**: `lib/plausible/session/cache_store.ex:55-65`

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

#### 3.5.2 新会话 - 记录来源

**位置**: `lib/plausible/session/cache_store.ex:125-161`

```elixir
defp new_session_from_event(event, session_attributes) do
  %Plausible.ClickhouseSessionV2{
    # ... 基础字段
    referrer: Map.get(session_attributes, :referrer),
    click_id_param: Map.get(session_attributes, :click_id_param),
    referrer_source: Map.get(session_attributes, :referrer_source),
    utm_medium: Map.get(session_attributes, :utm_medium),
    utm_source: Map.get(session_attributes, :utm_source),
    utm_campaign: Map.get(session_attributes, :utm_campaign),
    utm_content: Map.get(session_attributes, :utm_content),
    utm_term: Map.get(session_attributes, :utm_term),
    # ... 其他字段
  }
end
```

**新会话记录的来源属性**:
- `referrer` - 原始推荐 URL
- `referrer_source` - 规范化的来源名称
- `click_id_param` - 广告点击 ID 类型
- 所有 UTM 参数 (`utm_source`, `utm_medium`, `utm_campaign`, `utm_content`, `utm_term`)
- 地理位置信息 (`country_code`, `city_geoname_id` 等)

#### 3.5.3 会话更新 - 保持原始来源

**位置**: `lib/plausible/session/cache_store.ex:95-123`

```elixir
defp update_session(session, event) do
  pageview? = event.name == "pageview"
  pageviews = if(pageview?, do: session.pageviews + 1, else: session.pageviews)

  %{
    session
    | timestamp: event.timestamp,
      entry_page:  # 仅在空时设置
        if(session.entry_page == "" and pageview?,
          do: event.pathname,
          else: session.entry_page
        ),
      hostname:  # 仅在空时设置
        if(pageview? and session.hostname == "",
          do: event.hostname,
          else: session.hostname
        ),
      exit_page: if(pageview?, do: event.pathname, else: session.exit_page),
      exit_page_hostname: if(pageview?, do: event.hostname, else: session.exit_page_hostname),
      is_bounce:
        if(session.is_bounce,
          do: not (pageviews >= 2 or (event.interactive? and not pageview?)),
          else: session.is_bounce
        ),
      duration: NaiveDateTime.diff(event.timestamp, session.start) |> abs,
      pageviews: pageviews,
      events: session.events + 1
  }
end
```

**关键发现**: 会话更新时**不会修改**任何来源相关字段！

| 字段 | 新会话时设置 | 更新时修改 |
|------|-------------|-----------|
| `referrer` | ✅ | ❌ |
| `referrer_source` | ✅ | ❌ |
| `utm_*` 参数 | ✅ | ❌ |
| `click_id_param` | ✅ | ❌ |
| `entry_page` | ✅ (仅空时) | ❌ |
| `exit_page` | ✅ | ✅ (每次 pageview) |
| `timestamp` | ✅ | ✅ |
| `duration` | 0 | ✅ (累加) |
| `pageviews` | 0/1 | ✅ (累加) |

**归因模型**: **首次接触归因 (First-Touch Attribution)**

- 会话内所有事件和转化都归属于**首次**带来用户的渠道
- 即使会话中间用户点击了其他来源的链接，只要在 30 分钟内，仍保持原始来源

#### 3.5.4 互动事件的特殊处理

**位置**: `lib/plausible/session/cache_store.ex:44-53`

```elixir
defp handle_event(%{name: "engagement"} = event, found_session, _, _) do
  if found_session do
    refresh_session_cache(found_session, event.timestamp)
    found_session
  else
    :no_session_for_engagement
  end
end
```

**engagement 事件规则**:
- 必须依附于已存在的会话
- 没有对应会话时会被丢弃 (`:no_session_for_engagement`)
- 仅用于刷新会话活跃时间，不创建新会话

### 3.6 归因边界总结

```
┌─────────────────────────────────────────────────────────────────┐
│                    事件归因边界规则                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1. 来源优先级                                                    │
│     utm_source > source > ref > Referer Header                  │
│                                                                   │
│  2. 会话边界                                                      │
│     ┌──────────────────────────────────────────────────────┐    │
│     │  30 分钟超时窗口                                        │    │
│     │  ┌──────┐    ┌──────┐    ┌──────┐                   │    │
│     │  │ Event│───▶│ Event│───▶│ Event│                   │    │
│     │  └──────┘    └──────┘    └──────┘                   │    │
│     │     ↑            ↑            ↑                        │    │
│     │  同一会话，同一来源归因                                │    │
│     └──────────────────────────────────────────────────────┘    │
│                                                                   │
│  3. 盐值轮换边界                                                  │
│     Day 1 (Salt A)          Day 2 (Salt B)                      │
│     ┌────────────┐          ┌────────────┐                      │
│     │ User_ID:   │   Salt   │ User_ID:   │                      │
│     │ 0xAAA111   │  轮换    │ 0xBBB222   │  ← 不同的 ID！       │
│     └────────────┘          └────────────┘                      │
│           ↑                         ↑                            │
│      同一会话结束              视为新用户/新会话                  │
│                                                                   │
│  4. 归因模型                                                      │
│     首次接触归因 (First-Touch)                                    │
│     ┌──────────────────────────────────────────────────────┐    │
│     │  Session Start                                          │    │
│     │  来源: Google / utm_medium=cpc                         │    │
│     │       ↓                                                 │    │
│     │  Event 1 (pageview) → 归属于 Google Paid              │    │
│     │       ↓                                                 │    │
│     │  Event 2 (点击外部链接，但 30 分钟内)                   │    │
│     │       ↓                                                 │    │
│     │  Event 3 (pageview) → 仍归属于 Google Paid            │    │
│     │                    (保持首次来源)                        │    │
│     └──────────────────────────────────────────────────────┘    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 附录：关键文件位置索引

| 功能模块 | 文件路径 | 关键行号 |
|----------|----------|----------|
| 事件处理管道 | `lib/plausible/ingestion/event.ex` | 130-151 |
| 机器人检测 (UA) | `lib/plausible/ingestion/event.ex` | 256-276 |
| User ID 生成 | `lib/plausible/ingestion/event.ex` | 553-579 |
| 盐值管理 | `lib/plausible/session/salts.ex` | 全文 |
| 会话缓存存储 | `lib/plausible/session/cache_store.ex` | 全文 |
| 来源解析 | `lib/plausible/ingestion/source.ex` | 全文 |
| 渠道归因 | `lib/plausible/ingestion/acquisition.ex` | 全文 |
| Shields 屏蔽 | `lib/plausible/shields.ex` | 全文 |
| 请求构建 | `lib/plausible/ingestion/request.ex` | 全文 |
| 机器人规则库 | `priv/ua_inspector/bot.bots.yml` | 全文 |
| 自定义来源 | `priv/custom_sources.json` | 全文 |
