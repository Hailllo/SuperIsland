# 参与贡献 SuperIsland

感谢你愿意为 SuperIsland 做贡献。下面是开始开发前需要了解的内容。

---

## 开始开发

```bash
git clone https://github.com/Hailllo/SuperIsland.git
cd SuperIsland
xcodegen generate
open SuperIsland.xcodeproj
```

在 Xcode 中运行 `SuperIsland` scheme，目标选择当前 Mac。应用需要辅助功能权限，才能让岛窗口显示在其他应用之上并处理相关交互。

---

## 可以做什么

可以从 issue 或当前 TODO 中选择任务。适合新贡献者的任务通常包括：修复 UI 小问题、完善文档、优化已有模块或补充扩展。

如果你准备做较大的改动，例如新增模块、调整架构或修改扩展运行时，请先创建 issue 或讨论方案，避免做无效工作。

---

## 修改代码

- 从 `main` 创建分支：`git checkout -b your-feature`
- 保持 commit 聚焦，每个 commit 只做一件事。
- 提交 PR 前，请运行应用并手动验证你的改动。
- 如果新增内置模块，请遵循现有模块结构：
  - 一个 `Manager` 单例，使用 `@Published` 暴露状态
  - 一个 `CompactView`
  - 一个 `ExpandedView`
  - 必要时增加 `FullExpandedView`
  - 在 `ModuleType` 中注册模块

---

## Pull Request

- 目标分支使用 `main`
- 说明改了什么以及为什么改
- UI 改动建议附截图或录屏
- 保持 PR 小而可审查。大型重构请先讨论

---

## 添加扩展

扩展是无需修改 Swift 主应用即可贡献功能的方式。完整说明请查看 [EXTENSIONS.md](EXTENSIONS.md)。

开发时，将扩展目录放入 `Extensions/`，应用会在开发环境中自动发现。

---

## 代码风格

- Swift：遵循现有风格；避免 force unwrap；涉及 UI 的代码使用 `@MainActor`；优先使用 `guard` 做提前返回。
- JavaScript 扩展：使用普通 ES5 兼容 JavaScript，或在提交前将 TypeScript 编译为 JavaScript。简单扩展不需要 bundler。
- PR 中不要提交注释掉的废弃代码，也不要保留调试用 `print`。

---

## 报告问题

报告 bug 时请提供：

- macOS 版本，以及设备是否有刘海
- 复现步骤
- 期望结果和实际结果
