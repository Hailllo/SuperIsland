# SuperIsland 项目分析

## 1. 项目定位

SuperIsland 是一个 macOS SwiftUI 桌面应用，目标是把 Mac 顶部刘海区域变成一个类似 Dynamic Island 的实时交互区域。

核心能力包括：

- 在 Mac 刘海区域显示可交互浮窗
- 支持 compact、expanded、fullExpanded 三种展示状态
- 内置 Now Playing、电池、天气、日历、通知、音量 HUD、网络、提词器、Shelf 等模块
- 支持 JavaScript 扩展系统
- 支持自动更新、菜单栏控制、开机启动、权限引导和 onboarding

运行要求：

- macOS 14 Sonoma 或更高版本
- Xcode 15+
- Swift 5.9
- XcodeGen
- Node.js 18+，仅开发扩展时需要

## 2. 技术栈

主要技术组成：

- SwiftUI + AppKit：主应用、设置页、刘海浮窗和窗口控制
- Combine：全局状态订阅和响应式更新
- JavaScriptCore / WebKit：扩展运行与扩展 UI 渲染相关能力
- XcodeGen：通过 `project.yml` 生成 Xcode 工程
- Aptabase Swift SDK：应用埋点分析
- Node.js / esbuild：用于部分扩展的构建，例如 WhatsApp provider

项目不是通过 `Package.swift` 管理主工程，而是通过 `project.yml` 声明 Xcode 工程结构。

## 3. 目录结构

关键目录如下：

```text
SuperIsland/
  App/              应用入口、AppDelegate、全局状态
  Modules/          内置模块
  Settings/         设置窗口视图
  Views/            刘海岛 UI
  Window/           浮窗窗口和屏幕定位
  Utilities/        权限、更新、启动项、媒体键等工具
ExtensionHost/      JavaScript 扩展宿主
Extensions/         内置扩展
scripts/            构建、打包、发布脚本
```

内置模块主要包括：

```text
SuperIsland/Modules/Battery
SuperIsland/Modules/Calendar
SuperIsland/Modules/Connectivity
SuperIsland/Modules/Focus
SuperIsland/Modules/Notifications
SuperIsland/Modules/NowPlaying
SuperIsland/Modules/Shelf
SuperIsland/Modules/SystemHUD
SuperIsland/Modules/Teleprompter
SuperIsland/Modules/Weather
```

## 4. 启动流程

应用入口位于：

```text
SuperIsland/App/SuperIslandApp.swift
```

主入口通过 `@main` 声明 `SuperIslandApp`，并使用：

```swift
@NSApplicationDelegateAdaptor(AppDelegate.self)
```

把主要应用生命周期交给 `AppDelegate`。

启动核心流程位于：

```text
SuperIsland/App/AppDelegate.swift
```

启动过程大致如下：

1. 启动 Analytics
2. 注册 URL handler，用于处理 OAuth callback
3. 安装退出快捷键保护
4. 判断是否需要显示 onboarding
5. onboarding 完成后调用 `bootstrapApp()`
6. 创建 Island 浮窗
7. 初始化已启用的系统模块
8. 发现并激活扩展
9. 检查自动更新

关键方法：

- `applicationDidFinishLaunching`
- `bootstrapApp`
- `initializeManagers`
- `setupIslandWindow`

## 5. 全局状态模型

核心状态类是：

```text
SuperIsland/App/AppState.swift
```

`AppState` 是项目的全局 UI 状态中心，管理内容包括：

- 当前岛状态：compact、expanded、fullExpanded
- 当前 active module
- hover 展开逻辑
- 自动 dismiss timer
- 模块启用状态
- UI 尺寸计算
- 全展开 tab
- Shelf 交互状态
- 通知展开逻辑
- 扩展模块是否可展示

主要状态枚举：

```swift
enum IslandState: Equatable {
    case compact
    case expanded
    case fullExpanded
}
```

内置模块枚举：

```swift
enum ModuleType: String, CaseIterable, Identifiable {
    case nowPlaying
    case volumeHUD
    case battery
    case shelf
    case connectivity
    case calendar
    case weather
    case notifications
    case teleprompter
}
```

整体上，项目 UI 是围绕 `AppState.shared` 驱动的状态机实现。

## 6. 内置模块设计

AppDelegate 启动时会根据用户设置初始化模块，例如：

```swift
if state.nowPlayingEnabled { _ = NowPlayingManager.shared }
if state.volumeHUDEnabled { _ = VolumeManager.shared }
if state.batteryEnabled { _ = BatteryManager.shared }
```

模块通常由以下几类文件组成：

- Manager：负责系统状态监听和数据获取
- CompactView：紧凑状态展示
- ExpandedView：展开状态展示
- FullExpandedView：部分模块支持全展开展示

例如 Now Playing 模块包含：

```text
NowPlayingManager.swift
NowPlayingCompactView.swift
NowPlayingExpandedView.swift
AlbumArtView.swift
SpectrogramView.swift
```

Weather 模块包含：

```text
WeatherManager.swift
WeatherCompactView.swift
WeatherExpandedView.swift
```

System HUD 模块包含：

```text
VolumeManager.swift
KeyboardBacklightManager.swift
SystemHUDCompactView.swift
SystemHUDExpandedView.swift
```

## 7. 扩展系统

扩展宿主位于：

```text
ExtensionHost/
```

核心文件包括：

- `ExtensionManager.swift`
- `ExtensionJSRuntime.swift`
- `ExtensionManifest.swift`
- `ExtensionRendererView.swift`
- `ExtensionSettings.swift`
- `ExtensionSandbox.swift`
- `ExtensionViewNode.swift`

扩展发现入口是：

```swift
func discoverExtensions()
```

扩展搜索目录包括：

- 用户安装扩展目录
- 开发扩展目录 `ExtensionsDev`
- 本地 `Extensions`
- repo fallback 扩展目录
- app bundle 内置 `BundledExtensions`

扩展激活入口是：

```swift
func activateDiscoveredExtensions()
```

激活扩展时会为 manifest 创建 JavaScript runtime：

```swift
let runtime = try ExtensionJSRuntime(manifest: manifest, manager: self)
```

扩展系统支持：

- manifest 声明
- permissions
- capabilities
- settings schema
- background refresh
- compact / expanded / fullExpanded UI
- notification feed 类型扩展
- 用户禁用状态持久化
- 新扩展默认禁用逻辑

用于持久化扩展状态的 key 包括：

```text
extensions.userDisabled
extensions.seenIDs
```

## 8. 内置扩展

当前内置扩展包括：

```text
ai-usage
pomodoro
agents-status
```

构建时，`project.yml` 会先清空 app bundle 的 `BundledExtensions` 目录，再把这些扩展复制进去，避免旧扩展残留。

### Pomodoro Timer

路径：

```text
Extensions/pomodoro
```

功能：

- 番茄钟
- 支持 compact / expanded / fullExpanded
- 支持 settings
- 每 1 秒刷新

### AI Usage Rings

路径：

```text
Extensions/ai-usage
```

功能：

- 显示 Claude / Codex 使用量
- 面向开发者效率场景
- 每 15 秒刷新

### Agents Status

路径：

```text
Extensions/agents-status
```

功能：

- 显示 Claude Code / Codex CLI session 状态
- 通过本地 Python bridge 获取状态
- 支持多 session focus
- 刷新间隔为 0.5 秒

主应用退出时会停止相关 bridge，避免本地端口和子进程残留。

## 9. 构建和运行

开发环境初始化：

```bash
xcodegen generate
open SuperIsland.xcodeproj
```

然后在 Xcode 中选择 `SuperIsland` scheme，目标选择当前 Mac，点击 Run。

发布构建：

```bash
./scripts/build-and-release.sh
```

需要配置：

```env
APPLE_ID=you@example.com
APP_SPECIFIC_PASSWORD=xxxx-xxxx-xxxx-xxxx
TEAM_ID=XXXXXXXXXX
SIGNING_IDENTITY=Developer ID Application: Your Name (TEAMID)
```

快速生成本地 unsigned DMG：

```bash
./scripts/build-dmg.sh
```

发布脚本会执行 archive、export、notarize，并生成签名后的 DMG。

## 10. 权限和 entitlements

`project.yml` 中配置了以下 entitlements：

```yaml
com.apple.security.personal-information.calendars: true
com.apple.security.personal-information.location: true
com.apple.security.automation.apple-events: true
```

首次启动时应用会请求：

- Accessibility
- Calendar
- Location

其中 Accessibility 可能用于全局事件监听、刘海交互、快捷键、emoji picker 等功能。

## 11. 项目优点

- 模块划分清晰，App、Modules、Views、Window、ExtensionHost 边界比较明确
- 内置模块和 JS 扩展系统分离较好
- `AppState` 集中驱动 UI 状态，适合单浮窗型产品
- 扩展 manifest 设计较完整，支持 permissions、capabilities、refreshInterval、settings
- 支持 bundled extensions 和 development extensions，方便扩展开发和调试
- 具备自动更新、DMG 打包、notarization 等产品化能力
- 支持 notification feed 类型扩展，可把第三方消息接入岛内通知流

## 12. 潜在风险和维护重点

### `AppState` 职责较重

`AppState.swift` 文件较大，承担了状态、尺寸、交互、dismiss、模块切换、Shelf、通知、全展开等多类职责。后续维护中可能成为复杂度集中点。

建议：

- 保持新增逻辑克制
- 对明显独立的状态域逐步拆分
- 避免继续把所有交互规则堆进 `AppState`

### 扩展系统安全边界需要持续关注

扩展系统涉及：

- JavaScript runtime
- 网络访问
- 本地存储
- 通知
- OAuth token
- notification feed
- 本地 bridge

建议重点关注：

- permission enforcement 是否严格
- 扩展能否越权访问 API
- 用户 token 是否应迁移到 Keychain
- 扩展网络请求和本地服务调用是否有边界控制

### 高频刷新扩展可能影响性能

部分扩展刷新频率较高：

- `agents-status`：0.5 秒
- `pomodoro`：1 秒

建议关注：

- CPU 占用
- 电量影响
- Timer 数量
- JS runtime 执行成本
- UI 频繁重绘

### 本地 bridge 和外部依赖可能带来环境问题

`agents-status` 会启动 Python bridge，这类功能可能受用户本机环境影响。

建议关注：

- 子进程生命周期
- 端口占用
- Python 环境是否存在
- 打包失败时的 fallback 行为
- 发布产物是否真的包含运行所需文件

## 13. 推荐阅读顺序

如果要继续开发这个项目，建议按以下顺序看代码。

### 启动和全局状态

```text
SuperIsland/App/SuperIslandApp.swift
SuperIsland/App/AppDelegate.swift
SuperIsland/App/AppState.swift
```

### 浮窗和 UI

```text
SuperIsland/Window/IslandWindowController.swift
SuperIsland/Window/IslandWindow.swift
SuperIsland/Views/IslandContainerView.swift
SuperIsland/Views/CompactView.swift
SuperIsland/Views/ExpandedView.swift
SuperIsland/Views/FullExpandedView.swift
```

### 扩展系统

```text
ExtensionHost/ExtensionManager.swift
ExtensionHost/ExtensionJSRuntime.swift
ExtensionHost/ExtensionManifest.swift
ExtensionHost/ExtensionRendererView.swift
EXTENSIONS.md
EXTENSIONS-API.md
```

### 构建和发布

```text
project.yml
scripts/build-dmg.sh
scripts/build-and-release.sh
```

## 14. 总体判断

SuperIsland 是一个已经比较产品化的 macOS notch utility。它的核心是 SwiftUI/AppKit 浮窗状态机，围绕 `AppState` 驱动多状态展示；扩展能力通过 JavaScript runtime 提供，内置扩展覆盖 productivity、developer tools、communication、music 等场景。

后续维护重点建议放在：

1. 控制 `AppState` 复杂度
2. 强化扩展系统安全边界
3. 将敏感 token 存储迁移到 Keychain
4. 优化高频刷新扩展的性能和电量影响
5. 保证本地 bridge 和打包流程在发布环境中稳定可复现
