# AI 使用量圆环扩展

在 SuperIsland 中用圆形指示器显示 Codex 和 Claude 的使用量 / 可用性。

## 颜色含义

- 绿色：状态健康 / 可用
- 橙色：额度偏低
- 红色：额度很低 / 受限

## 权限

- `usage`：必需，用于调用 `SuperIsland.system.getAIUsage()`

## 数据来源

- Codex：
  - `~/.codex/usage-summary.json` 或 `~/.codex/usage/summary.json`
  - fallback：使用 `~/.codex/auth.json` 中的 token 调用 ChatGPT OAuth 使用量 API：`https://chatgpt.com/backend-api/wham/usage`
- Claude：
  - `~/.claude/usage-summary.json` 或 `~/.config/claude/usage-summary.json`
  - fallback：使用以下位置的 token 调用 Anthropic OAuth 使用量 API：`https://api.anthropic.com/api/oauth/usage`
    - `~/.claude/.credentials.json` / `~/.claude/credentials.json`
    - macOS Keychain service：`Claude Code-credentials`
  - 最后 fallback：`~/.claude/stats-cache.json`

SuperIsland 会先尝试静默读取 Claude Keychain 项。如果 macOS 需要弹出密码提示，SuperIsland 只会询问一次；如果无法静默读取，则回退到本地 Claude 使用量 / 缓存数据。

## 刷新行为

- 原生 usage provider 缓存每 5 分钟刷新一次。
- Codex 仍会从本地 summary 或 OAuth API 数据更新。
- Claude 会在可用时从 OAuth usage 中读取 session（`five_hour`）和 weekly（`seven_day*`）窗口。
- 当源数据缺失时，周 / 会话值不再镜像 overall remaining，而是显示 `--%`。
