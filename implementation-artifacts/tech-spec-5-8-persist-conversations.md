# Tech Spec: Story 5-8 – Persist Conversations & Messages

## Summary
Persist Conversations per project in RuntimeStore (metadata + message history), exposed via IPC and integrated with `appStore` using **lazy message loading**.

---

## 1. Backend (RuntimeStore)

### 1.1 Directory Structure
```
$RUNTIME_DATA/projects/<projectId>/conversations/
├── index.json                   # ConversationMetadata[]
└── <conversationId>/
    └── messages.json            # ConversationMessage[]
```

### 1.2 Types
```typescript
type RunPhase = 'idle' | 'running' | 'waiting-user' | 'completed' | 'failed'

interface ConversationRunMetadata {
  runId: string
  projectId: string
  packageId: string
  workflowRef: string
  activeAgentId: string
  phase: RunPhase
  createdAt: string
}

interface ConversationMetadata {
  id: string
  title: string
  entryType: 'workflow' | 'agent' | 'chat'
  activeType: 'workflow' | 'agent' | 'chat'
  createdAt: string
  lastActiveAt: string
  runs: ConversationRunMetadata[]
  selectedWorkflowId?: string
  selectedAgentId?: string
}

interface ConversationMessage {
  id: string
  role: 'user' | 'assistant' | 'system'
  content: string
  createdAt: string
}
```

### 1.3 Methods

#### `loadConversations(projectRoot: string): ConversationMetadata[]`
1. Ensure `projects/<projectId>/conversations/` exists
2. Read `index.json` (if missing → `[]`)
3. Return `ConversationMetadata[]` (messages are not loaded here)

#### `saveConversation(projectRoot: string, conversation: ConversationMetadata): { success: boolean }`
1. Read existing `index.json`
2. Upsert `conversation` by `id` (prepend if new)
3. Write back to `index.json`
4. Ensure `<conversationId>/` exists

#### `loadConversationMessages(projectRoot: string, conversationId: string): ConversationMessage[]`
1. Read `<conversationId>/messages.json` (if missing → `[]`)
2. Return `ConversationMessage[]`

#### `appendConversationMessage(projectRoot: string, conversationId: string, message: ConversationMessage): { success: boolean }`
1. Read `<conversationId>/messages.json`
2. Append `message`
3. Atomically write messages.json (tmp → rename)
4. Update `lastActiveAt` in `index.json` (atomically)

#### `deleteConversation(projectRoot: string, conversationId: string): { success: boolean }`
1. Delete `<conversationId>/` folder
2. Remove from `index.json`

---

## 2. IPC Layer

### preload.ts
```typescript
loadConversations: (payload: { projectRoot: string }) => ipcRenderer.invoke('conversations:load', payload),
saveConversation: (payload: { projectRoot: string; conversation: ConversationMetadata }) =>
  ipcRenderer.invoke('conversations:save', payload),
loadConversationMessages: (payload: { projectRoot: string; conversationId: string }) =>
  ipcRenderer.invoke('conversations:loadMessages', payload),
appendConversationMessage: (payload: { projectRoot: string; conversationId: string; message: ConversationMessage }) =>
  ipcRenderer.invoke('conversations:appendMessage', payload),
deleteConversation: (payload: { projectRoot: string; conversationId: string }) =>
  ipcRenderer.invoke('conversations:delete', payload),
```

### main.ts
```typescript
ipcMain.handle('conversations:load', (_, payload) => runtimeStore.loadConversations(payload.projectRoot))
ipcMain.handle('conversations:save', (_, payload) => runtimeStore.saveConversation(payload.projectRoot, payload.conversation))
ipcMain.handle('conversations:loadMessages', (_, payload) => runtimeStore.loadConversationMessages(payload.projectRoot, payload.conversationId))
ipcMain.handle('conversations:appendMessage', (_, payload) => runtimeStore.appendConversationMessage(payload.projectRoot, payload.conversationId, payload.message))
ipcMain.handle('conversations:delete', (_, payload) => runtimeStore.deleteConversation(payload.projectRoot, payload.conversationId))
```

---

## 3. Frontend (appStore / Works)

### 3.1 Project Open (metadata only)
- On project open: call `conversations:load` and populate `conversationsByProject[projectId]`
- Each conversation is initialized with:
  - `messages: []`
  - `messagesLoaded: false`
  - `runs[].phase` normalized to `RunPhase`

### 3.2 Lazy message loading
- When a conversation is selected: call `conversations:loadMessages` once and set `messagesLoaded: true`

### 3.3 Persist hooks
- On create/update selection/type/runs: call `conversations:save`
- On message send: call `conversations:appendMessage`
- On delete: call `conversations:delete`

### 3.4 UI note
- Works list provides a delete action with confirmation; deleting clears selection if it was active.

---

## 3. Frontend (appStore)

### 3.1 Load Conversations
In `openProjectByPayload`, after setting active project:
```typescript
const conversations = await window.ipcRenderer.loadConversations(project.projectId)
set((state) => ({
  conversationsByProject: {
    ...state.conversationsByProject,
    [project.projectId]: conversations,
  },
}))
```

### 3.2 Modify `createConversation`
After creating in-memory, persist:
```typescript
const meta: ConversationMeta = { id, title, entryType, activeType, createdAt, lastActiveAt, selectedWorkflowId, selectedAgentId }
void window.ipcRenderer.saveConversation(activeProject.projectId, meta)
```

### 3.3 Modify `appendConversationMessage`
After updating in-memory, persist:
```typescript
void window.ipcRenderer.appendConversationMessage(activeProject.projectId, conversationId, message)
```

### 3.4 Modify `setConversationType` / `updateConversationSelection`
After updating in-memory, persist updated meta:
```typescript
void window.ipcRenderer.saveConversation(activeProject.projectId, updatedMeta)
```

---

## 4. Testing

### 4.1 Unit Tests (runtimeStore.test.ts)
| Test | Assertion |
|------|-----------|
| `loadConversations` returns empty for new project | `[]` |
| `saveConversationMeta` creates index.json + messages.json | files exist |
| `appendMessage` adds to messages.json | length increases |
| `deleteConversation` removes folder | folder gone |
| `loadConversations` handles missing index.json | returns `[]` |
| `appendMessage` updates `lastActiveAt` | timestamp updated |

### 4.2 End-to-End Test Cases

#### E2E-1: Conversation Persistence Across App Restart
**Precondition**: Fresh project with no conversations.

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Open project | Works page shows "No conversations" |
| 2 | Create conversation: "Test Chat" | Conversation appears in list |
| 3 | Send user message: "Hello" | Message appears in chat |
| 4 | Send user message: "World" | Second message appears |
| 5 | Close app completely | App closes |
| 6 | Reopen app, open same project | Project opens |
| 7 | Navigate to Works page | "Test Chat" visible |
| 8 | Open "Test Chat" | Both messages visible in order |

**Disk Verification**:
- `index.json` contains "Test Chat" entry
- `<conversationId>/messages.json` has 2 messages

---

#### E2E-2: Multiple Conversations in One Project

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Create second conversation: "Feature Design" | 2 conversations in list |
| 2 | Add message to "Feature Design" | Message saved |
| 3 | Switch to "Test Chat" | Original messages visible |
| 4 | Restart app | Both conversations persist |

---

#### E2E-3: ActiveType Change Persistence

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Create conversation with entryType: 'workflow' | Shows as Workflow |
| 2 | Switch activeType to 'agent' | UI updates to Agent mode |
| 3 | Restart app, reopen project | Conversation loads |
| 4 | Verify | activeType='agent', entryType='workflow' |

---

#### E2E-4: Delete Conversation

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Delete "Feature Design" | Removed from UI |
| 2 | Verify disk | Folder deleted |
| 3 | Restart app | Only 1 conversation remains |

---

#### E2E-5: Large Message History

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Create conversation | Success |
| 2 | Send 50 messages | All saved |
| 3 | Restart app | All 50 load correctly |
| 4 | Scroll | No performance lag |

---

#### E2E-6: Concurrent Project Switching

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Open Project A, verify convos | Project A convos visible |
| 2 | Switch to Project B | Project B convos load |
| 3 | Add message in Project B | Persisted |
| 4 | Switch back to Project A | Project A intact |
| 5 | No cross-contamination | Each project has own data |

---

### 4.3 Error Handling Tests

| Scenario | Expected Behavior |
|----------|-------------------|
| Corrupted index.json | Return empty, log error |
| Missing messages.json | Return convo with empty messages |
| Disk full on append | Show error toast, memory state intact |
| Read-only file system | Show error, don't crash |

---

## 5. Edge Cases
- **Empty messages.json**: Return `[]`.
- **Missing index.json**: Return `[]`.
- **Concurrent writes**: Use atomic file writes (temp → rename).
- **Invalid conversation ID**: Return error, don't create folder.
