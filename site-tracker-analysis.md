# 站点 Tracker Script 配置与域名校验闭环分析

## 1. 概述

本文档详细分析了 Plausible 站点创建后，tracker script 配置与域名校验的完整闭环流程。重点关注安装引导、事件匹配、验证反馈和边界情况处理四个核心环节。

## 2. 安装引导流程

### 2.1 流程架构

安装引导流程由 `PlausibleWeb.Live.Installation` LiveView 模块协调，为用户提供多种安装方式的选择和配置。

**关键文件：**
- `lib/plausible_web/live/installation.ex` - 主安装引导 LiveView
- `lib/plausible_web/live/installation/instructions.ex` - 安装说明组件
- `lib/plausible_web/tracker.ex` - Tracker 脚本核心模块

### 2.2 安装类型检测与选择

系统支持以下安装类型：

| 安装类型 | 描述 | 可用版本 |
|---------|------|---------|
| `manual` | 手动嵌入脚本标签 | CE/EE |
| `wordpress` | WordPress 插件安装 | CE/EE |
| `npm` | NPM 包集成 | CE/EE |
| `gtm` | Google Tag Manager 模板 | EE Only |

**自动检测机制（EE版）：**

```elixir
# lib/plausible_web/live/installation.ex:208-222
defp detect_recommended_installation_type(flow, site) do
  with {:ok, detection_result} <-
         Detection.Checks.run_with_rate_limit(nil, site.domain,
           detect_v1?: flow == Flows.review(),
           report_to: nil,
           slowdown: 0,
           async?: false
         ),
       %Result{ok?: true, data: data} <-
         Detection.Checks.interpret_diagnostics(detection_result) do
    {data.suggested_technology, data.v1_detected}
  else
    _ -> {PlausibleWeb.Tracker.fallback_installation_type(), false}
  end
end
```

### 2.3 Tracker Script 配置模型

**数据结构定义：**

```elixir
# lib/plausible/site/tracker_script_configuration.ex:23-39
@primary_key {:id, Plausible.Ecto.Types.TrackerScriptNanoid, autogenerate: true}
schema "tracker_script_configuration" do
  field :installation_type, Ecto.Enum, values: [:manual, :wordpress, :gtm, :npm, nil]

  field :track_404_pages, :boolean, default: false
  field :hash_based_routing, :boolean, default: false
  field :outbound_links, :boolean, default: false
  field :file_downloads, :boolean, default: false
  field :revenue_tracking, :boolean, default: false
  field :tagged_events, :boolean, default: false
  field :form_submissions, :boolean, default: false
  field :pageview_props, :boolean, default: false

  belongs_to :site, Plausible.Site
  timestamps()
end
```

### 2.4 脚本生成机制

**动态脚本构建：**

```elixir
# lib/plausible_web/tracker.ex:49-67
def build_script(
      %TrackerScriptConfiguration{site: %{domain: _domain}} = tracker_script_configuration
    ) do
  config_js_content =
    tracker_script_configuration
    |> plausible_main_config()
    |> Enum.flat_map(fn
      {key, value} when is_binary(value) -> ["#{key}:#{JSON.encode!(value)}"]
      {key, true} -> ["#{key}:!0"]
      {_key, false} -> []
    end)
    |> Enum.sort_by(&String.length/1, :desc)
    |> Enum.join(",")

  @plausible_main_script
  |> String.replace("\"<%= @config_js %>\"", "{#{config_js_content}}")
end
```

**脚本嵌入代码（Manual 安装）：**

```html
<!-- Privacy-friendly analytics by Plausible -->
<script async src="https://plausible.io/js/{script_id}.js"></script>
<script>
  window.plausible=window.plausible||function(){(plausible.q=plausible.q||[]).push(arguments)},plausible.init=plausible.init||function(i){plausible.o=i||{}};
  plausible.init()
</script>
```

### 2.5 可选功能配置

用户可在安装界面启用以下可选测量功能：

| 功能 | 描述 | 默认值 |
|-----|------|--------|
| Outbound Links | 自动追踪外部链接点击 | true |
| File Downloads | 自动追踪文件下载 | true |
| Form Submissions | 自动追踪表单提交 | true |
| 404 Error Pages | 追踪404错误页面 | true |
| Hash-based Routing | 支持哈希路由 | false |
| Tagged Events | 手动标签事件 | false |
| Custom Properties | 自定义属性 | false |
| E-commerce Revenue | 电商收入追踪 | false |

**配置与 Goal 同步机制：**

```elixir
# lib/plausible_web/tracker.ex:214-230
defp sync_goals(site, original_config, updated_config) do
  [:track_404_pages, :outbound_links, :file_downloads, :form_submissions]
  |> Enum.map(fn key ->
    {key, Map.get(original_config, key, false), Map.get(updated_config, key, false)}
  end)
  |> Enum.each(fn
    {:track_404_pages, false, true} -> Plausible.Goals.create_404(site)
    {:track_404_pages, true, false} -> Plausible.Goals.delete_404(site)
    {:outbound_links, false, true} -> Plausible.Goals.create_outbound_links(site)
    {:outbound_links, true, false} -> Plausible.Goals.delete_outbound_links(site)
    {:file_downloads, false, true} -> Plausible.Goals.create_file_downloads(site)
    {:file_downloads, true, false} -> Plausible.Goals.delete_file_downloads(site)
    {:form_submissions, false, true} -> Plausible.Goals.create_form_submissions(site)
    {:form_submissions, true, false} -> Plausible.Goals.delete_form_submissions(site)
    _ -> nil
  end)
end
```

## 3. 事件匹配机制

### 3.1 验证架构概览

验证流程通过浏览器自动化服务（Browserless）在目标网站上执行验证脚本，检测 tracker script 的安装状态和事件发送能力。

**关键文件：**
- `extra/lib/plausible/installation_support/checks/verify_installation.ex` - 验证检查执行器
- `tracker/installation_support/verifier.js` - 浏览器端验证脚本
- `extra/lib/plausible/installation_support/verification/diagnostics.ex` - 诊断结果解析

### 3.2 浏览器端验证流程

**验证脚本执行架构：**

```javascript
// tracker/installation_support/verifier.js:12-119
async function verifyPlausibleInstallation(options) {
  const disallowedByCsp = checkDisallowedByCSP(responseHeaders, cspHostToCheck)
  
  forceIgnoreWebdriverCondition()
  const { stopRecording, getInterceptedFetch } = startRecordingEventFetchCalls()

  const {
    plausibleIsInitialized,
    plausibleIsOnWindow,
    plausibleVersion,
    plausibleVariant,
    testEvent,
    cookiesConsentResult,
    error: testPlausibleFunctionError
  } = await testPlausibleFunction({ timeoutMs, debug })
  
  const trackerIsInHtml = isInHtml(trackerScriptSelector)
  
  let interceptedTestEvent = getInterceptedFetch('verification-agent-test')
  
  // 兼容旧版 v1 脚本
  if (!interceptedTestEvent && [200, 202].includes(testEvent.callbackResult?.status)) {
    // 处理 legacy data-domain 方式
  }
}
```

### 3.3 测试事件发送机制

**事件发送与拦截：**

```javascript
// tracker/installation_support/verifier.js:208-308
async function testPlausibleFunction({ timeoutMs, debug }) {
  return new Promise((_resolve) => {
    // 1. 轮询检测 window.plausible 是否存在
    plausibleOnWindowPollInterval = setInterval(
      () => plausibleIsOnWindow
        ? clearInterval(plausibleOnWindowPollInterval)
        : (plausibleIsOnWindow = isPlausibleOnWindow()),
      10
    )

    // 2. 轮询检测是否已初始化
    plausibleInitializedPollInterval = setInterval(() => {
      if (plausibleIsInitialized) {
        plausibleVersion = getPlausibleVersion()
        plausibleVariant = getPlausibleVariant()
        clearInterval(plausibleInitializedPollInterval)
      } else {
        plausibleIsInitialized = isPlausibleInitialized()
      }
    }, 10)

    // 3. 发送测试事件
    testEventPollInterval = setInterval(() => {
      if (plausibleIsOnWindow && plausibleIsInitialized) {
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

**Fetch 拦截机制：**

```javascript
// tracker/installation_support/verifier.js:144-186
function startRecordingEventFetchCalls() {
  const interceptions = new Map()
  const originalFetch = window.fetch
  
  window.fetch = function (url, options = {}) {
    let identifier = null
    const normalizedEventBody = getNormalizedPlausibleEventBody(options)
    
    if (normalizedEventBody) {
      identifier = normalizedEventBody.name
      interceptions.set(identifier, {
        request: { url, normalizedBody: normalizedEventBody }
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

### 3.4 事件体规范化

**事件体解析：**

```javascript
// tracker/installation_support/verifier.js:121-142
function getNormalizedPlausibleEventBody(fetchOptions) {
  try {
    const body = JSON.parse(fetchOptions.body ?? '{}')
    let name = null
    let domain = null
    let version = null

    if (
      fetchOptions.method === 'POST' &&
      (typeof body?.n === 'string' || typeof body?.name === 'string') &&
      (typeof body?.d === 'string' || typeof body?.domain === 'string')
    ) {
      name = body?.n || body?.name
      domain = body?.d || body?.domain
      version = body?.v || body?.version
    }
    return name && domain ? { name, domain, version } : null
  } catch (_error) {
    // ignore error
  }
}
```

**支持的字段格式（兼容新旧版本）：**

| 新字段 | 旧字段 | 描述 |
|-------|-------|------|
| `name` | `n` | 事件名称 |
| `domain` | `d` | 站点域名 |
| `version` | `v` | 脚本版本 |

### 3.5 服务端验证代理执行

**Browserless 集成：**

```elixir
# extra/lib/plausible/installation_support/checks/verify_installation.ex:20-72
@puppeteer_wrapper_code """
export default async function({ page, context: { url, userAgent, maxAttempts, timeoutBetweenAttemptsMs, ...functionContext } }) {
  try {
    await page.setUserAgent(userAgent)
    const response = await page.goto(url)
    const responseStatus = response.status()
    const responseHeaders = response.headers()

    async function verify() {
      await page.evaluate(() => {#{@verifier_code}}) // 注入验证脚本
      return await page.evaluate(
        (c) => window.verifyPlausibleInstallation(c),
        { ...functionContext, responseHeaders }
      );
    }

    // 重试机制
    let lastError;
    for (let attempts = 1; attempts <= maxAttempts; attempts++) {
      try {
        const output = await verify();
        return {
          data: {
            ...output.data,
            attempts,
            responseStatus
          },
        };
      } catch (error) {
        lastError = error;
        if (typeof error?.message === "string" &&
            error.message.toLowerCase().includes("execution context")) {
          await new Promise((resolve) => setTimeout(resolve, timeoutBetweenAttemptsMs));
          continue;
        }
        throw error
      }
    }
    throw lastError;
  } catch (error) {
    return {
      data: {
        completed: false,
        error: { message: error?.message ?? JSON.stringify(error) }
      }
    }
  }
}
"""
```

## 4. 验证反馈机制

### 4.1 诊断数据结构

**诊断字段定义：**

```elixir
# extra/lib/plausible/installation_support/verification/diagnostics.ex:7-23
defstruct [
  :selected_installation_type,
  :disallowed_by_csp,
  :tracker_is_in_html,
  :plausible_is_on_window,
  :plausible_is_initialized,
  :plausible_version,
  :plausible_variant,
  :diagnostics_are_from_cache_bust,
  :test_event,
  :cookies_consent_result,
  :response_status,
  :service_error,
  :attempts
]
```

### 4.2 诊断解析流程

**解析入口：**

```elixir
# extra/lib/plausible/installation_support/verification/diagnostics.ex:68-288
def interpret(diagnostics, expected_domain, url)
```

### 4.3 成功判定条件

**成功场景：**

```elixir
# extra/lib/plausible/installation_support/verification/diagnostics.ex:87-102
def interpret(
      %__MODULE__{
        test_event: %{
          "normalizedBody" => %{
            "domain" => domain
          },
          "responseStatus" => response_status
        },
        service_error: nil
      },
      expected_domain,
      _url
    )
    when response_status in [200, 202] and
           domain == expected_domain,
    do: success()
```

**成功条件：**
1. 测试事件响应状态码为 200 或 202
2. 事件中的 `domain` 与期望域名完全匹配
3. 无服务端错误

### 4.4 错误类型与反馈

**错误分类体系：**

| 错误类型 | 触发条件 | 用户反馈 |
|---------|---------|---------|
| 缓存问题 | 仅在清除缓存后成功 | 提示清除站点缓存 |
| 域名不匹配 | 事件 domain 与期望不符 | 检查脚本配置 |
| CSP 阻止 | CSP 禁止加载脚本 | 添加 plausible.io 到白名单 |
| 脚本未找到 | HTML 中无脚本且 window 无对象 | 检查脚本安装 |
| 网络错误 | 非 200/202 响应 | 检查代理或网络配置 |
| 站点不可达 | 无法访问目标 URL | 检查域名或手动验证 |
| 服务超时 | Browserless 超时 | 稍后重试 |

**关键错误处理代码：**

```elixir
# 缓存问题处理
@error_succeeds_only_after_cache_bust Error.new!(%{
  message: "We detected an issue with your site's cache",
  recommendation: "Please clear the cache for your site...",
  url: "https://plausible.io/docs/troubleshoot-integration#have-you-cleared-the-cache-of-your-site"
})

# CSP 阻止处理
@error_csp_disallowed Error.new!(%{
  message: "We encountered an issue with your site's Content Security Policy (CSP)",
  recommendation: "Please add plausible.io domain specifically to the allowed list...",
  url: "https://plausible.io/docs/troubleshoot-integration#does-your-site-use-a-content-security-policy-csp"
})

# 域名不匹配处理
defp error_unexpected_domain(selected_installation_type) do
  case selected_installation_type do
    "npm" -> @error_unexpected_domain_for_npm
    "gtm" -> @error_unexpected_domain_for_gtm
    "wordpress" -> @error_unexpected_domain_for_wordpress
    _ -> @error_unexpected_domain_for_manual
  end
end
```

### 4.5 前端验证状态展示

**验证组件渲染逻辑：**

```elixir
# lib/plausible_web/live/components/verification.ex:30-53
def render(assigns) do
  ~H"""
  <div id="verification-ui">
    <.render_progress :if={not @finished?} message={@message} />
    <.render_success
      :if={@finished? and @success?}
      awaiting_first_pageview?={@awaiting_first_pageview?}
      domain={@domain}
    />
    <.render_failed
      :if={@finished? and not @success?}
      interpretation={@interpretation}
      attempts={@attempts}
      domain={@domain}
      flow={@flow}
      installation_type={@installation_type}
    />
  </div>
  """
end
```

## 5. 边界情况处理

### 5.1 旧版脚本兼容（v1 检测）

**v1 脚本自动检测：**

```elixir
# lib/plausible_web/live/installation.ex:229-259
defp outdated_script_notice(assigns) do
  ~H"""
  <div :if={
    @recommended_installation_type.result == "manual" and
      @installation_type.result == "manual"
  }>
    <.notice class="mt-4" theme={:yellow}>
      Your website is running an outdated version of the tracking script. Please
      <.styled_link new_tab href="https://plausible.io/docs/script-update-guide">
        update
      </.styled_link>
      your tracking script before continuing
    </.notice>
  </div>
  """
end
```

**v1 脚本事件兼容处理：**

```javascript
// tracker/installation_support/verifier.js:64-89
if (
  !interceptedTestEvent &&
  [200, 202].includes(testEvent.callbackResult?.status)
) {
  log(
    `The callback result indicates a successful request, assuming legacy .compat installation that uses XMLHttpRequest`
  )
  const firstLegacySnippet = document.querySelector(
    'script[data-domain][src]'
  )
  if (firstLegacySnippet) {
    const domainString = firstLegacySnippet.getAttribute('data-domain')
    const firstDomain = domainString && domainString.split(',').shift()

    interceptedTestEvent = {
      request: {
        normalizedBody: {
          __legacyCompatInstallation: true,
          domain: firstDomain
        }
      },
      response: { status: testEvent.callbackResult.status }
    }
  }
}
```

### 5.2 自定义 URL 验证

**非标准路径支持：**

```elixir
# lib/plausible_web/live/verification.ex:97-105
def handle_event("verify-custom-url", %{"custom_url" => custom_url}, socket) do
  socket =
    socket
    |> assign(url_to_verify: custom_url)
    |> assign(custom_url_input?: false)

  launch_delayed(socket)
  {:noreply, reset_component(socket)}
end
```

**适用场景：**
- 站点部署在子路径（如 `/blog`）
- 使用非标准端口
- 需要验证特定页面

### 5.3 速率限制与重试机制

**验证频率限制：**

```elixir
# lib/plausible_web/live/verification.ex:114-121
case Plausible.RateLimit.check_rate(
       "site_verification:#{domain}",
       :timer.minutes(60),
       3
     ) do
  {:allow, _} -> :ok
  {:deny, _} -> :timer.sleep(@slowdown_for_frequent_checking)
end
```

**限制规则：**
- 每 60 分钟最多 3 次验证
- 超过限制后添加 5 秒延迟

**多级重试机制：**

1. **浏览器内重试**（max_attempts: 2）：处理页面导航延迟
2. **HTTP 请求重试**（max_retries: 1）：处理临时网络问题
3. **用户手动重试**：通过 "Verify installation again" 按钮

### 5.4 验证代理事件过滤

**防止测试事件污染统计：**

```elixir
# lib/plausible/ingestion/event.ex:197-210
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
else
  defp drop_verification_agent(%__MODULE__{} = event, _context), do: event
end
```

**过滤机制：**
- 使用特定的 User-Agent 标识验证请求
- 在事件处理管道早期丢弃这些事件
- 不消耗页面视图配额

### 5.5 域名变更过渡期

**域名切换双接受机制：**

```elixir
# lib/plausible/site/domain.ex:4-21
@moduledoc """
Basic interface for domain changes.

We will set a transition period of #{@expire_threshold_hours} hours
during which, both old and new domains, will be accepted as traffic
identifiers to the same site.
"""

@expire_threshold_hours 72
```

**过渡期特性：**
- 72 小时内同时接受新旧域名的事件
- 定期任务清理过期过渡状态
- 数据库触发器确保域名唯一性

### 5.6 Cookie Consent 处理

**自动处理 CMP（Consent Management Platform）：**

```javascript
// tracker/installation_support/verifier.js:286-306
cookiesConsentResult = initializeCookieConsentEngine({
  debug,
  onConsentDone: (cmp) => {
    if (resolved) return
    cookiesConsentResult = { handled: true, cmp }
  },
  onConsentError: (err) => {
    if (resolved) return
    cookiesConsentResult = { handled: false, error: err }
  },
  onLifecycleUpdate: (lifecycle) => {
    if (resolved) return
    if (cookiesConsentResult.handled !== null) return
    if (lifecycle === 'done') {
      cookiesConsentResult = { handled: true }
    } else {
      cookiesConsentResult.engineLifecycle = lifecycle
    }
  }
})
```

### 5.7 超级管理员诊断信息

**内部调试支持：**

```elixir
# lib/plausible_web/live/components/verification.ex:171-201
defp render_super_admin_diagnostics(assigns) do
  ~H"""
  <.focus_box>
    <div
      class="flex flex-col dark:text-gray-200"
      x-data="{ showDiagnostics: false }"
      id="super-admin-report"
    >
      <p class="text-sm">
        <a href="#" @click.prevent="showDiagnostics = !showDiagnostics" class="bg-yellow-100 dark:bg-yellow-800/40">
          As a super-admin, you're eligible to see diagnostics details. Click to expand.
        </a>
      </p>
      <div x-show="showDiagnostics" x-cloak>
        <.focus_list>
          <:item :for={{diag, value} <- Map.from_struct(@verification_state.diagnostics)}>
            <span class="text-sm">
              {Phoenix.Naming.humanize(diag)}:
              <span class="font-mono">{to_string_value(value)}</span>
            </span>
          </:item>
        </.focus_list>
      </div>
    </div>
  </.focus_box>
  """
end
```

## 6. 完整闭环流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           站点创建后 Tracker 配置与验证闭环                     │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  1. 站点创建  │────▶│ 2. 安装引导  │────▶│ 3. 脚本配置  │────▶│ 4. 验证触发  │
│              │     │              │     │              │     │              │
│ - 输入域名   │     │ - 检测技术栈 │     │ - 生成Script │     │ - 提交验证   │
│ - 创建记录   │     │ - 选择安装   │     │ - 配置Options│     │ - 跳转验证页 │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
                                                          │
                                                          ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ 8. 结果反馈  │◀────│ 7. 诊断解析  │◀────│ 6. 事件匹配  │◀────│ 5. 验证执行  │
│              │     │              │     │              │     │              │
│ - 成功UI    │     │ - 成功判定   │     │ - 拦截Fetch  │     │ - Browserless│
│ - 失败详情  │     │ - 错误分类   │     │ - 提取Domain │     │ - 执行脚本   │
│ - 重试入口  │     │ - 建议生成   │     │ - 状态码校验 │     │ - 发送测试   │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              9. 后续流程                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│ 成功:                                                                         │
│   - 显示"Go to dashboard"按钮                                                 │
│   - 若尚无 pageview，显示 "Awaiting your first pageview..."                  │
│   - 后台轮询 pageview，有数据后自动跳转仪表板                                  │
│                                                                               │
│ 失败:                                                                         │
│   - 显示具体错误信息和修复建议                                                  │
│   - 提供"Verify installation again"按钮                                       │
│   - 尝试 >=3 次（EE）后显示联系支持入口                                        │
│   - 提供"回到安装说明"和"跳过验证去设置"链接                                   │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 7. 关键文件索引

| 模块 | 文件路径 | 职责 |
|-----|---------|------|
| 安装引导 LiveView | `lib/plausible_web/live/installation.ex` | 协调安装流程、类型选择 |
| 安装说明组件 | `lib/plausible_web/live/installation/instructions.ex` | 各安装类型的具体说明 |
| Tracker 核心模块 | `lib/plausible_web/tracker.ex` | 脚本生成、配置管理、缓存 |
| 配置数据模型 | `lib/plausible/site/tracker_script_configuration.ex` | 数据库 Schema |
| 验证 LiveView | `lib/plausible_web/live/verification.ex` | 验证流程协调 |
| 验证组件 | `lib/plausible_web/live/components/verification.ex` | 验证 UI 渲染 |
| 诊断解析 | `extra/lib/plausible/installation_support/verification/diagnostics.ex` | 结果解析、错误分类 |
| 验证执行器 | `extra/lib/plausible/installation_support/checks/verify_installation.ex` | Browserless 集成 |
| 浏览器验证脚本 | `tracker/installation_support/verifier.js` | 页面内验证逻辑 |
| 事件处理 | `lib/plausible/ingestion/event.ex` | 事件接收、过滤、处理 |
| 域名管理 | `lib/plausible/site/domain.ex` | 域名变更、过渡期 |

## 8. 总结

### 8.1 闭环核心机制

1. **多安装类型支持**：Manual、WordPress、NPM、GTM（EE），自动检测推荐
2. **动态脚本生成**：基于配置动态生成，包含启用的功能模块
3. **真实浏览器验证**：通过 Browserless 执行真实页面环境中的验证
4. **事件级验证**：不仅检测脚本存在，还验证事件发送和域名匹配
5. **精细错误反馈**：针对不同失败场景提供针对性的修复建议
6. **多级重试机制**：浏览器内重试、HTTP 重试、用户手动重试
7. **完善边界处理**：旧版兼容、自定义 URL、Cookie Consent、速率限制等

### 8.2 安全与隔离

- 验证事件使用特殊 User-Agent，被服务端过滤不记入统计
- 域名变更有过渡期，确保平滑迁移
- 速率限制防止滥用验证服务

### 8.3 用户体验优化

- 自动检测技术栈，推荐最佳安装方式
- 可视化验证进度和状态
- 失败时提供具体的错误信息和修复链接
- 支持自定义 URL 验证非标准部署场景
