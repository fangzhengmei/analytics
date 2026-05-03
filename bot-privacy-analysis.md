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
  maybe_infer_medium,           # 媒介推断 (可能修改 utm_medium)
  put_props, put_revenue,       # 扩展属性
  put_salts, put_user_id,       # 隐私保护 (关键)
  validate_clickhouse_event,    # 数据验证
  register_session              # 持久化入库
]
```

---

## 2. 请求信息边界与信任模型

### 2.1 核心发现：请求头信任层次

**关键洞察**：Plausible 的数据来源分为三类，信任级别截然不同：

| 数据类别 | 来源 | 信任级别 | 安全约束 |
|---------|------|---------|---------|
| **自定义代理头** | `x-plausible-ip`, `x-plausible-ip-type` | ⚠️ **必须依赖代理** | 需配置反向代理注入/过滤，禁止直接暴露应用 |
| **CDN/标准代理头** | `cf-connecting-ip`, `x-forwarded-for`, `forwarded` | ⚠️ **必须依赖代理** | 需配置可信代理链，取最左/最右取决于配置 |
| **Tracker 请求体** | `r`/`referrer`, `u`/`url`, `d`/`domain` 等 | ❌ **客户端可伪造** | JavaScript 发送，可被篡改 |
| **HTTP 标准头** | `User-Agent` | ❌ **客户端可伪造** | 浏览器可随意设置 |

### 2.2 关键代码证据：Remote IP 获取

**实现文件**：`lib/plausible_web/remote_ip.ex:6-36`

```elixir
def get(conn) do
  x_plausible_ip = List.first(Plug.Conn.get_req_header(conn, "x-plausible-ip")) || ""
  cf_connecting_ip = List.first(Plug.Conn.get_req_header(conn, "cf-connecting-ip")) || ""
  x_forwarded_for = List.first(Plug.Conn.get_req_header(conn, "x-forwarded-for")) || ""
  b_forwarded_for = List.first(Plug.Conn.get_req_header(conn, "b-forwarded-for")) || ""
  forwarded = List.first(Plug.Conn.get_req_header(conn, "forwarded")) || ""

  cond do
    byte_size(x_plausible_ip) > 0 ->
      clean_ip(x_plausible_ip)         # 优先级 1：自定义代理头

    byte_size(cf_connecting_ip) > 0 ->
      clean_ip(cf_connecting_ip)       # 优先级 2：Cloudflare

    byte_size(b_forwarded_for) > 0 ->
      parse_forwarded_for(b_forwarded_for)  # 优先级 3：BunnyCDN

    byte_size(x_forwarded_for) > 0 ->
      parse_forwarded_for(x_forwarded_for)  # 优先级 4：标准 X-Forwarded-For

    byte_size(forwarded) > 0 ->
      Regex.named_captures(~r/for=(?<for>[^;,]+).*$/, forwarded)
      |> Map.get("for")
      |> String.trim("\"")
      |> clean_ip()

    true ->
      to_string(:inet_parse.ntoa(conn.remote_ip))  # 回退：TCP 层 IP
  end
end

defp parse_forwarded_for(header) do
  String.split(header, ",")
  |> Enum.map(&String.trim/1)
  |> List.first()    # ⚠️ 取第一个 IP（最接近客户端）
  |> clean_ip()
end
```

### 2.3 关键代码证据：IP Classification 获取

**实现文件**：`lib/plausible/ingestion/request.ex:347-354`

```elixir
defp put_ip_classification(changeset, %Plug.Conn{} = conn) do
  value =
    conn
    |> Plug.Conn.get_req_header("x-plausible-ip-type")
    |> List.first()  # ⚠️ 直接取第一个头，无任何验证

  Changeset.put_change(changeset, :ip_classification, value)
end
```

**这意味着**：
- `x-plausible-ip-type` 头**直接决定** IP 分类
- 可能的值：`dc_ip` (丢弃), `threat_ip` (丢弃), `anonymous_vpn_ip` (标记 A1)
- **如果没有代理过滤，攻击者可以：**
  - 设置 `x-plausible-ip-type: dc_ip` 让真实用户流量被丢弃
  - 或者不设置该头，绕过 IP 信誉检查

### 2.4 关键代码证据：Referrer 来源

**重要发现**：`referrer` 不是从 HTTP `Referer` 头读取，而是从 **Tracker 请求体** 读取！

**实现文件**：`lib/plausible/ingestion/request.ex:166-175`

```elixir
defp put_referrer(changeset, %{} = request_body) do
  referrer = request_body["r"] || request_body["referrer"]  # ⚠️ 从请求体！

  if is_binary(referrer) do
    referrer = String.slice(referrer, 0..(@max_url_size - 1))
    Changeset.put_change(changeset, :referrer, referrer)
  else
    changeset
  end
end
```

**对比**：Tracker 脚本发送的数据（大致逻辑）：
```javascript
// plausible.js 发送的内容
{
  "n": "pageview",              // 事件名
  "u": "https://example.com/page",  // 当前页面 URL
  "d": "example.com",           // 域名
  "r": document.referrer,       // ⚠️ JS 读取的 referrer，可伪造！
  "w": screen.width,
  "h": screen.height
  // ...
}
```

### 2.5 安全部署要求总结

**必须配置的反向代理规则**（以 Nginx 为例）：

```nginx
# 关键：清除/覆盖客户端可能伪造的头
location /api/event {
    # 清除客户端可能伪造的 Plausible 自定义头
    proxy_set_header x-plausible-ip "";
    proxy_set_header x-plausible-ip-type "";

    # 注入真实客户端 IP（使用 ngx_http_realip_module）
    proxy_set_header x-plausible-ip $real_ip_override;

    # 可选：如果有 IP 信誉服务，注入分类
    # proxy_set_header x-plausible-ip-type $ip_classification;

    # 标准 X-Forwarded-For 配置
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    proxy_pass http://plausible_backend;
}
```

---

## 3. Bot 过滤机制

### 3.1 多层过滤策略

系统采用 **四层 Bot/垃圾流量过滤** 机制：

| 过滤层级 | 实现位置 | 检测方式 | 丢弃原因 | 依赖信任 |
|---------|---------|---------|---------|---------|
| L1: 验证代理 | `event.ex:196-210` | UA 匹配预设值 | `:verification_agent` | 低（UA 可伪造） |
| L2: IP 信誉 | `event.ex:212-230` | `x-plausible-ip-type` 头 | `:dc_ip`, `:threat_ip` | ⚠️ 高（依赖代理） |
| L3: 规则引擎 | `event.ex:232-360` | Shields 规则 | 各类 blocklist | 中（基于 IP/Hostname） |
| L4: UA 解析 | `event.ex:256-276` | UAInspector Bot 检测 | `:bot` | 低（UA 可伪造） |

### 3.2 核心 Bot 检测逻辑

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
      })

    _any -> event  # 解析失败不丢弃，保持事件
  end
end
```

### 3.3 数据中心 IP 过滤

**关键实现** (`event.ex:212-230`):

```elixir
defp drop_datacenter_ip(%__MODULE__{} = event, _context) do
  case event.request.ip_classification do
    "dc_ip" -> drop(event, :dc_ip)  # ⚠️ 无条件丢弃，完全依赖 x-plausible-ip-type 头
    _any -> event
  end
end

defp drop_threat_ip(%__MODULE__{} = event, _context) do
  case event.request.ip_classification do
    "threat_ip" -> drop(event, :threat_ip)
    _any -> event
  end
end
```

---

## 4. 隐私保护机制

### 4.1 核心设计原则

Plausible 采用 **"隐私-by-design"** 架构，核心保护机制包括：
1. 无 Cookie 追踪
2. 盐值轮换的用户哈希
3. 敏感 IP 模糊化
4. 定期数据清理

### 4.2 用户 ID 生成机制

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

**哈希输入因子与信任级别**：

| 因子 | 来源 | 可伪造性 | 隐私影响 |
|-----|------|---------|---------|
| `salt` | 数据库 + ETS 缓存 | 不可伪造 | 核心隐私保护 |
| `user_agent` | HTTP `User-Agent` 头 | 高 | 设备指纹组件 |
| `remote_ip` | 代理头（见 2.2 节） | 中（依赖代理） | 网络指纹组件 |
| `domain` | 请求体 `"d"` 字段 | 高 | 站点隔离 |
| `root_domain` | 从 hostname 解析 | 高 | 子域聚合 |

### 4.3 盐值轮换机制

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

### 4.4 双盐值设计与用户跨期识别

**关键代码** (`event.ex:414-433`):

```elixir
defp register_session(%__MODULE__{} = event, context) do
  # 使用 current 盐值生成的 user_id 已存在于 clickhouse_event_attrs
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

### 4.5 IP 分类与隐私处理

**处理流程** (`event.ex:324-333`, `event.ex:212-230`):

```
                  ┌──────────────────────┐
                  │  x-plausible-ip-type │
                  │  (必须由代理注入)     │
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

---

## 5. 访客归因机制与准确入库字段

### 5.1 核心代码证据

**归因数据的来源** (`event.ex:292-310`):

```elixir
defp put_source_info(%__MODULE__{} = event, _context) do
  query_params = event.request.query_params

  # utm_source: 原始值，直接从 query_params 取
  tagged_source =
    query_params["utm_source"] ||
      query_params["source"] ||
      query_params["ref"]

  update_session_attrs(event, %{
    # referrer_source: 标准化后的值，经过 Source.resolve() + find_mapping()
    referrer_source: Plausible.Ingestion.Source.resolve(event.request),

    # referrer: 格式化后的完整引用 URL
    referrer: Plausible.Ingestion.Source.format_referrer(event.request),

    click_id_param: get_click_id_param(event.request.query_params),

    # utm_* 系列：原始值，直接从 query_params 取
    utm_source: tagged_source,
    utm_medium: query_params["utm_medium"],
    utm_campaign: query_params["utm_campaign"],
    utm_content: query_params["utm_content"],
    utm_term: query_params["utm_term"]
  })
end
```

**Source.resolve() 的内部逻辑** (`source.ex:66-80`):

```elixir
def resolve(request) do
  tagged_source =
    request.query_params["utm_source"] ||
      request.query_params["source"] ||
      request.query_params["ref"]

  source =
    cond do
      tagged_source -> tagged_source                                    # 优先级 1: 有 UTM 标记
      has_valid_referral?(request) -> parse(request.referrer)          # 优先级 2: 有 referrer
      true -> nil                                                        # 直接访问
    end

  find_mapping(source)  # ⚠️ 关键：对结果进行标准化映射！
end

def find_mapping(nil), do: nil

def find_mapping(source) do
  case src(String.downcase(source)) do
    name when is_binary(name) -> name   # 有映射：返回标准化名称
    _ -> source                          # 无映射：返回原值
  end
end
```

**媒介推断** (`event.ex:312-322`):

```elixir
defp maybe_infer_medium(%__MODULE__{} = event, _context) do
  inferred_medium =
    case event.clickhouse_session_attrs do
      # 显式 utm_medium 优先
      %{utm_medium: medium} when is_binary(medium) -> medium

      # Google + gclid = 推断为 "(gclid)"
      %{utm_medium: nil, referrer_source: "Google", click_id_param: "gclid"} -> "(gclid)"

      # Bing + msclkid = 推断为 "(msclkid)"
      %{utm_medium: nil, referrer_source: "Bing", click_id_param: "msclkid"} -> "(msclkid)"

      _ -> nil
    end

  update_session_attrs(event, %{utm_medium: inferred_medium})  # ⚠️ 会修改 utm_medium！
end
```

### 5.2 入库字段精确定义

Plausible 使用 **两张核心表** 存储数据：`sessions_v2` 和 `events_v2`。两张表的归因字段定义相同。

#### 表结构定义

**Session 表** (`lib/plausible/clickhouse_session_v2.ex:35-78`):

```elixir
schema "sessions_v2" do
  # 标识字段
  field :user_id, Ch, type: "UInt64"      # 哈希后的用户 ID
  field :session_id, Ch, type: "UInt64"   # 会话 ID（随机生成）
  field :site_id, Ch, type: "UInt64"

  # ============================================
  # 归因字段 - 精确定义
  # ============================================

  # 【字段 1】完整引用 URL（格式化后）
  # 来源：Source.format_referrer(request.referrer)
  # 处理：去协议(https://)、去 www.、保留路径
  # 示例：
  #   输入: "https://www.google.co.uk/search?q=plausible"
  #   输出: "google.co.uk/search?q=plausible"
  # 无效引用: nil
  field :referrer, :string

  # 【字段 2】标准化来源名称
  # 来源：Source.resolve() 结果
  # 处理：
  #   - 优先用 utm_source/source/ref（如果有）
  #   - 否则用 request.referrer 解析
  #   - 最后经过 find_mapping() 标准化映射
  # 示例：
  #   utm_source="ig" → 映射 → "Instagram"
  #   utm_source="google" → 映射 → "Google"
  #   referrer="https://google.co.uk/..." → RefInspector → "Google"
  #   直接访问 → nil
  field :referrer_source, :string

  # 【字段 3】广告点击 ID 参数名
  # 来源：检测 query_params 中的 click_id 参数
  # 支持：gclid, gbraid, wbraid, msclkid, fbclid, twclid
  # 示例："gclid", "msclkid"
  # 注意：存储的是参数**名称**，不是值
  field :click_id_param, Ch, type: "LowCardinality(String)"

  # 【字段 4-8】UTM 参数（原始值）
  # 来源：直接从 query_params 取
  # 注意：这些字段**不经过**标准化映射
  # 注意：utm_medium 可能被 maybe_infer_medium() 修改

  # 来源：query_params["utm_source"] || query_params["source"] || query_params["ref"]
  # 示例："google", "ig", "newsletter"
  field :utm_source, :string

  # 来源：query_params["utm_medium"]，可能被推断修改
  # 示例："cpc", "organic", "email", "(gclid)"
  field :utm_medium, :string

  # 来源：query_params["utm_campaign"]
  field :utm_campaign, :string

  # 来源：query_params["utm_content"]
  field :utm_content, :string

  # 来源：query_params["utm_term"]
  field :utm_term, :string

  # 地理位置字段
  field :country_code, Ch, type: "LowCardinality(FixedString(2))"  # 如 "US", "A1"
  field :subdivision1_code, Ch, type: "LowCardinality(String)"
  field :subdivision2_code, Ch, type: "LowCardinality(String)"
  field :city_geoname_id, Ch, type: "UInt32"

  # 设备字段
  field :screen_size, Ch, type: "LowCardinality(String)"
  field :operating_system, Ch, type: "LowCardinality(String)"
  field :operating_system_version, Ch, type: "LowCardinality(String)"
  field :browser, Ch, type: "LowCardinality(String)"
  field :browser_version, Ch, type: "LowCardinality(String)"

  # 计算字段（只读）
  field :acquisition_channel, Ch, type: "LowCardinality(String)", writable: :never
end
```

**Event 表** (`lib/plausible/clickhouse_event_v2.ex:8-53`):

```elixir
schema "events_v2" do
  # 基础字段
  field :name, Ch, type: "LowCardinality(String)"
  field :site_id, Ch, type: "UInt64"
  field :hostname, :string
  field :pathname, :string
  field :user_id, Ch, type: "UInt64"
  field :session_id, Ch, type: "UInt64"
  field :timestamp, :naive_datetime

  # Session 属性（与 sessions_v2 相同的归因字段，冗余存储）
  field :referrer, :string
  field :referrer_source, :string
  field :click_id_param, Ch, type: "LowCardinality(String)"
  field :utm_medium, :string
  field :utm_source, :string
  field :utm_campaign, :string
  field :utm_content, :string
  field :utm_term, :string

  # 地理位置（与 sessions_v2 相同）
  field :country_code, Ch, type: "FixedString(2)"
  field :subdivision1_code, Ch, type: "LowCardinality(String)"
  field :subdivision2_code, Ch, type: "LowCardinality(String)"
  field :city_geoname_id, Ch, type: "UInt32"

  # 设备信息（与 sessions_v2 相同）
  field :screen_size, Ch, type: "LowCardinality(String)"
  field :operating_system, Ch, type: "LowCardinality(String)"
  field :operating_system_version, Ch, type: "LowCardinality(String)"
  field :browser, Ch, type: "LowCardinality(String)"
  field :browser_version, Ch, type: "LowCardinality(String)"
end
```

### 5.3 字段关系与数据流

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         归因字段完整数据流                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  输入数据（来自 Tracker 请求体）                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  request_body = {                                                     │  │
│  │    "u": "https://example.com/page?utm_source=ig&utm_medium=cpc",   │  │
│  │    "r": "https://www.google.co.uk/search?q=test",                  │  │
│  │    "d": "example.com"                                                │  │
│  │  }                                                                    │  │
│  └───────────────────────────────────┬─────────────────────────────────┘  │
│                                      │                                        │
│                                      ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  解析阶段（Request.build()）                                          │  │
│  │                                                                      │  │
│  │  request.uri = %URI{                                                 │  │
│  │    host: "example.com",                                              │  │
│  │    query: "utm_source=ig&utm_medium=cpc"                            │  │
│  │  }                                                                    │  │
│  │                                                                      │  │
│  │  request.query_params = %{                                           │  │
│  │    "utm_source" => "ig",                                             │  │
│  │    "utm_medium" => "cpc"                                             │  │
│  │  }                                                                    │  │
│  │                                                                      │  │
│  │  request.referrer = "https://www.google.co.uk/search?q=test"       │  │
│  │  request.hostname = "example.com"                                    │  │
│  └───────────────────────────────────┬─────────────────────────────────┘  │
│                                      │                                        │
│                                      ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  put_source_info 阶段（event.ex:292-310）                           │  │
│  │                                                                      │  │
│  │  ┌─────────────────────────────────────────────────────────────┐    │  │
│  │  │ 步骤 1: 提取 utm_source 原始值                                 │    │  │
│  │  │ tagged_source =                                               │    │  │
│  │  │   query_params["utm_source"] ||                               │    │  │
│  │  │   query_params["source"] ||                                   │    │  │
│  │  │   query_params["ref"]                                         │    │  │
│  │  │                                                                │    │  │
│  │  │ 本例：tagged_source = "ig"                                     │    │  │
│  │  └─────────────────────────────────────────────────────────────┘    │  │
│  │                                                                      │  │
│  │  ┌─────────────────────────────────────────────────────────────┐    │  │
│  │  │ 步骤 2: 计算 referrer_source（标准化）                        │    │  │
│  │  │ Source.resolve(request)                                      │    │  │
│  │  │   → tagged_source = "ig" (有值，优先)                        │    │  │
│  │  │   → source = "ig"                                             │    │  │
│  │  │   → find_mapping("ig")                                        │    │  │
│  │  │   → src("ig") → 查表 → "Instagram"                           │    │  │
│  │  │                                                                │    │  │
│  │  │ 本例：referrer_source = "Instagram"                           │    │  │
│  │  └─────────────────────────────────────────────────────────────┘    │  │
