# Story 7-7: Integrate Context Builder into Run Mode (ExecutionEngine)

## Overview
**Epic**: 7 – Unified Conversation Context  
**Priority**: Phase 3 (depends on 7.5)  
**Status**: `done`

## Goal
将 ExecutionEngine 的 `session.history` 替换为对话日志支持，使 Run 模式与统一日志同步。实现可恢复性，消除内存历史与持久化消息的分离。

## Business Value
- **可恢复**：重启后可从对话日志恢复 Run 进度
- **统一数据源**：消除 `session.history` 与持久化消息的分离
- **一致性**：Run 模式、Chat 模式、Agent 模式使用统一的日志系统

## Acceptance Criteria
1. **数据源替换**：
   - 删除内存中的 `session.history`
   - 改为从对话日志读写
   - 允许内存缓存（但必须可从持久化重建）
2. **Run 上下文注入**：`mode='run'` 时额外注入 `RUN_DIRECTIVE` + `NODE_BRIEF`
3. **消息标记**：带 `runId` 区分多次运行
4. **删除 trimHistory**：压缩逻辑移至 Context Compression 模块

## Out of Scope
- UI 上下文指示器（Story 7.8）
- 对话删除清理（Story 7.9）

## Dependencies
- **Story 7-5**: Integrate Context Builder into Chat Mode (done)
- **Story 7-4**: Context Compression Module (done)

---

## Technical Context

### 当前 ExecutionEngine 架构

```typescript
// executionEngine.ts
class ExecutionEngineImpl {
    private sessions = new Map<string, Session>()
    
    interface Session {
        history: OpenAIChatMessage[]  // 内存中的对话历史
        // ...
    }
    
    private trimHistory(history: OpenAIChatMessage[]) {
        // 基于消息数量截断 - 需要删除
    }
}
```

**问题**：
- `session.history` 仅存在于内存，重启后丢失
- `trimHistory` 仅基于消息数量，不考虑 token 预算
- 与持久化的 `messages.json` 不同步

### 目标架构

```
┌─────────────────────────────────────────────────────────────────┐
│ ExecutionEngine                                                  │
│                                                                  │
│  runs:continue IPC                                               │
│       ↓                                                          │
│  1. 获取/创建对话 ID (基于 runId)                                 │
│       ↓                                                          │
│  2. 加载对话历史 (loadConversationMessages)                       │
│       ↓                                                          │
│  3. buildLlmMessagesFromConversation({ mode: 'run', ... })       │
│       ↓                                                          │
│  4. CompressionPipeline.compressIfNeeded                         │
│       ↓                                                          │
│  5. chatToolLoop (带 compressionPipeline)                        │
│       ├── onAssistantMessage → 持久化                            │
│       └── onToolMessage → 持久化                                 │
│       ↓                                                          │
│  6. 返回结果                                                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Changes

### 1. 删除 session.history

```diff
interface Session {
-   history: OpenAIChatMessage[]
    runId: string
    conversationId: string  // 新增：关联的对话 ID
    // ...
}
```

### 2. 删除 trimHistory

```diff
- private trimHistory(history: OpenAIChatMessage[]) {
-     if (history.length <= this.maxHistoryMessages) return
-     history.splice(0, history.length - this.maxHistoryMessages)
- }
```

### 3. 使用 Context Builder

```typescript
// ExecutionEngine.continue 方法改造

async continue(params: ContinueParams) {
    const session = this.getOrCreateSession(params)
    
    // 加载对话历史
    const history = runtimeStore.loadConversationMessages(
        params.projectRoot,
        session.conversationId
    )
    
    // 使用 Context Builder
    const { messages } = buildLlmMessagesFromConversation({
        messages: history,
        mode: 'run',
        agent: this.getAgentInfo(params),
        runContext: {
            packageName: params.packageId,
            workflowName: params.workflowId,
            currentStepId: session.currentNodeId,
            currentStepName: session.currentNodeTitle,
            stepInstruction: session.stepInstruction,
        },
    })
    
    // 压缩
    const pipeline = new CompressionPipeline({ tokenBudget: ... })
    // ... 
    
    // chatToolLoop
    await chatToolLoop({
        messages,
        compressionPipeline: pipeline,
        onAssistantMessage: (msg) => {
            runtimeStore.appendConversationMessage(projectRoot, session.conversationId, {
                ...msg,
                mode: 'run',
                runId: params.runId,
            })
        },
        onToolMessage: (msg) => {
            runtimeStore.appendConversationMessage(projectRoot, session.conversationId, {
                ...msg,
                mode: 'run',
                runId: params.runId,
            })
        },
    })
}
```

### 4. Run 恢复逻辑

```typescript
// 重启后恢复 Run
async resumeRun(params: ResumeParams) {
    // 从对话日志加载历史
    const history = runtimeStore.loadConversationMessages(
        params.projectRoot,
        params.conversationId
    )
    
    // 重建 session 状态
    const session = this.rebuildSessionFromHistory(history, params)
    
    // 继续执行
    return this.continue(params)
}
```

---

## Files to Modify

| File | Changes |
|------|---------|
| `electron/services/executionEngine.ts` | 主要改动：删除 session.history、trimHistory，集成 context-builder |

## Files to Use

| Module | From |
|--------|------|
| `buildLlmMessagesFromConversation` | `./core/context-builder` |
| `CompressionPipeline` | `./core/context-compression` |

---

## Testing Strategy

### Unit Tests
```typescript
describe('ExecutionEngine - Context Integration', () => {
    it('should load history from conversation log', () => {})
    it('should use context-builder for run mode', () => {})
    it('should include runContext in system prompt', () => {})
    it('should persist messages with runId', () => {})
    it('should apply compression when needed', () => {})
})
```

### Manual Testing
1. 启动 Run，执行几个节点
2. 重启应用
3. 查看 Run 是否能从对话日志恢复

---

## 行动项

### 问题清单
1. **上下文压缩预算不匹配模型**：默认 128k token 预算可能导致 8k/16k/32k 模型在压缩触发前就已超限。建议按模型配置 `tokenBudget` 或恢复保守上限。
2. **用户输入封装不一致**：Chat/Agent 仅在“最新消息不是用户输入”时注入 `USER_INPUT`，而 UI 持久化的是原始 `WIDGET_SUBMIT`，与系统提示不一致，可能影响解析。建议统一规范化存储或始终包装。
3. **工具输出持久化异常未捕获**：`prepareToolMessageForStorage` 的同步写文件在 try/catch 之外，失败会中断 Run。建议把文件 IO 放入受控异常处理。
4. **Run Protocol 重复注入**：系统提示与 Run 指令都包含协议文本，增加 token 与冲突风险。建议统一来源。
5. **Run Protocol 注入位置规则**：`run-directive-protocol.md` 应只出现在 `RUN_DIRECTIVE`（User 内容块）中，System Prompt 不再注入。

### 开放问题
1. Run Protocol **固定仅保留在** `RUN_DIRECTIVE`（User 内容块），System Prompt **不再注入**。
2. Chat/Agent 的用户输入（含 widget 提交）**统一使用** `USER_INPUT` 包装。

---

## Related Artifacts
- **Design Document**: [design-7-7-integrate-context-builder-run-mode.md](./design-7-7-integrate-context-builder-run-mode.md)
- **Validation Report**: [validation-report-story-7-7.md](./validation-report-story-7-7.md)
- Architecture: `_bmad-output/architecture/unified-conversation-context.md`
- Epic Plan: `_bmad-output/implementation-artifacts/7-unified-conversation-context.md`
- Story 7-5: Chat Mode Integration (done)
- Story 7-6: Agent Mode Integration (validated)
