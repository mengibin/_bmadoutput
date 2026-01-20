# Design: Persist Tool-Call Protocol Messages

**Story:** `7-2-persist-tool-call-protocol-messages.md`  
**Validation:** `validation-report-story-7-2.md`

## Goals

- 在 `chatToolLoop` 每次产生 `assistant` 或 `tool` 消息时触发持久化
- 支持大输出文件存储（>50KB 写入独立文件）
- 保持向后兼容

---

## Callback Design

### 新增回调参数

```typescript
// chatToolLoop.ts
export async function chatToolLoop(params: {
    // ... existing params ...
    
    // NEW: 消息持久化回调
    onAssistantMessage?: (message: {
        role: 'assistant'
        content: string | null
        tool_calls?: OpenAIToolCall[]
    }) => void
    
    onToolMessage?: (message: {
        role: 'tool'
        tool_call_id: string
        content: string
        toolName: string
        duration: number
    }) => void
})
```

### 调用点

```typescript
// Line ~403: 推送 assistant 后
params.messages.push(assistant)
params.onAssistantMessage?.(assistant)  // NEW

// Line ~529: 推送 tool 结果后
params.messages.push({
    role: 'tool',
    tool_call_id: call.id,
    content: JSON.stringify(result),
})
params.onToolMessage?.({                 // NEW
    role: 'tool',
    tool_call_id: call.id,
    content: JSON.stringify(result),
    toolName: call.function.name,
    duration: Date.now() - toolStartedAt,
})
```

---

## Large Output Handling

### 阈值配置

```typescript
// conversationTypes.ts
export const TOOL_OUTPUT_THRESHOLD = 50 * 1024  // 50KB
export const TOOL_OUTPUT_PREVIEW_LENGTH = 500

export interface ConversationMessage {
    // ... existing fields ...
    fullOutputPath?: string  // NEW: 大输出文件路径
}
```

### 存储路径

```
RuntimeStore/projects/<projectId>/
├── conversations/
│   └── <conversationId>/
│       └── messages.json
└── state/
    └── logs/
        └── tool-results/
            └── <conversationId>/
                └── <messageId>.json
```

### 处理函数

```typescript
// runtimeStore.ts
function prepareToolMessageForStorage(
    projectRoot: string,
    conversationId: string,
    message: ConversationMessage
): ConversationMessage {
    if (message.role !== 'tool' || !message.content) return message
    
    const contentBytes = Buffer.byteLength(message.content, 'utf-8')
    if (contentBytes <= TOOL_OUTPUT_THRESHOLD) return message
    
    // 写入完整输出
    const outputPath = path.join(
        this.getProjectRuntimeRoot(projectRoot),
        'state/logs/tool-results',
        conversationId,
        `${message.id}.json`
    )
    fs.mkdirSync(path.dirname(outputPath), { recursive: true })
    fs.writeFileSync(outputPath, message.content, 'utf-8')
    
    // 返回带预览的消息
    return {
        ...message,
        content: message.content.slice(0, TOOL_OUTPUT_PREVIEW_LENGTH) + 
                 `\n\n[Full output: ${outputPath}]`,
        fullOutputPath: outputPath
    }
}
```

---

## Integration Points

### 1. ExecutionEngine (Run Mode)

```typescript
// executionEngine.ts
const loopResult = await chatToolLoop({
    // ... existing params ...
    onAssistantMessage: (msg) => {
        this.persistMessage(conversationId, {
            id: randomUUID(),
            role: msg.role,
            content: msg.content,
            tool_calls: msg.tool_calls,
            createdAt: new Date().toISOString(),
            mode: 'run',
            runId: this.currentRunId,
        })
    },
    onToolMessage: (msg) => {
        this.persistMessage(conversationId, {
            id: randomUUID(),
            role: msg.role,
            content: msg.content,
            tool_call_id: msg.tool_call_id,
            toolName: msg.toolName,
            duration: msg.duration,
            createdAt: new Date().toISOString(),
            mode: 'run',
            runId: this.currentRunId,
        })
    }
})
```

### 2. Chat Mode (WorksPage)

Chat 模式通过 IPC 事件接收消息：

```typescript
// main.ts - 新增 IPC handler
ipcMain.on('chat:assistantMessage', (event, payload) => {
    const { conversationId, message } = payload
    runtimeStore.appendConversationMessage(projectRoot, conversationId, message)
})

ipcMain.on('chat:toolMessage', (event, payload) => {
    const { conversationId, message } = payload
    const prepared = runtimeStore.prepareToolMessageForStorage(projectRoot, conversationId, message)
    runtimeStore.appendConversationMessage(projectRoot, conversationId, prepared)
})
```

---

## Files to Modify

| File | Changes |
|------|---------|
| `shared/conversationTypes.ts` | 添加 `fullOutputPath` 字段和常量 |
| `electron/services/chatToolLoop.ts` | 添加 `onAssistantMessage` / `onToolMessage` 回调 |
| `electron/services/executionEngine.ts` | 实现回调处理 |
| `electron/stores/runtimeStore.ts` | 添加 `prepareToolMessageForStorage` |
| `electron/main.ts` | 添加 Chat 模式 IPC handlers |

---

## Unit Test Design

```typescript
describe('chatToolLoop message callbacks', () => {
    it('should call onAssistantMessage when LLM responds', async () => {
        const onAssistantMessage = vi.fn()
        await chatToolLoop({
            // ... mock params ...
            onAssistantMessage
        })
        expect(onAssistantMessage).toHaveBeenCalledWith(
            expect.objectContaining({ role: 'assistant' })
        )
    })
    
    it('should call onToolMessage after tool execution', async () => {
        const onToolMessage = vi.fn()
        // ... mock LLM to return tool_calls ...
        await chatToolLoop({
            // ... mock params ...
            onToolMessage
        })
        expect(onToolMessage).toHaveBeenCalledWith(
            expect.objectContaining({ role: 'tool', tool_call_id: expect.any(String) })
        )
    })
})

describe('prepareToolMessageForStorage', () => {
    it('should write large output to file and return preview', () => {
        const largeContent = 'x'.repeat(60 * 1024)  // 60KB
        const result = prepareToolMessageForStorage(projectRoot, convId, {
            id: 'msg1', role: 'tool', content: largeContent, createdAt: '...'
        })
        expect(result.content?.length).toBeLessThan(1000)
        expect(result.fullOutputPath).toBeDefined()
    })
})
```

---

## Backward Compatibility

| 场景 | 处理 |
|------|------|
| 旧消息无 `fullOutputPath` | 正常，字段可选 |
| 新回调不存在 | 可选回调，不影响现有逻辑 |
