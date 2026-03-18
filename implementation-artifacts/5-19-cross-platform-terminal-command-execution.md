# Story 5.19: Cross-Platform Terminal Command Execution

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. Design complete. -->

## Story

As a **Runtime user**,  
I want the Agent to execute local terminal commands through built-in Runtime tools,  
So that the LLM can directly run project commands such as `git`, `npm test`, `ls`, and `bash script.sh` on my machine instead of only reading and writing files.

## Acceptance Criteria

### AC-1: `terminal.run` 直接命令执行

**Given** the Agent needs to run a local command that can be expressed as `command + args`  
**When** it calls `terminal.run` with `command`, optional `args`, optional `cwd`, and optional timeout  
**Then** Runtime executes the process in Electron Main via `spawn` with shell disabled by default  
**And** returns bounded `stdout`, `stderr`, `exitCode`, `durationMs`, and truncation metadata to the LLM as the tool result  
**And** commands such as `git status`, `npm test`, and `bash script.sh` are supported through this path when they are naturally expressible as executable + args

### AC-2: `shell.exec` 明确的 Shell 模式

**Given** the Agent needs shell syntax, compound commands, redirection, pipelines, variable expansion, or other shell semantics  
**When** it calls `shell.exec` with a command string and optional shell hint  
**Then** Runtime resolves an appropriate shell for the current platform and executes the command  
**And** returns bounded `stdout`, `stderr`, `exitCode`, `durationMs`, and resolved shell metadata to the LLM  
**And** if the requested shell is unavailable, Runtime returns a structured error instead of failing silently  
**And** `shell.exec` is documented as the exception path rather than the default path

### AC-3: Prompt / Tool 定义与执行边界一致

**Given** the model receives system guidance and tool definitions  
**When** terminal execution is enabled for that run  
**Then** the prompt and tool descriptions explicitly teach the model to prefer `terminal.run` for portable `command + args` execution  
**And** use `shell.exec` only when shell semantics are actually required  
**And** the prompt includes concrete examples for `git status`, `npm test`, `bash script.sh`, and directory listing with platform caveats  
**And** the system prompt includes a Runtime Execution Context summary with current platform, terminal capability flags, cwd constraints, and shell resolution guidance without exposing raw host environment details  
**And** prompt declaration, visible tools, and ToolHost enforcement use the same effective terminal tool policy

### AC-4: macOS / Windows 跨平台是一等支持目标

**Given** the Runtime runs on either macOS or Windows  
**When** the terminal tools are used  
**Then** executable lookup, shell resolution, path normalization, encoding, timeout handling, and process termination follow platform-specific rules  
**And** both macOS and Windows are required first-class supported targets for this story  
**And** Runtime does not assume Linux-only behavior  
**And** “tool is cross-platform” is explicitly defined as consistent capability and error model, not as a promise that every command string runs unchanged on both OSes

### AC-5: 结果返回给 LLM，UI 仅展示调用摘要

**Given** a terminal command is executed  
**When** the tool completes, fails, times out, or is aborted  
**Then** bounded `stdout`, `stderr`, and execution metadata are returned to the LLM as the tool result  
**And** ChatPanel only shows a concise invocation summary similar to `fs.*` tools (tool name, command/cwd summary, success/failure, exit status)  
**And** this story does not require a dedicated terminal output panel or full terminal output rendering in the UI

### AC-6: 日志、超时与取消

**Given** a terminal command is running  
**When** it produces output, exceeds timeout, or the user stops execution  
**Then** Runtime captures output in a bounded way and preserves partial stdout/stderr for debugging  
**And** terminates the process tree safely on macOS and Windows  
**And** records the execution, exit status, and failure reason in the run log / audit log with redacted or truncated previews

### AC-7: 安全与策略控制

**Given** terminal execution is a high-risk capability  
**When** it is exposed to the LLM  
**Then** Runtime gates it behind explicit terminal tool policy / settings switches independent from `fs.*` and MCP  
**And** restricts working directory to `@project` or other allowlisted roots  
**And** applies output byte limits, timeout limits, optional environment allowlisting, and a separate `allowShellExec` control  
**And** avoids exposing a raw unrestricted shell by default

### AC-8: MVP 范围明确

**Given** this is the initial delivery of local command execution  
**When** Story 5.19 is implemented  
**Then** MVP supports single-shot command execution only  
**And** one-shot shell execution through `shell.exec` is in scope for this story  
**And** PTY-backed persistent sessions, interactive stdin, and full terminal emulation are explicitly out of scope  
**And** commands that require persistent GUI interaction, watch mode, REPL input, or other long-lived interactive sessions are out of scope for MVP  
**And** any `terminal.session.*` capability is deferred to a follow-up story

## Technical Notes

- Delivery pattern: vertical Runtime story covering Main process tool execution, ToolHost integration, prompt/tool policy updates, and tests together.
- This story is the **Codex-like local command execution capability** for Runtime, but the first delivery should focus on **one-shot command execution** rather than a full terminal emulator.
- Reference implementation should follow existing Runtime patterns from `fs.*`, `python.run`, and MCP stdio execution.
- Command execution results are primarily for the LLM; UI scope is limited to tool invocation summaries in ChatPanel.
- This story introduces terminal execution as a builtin tool capability; it does **not** redefine the existing menu `exec` contract, which still means Markdown script entry.
- Commands that launch or depend on persistent GUI interaction, watch mode, or REPL-style user input are not target scenarios for this MVP unless the launcher process returns immediately and behaves like a bounded one-shot command.

## Design

Design and implementation complete. Status set to `review`.

### UX / UI

- Terminal execution is a tool capability surfaced to the LLM, not a new primary page.
- ChatPanel should show only a concise tool invocation summary for terminal calls, aligned with existing `fs.*` tool call presentation.
- Full terminal stdout/stderr should not be rendered in ChatPanel for this story.
- Settings must provide an explicit enable/disable control for terminal execution, separate from `fs.*` and MCP switches.
- Settings must also provide a distinct shell execution control (for example `allowShellExec`) rather than implicitly allowing shell mode whenever terminal tools are enabled.
- No PTY UI or dedicated terminal panel is required for MVP.

### API / Contracts

**Proposed tools**

```ts
// Preferred for portable execution
terminal.run({
  command: "git",
  args: ["status", "--short"],
  cwd?: "@project" | "@project/subdir" | absoluteAllowedPath,
  timeoutMs?: number,
  env?: Record<string, string>
})

// Explicit shell mode for shell-specific syntax
shell.exec({
  script: "npm test && npm run lint",
  shell?: "auto" | "bash" | "zsh" | "powershell" | "cmd",
  cwd?: "@project" | "@project/subdir" | absoluteAllowedPath,
  timeoutMs?: number,
  env?: Record<string, string>
})
```

**Usage boundary**

- `terminal.run` is the default path for commands that naturally fit executable + args, including `git status`, `npm test`, and `bash script.sh`.
- `shell.exec` is reserved for commands that require shell parsing, such as `&&`, `|`, redirection, globbing, or environment-variable expansion.
- Tool descriptions and system guidance must teach this distinction directly to the model.
- PromptComposer should inject a Runtime Execution Context block that tells the model the current platform, terminal capability flags, cwd policy, and shell-resolution rules without exposing raw host env or other sensitive machine details.
- Long-lived interactive commands such as watch mode, GUI-bound flows, and REPL loops are out of scope for this contract.

**Proposed result shape**

```ts
type TerminalRunResult =
  | {
      ok: true
      stdout: string
      stderr: string
      exitCode: number
      durationMs: number
      stdoutTruncated?: boolean
      stderrTruncated?: boolean
      shellResolved?: string
      platform: "darwin" | "win32"
    }
  | { ok: false; error: ToolError }
```

**Platform resolution**

- `terminal.run`: always `spawn(command, args, { shell: false })`
- `shell.exec`:
  - macOS default: resolve a discoverable POSIX shell, preferring configured shell, then `/bin/bash`, then `/bin/zsh`
  - Windows default: prefer `pwsh.exe`, then `powershell.exe`, then `cmd.exe`
  - explicit `bash` on Windows must only run if a real bash executable is discoverable; otherwise return `SHELL_NOT_AVAILABLE`

**Prompt / Tool policy alignment**

- System prompt and tool descriptions must expose `terminal.run` / `shell.exec` only when the effective terminal policy allows them.
- Prompt guidance must explicitly say that “cross-platform tool” does not mean “every command string is portable”.
- Prompt guidance should include a Runtime Execution Context summary so the model can make OS-aware terminal decisions without guessing the local machine shape.
- Tool policy merge/enforcement must stay same-source with prompt declaration to avoid prompt/runtime mismatch.

### Data / Storage

- `AgentToolPolicy` and Runtime settings should gain an explicit `terminal` branch independent from `fs` and `mcp`.
- Settings extension should include:
  - `tools.terminal.enabled`
  - `tools.terminal.maxStdoutBytes`
  - `tools.terminal.maxStderrBytes`
  - `tools.terminal.timeoutMs`
  - optional `tools.terminal.allowedRoots`
  - optional `tools.terminal.allowShellExec`
  - optional `tools.terminal.envAllowlist`
- Audit records should include command name, cwd, shell type, duration, exit code, and redacted/truncated output previews.
- No terminal history persistence is required for MVP beyond existing execution logs.

### Errors / Edge Cases

- `COMMAND_NOT_FOUND`: executable not found in PATH / configured location
- `SHELL_NOT_AVAILABLE`: requested shell not installed on current platform
- `CWD_NOT_ALLOWED`: cwd outside allowed roots
- `EXEC_TIMEOUT`: command exceeded timeout and was terminated
- `EXEC_ABORTED`: run/chat was stopped by user or controller
- `TOOL_NOT_AVAILABLE`: terminal execution disabled by effective tool policy
- `OUTPUT_LIMIT_EXCEEDED`: stdout/stderr truncated to safety limits (or equivalent truncation metadata on success)
- `INTERACTIVE_COMMAND_NOT_SUPPORTED`: optional explicit error or hint for commands that clearly require persistent GUI / REPL / watch-mode interaction
- `NON_PORTABLE_COMMAND`: optional hint when the model chooses a command known to mismatch the current platform
- Windows-specific path quoting and PowerShell escaping must not reuse POSIX assumptions
- macOS-specific process-group termination must not be copied directly to Windows without `taskkill` or equivalent handling

### Test Plan

- Unit tests for command resolution, cwd validation, timeout, truncation, and structured errors.
- Unit tests for platform shell selection on macOS and Windows.
- Unit tests for terminal tool policy merge, visibility, and enforcement independent from `fs.*`.
- Unit tests for prompt/tool description generation so the model receives `terminal.run` / `shell.exec` usage guidance and examples.
- Unit tests or design review coverage for out-of-scope interactive commands (GUI launcher vs long-lived GUI process, watch mode, REPL).
- Integration tests:
  - macOS: `git status`, `npm test`, `bash script.sh` via `terminal.run`, plus a shell syntax flow via `shell.exec`
  - Windows: `git status`, `npm test`, PowerShell / `cmd` execution flow, and explicit `bash` unavailable path when bash is missing
- Regression tests ensuring ChatPanel shows only invocation summary rather than full terminal output.
- Manual verification that stopping a run kills long-running commands and preserves partial output.

## Tasks / Subtasks

- [x] Task 1: 定义 terminal tool contract（AC: 1,2,3,5,7）
  - [x] 1.1 在 `ToolHost` / `FileSystemToolHost` 中新增 `terminal.run`
  - [x] 1.2 新增 `shell.exec`，并与 `terminal.run` 保持明确语义分离
  - [x] 1.3 扩展 ToolResult、ToolCall 摘要与日志脱敏结构

- [x] Task 2: Main Process 执行服务（AC: 1,2,4,6,7）
  - [x] 2.1 新建 `terminalService.ts`（或等价服务）封装 `spawn`
  - [x] 2.2 实现 macOS / Windows 可执行查找与 shell 解析
  - [x] 2.3 实现超时、停止、进程树终止、输出截断
  - [x] 2.4 实现 cwd / env 安全限制

- [x] Task 3: Tool Policy / Settings 集成（AC: 3,5,7）
  - [x] 3.1 为 `AgentToolPolicy` / Runtime settings 新增独立 `terminal` 策略分支
  - [x] 3.2 在 Settings 中新增 terminal 开关、限额与 `allowShellExec`
  - [x] 3.3 确保 terminal policy 与 `fs.*` / MCP 分离，不共享 enable 语义

- [x] Task 4: Prompt / Tool Guidance 对齐（AC: 3,4,7）
  - [x] 4.1 更新 system prompt/tool descriptions，明确 `terminal.run` 与 `shell.exec` 的使用差异
  - [x] 4.2 在 PromptComposer 中注入 terminal tool examples、平台 caveats 与 Runtime Execution Context 摘要
  - [x] 4.3 确保 prompt 声明、visible tools、ToolHost enforcement 使用同一份 effective policy

- [x] Task 5: Chat / Audit 集成（AC: 5,6）
  - [x] 5.1 将 terminal tool result 返回给 LLM，而不是新增 UI 终端输出面板
  - [x] 5.2 ChatPanel 仅展示 terminal 调用摘要，保持与 `fs.*` 工具的交互风格一致
  - [x] 5.3 run log / audit log 记录命令、cwd、exit status、duration 与截断预览

- [ ] Task 6: 跨平台验证（AC: 1,2,3,4,6）
  - [ ] 6.1 macOS 自动化与手工验证
  - [ ] 6.2 Windows 自动化与手工验证
  - [x] 6.3 验证 `git` / `npm test` / `bash script.sh` / directory listing 的真实行为与提示

- [x] Task 7: PTY 范围决策（AC: 8）
  - [x] 7.1 明确本 story 只包含 one-shot command execution
  - [x] 7.2 输出 follow-up story，补充 `terminal.session.*` 设计边界

## Dev Notes

- 这条需求本质上是在 Runtime 内引入“本地命令执行工具”，不是把 `exec` 菜单语义改成 shell 执行。
- `terminal.run` 应该是默认推荐路径，因为 `command + args` 比字符串 shell 命令更容易跨平台、更少 quoting bug，也更安全。
- `shell.exec` 需要明确是“高差异、高风险”的补充能力，不能让模型默认所有命令都走 shell。
- 命令执行的详细结果应该回给 LLM；UI 只需像 `fs.*` 一样展示工具调用摘要，不需要完整终端输出面板。
- 跨平台的核心边界不是“同一个命令文本 everywhere 都能运行”，而是“同一个 Runtime capability 在 macOS / Windows 上都有清晰、稳定、可观测的实现与错误模型”。
- 现有 `python.run`、`McpClientService`、`nodeService` 已经提供 `spawn`、bundled runtime、日志与错误处理的参考模式，应优先复用同类约束。
- 若要追求真正接近 Codex 的交互式终端体验，最终很可能需要单独的 PTY session story，而不是把所有交互复杂度压进这条 one-shot command story。
- 当前实现已通过定向 Vitest 用例与 `npm run build:ci`；Windows 实机验证仍待补充。

### Project Structure Notes

- 主要后端落点：
  - `crewagent-runtime/electron/services/fileSystemToolHost.ts`
  - `crewagent-runtime/electron/services/toolHost.ts`
  - `crewagent-runtime/electron/services/toolPolicy.ts`
  - `crewagent-runtime/electron/services/promptComposer.ts`
  - `crewagent-runtime/electron/services/executionEngine.ts`
  - `crewagent-runtime/electron/services/prompt-templates/system-base-rules.md`
  - `crewagent-runtime/electron/main.ts`
  - `crewagent-runtime/electron/preload.ts`
  - `crewagent-runtime/electron/stores/runtimeStore.ts`
  - `crewagent-runtime/shared/agentToolPolicy.ts`
- 建议新增：
  - `crewagent-runtime/electron/services/terminalService.ts`
  - `crewagent-runtime/electron/services/terminalService.test.ts`
- 主要前端 / 设置落点：
  - `crewagent-runtime/src/stores/appStore.ts`
  - `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx`
  - existing ChatPanel / tool-call summary rendering path

### References

- `_bmad-output/epics.md` - Epic 5 story cluster and Runtime capability roadmap
- `_bmad-output/implementation-artifacts/design-5-19-cross-platform-terminal-command-execution.md` - design decisions and scope boundary
- `_bmad-output/implementation-artifacts/tech-spec-5-19-cross-platform-terminal-command-execution.md` - implementation contract and file-level integration plan
- `_bmad-output/implementation-artifacts/5-14-tool-policy-and-chatmode-alignment.md` - tool policy merge/enforcement pattern
- `_bmad-output/implementation-artifacts/5-15-python-script-execution.md` - existing local process execution pattern
- `_bmad-output/implementation-artifacts/5-18-python-mcp-local-install-and-run.md` - cross-platform packaged runtime precedent
- `_bmad-output/implementation-artifacts/tech-spec-4-6-execute-tool-calls-sandboxed-filesystem.md` - ToolHost execution and audit logging guardrails
- `_bmad-output/implementation-artifacts/tech-spec-4-10-execute-mcp-tool-calls.md` - stdio/spawn patterns and Windows/mac packaging notes
- `crewagent-runtime/docs/runtime-spec.md` - Runtime tool execution and mount alias context
- `crewagent-runtime/docs/agent-menu-command-contract.md` - current `exec` semantics are Markdown script entry, not shell execution

> 设计文档：`_bmad-output/implementation-artifacts/design-5-19-cross-platform-terminal-command-execution.md`  
> 技术规格：`_bmad-output/implementation-artifacts/tech-spec-5-19-cross-platform-terminal-command-execution.md`

## File List

- `_bmad-output/implementation-artifacts/5-19-cross-platform-terminal-command-execution.md`
- `_bmad-output/implementation-artifacts/tech-spec-5-19-cross-platform-terminal-command-execution.md`
- `_bmad-output/implementation-artifacts/code-review-5-19-cross-platform-terminal-command-execution.md`
- `crewagent-runtime/shared/agentToolPolicy.ts`
- `crewagent-runtime/electron/services/terminalService.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.test.ts`
- `crewagent-runtime/electron/services/executionEngine.ts`
- `crewagent-runtime/electron/services/executionEngine.test.ts`
- `crewagent-runtime/electron/services/promptComposer.ts`
- `crewagent-runtime/electron/services/promptComposer.test.ts`
- `crewagent-runtime/electron/services/prompt-templates/system-tool-policy.md`
- `crewagent-runtime/electron/services/prompt-templates/system-base-rules.md`
- `crewagent-runtime/electron/services/prompt-templates/system-base-rules-chat.md`
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.css`

## Dev Agent Record

### Agent Model Used

GPT-5（Codex CLI）

### Debug Log References

- `npm -C crewagent-runtime test -- electron/services/promptComposer.test.ts`
- `npm -C crewagent-runtime test -- electron/services/fileSystemToolHost.test.ts electron/services/executionEngine.test.ts`
- `npm -C crewagent-runtime run build:ci`

### Completion Notes List

- 按照 code review 结果修复了 terminal env 继承策略：`terminal.run` / `shell.exec` 不再透传完整宿主 `process.env`，而是只保留最小基础运行环境，并叠加 `envAllowlist` 允许的模型传入变量。
- 修复 ExecutionEngine 对 terminal 非零退出码的失败语义：terminal tool result 仍返回 bounded `stdout/stderr/exitCode` 给 LLM，但 engine/audit/loop detection 现在会把 `exitCode !== 0` 视为失败。
- 移除了 terminal 工具执行后的无条件 `onFilesChanged('@project')`，避免 `git status`、`ls` 等只读命令误触发项目刷新提示。
- 新增回归测试，覆盖宿主环境变量泄漏、terminal 调用不触发文件变更提示、以及 repeated terminal non-zero exit 进入 `ENGINE_LOOP_DETECTED`。
- 在 system prompt 中新增 Runtime Execution Context 摘要，向 LLM 暴露当前平台、terminal/shell 可用性、cwd 约束和 shell 解析约定，同时避免注入敏感主机细节。
- 根据当前产品决策，将系统默认 terminal policy 调整为开启 `terminal.run` 与 `shell.exec`，同时保持 timeout / stdout / stderr 限额不变、`allowedRoots` 与 `envAllowlist` 继续默认留空。
- 统一了 Settings 页 terminal 两个多行输入框的视觉样式，使其和其他表单输入控件保持一致。
- 当前 story 继续保持 `review`，原因不在代码回归，而是 Task 6 的 macOS / Windows 端到端验证仍未全部完成。

## Change Log

- 2026-03-17: 根据 code review 修复 terminal env 隔离、非零退出码失败语义和无条件项目刷新问题，并补充对应回归测试；状态保持为 `review`，等待跨平台实机验证完成。
- 2026-03-17: 补充 Runtime Execution Context prompt 注入，在 system prompt 中显式告知 LLM 当前平台、terminal 能力与 cwd/shell 边界，并增加对应 PromptComposer 测试。
- 2026-03-17: 将 terminal 默认策略调整为启用 `terminal.run` / `shell.exec`，保留其余限制为保守默认值，并修正 Settings 页 terminal 多行输入框样式。
