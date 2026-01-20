# Story 7-3: Create Context Builder Module

## Overview
**Epic**: 7 – Unified Conversation Context  
**Priority**: Phase 2 (depends on 7.1, 7.2)  
**Status**: `done`

## Goal
创建独立的上下文构建模块 `context-builder/`，供 Chat / Agent / Run 三种模式统一调用。将持久化的 `ConversationMessage[]` 转换为 OpenAI `messages[]` 格式。

## Business Value
- **代码复用**：三种模式共用一套构建逻辑
- **易维护**：逻辑集中、可独立测试
- **灵活性**：支持 System Prompt 动态拼接

## Acceptance Criteria
1. **模块结构**：
   ```
   electron/core/context-builder/
   ├── index.ts
   ├── types.ts
   ├── buildLlmMessages.ts
   ├── systemPromptComposer.ts
   └── context-builder.test.ts
   ```
2. **Public API**：
   ```typescript
   export function buildLlmMessagesFromConversation(
     options: ContextBuildOptions
   ): ContextBuildResult
   ```
3. **输入**：`messages`, `mode`, `agentDefinition`, `runContext`, `settings`
4. **输出**：`OpenAIChatMessage[]` + debug metadata
5. **System Prompt 处理**：
   - 不持久化到历史
   - 根据 mode 动态拼接（base rules + tool policy + persona）
   - 添加 Mode Banner

## Tasks / Subtasks

### Review Follow-ups (AI)
- [x] [AI-Review][HIGH] Ensure `maxMessages` trimming preserves assistant tool_calls + tool result groups (or disable trimming until grouping is implemented) [crewagent-runtime/electron/core/context-builder/buildLlmMessages.ts:85]
- [x] [AI-Review][MEDIUM] Do not drop all `role: system` messages; only exclude generated system prompt or honor `includeInContext` for system summaries [crewagent-runtime/electron/core/context-builder/buildLlmMessages.ts:75]
- [x] [AI-Review][MEDIUM] Align public API types with Story AC (`agentDefinition`, `maxTokens`, metadata field names) or update the story/spec to match implementation [crewagent-runtime/electron/core/context-builder/types.ts:70]
- [x] [AI-Review][MEDIUM] Persist tool messages with `partType: 'tool_result'` (or render `role: 'tool'` in UI) so tool results aren't shown as raw JSON assistant text [crewagent-runtime/electron/main.ts:1519]
- [x] [AI-Review][MEDIUM] Add/update Dev Agent Record File List in this story to include actual changed files in the runtime changeset [_bmad-output/implementation-artifacts/7-3-create-context-builder-module.md:1]
- [x] [AI-Review][LOW] Move mid-file import block to the top to satisfy `import/first` if enforced [crewagent-runtime/electron/stores/runtimeStore.ts:550]

## Out of Scope
- Context Compression（Story 7.4）
- 模式集成（Story 7.5-7.7）

## Dependencies
- **Story 7-1**: Extend ConversationMessage Data Model (done)
- **Story 7-2**: Persist Tool-Call Protocol Messages (done)

---

## Technical Context

### Current State

目前各模式分散构建 messages：
- Chat: 在 `chatToolLoop` 调用前手动构建
- Agent: 在 `AgentHandler` 中处理
- Run: 在 `ExecutionEngine.session.history` 中管理

需要统一到 `context-builder` 模块。

### Target State

```typescript
// electron/core/context-builder/types.ts
export interface ContextBuildOptions {
  messages: ConversationMessage[]
  mode: 'chat' | 'agent' | 'run'
  agentDefinition?: AgentDefinition
  runContext?: {
    packageName: string
    workflowName: string
    currentStepName: string
    stepInstruction: string
  }
  settings?: {
    maxTokens?: number
    includeSystemPrompt?: boolean
  }
}

export interface ContextBuildResult {
  messages: OpenAIChatMessage[]
  metadata: {
    totalMessages: number
    includedMessages: number
    estimatedTokens: number
    systemPromptIncluded: boolean
  }
}
```

```typescript
// electron/core/context-builder/buildLlmMessages.ts
export function buildLlmMessagesFromConversation(
  options: ContextBuildOptions
): ContextBuildResult {
  const result: OpenAIChatMessage[] = []
  
  // 1. Add system prompt (if not in history)
  if (options.settings?.includeSystemPrompt !== false) {
    const systemPrompt = composeSystemPrompt(options)
    result.push({ role: 'system', content: systemPrompt })
  }
  
  // 2. Convert messages
  for (const msg of options.messages) {
    if (msg.includeInContext === false) continue
    result.push(toOpenAIMessage(msg))
  }
  
  return { messages: result, metadata: { ... } }
}
```

```typescript
// electron/core/context-builder/systemPromptComposer.ts
export function composeSystemPrompt(options: ContextBuildOptions): string {
  const parts: string[] = []
  
  // Mode banner
  parts.push(`## Mode: ${options.mode.toUpperCase()}`)
  
  // Base rules
  parts.push(BASE_RULES)
  
  // Agent persona
  if (options.mode === 'agent' && options.agentDefinition) {
    parts.push(formatAgentPersona(options.agentDefinition))
  }
  
  // Run context
  if (options.mode === 'run' && options.runContext) {
    parts.push(formatRunContext(options.runContext))
  }
  
  return parts.join('\n\n')
}
```

---

## Files to Create

| File | Description |
|------|-------------|
| `electron/core/context-builder/index.ts` | 模块入口，导出 public API |
| `electron/core/context-builder/types.ts` | 类型定义 |
| `electron/core/context-builder/buildLlmMessages.ts` | 核心转换函数 |
| `electron/core/context-builder/systemPromptComposer.ts` | System Prompt 拼接 |
| `electron/core/context-builder/context-builder.test.ts` | 单元测试 |

---

## Testing Strategy

### Unit Tests
```typescript
describe('buildLlmMessagesFromConversation', () => {
  it('should convert ConversationMessage[] to OpenAIChatMessage[]', () => { ... })
  it('should filter out messages with includeInContext=false', () => { ... })
  it('should prepend system prompt by default', () => { ... })
  it('should skip system prompt when settings.includeSystemPrompt=false', () => { ... })
})

describe('composeSystemPrompt', () => {
  it('should include mode banner', () => { ... })
  it('should include agent persona for agent mode', () => { ... })
  it('should include run context for run mode', () => { ... })
})
```

### Verification Command
```bash
cd crewagent-runtime && npm test -- electron/core/context-builder/context-builder.test.ts
```

---

## Related Artifacts
- Architecture: `_bmad-output/architecture/unified-conversation-context.md`
- Epic Plan: `_bmad-output/implementation-artifacts/7-unified-conversation-context.md`
- Story 7-1: Done
- Story 7-2: Done

## Dev Agent Record

### Agent Model Used
GPT-5 (Codex CLI)

### Debug Log References
- `crewagent-runtime`: `npm test -- electron/core/context-builder/context-builder.test.ts`

### Completion Notes List
- Added tool-call-safe trimming to avoid breaking assistant/tool protocol when `maxMessages` is set.
- Kept historical system messages (e.g., summaries) while still composing a fresh system prompt.
- Aligned public API naming/metadata with Story 7-3 while retaining backward compatibility.
- Persisted tool result messages with `partType: tool_result` for proper UI rendering.
- Moved shared conversation types import to top-level to satisfy import ordering.
- Ran context-builder unit tests (npm reported user config warnings for `python`/`install`).

### File List
- crewagent-runtime/electron/core/context-builder/index.ts
- crewagent-runtime/electron/core/context-builder/types.ts
- crewagent-runtime/electron/core/context-builder/buildLlmMessages.ts
- crewagent-runtime/electron/core/context-builder/systemPromptComposer.ts
- crewagent-runtime/electron/core/context-builder/context-builder.test.ts
- crewagent-runtime/shared/conversationTypes.ts
- crewagent-runtime/src/shared/conversationTypes.test.ts
- crewagent-runtime/electron/main.ts
- crewagent-runtime/electron/services/chatToolLoop.ts
- crewagent-runtime/electron/services/chatToolLoop.test.ts
- crewagent-runtime/electron/services/openaiProtocol.ts
- crewagent-runtime/electron/stores/runtimeStore.ts
- crewagent-runtime/electron/stores/runtimeStore.test.ts
- crewagent-runtime/src/pages/RunsPage/components/MessageItem.tsx
- crewagent-runtime/src/stores/appStore.ts

## Change Log
- 2026-01-18: Addressed code review findings; updated context builder behavior, tool persistence UI metadata, and story documentation.
- 2026-01-18: Ran context builder tests and marked story done.
