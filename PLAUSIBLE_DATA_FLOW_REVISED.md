# Plausible Analytics 数据链路深度分析报告（修正版）

> 分析日期：2026-05-02
> 修正内容：追踪脚本主路径、事件字段、指标口径映射、缓冲刷新时效性影响

---

## 一、数据链路总览（修正版）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      数据链路全景图                                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────┐                                                        │
│  │  1. 前端追踪    │                                                        │
│  │  采集层        │                                                        │
│  └────────┬────────┘                                                        │
│           │                                                                 │
│           ▼                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  tracker/src/ (源代码目录)                                            │   │
│  │  - plausible.js      ← 主入口，初始化模块                              │   │
│  │  - track.js         ← 核心追踪逻辑，构建 payload                      │   │
│  │  - networking.js    ← 发送 POST 请求 /api/event                       │   │
│  │  - config.js        ← 配置管理                                         │   │
│  │  - autocapture.js   ← 自动捕获页面浏览                                  │   │
│  │  - engagement.js    ← 参与度追踪（滚动、停留时间）                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│           │                                                                 │
│           │ POST /api/event (Content-Type: text/plain)                     │
│           ▼                                                                 │
│  ┌─────────────────┐                                                        │
│  │  2. 服务端请求  │                                                        │
│  │  接收与处理层   │                                                        │
│  └────────┬────────┘                                                        │
│           │                                                                 │
│           ▼                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  lib/plausible_web/                                                   │   │
│  │  - router.ex                    → /api/event 端点                      │   │
│  │  - controllers/api/external_controller.ex                            │   │
│  │    → event/2 处理请求，返回 202 Accepted                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│           │                                                                 │
│           ▼                                                                 │
│  ┌─────────────────┐                                                        │
│  │  3. 事件处理    │                                                        │
│  │  与会话管理    │                                                        │
│  └────────┬────────┘                                                        │
│           │                                                                 │
│           ▼                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  lib/plausible/ingestion/                                            │   │
│  │  - event.ex         → 15步管道处理（过滤→提取→验证）                  │   │
│  │  - request.ex       → 构建 Request 结构体                              │   │
│  │  - persistor/embedded.ex                                              │   │
│  │    → 会话缓存 + 写入缓冲区                                             │   │
│  │                                                                         │   │
│  │  lib/plausible/session/                                                │   │
│  │  - cache_store.ex   → 会话缓存管理（30分钟超时）                       │   │
│  │  - write_buffer.ex  → 会话写入缓冲区                                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│           │                                                                 │
│           ▼                                                                 │
│  ┌─────────────────┐                                                        │
│  │  4. ClickHouse  │                                                        │
│  │  批量写入层     │                                                        │
│  └────────┬────────┘                                                        │
│           │                                                                 │
│           ▼                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  lib/plausible/ingestion/write_buffer.ex                              │   │
│  │                                                                         │   │
│  │  默认配置:                                                              │   │
│  │  - CLICKHOUSE_FLUSH_INTERVAL_MS = 5000  (5秒)                        │   │
│  │  - CLICKHOUSE_MAX_BUFFER_SIZE_BYTES = 100000  (100KB)                │   │
│  │                                                                         │   │
│  │  刷新触发条件:                                                          │   │
│  │  1. 缓冲大小 >= 100KB                                                   │   │
│  │  2. 定时 tick (每5秒)                                                   │   │
│  │  3. 进程终止 (terminate callback)                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│           │                                                                 │
│           ▼ (写入 ClickHouse)                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  ClickHouse 表                                                        │   │
│  │  - events_v2     → 事件表（事实表）                                   │   │
│  │  - sessions_v2   → 会话表（使用 sign 字段实现 MVCC 式更新）            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│           │                                                                 │
│           │ (查询方向: 从 ClickHouse 读取数据)                              │
│           ▼                                                                 │
│  ┌─────────────────┐                                                        │
│  │  5. 指标查询    │                                                        │
│  │  与口径映射    │                                                        │
│  └────────┬────────┘                                                        │
│           │                                                                 │
│           ▼                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  lib/plausible/stats/                                                 │   │
│  │  - metrics.ex       → 指标定义（原子指标 + 特殊指标）                  │   │
│  │  - sql/special_metrics.ex                                             │   │
│  │    → 特殊指标计算（conversion_rate, bounce_rate 等）                   │   │
│  │  - query_runner.ex  → 查询执行器                                       │   │
│  │  - current_visitors.ex                                                 │   │
│  │    → 实时访客（直接查过去5分钟数据，不经过缓存）                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│           │                                                                 │
│           ▼                                                                 │
│  ┌─────────────────┐                                                        │
│  │  6. 前端仪表盘  │                                                        │
│  │  展示层        │                                                        │
│  └────────┬────────┘                                                        │
│           │                                                                 │
│           ▼                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  assets/js/dashboard/                                                 │   │
│  │  - stats/graph/fetch-top-stats.ts                                     │   │
│  │    → chooseMetrics() 根据过滤条件动态选择指标                           │   │
│  │  - stats/metrics.ts                                                    │   │
│  │    → getMetricLabel() 指标标签映射                                      │   │
│  │  - api.ts                                                              │   │
│  │    → stats() POST /api/stats/:domain/query                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 二、关键修正点

### 2.1 追踪脚本主路径（重大修正）

#### 之前的误判

| 误判内容 | 实际情况 |
|----------|----------|
| `tracker.ex` 生成追踪脚本 | `tracker.ex` 只负责**动态配置注入**，不生成脚本代码 |
| `p.js` 是主脚本 | `p.js` 是**编译产物**，不是源代码 |
| 脚本逻辑分散 | 真正的源代码在 `tracker/src/` 目录下 |

#### 正确的架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      追踪脚本架构（正确版）                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  【源代码目录】                                                              │
│  tracker/src/                                                                │
│  ├── plausible.js           ← 主入口文件，初始化所有模块                    │
│  ├── track.js               ← 核心追踪逻辑，构建事件 payload                │
│  ├── networking.js          ← 发送 POST 请求到 /api/event                   │
│  ├── config.js              ← 配置管理（脚本配置 + 站点配置）               │
│  ├── autocapture.js         ← 自动捕获页面浏览                              │
│  ├── engagement.js          ← 参与度追踪（滚动深度、停留时间）               │
│  ├── custom-events.js       ← 自定义事件处理                                │
│  └── revenue.js             ← 收入追踪（EE 版本）                           │
│                                                                              │
│  【编译配置】                                                                │
│  tracker/compiler/variants.json                                             │
│  ├── manualVariants     ← 主要变体（推荐使用）                              │
│  │   ├── plausible-web.js     ← 新版 Web snippet                           │
│  │   └── npm_package/        ← NPM 包                                      │
│  └── legacyVariants     ← 旧版变体（兼容旧集成方式）                         │
│      ├── plausible.js           ← 基础版                                    │
│      ├── plausible.hash.js      ← 支持 hash 路由                           │
│      ├── plausible.outbound-links.js  ← 支持出站链接追踪                   │
│      └── ... 共 100+ 种变体组合                                             │
│                                                                              │
│  【编译产物目录】                                                            │
│  priv/tracker/js/                                                            │
│  ├── p.js           ← 压缩后的脚本（实际提供给用户下载的文件）               │
│  └── .gitkeep       ← 占位文件                                              │
│                                                                              │
│  【动态配置注入】                                                            │
│  lib/plausible_web/tracker.ex                                                │
│  ├── build_script/1         ← 读取编译后的脚本                              │
│  ├── 替换 <%= @config_js %>  ← 注入站点级配置                               │
│  └── plausible_main_config/1 ← 从数据库读取配置                             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 编译时功能开关（COMPILE_* 全局变量）

编译时通过 `COMPILE_*` 全局变量控制功能开关，未使用的代码会被编译器移除（Tree Shaking）。

| 全局变量 | 功能说明 | 包含的变体 |
|----------|----------|------------|
| `COMPILE_PLAUSIBLE_WEB` | 新版 Web snippet 模式 | `plausible-web.js` |
| `COMPILE_PLAUSIBLE_LEGACY_VARIANT` | 旧版变体模式 | 所有 `plausible.*.js` |
| `COMPILE_PLAUSIBLE_NPM` | NPM 包模式 | `npm_package/plausible.js` |
| `COMPILE_HASH` | 支持 hash 路由 | `*.hash.*.js` |
| `COMPILE_EXCLUSIONS` | 支持页面排除规则 | `*.exclusions.*.js` |
| `COMPILE_OUTBOUND_LINKS` | 自动追踪出站链接 | `*.outbound-links.*.js` |
| `COMPILE_FILE_DOWNLOADS` | 自动追踪文件下载 | `*.file-downloads.*.js` |
| `COMPILE_MANUAL` | 手动触发模式 | `*.manual.*.js` |
| `COMPILE_LOCAL` | 支持 localhost 调试 | `*.local.*.js` |
| `COMPILE_PAGEVIEW_PROPS` | 页面浏览自定义属性 | `*.pageview-props.*.js` |
| `COMPILE_TAGGED_EVENTS` | 标记事件 | `*.tagged-events.*.js` |
| `COMPILE_REVENUE` | 收入追踪 | `*.revenue.*.js` |
| `COMPILE_CONFIG` | 动态配置支持 | 所有 Web/NPM 变体 |
| `COMPILE_COMPAT` | 兼容性模式（XMLHttpRequest） | `*.compat.*.js` |

#### 主入口：plausible.js

```javascript
// tracker/src/plausible.js

import { init as initEngagementTracking } from './engagement'
import { init as initConfig, getOptionsWithDefaults, config } from './config'
import { init as initCustomEvents } from './custom-events'
import { init as initAutocapture } from './autocapture'
import { track } from './track'

function init(overrides) {
  const options = getOptionsWithDefaults(overrides || {})
  
  initConfig(options)           // 初始化配置
  initEngagementTracking()      // 启动参与度追踪（滚动、停留时间）
  
  // 如果不是 manual 模式，或者配置了 autoCapturePageviews
  if (!COMPILE_MANUAL || (COMPILE_CONFIG && config.autoCapturePageviews)) {
    initAutocapture(track)      // 自动捕获页面浏览
  }
  
  // 初始化自定义事件（出站链接、文件下载等）
  if (COMPILE_PLAUSIBLE_WEB || COMPILE_PLAUSIBLE_NPM || 
      COMPILE_OUTBOUND_LINKS || COMPILE_FILE_DOWNLOADS || COMPILE_TAGGED_EVENTS) {
    initCustomEvents()
  }
  
  // Web 变体：处理之前排队的事件
  if (COMPILE_PLAUSIBLE_WEB || COMPILE_PLAUSIBLE_LEGACY_VARIANT) {
    var queue = (window.plausible && window.plausible.q) || []
    for (var i = 0; i < queue.length; i++) {
      track.apply(this, queue[i])
    }
    
    window.plausible = track
    window.plausible.init = init
    window.plausible.v = COMPILE_TRACKER_SCRIPT_VERSION
    window.plausible.l = true
  }
}

// Web 变体自动初始化
if (COMPILE_PLAUSIBLE_WEB) {
  window.plausible = (window.plausible || {})
  if (plausible.o) {
    init(plausible.o)    // 如果有预配置的选项，立即初始化
  }
  plausible.init = init
} else if (COMPILE_PLAUSIBLE_LEGACY_VARIANT) {
  init()  // 旧版变体自动初始化
}
```

#### 动态配置注入机制

**服务端配置注入** (`lib/plausible_web/tracker.ex:46-77`)

```elixir
def build_script(tracker_script_configuration) do
  # 从数据库提取配置，转换为 JS 格式
  config_js_content =
    tracker_script_configuration
    |> plausible_main_config()
    |> Enum.flat_map(fn
      {key, value} when is_binary(value) -> ["#{key}:#{JSON.encode!(value)}"]
      {key, true} -> ["#{key}:!0"]  # 压缩 true 为 !0 节省字节
      {_key, false} -> []             # false 不包含，让默认值生效
    end)
    |> Enum.join(",")

  # 替换脚本中的配置占位符
  @plausible_main_script
  |> String.replace("\"<%= @config_js %>\"", "{#{config_js_content}}")
end

# 可配置项
defp plausible_main_config(config) do
  %{
    domain: config.domain,
    endpoint: config.custom_event_endpoint,
    hashBasedRouting: config.hash_mode,
    outboundLinks: config.outbound_link_tracking_enabled,
    fileDownloads: config.file_download_tracking_enabled,
    formSubmissions: config.form_submissions_tracking_enabled,
    taggedEvents: config.tagged_events_enabled,
    revenue: config.revenue_goals_enabled,
    autoCapturePageviews: config.automatic_pageview_tracking_enabled,
    customProperties: custom_properties_config(config),
    excludedDomains: config.excluded_domains,
    fileTypes: config.file_types,
    captureOnLocalhost: config.capture_on_localhost,
    logging: config.logging
  }
end
```

**前端配置加载** (`tracker/src/config.js:38-64`)

```javascript
export function init(options) {
  if (COMPILE_PLAUSIBLE_WEB) {
    // 这个占位符会被服务端动态替换
    config = '<%= @config_js %>'
    Object.assign(config, options, {
      // domain 不能被用户覆盖，必须使用服务端配置
      domain: config.domain
    })
  } else if (COMPILE_PLAUSIBLE_NPM) {
    // NPM 模式需要手动初始化
    if (!options || !options.domain) {
      throw new Error('plausible.init(): domain argument is required')
    }
    Object.assign(config, options)
    config.isInitialized = true
  } else {
    // Legacy 变体从 script 标签 data-* 属性读取
    config.endpoint = scriptEl.getAttribute('data-api') || defaultEndpoint()
    config.domain = scriptEl.getAttribute('data-domain')
    config.logging = true
  }
}
```

---

### 2.2 事件字段（重大修正）

#### 之前的误判

| 误判内容 | 实际情况 |
|----------|----------|
| `payload.m` 是自定义属性 | `payload.m` 仅在 **legacy 变体**中存在，用于旧版 `meta` 参数 |
| `payload.name` 是事件名 | 实际是 `payload.n`，缩写为单个字母节省字节 |
| 字段都是完整单词 | 所有字段都是**单个字母缩写**，最小化脚本体积 |

#### 正确的事件字段

**核心追踪逻辑** (`tracker/src/track.js:86-163`)

```javascript
export function track(eventName, options) {
  // ... 前置检查和过滤（localhost、机器人、exclusions 等）
  
  var payload = {}
  
  // 事件名称 (n = name)
  payload.n = eventName
  
  // 追踪脚本版本 (v = version)
  payload.v = COMPILE_TRACKER_SCRIPT_VERSION
  
  // 页面 URL (u = url)
  if (COMPILE_MANUAL) {
    var customURL = options && (options.u || options.url)
    payload.u = customURL ? customURL : location.href
  } else {
    payload.u = location.href
  }
  
  // 域名 (d = domain)
  payload.d = config.domain
  
  // 来源页 (r = referrer)
  payload.r = document.referrer || null
  
  // 旧版元数据 (m = meta) - 仅 legacy 变体支持
  if (COMPILE_PLAUSIBLE_LEGACY_VARIANT && options && options.meta) {
    payload.m = JSON.stringify(options.meta)
  }
  
  // 自定义属性 (p = props) - 新版推荐使用
  if (options && options.props) {
    payload.p = options.props
  }
  
  // 是否交互事件 (i = interactive)
  if (options && options.interactive === false) {
    payload.i = false
  }
  
  // 收入数据 ($ = revenue) - EE 版本
  if (COMPILE_REVENUE) {
    if (options && options.revenue) {
      payload.$ = options.revenue
    }
  }
  
  // 页面浏览脚本标签属性
  if (COMPILE_PAGEVIEW_PROPS) {
    var propAttributes = scriptEl.getAttributeNames().filter(function (name) {
      return name.substring(0, 6) === 'event-'
    })
    var props = payload.p || {}
    propAttributes.forEach(function (attribute) {
      var propKey = attribute.replace('event-', '')
      var propValue = scriptEl.getAttribute(attribute)
      props[propKey] = props[propKey] || propValue
    })
    payload.p = props
  }
  
  // 函数配置的自定义属性
  if (COMPILE_CUSTOM_PROPERTIES && config.customProperties) {
    var props = config.customProperties
    if (typeof props === 'function') {
      props = config.customProperties(eventName)
    }
    if (typeof props === 'object') {
      payload.p = Object.assign({}, props, payload.p)
    }
  }
  
  // Hash 路由模式标志 (h = hash)
  if (COMPILE_HASH && (!COMPILE_CONFIG || config.hashBasedRouting)) {
    payload.h = 1
  }
  
  // 允许用户修改请求
  if ((COMPILE_PLAUSIBLE_WEB || COMPILE_PLAUSIBLE_NPM) &&
      typeof config.transformRequest === 'function') {
    payload = config.transformRequest(payload)
    if (!payload) {
      return onIgnoredEvent(eventName, options, 'transformRequest')
    }
  }
  
  // 发送请求
  sendRequest(config.endpoint, payload, options)
}
```

#### 事件字段对照表

| 缩写字段 | 完整含义 | 来源 | 必填 |
|----------|----------|------|------|
| `n` | event name（事件名称） | 自动生成或手动传入 | 是 |
| `v` | tracker script version（追踪脚本版本） | 编译时注入 | 是 |
| `u` | URL（页面完整 URL） | `location.href` | 是 |
| `d` | domain（目标域名） | 站点配置 | 是 |
| `r` | referrer（来源页面） | `document.referrer` | 否 |
| `p` | props（自定义属性） | `options.props` 或脚本标签属性 | 否 |
| `i` | interactive（是否交互事件） | `options.interactive` 默认 `true` | 否 |
| `$` | revenue（收入数据） | `options.revenue`（EE 版本） | 否 |
| `h` | hash mode（hash 路由标志） | 配置决定 | 否 |
| `m` | meta（旧版元数据） | `options.meta`（仅 legacy 变体） | 否 |

#### 网络请求发送

**发送逻辑** (`tracker/src/networking.js`)

```javascript
export function sendRequest(endpoint, payload, options) {
  if (COMPILE_COMPAT) {
    // 兼容性模式：使用 XMLHttpRequest
    var request = new XMLHttpRequest()
    request.open('POST', endpoint, true)
    request.setRequestHeader('Content-Type', 'text/plain')  // 注意：text/plain 避免预检请求
    request.send(JSON.stringify(payload))
    // ... 回调处理
  } else {
    // 现代模式：使用 fetch
    if (window.fetch) {
      fetch(endpoint, {
        method: 'POST',
        headers: {
          'Content-Type': 'text/plain'  // 注意：text/plain 避免预检请求
        },
        keepalive: true,  // 允许页面卸载后继续发送
        body: JSON.stringify(payload)
      })
      // ... 回调处理
    }
  }
}
```

**关键点**：
1. 使用 `Content-Type: text/plain` 而非 `application/json`，**避免浏览器发出 CORS 预检请求**
2. `keepalive: true` 确保页面卸载时请求仍能发送
3. 现代浏览器使用 `fetch`，旧浏览器使用 `XMLHttpRequest`

---

## 三、服务端事件处理链路

### 3.1 请求接收与验证

**API 端点** (`lib/plausible_web/controllers/api/external_controller.ex:13-60`)

```elixir
def event(conn, _params) do
  with {:ok, request, conn} <- Ingestion.Request.build(conn),
       _ <- Sentry.Context.set_extra_context(%{request: request}) do
    case Ingestion.Event.build_and_buffer(request) do
      {:ok, %{dropped: [], buffered: _buffered}} ->
        conn
        |> put_status(202)       # 202 Accepted：请求已接受，但未处理完成
        |> text("ok")
      
      {:ok, %{dropped: dropped, buffered: _}} ->
        # 部分事件被丢弃，仍然返回 202，但在响应头中提示
        conn
        |> put_resp_header("x-plausible-dropped", "#{Enum.count(dropped)}")
        |> put_status(202)
        |> text("ok")
    end
  else
    {:error, %Ecto.Changeset{} = changeset} ->
      conn
      |> put_status(400)       # 400 Bad Request：请求参数无效
      |> json(%{errors: changeset_errors(changeset)})
  end
end
```

**请求构建** (`lib/plausible/ingestion/request.ex:76-124`)

```elixir
def build(%Plug.Conn{} = conn, now \\ NaiveDateTime.utc_now()) do
  changeset =
    %__MODULE__{}
    |> Changeset.change()
    |> Changeset.put_change(:timestamp, NaiveDateTime.truncate(now, :second))

  case parse_body(conn) do
    {:ok, request_body, conn} ->
      request =
        changeset
        |> put_ip_classification(conn)        # IP 类型分类（数据中心/威胁/匿名）
        |> put_remote_ip(conn)                # 提取真实客户端 IP（处理 X-Forwarded-For）
        |> put_uri(request_body)               # 解析 URL
        |> put_hostname()                      # 主机名
        |> put_user_agent(conn)                # User-Agent
        |> put_request_params(request_body)    # 解析请求体参数
        |> put_referrer(request_body)          # 来源页
        |> put_pathname()                      # 路径
        |> put_props(request_body)             # 自定义属性
        |> put_engagement_fields(request_body) # 参与度字段
        |> put_query_params()                  # URL 查询参数
        |> put_revenue_source(request_body)    # 收入数据（EE）
        |> put_interactive(request_body)       # 是否交互事件
        |> put_tracker_script_version(request_body) # 脚本版本
        |> map_domains(request_body)           # 域名列表
        |> Changeset.validate_required([
             :event_name, :hostname, :pathname, :timestamp
           ])
        |> Changeset.apply_action(nil)
      
      case request do
        {:ok, request} -> {:ok, request, conn}
        {:error, changeset} -> {:error, changeset}
      end
  end
end
```

**Request 结构体** (`lib/plausible/ingestion/request.ex:43-67`)

```elixir
embedded_schema do
  field :remote_ip, :string                    # 客户端 IP
  field :user_agent, :string                   # User-Agent 字符串
  field :event_name, Plausible.Ecto.EventName  # 事件名称（如 "pageview"、"engagement"）
  field :uri, :map                             # 解析后的 URL 组件
  field :hostname, :string                     # 主机名
  field :referrer, :string                     # 来源页面 URL
  field :domains, {:array, :string}            # 目标域名列表（用于跨域名追踪）
  field :ip_classification, :string            # IP 分类（数据中心/威胁/匿名等）
  field :hash_mode, :integer                   # Hash 路由模式标志
  field :pathname, :string                     # 路径
  field :props, :map                           # 自定义属性（从 payload.p 解析）
  field :scroll_depth, :integer                # 滚动深度 (0-100)
  field :engagement_time, :integer             # 参与时间（秒）
  field :tracker_script_version, :integer      # 追踪脚本版本
  field :interactive?, :boolean, default: true  # 是否为交互事件
  field :query_params, :map                    # URL 查询参数
  field :timestamp, :naive_datetime            # 事件时间戳
end
```

### 3.2 事件处理管道

**管道入口** (`lib/plausible/ingestion/event.ex:56-80`)

```elixir
def build_and_buffer(%Request{domains: domains} = request, context \\ []) do
  processed_events =
    if spam_referrer?(request) do
      # 垃圾来源直接丢弃
      for domain <- domains, do: drop(new(domain, request), :spam_referrer)
    else
      Enum.reduce(domains, [], fn domain, acc ->
        case GateKeeper.check(domain) do
          {:allow, site} ->
            # 管道处理
            processed =
              domain
              |> new(site, request)
              |> process_unless_dropped(pipeline(), context)
            
            [processed | acc]
          
          {:deny, reason} ->
            [drop(new(domain, request), reason) | acc]
        end
      end)
    end

  {dropped, buffered} = Enum.split_with(processed_events, & &1.dropped?)
  {:ok, %{dropped: dropped, buffered: buffered}}
end
```

**15 步处理管道** (`lib/plausible/ingestion/event.ex:130-151`)

```elixir
defp pipeline() do
  [
    # ========== 第一阶段：验证和过滤层 ==========
    drop_verification_agent: &drop_verification_agent/2,    # 丢弃验证机器人
    drop_datacenter_ip: &drop_datacenter_ip/2,              # 丢弃数据中心 IP
    drop_threat_ip: &drop_threat_ip/2,                      # 丢弃威胁 IP
    drop_shield_rule_hostname: &drop_shield_rule_hostname/2, # 主机名白名单
    drop_shield_rule_page: &drop_shield_rule_page/2,        # 页面黑名单
    drop_shield_rule_ip: &drop_shield_rule_ip/2,            # IP 黑名单
    
    # ========== 第二阶段：信息提取层 ==========
    put_geolocation: &put_geolocation/2,                     # 地理定位（国家、城市）
    drop_shield_rule_country: &drop_shield_rule_country/2,  # 国家黑名单（必须在定位后）
    put_user_agent: &put_user_agent/2,                       # User-Agent 解析
    put_basic_info: &put_basic_info/2,                       # 基础信息
    put_source_info: &put_source_info/2,                     # 来源信息（来源渠道）
    maybe_infer_medium: &maybe_infer_medium/2,               # 推断媒介
    put_props: &put_props/2,                                  # 自定义属性
    put_revenue: &put_revenue/2,                              # 收入数据
    put_salts: &put_salts/2,                                  # 获取盐值
    put_user_id: &put_user_id/2,                              # 生成用户 ID
    
    # ========== 第三阶段：验证和持久化层 ==========
    validate_clickhouse_event: &validate_clickhouse_event/2, # 验证事件结构
    register_session: &register_session/2                     # 注册会话并持久化
  ]
end
```

**事件丢弃原因** (`lib/plausible/ingestion/event.ex:24-41`)

```elixir
@type drop_reason() ::
        :bot                      # 机器人或无头浏览器
        | :spam_referrer          # 垃圾来源
        | :invalid                 # 验证失败
        | :dc_ip                   # 数据中心 IP
        | :threat_ip               # 威胁 IP
        | :site_ip_blocklist       # 站点 IP 黑名单
        | :site_country_blocklist  # 站点国家黑名单
        | :site_page_blocklist     # 站点页面黑名单
        | :site_hostname_allowlist # 站点主机名白名单
        | :verification_agent      # 验证代理
        | :lock_timeout            # 会话锁超时
        | :no_session_for_engagement  # 参与度事件无对应会话
        | :persist_timeout         # 持久化超时
        | :persist_error           # 持久化错误
        | :persist_decode_error    # 持久化解码错误
```

---

## 四、会话管理与缓冲写入

### 4.1 会话管理

**持久化入口** (`lib/plausible/ingestion/persistor/embedded.ex:10-41`)

```elixir
defmodule Plausible.Ingestion.Persistor.Embedded do
  def persist_event(ingest_event, previous_user_id, opts) do
    event = ingest_event.clickhouse_event
    session_attrs = ingest_event.clickhouse_session_attrs
    
    # 缓冲区插入函数（可在测试中 Mock）
    session_write_buffer_insert =
      Keyword.get(opts, :session_write_buffer_insert, 
                  &Plausible.Session.WriteBuffer.insert/1)
    
    event_write_buffer_insert =
      Keyword.get(opts, :event_write_buffer_insert, 
                  &Plausible.Event.WriteBuffer.insert/1)
    
    # 核心会话处理
    session_result =
      Plausible.Session.CacheStore.on_event(
        event,
        session_attrs,
        previous_user_id,
        buffer_insert: session_write_buffer_insert
      )
    
    case session_result do
      {:ok, :no_session_for_engagement} ->
        {:error, :no_session_for_engagement}
      
      {:error, :timeout} ->
        {:error, :lock_timeout}
      
      {:ok, session} ->
        # 合并会话属性到事件（事件表包含冗余的会话维度，避免 JOIN）
        event = ClickhouseEventV2.merge_session(event, session)
        {:ok, _} = event_write_buffer_insert.(event)
        
        {:ok, %{ingest_event | clickhouse_event: event}}
    end
  end
end
```

**会话缓存存储** (`lib/plausible/session/cache_store.ex`)

```elixir
def on_event(event, session_attributes, prev_user_id, opts \\ []) do
  buffer_insert = Keyword.get(opts, :buffer_insert, &WriteBuffer.insert/1)
  
  try do
    response =
      Plausible.Session.Balancer.dispatch(
        event.user_id,
        fn ->
          # 查找现有会话（当前盐值或前一天盐值，用于跨天延续）
          found_session =
            find_session(event, event.user_id) || 
            find_session(event, prev_user_id)
          
          handle_event(event, found_session, session_attributes, buffer_insert)
        end,
        timeout: @lock_timeout,      # 默认 1 秒
        local?: skip_balancer?
      )
    
    {:ok, response}
  catch
    :exit, {:timeout, _} ->
      {:error, :timeout}
  end
end
```

**会话处理逻辑** (`lib/plausible/session/cache_store.ex:44-65`)

```elixir
defp handle_event(%{name: "engagement"} = event, found_session, _, _) do
  if found_session do
    # 参与度事件：只更新缓存时间戳，不写入数据库
    refresh_session_cache(found_session, event.timestamp)
    found_session
  else
    :no_session_for_engagement
  end
end

defp handle_event(event, found_session, session_attributes, buffer_insert) do
  if found_session do
    # 更新现有会话：MVCC 式更新
    # - sign=-1 表示旧记录作废
    # - sign=1 表示新记录生效
    updated_session = update_session(found_session, event)
    buffer_insert.([
      %{found_session | sign: -1},   # 作废旧记录
      %{updated_session | sign: 1}   # 插入新记录
    ])
    update_session_cache(updated_session)
  else
    # 创建新会话
    new_session = new_session_from_event(event, session_attributes)
    buffer_insert.([new_session])
    update_session_cache(new_session)
  end
end
```

**会话超时判定** (`lib/plausible/session/cache_store.ex:69-81`)

```elixir
defp find_session(event, user_id) do
  from_cache = Plausible.Cache.Adapter.get(:sessions, {event.site_id, user_id})
  
  case from_cache do
    nil -> nil
    session ->
      # 会话超时：30 分钟无活动
      if NaiveDateTime.diff(event.timestamp, session.timestamp, :minute) <= 30 do
        session
      end
  end
end
```

### 4.2 写入缓冲机制

**通用缓冲区** (`lib/plausible/ingestion/write_buffer.ex`)

```elixir
defmodule Plausible.Ingestion.WriteBuffer do
  use GenServer
  
  def init(opts) do
    buffer = opts[:buffer] || []
    # 默认配置：从 Application 配置读取
    max_buffer_size = opts[:max_buffer_size] || default_max_buffer_size()
    flush_interval_ms = opts[:flush_interval_ms] || default_flush_interval_ms()
    
    Process.flag(:trap_exit, true)  # 捕获退出信号，确保缓冲被刷新
    timer = Process.send_after(self(), :tick, flush_interval_ms)
    
    {:ok,
     %{
       buffer: buffer,                          # 累积的 RowBinary 数据
       buffer_size: IO.iodata_length(buffer),   # 当前缓冲大小
       max_buffer_size: max_buffer_size,         # 最大缓冲大小（默认 100KB）
       flush_interval_ms: flush_interval_ms,     # 刷新间隔（默认 5 秒）
       timer: timer,
       name: Keyword.fetch!(opts, :name),
       insert_sql: Keyword.fetch!(opts, :insert_sql),
       insert_opts: Keyword.fetch!(opts, :insert_opts),
       header: Keyword.fetch!(opts, :header)
     }}
  end
  
  # 默认配置来源
  defp default_flush_interval_ms do
    Keyword.fetch!(Application.get_env(:plausible, IngestRepo), :flush_interval_ms)
  end
  
  defp default_max_buffer_size do
    Keyword.fetch!(Application.get_env(:plausible, IngestRepo), :max_buffer_size)
  end
end
```

**刷新触发条件**

```elixir
# 条件 1：缓冲大小达到阈值
def handle_cast({:insert, row_binary}, state) do
  state = %{
    state
    | buffer: [state.buffer | row_binary],
      buffer_size: state.buffer_size + IO.iodata_length(row_binary)
  }
  
  if state.buffer_size >= state.max_buffer_size do
    Logger.notice("#{state.name} buffer full, flushing to ClickHouse")
    Process.cancel_timer(state.timer)
    do_flush(state)
    new_timer = Process.send_after(self(), :tick, state.flush_interval_ms)
    {:noreply, %{state | buffer: [], timer: new_timer, buffer_size: 0}}
  else
    {:noreply, state}
  end
end

# 条件 2：定时刷新
def handle_info(:tick, state) do
  do_flush(state)
  timer = Process.send_after(self(), :tick, state.flush_interval_ms)
  {:noreply, %{state | buffer: [], buffer_size: 0, timer: timer}}
end

# 条件 3：进程终止
def terminate(_reason, %{name: name} = state) do
  Logger.notice("Flushing #{name} buffer before shutdown...")
  do_flush(state)
end
```

**实际写入 ClickHouse**

```elixir
defp do_flush(state) do
  %{
    buffer: buffer,
    buffer_size: buffer_size,
    insert_opts: insert_opts,
    insert_sql: insert_sql,
    header: header,
    name: name
  } = state
  
  case buffer do
    [] ->
      nil  # 空缓冲，不执行操作
    
    _not_empty ->
      Logger.notice("Flushing #{buffer_size} byte(s) RowBinary from #{name}")
      # 使用 RowBinaryWithNamesAndTypes 格式批量插入
      IngestRepo.query!(insert_sql, [header | buffer], insert_opts)
  end
end
```

### 4.3 运行时配置

**配置定义** (`config/runtime.exs:132-146`)

```elixir
# 刷新间隔：默认 5000 毫秒 (5秒)
{ch_flush_interval_ms, ""} =
  config_dir
  |> get_var_from_path_or_env("CLICKHOUSE_FLUSH_INTERVAL_MS", "5000")
  |> Integer.parse()

# 最大缓冲大小：默认 100,000 字节 (100KB)
# 注意：旧变量 CLICKHOUSE_MAX_BUFFER_SIZE 已弃用
if get_var_from_path_or_env(config_dir, "CLICKHOUSE_MAX_BUFFER_SIZE") do
  Logger.warning(
    "CLICKHOUSE_MAX_BUFFER_SIZE is deprecated, 
     please use CLICKHOUSE_MAX_BUFFER_SIZE_BYTES instead"
  )
end

{ch_max_buffer_size, ""} =
  config_dir
  |> get_var_from_path_or_env("CLICKHOUSE_MAX_BUFFER_SIZE_BYTES", "100000")
  |> Integer.parse()
```

**配置应用** (`config/runtime.exs:649-650`)

```elixir
config :plausible, Plausible.IngestRepo,
  flush_interval_ms: ch_flush_interval_ms,
  max_buffer_size: ch_max_buffer_size,
  # ... 其他配置
```

---

## 五、缓冲刷新对看板时效性的影响

### 5.1 数据延迟分析

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      数据延迟时间线                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  用户事件发生时刻                                                            │
│       │                                                                      │
│       │ 0-2 秒（网络传输 + 服务端处理）                                      │
│       ▼                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  事件进入写入缓冲区                                                     │   │
│  │  (lib/plausible/event/write_buffer.ex                                 │   │
│  │   lib/plausible/session/write_buffer.ex)                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                      │
│       │ 等待刷新                                                             │
│       │                                                                      │
│       ├──── 最快情况：缓冲立即满（> 100KB）→ 立即刷新                       │
│       │                                                                      │
│       ├──── 最慢情况：缓冲不满 → 等待 5 秒定时刷新                          │
│       │                                                                      │
│       ▼                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  数据写入 ClickHouse                                                   │   │
│  │  INSERT INTO events_v2 FORMAT RowBinaryWithNamesAndTypes             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                      │
│       │ 0-1 秒（ClickHouse 插入 + 后台 Merge）                              │
│       ▼                                                                      │
│  数据对查询可见（绝大多数情况下立即可见）                                      │
│                                                                              │
│                                                                              │
│  【总延迟】                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  最小延迟：约 0.5 秒（高流量场景，缓冲快速填满）                         │   │
│  │  最大延迟：约 6 秒（低流量场景，等待 5 秒定时刷新 + 网络/处理时间）       │   │
│  │  典型延迟：约 2-5 秒                                                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 实时访客的特殊处理

**实时访客不经过缓冲** (`lib/plausible/stats/current_visitors.ex`)

```elixir
defmodule Plausible.Stats.CurrentVisitors do
  use Plausible.ClickhouseRepo
  use Plausible.Stats.SQL.Fragments

  @spec current_visitors(Plausible.Site.t(), Duration.duration()) :: non_neg_integer
  def current_visitors(site, duration \\ Duration.new!(minute: -5)) do
    first_datetime =
      NaiveDateTime.utc_now()
      |> NaiveDateTime.shift(duration)
      |> NaiveDateTime.truncate(:second)

    ClickhouseRepo.one(
      from e in "events_v2",
        where: ^Plausible.Sites.site_id_query_filter(site),
        where: e.timestamp >= ^first_datetime,  # 过去 5 分钟
        where: e.name != "engagement",           # 排除参与度事件
        select: uniq(e.user_id)
    )
  end
end
```

**关键点**：
1. **直接查询 ClickHouse**：不经过任何应用层缓存
2. **时间窗口**：过去 **5 分钟** 的数据
3. **与缓冲刷新的关系**：
   - 如果数据还在缓冲区（未刷新到 ClickHouse），**不会被计入**
   - 实时访客数字也会有 **5-10 秒** 的延迟

### 5.3 时效性对比

| 数据类型 | 延迟来源 | 典型延迟 | 影响因素 |
|----------|----------|----------|----------|
| **页面浏览、会话等** | 写入缓冲区 | 2-5 秒 | 流量大小、缓冲配置 |
| **实时访客** | 写入缓冲区 + 5分钟窗口 | 5-10 秒 | 同上 |
| **跳出率、转化率等** | 写入缓冲区 + 会话属性合并 | 3-8 秒 | 同上，且需要会话更新 |
| **滚动深度、参与时间** | 写入缓冲区 + 特殊计算 | 3-10 秒 | 需要计算平均滚动深度等 |

### 5.4 配置调优建议

**高流量场景（需要更低延迟）**：
```bash
# 缩短刷新间隔到 1 秒
CLICKHOUSE_FLUSH_INTERVAL_MS=1000

# 减小缓冲大小到 50KB（更频繁刷新）
CLICKHOUSE_MAX_BUFFER_SIZE_BYTES=50000
```

**低流量场景（需要更高吞吐量）**：
```bash
# 延长刷新间隔到 30 秒
CLICKHOUSE_FLUSH_INTERVAL_MS=30000

# 增大缓冲大小到 1MB
CLICKHOUSE_MAX_BUFFER_SIZE_BYTES=1000000
```

---

## 六、指标口径映射

### 6.1 完整指标体系

**后端指标定义** (`lib/plausible/stats/metrics.ex`)

```elixir
@all_metrics [
  # ========== 原子指标（直接从表计数） ==========
  :visitors,           # 独立访客数（uniq(user_id)）
  :visits,             # 会话数（uniq(session_id)）
  :pageviews,          # 页面浏览数（count(event) where name = 'pageview'）
  :events,             # 总事件数（count(event)）
  
  # ========== 特殊指标（需要额外计算） ==========
  :bounce_rate,        # 跳出率
  :visit_duration,     # 访问时长
  :views_per_visit,    # 每次访问页面数
  :exit_rate,          # 退出率
  :time_on_page,       # 页面停留时间
  :scroll_depth,       # 滚动深度
  :percentage,         # 百分比（用于细分列表）
  :conversion_rate,    # 转化率
  :group_conversion_rate, # 分组转化率
  
  # ========== 收入指标（EE 版本） ==========
  :total_revenue,      # 总收入
  :average_revenue,    # 平均收入
] ++ @revenue_metrics
```

### 6.2 特殊指标计算

**跳出率、退出率、转化率等** (`lib/plausible/stats/sql/special_metrics.ex`)

```elixir
defmodule Plausible.Stats.SQL.SpecialMetrics do
  @special_metrics [
    :percentage,
    :conversion_rate,
    :group_conversion_rate,
    :scroll_depth,
    :exit_rate
  ]
  
  # ========== 转化率计算 ==========
  # 计算公式：(目标访客数 / 总访客数) * 100
  defp add_special_metric(q, :conversion_rate, site, query) do
    # 总访客数查询（移除目标和属性过滤）
    total_query =
      query
      |> Query.remove_top_level_filters(["event:goal", "event:props"])
      |> Query.set(dimensions: [], ...)
    
    q
    |> select_merge_as([], total_visitors_subquery(site, total_query, ...))
    |> select_merge_as([e], %{
      conversion_rate:
        fragment(
          "if(? > 0, round(? / ? * 100, 2), 0)",
          selected_as(:total_visitors),      # 分母：总访客数
          selected_as(:visitors),             # 分子：目标访客数
          selected_as(:total_visitors)
        )
    })
  end
  
  # ========== 滚动深度计算 ==========
  # 计算公式：sum(每个会话最大滚动深度) / 参与会话数
  defp add_special_metric(q, :scroll_depth, _site, query) do
    # 第一步：每个会话的最大滚动深度
    max_per_session_q =
      Base.base_event_query(query)
      |> where([e], e.name == "engagement" and e.scroll_depth <= 100)
      |> select([e], %{
        session_id: e.session_id,
        max_scroll_depth: max(e.scroll_depth)
      })
      |> group_by([e], e.session_id)
    
    # 第二步：计算平均滚动深度
    total_scroll_depth_q =
      subquery(max_per_session_q)
      |> select([], %{
        total_scroll_depth: fragment("sum(?)", p.max_scroll_depth),
        total_scroll_depth_visits: fragment("uniq(?)", p.session_id)
      })
    
    # 最终：round(total_scroll_depth / total_scroll_depth_visits)
    # ...
  end
  
  # ========== 退出率计算 ==========
  # 计算公式：(作为退出页的会话数 / 页面浏览数) * 100
  defp add_special_metric(q, :exit_rate, site, query) do
    # 总页面浏览查询
    total_pageviews_query =
      query
      |> Query.remove_top_level_filters(["visit:exit_page"])
      |> Query.set(metrics: [:pageviews], dimensions: ["event:page"])
    
    q
    |> join(:left, [], p in subquery(SQL.QueryBuilder.build(total_pageviews_query, site)),
      on: selected_as(^shortname(query, "visit:exit_page")) == 
           field(p, ^shortname(total_pageviews_query, "event:page"))
    )
    |> select_merge_as([..., p], %{
      exit_rate:
        fragment(
          "if(? > 0, round(? / ? * 100, 1), NULL)",
          fragment("any(?)", p.pageviews),      # 分母：该页面总浏览数
          selected_as(:__internal_visits),      # 分子：作为退出页的会话数
          fragment("any(?)", p.pageviews)
        )
    })
  end
end
```

### 6.3 前端指标选择逻辑

**动态选择指标** (`assets/js/dashboard/stats/graph/fetch-top-stats.ts:77-112`)

```typescript
export function chooseMetrics(
  site: Pick<PlausibleSite, 'revenueGoals'>,
  dashboardState: DashboardState
): Metric[] {
  const revenueMetrics: Metric[] =
    site.revenueGoals.length > 0 ? ['total_revenue', 'average_revenue'] : []

  // ========== 实时面板（30分钟窗口） ==========
  if (
    isRealTimeDashboard(dashboardState) &&
    hasConversionGoalFilter(dashboardState)
  ) {
    // 实时 + 转化目标过滤
    return ['visitors', 'events']
  } else if (isRealTimeDashboard(dashboardState)) {
    // 实时面板
    return ['visitors', 'pageviews']
  } 
  
  // ========== 有转化目标过滤 ==========
  else if (hasConversionGoalFilter(dashboardState)) {
    return ['visitors', 'events', ...revenueMetrics, 'conversion_rate']
  } 
  
  // ========== 有页面过滤 ==========
  else if (hasPageFilter(dashboardState)) {
    return [
      'visitors',
      'visits',
      'pageviews',
      'bounce_rate',
      'scroll_depth',
      'time_on_page'    // 注意：页面过滤时显示 time_on_page 而非 visit_duration
    ]
  } 
  
  // ========== 默认情况（无特殊过滤） ==========
  else {
    return [
      'visitors',
      'visits',
      'pageviews',
      'views_per_visit',
      'bounce_rate',
      'visit_duration'  // 默认显示 visit_duration
    ]
  }
}
```

### 6.4 指标标签映射

**前端显示标签** (`assets/js/dashboard/stats/metrics.ts`)

```typescript
export const getMetricLabel = (
  metric: Metric,
  { hasConversionGoalFilter }: { hasConversionGoalFilter: boolean }
): string => {
  switch (metric) {
    case 'visitors':
      // 有转化目标过滤时显示为"Unique conversions"
      return hasConversionGoalFilter ? 'Unique conversions' : 'Unique visitors'
    
    case 'events':
      // 有转化目标过滤时显示为"Total conversions"
      return hasConversionGoalFilter ? 'Total conversions' : 'Total events'
    
    case 'visits':
      return 'Total visits'
    
    case 'pageviews':
      return 'Total pageviews'
    
    case 'views_per_visit':
      return 'Views per visit'
    
    case 'bounce_rate':
      return 'Bounce rate'
    
    case 'visit_duration':
      return 'Visit duration'
    
    case 'time_on_page':
      return 'Time on page'
    
    case 'scroll_depth':
      return 'Scroll depth'
    
    case 'conversion_rate':
      return 'Conversion rate'
    
    case 'total_revenue':
      return 'Total revenue'
    
    case 'average_revenue':
      return 'Average revenue'
    
    case 'percentage':
      return 'Percentage'
    
    case 'group_conversion_rate':
      return 'Conversion rate'
  }
}
```

### 6.5 指标口径对照表

| 后端指标名 | 前端显示名 | 计算公式 | 数据源 |
|------------|------------|----------|--------|
| `visitors` | Unique visitors / Unique conversions | `uniq(user_id)` | events_v2 |
| `visits` | Total visits | `uniq(session_id)` | events_v2 / sessions_v2 |
| `pageviews` | Total pageviews | `count(event where name='pageview')` | events_v2 |
| `events` | Total events / Total conversions | `count(event)` | events_v2 |
| `views_per_visit` | Views per visit | `pageviews / visits` | 衍生 |
| `bounce_rate` | Bounce rate | `bounces / visits * 100` | sessions_v2 |
| `visit_duration` | Visit duration | `avg(session.duration)` | sessions_v2 |
| `time_on_page` | Time on page | 复杂计算（需要前后页面浏览时间差） | events_v2 |
| `scroll_depth` | Scroll depth | `avg(max(scroll_depth per session))` | events_v2 (engagement) |
| `exit_rate` | Exit rate | `exit_sessions / pageviews * 100` | sessions_v2 / events_v2 |
| `conversion_rate` | Conversion rate | `goal_visitors / total_visitors * 100` | 衍生 |
| `percentage` | Percentage | `visitors / total_visitors * 100` | 衍生 |

---

## 七、完整数据流图（修正版）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│                      数据采集阶段                                              │
│                                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  用户网站                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                         │   │
│  │  方式 1: Web Snippet（推荐）                                           │   │
│  │  ┌─────────────────────────────────────────────────────────────────┐ │   │
│  │  │ <script defer                                                  │ │   │
│  │  │   data-domain="example.com"                                   │ │   │
│  │  │   src="https://plausible.io/js/script.js">                   │ │   │
│  │  │ </script>                                                      │ │   │
│  │  └─────────────────────────────────────────────────────────────────┘ │   │
│  │       │                                                                 │   │
│  │       ▼                                                                 │   │
│  │  加载 plausible-web.js（新版 Web snippet）                              │   │
│  │  - 包含所有功能（hash, outbound-links, file-downloads 等）              │   │
│  │  - 服务端动态注入站点配置                                                │   │
│  │                                                                         │   │
│  │  方式 2: Legacy Variants（兼容旧集成）                                  │   │
│  │  ┌─────────────────────────────────────────────────────────────────┐ │   │
│  │  │ <script defer                                                  │ │   │
│  │  │   data-domain="example.com"                                   │ │   │
│  │  │   src="/js/plausible.hash.outbound-links.js">                │ │   │
│  │  │ </script>                                                      │ │   │
│  │  └─────────────────────────────────────────────────────────────────┘ │   │
│  │       │                                                                 │   │
│  │       ▼                                                                 │   │
│  │  加载对应变体（文件名后缀决定功能开关）                                   │   │
│  │  - 从 data-* 属性读取配置                                              │   │
│  │                                                                         │   │
│  │  方式 3: NPM 包                                                         │   │
│  │  ┌─────────────────────────────────────────────────────────────────┐ │   │
│  │  │ import Plausible from 'plausible-tracker'                      │ │   │
│  │  │                                                                  │ │   │
│  │  │ Plausible.init({                                                │ │   │
│  │  │   domain: 'example.com',                                        │ │   │
│  │  │   trackLocalhost: true                                          │ │   │
│  │  │ })                                                               │ │   │
│  │  └─────────────────────────────────────────────────────────────────┘ │   │
│  │                                                                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                      │
│       │ 事件触发                                                             │
│       │ - 页面加载 → pageview 事件                                          │
│       │ - 滚动 → engagement 事件（滚动深度）                                │
│       │ - 停留 → engagement 事件（参与时间）                                │
│       │ - 点击出站链接 → Outbound Link: 事件                                │
│       │ - 文件下载 → File Download: 事件                                    │
│       │ - 手动调用 plausible('CustomEvent') → 自定义事件                   │
│       ▼                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  track() 函数构建 payload                                              │   │
│  │                                                                         │   │
│  │  var payload = {                                                       │   │
│  │    n: 'pageview',         // 事件名称                                  │   │
│  │    v: 123,               // 脚本版本                                   │   │
│  │    u: location.href,     // 完整 URL                                   │   │
│  │    d: 'example.com',     // 域名                                       │   │
│  │    r: document.referrer, // 来源页                                     │   │
│  │    p: {                  // 自定义属性                                   │   │
│  │      auth: 'logged_in',                                                 │   │
│  │      theme: 'dark'                                                       │   │
│  │    }                                                                     │   │
│  │  }                                                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                      │
│       │ fetch(endpoint, {                                                   │
│       │   method: 'POST',                                                   │
│       │   headers: { 'Content-Type': 'text/plain' },  // 避免 CORS 预检    │
│       │   keepalive: true,                      // 页面卸载后仍能发送        │
│       │   body: JSON.stringify(payload)                                     │
│       │ })                                                                   │
│       ▼                                                                      │
│  POST https://plausible.io/api/event                                         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│                      服务端处理阶段                                            │
│                                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Phoenix Router                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  scope "/api", PlausibleWeb do                                       │   │
│  │    pipe_through :external_api                                         │   │
│  │    post "/event", Api.ExternalController, :event                     │   │
│  │  end                                                                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                      │
│       ▼                                                                      │
│  ExternalController.event/2                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  1. 构建 Request: Ingestion.Request.build(conn)                      │   │
│  │     - 解析 JSON payload                                               │   │
│  │     - 提取 IP、User-Agent                                             │   │
│  │     - 验证必需字段                                                     │   │
│  │                                                                         │   │
│  │  2. 处理事件: Ingestion.Event.build_and_buffer(request)              │   │
│  │     - 15 步管道处理                                                    │   │
│  │                                                                         │   │
│  │  3. 返回响应                                                           │   │
│  │     - 成功: 202 Accepted + 响应头 x-plausible-dropped                 │   │
│  │     - 失败: 400 Bad Request + 错误详情                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                      │
│       ▼                                                                      │
│  15 步事件处理管道                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                         │   │
│  │  阶段 1: 验证和过滤                                                    │   │
│  │  ┌─────────────────────────────────────────────────────────────────┐ │   │
│  │  │ 1. drop_verification_agent → 丢弃验证机器人                      │ │   │
│  │  │ 2. drop_datacenter_ip → 丢弃数据中心 IP                          │ │   │
│  │  │ 3. drop_threat_ip → 丢弃威胁 IP                                  │ │   │
│  │  │ 4. drop_shield_rule_hostname → 主机名白名单                      │ │   │
│  │  │ 5. drop_shield_rule_page → 页面黑名单                            │ │   │
│  │  │ 6. drop_shield_rule_ip → IP 黑名单                                │ │   │
│  │  └─────────────────────────────────────────────────────────────────┘ │   │
│  │                                                                         │   │
│  │  阶段 2: 信息提取                                                      │   │
│  │  ┌─────────────────────────────────────────────────────────────────┐ │   │
│  │  │ 7. put_geolocation → 地理定位（国家、城市）                        │ │   │
│  │  │ 8. drop_shield_rule_country → 国家黑名单                          │ │   │
│  │  │ 9. put_user_agent → User-Agent 解析                               │ │   │
│  │  │ 10. put_basic_info → 基础信息                                      │ │   │
│  │  │ 11. put_source_info → 来源信息                                     │ │   │
│  │  │ 12. put_props / put_revenue → 自定义属性、收入                      │ │   │
│  │  │ 13. put_salts / put_user_id → 盐值、用户 ID                        │ │   │
│  │  └─────────────────────────────────────────────────────────────────┘ │   │
│  │                                                                         │   │
│  │  阶段 3: 持久化                                                        │   │
│  │  ┌─────────────────────────────────────────────────────────────────┐ │   │
│  │  │ 14. validate_clickhouse_event → 验证事件结构                      │ │   │
│  │  │ 15. register_session → 注册会话并写入缓冲区                       │ │   │
│  │  └─────────────────────────────────────────────────────────────────┘ │   │
│  │                                                                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                      │
│       ▼                                                                      │
│  会话管理 (Session.CacheStore.on_event/4)                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                         │   │
│  │  1. 分布式锁: Session.Balancer.dispatch(user_id, fn -> ... end)      │   │
│  │     - 同一用户的事件串行处理                                           │   │
│  │     - 避免并发更新导致不一致                                            │   │
│  │                                                                         │   │
│  │  2. 查找会话: CacheAdapter.get(:sessions, {site_id, user_id})         │   │
│  │     - 检查 30 分钟超时                                                 │   │
│  │     - 支持前一天盐值（跨天延续）                                        │   │
│  │                                                                         │   │
│  │  3. 处理事件:                                                          │   │
│  │     ├── engagement 事件: 只更新缓存（不写入数据库）                    │   │
│  │     ├── 新会话: 创建新记录，sign=1                                     │   │
│  │     └── 旧会话: 先 sign=-1 作废旧记录，再 sign=1 插入新记录           │   │
│  │                                                                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                      │
│       ▼                                                                      │
│  写入缓冲区 (WriteBuffer.insert/1)                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                         │   │
│  │  两个独立缓冲区:                                                        │   │
│  │  - Plausible.Event.WriteBuffer  → 事件表缓冲区                        │   │
│  │  - Plausible.Session.WriteBuffer  → 会话表缓冲区                      │   │
│  │                                                                         │   │
│  │  默认配置:                                                              │   │
│  │  - CLICKHOUSE_FLUSH_INTERVAL_MS = 5000  (5秒)                         │   │
│  │  - CLICKHOUSE_MAX_BUFFER_SIZE_BYTES = 100000  (100KB)                │   │
│  │                                                                         │   │
│  │  刷新触发条件:                                                          │   │
│  │  ┌─────────────────────────────────────────────────────────────────┐ │   │
│  │  │ 条件 1: buffer_size >= max_buffer_size (100KB)                   │ │   │
│  │  │ 条件 2: 定时 tick (每 5 秒)                                       │ │   │
│  │  │ 条件 3: 进程终止 (terminate callback)                             │ │   │
│  │  └─────────────────────────────────────────────────────────────────┘ │   │
│  │                                                                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                      │
│       ▼                                                                      │
│  ClickHouse 表                                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                         │   │
│  │  events_v2 (事件表，事实表):                                           │   │
│  │  - 存储所有事件（pageview, engagement, 自定义事件等）                  │   │
│  │  - 包含冗余的会话属性（避免 JOIN 查询）                                 │   │
│  │                                                                         │   │
│  │  sessions_v2 (会话表，维度表):                                         │   │
│  │  - 使用 VersionedCollapsingMergeTree 引擎                             │   │
│  │  - sign=1 表示有效记录                                                 │   │
│  │  - sign=-1 表示作废记录                                                │   │
│  │  - 聚合时 sum(sign) 得到最终状态                                       │   │
│  │                                                                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ (查询方向)
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│                      数据查询与展示阶段                                        │
│                                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  前端仪表盘                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                         │   │
│  │  Top Stats 查询:                                                        │   │
│  │  - chooseMetrics() 根据过滤条件动态选择指标                             │   │
│  │  - 默认: ['visitors', 'visits', 'pageviews', 'views_per_visit',       │   │
│  │           'bounce_rate', 'visit_duration']                             │   │
│  │                                                                         │   │
│  │  API 调用:                                                              │   │
│  │  POST /api/stats/:domain/query                                          │   │
│  │  Body: { metrics: [...], filters: [...], date_range: ... }            │   │
│  │                                                                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                      │
│       ▼                                                                      │
│  StatsController.query/2                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                         │   │
│  │  1. 解析参数: ApiQueryParser.parse(params)                            │   │
│  │  2. 构建查询: QueryBuilder.build(site, parsed_params)                 │   │
│  │  3. 执行查询: Plausible.Stats.query(site, query)                     │   │
│  │  4. 返回 JSON 响应                                                     │   │
│  │                                                                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                      │
│       ▼                                                                      │
│  QueryRunner.run/2                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                         │   │
│  │  执行流程:                                                              │   │
│  │  1. 优化查询: QueryOptimizer.optimize(query)                          │   │
│  │  2. 执行主查询                                                          │   │
│  │  3. 执行对比查询（如果需要）                                             │   │
│  │  4. 构建结果列表                                                        │   │
│  │  5. 转换为 QueryResult                                                  │   │
│  │                                                                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                      │
│       ▼                                                                      │
│  SQL.QueryBuilder.build/2                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                         │   │
│  │  生成 ClickHouse SQL:                                                   │   │
│  │                                                                         │   │
│  │  原子指标（直接计数）:                                                  │   │
│  │  - visitors: uniq(user_id)                                             │   │
│  │  - visits: uniq(session_id)                                            │   │
│  │  - pageviews: count(event) where name = 'pageview'                    │   │
│  │                                                                         │   │
│  │  特殊指标（需要额外计算）:                                               │   │
│  │  - conversion_rate: goal_visitors / total_visitors * 100             │   │
│  │  - scroll_depth: avg(max(scroll_depth per session))                   │   │
│  │  - exit_rate: exit_sessions / pageviews * 100                         │   │
│  │                                                                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 八、关键修正总结

### 8.1 追踪脚本主路径（重大修正）

| 之前的误判 | 正确情况 |
|------------|----------|
| `tracker.ex` 生成追踪脚本 | `tracker.ex` 只负责**动态配置注入**，不生成脚本代码 |
| `p.js` 是主脚本 | `p.js` 是**编译产物**，真正的源码在 `tracker/src/` |
| 脚本路径不明确 | 主入口: `tracker/src/plausible.js` |

**正确的架构**：
```
源代码目录: tracker/src/
├── plausible.js      ← 主入口，初始化所有模块
├── track.js          ← 核心追踪逻辑，构建 payload
├── networking.js     ← 发送 POST 请求
├── config.js         ← 配置管理
├── autocapture.js    ← 自动捕获页面浏览
├── engagement.js     ← 参与度追踪
├── custom-events.js  ← 自定义事件
└── revenue.js        ← 收入追踪（EE）

编译配置: tracker/compiler/variants.json
├── manualVariants (主要变体)
│   ├── plausible-web.js  → 新版 Web snippet（推荐）
│   └── npm_package/      → NPM 包
└── legacyVariants (旧版变体，100+ 种组合)
    └── plausible.*.js    → 通过文件名后缀控制功能

编译产物: priv/tracker/js/
└── p.js  ← 压缩后的脚本（实际提供给用户）
```

### 8.2 事件字段（重大修正）

| 之前的误判 | 正确情况 |
|------------|----------|
| `payload.m` 是自定义属性 | `payload.m` 仅在 **legacy 变体**中存在 |
| 字段都是完整单词 | 所有字段都是**单个字母缩写**，最小化体积 |
| `payload.name` 是事件名 | 实际是 `payload.n` |

**正确的事件字段**：

| 缩写字段 | 完整含义 | 来源 | 必填 |
|----------|----------|------|------|
| `n` | event name | 自动生成或手动传入 | 是 |
| `v` | tracker script version | 编译时注入 | 是 |
| `u` | URL | `location.href` | 是 |
| `d` | domain | 站点配置 | 是 |
| `r` | referrer | `document.referrer` | 否 |
| `p` | props | `options.props` | 否 |
| `i` | interactive | `options.interactive` 默认 `true` | 否 |
| `$` | revenue | `options.revenue`（EE） | 否 |
| `h` | hash mode | 配置决定 | 否 |
| `m` | meta（旧版） | `options.meta`（仅 legacy） | 否 |

### 8.3 指标口径映射

**后端指标** (`lib/plausible/stats/metrics.ex`)：
```elixir
@all_metrics [
  # 原子指标
  :visitors, :visits, :pageviews, :events,
  
  # 特殊指标
  :bounce_rate, :visit_duration, :views_per_visit, 
  :exit_rate, :time_on_page, :scroll_depth,
  :percentage, :conversion_rate, :group_conversion_rate,
  
  # 收入指标（EE）
  :total_revenue, :average_revenue
]
```

**前端动态选择** (`assets/js/dashboard/stats/graph/fetch-top-stats.ts`)：
```typescript
// 默认情况
['visitors', 'visits', 'pageviews', 'views_per_visit', 'bounce_rate', 'visit_duration']

// 实时面板（30分钟）
['visitors', 'pageviews']

// 有页面过滤
['visitors', 'visits', 'pageviews', 'bounce_rate', 'scroll_depth', 'time_on_page']

// 有转化目标过滤
['visitors', 'events', 'total_revenue', 'average_revenue', 'conversion_rate']
```

### 8.4 缓冲刷新对时效性的影响

**默认配置**：
- `CLICKHOUSE_FLUSH_INTERVAL_MS` = **5000ms (5秒)**
- `CLICKHOUSE_MAX_BUFFER_SIZE_BYTES` = **100,000 bytes (100KB)**

**刷新触发条件**：
1. 缓冲大小 >= 100KB → 立即刷新
2. 定时 tick → 每 5 秒刷新
3. 进程终止 → 刷新剩余缓冲

**数据延迟**：

| 场景 | 最小延迟 | 最大延迟 | 典型延迟 |
|------|----------|----------|----------|
| 高流量（缓冲快速填满） | ~0.5秒 | ~2秒 | ~1秒 |
| 中等流量 | ~1秒 | ~6秒 | ~2-3秒 |
| 低流量（等待定时刷新） | ~5秒 | ~7秒 | ~5-6秒 |

**实时访客的特殊处理**：
- `CurrentVisitors.current_visitors/2` 直接查询 ClickHouse
- 时间窗口：过去 **5 分钟**
- 如果数据还在缓冲区，**不会被计入**
- 实时访客数字也会有 **5-10 秒** 的延迟

**配置调优建议**：

```bash
# 高流量场景（需要更低延迟）
CLICKHOUSE_FLUSH_INTERVAL_MS=1000
CLICKHOUSE_MAX_BUFFER_SIZE_BYTES=50000

# 低流量场景（需要更高吞吐量）
CLICKHOUSE_FLUSH_INTERVAL_MS=30000
CLICKHOUSE_MAX_BUFFER_SIZE_BYTES=1000000
```

---

## 九、关键文件索引（修正版）

| 层级 | 模块 | 文件路径 | 说明 |
|------|------|----------|------|
| **前端追踪（源码）** | 主入口 | `tracker/src/plausible.js` | 初始化所有模块 |
| | 核心追踪 | `tracker/src/track.js` | 构建 payload，发送请求 |
| | 网络请求 | `tracker/src/networking.js` | `fetch` / `XMLHttpRequest` |
| | 配置管理 | `tracker/src/config.js` | 配置加载和管理 |
| | 编译配置 | `tracker/compiler/variants.json` | 变体定义和全局变量 |
| | 编译产物 | `priv/tracker/js/p.js` | 压缩后的脚本 |
| **服务端配置注入** | 脚本服务 | `lib/plausible_web/tracker.ex` | 动态配置注入 |
| | 脚本路由 | `lib/plausible_web/plugs/tracker_plug.ex` | 脚本端点路由 |
| **服务端处理** | 请求接收 | `lib/plausible_web/controllers/api/external_controller.ex` | `/api/event` 端点 |
| | 请求构建 | `lib/plausible/ingestion/request.ex` | 构建 `Request` 结构体 |
| | 事件处理 | `lib/plausible/ingestion/event.ex` | 15 步管道处理 |
| | 会话管理 | `lib/plausible/session/cache_store.ex` | 会话缓存管理 |
| **缓冲写入** | 通用缓冲 | `lib/plausible/ingestion/write_buffer.ex` | GenServer 缓冲实现 |
| | 事件缓冲 | `lib/plausible/event/write_buffer.ex` | `events_v2` 缓冲 |
| | 会话缓冲 | `lib/plausible/session/write_buffer.ex` | `sessions_v2` 缓冲 |
| **指标查询** | 指标定义 | `lib/plausible/stats/metrics.ex` | 所有可用指标 |
| | 特殊指标计算 | `lib/plausible/stats/sql/special_metrics.ex` | 转化率、滚动深度等 |
| | 查询执行 | `lib/plausible/stats/query_runner.ex` | 执行查询 |
| | SQL 生成 | `lib/plausible/stats/sql/query_builder.ex` | 生成 ClickHouse SQL |
| | 实时访客 | `lib/plausible/stats/current_visitors.ex` | 查询过去 5 分钟数据 |
| **前端仪表盘** | API 调用 | `assets/js/dashboard/api.ts` | 后端 API 调用 |
| | 指标选择 | `assets/js/dashboard/stats/graph/fetch-top-stats.ts` | 动态选择指标 |
| | 指标标签 | `assets/js/dashboard/stats/metrics.ts` | 前端显示标签映射 |

---

**报告完成日期**：2026-05-02