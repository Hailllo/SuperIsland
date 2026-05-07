# SuperIsland 扩展 SDK — JavaScript API

## 背景

SuperIsland 是一个原生 macOS 应用（Swift/SwiftUI，macOS 14+），用于将 MacBook 刘海区域变成可交互的 Super Island。当前内置模块包括正在播放、音量 HUD、电池、连接、日历、天气、通知、Shelf、提词器等。

我们希望 SuperIsland 具备可 hack、可扩展的能力，让社区可以像 Raycast 扩展一样，使用 JavaScript / TypeScript 构建、分发和安装第三方扩展。扩展运行在 JavaScriptCore 沙盒中，并用声明式方式描述 UI；宿主应用会用 SwiftUI 原生渲染这些 UI。

当前仓库结构：

- `ExtensionHost/`：macOS 应用侧的扩展运行时
- `Extensions/`：开发期间可发现的本地 / 示例扩展

---

## 目标

构建基于 JavaScript 的扩展 SDK。扩展是简单的 JS/TS 项目，可以提供：

- **紧凑视图**：适配胶囊区域，约 188×34 pt
- **极简刘海视图**：用于带刘海 Mac，中心隐藏在硬件刘海中，两侧分别显示 leading / trailing 内容
- **展开视图**：抽屉区域，约 360×80 pt
- **全展开视图**：详情面板，约 400×200 pt
- **后台服务**：可轮询、监听或计算数据
- **设置 UI**：通过声明式 schema 自动渲染

社区可以构建的扩展示例：

- 番茄钟 / 专注计时器
- 歌词叠层
- CPU / RAM / 磁盘监控
- 股票 / 加密货币价格
- GitHub 通知 / PR 状态
- 剪贴板历史
- 快捷启动器
- 会议倒计时
- 习惯追踪
- 空气质量 / 紫外线指数
- 快递追踪
- 智能家居设备控制
- Tailscale / VPN 状态

---

## 当前架构参考

### 状态机

```text
IslandState: .compact (188×34) → .expanded (360×80) → .fullExpanded (400×200)
```

### 模块模式

每个内置模块通常遵循以下结构：

1. **Manager**：`ObservableObject` 单例，持有 `@Published` 状态并运行后台逻辑
2. **CompactView**：胶囊区域中的 SwiftUI 视图
3. **ExpandedView**：展开抽屉中的 SwiftUI 视图
4. **Registration**：在 `ModuleType` 中注册，并在 `CompactView.swift`、`ExpandedView.swift`、`FullExpandedView.swift` 等处路由

### 激活方式

- `AppState.shared.setActiveModule(.module)`：设置当前模块
- `AppState.shared.showHUD(module:, autoDismiss:)`：显示模块，并在延迟后自动收起
- `AppState.shared.cycleModule(forward:)`：左右滑动切换模块

### 尺寸与渲染

- 岛窗口是透明 `NSPanel`，使用 `.nonactivatingPanel`、`canBecomeKey=false`、`.statusBar` level
- 面板始终保持最大尺寸，视觉尺寸由 SwiftUI 内容控制
- `PillShape` 在状态切换时动画调整圆角
- 转场使用 spring 动画

---

## 1. 扩展包格式

扩展是普通目录，简单扩展不需要编译：

```text
pomodoro/
├── manifest.json          # 扩展元数据和权限
├── index.js               # 主入口，或由 index.ts 编译得到
├── assets/
│   ├── icon.png           # 64×64 扩展图标
│   └── icon-compact.png   # 紧凑视图图标，可选
└── settings.json          # 设置 schema，可选
```

发布扩展时，可以打包为 `.zip`。如果使用 TypeScript，应在打包前编译为 JavaScript。

---

## 2. manifest.json

```json
{
  "id": "com.developer.pomodoro",
  "name": "Pomodoro Timer",
  "version": "1.0.0",
  "minAppVersion": "1.0.0",
  "main": "index.js",
  "author": {
    "name": "Jane Developer",
    "url": "https://github.com/jane/pomodoro-island"
  },
  "description": "Focus timer with Pomodoro technique. Shows countdown in compact view, controls in expanded.",
  "icon": "assets/icon.png",
  "license": "MIT",
  "categories": ["productivity", "timer"],
  "permissions": ["notifications", "storage"],
  "capabilities": {
    "compact": true,
    "expanded": true,
    "fullExpanded": true,
    "minimalCompact": true,
    "backgroundRefresh": true,
    "settings": true,
    "notificationFeed": false
  },
  "refreshInterval": 1.0,
  "activationTriggers": ["manual", "timer"]
}
```

当前支持的权限：

- `notifications`：发送 macOS 通知，并读取由 `SuperIsland.system` 暴露的镜像通知流
- `storage`：持久化扩展级键值状态
- `network`：通过 `SuperIsland.http.fetch()` 发起请求
- `media`：通过 `SuperIsland.system.getNowPlaying()` 读取宿主归一化后的正在播放信息
- `usage`：通过 `SuperIsland.system.getAIUsage()` 读取本地 Codex 和 Claude 使用量摘要

`capabilities.notificationFeed`：

- 为 `true` 时，扩展不会作为独立模块出现在岛的模块轮播中。
- `SuperIsland.island.activate()` 会打开共享通知模块。
- `SuperIsland.notifications.send(...)` 会写入共享的 SuperIsland 通知流。

---

## 3. JavaScriptCore Bridge

宿主应用会为每个扩展创建一个 `JSContext`。在加载扩展的 `index.js` 前，会注入全局对象 `SuperIsland`。

扩展通过以下方式注册模块：

```js
SuperIsland.registerModule({
  compact: function() {
    return View.hstack([
      View.icon("bolt.fill", { color: "yellow" }),
      View.text("Hello", { style: "body" })
    ]);
  },

  expanded: function() {
    return View.text("Expanded view", { style: "title" });
  },

  onAction: function(actionID, value) {
    console.log("Action:", actionID, value);
  }
});
```

---

## 4. 运行时生命周期

一个扩展只注册一个模块：

```js
SuperIsland.registerModule({
  compact,
  expanded,
  fullExpanded,
  minimalCompact,
  onActivate,
  onDeactivate,
  onAction
});
```

### 生命周期回调

- `onActivate()`：runtime 激活时调用
- `onDeactivate()`：runtime 销毁前调用
- `onAction(actionID, value?)`：用户与扩展 UI 交互时调用

### 渲染回调

- `compact()`：渲染到紧凑岛
- `expanded()`：渲染到展开岛
- `fullExpanded()`：渲染到全展开面板；如果缺失，会复用 expanded
- `minimalCompact.leading()` / `minimalCompact.trailing()`：在带刘海 Mac 的紧凑变体中渲染
- `minimalCompact.precedence`：可选数字或回调。`1` 与媒体优先级相同，`2` 表示媒体活跃时音乐优先

---

## 5. SuperIsland 全局 API

### `SuperIsland.registerModule(config)`

注册扩展模块配置。

### `SuperIsland.island`

- `activate(autoDismiss = true)`
- `dismiss()`
- `state`: `"compact" | "expanded" | "fullExpanded"`
- `isActive`: boolean

如果 `manifest.capabilities.notificationFeed` 为 `true`，`activate()` 会打开主通知模块，而不是扩展自己的岛位。

后台计时器中推荐这样唤起岛：

```js
function revealIsland() {
  SuperIsland.island.activate(false);
  setTimeout(function() {
    SuperIsland.island.activate(false);
  }, 120);
}
```

### `SuperIsland.store`

扩展级持久化键值存储。

- `get(key)`：返回值或 `null`
- `set(key, value)`：写入值；传入 `null` 会清除该 key

支持标量、JSON 兼容对象和数组。

### `SuperIsland.settings`

扩展设置键值存储，通常与 `settings.json` 配合使用。

- `get(key)`：返回值或 `null`
- `set(key, value)`：写入设置值

### `SuperIsland.notifications`

- `send(options)`
  - `title`：字符串
  - `body`：字符串
  - `sound?`：布尔值
  - `id?`：稳定 ID，用于通知去重 / 更新
  - `appName?`：通知栏中显示的应用名
  - `bundleIdentifier?`：用于 app icon fallback 的 bundle ID
  - `senderName?`：联系人 / 发送者名称
  - `previewText?`：消息 / 内容预览
  - `avatarURL?`：发送者头像，支持 `file://`、绝对路径、`http(s)://`
  - `appIconURL?`：扩展 / 应用图标，支持 `file://`、绝对路径、`http(s)://`
  - `systemNotification?`：默认 `true`；为 `false` 时只更新 SuperIsland 通知流

对于 `capabilities.notificationFeed: true` 的扩展，发送的通知会镜像到共享通知模块。

### `SuperIsland.http`

- `fetch(url, options?) -> Promise<{ status, data, text, error? }>`

`options`：

- `method`：默认 GET
- `headers`：`Record<string, string>`
- `body`：字符串

需要 `network` 权限。没有权限时，`fetch` 会抛错。网络错误会返回带 `error` 字段的 resolved object。

### `SuperIsland.system`

- `getAIUsage()`：返回使用量对象或 `null`
- `getNowPlaying()`：返回归一化后的正在播放快照，或 `null`
- `getLatestNotification()`：返回最新镜像通知，或 `null`
- `getRecentNotifications(limit?)`：返回镜像通知数组，按时间倒序

权限要求：

- `getAIUsage()` 需要 `usage`
- `getNowPlaying()` 需要 `media`
- 通知相关 API 需要 `notifications`

`getNowPlaying()` 返回结构大致为：

```js
{
  sourceApp,
  bundleIdentifier,
  title,
  artist,
  album,
  albumArtist,
  durationSeconds,
  elapsedSeconds,
  artworkURL,
  playbackState,
  trackIdentifier,
  isLocalFile,
  capturedAtEpochMs
}
```

通知 API 返回结构大致为：

```js
{
  id,
  localID,
  appName,
  bundleIdentifier,
  appIcon,
  appIconURL,
  title,
  body,
  senderName,
  previewText,
  avatarURL,
  timestamp
}
```

### `SuperIsland.playFeedback(type)`

支持：

- `"success"`
- `"warning"`
- `"error"`
- `"selection"`

### `SuperIsland.openURL(url)`

使用默认浏览器打开 URL。

### `SuperIsland.mascot`

控制用户在设置中配置的共享吉祥物。

- `setExpression(expression)`：设置当前表情
  - `"idle"`：默认中性状态
  - `"working"`：专注 / 工作中
  - `"alert"`：需要注意
  - `"happy"`：积极 / 庆祝
  - `"tired"`：低能量 / 暂停
  - `"clicked"`：用户交互反馈，会自动回到 idle
- `getExpression()`：返回当前表情字符串
- `getSelected()`：返回用户选择的吉祥物 `{ id, name, previewIcon }`
- `list()`：返回所有可用吉祥物数组
- `setInput(name, value)`：设置状态机输入，便于后续兼容 masko-code 风格输入

注意：吉祥物是共享资源，所有扩展会写入同一个表情状态。使用 `View.mascot()` 可在扩展 UI 中渲染吉祥物。

---

## 6. 全局 Timer 和 Console API

宿主会注入：

- `setTimeout(callback, ms)`
- `clearTimeout(id)`
- `setInterval(callback, ms)`
- `clearInterval(id)`
- `console.log(...)`
- `console.error(...)`

建议优先使用 `onRefresh()` 处理周期性轮询，避免扩展自行创建过多定时器。

---

## 7. View API 概览

扩展 UI 使用声明式 `View` 树。常见构建器包括：

- `View.text(value, options)`
- `View.icon(name, options)`
- `View.hstack(children, options)`
- `View.vstack(children, options)`
- `View.zstack(children, options)`
- `View.spacer()`
- `View.button(content, actionID)`
- `View.progress(value, options)`
- `View.circularProgress(value, options)`
- `View.mascot(options)`
- `View.frame(content, options)`
- `View.background(content, color)`
- `View.cornerRadius(content, radius)`
- `View.animate(content, animation)`

具体支持项以 `ExtensionHost/ExtensionViewNode.swift` 和 `ExtensionHost/ExtensionRendererView.swift` 的当前实现为准。

---

## 8. 建议

- 紧凑视图空间很小，只展示最关键的信息。
- 需要跨重启保留的状态使用 `SuperIsland.store`。
- 设置项使用 `settings.json` 声明，让宿主自动渲染原生设置 UI。
- 需要访问网络、媒体、通知或使用量数据时，在 `manifest.json` 中声明对应权限。
- 扩展日志会进入宿主扩展日志面板，开发时也可在 Xcode 输出中查看。
