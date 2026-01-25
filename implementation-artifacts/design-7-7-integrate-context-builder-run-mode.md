# Design: Integrate Context Builder into Run Mode (ExecutionEngine)

**Story:** `7-7-integrate-context-builder-run-mode.md`  
**设计原则:** 最小改动、持久化优先、可恢复性

---

## 设计目标

1. **数据持久化**：消除 `session.history` 与 `messages.json` 的分离
2. **可恢复性**：重启后可从对话日志恢复 Run 进度
3. **压缩集成**：删除 `trimHistory`，使用 `CompressionPipeline`

补充（后续 Story）：当 workflow 已 `completed` 时进入 Post-Completion Prompt Profile，不再注入任何 “active step” 信息（不包含 currentStepId/currentStepName/outgoingEdges 等）。详见 Story 7-11。

---

## 改动概览

| 类别 | 改动项 | Line |
|------|--------|------|
| **删除** | `RunSession.history` | L100 |
| **删除** | `trimHistory` 方法 | L308-311 |
| **删除** | `trimHistory` 调用 (5处) | L529, L669, L687, L806, L825 |
| **修改** | 消息读取逻辑 | L618 |
| **新增** | 5处消息持久化 | L528, L660, L686, L801, L824 |
| **新增** | `conversationId` 到 RunSession | L99-104 |
| **集成** | CompressionPipeline | L618 附近 |

---

## 详细改动

### 1. RunSession 类型修改

```diff
type RunSession = {
-   history: OpenAIChatMessage[]
+   conversationId: string  // 关联的对话 ID
    lastFailFingerprint: string | null
    consecutiveFailCount: number
    streamSequence: number
}
```

### 2. getOrCreateSession 修改

```diff
private getOrCreateSession(runId: string): RunSession {
    const existing = this.sessions.get(runId)
    if (existing) return existing
-   const created: RunSession = { history: [], lastFailFingerprint: null, consecutiveFailCount: 0, streamSequence: 0 }
+   const created: RunSession = { 
+       conversationId: `run-${runId}`,  // 使用 runId 作为对话 ID
+       lastFailFingerprint: null, 
+       consecutiveFailCount: 0, 
+       streamSequence: 0 
+   }
    this.sessions.set(runId, created)
    return created
}
```

### 3. 删除 trimHistory

```diff
- private trimHistory(history: OpenAIChatMessage[]) {
-     if (history.length <= this.maxHistoryMessages) return
-     history.splice(0, history.length - this.maxHistoryMessages)
- }
```

### 4. 消息持久化 (5处改动)

#### L528: 用户输入
```diff
if (turn === 0 && typeof params.userInput === 'string' && params.userInput.trim()) {
-   session.history.push({ role: 'user', content: this.formatRunUserInput(state.currentNodeId, params.userInput) })
-   this.trimHistory(session.history)
+   this.deps.runtimeStore.appendConversationMessage(projectRoot, session.conversationId, {
+       id: randomUUID(),
+       role: 'user',
+       content: this.formatRunUserInput(state.currentNodeId, params.userInput),
+       createdAt: nowIso(),
+       mode: 'run',
+       runId,
+   })
}
```

#### L660-669: LLM 错误
```diff
- session.history.push({
-     role: 'assistant',
-     content: this.formatLlmFailureForHistory({ ... }),
- })
- this.trimHistory(session.history)
+ this.deps.runtimeStore.appendConversationMessage(projectRoot, session.conversationId, {
+     id: randomUUID(),
+     role: 'assistant',
+     content: this.formatLlmFailureForHistory({ ... }),
+     createdAt: nowIso(),
+     mode: 'run',
+     runId,
+     partType: 'error',
+ })
```

#### L686-687: Assistant 消息
```diff
const assistant = llmRes.message
- session.history.push(assistant)
- this.trimHistory(session.history)
+ this.deps.runtimeStore.appendConversationMessage(projectRoot, session.conversationId, {
+     id: randomUUID(),
+     role: 'assistant',
+     content: assistant.content,
+     tool_calls: assistant.tool_calls,
+     createdAt: nowIso(),
+     mode: 'run',
+     runId,
+ })
```

#### L801-806: 工具结果
```diff
- session.history.push({
-     role: 'tool',
-     tool_call_id: call.id,
-     content: toolResultContent,
- })
- this.trimHistory(session.history)
+ this.deps.runtimeStore.appendConversationMessage(projectRoot, session.conversationId, {
+     id: randomUUID(),
+     role: 'tool',
+     content: toolResultContent,
+     tool_call_id: call.id,
+     createdAt: nowIso(),
+     mode: 'run',
+     runId,
+     partType: 'tool_result',
+     toolName,
+     duration: Date.now() - toolStartedAt,
+ })
```

#### L824-825: Assistant 消息 (后续)
与 L686-687 相同模式。

### 5. 消息读取 (L618)

```diff
- const requestMessages: OpenAIChatMessage[] = [...toOpenAiMessages(baseMessages), ...session.history]
+ // 加载对话历史
+ const history = this.deps.runtimeStore.loadConversationMessages(projectRoot, session.conversationId)
+ 
+ // 使用 Context Builder
+ const { messages: contextMessages } = buildLlmMessagesFromConversation({
+     messages: history,
+     mode: 'run',
+     agent: {
+         id: effectiveAgentDefinition.id,
+         name: effectiveAgentDefinition.metadata?.name ?? effectiveAgentDefinition.id,
+         role: effectiveAgentDefinition.persona?.role ?? 'Assistant',
+         identity: effectiveAgentDefinition.persona?.identity,
+         principles: effectiveAgentDefinition.persona?.principles,
+     },
+     runContext: {
+         packageName: packageId,
+         workflowName: workflowId,
+         currentStepId: state.currentNodeId,
+         currentStepName: node.title,
+         stepInstruction: workflowRes.definition.steps[state.currentNodeId],
+         state: {
+             stepsCompleted: state.stepsCompleted,
+         },
+         graph: {
+             outgoingEdges: workflowRes.definition.graph.edges
+                 .filter(e => e.from === state.currentNodeId)
+                 .map(e => ({
+                     label: e.label,
+                     targetNodeId: e.to,
+                     isDefault: e.isDefault,
+                 })),
+         },
+     },
+     settings: { includeSystemPrompt: true },
+ })
+ 
+ // 压缩
+ const compressionPipeline = new CompressionPipeline({
+     tokenBudget: settings.llm.tokenBudget ?? 128000,
+ })
+ let requestMessages = contextMessages
+ if (compressionPipeline.shouldCompress(history)) {
+     const compressed = compressionPipeline.compressIfNeeded(history)
+     // 重新构建
+     const { messages: compressedMessages } = buildLlmMessagesFromConversation({
+         messages: compressed.messages,
+         mode: 'run',
+         // ... same options ...
+     })
+     requestMessages = compressedMessages
+ }
```

---

## 依赖添加

```typescript
// 顶部 import
import { buildLlmMessagesFromConversation } from '../core/context-builder'
import { CompressionPipeline } from '../core/context-compression'
```

---

## RuntimeStoreLike 接口扩展

```typescript
export interface RuntimeStoreLike {
    // ... existing methods ...
    
    // 新增
    loadConversationMessages(projectRoot: string, conversationId: string): ConversationMessage[]
    appendConversationMessage(projectRoot: string, conversationId: string, message: ConversationMessage): void
}
```

---

## 测试策略

### 现有测试文件
`electron/services/executionEngine.test.ts`

### 新增测试用例

```typescript
// 在 executionEngine.test.ts 中添加

describe('ExecutionEngine - Context Integration', () => {
    it('should persist user input to conversation log', async () => {
        const mockRuntimeStore = createMockRuntimeStore()
        const engine = new ExecutionEngineImpl({ runtimeStore: mockRuntimeStore, ... })
        
        await engine.continue({
            projectRoot: '/test',
            runId: 'run-1',
            userInput: 'test input',
            // ...
        })
        
        expect(mockRuntimeStore.appendConversationMessage).toHaveBeenCalledWith(
            '/test',
            'run-run-1',
            expect.objectContaining({
                role: 'user',
                mode: 'run',
                runId: 'run-1',
            })
        )
    })
    
    it('should load history from conversation log', async () => {
        const mockRuntimeStore = createMockRuntimeStore()
        mockRuntimeStore.loadConversationMessages.mockReturnValue([
            { id: '1', role: 'user', content: 'previous', createdAt: '...' }
        ])
        const engine = new ExecutionEngineImpl({ runtimeStore: mockRuntimeStore, ... })
        
        await engine.continue({ ... })
        
        expect(mockRuntimeStore.loadConversationMessages).toHaveBeenCalledWith(
            '/test',
            'run-run-1'
        )
    })
    
    it('should persist assistant messages with runId', async () => {
        // ...
    })
    
    it('should persist tool results with runId', async () => {
        // ...
    })
    
    it('should NOT call trimHistory (method removed)', async () => {
        const engine = new ExecutionEngineImpl({ ... })
        expect((engine as any).trimHistory).toBeUndefined()
    })
})
```

### 运行测试命令

```bash
cd crewagent-runtime
npm test -- electron/services/executionEngine.test.ts
```

### 手动测试 Checklist

| 测试项 | 步骤 | 预期结果 |
|--------|------|---------|
| Run 执行 | 启动一个 Run，执行几个节点 | 正常完成 |
| 消息持久化 | 检查 messages.json | 包含 mode='run' 和 runId |
| Run 恢复 | 重启应用，继续 Run | 能看到之前的对话历史 |
| 压缩触发 | 运行长时间 Run | 日志显示 "Context compression applied" |

---

## 风险与缓解

| 风险 | 级别 | 缓解措施 |
|------|------|----------|
| 频繁 JSON 读写影响性能 | Medium | 考虑批量写入或异步写入 |
| 现有测试可能失败 | Medium | 更新 mock 以支持新方法 |
| promptComposer vs context-builder 冲突 | Low | 逐步迁移，保留 promptComposer |

---

## 实现顺序建议

1. 扩展 `RuntimeStoreLike` 接口
2. 修改 `RunSession` 类型
3. 修改 `getOrCreateSession`
4. 逐个替换 5 处消息写入
5. 修改 L618 消息读取
6. 删除 `trimHistory`
7. 更新测试
8. 验证

---

## Related Artifacts
- Story 7-5: Chat Mode Integration (done)
- Story 7-6: Agent Mode Integration (validated)
