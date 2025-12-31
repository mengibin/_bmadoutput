# Story 5-8: Persist Conversations & Messages

## Overview
**Epic**: 5 – Runtime Frontend  
**Priority**: Phase 3 (after Files Page)  
**Status**: `ready-for-dev`

## Goal
Persist Conversation metadata and message history per project in RuntimeStore, loading them when a project is reopened and appending new messages during session.

## Business Value
- **Durability**: Users can close the app and return to their previous conversations.
- **Context Continuity**: The "Works" page shows conversation history across sessions.

## Acceptance Criteria
1. **Load on Project Open**: When a project is opened, its stored conversations are loaded into `appStore.conversationsByProject`.
2. **Persist on Create**: When a new Conversation is created, it's immediately written to disk.
3. **Persist on Message Append**: When a message is appended, it's immediately written to disk.
4. **Persist on Selection Change**: When workflow/agent selection changes, changes are persisted.
5. **File Location**: Conversations stored at `RuntimeStore/projects/<projectId>/conversations/`.
6. **User Feedback on Failure**: Persistence failures surface a user-visible error (toast/alert) instead of silent console-only logs.
7. **Atomic Writes**: Conversation index and message files are saved atomically (tmp → rename) to avoid corruption on crash.
8. **Run Phase Alignment**: `runs[].phase` is stored as `RunPhase` or normalized to a valid `RunPhase` value on load.

## Out of Scope
- Run history persistence (handled by Story 4-8).
- Cross-device sync.

## Dependencies
- **5-0 UI Shell**: Provides Works page structure (done).
- **appStore Conversation types**: Already defined (done).

## Validation Notes ✅
**Code Review Completed**: 2025-12-30

1. **RuntimeStore Infrastructure**: 
   - `projectsRuntimePath` exists at `runtime-store/projects/`
   - `getProjectRuntimeRoot(projectRoot)` returns `projects/<projectId>/`
   - `ensureProjectRuntimeDirs()` already creates `runs/` and `state/` subfolders
   - **Story 5-8 will add `conversations/` using the same pattern**

2. **IPC Layer**: 
   - No conflicts with existing channels (`conversations:*` namespace is available)
   - Pattern matches existing channels (e.g., `packages:list`, `files:read`)

3. **appStore Integration**:
   - `conversationsByProject` already exists in state
   - `createConversation`, `appendConversationMessage` methods ready for persistence hooks

4. **Feasibility**: ✅ Design is fully compatible with existing codebase

---

## Technical Context

### Current State (appStore.ts)
```typescript
interface Conversation {
  id: string
  title: string
  entryType: ConversationType  // 'workflow' | 'agent' | 'chat'
  activeType: ConversationType
  createdAt: string
  lastActiveAt: string
  runs: Run[]
  messages: ConversationMessage[]
  selectedWorkflowId?: string
  selectedAgentId?: string
}

interface ConversationMessage {
  id: string
  role: ConversationMessageRole  // 'user' | 'assistant' | 'system'
  content: string
  createdAt: string
}
```

### Storage Design
```
$RUNTIME_DATA/projects/<projectId>/
├── conversations/
│   ├── index.json          # List of conversation metadata (without messages)
│   └── <conversationId>/
│       └── messages.json   # Array of ConversationMessage
```

### RuntimeStore Methods to Add
| Method | Purpose |
|--------|---------|
| `loadConversations(projectRoot)` | Load all conversations for a project |
| `saveConversation(projectRoot, conversation)` | Create or update conversation metadata |
| `loadConversationMessages(projectRoot, conversationId)` | Load message history for a conversation |
| `appendConversationMessage(projectRoot, conversationId, message)` | Append message to messages.json |
| `deleteConversation(projectRoot, conversationId)` | Remove conversation folder |

### IPC Channels
| Channel | Handler |
|---------|---------|
| `conversations:load` | Returns `Conversation[]` |
| `conversations:save` | Persists conversation metadata |
| `conversations:loadMessages` | Loads message history for a conversation |
| `conversations:appendMessage` | Persists new message |
| `conversations:delete` | Removes conversation |

---

## Testing Strategy

### Automated Tests
- Unit test `RuntimeStore.loadConversations` with mock file system
- Unit test `RuntimeStore.appendMessage` verifies file update

### Manual Verification
1. Create a new Conversation in Works page
2. Add a message (type in chat)
3. Close and reopen app
4. Verify conversations and messages are restored
5. Delete a conversation, verify it's removed from disk

---

## Design Decisions

### Split Storage (index.json + messages.json)
- **Rationale**: Loading all messages for all conversations on project open would be slow. Instead, load only metadata (index.json), then lazy-load messages when a specific conversation is opened.

### Runs Not Persisted Here
- Runs are persisted in Story 4-8 (`RuntimeStore/projects/<projectId>/runs/`). The Conversation only holds `runId` references, not full Run objects.

---

## Dev Agent Record / File List
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/preload.ts`
- `crewagent-runtime/src/stores/appStore.ts`
- `crewagent-runtime/electron/stores/runtimeStore.test.ts`

---

## Related Artifacts
- [Tech Spec](file:///Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/tech-spec-5-8-persist-conversations.md) (to be created)
