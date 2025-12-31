# Code Review: Story 5-8 – Persist Conversations & Messages

**Date:** 2025-12-30  
**Reviewer:** AI Code Reviewer (BMAD Method)  
**Story File:** `5-8-persist-conversations.md`

---

## Summary

| Metric | Value |
|--------|-------|
| Git vs Story Discrepancies | 2 |
| HIGH Issues | 3 |
| MEDIUM Issues | 4 |
| LOW Issues | 2 |
| Unit Tests | ✅ 4/4 passed |
| Lint | ❌ 2 errors |

---

## 🔴 CRITICAL ISSUES (HIGH)

### H1. Type Mismatch: `ConversationRunMetadata.phase` vs `Run.phase`
- **File:** `src/stores/appStore.ts:172`
- **Description:** `ConversationRunMetadata` (backend) has `phase: string`, but frontend `Run` expects `phase: RunPhase`.
- **Evidence:**
  ```typescript
  // runtimeStore.ts:296
  phase: string  // backend

  // appStore.ts:65
  phase: RunPhase  // frontend - 'not-started' | 'running' | ...
  ```
- **Fix:** Align types or cast explicitly when loading.

---

### H2. Lint Error: Unused Destructured Variables
- **File:** `src/stores/appStore.ts:155`
- **Description:** ESLint `no-unused-vars` error for `messages` and `messagesLoaded`.
- **Evidence:**
  ```typescript
  const { messages, messagesLoaded, ...metadata } = conversation
  ```
- **Fix:** Prefix with underscore: `messages: _messages, messagesLoaded: _messagesLoaded`

---

### H3. Missing `loadConversationMessages` Channel Documentation
- **File:** `5-8-persist-conversations.md:94-99`
- **Description:** Story lists 4 IPC channels but implementation added 5. `conversations:loadMessages` is missing from docs.
- **Fix:** Update Story documentation.

---

## 🟡 MEDIUM ISSUES

### M1. Story File List Missing
- **File:** `5-8-persist-conversations.md`
- **Description:** No "Dev Agent Record" or "File List" section.
- **Fix:** Add:
  - `electron/stores/runtimeStore.ts`
  - `electron/main.ts`
  - `electron/preload.ts`
  - `src/stores/appStore.ts`
  - `electron/stores/runtimeStore.test.ts`

---

### M2. No Error Toast on Persistence Failure
- **File:** `src/stores/appStore.ts:556-564`
- **Description:** IPC failures only logged to console. User gets no feedback.
- **Fix:** Add error handling with toast notification.

---

### M3. Duplicate IPC Calls in `appendConversationMessage`
- **File:** `src/stores/appStore.ts:553-565`
- **Description:** Both `appendConversationMessage` and `saveConversation` called. Redundant.
- **Fix:** Remove redundant `saveConversation` call.

---

### M4. Missing Atomic Write Operations
- **File:** `electron/stores/runtimeStore.ts:1033, 1078`
- **Description:** File writes not atomic. Crash during write = corrupted JSON.
- **Fix:** Use atomic write (temp file → rename).

---

## 🟢 LOW ISSUES

### L1. Missing JSDoc Comments
- **File:** `electron/stores/runtimeStore.ts:1017-1102`
- **Description:** New methods lack JSDoc documentation.

### L2. Console Logging Instead of Structured Logging
- **File:** `electron/stores/runtimeStore.ts`
- **Description:** Uses `console.error` directly.

---

## Acceptance Criteria Verification

| AC | Status | Evidence |
|----|--------|----------|
| AC1: Load on Project Open | ✅ | `loadConversationsForProject()` in appStore |
| AC2: Persist on Create | ✅ | `createConversation()` calls `saveConversation` |
| AC3: Persist on Message Append | ✅ | `appendConversationMessage()` calls IPC |
| AC4: Persist on Selection Change | ✅ | `setConversationType()` calls `saveConversation` |
| AC5: File Location | ✅ | `projects/<id>/conversations/` |

---

## Next Actions

- [ ] [AI-Review][HIGH] Fix type mismatch H1 `src/stores/appStore.ts:172`
- [ ] [AI-Review][HIGH] Fix lint error H2 `src/stores/appStore.ts:155`
- [ ] [AI-Review][HIGH] Update story docs H3 `5-8-persist-conversations.md`
- [ ] [AI-Review][MEDIUM] Add File List to story M1
- [ ] [AI-Review][MEDIUM] Add error toast M2 `src/stores/appStore.ts`
- [ ] [AI-Review][MEDIUM] Remove duplicate IPC M3 `src/stores/appStore.ts`
- [ ] [AI-Review][MEDIUM] Implement atomic writes M4 `runtimeStore.ts`
