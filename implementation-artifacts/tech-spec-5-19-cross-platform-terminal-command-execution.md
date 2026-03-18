# Tech Spec: Story 5.19 - Cross-Platform Terminal Command Execution

**Created:** 2026-03-17 10:37:32 +0800  
**Status:** Ready for Development  
**Source Story:** `_bmad-output/implementation-artifacts/5-19-cross-platform-terminal-command-execution.md`

## Overview

### Problem Statement

CrewAgent Runtime currently lets the LLM read/write files and run bundled Python, but it does not expose a general builtin command execution capability comparable to Codex-style local command runs. The user goal for Story 5.19 is to let the LLM directly execute local project commands such as `git`, `npm test`, `ls`, and `bash script.sh` through Runtime tools.

The implementation must satisfy these constraints:

- `terminal.run` and `shell.exec` are builtin tools, not menu `exec` handlers.
- Full bounded results go back to the LLM.
- ChatPanel should only show tool invocation summary, not terminal output.
- macOS and Windows are both first-class supported targets.
- MVP is one-shot only: no PTY session, no persistent interactive shell, no interactive stdin.

### Solution

Add a dedicated terminal execution layer in Electron Main and expose it through builtin Runtime tools:

- `terminal.run`
  - direct executable + args
  - `spawn(..., { shell: false })`
  - default / preferred path
- `shell.exec`
  - one-shot shell script string
  - platform-specific shell resolution
  - separately gated by policy

Integrate terminal execution at four layers:

1. `AgentToolPolicy` + settings gain an independent `terminal` branch.
2. `FileSystemToolHost` exposes and enforces terminal tools independently from `fs.*`.
3. Prompt/tool descriptions teach the model how and when to use `terminal.run` vs `shell.exec`, and inject a Runtime Execution Context summary with platform/capability constraints.
4. `ExecutionEngine` keeps full results in the LLM/tool protocol, but emits only summary-safe payloads to streaming UI indicators.

### Scope (In / Out)

**In scope**

- One-shot local command execution via `terminal.run` and `shell.exec`
- macOS / Windows shell resolution and process-tree termination
- timeout / abort / bounded output capture
- terminal-specific tool policy and settings
- prompt guidance and tool descriptions
- Runtime Execution Context prompt summary (platform, terminal capability flags, cwd policy, shell resolution guidance)
- summary-only ChatPanel indicator flow
- audit logging and run-log support

**Out of scope / Deferred**

- PTY-backed persistent sessions (`terminal.session.*`)
- interactive stdin / REPL flows
- watch mode / long-lived GUI interaction / persistent shell sessions
- terminal emulator UI or terminal panel
- changing menu `exec` semantics

## Context for Development

### Existing Codebase Patterns

- Builtin tools are exposed through `FileSystemToolHost.getVisibleTools()` and executed by `FileSystemToolHost.executeToolCall()`.
- Tool results are serialized as JSON and appended to conversation history by `ExecutionEngine`.
- Tool stream indicators are emitted separately from the actual `role='tool'` messages.
- `PromptComposer` currently builds tool policy text from `AgentToolPolicy` and injects base rules from:
  - `system-base-rules.md`
  - `system-base-rules-chat.md`
  - `system-tool-policy.md`
- Current `AgentToolPolicy` only has `fs` and `mcp`; `terminal` must be added without overloading `fs`.

### Relevant Files

- Story: `_bmad-output/implementation-artifacts/5-19-cross-platform-terminal-command-execution.md`
- Tool result contracts: `crewagent-runtime/electron/services/toolHost.ts`
- Builtin tool host: `crewagent-runtime/electron/services/fileSystemToolHost.ts`
- Tool policy merge: `crewagent-runtime/electron/services/toolPolicy.ts`
- Runtime settings: `crewagent-runtime/electron/stores/runtimeStore.ts`
- Shared tool policy type: `crewagent-runtime/shared/agentToolPolicy.ts`
- Prompt composition:
  - `crewagent-runtime/electron/services/promptComposer.ts`
  - `crewagent-runtime/electron/services/prompt-templates/system-base-rules.md`
  - `crewagent-runtime/electron/services/prompt-templates/system-base-rules-chat.md`
  - `crewagent-runtime/electron/services/prompt-templates/system-tool-policy.md`
- Tool streaming / persistence:
  - `crewagent-runtime/electron/services/executionEngine.ts`
  - `crewagent-runtime/electron/main.ts`
  - `crewagent-runtime/electron/preload.ts`
  - `crewagent-runtime/electron/electron-env.d.ts`
- Existing indicator UI:
  - `crewagent-runtime/src/pages/RunsPage/components/ToolStatusIndicator.tsx`
  - `crewagent-runtime/src/pages/RunsPage/components/MessageItem.tsx`
  - `crewagent-runtime/src/pages/RunsPage/RunWorkspace.tsx`
  - `crewagent-runtime/src/pages/WorksPage/WorksPage.tsx`

### Technical Decisions

1. **Terminal capability is its own policy branch**
   - Do not reuse `fs.enabled` or `mcp.enabled`.
   - `terminal` must be independently visible/enforced.

2. **`terminal.run` is the default path**
   - If a command naturally fits executable + args, the model should use `terminal.run`.
   - `bash script.sh` remains in-scope for `terminal.run`.

3. **`shell.exec` is one-shot only**
   - Supported for shell parsing.
   - Not a persistent shell session.

4. **Full results go to the model, summary only goes to UI**
   - The `role='tool'` message keeps bounded stdout/stderr.
   - Stream indicator payloads for terminal tools must omit raw terminal output.

5. **Cross-platform means contract parity, not command-text parity**
   - The Runtime must behave consistently across macOS and Windows.
   - Command availability still depends on OS/shell.

6. **System prompt should expose execution context, not raw machine internals**
   - Inject platform, terminal capability flags, cwd policy, and shell-resolution guidance.
   - Do not expose full host `process.env`, usernames, machine names, or absolute allowlisted root details.

## Contracts

### Shared Tool Policy Extension

Extend `AgentToolPolicy` with a `terminal` branch:

```ts
export interface AgentToolPolicy {
  fs: {
    enabled: boolean
    maxReadBytes?: number
    maxWriteBytes?: number
  }
  mcp: {
    enabled: boolean
    allowedServers?: string[]
  }
  terminal: {
    enabled: boolean
    allowShellExec?: boolean
    maxStdoutBytes?: number
    maxStderrBytes?: number
    timeoutMs?: number
    allowedRoots?: string[]
    envAllowlist?: string[]
  }
}
```

### Policy Merge Rules

Extend `mergeToolPolicies()` to merge `terminal` with the same “can only tighten” rule:

- `enabled`: `system.enabled && (agent.enabled ?? true)`
- `allowShellExec`: `system.allowShellExec && (agent.allowShellExec ?? true)`
- `maxStdoutBytes` / `maxStderrBytes` / `timeoutMs`: numeric `min(system, agent)` ignoring `undefined`
- `allowedRoots`: intersection if both provided, otherwise the defined side
- `envAllowlist`: intersection if both provided, otherwise the defined side

### Runtime Settings Defaults

`defaultSettings.tools` should gain:

```ts
terminal: {
  enabled: true,
  allowShellExec: true,
  maxStdoutBytes: 50_000,
  maxStderrBytes: 20_000,
  timeoutMs: 60_000,
  allowedRoots: [],
  envAllowlist: [],
}
```

Notes:

- Default terminal execution is enabled for the Codex-like local command workflow.
- `shell.exec` is also enabled by default, but prompt guidance must still steer the model toward `terminal.run` unless shell parsing is actually required.
- Empty `allowedRoots` means “no extra roots”; `@project` remains implicitly allowed.
- Empty `envAllowlist` means the model does not get arbitrary custom env keys unless the operator explicitly allows them.

### Tool Result Shapes

Add terminal result types in `toolHost.ts`:

```ts
export type TerminalRunResult =
  | {
      ok: true
      stdout: string
      stderr: string
      exitCode: number
      durationMs: number
      stdoutTruncated?: boolean
      stderrTruncated?: boolean
      shellResolved?: string
      platform: 'darwin' | 'win32'
    }
  | { ok: false; error: ToolError }
```

`ToolResult` must include `TerminalRunResult`.

### Tool Schemas

#### `terminal.run`

```ts
{
  type: 'function',
  function: {
    name: 'terminal.run',
    description: 'Execute a local command using executable + args. Prefer this over shell.exec when possible.',
    parameters: {
      type: 'object',
      properties: {
        command: { type: 'string' },
        args: { type: 'array', items: { type: 'string' } },
        cwd: { type: 'string' },
        timeoutMs: { type: 'integer', minimum: 1 },
        env: {
          type: 'object',
          additionalProperties: { type: 'string' },
        },
      },
      required: ['command'],
      additionalProperties: false,
    },
  },
}
```

#### `shell.exec`

```ts
{
  type: 'function',
  function: {
    name: 'shell.exec',
    description: 'Execute a one-shot shell command string. Use only when shell parsing is required.',
    parameters: {
      type: 'object',
      properties: {
        script: { type: 'string' },
        shell: { type: 'string', enum: ['auto', 'bash', 'zsh', 'powershell', 'cmd'] },
        cwd: { type: 'string' },
        timeoutMs: { type: 'integer', minimum: 1 },
        env: {
          type: 'object',
          additionalProperties: { type: 'string' },
        },
      },
      required: ['script'],
      additionalProperties: false,
    },
  },
}
```

### Runtime Execution Context Prompt Block

`PromptComposer` should inject a compact system-message block inside the tool-policy prompt that includes:

- current platform: `darwin` / `win32` plus human-friendly OS label
- `terminal.run` availability
- `shell.exec` availability
- default working root: `@project`
- cwd policy summary: `@project only` or `@project plus configured allowlisted roots`
- shell auto-resolution summary for the current platform
- reminder that command availability is determined at execution time via structured errors such as `COMMAND_NOT_FOUND` / `SHELL_NOT_AVAILABLE`

Guardrails:

- Do not include raw environment variable values.
- Do not include usernames, hostnames, full PATH contents, or absolute root paths.
- Keep the block deterministic and low-churn so prompt diffs stay stable between runs on the same machine.

### Errors

Expected terminal-specific error codes:

- `TOOL_NOT_AVAILABLE`
- `COMMAND_NOT_FOUND`
- `SHELL_NOT_AVAILABLE`
- `CWD_NOT_ALLOWED`
- `EXEC_TIMEOUT`
- `EXEC_ABORTED`
- `OUTPUT_LIMIT_EXCEEDED`
- `INTERACTIVE_COMMAND_NOT_SUPPORTED` (optional explicit detection / hint)
- `E_INVALID_ARGS`
- `E_INTERNAL`

## Architecture

### Terminal Service

Add a new Electron Main service:

- `crewagent-runtime/electron/services/terminalService.ts`

Suggested API:

```ts
export type TerminalPolicy = {
  enabled: boolean
  allowShellExec: boolean
  maxStdoutBytes: number
  maxStderrBytes: number
  timeoutMs: number
  allowedRoots?: string[]
  envAllowlist?: string[]
}

export type TerminalRunParams = {
  command: string
  args?: string[]
  cwd?: string
  timeoutMs?: number
  env?: Record<string, string>
}

export type ShellExecParams = {
  script: string
  shell?: 'auto' | 'bash' | 'zsh' | 'powershell' | 'cmd'
  cwd?: string
  timeoutMs?: number
  env?: Record<string, string>
}

export interface TerminalService {
  run(params: TerminalRunParams, options: TerminalExecutionOptions): Promise<TerminalRunResult>
  execShell(params: ShellExecParams, options: TerminalExecutionOptions): Promise<TerminalRunResult>
}
```

`TerminalExecutionOptions` should carry:

- resolved `projectRoot`
- effective terminal policy
- `AbortSignal`

### Execution Flow

#### `terminal.run`

1. Validate `command` is a non-empty string.
2. Validate `args` is a string array if provided.
3. Resolve `cwd` against `@project` and allowed roots.
4. Filter env vars using `envAllowlist`.
5. Spawn with:

```ts
spawn(command, args, {
  shell: false,
  cwd,
  env,
  detached: process.platform !== 'win32',
  windowsHide: true,
})
```

6. Capture bounded stdout/stderr.
7. On timeout / abort, kill the process tree.
8. Return bounded result to ToolHost.

#### `shell.exec`

1. Validate `allowShellExec`.
2. Resolve requested shell or choose default shell for the platform.
3. Convert `script` to shell-specific argv:

- bash / zsh:
  - `['-lc', script]`
- pwsh / powershell:
  - `['-NoLogo', '-NoProfile', '-NonInteractive', '-Command', script]`
- cmd:
  - `['/d', '/s', '/c', script]`

4. Execute via `spawn(shellExecutable, shellArgs, { shell: false, ... })`.
5. Reuse the same capture / timeout / abort path as `terminal.run`.

### Shell Resolution

#### macOS

Resolution order:

1. explicit shell if requested and available
2. configured/default POSIX shell if implemented
3. `/bin/bash`
4. `/bin/zsh`

#### Windows

Resolution order:

1. explicit shell if requested and available
2. `pwsh.exe`
3. `powershell.exe`
4. `cmd.exe`

Explicit `bash` on Windows is only allowed if a real bash executable is discoverable. Otherwise return `SHELL_NOT_AVAILABLE`.

### CWD Resolution Rules

Allowed input forms:

- omitted -> use `@project`
- `@project`
- `@project/subdir`
- absolute path only if it resolves inside configured `allowedRoots`

Rejected:

- `@pkg`
- `@state`
- mount aliases other than `@project`
- paths outside `@project` / allowlist

Rationale:

- keep execution scoped to project work by default
- avoid running arbitrary host commands in unrelated directories

### Environment Rules

Do not pass raw unfiltered model-provided env directly.

Recommended behavior:

1. Start with inherited process env required for child execution:
   - `PATH`
   - `HOME` / `USERPROFILE`
   - `TMPDIR` / `TEMP`
   - `SystemRoot` on Windows
2. Apply model-provided `env` only for keys allowed by `envAllowlist`.
3. Redact env values from audit logs.

### Output Capture / Truncation

Reuse the bounded-buffer pattern already used in `python.run`:

- maintain separate stdout/stderr buffers
- enforce byte limits, not character limits
- set `stdoutTruncated` / `stderrTruncated` when limits are hit
- preserve partial output on timeout / abort

### Timeout / Abort / Process Tree Kill

Use a watchdog timer:

- timeout -> mark `EXEC_TIMEOUT`
- user stop / abort signal -> mark `EXEC_ABORTED`

Process termination:

- POSIX: kill process group via `process.kill(-pid, ...)` when spawned detached
- Windows: use `taskkill /T /F /PID <pid>` or equivalent child-tree termination helper

This must be implemented as a dedicated helper rather than copying POSIX-only kill logic.

## ToolHost Integration

### `FileSystemToolHost`

Refactor `getVisibleTools()` so it no longer returns early based only on `fs.enabled`.

Desired behavior:

- always include `ui.ask_user`
- include `fs.*` only if `fs.enabled`
- include `terminal.run` if `terminal.enabled`
- include `shell.exec` only if `terminal.enabled && allowShellExec`
- include MCP tools only if `mcp.enabled`

`executeToolCall()` must enforce terminal policy before dispatching:

- `terminal.run` rejected with `TOOL_NOT_AVAILABLE` if terminal disabled
- `shell.exec` rejected with `TOOL_NOT_AVAILABLE` if shell execution disabled

Recommended helper methods:

- `getTerminalPolicy(packageId, agentId)`
- `summarizeTerminalArgsForAudit(args)`

### Conversation / Tool Protocol

No change to core protocol:

- `ExecutionEngine` still appends the full `role='tool'` JSON result for LLM continuity.
- The terminal tool result remains available to later turns.

## Prompt Integration

### PromptComposer / Tool Policy Text

`PromptComposer.buildToolPolicy()` currently renders only `fs` and `mcp`.

It must be extended to include terminal policy state, for example:

- `terminal: enabled (allowShellExec=false, timeoutMs=60000, maxStdoutBytes=50000, maxStderrBytes=20000)`

### System Guidance

Update both:

- `system-base-rules.md`
- `system-base-rules-chat.md`

Add guidance such as:

- prefer `terminal.run` when the command fits executable + args
- use `shell.exec` only when shell parsing is required
- cross-platform tool does not mean command-string portability
- terminal tools are one-shot only
- do not use terminal tools for GUI-bound / REPL / watch-mode interaction

### Tool Policy Template

Update `system-tool-policy.md` so terminal capability is declared alongside `fs` and `mcp`.

## UI / Streaming Integration

### Summary-Only Stream Payload

Current streaming UI receives raw `result` in `onLlmStreamToolEnd`, and `RunWorkspace` / `WorksPage` stringifies that payload into indicator details. For terminal tools this would expose stdout/stderr in the UI, which conflicts with the story.

Change `ExecutionEngine` to emit a summary-safe UI payload for terminal tools:

```ts
type ToolUiSummary =
  | undefined
  | {
      ok: boolean
      exitCode?: number
      durationMs?: number
      platform?: 'darwin' | 'win32'
      shellResolved?: string
      stdoutTruncated?: boolean
      stderrTruncated?: boolean
      errorCode?: string
    }
```

Behavior:

- store full result in conversation `role='tool'`
- emit only `ToolUiSummary` to `onLlmStreamToolEnd` for terminal tools
- existing indicator UI can keep using `toolName`, `status`, and duration

This keeps terminal output available to the model without rendering it in ChatPanel.

### Existing UI Components

No new terminal UI is required. Reuse:

- `ToolStatusIndicator.tsx`
- `MessageItem.tsx`
- `RunWorkspace.tsx`
- `WorksPage.tsx`

Minimal renderer changes may still be needed so terminal tool detail expansion never shows raw stdout/stderr.

## Implementation Plan by File

| File | Change |
|:-----|:-------|
| `crewagent-runtime/shared/agentToolPolicy.ts` | Add `terminal` policy branch. |
| `crewagent-runtime/electron/stores/runtimeStore.ts` | Add terminal defaults, load/merge/update handling. |
| `crewagent-runtime/electron/services/toolPolicy.ts` | Extend merge logic for `terminal`. |
| `crewagent-runtime/electron/services/toolHost.ts` | Add terminal result type to `ToolResult`. |
| `crewagent-runtime/electron/services/terminalService.ts` | New service for execution, capture, shell resolution, timeout, and kill-tree behavior. |
| `crewagent-runtime/electron/services/fileSystemToolHost.ts` | Expose terminal tools, enforce terminal policy, dispatch to `terminalService`. |
| `crewagent-runtime/electron/services/promptComposer.ts` | Render terminal policy text. |
| `crewagent-runtime/electron/services/prompt-templates/system-base-rules.md` | Add run-mode terminal guidance. |
| `crewagent-runtime/electron/services/prompt-templates/system-base-rules-chat.md` | Add chat-mode terminal guidance. |
| `crewagent-runtime/electron/services/prompt-templates/system-tool-policy.md` | Add terminal policy description template. |
| `crewagent-runtime/electron/services/executionEngine.ts` | Emit summary-only stream payload for terminal tools while keeping full tool message content for the model. |
| `crewagent-runtime/electron/main.ts` | No contract expansion expected beyond existing stream event pass-through, unless typed payload adjustments are needed. |
| `crewagent-runtime/electron/preload.ts` | Update typings only if stream payload shape changes. |
| `crewagent-runtime/electron/electron-env.d.ts` | Update terminal-related event payload typings. |
| `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx` | Add terminal policy controls. |
| `crewagent-runtime/src/pages/RunsPage/RunWorkspace.tsx` | Keep summary rendering behavior for terminal tools. |
| `crewagent-runtime/src/pages/WorksPage/WorksPage.tsx` | Same as RunWorkspace if tool stream indicators are shown there. |

## Test Plan

### Unit

- `toolPolicy.ts`
  - terminal enabled merge uses AND
  - `allowShellExec` uses AND
  - timeout/stdout/stderr limits use `min`
  - allowlists use intersection
- `terminalService.ts`
  - command validation
  - shell resolution on macOS and Windows
  - cwd validation
  - env allowlist filtering
  - bounded output capture
  - timeout and abort behavior
  - structured errors for missing command / missing shell
- `fileSystemToolHost.ts`
  - terminal visibility independent from fs visibility
  - `shell.exec` hidden/rejected when `allowShellExec=false`
  - terminal tools still available when `fs.enabled=false`
- `promptComposer.ts`
  - terminal tool policy text rendered correctly
  - prompt includes usage guidance / platform caveat text
- `executionEngine.ts`
  - full tool result persists to conversation
  - terminal tool stream payload is summary-only

### Integration

- macOS:
  - `terminal.run` with `git status`
  - `terminal.run` with `npm test`
  - `terminal.run` with `bash script.sh`
  - `shell.exec` with `&&` or pipe usage
- Windows:
  - `terminal.run` with `git status`
  - `terminal.run` with `npm test`
  - `shell.exec` with PowerShell
  - explicit `bash` returns `SHELL_NOT_AVAILABLE` when absent

### Manual

1. Enable terminal tools, keep `allowShellExec` off:
   - `terminal.run` visible and usable
   - `shell.exec` hidden or rejected
2. Enable `allowShellExec`:
   - `shell.exec` visible and usable
3. Run a long command and confirm timeout returns partial output.
4. Stop a running command and confirm abort result.
5. Expand tool indicator in ChatPanel and confirm terminal stdout/stderr is not rendered.
6. Verify run log / audit log contain summary + redacted/truncated preview.

## Risks / Mitigations

| Risk | Level | Mitigation |
|:-----|:------|:-----------|
| Terminal policy accidentally coupled to `fs.enabled` due to current host structure | High | Refactor additive tool visibility instead of fs-only early return. |
| Raw terminal output leaks into ChatPanel through stream event payloads | High | Emit summary-only UI payload for terminal tools. |
| Windows process tree termination is incomplete | High | Implement dedicated Windows kill-tree helper and test explicitly. |
| Shell quoting differs across platforms | Medium | Keep `terminal.run` as the preferred path; document shell-specific variance. |
| Model overuses `shell.exec` | Medium | Put examples and explicit preference text in tool descriptions and prompts. |
| GUI/watch/REPL flows hang | Medium | Document as out-of-scope; enforce timeout / abort path. |
