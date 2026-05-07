# Onboarding 页面完整重设计提示词

## 概览

重建 **SuperIsland**（macOS SwiftUI 应用）的完整 onboarding 流程。onboarding 是一个**独立窗口**，不需要像原生 macOS 窗口。它应该像一个居中的、紧凑的、悬浮的黑色 A4 风格卡片。不要标题栏 chrome，不要工具栏，只保留一个干净、深色、自包含、圆角的面板，不显示任何原生窗口装饰。

窗口尺寸约为 **840×620 points**，使用 `titlebarAppearsTransparent = true`、`titleVisibility = .hidden`、`.fullSizeContentView` style mask、`isOpaque = false`、`backgroundColor = .clear`。内容铺满窗口。整体形状是一个大的 **continuous rounded rectangle**，圆角约 28，并裁剪到窗口边界，使其看起来像桌面上的深色悬浮卡片。

---

## 全局设计语言

### 色彩方案（深色主题）

- **背景底色**：从浓黑 `#0A0A0E` 到深炭灰 `#111117` 的渐变，不要使用纯 `#000000`，需要轻微冷暖底色。
- **主文本**：`rgba(255, 255, 255, 0.94)`，接近白色，但不要刺眼。
- **次级文本**：`rgba(255, 255, 255, 0.62)`，用于描述。
- **三级文本**：`rgba(255, 255, 255, 0.38)`，用于提示和脚注。
- **冷色强调色**：柔和浅紫蓝 `#7DB4FF` 到深蓝 `#4A87F5`。
- **暖色强调色**：琥珀金 `#F5A84B` 到焦橙 `#E07832`。
- **成功绿色**：`#34D399`，90% 不透明度。
- **卡片表面**：`rgba(255, 255, 255, 0.05)` 填充，`rgba(255, 255, 255, 0.08)` 1px 边框。
- **交互元素边框**：`rgba(255, 255, 255, 0.12)`。

### 背景：Liquid Glass 紫色动态渐变

所有页面内容背后都渲染一个微妙的动态渐变层，使用缓慢漂移的模糊圆形：

1. **紫色光球**：`#8B5CF6`，18% 不透明度，直径约 380pt，blur 半径 50。以 10 秒 `easeInOut` 在左上与中左之间对角漂移，永久往返。
2. **靛蓝光球**：`#6366F1`，14% 不透明度，直径约 300pt，blur 半径 44。从中右到右下漂移，周期 12 秒。
3. **紫粉光球**：`#A78BFA`，10% 不透明度，直径约 260pt，blur 半径 38。在底部中央与中心附近漂浮，周期 14 秒。

这些光球位于半透明黑色渐变覆盖层之后：`#0A0A0E` 85% 到 `#111117` 70%，从左上到右下，以确保文字始终清晰可读。动画感觉应像深色玻璃背后轻微流动的极光，可见但不能抢夺注意力。

**关键要求**：渐变不能冲淡或覆盖任何文本和 UI 元素，只作为氛围背景。

### 字体

- 全部使用 `.system` 字体，即 San Francisco。
- Hero 标题：大小 44–48，weight `.bold`；Hello 效果可用。分区标题大小 36，weight `.semibold`。
- 正文：大小 16–17，weight `.regular`。
- Caption / chip 文本：大小 12–13，weight `.semibold`。
- 除卡片布局外，文本默认居中；卡片内左对齐。

### 页面指示器

窗口右上角放一排 **3 个胶囊点**：

- 当前页面：长胶囊，宽 28、高 8，白色 85%。
- 非当前页面：圆点，宽高 8，白色 18%。
- 使用 `.spring(response: 0.34, dampingFraction: 0.82)` 动画宽度变化。
- 外层放在一个胶囊容器中，填充 `rgba(255,255,255, 0.06)`，边框 `rgba(255,255,255, 0.12)` 1px。

### 页面转场

所有页面转场使用 `.asymmetric`：

- 插入：`.move(edge: .trailing).combined(with: .opacity)`
- 移除：`.move(edge: .leading).combined(with: .opacity)`
- 动画：`.spring(response: 0.38, dampingFraction: 0.9)`

---

## 第 1 页：欢迎页（Apple “Hello” 效果）

### 布局（自上而下，居中）

#### 1. Hero “Hello” 文本

重现 Apple 标志性的 “Hello” 动画：

- 用大号优雅手写 / 脚本风格渲染 **“Hello.”**。SwiftUI 没有完全一致的 Apple Hello 字体，可以用以下方式近似：
  - 使用 `.font(.system(size: 72, weight: .thin, design: .serif))` 营造手写感。
  - 或使用自定义轻量 script 字体。
  - 也可使用大号、极细 San Francisco（size 64，`.ultraLight`）并加字距，呈现 Apple keynote 风格。
- **动画**：使用水平 `LinearGradient` mask 从左到右擦除显示，约 1.8 秒 `easeInOut`，让文字像被写出来一样逐步出现。
- **颜色**：文字本身使用缓慢变化的渐变填充：
  - 柔紫 `#A78BFA` → 蓝 `#60A5FA` → 青 `#2DD4BF` → 粉 `#F472B6` → 回到紫色。
  - 可使用 `AngularGradient` 或 `LinearGradient`，并在 6 秒内循环移动 startPoint / endPoint。

#### 2. 副标题

Hello 文本下方 20pt 间距：

- **“Welcome to SuperIsland”**：大小 22，weight `.semibold`，主文本色。
- 下方 8pt：**“Your Mac's notch, reimagined.”**：大小 16，weight `.regular`，次级文本色。

#### 3. 功能标签

一排 3 个小胶囊标签：

- “Calm by default” · “Notch-native” · “Made for macOS”
- 每个 chip：大小 12，semibold，三级文本色，胶囊填充 `rgba(255,255,255, 0.05)`，边框 `rgba(255,255,255, 0.10)` 1px。
- HStack 间距 8pt。

#### 4. Continue 按钮

- 文案：“Continue”。
- 使用冷色强调渐变 `#7DB4FF` → `#4A87F5`。
- 胶囊形状，高 46pt，水平 padding 24。
- 白色文本，大小 15，semibold。
- Hover：scale 1.04，阴影增强，使用冷色强调 28% 不透明度、radius 18 的 glow。
- Hover 动画：`.spring(response: 0.32, dampingFraction: 0.78)`。

---

## 第 2 页：权限

### 布局

#### 1. 标题

- **“Let's set things up”**：大小 34，weight `.semibold`，主文本色，居中。
- 下方 8pt：**“Grant a few permissions so everything works smoothly.”**：大小 16，次级文本色，居中。

#### 2. 权限卡片列表

使用 `VStack(spacing: 14)` 展示权限卡片。每张卡片采用参考图中的 liquid glass 图标风格：图标在圆角矩形容器中，带虹彩渐变填充。

必需权限：

| 权限 | SF Symbol | 描述 |
|---|---|---|
| **Screen Recording** | `display` | “Lets SuperIsland detect your active workspace and render over the notch.” |
| **Accessibility** | `figure.stand` | “Required for gesture detection, window interaction, and productivity overlays.” |

可选权限：

| 权限 | SF Symbol | 描述 |
|---|---|---|
| **Calendar** | `calendar` | “Show upcoming events right in the island.” |
| **Notifications** | `bell.badge` | “Mirror system notifications in the Super Island.” |
| **Microphone** | `mic.fill` | “Powers the audio spectrogram visualizer.” |
| **Location** | `location.fill` | “Displays local weather information.” |
| **Bluetooth** | `wave.3.right` | “Shows connected device status.” |

#### 3. 权限卡片设计

```text
┌──────────────────────────────────────────────────────────────────┐
│  ┌─────────┐                                                    │
│  │  icon   │  Title                          [Grant Access]     │
│  │ (glass) │  Description text here...        or ✓ Granted      │
│  └─────────┘                                                    │
└──────────────────────────────────────────────────────────────────┘
```

- **卡片背景**：`rgba(255,255,255, 0.05)` 填充，`rgba(255,255,255, 0.08)` 1px continuous rounded rect 边框，cornerRadius 22。
- **Padding**：20pt。
- **图标容器**：54×54，圆角矩形，cornerRadius 16，continuous。
- 必需权限图标容器：紫 `#8B5CF6` → 蓝 `#3B82F6` → 金 `#F59E0B` 的 liquid glass 渐变。
- 可选权限图标容器：更弱的深色表面，加微弱虹彩边框。
- 图标居中，白色 92%，size 22 semibold。
- 授权后：图标容器增加微弱绿色 tint。
- 标题：size 17，weight `.semibold`，主文本色。
- 描述：size 13，weight `.regular`，次级文本色。
- 右侧按钮：未授权显示 “Grant Access”，已授权显示绿色 check badge。
- Hover：卡片上移 2pt，阴影增强。
- 出现动画：卡片逐个出现，每张延迟 100ms，从下方 18pt + opacity 0 进入。

#### 4. Required / Optional 分割

使用微妙的横线或文本：

- “Optional — you can enable these later”
- 三级文本色，左右用细线分隔。

#### 5. Continue 按钮

- 与第 1 页相同冷色按钮。
- 在两个必需权限都授权之前禁用。
- 禁用时透明度 0.48，不响应 hover。
- 按钮下方显示：“Enable required permissions above to continue.”。
- 使用 `.task` 每 800ms 轮询 `PermissionsManager.shared`。

---

## 第 3 页：手势教程与完成

### 布局

#### 1. 标题

- **“How to use SuperIsland”**：size 34，weight `.semibold`，主文本色，居中。
- 下方：**“A few quick gestures and you're ready.”**：size 16，次级文本色，居中。

#### 2. 手势说明卡片

居中展示 3 张手势卡片：

##### 卡片 A：左右滑动

- 动效：一个迷你 SuperIsland 胶囊，配合左右箭头或手势图标来回移动。
- 标题：“Swipe Left & Right”。
- 描述：“Cycle through modules — music, timer, calendar, and more.”。

##### 卡片 B：上下滑动

- 动效：迷你岛胶囊，上下箭头轻微 bounce。
- 标题：“Swipe Up & Down”。
- 描述：“Swipe up to expand, swipe down to dismiss.”。

##### 卡片 C：锁定按钮

- 动效：`lock.fill` 与 `lock.open` 循环切换。
- 标题：“Lock Open”。
- 描述：“Tap the lock to keep the island expanded until you dismiss it.”。

#### 3. 卡片样式

- 宽度：填满可用空间。两列布局时最大约 340pt；垂直堆叠时约 680pt。
- 高度自适应，padding 24。
- 背景：`rgba(255,255,255, 0.04)`，边框 `rgba(255,255,255, 0.08)` 1px，cornerRadius 22。
- 顶部动效区域高约 100pt，居中。
- 标题：size 17，weight `.semibold`，主文本色。
- 描述：size 14，weight `.regular`，次级文本色。
- 使用与权限卡片相同的 stagger 出现动画。

#### 4. 布局选项

优先使用上方两列、下方居中的布局：

```text
┌──────────────┐  ┌──────────────┐
│  Swipe L/R   │  │  Swipe U/D   │
└──────────────┘  └──────────────┘
       ┌──────────────┐
       │   Lock Open  │
       └──────────────┘
```

如果空间不足，也可以使用垂直 `VStack(spacing: 14)`。

#### 5. Get Started 按钮

- 文案：“Get Started”。
- 使用暖色强调渐变 `#F5A84B` → `#E07832`。
- 胶囊形状，高 46pt。
- 点击时触发 sparkle burst 动画，6 个小星星向外飞出并淡出，0.5 秒后关闭 onboarding。
- 启动中时按钮文字变成 “Launching…”，并禁用。
- 下方提供 “Open Settings Later” 三级文本按钮。

---

## 微交互与打磨

### Hover 状态

所有交互元素（按钮、权限卡片）都应有：

- Scale：1.0 → 1.03。
- 阴影增强。
- 动画：`.spring(response: 0.32, dampingFraction: 0.78)`。

### Liquid Glass 图标风格

参考图中的图标处理方式：

- 图标位于圆角方形容器中，容器使用虹彩渐变填充，混合紫、蓝、金、琥珀色。
- 渐变应有金属、折射质感，像光照在肥皂泡或油膜上。
- SwiftUI 实现方式：使用带紫、蓝、金、橙等 stop 的 `AngularGradient`，缓慢旋转角度（8 秒循环）。叠加微弱 `RadialGradient` 作为高光。
- 添加 `rgba(255,255,255, 0.15)` 1px 边框，形成玻璃边缘感。
- 阴影：`Color.black.opacity(0.20), radius: 16, y: 8`。

### 窗口行为

- 窗口不可调整大小，固定尺寸。
- `isMovableByWindowBackground = true`，允许从任意位置拖动。
- 不需要最小化按钮行为。
- 关闭按钮触发 `onClose` 回调。
- 窗口圆角应裁剪为 continuous rounded rectangle，避免露出 macOS 原生直角。

### 无障碍

- 所有交互元素必须设置 `.accessibilityLabel()` 和 `.accessibilityValue()`。
- 标题使用 `.accessibilityAddTraits(.isHeader)`。
- 权限卡片使用 `.accessibilityElement(children: .combine)`，提供描述性 label。
- VoiceOver 必须能够线性导航所有页面。

---

## 文件结构建议

- `OnboardingWindowController.swift`：窗口创建、生命周期、关闭回调。
- `OnboardingView.swift`：页面容器和状态流转。
- `PermissionsManager.swift`：权限检查和请求。
- 共享样式可抽到局部 helper / private view 中。

---

## 验收标准

- onboarding 作为独立悬浮卡片出现。
- 没有可见原生标题栏或工具栏。
- 背景有微妙动态渐变，但不影响阅读。
- 页面转场流畅。
- 权限状态可实时更新。
- 必需权限未授权时不能继续。
- 所有按钮和卡片 hover 状态自然。
- VoiceOver 可用。
