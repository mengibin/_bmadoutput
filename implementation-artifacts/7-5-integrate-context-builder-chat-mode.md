# Story 7-5: Integrate Context Builder into Chat Mode

## Overview
**Epic**: 7 – Unified Conversation Context  
**Priority**: Phase 3 (depends on 7.3, 7.4)  
**Status**: `done`

## Goal
将 Chat 模式的 `chat:send` 改为使用统一的 Context Builder，使其带上对话历史，实现多轮对话上下文连续性。

## Business Value
- **上下文连续性**：用户可以进行多轮对话，LLM 能"记住"之前的内容
- **继续支持**：错误后可以说"继续"
- **错误可见性**：错误显示为 assistant 消息，而不仅仅是 toast

## Acceptance Criteria# 
1. **历史加载**：从 `messages.json` 加载对话历史
2. **上下文构建**：调用 `buildLlmMessagesFromConversation({ mode: 'chat', ... })`
3. **消息持久化**：
   - 用户消息存入对话日志
   - LLM 响应存入对话日志（包含 `tool_calls`）
   - 工具结果存入对话日志（`role: 'tool'`）
4. **错误展示**：LLM 错误显示为 assistant 消息（`partType: 'error'`）
5. **压缩集成**：在构建上下文时检查并按需压缩（调用 `CompressionPipeline`）

## Tasks / Subtasks

### Review Follow-ups (AI)
- [x] [AI-Review][HIGH] Implement history load + context-builder usage in `chat:send` so chat mode uses persisted conversation history [crewagent-runtime/electron/main.ts:1899]
- [x] [AI-Review][HIGH] Persist user messages and assistant content messages to conversation log (not only tool_calls) [crewagent-runtime/electron/main.ts:1505]
- [x] [AI-Review][MEDIUM] Ensure non-LLM error paths are also persisted as assistant messages for "continue" support [crewagent-runtime/electron/main.ts:1541]
- [x] [AI-Review][MEDIUM] Add compression integration for chat loop (compressionPipeline param + loop historical compression) [crewagent-runtime/electron/services/chatToolLoop.ts:331]
- [x] [AI-Review][MEDIUM] Add Dev Agent Record + File List documenting actual changes [ _bmad-output/implementation-artifacts/7-5-integrate-context-builder-chat-mode.md:1 ]

## Out of Scope
- Agent 模式集成（Story 7.6）
- Run 模式集成（Story 7.7）

## Dependencies
- **Story 7-3**: Create Context Builder Module (done)
- **Story 7-4**: Create Context Compression Module

---

## Technical Context

### 当前实现分析

需要分析 `chat:send` IPC handler 的当前实现，识别需要修改的位置。

### 目标实现流程

```
用户发送消息
    ↓
1. 持久化用户消息 (appendConversationMessage)
    ↓
2. 加载对话历史 (loadConversationMessages)
    ↓
3. 压缩检查 (CompressionPipeline.compressIfNeeded)
    ↓
4. 构建 LLM 消息 (buildLlmMessagesFromConversation)
    ↓
5. 调用 chatToolLoop
    ↓
6. 持久化 LLM 响应 (onAssistantMessage callback)
    ↓
7. 持久化工具结果 (onToolMessage callback)
    ↓
8. 返回结果 / 错误处理
```

### 代码修改位置

```typescript
// electron/main.ts 中的 chat:send handler
ipcMain.handle('chat:send', async (event, params) => {
    const { projectRoot, conversationId, content } = params
    
    // 1. 持久化用户消息
    const userMessage: ConversationMessage = {
        id: randomUUID(),
        role: 'user',
        content,
        createdAt: new Date().toISOString(),
        mode: 'chat',
    }
    runtimeStore.appendConversationMessage(projectRoot, conversationId, userMessage)
    
    // 2. 加载对话历史
    const history = runtimeStore.loadConversationMessages(projectRoot, conversationId)
    
    // 3. 压缩检查
    const pipeline = new CompressionPipeline({ tokenBudget: settings.tokenBudget })
    const { messages: compressedMessages } = pipeline.compressIfNeeded(history)
    
    // 4. 构建 LLM 消息
    const { messages } = buildLlmMessagesFromConversation({
        messages: compressedMessages,
        mode: 'chat',
        settings: { includeSystemPrompt: true },
    })
    
    // 5. 调用 chatToolLoop
    const result = await chatToolLoop({
        // ...existing params...
        messages,
        onAssistantMessage: (msg) => {
            // 6. 持久化 LLM 响应
            runtimeStore.appendConversationMessage(projectRoot, conversationId, {
                id: randomUUID(),
                ...msg,
                createdAt: new Date().toISOString(),
                mode: 'chat',
            })
        },
        onToolMessage: (msg) => {
            // 7. 持久化工具结果
            runtimeStore.appendConversationMessage(projectRoot, conversationId, {
                id: randomUUID(),
                ...msg,
                createdAt: new Date().toISOString(),
                mode: 'chat',
                partType: 'tool_result',
            })
        },
    })
    
    // 8. 错误处理
    if (!result.success) {
        const errorMessage: ConversationMessage = {
            id: randomUUID(),
            role: 'assistant',
            content: `Error: ${result.error.message}`,
            createdAt: new Date().toISOString(),
            mode: 'chat',
            partType: 'error',
        }
        runtimeStore.appendConversationMessage(projectRoot, conversationId, errorMessage)
    }
    
    return result
})
```

---

## Files to Modify

| File | Changes |
|------|---------|
| `electron/main.ts` | 修改 `chat:send` handler，集成 context-builder 和 compression |

## Files to Import

| Module | From |
|--------|------|
| `buildLlmMessagesFromConversation` | `./core/context-builder` |
| `CompressionPipeline` | `./core/context-compression` |

---

## Testing Strategy

### Manual Testing
1. 发送消息，验证响应正常
2. 发送多轮消息，验证 LLM 能"记住"之前的内容
3. 关闭应用重开，验证对话历史加载正确
4. 触发 LLM 错误，验证错误显示为 assistant 消息

### Unit Tests (Optional)
```typescript
describe('chat:send integration', () => {
    it('should persist user message before LLM call', () => {})
    it('should persist assistant message after LLM response', () => {})
    it('should persist tool results', () => {})
    it('should persist error as assistant message', () => {})
    it('should compress messages when over threshold', () => {})
})
```

---

## Related Artifacts
- Architecture: `_bmad-output/architecture/unified-conversation-context.md`
- Epic Plan: `_bmad-output/implementation-artifacts/7-unified-conversation-context.md`
- Story 7-3: Context Builder (done)
- Story 7-4: Context Compression

## Dev Agent Record

### Agent Model Used
GPT-5 (Codex CLI)

### Debug Log References
- `crewagent-runtime`: `npm test -- electron/services/chatContextBuilder.test.ts` (npm warned about unknown user config `python`/`install`).

### Completion Notes List
- Added chat context builder helper to keep callAgentChat logic testable.
- Added integration tests for chat context building, injection, and compression pipeline usage.
- Avoided duplicate persistence by relying on chat UI for user/assistant content while persisting tool-call messages for context.
- Injected current user input into context when it has not yet been persisted.
- Persisted non-LLM chat errors as assistant messages (skipping aborted requests).
- Wired compression pipeline into chat tool loop for per-turn historical compression.

### File List
- crewagent-runtime/electron/main.ts
- crewagent-runtime/electron/services/chatToolLoop.ts
- crewagent-runtime/electron/services/chatContextBuilder.ts
- crewagent-runtime/electron/services/chatContextBuilder.test.ts

## Change Log
- 2026-01-18: Completed chat-mode context integration updates and documented changes.
- 2026-01-19: Added chat context builder helper and integration tests for chat context assembly.
