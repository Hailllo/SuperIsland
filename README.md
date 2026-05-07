<p align="center">
  <img src="assets/logo.png" width="96" height="96" alt="SuperIsland" />
</p>

<h1 align="center">SuperIsland</h1>

<p align="center">
  将 Mac 的刘海区域变成实时、可交互的灵动岛。<br />
  正在播放 · 电池 · 天气 · 日历 · 通知 · 扩展
</p>

<p align="center">
  中文 · <a href="README.en.md">English</a>
</p>

---

## 项目说明

SuperIsland 是一个 macOS 桌面应用，用于把 Mac 顶部刘海区域变成类似 Dynamic Island 的交互区域，显示系统状态、媒体播放、通知和扩展内容。

本仓库 fork 自 [`shobhit99/SuperIsland`](https://github.com/shobhit99/SuperIsland)，并在此基础上进行了本地化和功能调整。

## 环境要求

- macOS 14 Sonoma 或更高版本
- Xcode 15+
- XcodeGen：`brew install xcodegen`
- Node.js 18+：仅在开发部分扩展时需要

## 开发运行

```bash
git clone https://github.com/Hailllo/SuperIsland.git
cd SuperIsland
xcodegen generate
open SuperIsland.xcodeproj
```

在 Xcode 中选择 `SuperIsland` scheme，运行目标选择当前 Mac，然后点击 Run 或按 `⌘R`。

首次启动时，应用会请求辅助功能、日历、定位等权限。相关模块需要这些权限才能正常工作。

## 构建发布 DMG

发布构建需要 Developer ID 证书和 notarization 凭据。复制 `.env.template` 为 `.env`，并填写：

```env
APPLE_ID=you@example.com
APP_SPECIFIC_PASSWORD=xxxx-xxxx-xxxx-xxxx
TEAM_ID=XXXXXXXXXX
SIGNING_IDENTITY=Developer ID Application: Your Name (TEAMID)
```

然后运行：

```bash
./scripts/build-and-release.sh
```

该脚本会 archive、export、notarize，并生成签名后的 `build/SuperIsland.dmg`。

快速生成本地未签名 DMG：

```bash
./scripts/build-dmg.sh
```

## 项目结构

```text
SuperIsland/
  App/              AppDelegate、AppState、应用入口
  Modules/          内置模块，例如电池、正在播放、天气等
  Settings/         设置窗口视图
  Utilities/        更新、权限、启动项等工具
  Views/            紧凑、展开、全展开等岛内视图
ExtensionHost/      JavaScript 扩展运行时、扩展管理器和桥接层
Extensions/         内置扩展
scripts/            构建和发布脚本
```

## 扩展

扩展是运行在沙盒 JavaScriptCore 环境中的 JavaScript 包。扩展开发说明可查看：

- [EXTENSIONS.md](EXTENSIONS.md)
- [EXTENSIONS-API.md](EXTENSIONS-API.md)

当前内置扩展包括：

- Pomodoro 番茄钟
- AI Usage Rings / AI 使用量圆环
- Agents Status / Agent 状态

## 更新

SuperIsland 启动时会自动检查更新。当发现新版本时，会显示更新弹窗，点击 **更新** 即可下载安装。

当前更新检查使用本仓库的 GitHub Releases。

## 贡献

贡献说明请查看 [CONTRIBUTING.md](CONTRIBUTING.md)。
