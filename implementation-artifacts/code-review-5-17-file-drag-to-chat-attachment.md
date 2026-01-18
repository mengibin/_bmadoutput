# Code Review: Story 5.17 – File Drag-to-Chat Attachment

**Date:** 2026-01-18  
**Reviewer:** AI Code Reviewer (BMAD Method)  
**Story File:** `5-17-file-drag-to-chat-attachment.md`

---

## Summary

| Metric | Value |
|--------|-------|
| Git vs Story Discrepancies | 0 |
| HIGH Issues | 0 |
| MEDIUM Issues | 0 |
| LOW Issues | 0 |
| Build | ✅ `npm -C crewagent-runtime run build:ci` passed |
| Unit Tests | ✅ `npm -C crewagent-runtime test` passed |
| Lint | ✅ `npm -C crewagent-runtime run lint` passed *(TypeScript version warning only)* |

---

## 🔴 HIGH ISSUES

None.

---

## 🟡 MEDIUM ISSUES

None.

---

## 🟢 LOW ISSUES

None.

---

## Resolved During Review

- ✅ Drag payload now includes a `relativePath` to avoid relying on absolute `projectRoot` slicing (`crewagent-runtime/src/pages/FilesPage/FilesPage.tsx:98`).
- ✅ Drop handler supports both alias paths (`@project/...`) and absolute paths (`crewagent-runtime/src/pages/RunsPage/components/ChatInput.tsx:109`).
- ✅ Attachments are cleared when switching conversations to avoid “leaking” attachments across conversations (`crewagent-runtime/src/pages/WorksPage/WorksPage.tsx:351`).
- ✅ Added a basic rendering test for attachments (`crewagent-runtime/src/pages/RunsPage/components/ChatInput.test.tsx:19`).

---

## Acceptance Criteria Verification

| AC | Status | Evidence |
|----|--------|----------|
| AC1: Files panel draggable | ✅ | `draggable={!isDirectory}` + `dataTransfer.setData(...)`: `crewagent-runtime/src/pages/FilesPage/FilesPage.tsx:90` |
| AC2: Chat input drop zone + hover styling | ✅ | Drop handlers + `drag-over` class: `crewagent-runtime/src/pages/RunsPage/components/ChatInput.tsx:62`; CSS: `crewagent-runtime/src/pages/RunsPage/RunsPage.css:405` |
| AC3: Attached files display + remove | ✅ | Tags + remove button: `crewagent-runtime/src/pages/RunsPage/components/ChatInput.tsx:163` |
| AC4: File metadata included in sent message + cleared after send | ✅ | Message prefix + `setAttachedFiles([])`: `crewagent-runtime/src/pages/WorksPage/WorksPage.tsx:997` |
| AC5: Multiple attachments + dedupe | ✅ | `handleFileDrop` path dedupe: `crewagent-runtime/src/pages/WorksPage/WorksPage.tsx:369` |
| AC6: Remove attached file | ✅ | `handleRemoveFile`: `crewagent-runtime/src/pages/WorksPage/WorksPage.tsx:378` |

---

## Notes

- The “Referenced Files” block includes both file name + relative path (this matches AC4); the Design section’s example shows paths-only.
- Build emits an existing Vite warning about mixed static + dynamic imports for `mcpInstallService` (unrelated to this story).
