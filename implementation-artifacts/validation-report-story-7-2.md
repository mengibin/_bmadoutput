# Validation Report: Story 7-2

**Story**: 7-2 – Persist Tool-Call Protocol Messages  
**Validated**: 2026-01-18  
**Status**: ✅ **APPROVED**

---

## 1. Code Review Summary

### 1.1 现有持久化入口

| 位置 | 方法 | 状态 |
|------|------|------|
| `runtimeStore.ts:2009` | `appendConversationMessage()` | ✅ 已支持新 `ConversationMessage` 类型 |
| `appStore.ts:782` | `appendConversationMessage()` | ✅ 调用 IPC |
| `WorksPage.tsx` | 多处调用 | ⚠️ 需扩展以包含 `tool_calls` |

### 1.2 chatToolLoop.ts 分析

关键代码位置：

```typescript
// Line 403: 推送 assistant 消息到内存
params.messages.push(assistant)

// Line 525-529: 推送 tool 结果到内存
params.messages.push({
    role: 'tool',
    tool_call_id: call.id,
    content: JSON.stringify(result),
})
```

**发现**：`chatToolLoop` 正确将 `assistant(tool_calls)` 和 `tool` 消息添加到 `params.messages`，但这仅存在于内存中，未持久化。

### 1.3 前端调用分析

`WorksPage.tsx` 的 `appendConversationMessage` 调用：

```typescript
// Line 437: 仅传 role + content
appendConversationMessage(conversationId, { role: 'assistant', content: text })

// Line 767: 用户消息
appendConversationMessage(selectedConversation.id, { role: 'user', content: trimmed })
```

**发现**：当前调用只传递 `role` 和 `content`，不包含 `tool_calls` 或 `tool_call_id`。

---

## 2. 架构分析

### 2.1 消息流向

```
User Input → WorksPage.tsx
    ↓
IPC 'chat:send'
    ↓
chatToolLoop() → LLM → assistant(tool_calls) → tool execution → tool result
    ↓
onStreamEnd/onStreamToolEnd 回调
    ↓
WorksPage.tsx → appendConversationMessage() → runtimeStore
```

### 2.2 改造点

1. **后端**：`chatToolLoop` 需要在以下时机发出持久化事件：
   - `assistant` 消息推送时（含 `tool_calls`）
   - `tool` 结果推送时

2. **IPC**：新增/复用事件通道传递完整消息

3. **前端**：接收事件后调用 `appendConversationMessage` 持久化

---

## 3. Feasibility Assessment

### 3.1 技术可行性

| 改造点 | 复杂度 | 可行性 |
|--------|--------|--------|
| 新增 `onAssistantMessage` 回调 | Low | ✅ |
| 新增 `onToolMessage` 回调 | Low | ✅ |
| 前端接收并持久化 | Low | ✅ |
| 大输出文件存储 | Medium | ✅ |

### 3.2 兼容性

- ✅ `ConversationMessage` 已扩展（Story 7-1 完成）
- ✅ `appendConversationMessage` 已支持新字段
- ✅ 原子写入已实现（`writeJsonAtomic`）

---

## 4. 实现方案

### Option A: 扩展现有回调（推荐）

在 `chatToolLoop` 中新增回调：

```typescript
export async function chatToolLoop(params: {
    // ... existing
    onAssistantMessage?: (message: OpenAIChatMessage & { role: 'assistant' }) => void
    onToolMessage?: (message: OpenAIChatMessage & { role: 'tool' }) => void
})
```

调用点：
- Line 403 后调用 `onAssistantMessage`
- Line 529 后调用 `onToolMessage`

### Option B: 返回完整消息历史

让 `chatToolLoop` 返回所有新增消息，由调用方批量持久化。

**推荐 Option A**：更细粒度，便于流式 UI 更新。

---

## 5. Risk Assessment

| 风险 | 等级 | 缓解措施 |
|------|------|---------|
| 消息丢失（crash） | Low | 原子写入已实现 |
| 重复持久化 | Low | 使用 message ID 去重 |
| 大输出内存溢出 | Low | 流式写入文件 |

---

## 6. Verdict

✅ **Story 7-2 可以开始实现**

- 技术方案可行
- 改造点明确
- 风险可控

---

## Implementation Checklist

- [ ] 在 `chatToolLoop` 添加 `onAssistantMessage` / `onToolMessage` 回调
- [ ] 在 `executionEngine` 中实现回调处理
- [ ] 添加大输出处理逻辑
- [ ] 更新 `WorksPage.tsx` Chat 模式调用
- [ ] 添加单元测试

---

## Related Files
- `electron/services/chatToolLoop.ts`
- `electron/services/executionEngine.ts`
- `electron/stores/runtimeStore.ts`
- `src/pages/WorksPage/WorksPage.tsx`
- `shared/conversationTypes.ts`
