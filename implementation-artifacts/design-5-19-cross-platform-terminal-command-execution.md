# Design: Cross-Platform Terminal Command Execution

**Story:** `5-19-cross-platform-terminal-command-execution.md`  
**Design principle:** Codex-like one-shot command execution, LLM-first result handling, summary-only UI

---

## Objectives

1. Expose builtin terminal tools so the LLM can directly execute local project commands.
2. Make `terminal.run` the default path and keep `shell.exec` as an explicit shell-only path.
3. Support macOS and Windows as first-class targets from the first release.
4. Keep UI scope minimal: reuse existing tool indicators instead of adding a terminal panel.

---

## Scope

### In scope

- Add builtin tools `terminal.run` and `shell.exec`.
- Return bounded execution results to the LLM.
- Add independent terminal tool policy/settings.
- Update prompt/tool guidance so the model knows when to use each tool.
- Show terminal calls in ChatPanel using existing tool status indicator patterns.
- Support timeout, abort, truncation, audit logging, and macOS/Windows process termination.

### Out of scope

- PTY-backed persistent terminal sessions.
- Interactive stdin flows.
- Full terminal emulator UI or dedicated terminal page.
- Commands that require persistent GUI interaction, watch mode, REPL loops, or other long-lived interactive sessions.
- Changing the existing menu `exec` contract away from Markdown-script execution.

---

## UX / UI

### ChatPanel behavior

- Reuse the existing tool status indicator flow used for `fs.*` and other tools.
- Show:
  - tool name
  - running/completed state
  - duration
- Do not render full `stdout` / `stderr` in ChatPanel for terminal tools.
- If expanded, terminal tool indicators should show only a concise execution summary, not raw terminal output.

### Settings

Add a terminal section under tool settings:

- `Enable terminal tools`
- `Allow shell.exec`
- `Default timeout`
- `Max stdout bytes`
- `Max stderr bytes`
- optional allowlisted roots
- optional env allowlist

This section must be independent from `fs.*` and MCP toggles.

---

## Tool Behavior

### `terminal.run`

Use for commands that naturally fit executable + args, including:

- `git status`
- `npm test`
- `bash script.sh`

Characteristics:

- shell disabled
- fewer quoting bugs
- better portability across macOS and Windows
- preferred path in prompt guidance

### `shell.exec`

Use only when shell parsing is required, for example:

- `npm test && npm run lint`
- `ls -la | grep src`
- redirection / globbing / variable expansion

Characteristics:

- explicit shell resolution per platform
- higher variance and risk than `terminal.run`
- may be disabled separately even when terminal tools are enabled

---

## Cross-Platform Boundary

“Cross-platform” means:

- the Runtime exposes the same capability contract on macOS and Windows
- errors are structured consistently
- timeout / abort / logging behavior is consistent

It does **not** mean:

- every command string is portable across both operating systems

Examples:

- `git status` is often portable
- `ls -la` is not guaranteed on Windows
- `Get-ChildItem` is not guaranteed on macOS

The prompt and tool descriptions must teach this boundary directly.

---

## Result Handling

- Full bounded tool results go back to the LLM as the tool result payload.
- ChatPanel should only receive a summary-safe payload.
- Audit/run logs should record command, cwd, exit status, duration, and redacted/truncated previews.

This keeps the model effective without turning the UI into a terminal viewer.

---

## Affected Files (Design-Level)

- `crewagent-runtime/shared/agentToolPolicy.ts`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/services/toolPolicy.ts`
- `crewagent-runtime/electron/services/toolHost.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.ts`
- `crewagent-runtime/electron/services/terminalService.ts` (new)
- `crewagent-runtime/electron/services/promptComposer.ts`
- `crewagent-runtime/electron/services/prompt-templates/system-base-rules.md`
- `crewagent-runtime/electron/services/prompt-templates/system-base-rules-chat.md`
- `crewagent-runtime/electron/services/prompt-templates/system-tool-policy.md`
- `crewagent-runtime/electron/services/executionEngine.ts`
- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/preload.ts`
- `crewagent-runtime/electron/electron-env.d.ts`
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx`
- `crewagent-runtime/src/pages/RunsPage/RunWorkspace.tsx`
- `crewagent-runtime/src/pages/WorksPage/WorksPage.tsx`

---

## Manual Test Checklist

1. `terminal.run` executes `git status` and returns result to the model.
2. `terminal.run` executes `npm test` in `@project`.
3. `terminal.run` executes `bash script.sh` when bash is available.
4. `shell.exec` runs a command with `&&` or `|`.
5. Disabled terminal policy hides/rejects terminal tools while `fs.*` behavior stays unchanged.
6. Disabled `allowShellExec` keeps `terminal.run` available but rejects `shell.exec`.
7. ChatPanel shows terminal tool indicator only, without full stdout/stderr.
8. Timeout kills a long-running command and preserves partial output in the tool result and audit log.
9. Stop action aborts a running command on macOS.
10. Stop action aborts a running command on Windows.
