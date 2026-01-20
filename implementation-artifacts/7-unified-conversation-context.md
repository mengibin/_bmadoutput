# Epic 7: Unified Conversation Context

**Goal**: 实现统一的对话上下文系统，使 Chat / Agent / Run 三种模式共享历史记录，采用 Claude Code 风格的阈值触发压缩。

**Architecture Reference**: `_bmad-output/architecture/unified-conversation-context.md`

**FRs covered**: 
- FR-MNG-03 (Execution Log)
- NFR-REL-01 (Graceful Recovery)

## Epic Overview

当前 Runtime 存在以下问题：
- Chat / Agent 模式不带历史上下文
- Run 模式的 `session.history` 与持久化消息分离
- 用户说"继续"时 LLM 无法正确恢复

本 Epic 通过统一对话日志 + 独立的上下文构建模块 + Claude Code 风格压缩解决这些问题。

---

## 规则更新（模板注入）

1. **Run 模式专用协议**：`run-directive-protocol.md` 只应出现在 **User 内容块**（`RUN_DIRECTIVE`）中；  
   **System Prompt 不再包含**该协议文本。
2. **适用范围**：该规则仅适用于 **Workflow / Run 模式**。

---

## Story 7.1: Extend ConversationMessage Data Model (Protocol-Complete)

**Epic**: 7 – Unified Conversation Context  
**Priority**: Phase 1  
**Status**: `done`

### Goal
扩展 `ConversationMessage` 类型以支持完整的 OpenAI 协议字段，包括 `role=tool`、`tool_calls`、`tool_call_id`。

### Business Value
- **协议完整性**：存储格式可直接转换为 OpenAI `messages[]`
- **可调试性**：完整保留工具调用链，便于回溯

### Acceptance Criteria
1. **Role 扩展**：`ConversationRole` 增加 `'tool'` 值
2. **新增字段**：
   - `tool_calls?: OpenAIToolCall[]` (assistant only)
   - `tool_call_id?: string` (tool only)
   - `mode?: 'chat' | 'agent' | 'run'`
   - `runId?: string`
   - `includeInContext?: boolean`
3. **向后兼容**：旧消息加载时缺失字段使用安全默认值
4. **类型导出**：新类型从 `shared/types` 导出供前后端共用

### Technical Context
```typescript
type ConversationRole = 'system' | 'user' | 'assistant' | 'tool'

interface ConversationMessage {
  id: string
  role: ConversationRole
  createdAt: string
  content?: string | null
  tool_calls?: OpenAIToolCall[]
  tool_call_id?: string
  mode?: 'chat' | 'agent' | 'run'
  runId?: string
  includeInContext?: boolean
  // ... existing UI fields
}
```

### Files to Modify
- `crewagent-runtime/src/types/conversation.ts`
- `crewagent-runtime/electron/types/conversation.ts`

---

## Story 7.2: Persist Tool-Call Protocol Messages

**Epic**: 7 – Unified Conversation Context  
**Priority**: Phase 1 (depends on 7.1)  
**Status**: `done`

### Goal
在对话持久化流程中存储完整的工具调用消息（`assistant(tool_calls)` → `tool(result)`），而不仅仅是用户可见消息。

### Business Value
- **上下文连续性**：重启后对话历史完整
- **可恢复性**：支持"继续"命令

### Acceptance Criteria
1. **LLM 响应持久化**：存储 `role=assistant` 消息（含 `tool_calls`）
2. **工具结果持久化**：存储 `role=tool` 消息（含 `tool_call_id`）
3. **错误持久化**：LLM 超时/网络错误存储为 `assistant` 消息 + `LLM_ERROR` 块
4. **大输出处理**：工具输出超过阈值时，完整结果写入 `@state/logs/tool-results/<id>.json`，消息中只存预览 + 路径
5. **原子写入**：使用 tmp → rename 模式

### Files to Modify
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/services/chatHandler.ts`
- `crewagent-runtime/electron/services/executionEngine.ts`

---

## Story 7.3: Create Context Builder Module

**Epic**: 7 – Unified Conversation Context  
**Priority**: Phase 2 (depends on 7.1, 7.2)  
**Status**: `done`

### Goal
创建独立的上下文构建模块 `context-builder/`，供 Chat / Agent / Run 三种模式统一调用。

### Business Value
- **代码复用**：三种模式共用一套构建逻辑
- **易维护**：逻辑集中、可独立测试

### Acceptance Criteria
1. **模块结构**：
   ```
   src/core/context-builder/
   ├── index.ts
   ├── types.ts
   ├── buildLlmMessages.ts
   ├── systemPromptComposer.ts
   └── __tests__/
   ```
2. **Public API**：
   ```typescript
   export function buildLlmMessagesFromConversation(
     options: ContextBuildOptions
   ): ContextBuildResult
   ```
3. **输入**：`projectRoot`, `conversationId`, `mode`, `agentDefinition`, `runContext`, `settings`
4. **输出**：`OpenAIChatMessage[]` + debug metadata (counts, tokens, kept/dropped)
5. **System Prompt 处理**：
   - 不持久化到历史
   - 根据 mode 动态拼接（base rules + tool policy + persona）
   - 添加 Mode Banner

### Files to Create
- `crewagent-runtime/src/core/context-builder/index.ts`
- `crewagent-runtime/src/core/context-builder/types.ts`
- `crewagent-runtime/src/core/context-builder/buildLlmMessages.ts`
- `crewagent-runtime/src/core/context-builder/systemPromptComposer.ts`

---

## Story 7.4: Create Context Compression Module

**Epic**: 7 – Unified Conversation Context  
**Priority**: Phase 2 (depends on 7.3)  
**Status**: `done`

### Goal
创建独立的上下文压缩模块 `context-compression/`，采用 Claude Code 风格的阈值触发模式。

### Business Value
- **可扩展**：策略模式支持未来升级（LLM 摘要、向量检索）
- **用户体验**：显示上下文剩余量

### Acceptance Criteria
1. **模块结构**：
   ```
   src/core/context-compression/
   ├── index.ts
   ├── types.ts
   ├── CompressionPipeline.ts
   ├── ContextUsageMonitor.ts
   ├── strategies/
   │   └── KeyMessageExtractionStrategy.ts
   └── __tests__/
   ```
2. **阈值触发**：当 `usagePercent > 80%` 时触发压缩
3. **压缩目标**：压缩后目标占用 `50%`
4. **v1 策略 (KeyMessageExtractionStrategy)**：
   - 保留所有用户消息
   - 保留产出/完成记录（`ARTIFACT_SAVED`, `NODE_COMPLETE`）
   - 保留完整工具调用组
   - 保留最近 N 条消息
   - 历史 system 消息默认不保留（避免重复），仅保留 SUMMARY 类 system 消息（如 `SUMMARY` / `CONVERSATION_SUMMARY`）
   - 可解释打分 + 预算内贪心选取（文件路径/代码块/数值/关键词/近期加分，冗长 tool 结果降分）
   - 超预算兜底（K=4）：裁剪工具结果与超长文本，仍超预算则仅保留最近 K 回合（user + assistant + 回合内 tool 结果）
5. **接口定义**：
   ```typescript
   interface ContextUsage {
     usedTokens: number
     totalBudget: number
     usagePercent: number
     remaining: number
   }
   ```

### Files to Create
- `crewagent-runtime/src/core/context-compression/index.ts`
- `crewagent-runtime/src/core/context-compression/types.ts`
- `crewagent-runtime/src/core/context-compression/CompressionPipeline.ts`
- `crewagent-runtime/src/core/context-compression/ContextUsageMonitor.ts`
- `crewagent-runtime/src/core/context-compression/strategies/KeyMessageExtractionStrategy.ts`

---

## Story 7.5: Integrate Context Builder into Chat Mode

**Epic**: 7 – Unified Conversation Context  
**Priority**: Phase 3 (depends on 7.3, 7.4)  
**Status**: `done`

### Goal
将 Chat 模式的 `chat:send` 改为使用统一的 Context Builder，使其带上对话历史。

### Business Value
- **上下文连续性**：用户可以进行多轮对话
- **继续支持**：错误后可以说"继续"

### Acceptance Criteria
1. **历史加载**：从 `messages.json` 加载对话历史
2. **上下文构建**：调用 `buildLlmMessagesFromConversation(mode='chat')`
3. **消息持久化**：LLM 响应和工具结果存入对话日志
4. **错误展示**：错误显示为 assistant 消息（不仅是 toast）
5. **压缩集成**：在构建上下文时检查并按需压缩

### Files to Modify
- `crewagent-runtime/electron/services/chatHandler.ts`

---

## Story 7.6: Integrate Context Builder into Agent Mode

**Epic**: 7 – Unified Conversation Context  
**Priority**: Phase 3 (depends on 7.5)  
**Status**: `done`

### Goal
将 Agent 模式改为使用统一的 Context Builder，包含 Persona。

### Acceptance Criteria
1. **同 Chat 模式**：历史加载、上下文构建、消息持久化
2. **Persona 注入**：`buildLlmMessagesFromConversation(mode='agent')` 自动包含 persona
3. **模式标识**：消息带 `mode='agent'` 标记

### Files to Modify
- `crewagent-runtime/electron/services/agentHandler.ts`

---

## Story 7.7: Integrate Context Builder into Run Mode (ExecutionEngine)

**Epic**: 7 – Unified Conversation Context  
**Priority**: Phase 3 (depends on 7.5)  
**Status**: `done`

### Goal
将 ExecutionEngine 的 `session.history` 替换为对话日志支持，使 Run 模式与统一日志同步。

### Business Value
- **可恢复**：重启后可从对话日志恢复
- **统一数据源**：消除 `session.history` 与持久化消息的分离

### Acceptance Criteria
1. **数据源替换**：
   - 删除内存中的 `session.history`
   - 改为从对话日志读写
   - 允许内存缓存（但必须可从持久化重建）
2. **Run 上下文注入**：`mode='run'` 时额外注入 `RUN_DIRECTIVE` + `NODE_BRIEF`
3. **消息标记**：带 `runId` 区分多次运行
4. **删除 trimHistory**：压缩逻辑移至 Context Compression 模块
5. **Run Protocol 注入位置**：`run-directive-protocol.md` 仅在 `RUN_DIRECTIVE`（User 内容块）中出现，System Prompt 不再注入

### Files to Modify
- `crewagent-runtime/electron/services/executionEngine.ts`

---

## Story 7.8: Add Context Usage UI Indicator (Optional)

**Epic**: 7 – Unified Conversation Context  
**Priority**: Phase 4 (optional)  
**Status**: `done`

### Goal
在 UI 中显示上下文剩余量指示器，类似 Claude Code。

### Acceptance Criteria
1. **IPC 事件**：后端发送 `context:usage` 事件
2. **状态显示**：
   - `normal`：正常（绿色）
   - `warning`：> 60%（黄色）
   - `compressing`：正在压缩（动画）
3. **UI 位置**：显示在聊天输入框附近

### Files to Modify
- `crewagent-runtime/electron/main.ts` (IPC)
- `crewagent-runtime/src/components/ChatInput/` (UI)

---

## Story 7.9: Conversation Deletion with Complete Cleanup

**Epic**: 7 – Unified Conversation Context  
**Priority**: Phase 4  
**Status**: `done`

### Goal
扩展 Conversation 删除功能，确保删除时清理所有相关数据（扩展后的消息、工具输出文件等）。

### Business Value
- **数据完整性**：不留下孤立的工具输出文件
- **磁盘空间**：及时清理不再需要的数据

### Acceptance Criteria
1. **消息清理**：删除 `<conversationId>/messages.json`（含新增的 `role=tool` 消息）
2. **工具输出清理**：删除 `@state/logs/tool-results/<conversationId>/` 下的所有文件
3. **元数据清理**：从 `index.json` 中移除 conversation
4. **原子操作**：先删除文件再更新 index，确保不会留下孤立引用
5. **UI 确认**：删除前提示将清理所有聊天记录和工具输出

### Technical Context
```typescript
async function deleteConversation(projectRoot: string, conversationId: string) {
  // 1. 删除 messages.json
  await fs.rm(getMessagesPath(projectRoot, conversationId), { force: true })
  
  // 2. 删除工具输出文件
  await fs.rm(getToolResultsPath(projectRoot, conversationId), { 
    recursive: true, 
    force: true 
  })
  
  // 3. 删除 conversation 目录
  await fs.rm(getConversationDir(projectRoot, conversationId), { 
    recursive: true 
  })
  
  // 4. 更新 index.json
  await updateIndex(projectRoot, index => 
    index.filter(c => c.id !== conversationId)
  )
}
```

### Files to Modify
- `crewagent-runtime/electron/stores/runtimeStore.ts` (扩展 `deleteConversation`)

---

## Story 7.10: Conversation Message UI Rendering

**Epic**: 7 – Unified Conversation Context  
**Priority**: Phase 3 (blocking for proper UX)  
**Status**: `done`

### Goal
修复对话重新加载后的消息 UI 渲染问题。Story 7.2 实现了工具调用消息的持久化，但前端 UI 没有更新过滤逻辑，导致重新加载对话时显示了不应展示的内部消息内容。

### Business Value
- **用户体验**：对话历史清晰可读，与实时对话一致
- **一致性**：所有模式（Chat/Agent/Run）统一渲染逻辑

### Acceptance Criteria
1. **tool_calls 消息**：`role='assistant'` 且 `tool_calls.length > 0` 的消息，隐藏或特殊处理
2. **tool 结果**：`role='tool'` 消息不显示为对话气泡
3. **所有模式**：Chat、Agent、Run 模式统一渲染规则

### Files to Modify
- `crewagent-runtime/src/utils/messageRenderer.ts` (新建)
- `crewagent-runtime/src/components/MessageList.tsx` 或等效组件

---

## Dependencies Graph

```
7.1 (Data Model)
    ↓
7.2 (Persist Tool Messages)
    ↓
7.3 (Context Builder Module) ←───┐
    ↓                            │
7.4 (Context Compression Module) │
    ↓                            │
7.5 (Integrate Chat)  ───────────┤
    ↓                            │
7.6 (Integrate Agent) ───────────┤
    ↓                            │
7.7 (Integrate Run) ─────────────┘
    ↓
7.8 (UI Indicator) [optional]
    ↓
7.9 (Message Deletion) [depends on 7.2]
```

---

## Testing Strategy

### Unit Tests
- Context Builder：正确保留最新用户消息、裁剪规则、Mode Banner
- Context Compression：阈值触发、关键消息提取、工具调用组完整性
- Persistence：`role=tool` 消息正确存储和加载

### Integration Tests
- Chat → 重启 → 历史保留
- Mode 切换：Chat → Agent → Run，历史保留
- 错误恢复：模拟超时后"继续"

### Manual Verification
1. 打开项目，发送多条消息
2. 重启应用，验证消息历史完整
3. 切换 Chat → Agent 模式，验证历史保留
4. 模拟长对话触发压缩，验证关键消息保留

---

## Related Artifacts
- Architecture: `_bmad-output/architecture/unified-conversation-context.md`
- Current persistence: `_bmad-output/implementation-artifacts/tech-spec-5-8-persist-conversations.md`
