# Code Review: Story 9-6 – Interactive HTML Artifact Save Bridge

**Date:** 2026-05-09  
**Reviewer:** AI Code Reviewer (BMAD Method)  
**Story File:** `9-6-interactive-html-artifact-save-bridge.md`

---

## 范围说明

- 按 Story 9-6 的 AC 对照审计 Runtime 侧 Interactive HTML Artifact 保存桥接。
- 重点检查：iframe 消息边界、路径解析、写入约束、输出类型、测试覆盖。
- 不评估工作区其他无关改动。

---

## Summary

| Metric | Value |
|--------|-------|
| HIGH Issues | 0 open |
| MEDIUM Issues | 0 open, 3 resolved |
| LOW Issues | 0 open, 3 resolved |
| Focused Tests | ✅ `npm -C crewagent-runtime test -- --run electron/stores/runtimeStore.test.ts src/components/files/fileViewers.test.tsx` passed, 92 tests |
| Build | ✅ `npm -C crewagent-runtime run build:ci` passed |
| Lint | ❌ `npm -C crewagent-runtime run lint` still fails on pre-existing `WorkflowGraphView.tsx` Fast Refresh warning, not introduced by Story 9-6 |
| Latest Re-review | ✅ 2026-05-09 re-review found no new Story 9-6 implementation issues |

---

## Latest Re-review

**Date:** 2026-05-09  
**Result:** No new findings

The current Story 9-6 implementation was re-reviewed after the M2, M3, and L3 fixes. The bridge now covers the reviewed AC-critical boundaries:

- strict base64 validation
- protocol-like output path segment rejection
- source HTML overwrite prevention, including case-variant paths on case-insensitive file systems
- realpath containment for symlinked output parent directories
- active iframe source validation
- async save result delivery to the original requester iframe

Re-run verification:

- ✅ `npm -C crewagent-runtime test -- --run electron/stores/runtimeStore.test.ts src/components/files/fileViewers.test.tsx` passed, 92 tests

No additional code changes were requested by this re-review.

---

## Resolved Issues

### M2. Source HTML overwrite check failed on case-insensitive file systems

**Status:** Fixed

**Fix**  
Runtime now checks existing output targets by filesystem identity, not only by normalized path string. This blocks case-variant overwrite attempts and also catches existing hardlink/symlink targets that resolve to the source HTML.

**Evidence**  
- Existing-file identity helper: `crewagent-runtime/electron/stores/runtimeStore.ts:225`
- Interactive target validation: `crewagent-runtime/electron/stores/runtimeStore.ts:243`
- Save path invokes real target validation before writing: `crewagent-runtime/electron/stores/runtimeStore.ts:5506`
- Regression test: `crewagent-runtime/electron/stores/runtimeStore.test.ts:192`

### M3. Symlinked output directories could escape the HTML directory boundary

**Status:** Fixed

**Fix**  
Runtime now validates the real path of the existing output parent ancestor before creating directories, and revalidates the real parent directory after creation. Existing output symlink files are rejected.

**Evidence**  
- Existing ancestor lookup: `crewagent-runtime/electron/stores/runtimeStore.ts:215`
- Real path containment helper: `crewagent-runtime/electron/stores/runtimeStore.ts:235`
- Parent revalidation after `mkdirSync`: `crewagent-runtime/electron/stores/runtimeStore.ts:5514`
- Regression test: `crewagent-runtime/electron/stores/runtimeStore.test.ts:219`

### L3. Async save results could be posted to a newer iframe after preview switches

**Status:** Fixed

**Fix**  
The renderer captures the original `event.source` after validating that it is the active iframe, and all success/error save results are posted back to that source window rather than whatever iframe ref is current when the async IPC call settles.

**Evidence**  
- `postSaveResult()` now takes a target `WindowProxy`: `crewagent-runtime/src/components/files/fileViewers.tsx:144`
- Source iframe captured from the message event: `crewagent-runtime/src/components/files/fileViewers.tsx:171`
- Async success path posts to captured source: `crewagent-runtime/src/components/files/fileViewers.tsx:207`
- Regression test: `crewagent-runtime/src/components/files/fileViewers.test.tsx:135`

### M1. Invalid base64 content was accepted and persisted as a successful binary save

**Status:** Fixed

**Fix**  
Added strict base64 validation before writing binary interactive artifact outputs. Runtime rejects whitespace, invalid characters, invalid padding, and decoded values that do not round-trip to the original base64 string.

**Evidence**  
- Strict decoder: `crewagent-runtime/electron/stores/runtimeStore.ts:205`
- Save path uses strict decoder: `crewagent-runtime/electron/stores/runtimeStore.ts:5518`
- Valid base64 regression test: `crewagent-runtime/electron/stores/runtimeStore.test.ts:99`
- Malformed base64 rejection test: `crewagent-runtime/electron/stores/runtimeStore.test.ts:171`

### L1. Protocol-like `outputPath` validation only checked the beginning of the path

**Status:** Fixed

**Fix**  
Runtime rejects protocol-like markers in any output path segment, so paths like `./outputs/https://example/result.txt` are rejected instead of being normalized into a confusing local folder structure.

**Evidence**  
- Segment-level protocol check: `crewagent-runtime/electron/stores/runtimeStore.ts:193`
- Regression test: `crewagent-runtime/electron/stores/runtimeStore.test.ts:148`

### L2. No renderer-level coverage for the iframe `postMessage` bridge

**Status:** Fixed

**Fix**  
Added jsdom component tests for `HtmlPreview` covering active iframe save forwarding, save result callback, outside-source rejection, and stale iframe result targeting.

**Evidence**  
- Bridge implementation: `crewagent-runtime/src/components/files/fileViewers.tsx:139`
- Renderer bridge tests: `crewagent-runtime/src/components/files/fileViewers.test.tsx:52`
- Outside-source rejection test: `crewagent-runtime/src/components/files/fileViewers.test.tsx:108`
- Stale iframe regression test: `crewagent-runtime/src/components/files/fileViewers.test.tsx:135`

---

## Acceptance Criteria Verification

| AC | Status | Evidence |
|----|--------|----------|
| AC-1 Project HTML Preview Can Request Saves | ✅ | `HtmlPreview` checks active iframe source before forwarding: `crewagent-runtime/src/components/files/fileViewers.tsx:166`; covered by `fileViewers.test.tsx:52` |
| AC-2 Output Path Is Scoped To HTML Directory | ✅ | Lexical and realpath containment checks are present, including symlink escape coverage: `crewagent-runtime/electron/stores/runtimeStore.ts:5506` |
| AC-3 Flexible Output File Types | ✅ | Extension is unrestricted; `utf-8` and strict `base64` writes are supported: `crewagent-runtime/electron/stores/runtimeStore.ts:5508` |
| AC-4 Unsafe Paths Are Rejected | ✅ | Traversal, `@project`, protocol-like segments, exact/case-variant source overwrite, and symlink escapes are covered |
| AC-5 Save Result Is Returned To HTML | ✅ | Results are returned to the source iframe, including async save completion after iframe ref changes: `crewagent-runtime/src/components/files/fileViewers.tsx:207` |

---

## Residual Risks

- No manual end-to-end run was performed with a real `@project` HTML file inside the Electron UI.
- Repository lint still has a pre-existing Fast Refresh warning in `WorkflowGraphView.tsx`.

---

## Next Actions

- [x] [AI-Review][MEDIUM] Reject source HTML overwrite using realpath/case-insensitive-safe comparison.
- [x] [AI-Review][MEDIUM] Enforce real directory containment across symlinked output parents.
- [x] [AI-Review][LOW] Return async save results to the original source iframe after preview switches.
