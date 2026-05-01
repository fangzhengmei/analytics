# Plausible Analytics 数据链路分析报告

> 分析日期：2026-05-02
> 分析对象：Plausible Analytics 代码库

---

## 一、数据链路总览

Plausible Analytics 的数据流转包含以下五个核心环节：

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  1. 前端追踪采集  │ ──▶ │ 2. 服务端请求接收 │ ──▶ │ 3. 事件处理管道  │
│  (Tracker JS)   │     │  (API Endpoint) │     │ (Ingestion Pipeline)│
└─────────────────┘     └─────────────────┘     └─────────────────┘
                                                              │
                                                              ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│ 6. 仪表盘展示    │ ◀── │ 5. 查询执行层    │ ◀── │ 4. ClickHouse  │
│  (React Dashboard)│     │  (Query Runner)  │     │   批量写入      │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

---

## 二、各环节详细分析

### 2.1 前端追踪采集层

#### 2.1.1 追踪脚本架构

前端追踪由两部分组成：

| 模块 | 文件路径 | 职责 |
|------|----------|------|
| 动态脚本生成 | `lib/plausible_web/tracker.ex` | 根据站点配置生成个性化追踪脚本 |
| 脚本路由 | `lib/plausible_web/plugs/tracker_plug.ex` | 提供 `/js/plausible.js` 等端点 |
| 核心追踪逻辑 | `priv/tracker/js/p.js` | 压缩后的追踪脚本核心 |

#### 2.1.2 追踪脚本的关键特性

**1. 动态配置注入** (`lib/plausible_web/tracker.ex:50-67`)

```elixir
def build_script(tracker_script_configuration) do
  config_js_content =
    tracker_script_configuration
    |> plausible_main_config()
    |> Enum.flat_map(fn
      {key, value} when is_binary(value) -> ["#{key}:#{JSON.encode!(value)}"]
      {key, true} -> ["#{key}:!0"]  # 缩写 true 为 !0 节省字节
      {_key, false} -> []
    end)
    |> Enum.join(",")

  @plausible_main_script
  |> String.replace("\"<%= @config_js %>\"", "{#{config_js_content}}")
end
```

配置项包括：
- `domain`: 目标域名
- `endpoint`: 数据上报端点（默认 `/api/event`）
- `outboundLinks`: 出站链接追踪开关
- `fileDownloads`: 文件下载追踪开关
- `formSubmissions`: 表单提交追踪开关

**2. 支持的脚本变体** (`lib/plausible_web/plugs/tracker_plug.ex:10-35`)

```elixir
base_variants = [
  "hash",           # 启用 hash 路由模式
  "outbound-links", # 自动追踪出站链接
  "exclusions",     # 支持页面排除规则
  "compat",         # 兼容性模式（兼容旧浏览器）
  "local",          # 本地开发模式
  "manual",         # 手动触发模式
  "file-downloads", # 文件下载追踪
  "pageview-props", # 页面浏览自定义属性
  "tagged-events",  # 标记事件
  "revenue",        # 收入追踪
  "pageleave"       # 页面离开事件
]
```

**3. 数据上报逻辑** (`priv/tracker/js/p.js`)

核心上报函数：
```javascript
function trigger(eventName, options) {
  // 构建事件数据
  const payload = {
    n: eventName,           // 事件名称
    u: window.location.href, // 当前页面 URL
    d: config.domain,        // 域名
    r: document.referrer,    // 来源页面
    w: window.innerWidth,    // 屏幕宽度
    m: options.props,         // 自定义属性
    ...
  };
  
  // 使用 XMLHttpRequest 发送
  const xhr = new XMLHttpRequest();
  xhr.open("POST", endpoint + "/api/event", true);
  xhr.setRequestHeader("Content-Type", "text/plain");
  xhr.send(JSON.stringify(payload));
}
```

#### 2.1.3 事件类型

| 事件名称 | 触发时机 | 说明 |
|----------|----------|------|
| `pageview` | 页面加载 | 页面浏览事件 |
| `engagement` | 页面滚动/停留 | 用户参与度事件 |
| 自定义事件 | 手动调用 `plausible('eventName')` | 业务自定义事件 |

---

### 2.2 服务端请求接收层

#### 2.2.1 路由配置

**路由定义** (`lib/plausible_web/router.ex:413-427`)

```elixir
scope "/api", PlausibleWeb do
  pipe_through :external_api
  
  post "/event", Api.ExternalController, :event  # 事件接收端点
  get "/error", Api.ExternalController, :error    # JS 错误上报
end
```

#### 2.2.2 控制器处理

**事件控制器** (`lib/plausible_web/controllers/api/external_controller.ex:13-45`)

```elixir
def event(conn, _params) do
  with {:ok, request, conn} <- Ingestion.Request.build(conn),
       _ <- Sentry.Context.set_extra_context(%{request: request}) do
    case Ingestion.Event.build_and_buffer(request) do
      {:ok, %{dropped: [], buffered: _buffered}} ->
        conn
        |> put_status(202)
        |> text("ok")
      
      {:ok, %{dropped: dropped, buffered: _}} ->
        # 处理部分事件被丢弃的情况
        conn
        |> put_resp_header("x-plausible-dropped", "#{Enum.count(dropped)}")
        |> put_status(202)
        |> text("ok")
    end
  else
    {:error, %Ecto.Changeset{} = changeset} ->
      conn
      |> put_status(400)
      |> json(%{errors: ...})
  end
end
```

关键设计点：
- **异步处理**: 返回 202 Accepted，不阻塞请求
- **快速失败**: 验证失败立即返回 400
- **部分失败**: 部分事件丢弃仍返回 202，但设置 `x-plausible-dropped` 响应头

#### 2.2.3 请求构建

**Request 结构体构建** (`lib/plausible/ingestion/request.ex:76-124`)

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
        |> put_ip_classification(conn)      # IP 类型（数据中心/威胁/匿名）
        |> put_remote_ip(conn)              # 真实客户端 IP
        |> put_uri(request_body)             # 解析 URL
        |> put_hostname()                    # 主机名
        |> put_user_agent(conn)              # User-Agent
        |> put_request_params(request_body)  # 请求参数
        |> put_referrer(request_body)        # 来源页
        |> put_pathname()                    # 路径
        |> put_props(request_body)           # 自定义属性
        |> put_engagement_fields(request_body) # 参与度字段
        |> put_query_params()                # URL 查询参数
        |> put_revenue_source(request_body)  # 收入数据（EE 版本）
        |> put_interactive(request_body)     # 是否交互事件
        |> put_tracker_script_version(request_body) # 追踪脚本版本
        |> map_domains(request_body)         # 域名列表
        |> Changeset.validate_required([:event_name, :hostname, :pathname, :timestamp])
        |> Changeset.apply_action(nil)
      ...
  end
end
```

**Request 结构体字段** (`lib/plausible/ingestion/request.ex:43-67`)

```elixir
embedded_schema do
  field :remote_ip, :string              # 客户端 IP
  field :user_agent, :string             # User-Agent
  field :event_name, Plausible.Ecto.EventName  # 事件名称
  field :uri, :map                       # 解析后的 URI
  field :hostname, :string               # 主机名
  field :referrer, :string               # 来源页
  field :domains, {:array, :string}      # 目标域名列表
  field :ip_classification, :string      # IP 分类
  field :hash_mode, :integer             # Hash 路由模式
  field :pathname, :string               # 路径
  field :props, :map                     # 自定义属性
  field :scroll_depth, :integer          # 滚动深度 (0-100)
  field :engagement_time, :integer       # 参与时间（秒）
  field :tracker_script_version, :integer # 追踪脚本版本
  field :interactive?, :boolean, default: true  # 是否交互事件
  field :query_params, :map              # URL 查询参数
  field :timestamp, :naive_datetime      # 事件时间戳
end
```

---

### 2.3 事件处理管道

#### 2.3.1 管道总览

事件处理是一个 **流水线（Pipeline）** 模式，每一步可以决定继续处理或丢弃事件。

**核心入口** (`lib/plausible/ingestion/event.ex:56-80`)

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

#### 2.3.2 处理管道步骤

**管道定义** (`lib/plausible/ingestion/event.ex:130-151`)

```elixir
defp pipeline() do
  [
    # 1. 验证和过滤层
    drop_verification_agent: &drop_verification_agent/2,   # 丢弃验证机器人
    drop_datacenter_ip: &drop_datacenter_ip/2,             # 丢弃数据中心 IP
    drop_threat_ip: &drop_threat_ip/2,                     # 丢弃威胁 IP
    drop_shield_rule_hostname: &drop_shield_rule_hostname/2, # 主机名白名单
    drop_shield_rule_page: &drop_shield_rule_page/2,       # 页面黑名单
    drop_shield_rule_ip: &drop_shield_rule_ip/2,           # IP 黑名单
    
    # 2. 信息提取层
    put_geolocation: &put_geolocation/2,                    # 地理定位
    drop_shield_rule_country: &drop_shield_rule_country/2, # 国家黑名单（需先定位）
    put_user_agent: &put_user_agent/2,                      # User-Agent 解析
    put_basic_info: &put_basic_info/2,                      # 基础信息
    put_source_info: &put_source_info/2,                    # 来源信息
    maybe_infer_medium: &maybe_infer_medium/2,              # 推断媒介
    put_props: &put_props/2,                                 # 自定义属性
    put_revenue: &put_revenue/2,                             # 收入数据
    put_salts: &put_salts/2,                                 # 获取盐值
    put_user_id: &put_user_id/2,                             # 生成用户 ID
    
    # 3. 验证和持久化层
    validate_clickhouse_event: &validate_clickhouse_event/2, # 验证事件结构
    register_session: &register_session/2                     # 注册会话并持久化
  ]
end
```

#### 2.3.3 关键步骤详解

**1. User-Agent 解析** (`lib/plausible/ingestion/event.ex:256-276`)

```elixir
defp put_user_agent(%__MODULE__{} = event, _context) do
  case parse_user_agent(event.request) do
    {:ok, %UAInspector.Result{client: %UAInspector.Result.Client{name: "Headless Chrome"}}} ->
      drop(event, :bot)  # 无头浏览器判定为机器人
    
    {:ok, %UAInspector.Result.Bot{}} ->
      drop(event, :bot)  # 已知机器人
    
    {:ok, %UAInspector.Result{} = user_agent} ->
      update_session_attrs(event, %{
        operating_system: os_name(user_agent),
        operating_system_version: os_version(user_agent),
        browser: browser_name(user_agent),
        browser_version: browser_version(user_agent),
        screen_size: screen_size(user_agent)  # Mobile/Tablet/Desktop
      })
    
    _any ->
      event
  end
end
```

**2. 地理定位** (`lib/plausible/ingestion/event.ex:324-333`)

```elixir
defp put_geolocation(%__MODULE__{} = event, _context) do
  case event.request.ip_classification do
    "anonymous_vpn_ip" ->
      update_session_attrs(event, %{country_code: "A1"})  # 匿名代理
    
    _any ->
      result = Plausible.Ingestion.Geolocation.lookup(event.request.remote_ip) || %{}
      update_session_attrs(event, result)  # 包含 country_code, city_geoname_id 等
  end
end
```

**3. 用户 ID 生成** (`lib/plausible/ingestion/event.ex:553-567`)

```elixir
defp generate_user_id(request, domain, hostname, salt) do
  cond do
    is_nil(salt) -> nil
    is_nil(domain) -> nil
    true ->
      user_agent = request.user_agent || ""
      root_domain = get_root_domain(hostname)
      
      # 使用 SipHash 生成不可逆的用户标识符
      SipHash.hash!(salt, user_agent <> request.remote_ip <> domain <> root_domain)
  end
end
```

**4. 会话注册和持久化** (`lib/plausible/ingestion/event.ex:414-433`)

```elixir
defp register_session(%__MODULE__{} = event, context) do
  persistor_opts = Keyword.get(context, :persistor_opts, [])
  
  # 生成前一天盐值的用户 ID（用于跨天会话延续）
  previous_user_id =
    generate_user_id(
      event.request,
      event.domain,
      event.clickhouse_event.hostname,
      event.salts.previous
    )
  
  case Plausible.Ingestion.Persistor.persist_event(event, previous_user_id, persistor_opts) do
    {:ok, event} ->
      emit_telemetry_buffered(event)
      event
    
    {:error, reason} ->
      drop(event, reason)
  end
end
```

#### 2.3.4 事件丢弃原因

所有可能的丢弃原因定义在 (`lib/plausible/ingestion/event.ex:24-41`)

```elixir
@type drop_reason() ::
        :bot                      # 机器人/无头浏览器
        | :spam_referrer          # 垃圾来源
        | GateKeeper.policy()     # 站点访问控制策略
        | :invalid                 # 验证失败
        | :dc_ip                   # 数据中心 IP
        | :threat_ip               # 威胁 IP
        | :site_ip_blocklist       # 站点 IP 黑名单
        | :site_country_blocklist  # 站点国家黑名单
        | :site_page_blocklist     # 站点页面黑名单
        | :site_hostname_allowlist # 站点主机名白名单
        | :verification_agent      # 验证代理
        | :lock_timeout            # 锁超时
        | :no_session_for_engagement  # 参与度事件无对应会话
        | :persist_timeout         # 持久化超时
        | :persist_error           # 持久化错误
        | :persist_decode_error    # 持久化解码错误
```

---

### 2.4 会话管理

#### 2.4.1 会话持久化入口

**Persistor 模块** (`lib/plausible/ingestion/persistor.ex`)

```elixir
defmodule Plausible.Ingestion.Persistor do
  @fallback_backend Plausible.Ingestion.Persistor.Embedded
  
  def persist_event(event, previous_user_id, opts) do
    {backend_override, opts} = Keyword.pop(opts, :backend)
    user_id = event.clickhouse_event.user_id
    
    # 支持灰度发布：根据 user_id 哈希选择后端
    backend(backend_override, user_id).persist_event(event, previous_user_id, opts)
  end
end
```

#### 2.4.2 嵌入式持久化实现

**核心实现** (`lib/plausible/ingestion/persistor/embedded.ex:10-41`)

```elixir
defmodule Plausible.Ingestion.Persistor.Embedded do
  def persist_event(ingest_event, previous_user_id, opts) do
    event = ingest_event.clickhouse_event
    session_attrs = ingest_event.clickhouse_session_attrs
    
    session_write_buffer_insert =
      Keyword.get(opts, :session_write_buffer_insert, &Plausible.Session.WriteBuffer.insert/1)
    
    event_write_buffer_insert =
      Keyword.get(opts, :event_write_buffer_insert, &Plausible.Event.WriteBuffer.insert/1)
    
    # 核心会话处理逻辑
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
        # 合并会话属性到事件
        event = ClickhouseEventV2.merge_session(event, session)
        {:ok, _} = event_write_buffer_insert.(event)
        
        {:ok, %{ingest_event | clickhouse_event: event}}
    end
  end
end
```

#### 2.4.3 会话缓存存储

**CacheStore 核心逻辑** (`lib/plausible/session/cache_store.ex`)

```elixir
def on_event(event, session_attributes, prev_user_id, opts \\ []) do
  buffer_insert = Keyword.get(opts, :buffer_insert, &WriteBuffer.insert/1)
  
  try do
    response =
      Plausible.Session.Balancer.dispatch(
        event.user_id,
        fn ->
          # 查找现有会话（当前盐值或前一天盐值）
          found_session =
            find_session(event, event.user_id) || find_session(event, prev_user_id)
          
          handle_event(event, found_session, session_attributes, buffer_insert)
        end,
        timeout: @lock_timeout,
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
    # 参与度事件只更新会话缓存，不创建新会话
    refresh_session_cache(found_session, event.timestamp)
    found_session
  else
    :no_session_for_engagement
  end
end

defp handle_event(event, found_session, session_attributes, buffer_insert) do
  if found_session do
    # 更新现有会话：使用 sign=-1 撤消旧记录，sign=1 写入新记录
    updated_session = update_session(found_session, event)
    buffer_insert.([%{found_session | sign: -1}, %{updated_session | sign: 1}])
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

**会话更新字段** (`lib/plausible/session/cache_store.ex:95-123`)

```elixir
defp update_session(session, event) do
  pageview? = event.name == "pageview"
  pageviews = if(pageview?, do: session.pageviews + 1, else: session.pageviews)
  
  %{
    session
    | timestamp: event.timestamp,
      entry_page: if(session.entry_page == "" and pageview?, do: event.pathname, else: session.entry_page),
      hostname: if(pageview? and session.hostname == "", do: event.hostname, else: session.hostname),
      exit_page: if(pageview?, do: event.pathname, else: session.exit_page),
      exit_page_hostname: if(pageview?, do: event.hostname, else: session.exit_page_hostname),
      is_bounce: if(session.is_bounce, do: not (pageviews >= 2 or (event.interactive? and not pageview?)), else: session.is_bounce),
      duration: NaiveDateTime.diff(event.timestamp, session.start) |> abs,
      pageviews: pageviews,
      events: session.events + 1
  }
end
```

---

### 2.5 ClickHouse 批量写入层

#### 2.5.1 写入缓冲区架构

Plausible 使用 **批量写入 + RowBinary 格式** 来优化 ClickHouse 写入性能。

**通用写入缓冲区** (`lib/plausible/ingestion/write_buffer.ex`)

```elixir
defmodule Plausible.Ingestion.WriteBuffer do
  use GenServer
  
  def init(opts) do
    buffer = opts[:buffer] || []
    max_buffer_size = opts[:max_buffer_size] || default_max_buffer_size()
    flush_interval_ms = opts[:flush_interval_ms] || default_flush_interval_ms()
    
    Process.flag(:trap_exit, true)
    timer = Process.send_after(self(), :tick, flush_interval_ms)
    
    {:ok,
     %{
       buffer: buffer,
       timer: timer,
       name: Keyword.fetch!(opts, :name),
       insert_sql: Keyword.fetch!(opts, :insert_sql),
       insert_opts: Keyword.fetch!(opts, :insert_opts),
       header: Keyword.fetch!(opts, :header),
       buffer_size: IO.iodata_length(buffer),
       max_buffer_size: max_buffer_size,
       flush_interval_ms: flush_interval_ms
     }}
  end
end
```

**触发刷新的条件** (`lib/plausible/ingestion/write_buffer.ex:44-67`)

```elixir
# 1. 缓冲区满
def handle_cast({:insert, row_binary}, state) do
  state = %{
    state
    | buffer: [state.buffer | row_binary],
      buffer_size: state.buffer_size + IO.iodata_length(row_binary)
  }
  
  if state.buffer_size >= state.max_buffer_size do
    # 达到最大缓冲大小时立即刷新
    Process.cancel_timer(state.timer)
    do_flush(state)
    new_timer = Process.send_after(self(), :tick, state.flush_interval_ms)
    {:noreply, %{state | buffer: [], timer: new_timer, buffer_size: 0}}
  else
    {:noreply, state}
  end
end

# 2. 定时刷新
def handle_info(:tick, state) do
  do_flush(state)
  timer = Process.send_after(self(), :tick, state.flush_interval_ms)
  {:noreply, %{state | buffer: [], buffer_size: 0, timer: timer}}
end

# 3. 进程退出时刷新
def terminate(_reason, %{name: name} = state) do
  Logger.notice("Flushing #{name} buffer before shutdown...")
  do_flush(state)
end
```

**实际写入 ClickHouse** (`lib/plausible/ingestion/write_buffer.ex:84-102`)

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
    [] -> nil
    _not_empty ->
      Logger.notice("Flushing #{buffer_size} byte(s) RowBinary from #{name}")
      # 使用 RowBinaryWithNamesAndTypes 格式批量插入
      IngestRepo.query!(insert_sql, [header | buffer], insert_opts)
  end
end
```

#### 2.5.2 编译时 Schema 准备

**编译时生成插入配置** (`lib/plausible/ingestion/write_buffer.ex:113-151`)

```elixir
def compile_time_prepare(schema) do
  fields =
    schema.__schema__(:fields)
    |> Enum.reject(&(&1 in fields_to_ignore()))
  
  # 提取字段类型
  types =
    Enum.map(fields, fn field ->
      type = schema.__schema__(:type, field) || raise "missing type for #{field}"
      type
      |> Ecto.Type.type()
      |> Ecto.Adapters.ClickHouse.Schema.remap_type(schema, field)
    end)
  
  encoding_types = Ch.RowBinary.encoding_types(types)
  
  # 生成 RowBinary 头部（字段名 + 类型）
  header =
    fields
    |> Enum.map(&to_string/1)
    |> Ch.RowBinary.encode_names_and_types(types)
    |> IO.iodata_to_binary()
  
  # 生成 INSERT SQL
  insert_sql =
    "INSERT INTO #{schema.__schema__(:source)} (#{Enum.join(fields, ", ")}) FORMAT RowBinaryWithNamesAndTypes"
  
  %{
    fields: fields,
    types: types,
    encoding_types: encoding_types,
    header: header,
    insert_sql: insert_sql,
    insert_opts: [...]
  }
end
```

#### 2.5.3 事件和会话写入缓冲区

**事件写入缓冲区** (`lib/plausible/event/write_buffer.ex`)

```elixir
defmodule Plausible.Event.WriteBuffer do
  # 编译时为 ClickhouseEventV2 生成配置
  %{
    header: header,
    insert_sql: insert_sql,
    insert_opts: insert_opts,
    fields: fields,
    encoding_types: encoding_types
  } = Plausible.Ingestion.WriteBuffer.compile_time_prepare(Plausible.ClickhouseEventV2)
  
  def insert(event) do
    # 将事件编码为 RowBinary 格式
    row_binary =
      [Enum.map(unquote(fields), fn field -> Map.fetch!(event, field) end)]
      |> Ch.RowBinary._encode_rows(unquote(encoding_types))
      |> IO.iodata_to_binary()
    
    :ok = Plausible.Ingestion.WriteBuffer.insert(__MODULE__, row_binary)
    {:ok, event}
  end
end
```

**会话写入缓冲区** (`lib/plausible/session/write_buffer.ex`)

```elixir
defmodule Plausible.Session.WriteBuffer do
  # 编译时为 ClickhouseSessionV2 生成配置
  %{...} = Plausible.Ingestion.WriteBuffer.compile_time_prepare(Plausible.ClickhouseSessionV2)
  
  def insert(sessions) do
    row_binary =
      sessions
      |> Enum.map(fn %{is_bounce: is_bounce} = session ->
        # 特殊处理布尔字段为 UInt8
        {:ok, is_bounce} = Plausible.ClickhouseSessionV2.BoolUInt8.dump(is_bounce)
        session = %{session | is_bounce: is_bounce}
        Enum.map(unquote(fields), fn field -> Map.fetch!(session, field) end)
      end)
      |> Ch.RowBinary._encode_rows(unquote(encoding_types))
      |> IO.iodata_to_binary()
    
    :ok = Plausible.Ingestion.WriteBuffer.insert(__MODULE__, row_binary)
    {:ok, sessions}
  end
end
```

#### 2.5.4 ClickHouse 表结构

**事件表** (`lib/plausible/clickhouse_event_v2.ex:1-54`)

```elixir
schema "events_v2" do
  # 基础字段
  field :name, Ch, type: "LowCardinality(String)"  # 事件名称
  field :site_id, Ch, type: "UInt64"               # 站点 ID
  field :hostname, :string                           # 主机名
  field :pathname, :string                           # 路径
  field :user_id, Ch, type: "UInt64"                # 用户 ID
  field :session_id, Ch, type: "UInt64"             # 会话 ID
  field :timestamp, :naive_datetime                  # 时间戳
  
  # 自定义属性
  field :"meta.key", {:array, :string}
  field :"meta.value", {:array, :string}
  
  # 参与度字段
  field :scroll_depth, Ch, type: "UInt8"            # 滚动深度 0-100
  field :engagement_time, Ch, type: "UInt32"        # 参与时间
  
  # 收入字段（EE 版本）
  field :revenue_source_amount, Ch, type: "Nullable(Decimal64(3))"
  field :revenue_source_currency, Ch, type: "FixedString(3)"
  field :revenue_reporting_amount, Ch, type: "Nullable(Decimal64(3))"
  field :revenue_reporting_currency, Ch, type: "FixedString(3)"
  
  # 会话属性（从会话合并）
  field :referrer, :string
  field :referrer_source, :string
  field :click_id_param, Ch, type: "LowCardinality(String)"
  field :utm_medium, :string
  field :utm_source, :string
  field :utm_campaign, :string
  field :utm_content, :string
  field :utm_term, :string
  
  # 地理位置
  field :country_code, Ch, type: "FixedString(2)"
  field :subdivision1_code, Ch, type: "LowCardinality(String)"
  field :subdivision2_code, Ch, type: "LowCardinality(String)"
  field :city_geoname_id, Ch, type: "UInt32"
  
  # 设备信息
  field :screen_size, Ch, type: "LowCardinality(String)"
  field :operating_system, Ch, type: "LowCardinality(String)"
  field :operating_system_version, Ch, type: "LowCardinality(String)"
  field :browser, Ch, type: "LowCardinality(String)"
  field :browser_version, Ch, type: "LowCardinality(String)"
end
```

---

### 2.6 仪表盘查询与展示层

#### 2.6.1 查询流程总览

```
前端 Dashboard
      │
      ▼
  API 调用 (POST /api/stats/:domain/query)
      │
      ▼
StatsController.query/2
      │
      ▼
QueryBuilder.build/3  (参数解析、验证)
      │
      ▼
QueryRunner.run/2     (查询执行)
      │
      ▼
SQL.QueryBuilder.build/2  (生成 ClickHouse SQL)
      │
      ▼
ClickhouseRepo.all/2  (执行查询)
```

#### 2.6.2 前端 API 调用

**前端 API 模块** (`assets/js/dashboard/api.ts`)

```typescript
export async function stats(site: PlausibleSite, statsQuery: StatsQuery) {
  const sharedLinkParams = getSharedLinkSearchParams()
  const queryString = sharedLinkParams.auth
    ? new URLSearchParams(sharedLinkParams).toString()
    : ''
  const path = url.apiPath(site, '/query')
  const response = await fetch(queryString ? `${path}?${queryString}` : path, {
    method: 'POST',
    signal: abortController.signal,  // 支持取消请求
    headers: {
      ...getHeaders(),
      'Content-Type': 'application/json',
      Accept: 'application/json'
    },
    body: JSON.stringify(statsQuery)
  })
  
  return handleApiResponse(response)
}
```

**查询参数构建** (`assets/js/dashboard/api.ts:53-103`)

```typescript
export function dashboardStateToParams(
  dashboardState: DashboardState,
  extraQuery: unknown[] = []
): Record<string, string> {
  const queryObj: Record<string, string> = {}
  if (dashboardState.period) {
    queryObj.period = dashboardState.period
  }
  if (dashboardState.date) {
    queryObj.date = formatISO(dashboardState.date)
  }
  if (dashboardState.from) {
    queryObj.from = formatISO(dashboardState.from)
  }
  if (dashboardState.to) {
    queryObj.to = formatISO(dashboardState.to)
  }
  if (dashboardState.filters) {
    queryObj.filters = serializeApiFilters(dashboardState.filters)
  }
  // 对比查询参数
  if (dashboardState.comparison) {
    queryObj.comparison = dashboardState.comparison
    queryObj.compare_from = dashboardState.compare_from
      ? formatISO(dashboardState.compare_from)
      : undefined
    queryObj.compare_to = dashboardState.compare_to
      ? formatISO(dashboardState.compare_to)
      : undefined
  }
  // ...
  return queryObj
}
```

#### 2.6.3 服务端查询处理

**StatsController** (`lib/plausible_web/controllers/api/stats_controller.ex:40-57`)

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
    
    json(conn, Plausible.Stats.query(site, query))
  else
    {:error, %QueryError{message: message}} -> bad_request(conn, message)
  end
end
```

**QueryBuilder 构建过程** (`lib/plausible/stats/query_builder.ex:31-60`)

```elixir
def build(site, %ParsedQueryParams{} = parsed_query_params, debug_metadata) do
  with {:ok, parsed_query_params} <- resolve_segments_in_filters(parsed_query_params, site),
       query = do_build(parsed_query_params, site, debug_metadata),
       :ok <- validate_order_by(query),
       :ok <- validate_custom_props_access(site, query),
       :ok <- validate_case_sensitive_filter_modifier(query),
       :ok <- validate_toplevel_only_filter_dimension(query),
       :ok <- validate_time_dimension_granularity(query),
       :ok <- validate_special_metrics_filters(query),
       :ok <- validate_behavioral_filters(query),
       :ok <- validate_filtered_goals_exist(query, parsed_query_params),
       :ok <- validate_revenue_metrics_access(site, query),
       :ok <- validate_metrics(query),
       :ok <- validate_include(query) do
    query =
      query
      |> set_time_on_page_data(site)
      |> put_comparison_utc_time_range()
      |> Query.put_imported_opts(site)
    
    {:ok, query}
  end
end
```

**QueryRunner 执行** (`lib/plausible/stats/query_runner.ex:37-48`)

```elixir
def run(site, query) do
  optimized_query = QueryOptimizer.optimize(query)
  
  Query.trace(optimized_query, optimized_query.metrics)
  
  %__MODULE__{main_query: optimized_query, site: site}
  |> execute_main_query()      # 执行主查询
  |> add_comparison_query()    # 添加对比查询（如果需要）
  |> execute_comparison_query() # 执行对比查询
  |> build_results_list()      # 构建结果列表
  |> QueryResult.from()        # 转换为最终结果格式
end
```

**实际 SQL 执行** (`lib/plausible/stats/query_runner.ex:146-150`)

```elixir
defp execute_query(query, site) do
  query
  |> SQL.QueryBuilder.build(site)  # 生成 ClickHouse SQL
  |> ClickhouseRepo.all(query: query)  # 执行查询
end
```

#### 2.6.4 其他统计 API 端点

**来源统计** (`lib/plausible_web/controllers/api/stats_controller.ex:59-97`)

```elixir
def sources(conn, params) do
  site = conn.assigns[:site]
  params = Map.put(params, "property", "visit:source")
  query = Query.from(site, params, debug_metadata: debug_metadata(conn))
  pagination = parse_pagination(params)
  
  extra_metrics =
    if params["detailed"],
      do: [:percentage, :bounce_rate, :visit_duration],
      else: [:percentage]
  
  metrics = breakdown_metrics(query, extra_metrics: extra_metrics, ...)
  
  %{results: results, meta: meta} = Stats.breakdown(site, query, metrics, pagination)
  
  json(conn, %{
    results: results,
    meta: Stats.Breakdown.formatted_date_ranges(query),
    skip_imported_reason: meta[:imports_skip_reason]
  })
end
```

类似的端点还包括：
- `channels/2` - 渠道统计
- `pages/2` - 页面统计
- `countries/2` - 国家统计
- `browsers/2` - 浏览器统计
- `operating_systems/2` - 操作系统统计
- `conversions/2` - 转化统计
- `custom_prop_values/2` - 自定义属性值

---

## 三、关键技术点

### 3.1 性能优化策略

| 策略 | 实现位置 | 说明 |
|------|----------|------|
| **批量写入** | `WriteBuffer` | 积累一定量数据后批量插入 ClickHouse |
| **RowBinary 格式** | `compile_time_prepare/1` | 比 JSON 更紧凑、更快的序列化格式 |
| **编译时生成** | `WriteBuffer.compile_time_prepare/1` | Schema 配置在编译时生成，避免运行时开销 |
| **会话缓存** | `Session.CacheStore` | 内存缓存会话，避免频繁查库 |
| **盐值轮转** | `Session.Salts` | 每日轮换盐值，保护用户隐私 |
| **Query 优化** | `QueryOptimizer` | 查询优化器，如谓词下推等 |

### 3.2 隐私保护机制

1. **用户 ID 哈希**：使用 SipHash + 每日轮转盐值，无法反向识别用户
2. **无 Cookie 追踪**：默认不使用持久化 Cookie，通过 IP + User-Agent + 域名 组合识别
3. **数据中心 IP 过滤**：自动丢弃已知数据中心的请求
4. **自定义排除规则**：支持 IP、国家、页面、主机名等多维度排除

### 3.3 数据一致性保证

**会话的 MVCC 式更新**：
```elixir
# 更新会话时，先写入 sign=-1 撤消旧记录，再写入 sign=1 新记录
buffer_insert.([%{found_session | sign: -1}, %{updated_session | sign: 1}])
```

ClickHouse 在聚合时会：
```sql
sum(sign)  -- 最终结果为 1（新记录）或 0（已删除）
```

### 3.4 事件类型和数据模型

**事件类型**：
- `pageview` - 页面浏览
- `engagement` - 参与度（滚动深度、停留时间）
- 自定义事件 - 业务自定义

**两张核心表**：
1. `events_v2` - 存储所有事件（事实表）
2. `sessions_v2` - 存储会话（维度表，使用 sign 字段实现更新）

---

## 四、数据流总结

### 4.1 完整数据流图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           1. 前端追踪采集层                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│  用户网站                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ <script defer data-domain="example.com" src="/js/script.js"></script> │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                          │
│                                    ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Tracker JS (p.js)                                                    │   │
│  │ - 监听页面加载、滚动、点击等事件                                       │   │
│  │ - 构建事件 payload: {n, u, d, r, w, m, ...}                         │   │
│  │ - POST /api/event (Content-Type: text/plain)                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           2. 服务端请求接收层                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│  Phoenix Router: POST /api/event → ExternalController.event/2               │
│                                    │                                          │
│                                    ▼                                          │
│  Ingestion.Request.build/2                                                   │
│  - 解析请求体 JSON                                                           │
│  - 提取 IP、User-Agent、URL 参数等                                           │
│  - 验证必需字段（event_name, hostname, pathname, timestamp）                  │
│  - 构建 %Request{} 结构体                                                    │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           3. 事件处理管道                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│  Ingestion.Event.build_and_buffer/2                                          │
│                                    │                                          │
│                                    ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 管道步骤 (Pipeline)                                                    │   │
│  │ ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                   │   │
│  │ │ 过滤层      │─▶│ 信息提取层  │─▶│ 持久化层    │                   │   │
│  │ │ - 机器人    │  │ - 地理定位  │  │ - 用户ID    │                   │   │
│  │ │ - 数据中心  │  │ - UA 解析   │  │ - 会话注册  │                   │   │
│  │ │ - 威胁IP    │  │ - 来源提取  │  │ - 事件验证  │                   │   │
│  │ │ - 自定义规则│  │ - 自定义属性│  │             │                   │   │
│  │ └─────────────┘  └─────────────┘  └─────────────┘                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                          │
│                                    ▼                                          │
│  输出: {buffered: [...], dropped: [...]}                                     │
│  - buffered: 成功处理，准备写入 ClickHouse                                    │
│  - dropped: 被丢弃，附带原因                                                   │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           4. 会话管理层                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│  Persistor.Embedded.persist_event/3                                          │
│                                    │                                          │
│                                    ▼                                          │
│  Session.CacheStore.on_event/4                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 1. 查找现有会话 (内存缓存)                                              │   │
│  │    - 按 {site_id, user_id} 查找                                        │   │
│  │    - 检查会话是否超时 (30分钟)                                          │   │
│  │    - 支持跨天延续 (prev_user_id)                                        │   │
│  │                                                                         │   │
│  │ 2. 处理会话                                                             │   │
│  │    - 新会话: 创建新记录，sign=1                                          │   │
│  │    - 旧会话: sign=-1 撤消旧记录, sign=1 写入新记录                      │   │
│  │    - 参与度事件: 仅更新缓存时间戳                                        │   │
│  │                                                                         │   │
│  │ 3. 写入缓冲区                                                           │   │
│  │    - Session.WriteBuffer.insert/1                                       │   │
│  │    - Event.WriteBuffer.insert/1                                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                          │
│                                    ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 分布式锁 (Session.Balancer)                                            │   │
│  │ - 同一 user_id 的事件串行处理                                           │   │
│  │ - 避免并发更新会话导致不一致                                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           5. ClickHouse 批量写入层                            │
├─────────────────────────────────────────────────────────────────────────────┤
│  WriteBuffer (GenServer)                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 缓冲机制                                                               │   │
│  │ - buffer: 累积的 RowBinary 数据                                        │   │
│  │ - buffer_size: 当前缓冲区大小                                           │   │
│  │ - max_buffer_size: 最大缓冲区大小 (配置)                                 │   │
│  │ - flush_interval_ms: 刷新间隔 (配置)                                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                          │
│                                    ▼                                          │
│  刷新触发条件                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐            │
│  │ buffer_size >=  │  │ 定时 tick      │  │ 进程终止        │            │
│  │ max_buffer_size │  │                 │  │ (terminate)     │            │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘            │
│                                    │                                          │
│                                    ▼                                          │
│  do_flush/1                                                                   │
│  - IngestRepo.query!(insert_sql, [header | buffer], insert_opts)            │
│  - 格式: RowBinaryWithNamesAndTypes                                           │
│  - 表: events_v2 / sessions_v2                                                │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────┐
                    │     ClickHouse 存储       │
                    │  ┌───────────────────┐   │
                    │  │    events_v2      │   │
                    │  │ - 所有事件记录    │   │
                    │  │ - 包含会话属性    │   │
                    │  └───────────────────┘   │
                    │  ┌───────────────────┐   │
                    │  │   sessions_v2     │   │
                    │  │ - 会话聚合记录    │   │
                    │  │ - sign 字段更新   │   │
                    │  └───────────────────┘   │
                    └───────────────────────────┘
                                    │
                                    ▼ (查询方向)
┌─────────────────────────────────────────────────────────────────────────────┐
│                           6. 仪表盘查询与展示层                               │
├─────────────────────────────────────────────────────────────────────────────┤
│  前端 Dashboard (React + TypeScript)                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ dashboard/api.ts                                                       │   │
│  │ - stats/2: 执行查询 POST /api/stats/:domain/query                     │   │
│  │ - get/3: 获取细分数据 GET /api/stats/:domain/:endpoint                │   │
│  │ - 支持 AbortController 取消请求                                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                          │
│                                    ▼                                          │
│  StatsController                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ query/2          - 通用查询接口                                        │   │
│  │ sources/2        - 来源统计                                            │   │
│  │ pages/2          - 页面统计                                            │   │
│  │ countries/2      - 国家统计                                            │   │
│  │ browsers/2       - 浏览器统计                                          │   │
│  │ conversions/2    - 转化统计                                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                          │
│                                    ▼                                          │
│  Query 构建与执行                                                             │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐                   │
│  │QueryBuilder │────▶│ QueryRunner │────▶│ClickhouseRepo│                  │
│  │ - 参数解析  │     │ - 主查询    │     │ - 执行 SQL  │                   │
│  │ - 验证      │     │ - 对比查询  │     │ - 返回结果  │                   │
│  │ - 构建 Query│     │ - 结果处理  │     │             │                   │
│  └─────────────┘     └─────────────┘     └─────────────┘                   │
│                                    │                                          │
│                                    ▼                                          │
│  SQL 生成 (SQL.QueryBuilder)                                                 │
│  - 动态生成 ClickHouse 专用 SQL                                              │
│  - 支持聚合、分组、过滤、时间粒度等                                           │
│  - 优化: 仅查询需要的列，使用 Materialized View 等                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 关键文件索引

| 层级 | 模块 | 文件路径 |
|------|------|----------|
| **前端追踪** | 动态脚本生成 | `lib/plausible_web/tracker.ex` |
| | 脚本路由 | `lib/plausible_web/plugs/tracker_plug.ex` |
| | 追踪脚本 | `priv/tracker/js/p.js` |
| **服务端接收** | 路由 | `lib/plausible_web/router.ex` |
| | 控制器 | `lib/plausible_web/controllers/api/external_controller.ex` |
| | 请求构建 | `lib/plausible/ingestion/request.ex` |
| **事件处理** | 管道入口 | `lib/plausible/ingestion/event.ex` |
| | 持久化 | `lib/plausible/ingestion/persistor.ex` |
| | 地理定位 | `lib/plausible/ingestion/geolocation.ex` |
| | 来源解析 | `lib/plausible/ingestion/source.ex` |
| **会话管理** | 缓存存储 | `lib/plausible/session/cache_store.ex` |
| | 会话写入缓冲 | `lib/plausible/session/write_buffer.ex` |
| | 盐值管理 | `lib/plausible/session/salts.ex` |
| **ClickHouse** | 通用写入缓冲 | `lib/plausible/ingestion/write_buffer.ex` |
| | 事件写入缓冲 | `lib/plausible/event/write_buffer.ex` |
| | 事件 Schema | `lib/plausible/clickhouse_event_v2.ex` |
| | 会话 Schema | `lib/plausible/clickhouse_session_v2.ex` |
| **查询层** | 查询构建 | `lib/plausible/stats/query_builder.ex` |
| | 查询执行 | `lib/plausible/stats/query_runner.ex` |
| | SQL 生成 | `lib/plausible/stats/sql/query_builder.ex` |
| | 统计控制器 | `lib/plausible_web/controllers/api/stats_controller.ex` |
| **前端展示** | API 调用 | `assets/js/dashboard/api.ts` |
| | Dashboard 入口 | `assets/js/dashboard/index.tsx` |

---

## 五、架构设计亮点

### 5.1 写入路径优化

1. **异步缓冲 + 批量写入**：事件不直接写入数据库，而是先进入内存缓冲区，积累到一定量后批量插入 ClickHouse，大幅减少网络往返和数据库连接开销。

2. **RowBinary 格式**：使用 ClickHouse 原生的 RowBinary 格式，比 JSON 更紧凑、解析更快。

3. **编译时优化**：Schema 的插入配置（字段列表、类型、SQL 语句）在编译时生成，避免运行时反射开销。

### 5.2 会话管理

1. **内存缓存**：会话信息存储在内存缓存中，查找和更新都是 O(1) 操作。

2. **分布式锁**：使用 Session.Balancer 对同一用户的事件进行串行化处理，避免并发更新导致的数据不一致。

3. **盐值轮转**：每日生成新的盐值用于用户 ID 哈希，既保证了用户识别的连续性（支持 prev_user_id 跨天延续），又保护了用户隐私。

4. **Sign-based 更新**：利用 ClickHouse 的 MergeTree 特性，通过 sign 字段实现"更新"（旧记录 sign=-1，新记录 sign=1，聚合时 sum(sign) 得到最终状态）。

### 5.3 隐私保护

1. **无 Cookie 设计**：默认不使用持久化 Cookie，通过 IP + User-Agent + 域名 + 盐值 的组合哈希来识别用户。

2. **数据最小化**：不收集不必要的信息，如屏幕分辨率只收集分类（Mobile/Tablet/Desktop）而非精确值。

3. **IP 过滤**：自动过滤数据中心 IP、威胁 IP，减少机器人流量干扰。

4. **自定义排除**：支持站点级别的 IP、国家、页面、主机名等多维度排除规则。

### 5.4 查询灵活性

1. **统一 Query 结构**：所有统计查询都通过 Query 结构体表示，支持：
   - 多种时间粒度（realtime, day, week, month, year, 自定义范围）
   - 多种指标（visitors, pageviews, bounce_rate, visit_duration 等）
   - 多种维度（source, page, country, browser 等）
   - 复杂过滤（支持 and/or、嵌套、行为过滤）
   - 对比查询（与上一周期对比）

2. **Query Optimizer**：查询优化器在执行前对 Query 进行优化，如谓词下推、不必要列移除等。

3. **灵活的 API**：提供两种查询方式：
   - 细分 API（`/sources`, `/pages`, `/countries` 等）：简单易用
   - 通用 Query API（`/query`）：功能强大，支持复杂组合

---

## 六、总结

Plausible Analytics 的数据链路设计体现了以下核心原则：

1. **性能优先**：通过批量写入、内存缓存、编译时优化等技术，在保证数据准确性的同时最大化吞吐量。

2. **隐私保护**：从架构层面重视用户隐私，无 Cookie 设计、盐值轮转、数据最小化等特性贯穿整个数据链路。

3. **可扩展性**：模块化的管道设计、可配置的后端选择、灰度发布支持等，使得系统易于扩展和维护。

4. **用户友好**：简洁的 API 设计、灵活的查询能力、直观的仪表盘，让数据分析变得简单易用。

这种架构使得 Plausible 能够在保持轻量级、隐私友好的同时，提供媲美 Google Analytics 的数据分析能力。
