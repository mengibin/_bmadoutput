# Story 7-1: Extend ConversationMessage Data Model (Protocol-Complete)

## Overview
**Epic**: 7 – Unified Conversation Context  
**Priority**: Phase 1  
**Status**: `done`

## Goal
扩展 `ConversationMessage` 类型以完整支持 OpenAI 工具调用协议，包括 `role=tool`、`tool_calls`、`tool_call_id` 等字段，使存储格式可直接转换为 OpenAI `messages[]`。

## Business Value
- **协议完整性**：支持完整的 OpenAI tool-call 协议 (`assistant(tool_calls)` → `tool(result)`)
- **可调试性**：完整保留工具调用链，便于回溯和恢复
- **上下文连续性**：为后续 Context Builder 提供完整的消息数据

## Acceptance Criteria
1. **Role 扩展**：`ConversationMessageRole` 类型增加 `'tool'` 值
2. **工具调用字段**：
   - `tool_calls?: OpenAIToolCall[]` - assistant 发起的工具调用
   - `tool_call_id?: string` - tool 消息关联的调用 ID
3. **上下文元数据**：
   - `mode?: 'chat' | 'agent' | 'run'` - 消息产生时的模式
   - `runId?: string` - 关联的 Run ID（区分多次运行）
   - `includeInContext?: boolean` - 是否纳入 LLM 上下文（默认 true）
4. **向后兼容**：
   - 旧消息加载时缺失字段使用安全默认值
   - 不破坏现有 UI 渲染逻辑
5. **类型共享**：新类型从 `shared/conversationTypes.ts` 导出供前后端共用

## Out of Scope
- 持久化逻辑改动（Story 7.2）
- Context Builder 实现（Story 7.3）

## Dependencies
- **Story 5-8**: Persist Conversations & Messages (done)

---

## Validation Notes ✅

**Validated**: 2026-01-18

### 代码审查结果

1. **类型定义位置**：
   - `src/stores/appStore.ts:154-164` - `ConversationMessage` ✅
   - `electron/stores/runtimeStore.ts:606-616` - `ConversationMessage` ✅（重复定义）
   - **发现**：两处重复定义，需统一到 `shared/`

2. **shared/ 目录现状**：
   ```
   shared/
   ├── agentToolPolicy.ts
   └── logTypes.ts
   ```
   ✅ 已有类型共享先例

3. **类型引用分析**：
   - `WorksPage.tsx` - import from appStore
   - `RunWorkspace.tsx` - import from appStore
   - `ChatPanel.tsx` - import from appStore
   - `MessageList.tsx` - import from appStore
   - `MessageItem.tsx` - import from appStore
   ✅ 所有前端组件都从 `appStore` 导入

4. **向后兼容性**：✅ 所有新字段都是可选的

5. **Verdict**: ✅ APPROVED

---

## Technical Design

### Current State
```typescript
// appStore.ts:11
export type ConversationMessageRole = 'user' | 'assistant' | 'system'

// appStore.ts:154-164, runtimeStore.ts:606-616 (重复)
export interface ConversationMessage {
  id: string
  role: ConversationMessageRole
  content: string
  createdAt: string
  partType?: MessagePartType
  toolName?: string
  duration?: number
  isCollapsed?: boolean
  widget?: WidgetPayload
}
```

### Target State
```typescript
// shared/conversationTypes.ts (新文件)
export type ConversationMessageRole = 'user' | 'assistant' | 'system' | 'tool'

export interface OpenAIToolCall {
  id: string
  type: 'function'
  function: {
    name: string
    arguments: string
  }
}

export interface ConversationMessage {
  id: string
  role: ConversationMessageRole
  createdAt: string

  // OpenAI-compatible payload
  content?: string | null
  tool_calls?: OpenAIToolCall[]          // assistant only
  tool_call_id?: string                  // tool only

  // UI extras (existing)
  partType?: MessagePartType
  toolName?: string
  duration?: number
  isCollapsed?: boolean
  widget?: WidgetPayload

  // Context metadata (new)
  mode?: 'chat' | 'agent' | 'run'
  runId?: string
  workflowId?: string
  agentId?: string
  includeInContext?: boolean
}
```

---

## Implementation Plan

### Step 1: Create shared/conversationTypes.ts

创建统一类型定义文件：

```typescript
// crewagent-runtime/shared/conversationTypes.ts

export type MessagePartType = 'content' | 'thinking' | 'tool_start' | 'tool_result'
export type WidgetType = 'form' | 'selection' | 'confirmation' | 'plan_review' | 'feedback' | 'action_selection'

// ... 复制 WidgetPayload 相关类型 ...

export type ConversationMessageRole = 'user' | 'assistant' | 'system' | 'tool'

export interface OpenAIToolCall {
  id: string
  type: 'function'
  function: {
    name: string
    arguments: string
  }
}

export interface ConversationMessage {
  // ... 完整定义 ...
}
```

### Step 2: Update appStore.ts

```typescript
// 删除本地定义，改为导入
import type { 
  ConversationMessage, 
  ConversationMessageRole,
  MessagePartType,
  WidgetPayload,
  // ...
} from '../../shared/conversationTypes'

// 重新导出供组件使用
export type { ConversationMessage, ConversationMessageRole, MessagePartType, WidgetPayload }
```

### Step 3: Update runtimeStore.ts

```typescript
// 删除本地定义，改为导入
import type { 
  ConversationMessage, 
  MessagePartType,
  WidgetPayload,
  // ...
} from '../../shared/conversationTypes'
```

### Step 4: Verify Imports

批量检查并更新组件导入路径（如需要）。

---

## Files to Create/Modify

| File | Action | Lines | Description |
|------|--------|-------|-------------|
| `shared/conversationTypes.ts` | CREATE | ~80 | 统一的类型定义 |
| `src/stores/appStore.ts` | MODIFY | ~20 | 删除本地类型，导入 shared |
| `electron/stores/runtimeStore.ts` | MODIFY | ~15 | 删除本地类型，导入 shared |

---

## Testing Strategy

### Automated Tests
1. **TypeScript 编译检查**
   ```bash
   cd crewagent-runtime && npm run build
   ```

### Manual Verification
1. 启动应用：`npm run dev`
2. 打开已有项目，验证历史消息正常显示
3. 发送新消息，验证 UI 正常
4. 检查 DevTools 无类型错误

---

## Design Decisions

### 统一类型定义
**Rationale**: 当前 `ConversationMessage` 在 `appStore.ts` 和 `runtimeStore.ts` 中重复定义，容易不同步。创建 `shared/conversationTypes.ts` 作为单一来源。

### content 改为可选
**Rationale**: OpenAI 协议中 `assistant` 消息可以 `content=null` 而只有 `tool_calls`。需要允许 `content` 为空。

### includeInContext 默认 true
**Rationale**: 绝大多数消息都需要纳入上下文，只有特殊 UI 事件（如 widget 交互）才需要标记为 false。

### role='tool' UI 处理
**Decision**: `role='tool'` 消息在 UI 中复用 `assistant` 样式，或默认折叠显示。具体 UI 改动可在后续 Story 中处理。

---

## Dev Agent Record / File List
- `crewagent-runtime/shared/conversationTypes.ts`
- `crewagent-runtime/src/stores/appStore.ts`
- `crewagent-runtime/electron/stores/runtimeStore.ts`

---

## Related Artifacts
- Architecture: `_bmad-output/architecture/unified-conversation-context.md`
- Epic Plan: `_bmad-output/implementation-artifacts/7-unified-conversation-context.md`
- Validation Report: `_bmad-output/implementation-artifacts/validation-report-story-7-1.md`
