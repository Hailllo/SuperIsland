# DPI：SuperIsland 插件接口（开发者指南）

本文档记录 SuperIsland 当前已暴露的扩展 API，方便开发者快速构建可运行的扩展。

如果你想确认当前 runtime 支持什么能力，请以本文档为准。

## 1. 快速开始

1. 在 `Extensions/` 下创建目录，例如 `Extensions/my-timer/`。
2. 添加 `manifest.json`。
3. 添加 `index.js`，并在其中调用一次 `SuperIsland.registerModule(...)`。
4. 可选：添加 `settings.json`，用于原生设置 UI。
5. 启动应用，打开 **设置 → 扩展**，然后启用你的扩展。

最小示例：

```js
SuperIsland.registerModule({
  compact() {
    return View.hstack([
      View.icon("bolt.fill", { color: "yellow" }),
      View.text("Hello", { style: "body" })
    ]);
  },
  expanded() {
    return View.text("Expanded view", { style: "title" });
  },
  onAction(actionID, value) {
    console.log("Action:", actionID, value);
  }
});
```

## 2. 扩展包结构

```text
my-extension/
  manifest.json
  index.js
  settings.json         (可选)
  assets/               (可选)
```

## 3. manifest.json 参考

必需字段：

- `id`：字符串，全局唯一 ID，例如 `com.you.extension`
- `name`：字符串
- `version`：字符串

常用字段：

- `main`：字符串，默认 `index.js`
- `minAppVersion`：字符串，默认 `1.0.0`
- `description`：字符串，默认 `""`
- `author`：`{ name, url? }`
- `icon`：图标路径字符串
- `license`：字符串
- `categories`：字符串数组
- `permissions`：字符串数组
- `capabilities`：
  - `compact`：默认 `true`
  - `expanded`：默认 `true`
  - `fullExpanded`：默认 `true`
  - `minimalCompact`：默认 `false`
  - `backgroundRefresh`：默认 `true`
  - `settings`：默认 `true`
  - `notificationFeed`：默认 `false`；扩展会隐藏于模块槽位之外，`SuperIsland.island.activate()` 会打开共享通知模块
- `refreshInterval`：秒数，默认 `1.0`，最小 `0.1`
- `activationTriggers`：默认 `["manual"]`

示例：

```json
{
  "id": "superisland.pomodoro",
  "name": "Pomodoro Timer",
  "version": "1.0.0",
  "main": "index.js",
  "permissions": ["notifications", "storage"],
  "refreshInterval": 1.0
}
```

## 4. Runtime 生命周期

扩展只注册一个模块：

```js
SuperIsland.registerModule({
  compact,                   // 必需
  expanded,                  // 推荐
  fullExpanded,              // 可选
  minimalCompact,            // 可选
  onActivate,                // 可选
  onDeactivate,              // 可选
  onAction                   // 可选
});
```

### 生命周期回调

- `onActivate()`：runtime 激活时运行。
- `onDeactivate()`：runtime 销毁前运行。
- `onAction(actionID, value?)`：UI 交互时运行。

### 渲染回调

- `compact()`：渲染到紧凑岛。
- `expanded()`：渲染到展开岛。
- `fullExpanded()`：渲染到全展开面板；如果缺失，则复用 expanded。
- `minimalCompact.leading()` / `minimalCompact.trailing()`：在带刘海 Mac 的紧凑布局中渲染。
- `minimalCompact.precedence`：可选数字或回调。`1` 匹配媒体优先级，`2` 表示媒体活跃时让音乐优先。

## 5. SuperIsland 全局 API

### `SuperIsland.registerModule(config)`

注册扩展模块配置。

### `SuperIsland.island`

- `activate(autoDismiss = true)`
- `dismiss()`
- `state`: `"compact" | "expanded" | "fullExpanded"`
- `isActive`: boolean

说明：如果 `manifest.capabilities.notificationFeed` 为 `true`，`activate()` 会打开主通知模块，而不是扩展自己的岛位。

适合定时器 / 后台任务的唤起方式：

```js
function revealIsland() {
  SuperIsland.island.activate(false);
  setTimeout(() => SuperIsland.island.activate(false), 120);
}
```

### `SuperIsland.store`

扩展级持久化键值存储。

- `get(key)`：返回值或 `null`
- `set(key, value)`：写入值

说明：

- `null` 会清除 / 移除 key。
- 支持标量以及 JSON 兼容对象 / 数组。

### `SuperIsland.settings`

扩展设置键值存储，与 `settings.json` 配合使用。

- `get(key)`：返回值或 `null`
- `set(key, value)`：写入值

### `SuperIsland.notifications`

- `send(options)`
  - `title`：字符串
  - `body`：字符串
  - `sound?`：布尔值
  - `id?`：稳定 ID，用于通知流去重 / 更新
  - `appName?`：通知栏中显示的应用名
  - `bundleIdentifier?`：用于 app 图标 fallback 的 bundle ID
  - `senderName?`：发送者 / 联系人名称
  - `previewText?`：消息 / 内容预览
  - `avatarURL?`：发送者头像，支持 `file://`、绝对路径和 `http(s)://`
  - `appIconURL?`：扩展 / 应用图标，支持 `file://`、绝对路径和 `http(s)://`
  - `systemNotification?`：默认 `true`；为 `false` 时只更新 SuperIsland 通知流

说明：

- 对于 `capabilities.notificationFeed: true` 的扩展，发送的通知会镜像到共享 SuperIsland 通知流。

### `SuperIsland.http`

- `fetch(url, options?) -> Promise<{ status, data, text, error? }>`

`options`：

- `method`：默认 GET
- `headers`：`Record<string, string>`
- `body`：字符串

说明：

- 需要 `"network"` 权限。
- 没有权限时，`fetch` 会抛错。
- 网络错误会返回带 `error` 字段的 resolved object。

### `SuperIsland.system`

- `getAIUsage()`：返回使用量对象或 `null`
- `getNowPlaying()`：返回归一化后的正在播放快照或 `null`
- `getLatestNotification()`：返回最新镜像通知对象或 `null`
- `getRecentNotifications(limit?)`：返回镜像通知数组，最新在前

说明：

- `getAIUsage()` 需要 `"usage"` 权限。
- `getNowPlaying()` 需要 `"media"` 权限。
- 镜像通知 API 需要 `"notifications"` 权限。
- `getNowPlaying()` 返回：
  - `{ sourceApp, bundleIdentifier, title, artist, album, albumArtist, durationSeconds, elapsedSeconds, artworkURL, playbackState, trackIdentifier, isLocalFile, capturedAtEpochMs }`
  - `artworkURL`、`album`、`albumArtist`、`trackIdentifier` 是可选字段，可能为 `null`
- Codex 数据源优先级：本地 summary 文件，然后通过 `~/.codex/auth.json` token 调用 ChatGPT OAuth 使用量 API。
- Claude 数据源优先级：本地 summary 文件，然后 Claude OAuth 使用量 API，再到本地 stats cache fallback。
- `codex.source` 和 `claude.source` 表示 payload 来源，例如 `local-summary`、`oauth-api`、`auth-token`、`stats-cache`、`unavailable`。
- 镜像通知 API 返回结构：
  - `{ id, localID, appName, bundleIdentifier, appIcon, appIconURL, title, body, senderName, previewText, avatarURL, timestamp }`
  - `previewText` / `avatarURL` 取决于 macOS 对该通知暴露了什么，隐私设置可能隐藏预览。

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

- `setExpression(expression)`：设置当前表情。支持：
  - `"idle"`：默认中性状态
  - `"working"`：活跃 / 专注状态
  - `"alert"`：需要注意
  - `"happy"`：积极 / 庆祝
  - `"tired"`：低能量 / 暂停
  - `"clicked"`：用户交互反馈，会自动重置为 idle
- `getExpression()`：返回当前表情字符串。
- `getSelected()`：返回用户选择的吉祥物 `{ id, name, previewIcon }`。
- `list()`：返回所有可用吉祥物数组：`[{ id, name, previewIcon }]`。
- `setInput(name, value)`：设置状态机输入，便于后续兼容 masko-code 风格输入。

说明：

- 吉祥物是共享资源，所有扩展都会写入同一个表情状态。
- 用户可在 **设置 → 通用 → 吉祥物** 中选择吉祥物。
- 使用 `View.mascot()` 可在扩展 UI 中渲染吉祥物。

## 6. 全局 Timer 和 Console API

宿主会注入：

- `setTimeout(callback, ms)`
- `clearTimeout(id)`
- `setInterval(callback, ms)`
- `clearInterval(id)`
- `console.log(...)`
- `console.error(...)`

## 7. 设置 schema

`settings.json` 使用声明式 schema，让宿主渲染原生设置 UI。

常见字段类型：

| type | 说明 |
|---|---|
| `text` | 文本输入 |
| `toggle` | 开关 |
| `slider` | 滑块 |
| `picker` | 下拉 / 分段选项 |
| `button` | 按钮 |

## 8. 建议

- 紧凑视图保持简洁。
- 需要持久化的状态使用 `SuperIsland.store`。
- 需要用户配置的内容使用 `settings.json`。
- 在 `manifest.json` 中声明最小必要权限。
- 开发时通过 Xcode 输出和扩展日志面板查看日志。
