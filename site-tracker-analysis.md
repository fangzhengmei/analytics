# 站点 Tracker Script 配置与域名校验闭环分析

## 1. 概述

本文档详细分析了 Plausible 站点创建后，tracker script 配置与域名校验的完整闭环流程。**重点补充**：配置参数从安装到验证的传递链路、测试事件域名解析比对逻辑、关键状态节点、边界分支和设计取舍。

---

## 2. 配置参数传递链路详解

### 2.1 完整数据流图

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        配置参数传递链路 (Installation → Verification)                  │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ 1. 用户选择   │────▶│ 2. 表单提交   │────▶│ 3. 数据库存储 │────▶│ 4. URL跳转   │
│ 安装类型     │     │              │     │              │     │              │
│              │     │ - params收集  │     │ - site_id    │     │ - domain     │
│ - manual    │     │ - installation│     │ - installation│    │ - flow       │
│ - wordpress │     │   _type      │     │   _type      │     │ - installation│
│ - gtm (EE)  │     │ - 功能选项    │     │ - 各功能开关  │     │   _type      │
│ - npm       │     │              │     │              │     │ (关键参数)   │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
                                                          │
                                                          ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ 7. 诊断解析  │◀────│ 6. 检查执行   │◀────│ 5. 验证页面   │
│              │     │              │     │ 读取优先级    │
│ - selected_   │     │ - data_domain│    │ 1. URL参数    │
│   installation│    │ (站点域名)    │     │ 2. 数据库存储  │
│   _type      │     │ - selected_   │     │ 3. 默认值      │
│              │     │   installation  │    │              │
│ 用于生成    │     │   _type        │    │              │
│ 错误消息    │     │              │     │              │
└──────────────┘     └──────────────┘     └──────────────┘
```

### 2.2 安装阶段参数收集

**用户选择安装类型** (`installation.ex:126-147`):

```elixir
# 用户通过 Tab 切换选择安装类型，通过 URL 参数传递
<.tab
  patch={"?type=manual&flow=#{@flow}"}
  selected={@installation_type.result == "manual"}
>
  <Icons.script_icon /> Script
</.tab>
```

**表单提交处理** (`installation.ex:295-311`):

```elixir
def handle_event("submit", %{"tracker_script_configuration" => params}, socket) do
  # params 包含用户选择的配置：
  # - installation_type (隐藏字段，从当前选择的 tab 获取)
  # - outbound_links (checkbox)
  # - file_downloads (checkbox)
  # - form_submissions (checkbox)
  # - track_404_pages (checkbox)
  
  config =
    PlausibleWeb.Tracker.update_script_configuration!(
      socket.assigns.site,
      params,
      :installation  # 标识这是安装流程中的更新
    )

  # 跳转到验证页面，携带关键参数
  {:noreply,
   push_navigate(socket,
     to:
       Routes.site_path(socket, :verification, socket.assigns.site.domain,
         flow: socket.assigns.flow,
         installation_type: config.installation_type  # 关键：通过 URL 传递安装类型
       )
   )}
end
```

### 2.3 数据库存储结构

**TrackerScriptConfiguration Schema** (`tracker_script_configuration.ex:23-39`):

```elixir
schema "tracker_script_configuration" do
  # 主键：自动生成的 NanoID，用于脚本 URL (如 /js/abc123.js)
  @primary_key {:id, Plausible.Ecto.Types.TrackerScriptNanoid, autogenerate: true}
  
  # 安装类型：决定错误消息的展示内容
  field :installation_type, Ecto.Enum, values: [:manual, :wordpress, :gtm, :npm, nil]

  # 功能配置：影响生成的脚本内容
  field :track_404_pages, :boolean, default: false
  field :hash_based_routing, :boolean, default: false
  field :outbound_links, :boolean, default: false
  field :file_downloads, :boolean, default: false
  field :revenue_tracking, :boolean, default: false
  field :tagged_events, :boolean, default: false
  field :form_submissions, :boolean, default: false
  field :pageview_props, :boolean, default: false

  belongs_to :site, Plausible.Site  # 关联到站点
  timestamps()
end
```

**关键点澄清**：
- `installation_type` **不影响脚本生成**，只用于**错误消息的展示
- 真正影响脚本内容的是各个功能开关字段 (`outbound_links`, `file_downloads` 等)
- `id` (NanoID) 是脚本 URL 的唯一标识，用于 CDN 缓存和脚本查找

### 2.4 验证阶段参数读取

**安装类型读取优先级** (`verification.ex:183-194`):

```elixir
defp get_installation_type(params, site) do
  cond do
    # 优先级 1: URL 参数 (最高优先级，允许用户动态切换)
    params["installation_type"] in PlausibleWeb.Tracker.supported_installation_types() ->
      params["installation_type"]

    # 优先级 2: 数据库中保存的配置
    (saved_installation_type = get_saved_installation_type(site)) in @supported_installation_types_atoms ->
      Atom.to_string(saved_installation_type)

    # 优先级 3: 默认值 ("manual")
    true ->
      PlausibleWeb.Tracker.fallback_installation_type()
  end
end

defp get_saved_installation_type(site) do
  case PlausibleWeb.Tracker.get_tracker_script_configuration(site) do
    %{installation_type: installation_type} ->
      installation_type
    _ ->
      nil
  end
end
```

**设计取舍分析**：

| 设计决策 | 原因 | 影响 |
|---------|------|------|
| URL 参数优先级最高 | 允许用户在验证页面直接切换安装类型，无需回到安装页面重新配置 | 增加了灵活性，但也意味着 URL 可以"绕过"数据库配置 |
| 支持动态切换 | 用户可以尝试不同安装方式的验证 | 需要确保切换时不会破坏已有配置 |
| 默认值兜底 | 确保验证流程不会因为缺少配置而中断 | 可能隐藏了潜在的配置丢失问题 |

### 2.5 检查执行阶段参数传递

**State 结构** (`state.ex:9-15`):

```elixir
defstruct url: nil,                    # 要验证的 URL (可能是自定义 URL)
          data_domain: nil,            # ⭐ 关键：站点的域名 (用于域名比对的基准)
          report_to: nil,               # 汇报进程 PID
          assigns: %{},                 # 检查间共享数据
          diagnostics: %{},             # 诊断结果
          skip_further_checks?: false
```

**检查初始化** (`verification/checks.ex:27-47`):

```elixir
def run(url, data_domain, installation_type, opts \\ []) do
  init_state =
    %State{
      url: url,
      data_domain: data_domain,  # 站点域名 (从 site.domain 来)
      report_to: report_to,
      diagnostics: %Verification.Diagnostics{
        selected_installation_type: installation_type  # 安装类型 (用于错误消息)
      }
    }

  # 执行三个检查：
  checks = [
    {Checks.Url, []},                              # URL 可达性检查
    {Checks.VerifyInstallation, [timeout: ...]},   # 核心：浏览器验证
    {Checks.VerifyInstallationCacheBust, [...]}      # 缓存清除重试
  ]
end
```

**关键概念澄清**：

| 变量名 | 来源 | 用途 |
|-------|------|------|
| `url` | 用户输入或默认 (`https://#{domain}` | 浏览器要访问的地址 |
| `data_domain` | `site.domain` | ⭐ 域名比对的**期望域名** |
| `selected_installation_type` | URL 参数或数据库 | 错误消息的**展示方式** |

---

## 3. 测试事件域名解析比对详解

### 3.1 域名来源完整链路

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          域名来源链路 (Site → Script → Event → Verification)                │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌──────────────┐
│ 1. 站点创建   │
│  (Site.domain │
│              │
│  "example.com"│
└──────┬───────┘
       │
       ▼
┌──────────────┐     ┌──────────────────────────────────────────────────────────────┐
│ 2. 脚本生成   │────▶│  lib/plausible_web/tracker.ex:36-46                      │
│              │     │                                                              │
│ plausible_   │     │  def plausible_main_config(config) do                            │
│ main_config  │     │    %{                                                        │
│              │     │      domain: config.site.domain,  ◀─── 关键：嵌入域名   │
│  domain:     │     │      endpoint: tracker_ingestion_endpoint(),              │
│  "example.com"│     │      outboundLinks: config.outbound_links,             │
│              │     │      ...                                                    │
│              │     │    }                                                         │
└──────┬───────┘     └──────────────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────┐     ┌──────────────────────────────────────────────────────────────┐
│ 3. 脚本构建   │────▶│  lib/plausible_web/tracker.ex:49-67                      │
│              │     │                                                              │
│ 替换占位符   │     │  def build_script(config) do                                │
│              │     │    config_js_content =                                       │
│ <%= @config_ │     │      config                                                 │
│ js %>        │     │      |> plausible_main_config()                             │
│              │     │      |> ... (构建 JS 对象)                                 │
│ 最终脚本:    │     │                                                              │
│ {            │     │    @plausible_main_script                                   │
│   domain:    │     │    |> String.replace("\"<%= @config_js %>\"",             │
│   "example.com"│    │               "{#{config_js_content}}")                    │
│   endpoint:  │     │  end                                                         │
│   "...",     │     └──────────────────────────────────────────────────────────────┘
│   ...        │
│ }            │
└──────┬───────┘
       │
       ▼ 脚本加载到用户网站
       │
┌──────────────┐     ┌──────────────────────────────────────────────────────────────┐
│ 4. 脚本初始化 │────▶│  tracker/src/config.js:38-64                                 │
│              │     │                                                              │
│ 读取配置     │     │  export function init(options) {                             │
│              │     │    if (COMPILE_PLAUSIBLE_WEB) {                             │
│ config =     │     │      // 占位符被替换为实际配置对象                            │
│ {            │     │      config = '<%= @config_js %>'  ◀─── 注意：是字符串？    │
│   domain:    │     │      Object.assign(config, options, {                         │
│   "example.com"│    │        // ⭐ 关键：domain 不可覆盖！                         │
│ }            │     │        domain: config.domain  ◀── 强制使用嵌入的域名       │
│              │     │      })                                                       │
│              │     │    }                                                           │
│              │     │    // NPM 方式：需要用户传入 domain                           │
│              │     │    if (COMPILE_PLAUSIBLE_NPM) {                              │
│              │     │      if (!options || !options.domain) {                       │
│              │     │        throw new Error('domain argument is required')         │
│              │     │      }                                                          │
│              │     │      Object.assign(config, options)                            │
│              │     │    }                                                           │
│              │     │  }                                                             │
└──────┬───────┘     └──────────────────────────────────────────────────────────────┘
       │
       ▼ 发送事件
       │
┌──────────────┐     ┌──────────────────────────────────────────────────────────────┐
│ 5. 事件发送   │────▶│  事件 payload 中的 domain 字段                               │
│              │     │                                                              │
│ POST /api/   │     │  {                                                            │
│ event        │     │    "n": "pageview",      // 或 "verification-agent-test"  │
│              │     │    "d": "example.com",     // ◀── 从 config.domain 来        │
│ Body:        │     │    "u": "https://example.com/page",                         │
│ {            │     │    "v": 2,                                                 │
│   "n": "pageview",  │     │    "r": null,                                             │
│   "d": "example.com"│    │    ...                                                  │
│ }            │     │  }                                                            │
│              │     │                                                              │
│              │     │  ⭐ 测试事件也是同样的逻辑：                                     │
│              │     │  window.plausible('verification-agent-test', { callback: ... })│
│              │     │  发送的事件中 d 字段 = config.domain                           │
└──────┬───────┘     └──────────────────────────────────────────────────────────────┘
       │
       ▼ 验证服务拦截事件
       │
┌──────────────┐
│ 6. 域名比对   │
│              │
│ 期望域名:    │
│ data_domain  │
│ (site.domain)│
│              │
│ 实际域名:    │
│ event.d        │
│ (从事件payload│
│ 中提取)      │
│              │
│ 比对:        │
│ event.d ==  │
│ data_domain? │
└──────────────┘
```

### 3.2 关键设计：Domain 不可覆盖性

**Web 脚本的安全设计** (`tracker/src/config.js:38-45`):

```javascript
export function init(options) {
  if (COMPILE_PLAUSIBLE_WEB) {
    // 这行代码在服务端被替换为实际的配置对象
    // 例如：config = '{domain:"example.com",endpoint:"...",outboundLinks:!0}'
    
    config = '<%= @config_js %>'
    
    Object.assign(config, options, {
      // ⭐ 关键设计：domain 显式放在最后，强制使用嵌入的域名
      // 即使用户通过 transformRequest 或其他方式尝试覆盖，也无效
      domain: config.domain
    })
  }
}
```

**设计意图分析**：

| 设计决策 | 原因 | 安全影响 |
|---------|------|---------|
| Web 脚本 domain 不可覆盖 | 防止用户意外或恶意修改目标站点 | 确保事件始终发送到正确的站点 |
| NPM 脚本需要显式传入 domain | NPM 用于 SPA/自定义场景，需要灵活性 | 用户负责确保 domain 正确性 |
| 旧版脚本从 data-domain 读取 | 兼容 v1 脚本 | 依赖 script 标签属性 |

**重要澄清**：
- **Web 脚本**：domain 是**硬编码**在脚本中的，无法通过前端配置修改
- 这意味着：如果用户复制了脚本 A（对应站点 A 的脚本），即使修改了页面上的 domain 属性或其他配置，事件仍然会发送到站点 A
- 这是**安全设计**，防止配置错误时会导致"域名不匹配"错误，而不是静默发送到错误站点

### 3.3 测试事件发送与拦截

**测试事件发送** (`tracker/installation_support/verifier.js:208-308`):

```javascript
async function testPlausibleFunction({ timeoutMs, debug }) {
  return new Promise((_resolve) => {
    // ... 轮询检测 window.plausible 是否存在和初始化 ...
    
    // 当 plausible 就绪后，发送测试事件
    testEventPollInterval = setInterval(() => {
      if (plausibleIsOnWindow && plausibleIsInitialized) {
        // 发送名为 'verification-agent-test' 的测试事件
        window.plausible('verification-agent-test', {
          callback: (testEventCallbackResult) => {
            resolve({
              testEvent: {
                callbackResult: testEventCallbackResult ?? 'undefined or null'
              }
            })
          }
        })
        clearInterval(testEventPollInterval)
      }
    }, 10)
  })
}
```

**Fetch 拦截机制** (`tracker/installation_support/verifier.js:144-186`):

```javascript
function startRecordingEventFetchCalls() {
  const interceptions = new Map()
  const originalFetch = window.fetch
  
  window.fetch = function (url, options = {}) {
    let identifier = null
    
    // 规范化事件体，提取关键信息
    const normalizedEventBody = getNormalizedPlausibleEventBody(options)
    
    if (normalizedEventBody) {
      identifier = normalizedEventBody.name  // 事件名称作为标识
      interceptions.set(identifier, {
        request: { 
          url, 
          normalizedBody: normalizedEventBody  // 包含 domain 字段
        }
      })
    }

    return originalFetch
      .apply(this, arguments)
      .then(async (response) => {
        const eventRequest = interceptions.get(identifier)
        if (eventRequest) {
          const responseClone = response.clone()
          const body = await responseClone.text()
          eventRequest.response = { status: response.status, body }
        }
        return response
      })
  }
  
  return {
    getInterceptedFetch: (identifier) => interceptions.get(identifier),
    stopRecording: () => { window.fetch = originalFetch }
  }
}
```

**事件体规范化** (`tracker/installation_support/verifier.js:121-142`):

```javascript
function getNormalizedPlausibleEventBody(fetchOptions) {
  try {
    const body = JSON.parse(fetchOptions.body ?? '{}')

    let name = null
    let domain = null
    let version = null

    // 支持新旧两种字段格式
    if (
      fetchOptions.method === 'POST' &&
      (typeof body?.n === 'string' || typeof body?.name === 'string') &&
      (typeof body?.d === 'string' || typeof body?.domain === 'string')
    ) {
      name = body?.n || body?.name        // 新: n, 旧: name
      domain = body?.d || body?.domain    // 新: d, 旧: domain
      version = body?.v || body?.version  // 新: v, 旧: version
    }
    return name && domain ? { name, domain, version } : null
  } catch (_error) {
    // 解析失败则忽略
  }
}
```

### 3.4 域名比对逻辑详解

**比对入口** (`verification/checks.ex:53-99`):

```elixir
def interpret_diagnostics(
      %State{
        diagnostics: diagnostics,
        data_domain: data_domain,  # ⭐ 期望域名
        url: url
      },
      opts \\ []
    ) do
  result =
    Verification.Diagnostics.interpret(
      diagnostics,
      data_domain,  # 传递给 interpret 函数
      url
    )
  # ...
end
```

**核心比对函数** (`verification/diagnostics.ex:87-122`):

```elixir
# 成功场景 1: 普通成功
def interpret(
      %__MODULE__{
        test_event: %{
          "normalizedBody" => %{
            "domain" => domain  # 实际域名（从事件中提取
          },
          "responseStatus" => response_status
        },
        service_error: nil
      },
      expected_domain,  # 期望域名（data_domain = site.domain
      _url
    )
    when response_status in [200, 202] and
           domain == expected_domain,  # ⭐ 核心比对：完全相等
    do: success()

# 成功场景 2: 仅在清除缓存后成功（视为缓存问题）
def interpret(
      %__MODULE__{
        test_event: %{
          "normalizedBody" => %{
            "domain" => domain
          },
          "responseStatus" => response_status
        },
        service_error: nil,
        diagnostics_are_from_cache_bust: true  # 标记：这是缓存清除后的重试
      },
      expected_domain,
      _url
    )
    when response_status in [200, 202] and
           domain == expected_domain,
    do: handled_error(@error_succeeds_only_after_cache_bust)

# 失败场景：域名不匹配
def interpret(
      %__MODULE__{
        test_event: %{
          "normalizedBody" => %{
            "domain" => domain
          },
          "responseStatus" => response_status
        },
        service_error: nil,
        selected_installation_type: selected_installation_type
      },
      expected_domain,
      _url
    )
    when response_status in [200, 202] and
           domain != expected_domain do  # ⭐ 域名不匹配
  error_unexpected_domain(selected_installation_type)
  |> handled_error()
end
```

### 3.5 域名比对边界分支详解

**完整比对决策树**：

```
                    ┌─────────────────────────────────────────────────────────────────┐
                    │              测试事件域名比对决策树                                │
                    └─────────────────────────────────────────────────────────┘
                                              │
                                              ▼
                              ┌───────────────────────────────┐
                              │ 测试事件是否成功发送？          │
                              │ (response_status ∈ [200,  │
                              │  202]?)                  │
                              └───────────────────────────────┘
                                   │               │
                              是 │               │ 否
                                   ▼               ▼
                    ┌───────────────┐    ┌──────────────────────────────────┐
                    │ 检查:        │    │ 检查: 网络/代理错误分支          │
                    │ domain ==  │    │                             │
                    │ expected_ │    │ - 检查 requestUrl 是否以     │
                    │ domain?   │    │   Plausible endpoint 开头  │
                    └───────────────┘    │                             │
                          │             │ 是代理 │ 否                  │
                     是 │ │ 否              ▼             ▼                     │
                          │    ┌──────────────┐  ┌──────────────┐         │
                          ▼    │ 代理网络错误 │  │ Plausible    │         │
                    ┌──────────┐  │ 提示检查代理 │  │ 网络错误    │         │
                    │ 成功   │  │ 配置       │  │ 提示稍后重试│         │
                    │分支  │  │  └──────────────┘  └──────────────┘         │
                    └──────┘  └──────────┘                             │
                       │           │                                      │
                       ▼           ▼                                      │
           ┌──────────────┐  ┌──────────────┐                              │
           │ 普通成功   │  │ 域名不匹配   │                              │
           │ (无缓存   │  │ 错误分支     │                              │
           │  标记)     │  │              │                              │
           │           │  │ 根据安装类型 │                              │
           │ 返回      │  │ 显示不同错误   │                              │
           │ success()  │  │ 消息         │                              │
           └──────────────┘  └──────────────┘                              │
                                                                          │
                                                                          │
                    ┌─────────────────────────────────────────────────────────┐
                    │ 其他失败场景（无有效测试事件）：                      │
                    │                                                   │
                    │ 1. tracker_is_in_html: false                   │
                    │    → "We couldn't detect Plausible..."     │
                    │                                                   │
                    │ 2. plausible_is_on_window: false                 │
                    │    → 同上（根据安装类型）                       │
                    │                                                   │
                    │ 3. CSP 阻止: disallowed_by_csp: true            │
                    │    → "We encountered an issue with CSP"      │
                    │                                                   │
                    │ 4. 服务错误: service_error 存在                  │
                    │    - :domain_not_found → 站点不可达             │
                    │    - :browserless_timeout → 服务超时            │
                    │    - :browserless_client_error → 网络错误          │
                    └─────────────────────────────────────────────────┘
```

### 3.6 域名不匹配错误的详细分析

**错误生成逻辑** (`verification/diagnostics.ex:324-359`):

```elixir
@unexpected_domain_message "Plausible test event is not for this site"

# 不同安装类型的错误消息：

# Manual 安装
@error_unexpected_domain_for_manual Error.new!(%{
  message: @unexpected_domain_message,
  recommendation:
    "Please check that the snippet on your site matches the installation instructions exactly",
  url: @verify_manually_url
})

# NPM 安装
@error_unexpected_domain_for_npm Error.new!(%{
  message: @unexpected_domain_message,
  recommendation:
    "Please check that you've initialized Plausible with the correct domain",
  url: @verify_manually_url
})

# GTM 安装
@error_unexpected_domain_for_gtm Error.new!(%{
  message: @unexpected_domain_message,
  recommendation:
    "Please check that you've entered the ID in the GTM template correctly",
  url: @verify_manually_url
})

# WordPress 安装
@error_unexpected_domain_for_wordpress Error.new!(%{
  message: @unexpected_domain_message,
  recommendation:
    "Please check that you've installed the WordPress plugin correctly",
  url: @verify_manually_url
})
```

**常见原因分析**：

| 安装类型 | 可能原因 | 排查方向 |
|---------|---------|--------|
| Manual | 复制了错误的脚本（其他站点的脚本） | 检查 script URL 中的 script ID |
| NPM | init() 时传入了错误的 domain | 检查初始化代码 |
| GTM | 模板中输入了错误的 Script ID | 检查 GTM 配置 |
| WordPress | 插件配置了错误的域名 | 检查插件设置 |

**重要澄清**：
- "域名不匹配"错误**不代表**脚本本身工作正常
- 它表示：脚本发送的事件中的 domain 与当前验证的站点不匹配
- 这通常是**配置错误**，不是代码错误

---

## 4. 关键状态节点与边界分支

### 4.1 安装流程状态机

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          安装流程状态机 (Installation Flow)                      │
└─────────────────────────────────────────────────────────────────────────────────────┘

                              ┌──────────────────┐
                              │   页面加载      │
                              │  mount/1          │
                              └────────┬─────────┘
                                       │
                                       ▼
                    ┌──────────────────────────────────────┐
                    │ 初始化数据:                     │
                    │                                 │
                    │ 1. 检测推荐安装类型 (EE)      │
                    │    - 扫描站点技术栈            │
                    │    - 检测 v1 旧脚本           │
                    │                                 │
                    │ 2. 获取或创建配置             │
                    │    - 默认启用所有自动捕获功能    │
                    │    - installation_type = 推荐类型    │
                    │                                 │
                    │ 3. 确定当前选中类型             │
                    │    - 优先级: URL参数 > 保存值 > 默认│
                    └────────────────┬─────────────┘
                                     │
                                     ▼
                    ┌──────────────────────────────────────┐
                    │     用户交互阶段                 │
                    │                                 │
                    │ 可能的操作：                    │
                    │                                 │
                    │ 1. 切换 Tab (修改 URL ?type=xx) │
                    │    → 更新 installation_type          │
                    │                                 │
                    │ 2. 勾选/取消勾选功能选项       │
                    │    → 更新表单数据                │
                    │                                 │
                    │ 3. 点击 "Verify X installation" │
                    │    → 提交表单                   │
                    └────────────────┬─────────────┘
                                     │
                                     ▼
                    ┌──────────────────────────────────────┐
                    │     表单提交处理                 │
                    │                                 │
                    │ 1. update_script_configuration!│
                    │    - 保存到数据库              │
                    │    - 同步 Goals (自动捕获功能)    │
                    │    - 清除 CDN 缓存 (如果需要)    │
                    │                                 │
                    │ 2. push_navigate 到验证页面   │
                    │    - URL: /:domain/verification│
                    │    - 参数: flow, installation_type│
                    └────────────────┬─────────────┘
                                     │
                                     ▼
                              ┌──────────────────┐
                              │   进入验证流程  │
                              └──────────────────┘
```

### 4.2 验证流程状态机

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          验证流程状态机 (Verification Flow)                      │
└─────────────────────────────────────────────────────────────────────────────────────┘

                              ┌──────────────────┐
                              │   页面加载      │
                              │  mount/1        │
                              └────────┬─────────┘
                                       │
                                       ▼
                    ┌──────────────────────────────────────┐
                    │ 初始化:                           │
                    │                                 │
                    │ 1. 获取 site                   │
                    │ 2. 检查是否有 pageviews?            │
                    │ 3. 确定 installation_type:       │
                    │    - URL参数 > 数据库 > 默认   │
                    │ 4. custom_url_input?           │
                    │    - 是: 显示自定义 URL 表单   │
                    │    - 否: 自动启动验证           │
                    └────────────────┬─────────────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              │                     │                      │
              ▼                     ▼                      ▼
    ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
    │ custom_url      │   │ 自动启动验证    │   │ CE 特殊处理     │
    │ 输入表单       │   │                 │   │                 │
    │                 │   │ launch_delayed │   │ 无 Browserless │
    │ 用户输入自定义   │   │                 │   │ 支持            │
    │ URL            │   │ 发送 {:start,    │   │                 │
    │                 │   │ self()}      │   │ 直接显示        │
    │ 点击 "Verify   │   │                 │   │ "Awaiting your │
    │ Installation" │   │ 速率限制检查:   │   │ first pageview" │
    │                 │   │ 60分钟内最多3次 │   │                 │
    └────────┬────────┘   └────────┬────────┘   └─────────────────┘
             │                         │
             ▼                         ▼
             │              ┌─────────────────┐
             │              │ 检查执行中...   │
             │              │                 │
             │              │ Checks.run()   │
             │              │                 │
             │              │ 1. Url 检查   │
             │              │ 2. Verify    │
             │              │    Installation│
             │              │ 3. Cache Bust  │
             │              │    重试        │
             │              └────────┬────────┘
             │                       │
             └───────────────────────┘
                                     │
                                     ▼
                    ┌──────────────────────────────────────┐
                    │     结果处理                       │
                    │                                 │
                    │ 收到 {:all_checks_done, state}    │
                    │                                 │
                    │ 1. interpret_diagnostics(state)   │
                    │    → 计算 success?            │
                    │                                 │
                    │ 2. 无 pageviews?                 │
                    │    → schedule_pageviews_check      │
                    │                                 │
                    │ 3. 更新组件状态:                 │
                    │    - finished?: true            │
                    │    - success?: interpretation.ok?│
                    │    - interpretation: 结果       │
                    └────────────────┬─────────────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              │                     │                      │
              ▼                     ▼                      ▼
    ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
    │ 成功            │   │ 失败            │   │ 等待 pageview    │
    │                 │   │                 │   │                 │
    │ 显示:          │   │ 显示:          │   │ 无 pageviews?   │
    │ "Success!"     │   │ 错误标题       │   │                 │
    │                 │   │ 修复建议       │   │ 轮询检查:      │
    │ 按钮:          │   │                 │   │ check_pageviews │
    │ "Go to the     │   │ 按钮:          │   │                 │
    │ dashboard"     │   │ "Verify again"  │   │ 有 pageview?     │
    │                 │   │                 │   │                 │
    │ 无 pageviews?   │   │ 链接:          │   │ redirect_to     │
    │ → "Awaiting    │   │ - 自定义 URL    │   │ stats           │
    │   your first   │   │ - 联系支持(EE)  │   │                 │
    │   pageview..." │   │ - 回到安装说明  │   │                 │
    │                 │   │ - 跳过验证      │   │                 │
    │ 轮询 pageviews │   │                 │   │                 │
    └─────────────────┘   └─────────────────┘   └─────────────────┘
```

### 4.3 关键边界分支详解

---

## 🔴 附录 A: 最容易误判的边界口径详解

### A.1 域名变更过渡期：数据层宽容 vs 验证层严格

#### 问题背景

用户可能会产生困惑：

> "我刚刚把域名从 `old.com` 改成了 `new.com`，系统说 72 小时过渡期内新旧域名都能接收数据。但为什么验证时还会因为 '域名不匹配' 失败？"

#### 核心矛盾：两层逻辑不一致

| 层级 | 逻辑 | 是否考虑 `domain_changed_from` |
|-----|------|-------------------------------|
| **数据接收层** | 宽容：新旧域名都能接收 | ✅ 考虑 |
| **验证层** | 严格：只匹配当前 `domain` | ❌ 不考虑 |

#### 代码证据

**数据接收层（宽容）** (`lib/plausible/site/cache.ex:95-103`):

```elixir
@impl true
def unwrap_cache_keys(items) do
  Enum.reduce(items, [], fn
    {domain, nil, object}, acc ->
      [{domain, object} | acc]  # 只有新域名

    {domain, domain_changed_from, object}, acc ->
      # ⭐ 关键：新旧域名都映射到同一个站点
      [{domain, object}, {domain_changed_from, object} | acc]
  end)
end
```

**验证层（严格）** (`extra/lib/plausible/installation_support/verification/diagnostics.ex:87-122`):

```elixir
# 成功场景
def interpret(
      %__MODULE__{
        test_event: %{
          "normalizedBody" => %{
            "domain" => domain  # 实际域名（从事件中提取）
          },
          "responseStatus" => response_status
        },
        service_error: nil
      },
      expected_domain,  # ⭐ 期望域名 = site.domain（新域名）
      _url
    )
  when response_status in [200, 202] and
         domain == expected_domain,  # ⭐ 严格相等！
  do: success()

# 域名不匹配场景
def interpret(
      %__MODULE__{
        test_event: %{
          "normalizedBody" => %{
            "domain" => domain  # 假设是 old.com（旧域名）
          },
          "responseStatus" => response_status
        },
        ...
      },
      expected_domain,  # 假设是 new.com（新域名）
      _url
    )
  when response_status in [200, 202] and
         domain != expected_domain,  # old.com != new.com ❌
  do:
    error_unexpected_domain(selected_installation_type)
    |> handled_error()
```

#### 完整状态机：域名变更过渡期

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    域名变更过渡期的双重标准问题                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘

                           域名变更前
                    site.domain = "old.com"
                    domain_changed_from = nil

                              │
                              ▼ 执行域名变更
                              │   site.domain = "new.com"
                              │   domain_changed_from = "old.com"
                              │   domain_changed_at = <now>
                              │
            ┌─────────────────┴─────────────────┐
            │                                     │
            ▼                                     ▼
┌───────────────────────┐           ┌───────────────────────┐
│    数据接收层         │           │     验证层            │
│   (事件 Ingestion)   │           │ (Verification Check)  │
└───────────────────────┘           └───────────────────────┘
            │                                     │
            ▼                                     ▼
┌───────────────────────┐           ┌───────────────────────┐
│  查找站点逻辑:         │           │  域名比对逻辑:         │
│                       │           │                       │
│  site_cache 中查找:   │           │  严格相等检查:         │
│  - "new.com" ✅ 找到   │           │  event.domain         │
│  - "old.com" ✅ 找到   │           │  ==                   │
│    (通过 domain_changed│           │  site.domain          │
│     _from 映射)        │           │                       │
│                       │           │  实际:                │
│  ⭐ 新旧域名事件都    │           │  "old.com"            │
│    能被正确接收        │           │  ==                   │
│                       │           │  "new.com"            │
│                       │           │  ⭐ 不相等！          │
│                       │           │                       │
│                       │           │  结果: "域名不匹配"    │
│                       │           │       错误 ❌          │
└───────────────────────┘           └───────────────────────┘
```

#### 设计取舍分析

| 维度 | 决策 | 原因 | 影响 |
|-----|------|------|------|
| **数据层宽容** | 支持 `domain_changed_from` | 平滑迁移，不丢失数据 | 用户体验好 |
| **验证层严格** | 只比较 `domain == expected_domain` | 强制用户完成迁移 | 过渡期内可能误判 |

**为什么验证层不做同样的宽容？**

从代码看，验证层的 `interpret/3` 函数**没有接收 `site` 对象**，只接收：
- `diagnostics`（诊断数据）
- `expected_domain`（字符串，来自 `site.domain`）
- `url`（访问 URL）

**设计取舍**：
1. **简单性 > 过渡友好**：验证逻辑只需要一个字符串 `expected_domain`，不需要完整的 Site 对象
2. **强制完成迁移**：过渡期是"宽限期"，不是"永久支持"，验证严格性可以推动用户尽快更新脚本
3. **安全性**：验证的目的是确认"当前脚本配置正确"，而不是"历史上曾经正确"

#### 用户视角的正确理解

**用户应该知道**：

| 场景 | 数据能否接收 | 验证能否通过 |
|-----|-----------|------------|
| 脚本发送 `new.com`（新域名） | ✅ 能 | ✅ 能通过 |
| 脚本发送 `old.com`（旧域名） | ✅ 能（过渡期内） | ❌ 会失败（显示"域名不匹配"） |

**过渡期结束后（72小时）**：
- `domain_changed_from` 被清空
- 旧域名事件**无法再接收**
- 验证仍然严格比对

#### 容易混淆的口径纠正

| 错误表述 | 正确表述 |
|---------|---------|
| "过渡期内新旧域名**都支持**" | "过渡期内新旧域名**在数据层**都能接收，但**验证层**只认可当前域名" |
| "验证失败说明脚本有问题" | "验证失败可能只是脚本还在发送旧域名，数据可能仍在正常接收" |
| "数据能接收说明验证应该通过" | "数据接收和验证是两套独立逻辑，有不同的目标" |

---

### A.2 缓存清理重试：首次结果 vs 缓存重试的诊断分流

#### 问题背景

用户可能会困惑：

> "为什么第一次验证失败，第二次（清缓存后）成功时，系统不显示 'Success!'，而是显示缓存问题警告？"

#### 核心逻辑：三轮检查 + 分流决策

**检查执行顺序** (`extra/lib/plausible/installation_support/verification/checks.ex:37-47`):

```elixir
checks = [
  {Checks.Url, []},                              # 第 1 轮：URL 可达性
  {Checks.VerifyInstallation, [...]},            # 第 2 轮：核心浏览器验证
  {Checks.VerifyInstallationCacheBust, [...]}    # 第 3 轮：缓存重试
]
```

**缓存重试触发条件** (`extra/lib/plausible/installation_support/checks/verify_installation_cache_bust.ex:22-41`):

```elixir
@impl true
def perform(%State{url: url} = state, _opts) do
  case InstallationSupport.Verification.Checks.interpret_diagnostics(state, telemetry?: false) do
    # 情况 1：已经成功 → 不执行缓存重试
    %InstallationSupport.Result{ok?: true} ->
      state

    # 情况 2：未处理错误（如服务内部错误）→ 不执行缓存重试
    %InstallationSupport.Result{data: %{unhandled: true}} ->
      state

    # 情况 3：已知安装失败 → 执行缓存重试
    _known_installation_failure ->
      reset_diagnostics = %InstallationSupport.Verification.Diagnostics{
        selected_installation_type: state.diagnostics.selected_installation_type
      }

      state
      |> struct!(diagnostics: reset_diagnostics)  # ⭐ 重置诊断（清除前一轮结果）
      |> struct!(url: InstallationSupport.URL.bust_url(url))  # URL 加随机参数
      |> InstallationSupport.Checks.VerifyInstallation.perform([])  # 重新验证
      |> put_diagnostics(diagnostics_are_from_cache_bust: true)  # ⭐ 打标记
  end
end
```

#### 诊断分流决策树

**模式匹配优先级** (`extra/lib/plausible/installation_support/verification/diagnostics.ex:68-122`):

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    诊断分流决策树（按优先级）                                        │
└─────────────────────────────────────────────────────────────────────────────────────┘

                       收到诊断数据
                             │
                             ▼
              ┌──────────────────────────────┐
              │ 检查: diagnostics_are_from_  │
              │        cache_bust == true?   │
              │  (是否是缓存重试的结果?)      │
              └──────────────────────────────┘
                       │               │
                  是 │               │ 否
                       ▼               ▼
          ┌───────────────┐   ┌──────────────────────────┐
          │ 缓存重试后    │   │ 检查: domain 匹配?       │
          │ 成功分支      │   │ response_status ∈ [200, │
          │               │   │                  202]?   │
          │ 即使成功也    │   └──────────────────────────┘
          │ 视为"缓存问题"│              │               │
          │               │          是 │               │ 否
          │ 返回:        │              ▼               ▼
          │ "We detected │    ┌───────────────┐   ┌───────────────┐
          │  an issue    │    │  普通成功     │   │ 域名不匹配    │
          │  with your   │    │               │   │ 或其他错误    │
          │  site's      │    │ 返回:        │   │               │
          │  cache"      │    │ "Success!"   │   │ 按错误类型    │
          │               │    │               │   │ 返回对应错误  │
          │ 用户提示:     │    │ ⭐ 这是真正  │   │               │
          │ "请清除缓存"  │    │   的"成功"   │   │               │
          │               │    └───────────────┘   └───────────────┘
          │ ⭐ 这是"警告 │
          │   级成功"     │
          │   不是真正成功 │
          └───────────────┘
```

#### 代码证据：分流逻辑

**缓存重试后成功 = 缓存问题错误** (`diagnostics.ex:68-85`):

```elixir
# 优先级 1：缓存重试后成功（最优先匹配）
def interpret(
      %__MODULE__{
        test_event: %{
          "normalizedBody" => %{"domain" => domain},
          "responseStatus" => response_status
        },
        service_error: nil,
        diagnostics_are_from_cache_bust: true  # ⭐ 关键标记
      },
      expected_domain,
      _url
    )
    when response_status in [200, 202] and
           domain == expected_domain,
    do: handled_error(@error_succeeds_only_after_cache_bust)  # ❌ 返回错误！
```

**普通成功 = 真正成功** (`diagnostics.ex:87-102`):

```elixir
# 优先级 2：普通成功（无缓存标记）
def interpret(
      %__MODULE__{
        test_event: %{
          "normalizedBody" => %{"domain" => domain},
          "responseStatus" => response_status
        },
        service_error: nil
        # ⭐ 没有 diagnostics_are_from_cache_bust 标记
      },
      expected_domain,
      _url
    )
    when response_status in [200, 202] and
           domain == expected_domain,
    do: success()  # ✅ 真正的成功
```

#### 完整状态流转图

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    缓存重试诊断分流完整状态机                                        │
└─────────────────────────────────────────────────────────────────────────────────────┘

                        ┌──────────────────┐
                        │   初始化检查     │
                        │                  │
                        │ State.url =      │
                        │   "https://site" │
                        │ State.diagnostics │
                        │   = 空           │
                        └────────┬─────────┘
                                 │
                                 ▼
                        ┌──────────────────┐
                        │ 第 1 轮检查      │
                        │                  │
                        │ Checks.Verify    │
                        │   Installation   │
                        │                  │
                        │ URL: 无随机参数  │
                        │ 标记: 无         │
                        └────────┬─────────┘
                                 │
                                 ▼
              ┌──────────────────┼──────────────────┐
              │                  │                  │
              ▼                  ▼                  ▼
    ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
    │   成功        │   │  未处理错误   │   │  已知失败      │
    │               │   │               │   │               │
    │ ok?: true    │   │ unhandled:    │   │ 其他错误类型   │
    │               │   │ true          │   │               │
    └───────┬───────┘   └───────┬───────┘   └───────┬───────┘
            │                   │                   │
            ▼                   ▼                   ▼
    ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
    │ 不执行缓存    │   │ 不执行缓存    │   │ 执行缓存重试  │
    │ 重试          │   │ 重试          │   │               │
    │               │   │               │   │ 重置诊断      │
    │ 直接返回      │   │ 直接返回      │   │ URL 加参数   │
    │ "Success!"   │   │ 原始错误      │   │ 打标记:       │
    │               │   │               │   │ cache_bust:   │
    │ ⭐ 真正成功   │   │               │   │ true          │
    └───────────────┘   └───────────────┘   └───────┬───────┘
                                                        │
                                                        ▼
                                               ┌──────────────────┐
                                               │ 第 2 轮检查      │
                                               │                  │
                                               │ Checks.Verify    │
                                               │   Installation   │
                                               │                  │
                                               │ URL: 有随机参数  │
                                               │ 标记: cache_bust │
                                               └────────┬─────────┘
                                                        │
                                    ┌───────────────────┼───────────────────┐
                                    │                   │                   │
                                    ▼                   ▼                   ▼
                           ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
                           │   成功        │   │  失败         │   │ 其他结果      │
                           │               │   │               │   │               │
                           │ 但有标记:     │   │ 用第二轮结果  │   │ 用第二轮结果  │
                           │ cache_bust    │   │               │   │               │
                           └───────┬───────┘   └───────┬───────┘   └───────┬───────┘
                                   │                   │                   │
                                   ▼                   ▼                   ▼
                           ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
                           │ 返回:         │   │ 返回:         │   │ 返回:         │
                           │ 缓存问题错误  │   │ 第二轮的错误  │   │ 第二轮结果    │
                           │               │   │               │   │               │
                           │ "We detected  │   │               │   │               │
                           │  an issue     │   │               │   │               │
                           │  with your    │   │               │   │               │
                           │  site's cache"│   │               │   │               │
                           │               │   │               │   │               │
                           │ ⭐ 不是真正   │   │               │   │               │
                           │   成功        │   │               │   │               │
                           └───────────────┘   └───────────────┘   └───────────────┘
```

#### 设计取舍分析

| 设计决策 | 原因 | 用户体验影响 |
|---------|------|------------|
| **缓存重试后成功 = 错误** | 避免报告"成功"但实际是缓存问题 | 用户看到警告，会去清除缓存 |
| **第二轮诊断完全替换第一轮** | 简化逻辑，只看最终结果 | 第一轮的具体错误信息丢失 |
| **只有"已知失败"才触发缓存重试** | 不浪费资源重试服务错误 | 服务错误不会被缓存重试掩盖 |

**为什么缓存重试后成功不视为"成功"？**

从模块文档可以看到设计意图 (`verify_installation_cache_bust.ex:1-12`):

```elixir
@moduledoc """
If the output of previous checks can not be interpreted as successful,
as a last resort, we try to bust the cache of the site under test...

The idea is to make sure that any issues we detect will be about the latest version of their website.

We also want to avoid reporting a successful installation if it took a special cache-busting action to make it work.
"""
```

**核心原则**：
> "我们不希望报告'成功安装'，如果需要特殊的缓存破坏操作才能使其工作。"

这意味着：
- **真正的成功**：第一次检查就成功（说明真实用户访问也会成功）
- **缓存后成功**：需要清除缓存才能工作（真实用户可能还在访问缓存的旧版本）

#### 测试用例验证

从测试用例可以验证这一逻辑 (`checks_test.exs:380-436`):

```elixir
describe "VerifyInstallation & VerifyInstallationCacheBust" do
  test "returns error when it 'succeeds', but only after cache bust" do
    # 模拟：第一次失败，第二次（缓存后）成功
    # ...
    
    assert_matches %Result{
                     ok?: false,  # ⭐ 注意：ok? 是 false！
                     errors: [^any(:string, ~r/.*cache.*/)],  # 提到缓存
                     recommendations: [
                       %{
                         text: ^any(:string, ~r/.*cache.*/),
                         url: "https://plausible.io/docs/troubleshoot-integration#have-you-cleared-the-cache-of-your-site"
                       }
                     ]
                   } = run_checks(verification_stub) |> Checks.interpret_diagnostics()
  end
end
```

#### 容易混淆的口径纠正

| 错误表述 | 正确表述 |
|---------|---------|
| "验证成功了，但显示缓存问题" | "验证没有**真正成功**，只是在清除缓存后才通过，这被视为'缓存问题错误'" |
| "ok?: true 就是成功" | "ok?: true 只表示**普通成功**，缓存后成功的 ok? 是 false" |
| "第二轮结果会合并第一轮" | "第二轮诊断**完全替换**第一轮，第一轮的具体错误信息会丢失" |
| "任何失败都会触发缓存重试" | "只有**已知安装失败**才会重试，服务内部错误（unhandled）不会重试" |

---

## 附录 B: 关键状态节点速查

### B.1 域名变更过渡期关键状态

| 状态变量 | 来源 | 含义 |
|---------|------|------|
| `site.domain` | 用户修改 | 当前生效的新域名 |
| `site.domain_changed_from` | 变更时设置 | 变更前的旧域名 |
| `site.domain_changed_at` | 变更时设置 | 变更时间（用于计算 72 小时） |
| `expected_domain` (验证) | `site.domain` | 验证时的期望域名（只有新域名） |

**判定顺序**：
1. 数据层：`domain == site.domain OR domain == site.domain_changed_from`
2. 验证层：`domain == site.domain`（**严格相等**）

### B.2 缓存重试关键状态

| 状态变量 | 来源 | 含义 |
|---------|------|------|
| `diagnostics_are_from_cache_bust` | 缓存重试时设置 | 标记当前诊断是否来自缓存重试 |
| `State.diagnostics` | 每次检查后更新 | 当前诊断数据（缓存重试会重置） |
| `State.url` | 缓存重试时修改 | 访问 URL（缓存重试会加随机参数） |

**判定顺序**（模式匹配优先级）：
1. **缓存重试后成功** → 缓存问题错误（`handled_error`）
2. **普通成功** → 真正成功（`success`）
3. **域名不匹配** → 对应安装类型的错误
4. **其他失败** → 按错误类型处理

---

### 4.3 关键边界分支详解（续）

#### 分支 1: 自定义 URL 输入

**触发条件**：
- URL 参数 `?custom_url=true`
- 或验证失败且 `offer_custom_url_input: true`

**场景**：
- 站点部署在子路径 (`https://example.com/blog`)
- 站点使用非标准端口
- 站点需要特定路径才能触发脚本加载

**代码** (`verification.ex:97-105`):

```elixir
def handle_event("verify-custom-url", %{"custom_url" => custom_url}, socket) do
  socket =
    socket
    |> assign(url_to_verify: custom_url)  # 自定义 URL
    |> assign(custom_url_input?: false)

  launch_delayed(socket)  # 使用自定义 URL 启动验证
  {:noreply, reset_component(socket)}
end
```

**重要澄清**：
- `url_to_verify` 是**浏览器要访问的地址
- `data_domain` 仍然是**站点域名**（用于比对）
- 这允许：验证 `https://example.com/blog` 上的脚本发送事件到 `example.com`

#### 分支 2: 缓存问题检测

**触发条件**：
- 第一次验证失败
- 第二次（清除缓存后验证成功

**错误类型**：
```elixir
# verification/diagnostics.ex:68-85
def interpret(
      %__MODULE__{
        # ... 成功条件 ...
        diagnostics_are_from_cache_bust: true  # 标记：这是缓存清除后的重试
      },
      expected_domain,
      _url
    )
    when response_status in [200, 202] and
           domain == expected_domain,
    do: handled_error(@error_succeeds_only_after_cache_bust)
```

**用户反馈**：
- 不是完全成功（显示警告）
- 提示用户清除站点缓存
- 提供缓存清除文档链接

#### 分支 3: 版本差异（CE vs EE）

| 特性 | Community Edition | Enterprise Edition |
|-----|-----------------|-------------------|
| Browserless 验证 | ❌ 不支持 | ✅ 支持 |
| 安装类型检测 | ❌ 不支持 | ✅ 支持 |
| GTM 安装类型 | ❌ 不支持 | ✅ 支持 |
| v1 脚本检测 | ❌ 不支持 | ✅ 支持 |
| 验证流程 | 直接等待 pageview | 完整浏览器验证 |

**CE 验证逻辑** (`verification_test.exs:32-39`):

```elixir
@tag :ce_build_only
test "static verification screen renders (ce)", %{conn: conn, site: site} do
  resp =
    get(conn, conn |> no_slowdown() |> get("/#{site.domain}") |> redirected_to)
    |> html_response(200)

  assert resp =~ "Awaiting your first pageview …"
end
```

#### 分支 4: 速率限制

**限制规则** (`verification.ex:114-121`):

```elixir
case Plausible.RateLimit.check_rate(
       "site_verification:#{domain}",
       :timer.minutes(60),  # 时间窗口：60 分钟
       3                      # 最大次数：3 次
     ) do
  {:allow, _} -> :ok
  {:deny, _} -> :timer.sleep(@slowdown_for_frequent_checking)  # 延迟 5 秒
end
```

**设计意图**：
- 防止 Browserless 服务被滥用
- 防止对用户站点被频繁请求
- 超过限制后不是完全拒绝，而是添加延迟

#### 分支 5: 未处理情况 (Unhandled Cases)

**触发条件**：
- 诊断结果无法匹配任何已知错误模式
- Browserless 服务内部错误

**处理逻辑** (`verification/checks.ex:70-96`):

```elixir
case {telemetry?, result.data} do
  {_, %{unhandled: true, browserless_issue: browserless_issue}} ->
    sentry_msg =
      if browserless_issue,
        do: "Browserless failure in verification",
        else: "Unhandled case for site verification"

    Sentry.capture_message(sentry_msg,
      extra: %{
        message: inspect(diagnostics),
        url: url,
        hash: :erlang.phash2(diagnostics)
      }
    )

    Logger.warning(
      "[VERIFICATION] Unhandled case (data_domain='#{data_domain}'): #{inspect(diagnostics)}"
    )

    :telemetry.execute(telemetry_event_unhandled(), %{})
end
```

**用户反馈**：
- 显示通用错误消息
- 提示"稍后重试"或"手动验证"
- 内部记录到 Sentry 和日志

---

## 5. 设计取舍与容易混淆的口径

### 5.1 关键设计取舍

| 设计决策 | 取舍 | 影响 |
|---------|------|------|
| **Web 脚本 domain 硬编码** | 安全性 > 灵活性 | 配置错误会导致验证失败，但不会发送到错误站点 |
| **URL 参数优先级 > 数据库** | 用户体验 > 一致性 | 用户可以动态切换安装类型，但可能与数据库不一致 |
| **验证失败不阻止使用** | 用户体验 > 正确性 | 用户可以跳过验证去查看仪表板 |
| **测试事件特殊 UA** | 数据准确性 > 实现简单 | 需要特殊处理防止污染统计 |
| **60 分钟 3 次限制** | 服务保护 > 用户体验 | 频繁验证会被延迟 |
| **CE 无 Browserless** | 简化部署 > 功能完整 | CE 用户只能等待 pageview |

### 5.2 容易混淆的概念澄清

#### 🔴 混淆点 1: "域名"的不同含义

**容易混淆的表述**：
> "验证域名"、"站点域名"、"事件域名"、"URL 域名"

**澄清**：

| 术语 | 定义 | 来源 | 用途 |
|-----|------|------|------|
| **站点域名** | `site.domain` | 用户创建站点时输入 | ⭐ 域名比对的**期望基准** |
| **事件域名** | `event.d` 或 `event.domain` | 脚本中硬编码的配置 | ⭐ 实际发送的**目标域名** |
| **URL 域名** | `url_to_verify` 中的 host | 用户输入或默认 | 浏览器要**访问的地址** |
| **data_domain** | State 中的字段 | `site.domain` | 同"站点域名" |

**验证成功的核心条件**：
```
事件域名 == 站点域名
(event.d)    (site.domain)
```

**URL 域名**可以不同**（例如子路径部署），只要事件域名匹配即可。

---

#### 🔴 混淆点 2: installation_type 的作用

**容易混淆的表述**：
> "installation_type 决定脚本如何生成"

**澄清**：

| 实际作用 | 不影响 |
|---------|--------|
| ❌ **不影响**脚本内容生成 | 脚本内容由功能开关决定 |
| ❌ **不影响**验证逻辑 | 验证只关心 domain 是否匹配 |
| ✅ **只影响**错误消息展示 | 根据安装类型显示不同的修复建议 |
| ✅ **影响**默认安装引导界面 | 显示对应安装类型的说明 |

**代码证据** (`tracker.ex:36-46`):
```elixir
def plausible_main_config(config) do
  %{
    domain: config.site.domain,           # 来自 site，不是 installation_type
    endpoint: tracker_ingestion_endpoint(),
    outboundLinks: config.outbound_links, # 功能开关
    fileDownloads: config.file_downloads,  # 功能开关
    formSubmissions: config.form_submissions # 功能开关
  }
end
```

---

#### 🔴 混淆点 3: "验证成功" vs "有 pageview"

**容易混淆的表述**：
> "验证成功就会有 pageview"

**澄清**：

| 概念 | 含义 | 触发条件 |
|-----|------|---------|
| **验证成功** | 脚本安装正确，能发送事件 | 测试事件 domain 匹配 |
| **有 pageview** | 真实用户访问产生了数据 | 真实用户访问页面 |

**两者的关系**：

```
验证成功 ──────► 脚本可以工作
                      │
                      ▼
                 真实用户访问 ──────► 产生 pageview
                      │
                      ▼
                 自动跳转到仪表板
```

**测试事件的特殊性**：
- 测试事件使用特殊 User-Agent
- 被 `drop_verification_agent` 过滤
- **不会**产生 pageview

**代码证据** (`ingestion/event.ex:197-210`):
```elixir
on_ee do
  @verification_user_agent Plausible.InstallationSupport.user_agent()

  defp drop_verification_agent(event, _context) do
    case event.request.user_agent do
      @verification_user_agent ->
        drop(event, :verification_agent)  # 丢弃验证事件
      _ ->
        event
    end
  end
end
```

---

#### 🔴 混淆点 4: "域名不匹配"错误的含义

**容易混淆的表述**：
> "域名不匹配意味着脚本坏了"

**澄清**：

| 实际含义 | 不代表 |
|---------|--------|
| ✅ 脚本**正在工作** | ❌ 脚本损坏 |
| ✅ 事件**正在发送** | ❌ 网络错误 |
| ✅ 但发送到了**错误的站点** | ❌ CSP 阻止 |
| ✅ **配置错误** | ❌ 代码错误 |

**常见场景**：
1. 用户 A 创建了站点 `example.com`
2. 用户 A 复制了脚本，但不小心用了站点 `other.com` 的脚本 ID
3. 脚本正常工作，事件发送到 `other.com`
4. 验证时发现：事件 domain (`other.com`) ≠ 期望 domain (`example.com`)
5. 显示"域名不匹配"错误

**这是**安全特性**，不是 bug**：
- 防止用户意外将数据发送到错误站点
- 强制用户检查配置正确性

---

#### 🔴 混淆点 5: 脚本 URL 中的 ID  vs 域名

**容易混淆的表述**：
> "脚本 URL 中的域名是目标域名"

**澄清**：

| 脚本 URL 示例 | 含义 |
|-------------|------|
| `https://plausible.io/js/abc123.js` | `plausible.io` 是 Plausible 服务域名 |
| `https://your-self-hosted.com/js/def456.js` | `your-self-hosted.com` 是自托管域名 |

**脚本 URL 中的域名**是 Plausible 服务的地址，**不是**被跟踪站点的域名。

**被跟踪站点的域名**是：
- Web 脚本：硬编码在脚本内容中（`config.domain`）
- NPM 脚本：`init()` 时传入的 `domain` 参数
- 旧版脚本：`data-domain` 属性

---

### 5.3 常见问题排查指南

#### Q1: 验证一直失败，显示"域名不匹配"

**排查步骤**：
1. 检查页面上的脚本 URL 中的 script ID
2. 确认该 ID 对应正确的站点
3. 对于 NPM：检查 `init({ domain: "..." })
4. 对于 GTM：检查模板中输入的 Script ID
5. 对于 WordPress：检查插件设置

**快速验证**：
在浏览器控制台执行：
```javascript
window.plausible  // 检查是否存在
window.plausible.l  // 检查是否初始化
// 发送测试事件查看实际 domain
window.plausible('test', { callback: r => console.log(r) })
```

---

#### Q2: 验证成功但仪表板没有数据

**可能原因**：
1. 验证成功只说明脚本配置正确
2. 但测试事件被过滤，不会产生 pageview
3. 需要真实用户访问才会产生数据

**解决方案**：
- 自己访问页面（使用普通浏览器，不是隐身模式可能被识别为 bot）
- 检查是否有广告拦截器
- 检查是否在 `localhost`（某些情况下可能不发送）

---

#### Q3: 切换安装类型后验证结果不变

**原因**：
- `installation_type` 只影响错误消息
- 不影响脚本生成和验证逻辑
- 真正影响的是脚本内容（功能开关）

**如果想真正改变**：
- 需要修改功能开关（出站链接、文件下载等）
- 或者修改脚本 ID（使用正确站点的脚本）

---

## 6. 完整闭环流程图（补充版）

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    站点 Tracker Script 配置与域名校验完整闭环                      │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────┐
│ Phase 1: 站点创建与配置初始化                                                        │
└─────────────────────────────────────────────────────────────────────────────────────┘

  用户输入域名 ──▶ 创建 Site 记录 ──▶ site.domain = "example.com"
                                                    │
                                                    ▼
                                          ┌─────────────────┐
                                          │ 初始化配置      │
                                          │                 │
                                          │ - 创建         │
                                          │   TrackerScript-│
                                          │   Configuration │
                                          │ - id: 自动生成  │
                                          │ - installation_ │
                                          │   type: "manual"│
                                          │ - 所有功能开关:  │
                                          │   true         │
                                          └────────┬────────┘
                                                   │
                                                   ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│ Phase 2: 脚本生成与嵌入                                                             │
└─────────────────────────────────────────────────────────────────────────────────────┘

                                          ┌─────────────────┐
                                          │ 构建脚本内容   │
                                          │                 │
                                          │ plausible_main_ │
                                          │ config():      │
                                          │ {              │
                                          │   domain:      │
                                          │   "example.com"│
                                          │   endpoint:    │
                                          │   "/api/event" │
                                          │   outboundLinks│
                                          │   : !0         │
                                          │   ...          │
                                          │ }              │
                                          └────────┬────────┘
                                                   │
                                                   ▼
                                          ┌─────────────────┐
                                          │ 替换占位符     │
                                          │                 │
                                          │ <%= @config_js  │
                                          │ %>  ──▶      │
                                          │ {domain:"examp │
                                          │ le.com,...}    │
                                          └────────┬────────┘
                                                   │
                                                   ▼
                                          ┌─────────────────┐
                                          │ 最终脚本       │
                                          │                 │
                                          │ URL:           │
                                          │ /js/{id}.js   │
                                          │                 │
                                          │ 包含硬编码:      │
                                          │ domain =      │
                                          │ "example.com"  │
                                          └────────┬────────┘
                                                   │
                                                   ▼
                                          用户复制到自己的网站
                                          嵌入到 <head> 中
                                                   │
                                                   ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│ Phase 3: 验证流程（EE 版）                                                          │
└─────────────────────────────────────────────────────────────────────────────────────┘

  用户点击 "Verify Script installation"
                    │
                    ▼
          ┌─────────────────┐
          │ 保存配置      │
          │               │
          │ - 更新        │
          │   Tracker-     │
          │   Script-    │
          │   Config-       │
          │   uration    │
          │ - 同步 Goals │
          │ - 清除 CDN   │
          │   缓存       │
          └────────┬────────┘
                   │
                   ▼
          ┌─────────────────┐
          │ 跳转到验证页面 │
          │               │
          │ URL:          │
          │ /example.com/ │
          │ verification  │
          │ ?flow=...&   │
          │ installation │
          │ _type=manual │
          └────────┬────────┘
                   │
                   ▼
          ┌─────────────────┐
          │ 初始化验证状态     │
          │                 │
          │ data_domain =    │
          │   "example.com"│
          │                 │
          │ installation_   │
          │ type = "manual" │
          │ (从 URL 参数)   │
          └────────┬────────┘
                   │
                   ▼
          ┌─────────────────┐
          │ 启动 Browserless│
          │ 检查          │
          │                 │
          │ 1. 访问 URL    │
          │ 2. 执行验证脚本 │
          │ 3. 发送测试事件 │
          └────────┬────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│ Phase 4: 测试事件发送与拦截                                                          │
└─────────────────────────────────────────────────────────────────────────────────────┘

                    用户网站的浏览器环境
                              │
                              ▼
                    ┌─────────────────┐
                    │ 脚本加载      │
                    │               │
                    │ window.         │
                    │ plausible 存在│
                    │               │
                    │ config.domain =│
                    │ "example.com" │
                    │ (硬编码)      │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ 验证脚本执行    │
                    │                 │
                    │ 1. 检测        │
                    │    window.     │
                    │    plausible   │
                    │ 2. 检测初始化  │
                    │ 3. 发送测试事件│
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ 发送测试事件    │
                    │                 │
                    │ window.         │
                    │ plausible(     │
                    │   'verification│
                    │   -agent-test', │
                    │   { callback:   │
                    │     ... }       │
                    │ )               │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ 事件体构建      │
                    │                 │
                    │ {              │
                    │   "n":        │
                    │   "verification│
                    │   -agent-test",│
                    │   "d":         │
                    │   "example.com"│
                    │   ◀── 来自     │
                    │   config.domain│
                    │ }              │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ POST /api/event │
                    │                 │
                    │ 被验证脚本拦截  │
                    │                 │
                    │ 提取 normalized │
                    │ Body:           │
                    │ {              │
                    │   name: "verif │
                    │   ication-agent│
                    │   -test",      │
                    │   domain:      │
                    │   "example.com"│
                    │ }              │
                    │                 │
                    │ responseStatus │
                    │ : 202         │
                    └────────┬────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│ Phase 5: 域名比对与结果反馈                                                          │
└─────────────────────────────────────────────────────────────────────────────────────┘

                    诊断数据返回给服务端
                              │
                              ▼
                    ┌─────────────────┐
                    │ 诊断解析      │
                    │               │
                    │ interpret(    │
                    │   diagnostics,│
                    │   expected_   │
                    │   domain,     │
                    │   url         │
                    │ )             │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ 核心比对        │
                    │               │
                    │ test_event.    │
                    │ normalizedBody │
                    │ .domain       │
                    │ ==             │
                    │ expected_domain │
                    │ (data_domain)  │
                    │                │
                    │ "example.com"  │
                    │ ==             │
                    │ "example.com"  │
                    │ ──▶ 成功!    │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ 结果反馈        │
                    │               │
                    │ success()        │
                    │ ok?: true     │
                    │               │
                    │ 显示:          │
                    │ "Success!"    │
                    │               │
                    │ 按钮:          │
                    │ "Go to the    │
                    │ dashboard"   │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ 等待          │
                    │ 真实 pageview  │
                    │               │
                    │ 轮询检查:      │
                    │ has_pageviews?│
                    │               │
                    │ 真实用户访问 ──▶│
                    │ 产生 pageview │
                    │               │
                    │ 自动跳转到     │
                    │ 仪表板         │
                    └─────────────────┘
```

---

## 7. 关键文件索引（更新版）

| 文件路径 | 职责 | 关键函数/结构 |
|---------|------|-----------|
| `lib/plausible_web/live/installation.ex` | 安装引导 LiveView | `handle_event("submit"...) |
| `lib/plausible_web/live/verification.ex` | 验证 LiveView | `get_installation_type/2` |
| `lib/plausible_web/tracker.ex` | 脚本生成核心 | `plausible_main_config/1` |
| `lib/plausible/site/tracker_script_configuration.ex` | 配置 Schema | `schema "tracker_script_configuration"` |
| `extra/lib/plausible/installation_support/verification/checks.ex` | 检查执行 | `interpret_diagnostics/1` |
| `extra/lib/plausible/installation_support/verification/diagnostics.ex` | 诊断解析 | `interpret/3` |
| `extra/lib/plausible/installation_support/state.ex` | 状态结构 | `defstruct data_domain:...` |
| `tracker/src/config.js` | 前端配置 | `init/1` |
| `tracker/installation_support/verifier.js` | 验证脚本 | `getNormalizedPlausibleEventBody/1` |
| `lib/plausible/ingestion/event.ex` | 事件处理 | `drop_verification_agent/2` |

---

## 8. 核心概念速查表

| 概念 | 值/行为 |
|-----|---------|
| **成功条件** | `test_event.normalizedBody.domain == data_domain` AND `response_status in [200, 202]` |
| **期望域名来源** | `site.domain`（用户创建站点时输入） |
| **实际域名来源** | 脚本中硬编码的 `config.domain` |
| **installation_type 作用** | 仅影响错误消息展示，不影响脚本生成 |
| **测试事件是否产生 pageview** | ❌ 否，被特殊 UA 过滤 |
| **URL 参数 vs 数据库** | URL 参数优先级更高 |
| **Web 脚本 domain 可覆盖** | ❌ 否，安全设计 |
| **CE 版验证方式** | 直接等待 pageview，无 Browserless |
