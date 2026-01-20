# Design: Integrate Context Builder into Chat Mode

**Story:** `7-5-integrate-context-builder-chat-mode.md`  
**设计原则:** 最小改动、向后兼容、增量集成

---

## 设计目标

1. **多轮对话**：Chat 模式带上对话历史，LLM 能"记住"之前的内容
2. **错误可恢复**：错误后可以说"继续"
3. **压缩集成**：Loop 内外均可触发压缩

---

## 改动范围

| 文件 | 改动类型 | 说明 |
|------|---------|------|
| `electron/main.ts` | MODIFY | `callAgentChat` 函数 |
| `electron/services/chatToolLoop.ts` | MODIFY | 添加 compressionPipeline 参数 |

---

## 当前 callAgentChat 流程

```
用户发送消息
    ↓
promptComposer.compose() → 生成 system + user 消息
    ↓
chatToolLoop() → LLM 调用，无历史
    ↓
返回结果
```

**问题**：每次调用都是全新对话，无上下文连续性

---

## 目标流程

```
用户发送消息
    ↓
1. 持久化用户消息 (appendConversationMessage)
    ↓
2. 加载对话历史 (loadConversationMessages)
    ↓
3. 构建 LLM 上下文 (buildLlmMessagesFromConversation)
    ↓
4. 初始压缩检查 (CompressionPipeline.compressIfNeeded)
    ↓
5. chatToolLoop (带 compressionPipeline 参数)
    ├── Loop 内压缩检查 (每轮)
    ├── onAssistantMessage → 增量持久化
    └── onToolMessage → 增量持久化
    ↓
6. 返回结果 / 错误处理
```

---

## 详细实现

### callAgentChat 改造

```typescript
// electron/main.ts

import { buildLlmMessagesFromConversation } from './core/context-builder'
import { CompressionPipeline } from './core/context-compression'

const callAgentChat = async (params: {
    projectRoot: string
    packageId: string
    agentId: string
    agentDefinition: AgentDefinition
    mode?: 'agent' | 'chat'
    conversationId?: string
    userInput?: string
    dataContext?: { label: string; path: string; preview: string }
    extraUserMessages?: string[]
    streamTarget?: WebContents
}) => {
    const { projectRoot, packageId, agentId, agentDefinition } = params
    const settings = runtimeStore.getSettings()
    const mode = params.mode === 'chat' ? 'chat' : 'agent'
    
    const conversationId = params.conversationId?.trim() || null
    const chatRunId = `chat-${conversationId || randomUUID()}`
    
    // ========================================
    // Step 1: 持久化用户消息
    // ========================================
    if (conversationId && params.userInput?.trim()) {
        const userMessage: ConversationMessage = {
            id: randomUUID(),
            role: 'user',
            content: `USER_INPUT\n${params.userInput}`,
            createdAt: new Date().toISOString(),
            mode,
            runId: chatRunId,
            agentId,
        }
        runtimeStore.appendConversationMessage(projectRoot, conversationId, userMessage)
    }
    
    // ========================================
    // Step 2: 加载对话历史
    // ========================================
    const history: ConversationMessage[] = conversationId
        ? runtimeStore.loadConversationMessages(projectRoot, conversationId)
        : []
    
    // ========================================
    // Step 3: 构建 LLM 上下文
    // ========================================
    const toolPolicy = mergeToolPolicies({ system: settings.tools, agent: agentDefinition.tools })
    
    const { messages } = buildLlmMessagesFromConversation({
        messages: history,
        mode,
        agent: {
            id: agentDefinition.id,
            name: agentDefinition.metadata?.name ?? agentDefinition.id,
            role: agentDefinition.persona?.role ?? 'Assistant',
            identity: agentDefinition.persona?.identity,
            communicationStyle: agentDefinition.persona?.communication_style,
            principles: agentDefinition.persona?.principles,
            systemPrompt: agentDefinition.systemPrompt,
        },
        toolPolicy: toolPolicy ? {
            allowedCategories: toolPolicy.allow?.categories,
            deniedCategories: toolPolicy.deny?.categories,
            allowedTools: toolPolicy.allow?.tools,
            deniedTools: toolPolicy.deny?.tools,
        } : undefined,
        settings: {
            includeSystemPrompt: true,
        },
    })
    
    // 追加 extraUserMessages（如 EXEC_SCRIPT）
    for (const extra of params.extraUserMessages ?? []) {
        if (extra?.trim()) {
            messages.push({ role: 'user', content: extra })
        }
    }
    
    // 追加 dataContext（如果有）
    if (params.dataContext) {
        const dataMsg = `DATA_CONTEXT\n- Label: ${params.dataContext.label}\n- Path: ${params.dataContext.path}\n\n${params.dataContext.preview}`
        messages.push({ role: 'user', content: dataMsg })
    }
    
    // ========================================
    // Step 4: 创建 CompressionPipeline
    // ========================================
    const compressionPipeline = new CompressionPipeline({
        tokenBudget: settings.llm.tokenBudget ?? 128000,
        triggerThreshold: 0.8,
        targetUsage: 0.5,
        minRecentMessages: 10,
    })
    
    // 初始压缩检查（Loop 开始前）
    const initialUsage = compressionPipeline.getUsage(messages)
    if (compressionPipeline.shouldCompress(messages)) {
        const compressed = compressionPipeline.compressIfNeeded(messages)
        messages.length = 0
        messages.push(...compressed.messages)
    }
    
    // ========================================
    // Step 5: chatToolLoop（带 compression 参数）
    // ========================================
    const tools = toolHost.getVisibleTools({ packageId, agentId, tools: toolPolicy })
    
    const loopRes = await chatToolLoop({
        llmAdapter,
        toolHost,
        messages,
        tools,
        config: llmConfig,
        context: { projectRoot, packageId, workflowId: 'chat', runId: chatRunId, agentId },
        maxTurns: Math.max(1, settings.engine?.maxTurns ?? 20),
        signal: controller.signal,
        
        // 压缩管道（Loop 内使用）
        compressionPipeline,
        
        // 持久化回调
        onAssistantMessage: (assistant) => {
            if (!assistant.tool_calls?.length) return
            appendConversationMessage({
                id: randomUUID(),
                role: 'assistant',
                content: assistant.content ?? null,
                tool_calls: assistant.tool_calls,
                createdAt: new Date().toISOString(),
                mode,
                runId: chatRunId,
                agentId,
            })
        },
        onToolMessage: (toolMessage) => {
            appendConversationMessage({
                id: randomUUID(),
                role: 'tool',
                content: toolMessage.content,
                tool_call_id: toolMessage.tool_call_id,
                createdAt: new Date().toISOString(),
                toolName: toolMessage.toolName,
                duration: toolMessage.duration,
                partType: 'tool_result',
                mode,
                runId: chatRunId,
                agentId,
            })
        },
        
        // ...existing callbacks...
    })
    
    // ========================================
    // Step 6: 错误处理
    // ========================================
    if (!loopRes.success) {
        const error = loopRes.error
        if (error?.code?.startsWith('LLM_')) {
            appendConversationMessage({
                id: randomUUID(),
                role: 'assistant',
                content: formatLlmErrorMessage(error),
                createdAt: new Date().toISOString(),
                partType: 'error',
                mode,
                runId: chatRunId,
                agentId,
            })
        }
        return loopRes
    }
    
    return { success: true, assistant: loopRes.message, usage: loopRes.usage }
}
```

### chatToolLoop 改造

```typescript
// electron/services/chatToolLoop.ts

import type { CompressionPipeline } from '../core/context-compression'

export interface ChatToolLoopParams {
    // ...existing params...
    
    /** 压缩管道（可选，用于 Loop 内压缩） */
    compressionPipeline?: CompressionPipeline
}

export async function chatToolLoop(params: ChatToolLoopParams) {
    const { messages, compressionPipeline } = params
    
    // 记录 Loop 开始时的消息边界
    let loopStartIndex = messages.length
    
    for (let turn = 0; turn < params.maxTurns; turn++) {
        // ====== Loop 内压缩检查 ======
        if (compressionPipeline) {
            const usage = compressionPipeline.getUsage(messages)
            
            if (usage.usagePercent > 0.8) {
                // 只压缩历史消息，保留当前 Loop 消息
                const result = compressionPipeline.compressHistoricalMessages(
                    messages,
                    loopStartIndex
                )
                
                messages.length = 0
                messages.push(...result.messages)
                loopStartIndex = result.newLoopStartIndex
                
                console.log(`[Loop Compression] ${result.messages.length} messages after compression`)
            }
        }
        
        // LLM 调用...
        // 工具执行...
        // 累积消息...
    }
}
```

---

## 数据流图

```
┌────────────────────────────────────────────────────────────────┐
│ callAgentChat                                                   │
│                                                                 │
│  ┌─────────────┐   ┌───────────────┐   ┌────────────────────┐  │
│  │ User Input  │──▶│ Persist User  │──▶│ Load History       │  │
│  └─────────────┘   │ Message       │   │ (messages.json)    │  │
│                    └───────────────┘   └────────────────────┘  │
│                                                 │               │
│                                                 ▼               │
│                    ┌───────────────────────────────────────┐   │
│                    │ buildLlmMessagesFromConversation      │   │
│                    │ (context-builder)                     │   │
│                    └───────────────────────────────────────┘   │
│                                                 │               │
│                                                 ▼               │
│                    ┌───────────────────────────────────────┐   │
│                    │ CompressionPipeline.compressIfNeeded  │   │
│                    │ (初始压缩)                             │   │
│                    └───────────────────────────────────────┘   │
│                                                 │               │
│                                                 ▼               │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ chatToolLoop                                             │   │
│  │  ┌─────────────────────────────────────────────────┐    │   │
│  │  │ Loop Turn N                                      │    │   │
│  │  │  ├─ Loop 内压缩检查 (历史部分)                    │    │   │
│  │  │  ├─ LLM 调用                                     │    │   │
│  │  │  ├─ onAssistantMessage → 持久化                  │    │   │
│  │  │  ├─ 工具执行                                     │    │   │
│  │  │  └─ onToolMessage → 持久化                       │    │   │
│  │  └─────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                 │               │
│                                                 ▼               │
│                    ┌───────────────────────────────────────┐   │
│                    │ Return Result / Error                 │   │
│                    └───────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────┘
```

---

## 向后兼容

| 场景 | 行为 |
|------|------|
| 无 conversationId | 不加载历史，行为与现在相同 |
| 无 compressionPipeline | chatToolLoop 跳过压缩检查 |
| 旧消息格式 | toOpenAIMessage 已处理兼容 |

---

## 测试策略

### 手动测试
1. 发送消息，验证响应正常
2. 发送多轮消息，验证 LLM 能引用之前的内容
3. 关闭应用重开，验证对话历史加载正确
4. 触发 LLM 错误，验证错误显示为 assistant 消息

### 集成测试
```typescript
describe('Chat Mode with Context Builder', () => {
    it('should persist user message before LLM call', () => {})
    it('should load conversation history', () => {})
    it('should include history in LLM context', () => {})
    it('should compress when over threshold', () => {})
    it('should persist error as assistant message', () => {})
})
```

---

## 依赖

- ✅ **Story 7-3**: Context Builder (done)
- ⏳ **Story 7-4**: Context Compression (待实现)

**建议**：先实现 Story 7-4，再实现 Story 7-5
