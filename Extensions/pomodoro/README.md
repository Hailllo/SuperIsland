# 番茄钟扩展

这是一个独立的 SuperIsland 扩展，仅使用 `SuperIsland` 和 `View` JavaScript API 实现。

## 文件

- `manifest.json`：扩展元数据和权限声明
- `index.js`：扩展逻辑和 UI，包括紧凑、极简、展开和全展开视图
- `settings.json`：声明式设置 schema，由宿主应用渲染为原生设置界面

## 说明

- 不依赖主应用代码库中的 Swift 实现。
- 可以复制到任何兼容 SuperIsland 的扩展宿主中运行。
- 设计上支持后续独立打包和分发，例如 zip、Git release 或 registry。
