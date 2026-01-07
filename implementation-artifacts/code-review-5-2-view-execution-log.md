# Code Review: Story 5-2 – Rich Execution Log Panel

**Date:** 2025-12-31  
**Reviewer:** AI Code Reviewer (BMAD Method)  
**Story File:** `5-2-view-execution-log.md`

---

## Summary

| Metric | Value |
|--------|-------|
| Git vs Story Discrepancies | 3 |
| HIGH Issues | 2 |
| MEDIUM Issues | 4 |
| LOW Issues | 4 |
| Build | ✅ `npm -C crewagent-runtime run build:ci` passed |
| Unit Tests | N/A (no `test` script) |
| Lint | ❌ 5 errors / 2 warnings (`npm -C crewagent-runtime run lint`) |

---

## 🔴 CRITICAL ISSUES (HIGH)

### H1. No log producer (panel is not testable end-to-end)
- **Description:** The UI supports rendering logs, but nothing produces log entries during normal usage. The story’s Tasks list claims a “Simulate Logs” button exists, but the codebase has no such UI entry point and nothing calls `useLogs().addLog()`.
- **Evidence:**
  - `crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.tsx` renders the panel but only reads `logs` and calls `clearLogs()`; `addLog()` is unused.
  - `crewagent-runtime/src/hooks/useLogs.ts` exposes `addLog()`, but it is never called anywhere.
  - `rg -n "\\baddLog\\(" crewagent-runtime` shows only IPC handler + store method + hook.
- **Impact:** Acceptance Criteria #2/#3/#5 cannot be validated by normal UI usage (panel stays empty).
- **Fix:** Add a dev-only “Simulate Logs” action (as the story says) or wire logs to real runtime events (workflow/agent execution).

### H2. “Real-time” behavior not implemented (no push/stream)
- **Description:** The story calls for “real-time execution log”, and mentions future streaming via `execution:log`. Current implementation is pull-based (`logs:get`) and only refreshes on mount or when renderer itself calls `logs:add`.
- **Evidence:**
  - `crewagent-runtime/src/hooks/useLogs.ts` loads once on mount; no subscription, polling, or IPC event listener.
  - `crewagent-runtime/electron/main.ts` implements `logs:get|add|clear` but no `execution:log` event channel.
- **Impact:** If backend adds logs (future engine loop), the UI will not update live.
- **Fix:** Implement a push channel (e.g. `execution:log`) and a renderer subscription that appends entries incrementally (no full refresh).

---

## 🟡 MEDIUM ISSUES

### M1. Logs are global (not per project / not cleared on project switch)
- **Description:** Tech spec states logs are “Transient, cleared on reload/project switch”. Current store holds a single in-memory `logs` array with no project/run scoping and no automatic clearing on project changes.
- **Evidence:** `crewagent-runtime/electron/stores/runtimeStore.ts` stores `private logs: LogEntry[] = []` with `getLogs()` returning the whole list.
- **Impact:** Logs can leak across projects in the same app session, confusing users.
- **Fix:** At minimum, clear logs when changing/opening projects; ideally scope logs by `projectId` and (future) `runId`.

### M2. IPC payload validation missing; `any` used (lint error)
- **Description:** Renderer can send arbitrary payloads via `logs:add`. The handler uses `entry: any` and forwards directly to `RuntimeStore.addLog(entry)` without validation or defaults.
- **Evidence:** `crewagent-runtime/electron/main.ts` `ipcMain.handle('logs:add', async (_, entry: any) => ...)` (also triggers ESLint `no-explicit-any`).
- **Impact:** Invalid entries can break rendering (missing `timestamp`, huge nested `toolOutput`, etc.) and cause memory spikes.
- **Fix:** Replace `any` with `unknown` and validate/normalize using a type guard or Ajv schema (Ajv is already a dependency).

### M3. Duplicate headers (“Execution Log” shown twice)
- **Description:** Workspace panel has its own header, and `LogViewer` also renders a header with title + toolbar.
- **Evidence:**
  - `crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.tsx` renders `.log-panel-header` with “📋 Execution Log”.
  - `crewagent-runtime/src/components/logs/LogViewer.tsx` renders `.log-viewer-header` with the same title and controls.
- **Impact:** UI looks cramped/redundant and wastes vertical space.
- **Fix:** Keep the toolbar in one place (either move title/close into `LogViewer` and remove the wrapper header, or make `LogViewer` render list-only and keep toolbar in wrapper).

### M4. UX features in design doc not implemented (shortcuts, resize)
- **Description:** The story design specifies panel resize handle and keyboard shortcuts (`Cmd/Ctrl+L`, `Cmd/Ctrl+K`). Current implementation is fixed height (300px) and uses a floating button, with no shortcuts.
- **Impact:** Diverges from design expectations and VS Code-like interaction.
- **Fix:** Either implement these UX requirements now, or explicitly defer them to a follow-up story and update the story’s “Dev Agent Record” / Tasks accordingly.

---

## 🟢 LOW ISSUES

### L1. `LogLevel` includes `success` but filter UI omits it
- **Evidence:** `crewagent-runtime/electron/stores/runtimeStore.ts` defines `LogLevel = ... | 'success'`, but `LogViewer` dropdown has no “Success”.
- **Fix:** Add “Success” filter or remove the level if not needed.

### L2. `filter` state typed as `string`
- **Evidence:** `crewagent-runtime/src/components/logs/LogViewer.tsx` `useState<string>('all')`.
- **Fix:** Use a union type: `'all' | LogLevel`.

### L3. Timestamp formatting hardcodes locale
- **Evidence:** `LogViewer.tsx` uses `toLocaleTimeString('en-US', ...)`.
- **Fix:** Use `undefined` locale (system default) or a runtime setting.

### L4. Story doc formatting: trailing code fence
- **Evidence:** `_bmad-output/implementation-artifacts/5-2-view-execution-log.md` ends with an extra “```”.
- **Fix:** Remove the stray fence to avoid markdown rendering issues.

---

## Acceptance Criteria Verification

| AC | Status | Evidence |
|----|--------|----------|
| AC1: Log Panel available | ✅ | `crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.tsx` renders toggle + panel |
| AC2: “Thinking Block” appears | ⚠️ Partial | `ThinkingBlock` exists, but no runtime producer to create thought logs |
| AC3: “Tool Block” appears | ⚠️ Partial | `ToolBlock` exists, but no runtime producer to create tool logs |
| AC4: Auto-scroll unless scrolled up | ✅ | `LogViewer` auto-scroll + manual scroll detection |
| AC5: Text log lines (info/error/etc) | ⚠️ Partial | `TextLogEntry` exists, but no runtime producer by default |
| AC6: Chat remains clean | ✅ | Log panel is separate UI surface |

---

## Next Actions

- [ ] [AI-Review][HIGH] Add a real log producer (dev “Simulate Logs” button or wire to workflow/agent execution events)
- [ ] [AI-Review][HIGH] Implement real-time updates (IPC push channel like `execution:log`, renderer subscription + incremental append)
- [ ] [AI-Review][MEDIUM] Scope logs per project/run (or at least clear logs on project switch)
- [ ] [AI-Review][MEDIUM] Validate `logs:add` payload and remove `any` in `crewagent-runtime/electron/main.ts`
- [ ] [AI-Review][MEDIUM] Remove duplicated header between workspace wrapper and `LogViewer`
- [ ] [AI-Review][LOW] Align filter options/types (`success`, union typing)
- [ ] [AI-Review][LOW] Fix story doc trailing “```” in `_bmad-output/implementation-artifacts/5-2-view-execution-log.md`
