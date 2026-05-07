# 扩展目录

此目录包含 SuperIsland 的可插拔扩展。

每个扩展都是自包含目录，可以独立分发，例如打包为 zip，或通过 Git 仓库分发。目录结构通常如下：

```text
<extension>/
  manifest.json
  index.js
  settings.json (可选)
  assets/ (可选)
```

开发期间，SuperIsland 会从当前 `Extensions/` 目录中发现扩展。

当前仓库保留的内置扩展：

- `pomodoro`：番茄钟
- `ai-usage`：AI 使用量圆环
- `agents-status`：Agent 状态
