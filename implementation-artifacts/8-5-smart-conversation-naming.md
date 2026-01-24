# Story 8.5: Smart Conversation Naming

## 概述

使用 LLM 根据对话内容自动生成有意义的会话名称，解决多个 Conversation 名称相同难以区分的问题。

---

## 用户故事

As a **Consumer**,
I want conversations to be automatically named based on content,
So that I can easily identify and find my conversation history.

---

## 问题描述

当前多个 Workflow 对话显示相同的截断名称（如 "Workflow: Supply Chain Deci..."），难以区分。

---

## 验收标准

### AC-1: 自动生成标题

**Given** I start a new conversation and send my first message
**When** the first assistant reply completes
**Then** the system uses LLM to generate a short, descriptive title (≤30 chars)
**And** the title reflects the key topic or intent of the conversation
**And** replaces the default "Workflow: xxx" or "New Conversation" name

### AC-2: 手动重命名

**Given** I want to manually rename a conversation
**When** I click on the conversation name
**Then** I can edit it inline
**And** the custom name overrides the auto-generated name

### AC-3: 完整名称 Tooltip

**Given** I hover over a truncated conversation name
**When** the tooltip appears
**Then** I see the full conversation title

---

## 智能命名示例

| 用户第一条消息 | 生成标题 |
|:--------------|:---------|
| "帮我分析这个供应链决策..." | "供应链决策分析" |
| "设计一个V型带轮..." | "V带轮设计计算" |
| "报销流程的合规检查" | "报销合规检查" |
| "Create user story for login" | "Login Feature Story" |

---

## 技术设计

### 1. ConversationMetadata 扩展

```typescript
interface ConversationMetadata {
    id: string
    createdAt: string
    updatedAt: string
    agentId?: string
    workflowId?: string
    runCount?: number
    
    // NEW: Smart naming fields
    generatedTitle?: string  // LLM 生成的标题
    customTitle?: string     // 用户自定义标题
}

// Display logic
function getDisplayTitle(conv: ConversationMetadata): string {
    return conv.customTitle || conv.generatedTitle || conv.workflowId || 'New Conversation'
}
```

### 2. 标题生成 Prompt

```
Generate a short title (max 30 chars) for this conversation based on the user's first message:

User: {first_user_message}

Requirements:
- Concise and descriptive
- Capture the main topic/intent
- No quotes or special characters
- If Chinese, output Chinese title
- If English, output English title

Output only the title, nothing else.
```

### 3. 生成时机

```typescript
// In ExecutionEngine or LlmAdapter
async function handleFirstAssistantReply(conversationId: string, userMessage: string) {
    // ... existing logic to get assistant reply ...
    
    // Generate title after first exchange
    const title = await generateConversationTitle(userMessage)
    
    // Update conversation metadata
    runtimeStore.updateConversation(projectRoot, conversationId, {
        generatedTitle: title
    })
}
```

### 4. UI 修改

```tsx
// ConversationList.tsx
<div 
    className="conversation-item"
    title={getDisplayTitle(conversation)}  // Full name tooltip
>
    <span className="conversation-title">
        {truncate(getDisplayTitle(conversation), 30)}
    </span>
    <EditButton onClick={() => startRename(conversation.id)} />
</div>
```

---

## 实现步骤

1. 扩展 `ConversationMetadata` 接口，添加 `generatedTitle` 和 `customTitle` 字段
2. 创建 `generateConversationTitle()` 函数，使用 LLM 生成标题
3. 在第一次对话交换后调用标题生成
4. 修改 `ConversationList` UI，显示生成/自定义标题
5. 添加内联重命名功能
6. 添加 Tooltip 显示完整名称

---

## 影响分析

| 组件 | 变更 | 风险 |
|:-----|:-----|:-----|
| `ConversationMetadata` | 新增字段 | 低 |
| LLM 调用 | 额外一次轻量调用 | 低（可用快速模型） |
| Conversation list UI | 显示逻辑修改 | 低 |
| RuntimeStore | 更新 conversation 方法 | 低 |

---

## Dev Agent Record

### Agent Model Used
GPT-5.2 (Codex CLI)

### Debug Log References
- `crewagent-runtime`: `npm -C crewagent-runtime test`

### Completion Notes List
- 对话首次回复后触发自动标题生成，结果清理/截断后写入并触发前端刷新。
- 支持会话标题点击内联重命名，并统一显示 generated/custom 标题与完整 tooltip。

### File List
- `crewagent-runtime/src/stores/appStore.ts`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/preload.ts`
- `crewagent-runtime/electron/electron-env.d.ts`
- `crewagent-runtime/electron/services/chatToolLoop.ts`
- `crewagent-runtime/electron/services/chatToolLoop.test.ts`
- `crewagent-runtime/src/pages/WorksPage/WorksPage.tsx`
- `crewagent-runtime/src/vite-env.d.ts`
