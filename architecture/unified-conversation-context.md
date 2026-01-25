# Unified Conversation Context

> Architecture reference: CrewAgent conversation / context system design (Codex-style approach).

## 1. Summary

This spec proposes a **Codex-style** approach to conversation handling:

1) **Store everything** in a single Conversation log (user-visible chat + internal LLM/tool/execution events + errors/partial output).
2) For every LLM call (Chat / Agent / Run), build a request from:
   - **Dynamic System Prompt** (depends on current mode)
   - **Conversation context window** (selected from the unified log with strict budget rules)
   - **Run-only injected context**:
     - workflow 未完成（active step 存在）：RUN_DIRECTIVE / NODE_BRIEF / step context
     - workflow 已 completed（Post-Completion Profile）：仅 RUN_DIRECTIVE（含 Post-Run Protocol），不注入 step/node 信息

Mode switching (Run / Agent / Chat) **does not reset history**; it only changes the system prefix used to build the next request.

---

## 2. Goals / Non-Goals

### Goals
- Single source of truth for all conversation-related records:
  - UI-visible messages
  - LLM assistant messages (including tool calls)
  - Tool results (`role=tool`)
  - LLM failures (timeouts/network), including partial streamed output
- Every LLM call uses **conversation history** (within a budget) so “继续/continue” has an anchor.
- Switching modes updates only the **system prompt**, not the message log.
- Preserve OpenAI tool-call protocol ordering (`assistant(tool_calls)` → `tool(result)` → `assistant(content)`).

### Non-Goals (MVP)
- Perfect long-term memory with unlimited context; budget enforcement is required.
- Cross-conversation memory.
- Full autonomous summarization pipeline (can be added in v2 as optional).

---

## 3. Current State (Observed)

### 3.1 Storage
- Conversations and messages are persisted per project:
  - `.../conversations/index.json`
  - `.../conversations/<conversationId>/messages.json`
- Stored messages are primarily user-visible (`role: user/assistant/system`), not OpenAI protocol-complete.

### 3.2 Context Building
- **Chat mode**: `chat:send` builds base prompt + current input only; does **not** include prior conversation messages.
- **Agent mode**: similar; not consistently contextual across turns.
- **Run mode**: `ExecutionEngine` uses an in-memory `session.history` and not the persisted conversation messages.

Result: failures/partial output may not appear in the UI message timeline; “continue” can be ambiguous after errors.

---

## 4. Proposed Design

### 4.1 Data Model: Persist a Protocol-Complete Conversation Log

#### Key idea
Persist entries that can be deterministically converted to OpenAI `messages[]` **without losing tool-call structure**.

#### Option A (Recommended): Extend `ConversationMessage` to be a superset of `OpenAIChatMessage`
Extend `ConversationMessage.role` to include `tool`, and add the fields required by tool-call protocol:

```ts
type ConversationRole = 'system' | 'user' | 'assistant' | 'tool'

interface ConversationMessage {
  id: string
  role: ConversationRole
  createdAt: string

  // OpenAI-compatible payload (protocol-complete)
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
  includeInContext?: boolean             // default true; allows UI-only events
}
```

Back-compat: existing messages without these new fields load as-is; missing fields default safely.

#### Option B: Introduce `ConversationEvent[]` (More flexible, larger refactor)
Store typed events (`user_message`, `assistant_message`, `tool_result`, `mode_switch`, `llm_error`) and generate UI messages + LLM messages from that. This is more robust but touches more code.

MVP recommendation: **Option A**.

---

### 4.2 Context Builder Module (Core)

> **设计原则**：Context Builder 应作为**独立模块**实现，与 Chat / Agent / Run 三种模式解耦，便于维护和单元测试。

#### 模块结构

```
src/core/context-builder/
├── index.ts                      # 主入口 & public API
├── types.ts                      # 类型定义
├── buildLlmMessages.ts           # 核心构建逻辑
├── budgetTrimmer.ts              # 预算裁剪策略
├── toolOutputFormatter.ts        # 工具输出格式化（preview + path）
├── systemPromptComposer.ts       # System Prompt 拼装
└── __tests__/                    # 单元测试
```

#### Public API

```typescript
// 主函数：供 Chat / Agent / Run 统一调用
export function buildLlmMessagesFromConversation(options: ContextBuildOptions): ContextBuildResult
```

**Inputs**
- `projectRoot`, `packageId`, `conversationId`
- `mode`: `'chat' | 'agent' | 'run'`
- `agentDefinition` + merged tool policy
- `runContext` when mode=`run` (workflowId/runId/current node info)
- `settings`: context budgets

**Outputs**
- `OpenAIChatMessage[]` to send to `llmAdapter.chatCompletions()`
- debug metadata: message counts, estimated tokens, which items were kept/dropped

---

### 4.3 Context Compression Module (独立模块)

> **设计原则**：上下文压缩是未来需要大量优化的核心模块，必须保证**高扩展性**和**完全独立性**，采用**策略模式 (Strategy Pattern)** 设计。

#### 模块结构

```
src/core/context-compression/
├── index.ts                        # Public API
├── types.ts                        # 策略接口 & 类型定义
├── CompressionPipeline.ts          # 压缩管道（阈值监控 + 策略执行）
├── ContextUsageMonitor.ts          # 上下文占用监控（计算剩余量）
├── strategies/
│   ├── KeyMessageExtractionStrategy.ts  # v1: 关键消息提取（无 LLM）
│   ├── LlmSummaryStrategy.ts            # v2: LLM 辅助摘要
│   └── VectorRetrievalStrategy.ts       # v2+: 向量检索（RAG-style）
└── __tests__/
```

#### 压缩模式：Claude Code 风格（阈值触发）

采用 Claude Code / Codex 的设计模式：

```
┌─────────────────────────────────────────────────────┐
│              Context Budget (100%)                  │
├─────────────────────────────────────────────────────┤
│ ████████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░ │
│ ← 已使用 (65%)                      剩余 (35%) →   │
│                                                     │
│ 当已使用 > 80% 时 → 触发压缩                        │
└─────────────────────────────────────────────────────┘
```

**核心接口**：

```typescript
interface ContextUsage {
  usedTokens: number
  totalBudget: number
  usagePercent: number      // 0-100
  remaining: number
}

interface CompressionConfig {
  // 阈值触发（Claude Code 风格）
  triggerThreshold: number   // e.g. 0.8 (当占用超过 80% 时触发)
  targetAfterCompression: number  // e.g. 0.5 (压缩后目标占用 50%)
  
  // 预算设置
  totalBudget: number        // 总 token 预算
  reservedForResponse: number  // 为 LLM 响应预留
}
```

**触发流程**：

```
每次构建 LLM 请求前
    ↓
计算当前 context usage
    ↓
if (usagePercent > triggerThreshold) {
    执行压缩策略
    - v1: 关键消息提取
    - v2: LLM 摘要
}
    ↓
构建并发送请求
```

#### v1 实现：KeyMessageExtractionStrategy（无 LLM）

不调用 LLM，通过**可解释打分 + 预算内贪心选择**提取关键消息，并在超预算时启用兜底降级：

```typescript
function extractKeyMessages(messages: Message[]): Message[] {
  // 1) 基础规则：标记“必保留”
  const mandatory = markMandatory(messages) // system / recent / key events / tool groups / latest user

  // 2) 评分：给剩余消息打分（可解释）
  const scored = score(messages, {
    keywords: ['must', 'should', 'need', '必须', '需要'],
    hasFilePath: true,
    hasCodeBlock: true,
    hasNumbers: true,
    recencyDecay: true,
    penalizeLargeToolResults: true,
  })

  // 3) 贪心选取：在预算内按分数挑选（保留完整 tool 组）
  const selected = greedySelect(scored, mandatory, budget)

  // 4) 兜底：若 mandatory + selected 仍超预算，则降级
  return fallbackTrim(selected, {
    toolResultPreview: true,
    truncateLongUser: true,
    truncateLongAssistant: true,
    // 极端兜底：仅保留最近 K 回合（K=4）
    keepRecentRounds: 4,
  })
}
```

**保留规则（常规）**：
- ✅ 所有用户消息（用户意图）
- ✅ 产出/完成记录（关键决策）
- ✅ 完整的工具调用组（协议完整性）
- ✅ 最近 N 条消息（即时上下文）
- ❌ 历史 system 消息（默认不保留，避免重复）
- ✅ 仅保留 **SUMMARY 类 system 消息**（v2 用于摘要，例如以 `SUMMARY` / `CONVERSATION_SUMMARY` 开头）
- ❌ 中间的冗长对话/调试信息

**超预算兜底（解决 111%→111%）**：
- 裁剪 tool 结果为 preview（保留 tool name + status + 前后 N 行/字符）。
- 裁剪超长 user / assistant 文本（保留首段 + 关键 bullet 行 / 首句或“结论/summary”句）。
- 若仍超预算：**仅保留最近 K 回合（K=4）**。
  - 回合定义：以 user 消息为界，保留该 user 之后到下一个 user 之前的 assistant 内容。
  - 若保留回合内出现 `tool_calls`，**必须同时保留对应 tool 结果**，保持协议完整性。

#### v2 扩展：LlmSummaryStrategy

v2 在关键消息提取基础上，对被删除的消息生成 LLM 摘要：

```typescript
async function summarize(droppedMessages: Message[]): Promise<string> {
  const response = await llm.chat({
    model: 'gpt-3.5-turbo',  // 使用低成本模型
    messages: [
      { role: 'system', content: SUMMARY_PROMPT },
      { role: 'user', content: formatMessages(droppedMessages) }
    ]
  })
  return response.content  // 约 200-500 字的摘要
}
```

摘要作为一条 `role=system` 消息插入压缩后的上下文开头。

#### 演进路径

| 版本 | 策略 | LLM | 说明 |
|------|------|-----|------|
| **v1** | `KeyMessageExtractionStrategy` | ❌ | 规则提取关键消息 |
| v2 | `LlmSummaryStrategy` | ✅ | 摘要被删除的内容 |
| v2+ | `VectorRetrievalStrategy` | ✅ | 根据意图检索相关历史 |
| v2+ | `HybridStrategy` | ✅ | 组合多策略 |

#### UI 集成（可选）

提供剩余量指示器，类似 Claude Code：

```typescript
// 前端可订阅
interface ContextStatusEvent {
  usagePercent: number
  status: 'normal' | 'warning' | 'compressing'
}
```

**Tool output strategy (Chosen)**
- Persist full tool results for debugging and replay, but **do not** inject full payloads into every LLM request.
- Inject **preview + “read logs/path”** into context:
  - Include a short preview (e.g. first N chars / structured summary).
  - Include an **alias path** under `@state/` or `@project/` where the full output was saved (so the LLM can `fs.read` it on demand).
  - Example (conceptual):
    - `TOOL_RESULT_PREVIEW ...`
    - `Full output: @state/logs/tool-results/<toolCallId>.json`

Note: Conversation files under RuntimeStore are not directly readable by tools via `@state`; “read logs/path” refers to files we intentionally write under `@state`/`@project`.

**System prompt handling**
- System prompt is **not** persisted in history (avoid conflicts).
- On every call, prepend:
  - `system-base-rules-(chat|run).md`
  - `system-tool-policy.md`
  - persona (agent mode + run mode only; omit in chat mode)
  - a short “Mode banner” system message:
    - `MODE\n- active: run|agent|chat\n- note: history may include other modes; follow current instructions.`
 - **Compression rule**: historical `role=system` messages are ignored by default, except for **summary-class system messages** (e.g., content starts with `SUMMARY` / `CONVERSATION_SUMMARY`) introduced in v2.

**Run-only injection**
- For mode=`run`, prepend deterministic run directive messages:
  - workflow 未完成（active step 存在）：RUN_DIRECTIVE + NODE_BRIEF（可选：step context）
  - workflow 已 completed（Post-Completion Profile）：仅 RUN_DIRECTIVE（含 Post-Run Protocol），不注入 step/node 信息
- The unified conversation log still stores the assistant/tool exchanges, errors, and user inputs.

---

### 4.3 Persistence Rules (What gets appended, and when)

Persist **both user-visible and internal** messages:

1) User input
   - Append `role=user` message (already done today).

2) LLM response
   - Append `role=assistant` message including `tool_calls` (even if `content=null`).
   - Append final assistant content message (if present).

3) Tool results
   - Append `role=tool` message for each tool result with `tool_call_id` (protocol-complete).
   - Store `toolName`, `duration`, and a UI-friendly collapsed preview.
   - For large tool outputs, write the full output to a dedicated file under `@state/logs/` (or `@project/artifacts/`) so the LLM can read it on demand; store only a preview + alias path in context.

4) Errors (critical for “continue”)
   - On LLM error/timeout/network:
     - Append `role=assistant` message that contains:
       - partial streamed text (if any)
       - a structured error block (`LLM_ERROR ...`)
     - This message must be included in context so retries can resume.

5) Mode switch
   - History is not cleared.
   - No explicit mode-switch events are required in the message log for MVP.
   - Persist the currently active mode via `ConversationMetadata.activeType` and have the context builder use the current mode’s system prefix.
   - Recommendation for correctness: tag messages with `runId` when produced during workflow execution so multiple runs in one conversation can be disambiguated without relying on mode-switch events.

---

## 5. Integration Points

### 5.1 Chat mode (`chat:send`)
- Load conversation messages for `conversationId`
- Use the unified context builder to assemble `messages[]`
- Execute via `chatToolLoop`
- Persist assistant/tool messages and errors into the same conversation log
- UI should show errors as assistant messages (not only `alert`)

### 5.2 Agent mode (`agent:dispatch` chat paths)
- Same as Chat mode, but `PromptComposer.compose(mode='agent')` includes persona.

### 5.3 Run mode (`ExecutionEngine`)
Unified approach (Chosen):
- Replace `session.history` as the source of truth with **conversation-backed history**.
- `ExecutionEngine` reads/writes the conversation log as the canonical history store (an in-memory cache is allowed for performance, but must be reconstructible from persisted messages on restart).

---

## 6. Backwards Compatibility / Migration

- Existing `messages.json` contain messages without `tool_calls/tool_call_id/role=tool`.
- Loading should treat missing fields as:
  - `content` = existing `content`
  - `includeInContext` = true
- New messages can be appended with extended schema without breaking older clients.

---

## 7. Settings (Proposed)

Add context controls (safe defaults, similar to Codex “sliding window”):

```ts
settings.llm.contextMaxMessages = 80
settings.llm.contextMaxChars = 120_000        // fallback
settings.llm.includeToolResultsInContext = true
settings.llm.includeCollapsedUiEventsInContext = false
```

MVP can start with `contextMaxMessages` only and iterate.

---

## 8. Test Plan

### Unit
- Context builder:
  - keeps latest user message
  - trims older messages correctly
  - preserves tool-call groups
  - mode banner and persona rules are correct (chat omits persona)
- Persistence:
  - tool results stored as `role=tool` with `tool_call_id`
  - LLM failures store `LLM_ERROR` + partial output and are included in subsequent request

### Integration
- Chat mode:
  - send → tool calls → final answer; reopen app; conversation still has tool messages; next message includes history.
- Mode switch:
  - chat → agent → run; history remains; system prompt changes; run still functions.
- Failure recovery:
  - simulate LLM timeout; verify a visible assistant error message exists and “continue” resumes from it.

---

## 9. Open Questions

1) Tool outputs in context: **Resolved** → preview + “read logs/path” (full output saved under `@state`/`@project`).
2) Automatic “conversation summary” message: optional v2. Meaning: when older messages are trimmed, insert a short summary message that represents the dropped segment so the LLM keeps continuity without full history.
3) Persist mode switches as explicit history events: **Resolved** → not required; store active mode in `ConversationMetadata.activeType` and use dynamic system prefix.
4) Unified conversation log as the only source for Run resumption: **Resolved** → yes.
