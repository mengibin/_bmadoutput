# Design: Extend ConversationMessage Data Model

**Story:** `7-1-extend-conversation-message-data-model.md`  
**Architecture:** `_bmad-output/architecture/unified-conversation-context.md`

## Goals

- 统一 `ConversationMessage` 类型定义到 `shared/` 目录
- 扩展支持 OpenAI tool-call 协议 (`role=tool`, `tool_calls`, `tool_call_id`)
- 添加上下文元数据 (`mode`, `runId`, `includeInContext`)
- 保持向后兼容（所有新字段可选）

---

## Type Design

### OpenAIToolCall

```typescript
export interface OpenAIToolCall {
  id: string
  type: 'function'
  function: {
    name: string
    arguments: string  // JSON string
  }
}
```

### ConversationMessage (Extended)

```typescript
export type ConversationMessageRole = 'user' | 'assistant' | 'system' | 'tool'

export interface ConversationMessage {
  id: string
  role: ConversationMessageRole
  createdAt: string

  // OpenAI-compatible (协议字段)
  content?: string | null           // 可为 null (assistant 只有 tool_calls 时)
  tool_calls?: OpenAIToolCall[]     // assistant 发起工具调用
  tool_call_id?: string             // tool 结果关联的调用 ID

  // UI extras (现有字段)
  partType?: MessagePartType
  toolName?: string
  duration?: number
  isCollapsed?: boolean
  widget?: WidgetPayload

  // Context metadata (新增)
  mode?: 'chat' | 'agent' | 'run'
  runId?: string
  includeInContext?: boolean        // 默认 true
}
```

---

## File Structure

```
shared/
├── agentToolPolicy.ts      (existing)
├── logTypes.ts             (existing)
└── conversationTypes.ts    (NEW) ← 所有对话相关类型
```

---

## Migration Steps

### Step 1: Create `shared/conversationTypes.ts`

包含:
- `MessagePartType`
- `WidgetType`, `WidgetOption`, `WidgetAction`, `WidgetFeedbackItem`, `WidgetPayload`
- `OpenAIToolCall`
- `ConversationMessageRole`
- `ConversationMessage`

### Step 2: Update `appStore.ts`

```diff
- export type ConversationMessageRole = 'user' | 'assistant' | 'system'
- export interface ConversationMessage { ... }
+ import type { ConversationMessage, ConversationMessageRole, ... } from '../../shared/conversationTypes'
+ export type { ConversationMessage, ConversationMessageRole, ... }
```

### Step 3: Update `runtimeStore.ts`

```diff
- export interface ConversationMessage { ... }
+ import type { ConversationMessage, ... } from '../../shared/conversationTypes'
```

### Step 4: Verify Build

```bash
cd crewagent-runtime && npm run build
```

---

## Backward Compatibility

| 场景 | 处理方式 |
|------|---------|
| 旧消息无 `tool_calls` | 字段为 `undefined`，不影响渲染 |
| 旧消息 `content` 为 string | 兼容，新类型允许 `string \| null` |
| 旧消息无 `mode`/`runId` | 字段为 `undefined` |

---

## UI Considerations

`role='tool'` 消息暂时复用 `assistant` 样式，默认 `isCollapsed=true`。具体 UI 优化在后续 Story 中处理。

---

## Unit Testing Design

### Test File

`shared/conversationTypes.test.ts`

### Test Cases

#### 1. Type Guard Tests

```typescript
import { isValidConversationMessage, isToolCallMessage } from './conversationTypes'

describe('ConversationMessage type guards', () => {
  it('should validate a minimal user message', () => {
    const msg = {
      id: '1',
      role: 'user' as const,
      content: 'Hello',
      createdAt: '2026-01-18T00:00:00Z'
    }
    expect(isValidConversationMessage(msg)).toBe(true)
  })

  it('should validate assistant message with tool_calls', () => {
    const msg = {
      id: '2',
      role: 'assistant' as const,
      content: null,
      tool_calls: [{
        id: 'call_1',
        type: 'function' as const,
        function: { name: 'read_file', arguments: '{"path": "test.txt"}' }
      }],
      createdAt: '2026-01-18T00:00:00Z'
    }
    expect(isValidConversationMessage(msg)).toBe(true)
    expect(isToolCallMessage(msg)).toBe(true)
  })

  it('should validate tool result message', () => {
    const msg = {
      id: '3',
      role: 'tool' as const,
      content: 'file contents here',
      tool_call_id: 'call_1',
      createdAt: '2026-01-18T00:00:00Z'
    }
    expect(isValidConversationMessage(msg)).toBe(true)
  })
})
```

#### 2. Backward Compatibility Tests

```typescript
describe('Backward compatibility', () => {
  it('should accept legacy message without new fields', () => {
    // 模拟旧版 messages.json 中的消息
    const legacyMsg = {
      id: '100',
      role: 'assistant',
      content: 'Hello world',
      createdAt: '2025-01-01T00:00:00Z',
      partType: 'content'
    }
    expect(isValidConversationMessage(legacyMsg)).toBe(true)
  })

  it('should handle missing optional fields gracefully', () => {
    const msg = {
      id: '101',
      role: 'user',
      content: 'Test',
      createdAt: '2026-01-18T00:00:00Z'
    }
    // 新字段应为 undefined
    expect(msg.tool_calls).toBeUndefined()
    expect(msg.tool_call_id).toBeUndefined()
    expect(msg.mode).toBeUndefined()
    expect(msg.includeInContext).toBeUndefined()
  })
})
```

#### 3. Type Conversion Tests

```typescript
import { toOpenAIMessage } from './conversationTypes'

describe('toOpenAIMessage conversion', () => {
  it('should convert user message to OpenAI format', () => {
    const msg = { id: '1', role: 'user', content: 'Hi', createdAt: '...' }
    const openAIMsg = toOpenAIMessage(msg)
    expect(openAIMsg).toEqual({ role: 'user', content: 'Hi' })
  })

  it('should convert assistant with tool_calls to OpenAI format', () => {
    const msg = {
      id: '2',
      role: 'assistant',
      content: null,
      tool_calls: [{ id: 'call_1', type: 'function', function: { name: 'foo', arguments: '{}' } }],
      createdAt: '...'
    }
    const openAIMsg = toOpenAIMessage(msg)
    expect(openAIMsg.role).toBe('assistant')
    expect(openAIMsg.tool_calls).toHaveLength(1)
  })

  it('should convert tool result to OpenAI format', () => {
    const msg = {
      id: '3',
      role: 'tool',
      content: 'result',
      tool_call_id: 'call_1',
      createdAt: '...'
    }
    const openAIMsg = toOpenAIMessage(msg)
    expect(openAIMsg).toEqual({ role: 'tool', content: 'result', tool_call_id: 'call_1' })
  })
})
```

### Helper Functions to Implement

```typescript
// shared/conversationTypes.ts

/** Type guard: validate message structure */
export function isValidConversationMessage(msg: unknown): msg is ConversationMessage

/** Type guard: check if message has tool_calls */
export function isToolCallMessage(msg: ConversationMessage): boolean

/** Convert to OpenAI API format (strip UI fields) */
export function toOpenAIMessage(msg: ConversationMessage): OpenAIChatMessage
```

### Test Command

```bash
cd crewagent-runtime && npm test -- shared/conversationTypes.test.ts
```

