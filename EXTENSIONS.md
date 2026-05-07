# SuperIsland 扩展

扩展是运行在 SuperIsland 沙盒运行时中的 JavaScript 包。扩展可以在紧凑胶囊、展开抽屉和全展开详情面板中渲染 UI，也可以运行后台逻辑来获取或计算数据。

---

## 扩展结构

```text
your-extension/
├── manifest.json       # 必需：元数据和能力声明
├── index.js            # 必需：扩展逻辑
├── settings.json       # 可选：设置 schema
└── assets/
    └── icon.svg        # 可选：显示在扩展列表中的图标
```

将扩展目录放入 `Extensions/`，运行应用时会被自动发现。

---

## manifest.json

```json
{
  "id": "superisland.your-extension",
  "name": "My Extension",
  "version": "1.0.0",
  "minAppVersion": "1.0.0",
  "main": "index.js",
  "author": {
    "name": "Your Name",
    "url": "https://github.com/yourname"
  },
  "description": "One sentence about what this does.",
  "icon": "assets/icon.svg",
  "license": "MIT",
  "categories": ["productivity"],
  "permissions": ["storage", "network", "media"],
  "capabilities": {
    "compact": true,
    "expanded": true,
    "fullExpanded": false,
    "backgroundRefresh": true,
    "settings": true
  },
  "refreshInterval": 5.0
}
```

### Permissions

| 权限 | 可用能力 |
|---|---|
| `storage` | 使用 `SuperIsland.store` 做扩展级键值持久化 |
| `network` | 使用 `SuperIsland.http.fetch()` 发起网络请求 |
| `media` | 使用 `SuperIsland.system.getNowPlaying()` 读取当前播放信息 |
| `notifications` | 发送 macOS 通知，并写入 SuperIsland 通知流 |
| `usage` | 读取 Codex / Claude 本地使用量摘要 |

### Capabilities

| Key | 说明 |
|---|---|
| `compact` | 在紧凑胶囊中渲染，约 188×34 pt |
| `expanded` | 在展开抽屉中渲染，约 360×80 pt |
| `fullExpanded` | 在全展开详情面板中渲染，约 400×200 pt |
| `minimalCompact` | 在带刘海 Mac 的紧凑布局中渲染左右两侧内容 |
| `backgroundRefresh` | 按 `refreshInterval` 调用 `onRefresh()` |
| `settings` | 读取 `settings.json`，并在设置页中展示 |
| `notificationFeed` | 作为通知流扩展，不参与模块轮播 |

---

## index.js API 简介

扩展脚本运行前，宿主会注入全局对象 `SuperIsland`。

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

### 生命周期

- `onActivate()`：扩展 runtime 激活时调用。
- `onDeactivate()`：扩展 runtime 销毁前调用。
- `onRefresh()`：当 `backgroundRefresh` 为 `true` 时，按 `refreshInterval` 周期调用。
- `onAction(actionID, value)`：用户点击按钮、切换开关或拖动控件时调用。
- `onSettingsChanged(key, value)`：用户修改扩展设置时调用。

### 常用 API

```js
SuperIsland.store.set("key", "value");
SuperIsland.store.get("key");

SuperIsland.settings.get("myKey");

SuperIsland.http.fetch("https://api.example.com/data")
  .then(function(response) {
    var data = JSON.parse(response.body);
  });

SuperIsland.notifications.send({
  title: "Time's up",
  body: "Take a break."
});

var snapshot = SuperIsland.system.getNowPlaying();

SuperIsland.island.activate();
SuperIsland.island.dismiss();
```

---

## 完整示例：股票价格扩展

```js
"use strict";

var symbol = "AAPL";
var price = "--";
var change = "--";

function onInit() {
  symbol = SuperIsland.settings.get("symbol") || "AAPL";
  render();
}

function onRefresh() {
  SuperIsland.http.fetch("https://query1.finance.yahoo.com/v8/finance/quote?symbols=" + symbol)
    .then(function(res) {
      var data = JSON.parse(res.body);
      var quote = data.quoteResponse.result[0];
      price  = "$" + quote.regularMarketPrice.toFixed(2);
      change = (quote.regularMarketChangePercent >= 0 ? "+" : "") +
               quote.regularMarketChangePercent.toFixed(2) + "%";
      render();
    });
}

function onSettingsChanged(key, value) {
  if (key === "symbol") {
    symbol = value;
    onRefresh();
  }
}

function render() {
  SuperIsland.island.setCompactView({
    left:   { type: "text", value: symbol },
    center: { type: "text", value: price  },
    right:  { type: "text", value: change }
  });

  SuperIsland.island.setExpandedView({
    rows: [
      { type: "text", value: symbol + "  " + price, style: "title" },
      { type: "text", value: "Change: " + change, style: "subtitle" }
    ]
  });
}

SuperIsland.extension.onInit(onInit);
SuperIsland.extension.onRefresh(onRefresh);
SuperIsland.extension.onSettingsChanged(onSettingsChanged);
```

对应的 `settings.json`：

```json
{
  "sections": [
    {
      "title": "Stock",
      "fields": [
        {
          "type": "text",
          "key": "symbol",
          "label": "Ticker symbol",
          "placeholder": "AAPL",
          "default": "AAPL"
        }
      ]
    }
  ]
}
```

---

## Settings schema 字段类型

| type | 选项 |
|---|---|
| `text` | `key`, `label`, `placeholder`, `default` |
| `toggle` | `key`, `label`, `default`，布尔值 |
| `slider` | `key`, `label`, `min`, `max`, `step`, `default` |
| `picker` | `key`, `label`, `options`，数组项为 `{label, value}`，以及 `default` |
| `button` | `key`, `label`, `action`，`action` 可选，默认等于 `key` |

---

## 建议

- 保持紧凑视图简洁，胶囊区域很小，每个位置只显示一个关键信息。
- 需要跨应用重启保留的状态，请使用 `SuperIsland.store` 持久化。
- 需要轮询数据时使用 `onRefresh`。避免直接使用 `setInterval`，调度应交给 runtime 控制。
- 开发时在 Xcode 中运行应用，扩展的 console 日志会显示在 Xcode 输出中。
