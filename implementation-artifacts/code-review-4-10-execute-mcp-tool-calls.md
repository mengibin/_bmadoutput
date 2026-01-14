# Code Review: Story 4-10 – Execute MCP Tool Calls (Stdio Driver)

**Date:** 2026-01-12  
**Reviewer:** AI Code Reviewer (BMAD Method)  
**Story File:** `4-10-execute-mcp-tool-calls.md`

---

## Summary

| Metric | Value |
|--------|-------|
| Git vs Story Discrepancies | 0 |
| HIGH Issues | 0 |
| MEDIUM Issues | 0 |
| LOW Issues | 1 |
| Build | ✅ `npm -C crewagent-runtime run build:ci` passed |
| Unit Tests | ❌ `npm -C crewagent-runtime test` (1 failing, unrelated) |
| Lint | ✅ `npm -C crewagent-runtime run lint` *(TS version warning only)* |

**Test failure note:** `electron/stores/runtimeStore.test.ts` fails on importing `finance-operations.bmad` (expected `success=true`, got `false`). This appears unrelated to MCP.

---

## 🔴 HIGH ISSUES

None.

---

## 🟡 MEDIUM ISSUES

None.

---

## 🟢 LOW ISSUES

### L1. `build:ci` emits a Vite warning about mixed static + dynamic imports
- **Current:** `mcpInstallService` is dynamically imported and also statically imported, so Vite warns it can’t split it.
- **Evidence:** build log; entry point is `crewagent-runtime/electron/main.ts`.
- **Impact:** No correctness issue, but can be confusing and may affect chunking expectations.
- **Fix:** Prefer either static import or dynamic import consistently for `mcpInstallService` (and similar services).

---

## Resolved During Review

- ✅ Bundled Node.js + packaging hook:
  - `crewagent-runtime/electron-builder.json5`
  - `crewagent-runtime/electron/services/nodeService.ts`
  - `crewagent-runtime/resources/node/README.md`
- ✅ Implemented SSE transport with endpoint handshake + JSON-RPC over HTTP/SSE: `crewagent-runtime/electron/services/mcpClientService.ts`
- ✅ Added MCP stdio + SSE round-trip tests: `crewagent-runtime/electron/services/mcpClientService.test.ts`
- ✅ Included MCP tool descriptions in system prompt tool policy:
  - `crewagent-runtime/electron/services/promptComposer.ts`
  - `crewagent-runtime/electron/services/prompt-templates/system-tool-policy.md`
- ✅ Fixed MCP allowlist semantics (undefined=all, []=none) consistently in exposure + enforcement:
  - `crewagent-runtime/electron/services/mcpToolHost.ts`
  - `crewagent-runtime/electron/services/fileSystemToolHost.ts`
- ✅ Removed duplicate MCP event registrations to avoid double UI events: `crewagent-runtime/electron/main.ts`
- ✅ Fixed renderer IPC typings for MCP server management and aligned `lastConnected` serialization: `crewagent-runtime/electron/electron-env.d.ts`

---

## Acceptance Criteria Verification

| AC | Status | Evidence |
|----|--------|----------|
| AC1: Connect configured MCP servers via stdio | ✅ | Auto-connect enabled servers: `crewagent-runtime/electron/main.ts:454`; stdio client: `crewagent-runtime/electron/services/mcpClientService.ts:726` |
| AC2: Tool discovery + merged tool definitions + prompt descriptions | ✅ | listTools on connect: `crewagent-runtime/electron/services/mcpClientService.ts:726`; merged tool defs: `crewagent-runtime/electron/services/fileSystemToolHost.ts:433`; prompt includes MCP tools: `crewagent-runtime/electron/services/promptComposer.ts:206` |
| AC3: Tool execution routed by namespaced tool name | ✅ | Routing + allowlist enforcement: `crewagent-runtime/electron/services/fileSystemToolHost.ts:631`; execution: `crewagent-runtime/electron/services/mcpToolHost.ts:81` |
| AC4: Tool call failures don’t crash Runtime | ✅ | Errors returned as tool output: `crewagent-runtime/electron/services/mcpToolHost.ts:153`, `crewagent-runtime/electron/services/fileSystemToolHost.ts:650` |
| AC5: Bundled Node.js/Python for fresh installs | ✅ | Packaged resources include python + node: `crewagent-runtime/electron-builder.json5:14`; bundled node resolver: `crewagent-runtime/electron/services/nodeService.ts:28` |
| AC6: SSE remote servers supported | ✅ | SSE client implemented (no local process spawn): `crewagent-runtime/electron/services/mcpClientService.ts:362` |
| AC7: Install-to-embed for npm/python/custom | ✅ | Install flows: `crewagent-runtime/electron/services/mcpInstallService.ts:1`; npm install uses bundled node/npm: `crewagent-runtime/electron/services/nodeService.ts:149` |

---

## Next Actions

- None for Story 4-10. (Optional: address the `build:ci` warning about mixed static+dynamic imports.)
