# Story 7-2: Persist Tool-Call Protocol Messages

## Overview
**Epic**: 7 – Unified Conversation Context  
**Priority**: Phase 1 (depends on 7.1)  
**Status**: `done`

## Goal
在对话持久化流程中存储完整的工具调用消息（`assistant(tool_calls)` → `tool(result)`），而不仅仅是用户可见消息。使消息历史可直接转换为 OpenAI `messages[]` 格式。

## Business Value
- **上下文连续性**：重启后对话历史完整
- **可恢复性**：支持"继续"命令
- **可调试性**：完整记录工具调用链

## Acceptance Criteria
1. **LLM 响应持久化**：存储 `role=assistant` 消息（含 `tool_calls`）
2. **工具结果持久化**：存储 `role=tool` 消息（含 `tool_call_id`）
3. **错误持久化**：LLM 超时/网络错误存储为 `assistant` 消息 + `partType=error`
4. **大输出处理**：工具输出超过阈值（如 50KB）时：
   - 完整结果写入 `@state/logs/tool-results/<messageId>.json`
   - 消息中只存预览 + 路径引用
5. **原子写入**：使用 tmp → rename 模式（已实现）

## Out of Scope
- Context Builder 实现（Story 7.3）
- 压缩策略（Story 7.4）
- UI 展示优化

## Dependencies
- **Story 7-1**: Extend ConversationMessage Data Model (done)

---

## Technical Context

### Current State

`appendConversationMessage` 方法已支持 `ConversationMessage` 持久化：

```typescript
// runtimeStore.ts:2009-2037
public appendConversationMessage(
    projectRoot: string,
    conversationId: string,
    message: ConversationMessage
): { success: boolean }
```

**问题**：当前调用方（Chat/Agent/Run）只传入用户可见消息，不包含完整的工具调用消息。

### Target State

1. **调用方改造**：LLM 响应后，调用方需要：
   - 存储 `role=assistant` 消息（可能含 `tool_calls`）
   - 存储每个 `role=tool` 结果消息

2. **大输出处理**：新增辅助函数处理工具输出

```typescript
// runtimeStore.ts
const TOOL_OUTPUT_THRESHOLD = 50 * 1024 // 50KB

function prepareToolMessage(
  projectRoot: string,
  conversationId: string,
  message: ConversationMessage
): ConversationMessage {
  if (message.role !== 'tool' || !message.content) return message
  
  const contentBytes = Buffer.byteLength(message.content, 'utf-8')
  if (contentBytes <= TOOL_OUTPUT_THRESHOLD) return message
  
  // 写入完整输出到文件
  const outputPath = `@state/logs/tool-results/${conversationId}/${message.id}.json`
  writeToolOutput(projectRoot, outputPath, message.content)
  
  // 返回带预览的消息
  return {
    ...message,
    content: message.content.slice(0, 500) + `\n\n[Full output: ${outputPath}]`,
    fullOutputPath: outputPath
  }
}
```

---

## Files to Modify

| File | Action | Description |
|------|--------|-------------|
| `electron/stores/runtimeStore.ts` | MODIFY | 添加大输出处理逻辑 |
| `electron/services/chatHandler.ts` | MODIFY | 存储 assistant 和 tool 消息 |
| `electron/services/executionEngine.ts` | MODIFY | 存储 assistant 和 tool 消息 |
| `shared/conversationTypes.ts` | MODIFY | 添加 `fullOutputPath` 字段 |

---

## Testing Strategy

### Existing Tests
- `runtimeStore.test.ts` 已有 `appendConversationMessage` 测试

### New Unit Tests
```typescript
describe('appendConversationMessage with tool messages', () => {
  it('should persist assistant message with tool_calls', () => { ... })
  it('should persist tool result message', () => { ... })
  it('should handle large tool output by writing to file', () => { ... })
})
```

### Manual Verification
1. 启动应用，进入 Chat 模式
2. 触发工具调用（如文件读取）
3. 查看 `messages.json`，确认包含 `tool_calls` 和 `tool` 消息
4. 重启应用，验证消息历史完整

---

## Design Decisions

### 大输出阈值 50KB
**Rationale**: 平衡消息可读性和存储效率。50KB 足以容纳大多数工具输出，超过此大小通常是文件内容/日志。

### 预览长度 500 字符
**Rationale**: 足够提供上下文，不会过度膨胀消息列表。

### 存储路径 @state/logs/tool-results/
**Rationale**: 与现有日志目录结构一致，便于清理和管理。

---

## Code Review Notes ✅

### Findings
1. **Chat/Agent 未落地持久化回调**：`chatToolLoop` 已提供回调，但调用方未接入，导致 `assistant(tool_calls)` 与 `tool(result)` 未写入 `messages.json`。
2. **大输出处理未接入写入链路**：`prepareToolMessageForStorage` 未被调用，且 `fullOutputPath` 使用 runtime 相对路径，后续读取不统一。
3. **MCP 强制重试前置持久化**：`onAssistantMessage` 在强制 MCP 逻辑前触发，可能保存错误的 assistant 消息。
4. **LLM 错误未持久化**：缺少 `partType=error` 与 LLM_ERROR 块记录。

### Fixes Applied
- 在 `callAgentChat` 接入 `onAssistantMessage/onToolMessage`，仅持久化含 `tool_calls` 的 assistant 与所有 tool 结果。
- 在 `appendConversationMessage` 中统一调用 `prepareToolMessageForStorage`，并将 `fullOutputPath` 改为 `@state/logs/tool-results/...`。
- 将 `onAssistantMessage` 调整到 MCP 重试之后再触发。
- `MessagePartType` 增加 `error`，并在 LLM 错误时写入 `LLM_ERROR` 消息。

---

## Related Artifacts
- Architecture: `_bmad-output/architecture/unified-conversation-context.md`
- Epic Plan: `_bmad-output/implementation-artifacts/7-unified-conversation-context.md`
- Story 7-1: `_bmad-output/implementation-artifacts/7-1-extend-conversation-message-data-model.md`
