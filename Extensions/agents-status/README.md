# Agent 状态扩展

在岛中以 5×5 像素动画显示编码 agent 会话状态（Working / Waiting / Idle / Error），并支持跨 agent、跨终端跟踪多个并发会话。

当前支持：

- **Claude Code**：通过 hooks 覆盖完整事件（Working / Waiting / Idle / Error）
- **Codex CLI**：通过 `~/.codex/config.toml` 和 `~/.codex/hooks.json` 使用官方 hooks，支持按 turn 跟踪 Working / Idle，并在活动 turn 异常退出时短暂显示 Error

## 工作原理

Agent hooks 是 shell 命令，而 SuperIsland 扩展运行在沙盒内，不能直接访问文件系统或 IPC。因此该扩展通过一个很小的本地 HTTP server 做桥接：

```text
Claude Code / Codex hooks  --curl POST-->  127.0.0.1:7823  <--GET poll--  extension
```

- hooks 会在每次 agent 事件发生时 POST 增强事件，包括 `session_id`、`cwd`、`title`、`terminal`。
- bridge 按 `(agent, session_id)` 跟踪状态，因此多个并发会话会显示在展开 / 全展开视图列表中。
- 点击展开或全展开视图中的会话行，会跳回对应终端会话。
  - 在 Warp 中，bridge 会根据自定义 tab title 定位当前 tab，并发送 `Cmd+<index>` 直接切换。
  - Terminal.app 会按 TTY 选择匹配 tab。
  - 不支持的终端会退化为仅将应用带到前台。
- 超过配置 TTL（10–60 分钟，默认 30 分钟）没有活动的会话会从列表中移除。
- Claude `Working` 会话在静默 30 秒后自动衰减为 `Idle`，避免 hook 崩溃导致状态卡住。
- Codex 会在活动 turn 中保持 `Working`，直到收到 `Stop`、CLI 退出，或 transcript 记录到中断标记 `<turn_aborted>`。

## 安装

该扩展已随 SuperIsland 内置，不需要手动运行脚本。只需在 **SuperIsland → 设置 → 扩展** 中启用它。

bridge server（`server.py`）会在扩展激活时自动启动，并在扩展停用或应用退出时停止。

要求：系统 PATH 中必须能找到 Python 3，例如：

- `/opt/homebrew/bin/python3`
- `/usr/local/bin/python3`
- `/usr/bin/python3`

在 macOS 上，通过 `xcode-select --install` 安装 Command Line Tools 通常即可满足要求。

验证 bridge 是否运行：

```bash
curl http://127.0.0.1:7823/health
# {"ok":true,"port":7823,"paused":false}
```

## 启用需要跟踪的 agent

在扩展设置中切换：

- **Claude Code**：将 hooks 合并到 `~/.claude/settings.json`
- **Codex CLI**：在 `~/.codex/config.toml` 中启用 `features.codex_hooks = true`，并在 `~/.codex/hooks.json` 中安装 command hooks

这些操作是幂等的，并会保留备份（`.agents-status.bak`）。关闭开关时，只会移除此扩展添加的条目。

## 测试

```bash
curl -s -X POST http://127.0.0.1:7823/event -H 'Content-Type: application/json' \
  -d '{"state":"Working","agent":"Claude","session_id":"demo","title":"testing","cwd":"/tmp","terminal":"iTerm"}'
curl http://127.0.0.1:7823/state
```

## 状态映射

| Claude Code hook | 状态 |
|---|---|
| `SessionStart` | Idle |
| `UserPromptSubmit` | Working |
| `PreToolUse` | Working |
| `PostToolUse` | Working |
| `PostToolUseFailure` (`is_interrupt=true`) | Idle |
| `Notification` | Waiting |
| `Stop` | Idle |
| `SessionEnd` | 移除会话 |

| Codex hook | 状态 |
|---|---|
| `SessionStart` | Idle |
| `UserPromptSubmit` | Working |
| `PreToolUse` | Working |
| `PostToolUse` | Working |
| `Stop` | Idle |

对于 Codex，Bash 命令退出码会被视为普通工具执行结果，而不是 agent 错误。只有活动 Codex turn 在 `Stop` 到达前异常消失时，才会显示 `Error`。

目前 Codex 没有专用的 interrupt hook，因此 ESC 中断会退回使用 transcript 中的 `<turn_aborted>` 标记判断。

`PostToolUse` 当前受 Codex 自身限制，仅包含 Bash payload。其他工具类型目前还不会在此处被拦截。

## 配置

扩展设置：

- **Claude Code / Codex CLI 开关**：变更后会自动协调 hooks。

活动会话会保持可见，直到对应 Claude / Codex 进程退出，或 agent 发出明确的 session-end 事件。

### Warp tab 命名

为了稳定切换 Warp tab，请为每个 agent tab 设置不同的自定义标题，例如 `repo-cc` 和 `repo-cx`。

岛中会直接显示该 tab 标题，点击会话行时会根据当前屏幕上的 tab index 切换到匹配的 Warp tab。

自动注入的 server 环境变量：

- `AGENTS_STATUS_PORT`：默认 `7823`
- `AGENTS_STATUS_WORKING_TIMEOUT`：陈旧 `Working` 状态衰减为 `Idle` 前的秒数，默认 `30`
- `AGENTS_STATUS_ERROR_DISPLAY_SECONDS`：Codex 异常退出显示 `Error` 的时长，默认 `45`
- `AGENTS_STATUS_CC_HOOK_SCRIPT`：Claude Code hook 脚本路径（`cc-event-hook.sh`）
- `AGENTS_STATUS_CODEX_HOOK_SCRIPT`：Codex hook bridge 脚本路径

## 从 1.2.x 或更早版本升级

bridge 现在由 SuperIsland 自身管理。旧版本依赖通过 `server/install.sh` 安装的 `launchd` 用户 agent。

新扩展首次启动时，会自动移除旧版本残留的 `com.superisland.agents-status` 或 `com.superisland.cc-status` LaunchAgent 及其 plist，因此通常不需要手动清理。

升级后，建议在设置中关闭再重新打开 Claude Code 和 / 或 Codex 开关，确保 `~/.claude/settings.json` 和 `~/.codex/hooks.json` 中的 hook 条目指向当前内置脚本。

## 故障排查

- **岛中显示 `offline`**：bridge 启动失败。打开 SuperIsland → 设置 → 扩展 → Agents Status → Logs 查看 Python 进程输出，或确认已安装 `python3`。
- **没有会话出现**：确认 hooks 已安装到 agent 实际读取的设置文件中。Claude Code 需要区分用户级和项目级设置；Codex 需要检查 `~/.codex/config.toml` 和 `~/.codex/hooks.json`。
- **点击会话无法在 Warp 中切换 tab**：macOS 可能阻止 UI scripting。请为 bridge 使用的 `osascript` 授予 Automation / Accessibility 权限后重试。
- **Codex 频繁显示 `Error`**：更新到最新 `agents-status`。较新版本只会在活动 turn 异常退出时显示 `Error`，不会把普通 Bash 命令失败当作 agent 错误。
