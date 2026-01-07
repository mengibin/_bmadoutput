# Tech Spec: Story 5-11 – Enhanced Chat Interface Foundation

## Summary
Refactor `RunWorkspace.tsx` to support **streaming UI**, **Iceberg Model** for hiding internal details, and a **dedicated `MessageMarkdown`** renderer for chat bubbles.

---

## 1. Component Hierarchy

```
ChatPanel (new)
├── MessageList (new)
│   └── MessageItem (new)
│       ├── UserBubble
│       ├── AssistantBubble → uses MessageMarkdown
│       ├── ThinkingIndicator (collapsed)
│       └── ToolStatusIndicator (collapsed)
├── ChatInput (existing, extract)
└── (Sidebar: RunDetails - existing)
```

---

## 2. Data Model

### 2.1 Extended Message Type
Extend `ConversationMessage` in `appStore.ts` to support internal events:

```typescript
// appStore.ts (extended)
export type MessagePartType = 'content' | 'thinking' | 'tool_start' | 'tool_result'

export interface ConversationMessage {
  id: string
  role: ConversationMessageRole // 'user' | 'assistant' | 'system'
  content: string
  createdAt: string
  // New fields for Iceberg Model
  partType?: MessagePartType   // default: 'content'
  toolName?: string            // for tool_start/tool_result
  duration?: number            // ms, for tool_result
  isCollapsed?: boolean        // UI state (not persisted)
}
```

### 2.2 Streaming State
Local state for streaming:
```typescript
interface StreamingState {
  isStreaming: boolean
  partialContent: string
  currentMessageId: string | null
}
```

---

## 3. IPC Events for Streaming

### 3.1 Main → Renderer Push Events
Add `ipcRenderer.on` listeners in Renderer for streaming:

| Event Name | Payload | Description |
|---|---|---|
| `llm:stream-start` | `{ messageId }` | Begin new assistant message |
| `llm:stream-chunk` | `{ messageId, delta, partType? }` | Append text chunk |
| `llm:stream-tool-start` | `{ messageId, toolName }` | Tool execution begins |
| `llm:stream-tool-end` | `{ messageId, toolName, result?, duration }` | Tool execution ends |
| `llm:stream-end` | `{ messageId }` | Complete message |

### 3.2 preload.ts Addition
```typescript
// preload.ts
onLlmStreamChunk: (callback: (payload: StreamChunkPayload) => void) => {
  ipcRenderer.on('llm:stream-chunk', (_, payload) => callback(payload))
  return () => ipcRenderer.removeAllListeners('llm:stream-chunk')
},
// ... similar for other events
```

---

## 4. Component Specifications

### 4.1 `MessageItem.tsx`
**Props:**
```typescript
interface MessageItemProps {
  message: ConversationMessage
  isStreaming?: boolean
  onExpandThought?: (messageId: string) => void // navigate to Log tab
}
```

**Rendering Logic:**
```
if (message.role === 'user') → UserBubble (primary color)
else if (message.partType === 'thinking') → ThinkingIndicator (collapsed)
else if (message.partType === 'tool_start' || 'tool_result') → ToolStatusIndicator (collapsed)
else → AssistantBubble with MessageMarkdown
```

### 4.2 `ThinkingIndicator.tsx`
**Visual:**
- Icon: `Brain` (lucide-react) + animated pulse
- Text: "Thinking..."
- Click: Calls `onExpandThought` → switches to Logs tab and scrolls to entry

### 4.3 `ToolStatusIndicator.tsx`
**Visual:**
- Icon: `Wrench` (lucide-react) + spinner while running
- Text: `Running ${toolName}...` / `Completed ${toolName}` (with duration)
- Click: Reveals tool args/result inline OR navigates to Logs tab

### 4.4 `MessageMarkdown.tsx`
**Purpose:** Render Markdown specifically for chat messages, isolated from file viewer styling.

**Features:**
- Copy `parseMarkdownBlocks` logic from `fileViewers.tsx` (do NOT import directly)
- Custom styles:
  - `font-size: 14px` (vs 13px in file viewer)
  - Code blocks with `border-radius: 8px`, no line numbers
  - Quote styling adjusted for bubble context
- No Mermaid support (keeps component light)

### 4.5 `ChatInput.tsx`
Extract existing input from `RunWorkspace.tsx`:
- `<textarea>` instead of `<input>` for multi-line support
- `Enter` to send, `Shift+Enter` for newline
- Auto-resize height

---

## 5. Auto-Scrolling Logic

### 5.1 Behavior
- **Auto-scroll ON** (default): When new content arrives, scroll to bottom.
- **Auto-scroll OFF**: If user scrolls up more than 50px from bottom, disable auto-scroll.
- **Re-enable**: When user scrolls back to bottom (within 50px), re-enable.

### 5.2 Implementation
Use `useRef` for scroll container and `useEffect` with debounce:
```typescript
const messagesEndRef = useRef<HTMLDivElement>(null)
const [userScrolledUp, setUserScrolledUp] = useState(false)

useEffect(() => {
  if (!userScrolledUp && messagesEndRef.current) {
    messagesEndRef.current.scrollIntoView({ behavior: 'smooth' })
  }
}, [messages, partialContent])
```

---

## 6. File Changes

| File | Action | Description |
|---|---|---|
| `src/pages/RunsPage/components/ChatPanel.tsx` | NEW | Main chat container |
| `src/pages/RunsPage/components/MessageList.tsx` | NEW | Message rendering loop |
| `src/pages/RunsPage/components/MessageItem.tsx` | NEW | Single message logic |
| `src/pages/RunsPage/components/ChatInput.tsx` | NEW | Input with multi-line |
| `src/pages/RunsPage/components/MessageMarkdown.tsx` | NEW | Chat-specific MD |
| `src/pages/RunsPage/components/ThinkingIndicator.tsx` | NEW | Collapsed thought |
| `src/pages/RunsPage/components/ToolStatusIndicator.tsx` | NEW | Collapsed tool |
| `src/pages/RunsPage/RunWorkspace.tsx` | MODIFY | Refactor to use new components |
| `src/pages/RunsPage/RunsPage.css` | MODIFY | Add new styles |
| `electron/preload.ts` | MODIFY | Add streaming event listeners |

---

## 7. Testing Plan

### 7.1 Unit Tests
| Component | Test Case |
|---|---|
| `MessageItem` | Renders UserBubble for role='user' |
| `MessageItem` | Renders ThinkingIndicator for partType='thinking' |
| `MessageMarkdown` | Renders code block with correct styling |
| `ChatInput` | Enter submits, Shift+Enter adds newline |

### 7.2 Manual Verification
1. **Streaming**: Mock `llm:stream-chunk` events. Verify text appears progressively.
2. **Iceberg - Thinking**: Send a `partType='thinking'` message. Confirm "Thinking..." indicator shown, content hidden.
3. **Iceberg - Tool**: Send `tool_start` then `tool_result`. Confirm indicator animates then shows duration.
4. **Auto-scroll**: Scroll up, send new message. Confirm no jump. Scroll to bottom, send new message. Confirm auto-scroll.
5. **Markdown**: Send message with code block and list. Verify correct rendering.

---

## 8. Edge Cases

- **Empty content**: `partType='thinking'` with empty content should show indicator, not empty bubble.
- **Long code blocks**: Ensure horizontal scroll, no layout break.
- **Rapid streaming**: Debounce renders to avoid performance issues.
- **Persistence**: `partType` and `toolName` added to `messages.json` schema; old messages without these fields default to `partType='content'`.
