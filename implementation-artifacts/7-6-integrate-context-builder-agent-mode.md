# Story 7-6: Integrate Context Builder into Agent Mode

## Overview
**Epic**: 7 – Unified Conversation Context  
**Priority**: Phase 3 (depends on 7.5)  
**Status**: `validated`

## Goal
将 Agent 模式改为使用统一的 Context Builder，包含 Persona 注入。由于 Agent 和 Chat 模式共用 `callAgentChat` 函数，Story 7-5 完成后本 Story 的大部分工作已自动完成。

## Business Value
- **上下文连续性**：Agent 模式也支持多轮对话
- **Persona 一致性**：统一使用 context-builder 的 Persona 构建
- **模式区分**：消息带 `mode='agent'` 标记

## Acceptance Criteria
1. **同 Chat 模式**：历史加载、上下文构建、消息持久化
2. **Persona 注入**：`buildLlmMessagesFromConversation({ mode: 'agent', agent: ... })` 自动包含 persona
3. **模式标识**：消息带 `mode='agent'` 标记
4. **验证**：Agent 模式多轮对话正常工作

## Out of Scope
- Run 模式集成（Story 7.7）

## Dependencies
- **Story 7-5**: Integrate Context Builder into Chat Mode

---

## Technical Context

### 当前架构分析

`callAgentChat` 函数同时服务于 Chat 和 Agent 两种模式：

```typescript
const callAgentChat = async (params: {
    // ...
    mode?: 'agent' | 'chat'  // 模式区分
    // ...
}) => {
    const mode = params.mode === 'chat' ? 'chat' : 'agent'
    // ...
}
```

### Story 7-5 完成后的状态

如果 Story 7-5 已正确实现，Agent 模式应该已经能够：
- ✅ 加载对话历史
- ✅ 使用 context-builder 构建上下文
- ✅ 持久化消息（带 `mode='agent'` 标记）
- ✅ 压缩支持

### 需要验证的点

| 验证点 | 预期行为 |
|--------|---------|
| Persona 注入 | System Prompt 包含 Agent 的 role/identity/principles |
| 模式标识 | 消息的 `mode` 字段为 `'agent'` |
| 多轮对话 | LLM 能引用之前的对话内容 |
| agent:dispatch | 通过 `agent:dispatch` IPC 调用时也能正常工作 |

---

## 实现方案

### 方案 A：无需额外代码改动（推荐）

如果 Story 7-5 的实现已经正确处理了 `mode` 参数，本 Story 只需：
1. 验证 Agent 模式工作正常
2. 更新文档

### 方案 B：补充改动（如需要）

检查以下代码是否正确处理 Agent 模式：

```typescript
// callAgentChat 中
const { messages } = buildLlmMessagesFromConversation({
    messages: history,
    mode,  // 'agent' 或 'chat'
    agent: {
        id: agentDefinition.id,
        name: agentDefinition.metadata?.name ?? agentDefinition.id,
        role: agentDefinition.persona?.role ?? 'Assistant',
        identity: agentDefinition.persona?.identity,
        communicationStyle: agentDefinition.persona?.communication_style,
        principles: agentDefinition.persona?.principles,
        systemPrompt: agentDefinition.systemPrompt,
    },
    // ...
})
```

---

## 验证策略

### 手动测试
1. 打开应用，选择一个 Agent
2. 发送多轮消息，验证 Agent 能引用之前的内容
3. 验证 System Prompt 包含 Agent Persona
4. 检查 `messages.json`，确认 `mode='agent'`

### 验证命令
```bash
# 查看消息日志
cat <projectRoot>/RuntimeStore/state/conversations/<conversationId>/messages.json | jq '.[] | select(.mode == "agent")'
```

---

## Files to Verify/Modify

| File | Action |
|------|--------|
| `electron/main.ts` | 验证 `callAgentChat` 正确处理 mode='agent' |
| `electron/core/context-builder/systemPromptComposer.ts` | 验证 Persona 构建逻辑 |

---

## Related Artifacts
- **Design Document**: [design-7-6-integrate-context-builder-agent-mode.md](./design-7-6-integrate-context-builder-agent-mode.md)
- **Validation Report**: [validation-report-story-7-6.md](./validation-report-story-7-6.md)
- Architecture: `_bmad-output/architecture/unified-conversation-context.md`
- Epic Plan: `_bmad-output/implementation-artifacts/7-unified-conversation-context.md`
- Story 7-5: Chat Mode Integration (done)
