# Plausible Analytics 核心机制深度分析报告（修正版）

## 目录
1. [机器人流量过滤机制](#1-机器人流量过滤机制)
2. [用户隐私保护机制](#2-用户隐私保护机制)
3. [事件归因边界处理](#3-事件归因边界处理)
4. [附录：关键代码位置索引](#4-附录关键代码位置索引)

---

## 1. 机器人流量过滤机制

### 1.1 多层过滤架构

Plausible Analytics 采用多层防御机制过滤机器人流量，在事件处理管道的不同阶段依次检查。

#### 过滤管道执行顺序

**位置**: `lib/plausible/ingestion/event.ex:130-151`

```elixir
defp pipeline() do
  [
    drop_verification_agent: &drop_verification_agent/2,
    drop_datacenter_ip: &drop_datacenter_ip/2,
    drop_threat_ip: &drop_threat_ip/2,
    drop_shield_rule_hostname: &drop_shield_rule_hostname/2,
    drop_shield_rule_page: &drop_shield_rule_page/2,
    drop_shield_rule_ip: &drop_shield_rule_ip/2,
    put_geolocation: &put_geolocation/2,
    drop_shield_rule_country: &drop_shield_rule_country/2,
    put_user_agent: &put_user_agent/2,
    put_basic_info: &put_basic_info/2,
    put_source_info: &put_source_info/2,
    maybe_infer_medium: &maybe_infer_medium/2,
    put_props: &put_props/2,
    put_revenue: &put_revenue/2,
    put_salts: &put_salts/2,
    put_user_id: &put_user_id/2,
    validate_clickhouse_event: &validate_clickhouse_event/2,
    register_session: &register_session/2
  ]
end
```

**管道特性**：
- 任一环节 `drop` 后，后续步骤不再执行
- 过滤顺序：IP 信誉检查 → 用户自定义规则 → 地理位置 → User-Agent 解析 → 来源归因

### 1.2 机器人识别判定规则详解

#### 1.2.1 User-Agent 检测（核心机制）

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

**判定规则**：

| 条件 | 处理方式 | 说明 |
|------|----------|------|
| `UAInspector.Result.Bot{}` | `drop(event, :bot)` | 任何被识别为 Bot 类型 |
| Client name == "Headless Chrome" | `drop(event, :bot)` | 无头浏览器 |
| 解析超时或失败 | 继续处理 | 不丢弃，避免误杀 |

#### 1.2.2 IP 分类过滤

**位置**: `lib/plausible/ingestion/event.ex:212-230`

```elixir
defp drop_datacenter_ip(%__MODULE__{} = event, _context) do
  case event.request.ip_classification do
    "dc_ip" -> drop(event, :dc_ip)
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

**IP 分类体系**：

| 分类值 | 来源 | 处理方式 | 说明 |
|--------|------|----------|------|
| `dc_ip` | `x-plausible-ip-type` | **丢弃** | 数据中心/云服务商 IP |
| `threat_ip` | `x-plausible-ip-type` | **丢弃** | 已知恶意 IP |
| `anonymous_vpn_ip` | `x-plausible-ip-type` | **保留** | 匿名 VPN/代理，国家设为 "A1" |

#### 1.2.3 Shields 用户自定义屏蔽规则

**位置**: `lib/plausible/shields.ex`

提供四种用户可配置的屏蔽规则：

| 规则类型 | 检查时机 | 限制数量 | 匹配模式 |
|----------|----------|----------|----------|
| IP 地址屏蔽 | `drop_shield_rule_ip` | 最多 30 条 | 精确匹配或 CIDR |
| 国家/地区屏蔽 | `drop_shield_rule_country` | 最多 30 条 | ISO 国家代码 |
| 页面路径屏蔽 | `drop_shield_rule_page` | 最多 30 条 | 通配符模式 |
| 主机名白名单 | `drop_shield_rule_hostname` | 最多 10 条 | 通配符模式（白名单） |

#### 1.2.4 垃圾推荐过滤

**位置**: `lib/plausible/ingestion/event.ex:56-60, 581-587`

- 执行时机：在管道**最早期**检查，甚至在 `GateKeeper.check` 之前
- 依赖库: `referrer_blocklist`

#### 1.2.5 丢弃原因完整枚举

**位置**: `lib/plausible/ingestion/event.ex:24-41`

```elixir
@type drop_reason() ::
        :bot                      # UA 检测为机器人
        | :spam_referrer          # 垃圾推荐域名
        | GateKeeper.policy()     # 站点访问控制
        | :invalid                # 数据格式无效
        | :dc_ip                  # 数据中心 IP
        | :threat_ip              # 威胁 IP
        | :site_ip_blocklist      # 用户配置 IP 屏蔽
        | :site_country_blocklist # 用户配置国家屏蔽
        | :site_page_blocklist    # 用户配置页面屏蔽
        | :site_hostname_allowlist # 主机名不匹配白名单
        | :verification_agent     # Plausible 安装验证代理
        | :lock_timeout           # 会话锁超时
        | :no_session_for_engagement # 互动事件无对应会话
        | :persist_timeout        # 持久化超时
        | :persist_error          # 持久化错误
        | :persist_decode_error   # 持久化解码错误
```

---

## 2. 用户隐私保护机制

### 2.1 核心设计原则

| 原则 | 实现方式 |
|------|----------|
| **无 Cookie 追踪** | 完全不使用持久化 Cookie |
| **单向哈希标识** | User ID 无法反向还原 |
| **定期盐值轮换** | 防止跨天长期追踪 |
| **数据最小化** | 仅收集必要的最小数据 |
| **会话超时** | 30 分钟无活动后结束 |

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

**输入因子**：`user_agent` + `remote_ip` + `domain` + `root_domain` + `salt`

### 2.3 盐值轮换机制深度解析

#### 2.3.1 双盐值查找机制（关键修正）

**之前的误解**：盐值轮换后 User ID 改变，必然创建新会话。

**实际行为**：系统会尝试用两个盐值生成的 ID 查找会话！

**位置**: `lib/plausible/session/cache_store.ex:24-27`

```elixir
found_session =
  find_session(event, event.user_id) ||      # 先用 current salt 的 ID 查找
  find_session(event, prev_user_id)           # 再用 previous salt 的 ID 查找
```

**测试验证** (`test/plausible_web/controllers/api/external_controller_test.exs:1281-1295`):

```elixir
test "salts rotating once does not", %{conn: conn, site: site} do
  # 第一次 pageview
  post(conn, "/api/event", %{n: "pageview", u: "https://test.com", d: site.domain})
  Plausible.Session.Salts.rotate()  # 盐值轮换！
  # 第二次 pageview（同一用户）
  post(conn, "/api/event", %{n: "pageview", u: "https://test.com", d: site.domain})

  # 关键断言：
  assert records |> Enum.map(& &1.user_id) |> Enum.uniq() |> Enum.count() == 1
  assert records |> Enum.map(& &1.session_id) |> Enum.uniq() |> Enum.count() == 1
end
```

**结论**：盐值轮换后，如果 30 分钟内有活动，会话会被复用！

#### 2.3.2 盐值轮换边界场景

| 场景 | 时间关系 | 行为 | 原因 |
|------|----------|------|------|
| 轮换后立即访问 | Salt A → Salt B，30 分钟内 | ✅ 复用同一会话 | `prev_user_id` 找到旧会话 |
| 轮换后超过 30 分钟 | Salt A → Salt B，超过 30 分钟 | ❌ 新建会话 | 旧会话已超时 |
| 48 小时后访问 | Salt A 已被删除 | ❌ 新建会话 | 旧盐值已清理 |

### 2.4 会话超时机制

**位置**: `lib/plausible/session/cache_store.ex:77-80`

- 超时时间：**30 分钟**
- 计时起点：`session.timestamp`（会话最后活跃时间）
- 每次活动更新 `timestamp`，重置超时计时

---

## 3. 事件归因边界处理

### 3.1 来源信息处理流程总览

```
输入: URL?utm_source=yt-ads&utm_medium=cpc + Referer: youtube.com

阶段 1: Request.build()
  ├── query_params: %{"utm_source" => "yt-ads", "utm_medium" => "cpc"}
  └── referrer: "https://www.youtube.com/..."

阶段 2: put_source_info()
  ├── utm_source = "yt-ads"        ← 原始值，用于付费检测
  ├── referrer_source = "Youtube"  ← 规范化后的值
  └── utm_medium = "cpc"

阶段 3: maybe_infer_medium()
  └── 仅当 utm_medium 为 nil 时才推断（此处已有值，跳过）

阶段 4: 渠道计算 (ClickHouse)
  acquisition_channel = get_channel(
    "Youtube",    # referrer_source (规范化)
    "cpc",        # utm_medium (原始)
    nil,          # utm_campaign
    "yt-ads",     # utm_source (原始，关键！)
    nil           # click_id_param
  )
  结果: "Paid Video"
  原因: utm_source="yt-ads" 在 @paid_sources 中
```

### 3.2 `utm_source` vs `referrer_source` 的关键区别

这是之前报告中的主要错误点，现在澄清：

| 字段 | 数据源 | 是否规范化 | 用途 |
|------|--------|------------|------|
| `utm_source` | `query_params["utm_source"]` 等 | **否** | 付费检测 (`paid_source?`) |
| `referrer_source` | `Source.resolve()` + `find_mapping()` | **是** | 显示名称、渠道分类 |

**实际示例**：

| URL 参数 | `utm_source` 值 | `referrer_source` 值 |
|----------|-----------------|----------------------|
| `?utm_source=ig` | `"ig"` | `"Instagram"` |
| `?utm_source=yt-ads` | `"yt-ads"` | `"Youtube"` |
| `?utm_source=fb-ads` | `"fb-ads"` | `"Facebook"` |

**为什么需要两个字段？**

```
场景: 用户点击 Youtube 广告，URL 带 utm_source=yt-ads

问题: 如何判断这是付费流量？

方案 A (只用 referrer_source="Youtube"):
  无法区分自然搜索的 Youtube 和广告点击的 Youtube
  因为 referrer_source 都是 "Youtube"！

方案 B (用原始 utm_source="yt-ads"):
  @paid_sources 包含 "yt-ads", "fb-ads" 等以 "ads"/"ad" 结尾的键
  paid_source?("yt-ads")  = true  ✓  判定为付费
  paid_source?("Youtube") = false ✗
```

### 3.3 渠道归属计算机制

#### 3.3.1 关键发现：`acquisition_channel` 是计算列

**位置**: `lib/plausible/clickhouse_session_v2.ex:77`

```elixir
field :acquisition_channel, Ch, type: "LowCardinality(String)", writable: :never
```

**重要属性**: `writable: :never`

这意味着：
1. **应用层不写入**这个字段
2. 它是 ClickHouse 的 **materialized column**（物化列）
3. 由 ClickHouse 内部的 SQL 函数动态计算

#### 3.3.2 付费来源检测逻辑

**位置**: `lib/plausible/ingestion/source.ex:13-16, 49-51`

```elixir
@paid_sources Map.keys(@custom_sources)
              |> Enum.filter(&String.ends_with?(&1, ["ads", "ad"]))
              |> then(&["adwords" | &1])
              |> MapSet.new()

def paid_source?(source) do
  MapSet.member?(@paid_sources, source)
end
```

**`@paid_sources` 集合构成**：
1. 从 `@custom_sources` 的 keys 中筛选以 `"ads"` 或 `"ad"` 结尾的键
2. 额外添加 `"adwords"`

**测试用例验证** (`test/plausible/ingestion/acquisition_test.exs`):

```elixir
# 测试: utm_source=yt-ads (原始值) 应判定为付费
%{
  referrer_source: "Youtube",   # 规范化后
  utm_source: "yt-ads",          # 原始值 ← 关键！
  expected: "Paid Video"
}

# 对比: utm_source=yt (原始值) 不应判定为付费
%{
  referrer_source: "Youtube",   # 相同
  utm_source: "yt",              # 不同 ← 不在 @paid_sources 中
  expected: "Organic Video"
}
```

#### 3.3.3 `maybe_infer_medium` 的真实作用

**位置**: `lib/plausible/ingestion/event.ex:312-322`

```elixir
defp maybe_infer_medium(%__MODULE__{} = event, _context) do
  inferred_medium =
    case event.clickhouse_session_attrs do
      %{utm_medium: medium} when is_binary(medium) -> medium  # 已有值，不覆盖
      %{utm_medium: nil, referrer_source: "Google", click_id_param: "gclid"} -> "(gclid)"
      %{utm_medium: nil, referrer_source: "Bing", click_id_param: "msclkid"} -> "(msclkid)"
      _ -> nil
    end
  update_session_attrs(event, %{utm_medium: inferred_medium})
end
```

**关键行为**：

| 条件 | 行为 |
|------|------|
| `utm_medium` 已有非空值 | **直接使用，不修改** |
| `utm_medium` 为 nil + Google + gclid | 设置为 `"(gclid)"` |
| `utm_medium` 为 nil + Bing + msclkid | 设置为 `"(msclkid)"` |
| 其他情况 | 保持 `nil` |

**重要说明**：
- 推断值 `"(gclid)"` **不匹配** `paid_medium?` 的正则表达式
- 但 `paid_search?` 会**直接检查** `source` 和 `click_id_param` 的组合：

```elixir
defp paid_search?(source, utm_medium, utm_source, click_id_param) do
  (search_source?(source) and paid_medium?(utm_medium)) or
    (search_source?(source) and paid_source?(utm_source)) or
    (source == "google" and click_id_param == "gclid") or  # ← 直接判断
    (source == "bing" and click_id_param == "msclkid")     # ← 直接判断
end
```

### 3.4 会话字段覆盖时机与逻辑

#### 3.4.1 新会话创建时的字段设置

**位置**: `lib/plausible/session/cache_store.ex:125-161`

新会话创建时，所有来源相关字段都从 `session_attributes` 读取：
- `referrer`, `referrer_source`, `utm_source`, `utm_medium`, `utm_campaign`, `utm_content`, `utm_term`, `click_id_param`

#### 3.4.2 会话更新时的字段行为

**位置**: `lib/plausible/session/cache_store.ex:95-123`

```elixir
defp update_session(session, event) do
  %{
    session
    | timestamp: event.timestamp,             # 更新
      entry_page: if(session.entry_page == "" and pageview?, ...),  # 仅空时
      hostname: if(pageview? and session.hostname == "", ...),      # 仅空时
      exit_page: if(pageview?, do: event.pathname, else: ...),     # 更新
      duration: NaiveDateTime.diff(...),                             # 更新
      pageviews: pageviews,                                          # 更新
      events: session.events + 1                                     # 更新
  }
end
```

**关键发现**：

`update_session` 函数：
1. **完全不使用**传入的 `session_attributes` 参数
2. **没有任何代码**更新来源相关字段

**会话更新时的字段变化汇总**：

| 字段 | 新会话时设置 | 更新时修改 | 说明 |
|------|-------------|-----------|------|
| `referrer` | ✅ | ❌ | **首次接触归因** |
| `referrer_source` | ✅ | ❌ | **首次接触归因** |
| `utm_source` | ✅ | ❌ | **首次接触归因** |
| `utm_medium` | ✅ | ❌ | **首次接触归因** |
| `utm_campaign` | ✅ | ❌ | **首次接触归因** |
| `click_id_param` | ✅ | ❌ | **首次接触归因** |
| `entry_page` | ✅ | 仅空时 | 首次 pageview 的页面 |
| `timestamp` | ✅ | ✅ | 每次更新 |
| `exit_page` | ✅ | ✅ | 每次 pageview 更新 |
| `duration` | ✅ | ✅ | 累加 |
| `pageviews` | ✅ | ✅ | 累加 |
| `events` | ✅ | ✅ | 累加 |

### 3.5 盐值轮换 + 会话窗口叠加边界

#### 3.5.1 边界场景分析

结合以下两个独立机制：
1. **盐值轮换**: 每日 UTC 0 点，生成新 salt，保留 previous salt 48 小时
2. **会话超时**: 30 分钟无活动后，会话无法再被找到

**叠加后的边界场景**：

```
场景 1: 盐值轮换后立即访问（30 分钟内）
─────────────────────────────────────────────────────────────
T=09:00 (Day 1, Salt=A):
  用户访问，创建会话 S1
  user_id_A = SipHash(Salt_A, UA + IP + ...) = UID_A1
  缓存中: {site_id, UID_A1} → S1

T=00:00 (Day 2, 盐值轮换):
  Salt_A → previous
  Salt_B → current
  S_A1 仍在缓存中 (超时时间 00:10 未到)

T=00:05 (Day 2, 轮换后 5 分钟):
  同一用户再次访问
  
  处理流程：
  1. 生成两个 user_id:
     - UID_B1 = SipHash(Salt_B, ...)
     - UID_A1 = SipHash(Salt_A, ...)
  
  2. 查找会话：
     find_session(UID_B1) → ❌ 未找到
     find_session(UID_A1) → ✅ 找到 S1 (仅 5 分钟，< 30 分钟)
  
  3. 更新会话 S1，来源字段保持不变

  结果: ✅ 复用同一会话，同一来源归因


场景 2: 盐值轮换后超过 30 分钟访问
─────────────────────────────────────────────────────────────
T=09:00 (Day 1, Salt=A): 创建会话 S1
T=00:00 (Day 2): 盐值轮换 (A→previous, B→current)
T=00:35 (Day 2): 用户访问（距离上次活动 15 小时 35 分钟）

  处理流程：
  1. UID_B = SipHash(Salt_B, ...)
  2. UID_A = SipHash(Salt_A, ...)
  3. 查找：
     find_session(UID_B) → ❌
     find_session(UID_A) → ❌ 会话已超时（15 小时 > 30 分钟）

  结果: ❌ 创建新会话 S2，使用当前事件的来源


场景 3: 48 小时后访问（旧 salt 已删除）
─────────────────────────────────────────────────────────────
T=09:00 (Day 1, Salt=A): 创建会话 S1
T=00:00 (Day 2): 盐值轮换
T=00:00 (Day 3): 清理 48 小时前的 salt → Salt_A 被删除
T=09:00 (Day 3): 用户访问

  处理流程：
  1. current salt = C, previous salt = B
  2. UID_C = SipHash(Salt_C, ...)
  3. UID_B = SipHash(Salt_B, ...)
  4. 两个 ID 都找不到会话（超时 + salt 不匹配）

  结果: ❌ 创建新会话，无法追溯 Salt_A 时期的活动
```

#### 3.5.2 边界决策表

| 场景 | 距离上次活动 | 盐值状态 | 行为 | 归因结果 |
|------|-------------|----------|------|----------|
| 正常活动 | < 30 分钟 | 无轮换 | 复用会话 | 保持首次来源 |
| 轮换后立即 | < 30 分钟 | current + previous 都有 | 复用会话 | 保持首次来源 |
| 轮换后超时 | > 30 分钟 | current + previous 都有 | 新建会话 | 使用**当前**事件来源 |
| 超过 48 小时 | 任意 | previous 已删除 | 新建会话 | 无法追溯旧来源 |

### 3.6 互动事件与跨日场景边界

#### 3.6.1 Engagement 事件的特殊处理

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

**关键特性**：

| 特性 | 说明 |
|------|------|
| **不能创建新会话** | `engagement` 事件**永远不会**触发新会话创建 |
| **必须依附现有会话** | 必须通过 `user_id` 或 `prev_user_id` 找到已有会话 |
| **仅刷新时间戳** | 找到会话后，仅更新 `timestamp`（重置 30 分钟超时） |
| **不修改其他字段** | 来源、pageviews、events 等都不变 |

#### 3.6.2 Engagement 事件边界场景

```
场景 1: 有活跃会话时发送 engagement
─────────────────────────────────────────────────────────────
T=0min: pageview → 创建会话 S1 (来源: Google Paid Search)
T=5min: engagement (滚动深度: 50%, 停留时间: 30s)

  处理：
  1. find_session(user_id) → ✅ 找到 S1
  2. refresh_session_cache(S1, new_timestamp)
  3. S1.timestamp 更新为 T=5min（超时重置为 T=35min）

  结果: ✅ 成功，会话保持活跃，来源归因不变


场景 2: 无会话时发送 engagement
─────────────────────────────────────────────────────────────
用户首次访问就发送 engagement 事件（单页应用特殊行为）

  处理：
  1. find_session(user_id) → ❌ 未找到
  2. find_session(prev_user_id) → ❌ 未找到
  3. 返回 :no_session_for_engagement

  结果: ❌ 事件被丢弃，drop_reason = :no_session_for_engagement


场景 3: 会话超时后发送 engagement
─────────────────────────────────────────────────────────────
T=0min: pageview → 创建会话 S1
T=40min: engagement (距离上次 40 分钟 > 30 分钟)

  处理：
  1. find_session(user_id) → ❌ 会话已超时
  2. 返回 :no_session_for_engagement

  结果: ❌ 事件被丢弃

  重要: 即使同一用户，超时后 engagement 也不会创建新会话
  需要先发送 pageview 或其他事件来创建新会话
```

#### 3.6.3 跨日场景完整示例

```
假设:
- 盐值在 UTC 00:00 轮换
- 用户 A: Chrome + IP=1.2.3.4
- 用户 B: Safari + IP=5.6.7.8 (不同用户)

═════════════════════════════════════════════════════════════

Day 1, 23:40 (Salt = A)
─────────────────────────────────────────────────────────────
用户 A 点击 Google 广告:
  URL: ?utm_source=google_ads&utm_medium=cpc

创建会话 S_A1:
  ├── user_id: SipHash(Salt_A, Chrome + 1.2.3.4 + ...) = UID_A1
  ├── referrer_source: "Google"
  ├── utm_source: "google_ads"
  ├── utm_medium: "cpc"
  ├── 渠道: "Paid Search"
  └── 超时: Day 2, 00:10 (30 分钟后)

═════════════════════════════════════════════════════════════

Day 2, 00:00 (盐值轮换)
─────────────────────────────────────────────────────────────
Salt_A → previous
Salt_B → current
S_A1 仍在缓存中 (超时时间 00:10 未到)

═════════════════════════════════════════════════════════════

Day 2, 00:05 (Salt B current, Salt A previous)
─────────────────────────────────────────────────────────────
用户 A 继续浏览 (同一浏览器，同一 IP)
点击 Facebook 链接，URL: ?utm_source=fb_ads

处理流程：
1. 生成两个 user_id:
   - UID_B1 = SipHash(Salt_B, Chrome + 1.2.3.4 + ...)
   - UID_A1 = SipHash(Salt_A, Chrome + 1.2.3.4 + ...)

2. 查找会话：
   find_session(UID_B1) → ❌ 未找到
   find_session(UID_A1) → ✅ 找到 S_A1 (仅 5 分钟，< 30 分钟)

3. 执行 update_session():
   - 使用找到的旧会话 S_A1
   - 传入新的 session_attributes (referrer_source="Facebook", utm_source="fb_ads")
   - ❌ update_session 完全忽略这些新的来源信息！

结果：
会话 S_A1 的来源字段**保持不变**：
  ├── referrer_source: "Google"    (仍是首次的)
  ├── utm_source: "google_ads"     (仍是首次的)
  ├── utm_medium: "cpc"             (仍是首次的)
  └── 渠道: "Paid Search"           (仍是首次的)

本次事件及后续事件都归属于 "Paid Search"

═════════════════════════════════════════════════════════════

Day 2, 00:15 (Salt B current, Salt A previous)
─────────────────────────────────────────────────────────────
用户 A 发送 engagement 事件（滚动深度 75%）

处理流程：
1. 生成两个 user_id: UID_B1, UID_A1
2. 查找会话：
   find_session(UID_B1) → ❌
   find_session(UID_A1) → ✅ 找到 S_A1 (更新后的超时是 00:35)
3. refresh_session_cache(S_A1, 00:15)
4. S_A1.timestamp = 00:15，超时重置为 00:45

结果: ✅ 成功，会话保持活跃，来源归因不变

═════════════════════════════════════════════════════════════

Day 2, 01:00 (超过 30 分钟无活动)
─────────────────────────────────────────────────────────────
用户 A 再次访问（同一浏览器，同一 IP）
点击 Twitter 链接，URL: ?utm_source=twitter-ads

处理流程：
1. 生成两个 user_id: UID_B1, UID_A1
2. 查找会话：
   find_session(UID_B1) → ❌ 未找到
   find_session(UID_A1) → ❌ 会话已超时 (00:45 < 01:00，超过 30 分钟)

3. 创建新会话 S_A2，使用当前事件的来源：
   ├── referrer_source: "Twitter"
   ├── utm_source: "twitter-ads"
   ├── utm_medium: nil
   └── 渠道: "Paid Social"

结果：
- 创建新会话 S_A2，归属于 "Paid Social"
- S_A1 已超时，无法再被更新
- 两个会话独立归因，互不影响

═════════════════════════════════════════════════════════════

Day 3, 09:00 (48 小时后，Salt_A 已被删除)
─────────────────────────────────────────────────────────────
用户 A 再次访问（同一浏览器，同一 IP）
直接访问，无来源参数

处理流程：
1. current salt = C, previous salt = B
2. 生成两个 user_id: UID_C, UID_B
3. 查找会话：
   find_session(UID_C) → ❌
   find_session(UID_B) → ❌

4. 创建新会话 S_A3，使用当前事件的来源：
   ├── referrer_source: nil
   └── 渠道: "Direct"

结果：
- 创建新会话 S_A3，归属于 "Direct"
- Salt_A 已被删除，无法追溯 Day 1 的活动
- 这是隐私设计的预期行为：无法跨天长期追踪同一用户
```

---

## 4. 附录：关键代码位置索引

| 功能模块 | 文件路径 | 关键行号 |
|----------|----------|----------|
| 事件处理管道 | `lib/plausible/ingestion/event.ex` | 130-151 |
| 机器人检测 (UA) | `lib/plausible/ingestion/event.ex` | 256-276 |
| 丢弃原因枚举 | `lib/plausible/ingestion/event.ex` | 24-41 |
| User ID 生成 | `lib/plausible/ingestion/event.ex` | 553-579 |
| 双盐值查找 | `lib/plausible/session/cache_store.ex` | 24-27 |
| 会话更新逻辑 | `lib/plausible/session/cache_store.ex` | 95-123 |
| Engagement 事件处理 | `lib/plausible/session/cache_store.ex` | 44-53 |
| 盐值管理 | `lib/plausible/session/salts.ex` | 全文 |
| 来源解析 | `lib/plausible/ingestion/source.ex` | 全文 |
| 付费来源检测 | `lib/plausible/ingestion/source.ex` | 13-16, 49-51 |
| 渠道归因 | `lib/plausible/ingestion/acquisition.ex` | 全文 |
| maybe_infer_medium | `lib/plausible/ingestion/event.ex` | 312-322 |
| acquisition_channel 定义 | `lib/plausible/clickhouse_session_v2.ex` | 77 |
| Shields 屏蔽 | `lib/plausible/shields.ex` | 全文 |
| 请求构建 | `lib/plausible/ingestion/request.ex` | 全文 |
| 机器人规则库 | `priv/ua_inspector/bot.bots.yml` | 全文 |
| 自定义来源 | `priv/custom_sources.json` | 全文 |
| 盐值轮换测试 | `test/plausible_web/controllers/api/external_controller_test.exs` | 1281-1295 |
| 渠道分类测试 | `test/plausible/ingestion/acquisition_test.exs` | 全文 |
