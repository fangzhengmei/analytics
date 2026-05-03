# 自动采集事件实现边界分析报告

## 概述

本文档分析 Plausible Analytics 中三种自动采集事件（Outbound Link、File Download、404）的实现边界，包括前端识别逻辑、事件语义、后端映射和冲突处理机制。

---

## 一、前端识别逻辑

### 1.1 Outbound Link（出站链接）

**识别函数**: `tracker/src/custom-events.js:148-155`

```javascript
function isOutboundLink(link) {
  return (
    link &&
    typeof link.href === 'string' &&
    link.host &&
    link.host !== location.host
  )
}
```

**识别条件**:
- 必须是 `<a>` 标签（通过 `tagName.toLowerCase() === 'a'` 检查）
- 必须有 `href` 属性
- `link.host` 必须存在且不等于 `location.host`

**触发时机**: 用户点击 `<a>` 标签时（监听 `click` 和 `auxclick` 事件）

---

### 1.2 File Download（文件下载）

**识别函数**: `tracker/src/custom-events.js:157-166`

```javascript
function isDownloadToTrack(url) {
  if (!url) {
    return false
  }

  var fileType = url.split('.').pop()
  return fileTypesToTrack.some(function (fileTypeToTrack) {
    return fileTypeToTrack === fileType
  })
}
```

**识别条件**:
- URL 存在
- 文件扩展名匹配 `fileTypesToTrack` 列表

**默认文件类型** (`tracker/src/custom-events.js:6-33`):
```javascript
DEFAULT_FILE_TYPES = [
  'pdf', 'xlsx', 'docx', 'txt', 'rtf', 'csv',
  'exe', 'key', 'pps', 'ppt', 'pptx',
  '7z', 'pkg', 'rar', 'gz', 'zip',
  'avi', 'mov', 'mp4', 'mpeg', 'wmv',
  'midi', 'mp3', 'wav', 'wma', 'dmg'
]
```

**URL 处理**: 会去除查询参数
```javascript
var hrefWithoutQuery =
  link && typeof link.href === 'string' && link.href.split('?')[0]
```

**可配置性**:
- 通过 `file-types` 属性（script 标签属性）
- 通过 `fileDownloads.fileExtensions` 配置对象
- 通过 `add-file-types` 属性追加到默认列表

---

### 1.3 404 事件

**重要发现**: 404 事件**没有前端自动识别逻辑**。

**配置项存在但未使用**:
- `lib/plausible/site/tracker_script_configuration.ex:27` 定义了 `track_404_pages` 字段
- 但前端 tracker 代码中**没有**使用该配置的逻辑

**触发方式**: 需要用户**手动调用**:
```javascript
plausible('404', { props: { path: '/some-non-existent-page' } })
```

**自动属性同步**: 后端会在 `props.path` 缺失时自动用 `pathname` 填充
- 见 `lib/plausible/ingestion/request.ex:184-194`

---

## 二、事件语义

### 2.1 系统事件定义

**定义文件**: `lib/plausible/event/system_events.ex`

| 事件名称 | 事件类型分类 | 特殊属性处理 |
|---------|-------------|-------------|
| `"Outbound Link: Click"` | `events_with_url_prop` | `props.url` |
| `"File Download"` | `events_with_url_prop` | `props.url` |
| `"404"` | `events_with_path_prop` | `props.path` (自动同步) |

### 2.2 事件属性规则

**URL 类事件** (Outbound Link, File Download):
- 前端手动设置 `props: { url: ... }`
- 后端**不自动填充**，依赖前端发送

**Path 类事件** (404):
- 如果请求中没有 `path` 属性，后端**自动**用 `pathname` 填充
- 逻辑: `lib/plausible/ingestion/request.ex:184-194`

```elixir
defp maybe_set_props_path_to_pathname(props_in_request, changeset) do
  if Plausible.Event.SystemEvents.sync_props_path_with_pathname?(
       Changeset.get_field(changeset, :event_name),
       props_in_request
     ) do
    [{"path", Changeset.get_field(changeset, :pathname)}] ++ props_in_request
  else
    props_in_request
  end
end
```

### 2.3 交互性标记

所有三个事件都被标记为 `interactive` 事件:
```elixir
@interactive_events @all_system_events
```

这意味着这些事件会被计为用户交互，影响会话统计。

---

## 三、后端映射

### 3.1 目标（Goal）创建

**模块**: `lib/plausible/goals/goals.ex`

| 函数 | 创建的目标 |
|------|-----------|
| `create_outbound_links/1` | `"Outbound Link: Click"` |
| `create_file_downloads/1` | `"File Download"` |
| `create_404/1` | `"404"` |

### 3.2 配置同步机制

**模块**: `lib/plausible_web/tracker.ex:214-230`

当 `tracker_script_configuration` 变更时，`sync_goals` 函数会自动管理目标:

```elixir
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
    # ...
  end)
end
```

**配置项与目标映射**:
| 配置项 | 对应的目标 |
|--------|-----------|
| `track_404_pages` | `"404"` |
| `outbound_links` | `"Outbound Link: Click"` |
| `file_downloads` | `"File Download"` |

### 3.3 事件接收与处理

**请求处理**: `lib/plausible/ingestion/request.ex`

事件通过 `POST /api/event` 发送，包含字段:
- `n` 或 `name`: 事件名称
- `u` 或 `url`: 页面 URL
- `p` 或 `props`: 自定义属性
- `d` 或 `domain`: 域名

**事件名称类型**: 支持字符串和整数（为了 404 追踪）
```elixir
defmodule Plausible.Ecto.EventName do
  def cast(val) when is_binary(val), do: {:ok, val}
  def cast(val) when is_integer(val), do: {:ok, Integer.to_string(val)}
  # Accepting integers is important for 404 tracking.
end
```

---

## 四、冲突处理与边界情况

### 4.1 事件优先级

**检查顺序** (`tracker/src/custom-events.js:84-109`):

```javascript
function handleLinkClickEvent(event) {
  // 1. 最高优先级: Tagged Events
  if (COMPILE_TAGGED_EVENTS) {
    if (isElementOrParentTagged(link, 0)) {
      return  // 直接返回，不触发其他事件
    }
  }

  // 2. 次高优先级: Outbound Links
  if (COMPILE_OUTBOUND_LINKS && config.outboundLinks) {
    if (isOutboundLink(link)) {
      return sendLinkClickEvent(...)
    }
  }

  // 3. 最低优先级: File Downloads
  if (COMPILE_FILE_DOWNLOADS && config.fileDownloads) {
    if (isDownloadToTrack(hrefWithoutQuery)) {
      return sendLinkClickEvent(...)
    }
  }
}
```

**优先级结论**:
```
Tagged Events > Outbound Links > File Downloads
```

### 4.2 关键边界场景

#### 场景 1: 出站链接 + 文件下载（冲突）

**情况**: 链接同时满足出站链接和文件下载条件
```html
<a href="https://other-domain.com/document.pdf">下载PDF</a>
```

**结果**: 只触发 `Outbound Link: Click`，**不触发** `File Download`

**原因**: 出站链接检查在文件下载检查之前，且一旦匹配就 `return`

#### 场景 2: Tagged 元素覆盖

**情况**: 链接有 `plausible-event-name` class
```html
<a href="https://other-domain.com/" class="plausible-event-name=MyCustomEvent">点击</a>
```

**结果**: 只触发自定义事件 `MyCustomEvent`，**不触发** `Outbound Link: Click`

**原因**: Tagged Events 优先级最高，匹配后直接 `return`

#### 场景 3: SVG 内的链接

**测试验证**: `tracker/test/outbound-links.spec.ts:246-293`

```html
<svg viewBox="0 0 100 100">
  <a href="https://other-domain.com/">
    <circle cx="50" cy="50" r="50" />
  </a>
</svg>
```

**结果**: **不追踪** SVG 内的链接

**原因**: `getLinkEl` 函数通过 `tagName` 检查，但 SVG 命名空间内的元素行为不同

#### 场景 4: 中键/新标签页打开

**行为**:
- 左键点击（正常）: 追踪事件
- Ctrl/Meta + 左键: 追踪事件，**不拦截**导航
- 中键点击 (`auxclick`): 追踪事件，**不拦截**导航
- `target="_blank"`: 追踪事件，**不拦截**导航

**拦截条件** (`tracker/src/custom-events.js:53-73`):
```javascript
function shouldInterceptNavigation(event, link) {
  // 不拦截: event.defaultPrevented
  // 不拦截: target 不是 _self/_parent/_top
  // 不拦截: ctrlKey || metaKey || shiftKey
  // 不拦截: event.type !== 'click'
}
```

#### 场景 5: 链接被其他脚本阻止

**情况**: 链接有 `onclick="event.preventDefault()"`

**结果**: **不追踪**该链接

**原因**: `event.defaultPrevented` 检查优先

#### 场景 6: 子元素点击

**情况**: 点击链接内的子元素
```html
<a href="https://other-domain.com/">
  <span>点击这里</span>
</a>
```

**结果**: **正确追踪**

**原因**: `getLinkEl` 函数会向上遍历父节点查找 `<a>` 标签
```javascript
function getLinkEl(link) {
  while (
    link &&
    (typeof link.tagName === 'undefined' || !isLink(link) || !link.href)
  ) {
    link = link.parentNode
  }
  return link
}
```

### 4.3 Compat 模式 vs 普通模式

**导航处理差异**:

| 模式 | 事件发送方式 | 导航延迟 | 超时保护 |
|------|-------------|---------|---------|
| 普通模式 | `fetch keepalive` | 无 | 无 |
| Compat 模式 | 同步等待 + 回调 | 最多 5 秒 | 5 秒超时强制导航 |

**Compat 模式逻辑** (`tracker/src/custom-events.js:111-146`):
```javascript
if (COMPILE_COMPAT) {
  if (shouldInterceptNavigation(event, link)) {
    event.preventDefault()  // 阻止默认导航
    track(eventAttrs.name, { callback: followLink })
    setTimeout(followLink, 5000)  // 5 秒超时保护
  }
}
```

---

## 五、总结与边界矩阵

### 5.1 实现边界对比表

| 特性 | Outbound Link | File Download | 404 |
|------|--------------|---------------|-----|
| **前端自动识别** | ✅ 是 | ✅ 是 | ❌ 否（需手动） |
| **识别依据** | `link.host !== location.host` | 文件扩展名 | 无 |
| **事件名称** | `"Outbound Link: Click"` | `"File Download"` | `"404"` |
| **默认属性** | `props.url` (完整 href) | `props.url` (无查询参数) | 无，后端自动填充 |
| **可配置性** | 开关 | 开关 + 自定义文件类型 | 配置项存在但未使用 |
| **目标自动创建** | ✅ 是 | ✅ 是 | ✅ 是 |
| **interactive** | ✅ 是 | ✅ 是 | ✅ 是 |

### 5.2 冲突决策树

```
用户点击 <a> 标签
         │
         ▼
┌─────────────────────────┐
│ 是 Tagged 元素?         │
│ (plausible-event-name) │
└───────────┬─────────────┘
            │
     ┌──────┴──────┐
     │             │
   是 ✅          否 ❌
     │             │
     ▼             ▼
  触发自定义    ┌─────────────────┐
  事件，返回    │ 是出站链接?     │
               │ (host 不同)     │
               └────────┬────────┘
                        │
                 ┌──────┴──────┐
                 │             │
               是 ✅          否 ❌
                 │             │
                 ▼             ▼
            触发 Outbound  ┌─────────────────┐
            Link: Click   │ 是文件下载?     │
            返回          │ (扩展名匹配)    │
                          └────────┬────────┘
                                   │
                            ┌──────┴──────┐
                            │             │
                          是 ✅          否 ❌
                            │             │
                            ▼             ▼
                       触发 File      无事件
                       Download
                       返回
```

### 5.3 已知限制

1. **出站文件下载**: 出站链接上的文件下载只会被计为出站链接，不会被计为文件下载
2. **SVG 链接**: SVG 命名空间内的链接无法被追踪
3. **404 自动追踪**: 配置项存在但前端未实现，需要手动集成
4. **查询参数**: 文件下载的 URL 会去除查询参数，而出站链接保留完整 URL

---

## 六、相关文件索引

| 功能 | 文件路径 | 关键行号 |
|------|----------|---------|
| 前端事件识别 | `tracker/src/custom-events.js` | 75-109, 148-166 |
| 系统事件定义 | `lib/plausible/event/system_events.ex` | 1-101 |
| 目标管理 | `lib/plausible/goals/goals.ex` | 296-362 |
| 配置同步 | `lib/plausible_web/tracker.ex` | 214-230 |
| 请求处理 | `lib/plausible/ingestion/request.ex` | 184-194 |
| 出站链接测试 | `tracker/test/outbound-links.spec.ts` | 全部 |
| 文件下载测试 | `tracker/test/file-downloads.spec.ts` | 全部 |
