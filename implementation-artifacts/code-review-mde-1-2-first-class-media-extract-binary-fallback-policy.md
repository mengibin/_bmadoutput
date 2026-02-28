# Code Review: Story MDE-1.2 – First-Class `media.extract` + Binary Fallback Policy

**Date:** 2026-02-28  
**Reviewer:** AI Code Reviewer (BMAD Method)  
**Story File:** `mde-1-2-first-class-media-extract-binary-fallback-policy.md`

---

## Summary

| Metric | Value |
|--------|-------|
| Git vs Story Discrepancies | 0 |
| HIGH Issues | 0 |
| MEDIUM Issues | 0 |
| LOW Issues | 0 |
| Post-Review Hardening | ✅ completed |
| Build | ✅ `npm -C crewagent-runtime run build:ci` passed |
| Unit Tests | ✅ `npm -C crewagent-runtime run test -- electron/services/fileSystemToolHost.test.ts electron/services/llmAdapter.test.ts` passed |

---

## HIGH ISSUES

None.

---

## MEDIUM ISSUES

None.

---

## LOW ISSUES

None.

---

## Acceptance Criteria Verification

| AC | Status | Evidence |
|----|--------|----------|
| AC1: Tool registry exposure | ✅ | `media.extract` remains visible in tools and includes `mode`; runtime also accepts provider-compatible alias route (`media-extract`) safely. |
| AC2: Direct extraction contract | ✅ | `media.extract` returns mode-aware envelope; PDF path now exposes actionable dependency diagnostics when embedded Python renderer is missing (`module/pipPackage/suggestion`). |
| AC3: fs.read binary fallback | ✅ | Binary fallback behavior remains intact and covered by tests (`FS_READ_BINARY_FALLBACK` + `suggestedTool=media.extract`). |
| AC4: Capability guard reuse | ✅ | Guard remains in `media.extract` execution paths before provider calls. |
| AC5: Backward compatibility | ✅ | `fs.read` text behavior unchanged; targeted regression tests pass. |

---

## Post-Review Hardening Notes

- Added tool-name compatibility hardening in LLM adapter: decode uses request-time tool mapping (fixes `media-extract` in non-stream + stream responses).
- Added ToolHost dispatch alias normalization: known hyphen aliases map back to dotted names (`media-extract` → `media.extract`).
- Added PDF renderer dependency diagnostics: missing `pypdfium2` now yields actionable `MEDIA_DECODE_FAILED` details.
- Added regression tests for all above paths.

---

## Verdict

**✅ Review passed:** no blocking HIGH/MEDIUM findings remain after MDE-1.2 follow-up hardening.
