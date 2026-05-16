---
stepsCompleted: [1, 2, 3, 4]
inputDocuments:
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/architecture/unified-conversation-context.md'
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/7-4-create-context-compression-module.md'
workflowType: 'prd'
lastStep: 4
project_name: 'CrewAgent Runtime Context Compression V2'
user_name: 'Mengbin'
date: '2026-05-14'
---

# CrewAgent Runtime Context Compression V2 PRD

## Implementation Status

- Status: `done`
- Completed scope: `CCV2-1.1 ~ CCV2-1.4`
- Closure artifact: `_bmad-output/implementation-artifacts/epic-ccv2-retrospective.md`
- Follow-up decision: no `CCV2-1.5` is required for Plan mode at this time because Plan stores and injects the latest current plan rather than accumulating old plan versions in history.

## Overview

CrewAgent Runtime 已有 `CompressionPipeline -> KeyMessageExtractionStrategy` 压缩链路，但现有实现主要是启发式关键消息选择和截断。它能覆盖简单长文本历史，但在工具调用密集、thinking 模型返回大量 `reasoning_content`、多模式长会话场景下，可能出现“logs 显示已压缩，但压缩后上下文 token 和消息量没有明显下降”的问题。

本 PRD 定义 Context Compression V2：一套跨 Chat、Agent、Run/Workflow 通用的历史上下文压缩能力。它必须协议安全、可观测、能处理 DeepSeek/Kimi 等 thinking + tool-call 模型，并能在结构压缩不足时使用 LLM 生成滚动语义摘要。

## Problem Statement

### Current Problem

现有压缩策略存在以下问题：

- `compressed: true` 当前只代表压缩流程被触发，不代表压缩结果有效。
- `assistant(tool_calls) + tool(result)` group 被强制完整保留，工具密集历史几乎没有可删空间。
- 历史 `reasoning_content` 会长期进入上下文，尤其是 DeepSeek/Kimi thinking 模型的工具调用 turn。
- 旧工具调用历史没有语义摘要层，只能保留、截断或丢弃。
- 压缩策略在 Chat、Agent、Run/Workflow 中共用，因此问题不是 Plan 模式独有。
- Plan 模式会注入当前 plan context，容易放大上下文压力，但不是根因。
- 第一个 conversation session 的第一个用户请求如果触发数百次 function call，历史压缩可能没有足够旧历史可压，当前工具链本身会成为上下文膨胀来源。

### Observed Failure Mode

在 2026-05-13 的 plan chat 会话中，历史消息约 634 条，工具调用和 tool result 占大头。系统 logs 能看到 `Context compression applied`，但 before/after token 基本不下降。根因是大量消息被策略判定为 mandatory，且多数 assistant tool-call 消息带有 `reasoning_content`，无法被当前策略摘要。

## Goals

- 建立可区分 attempted、effective、no-op 的压缩结果模型。
- 保证压缩后的 OpenAI-compatible messages 始终符合 tool-call protocol。
- 对 thinking 模型的历史 `reasoning_content` 做 provider-aware 处理。
- 对旧 tool-call group 做整组摘要化，而不是保留半截协议。
- 引入 LLM rolling summary，用语义摘要承接被移除历史。
- 在 Chat、Agent、Run/Workflow 三类入口统一使用同一套核心压缩能力。
- 提供清晰日志、usage 指标和 debug metadata，避免误导用户。
- 当历史压缩后仍接近上下文上限时，继续压缩当前工具链中已经完成的旧工具调用组。
- 当历史压缩和当前工具链压缩仍不足时，进入明确兜底，避免直接发送超预算请求。

## Non-Goals

- 不实现无限长期记忆或跨会话记忆。
- 不把所有历史 reasoning 永久保留作为默认行为。
- 不允许 LLM 直接重写正在进行中的 tool-call protocol。
- 不把 Plan 模式作为唯一修复点。
- 不在 V2 中引入向量数据库或 RAG 检索作为必需组件。
- 不压缩未闭合的当前 tool-call group。
- 不把被压缩工具组的 `reasoning_content` 写入摘要。

## User Personas

### Chat User

用户在普通 Chat 或 Plan Chat 中长时间对话，希望系统在上下文变长时自动压缩，并且不会丢失关键目标、约束和执行状态。

### Agent User

用户让 Agent 连续读文件、写文件、运行命令，希望工具历史不会把上下文撑爆，同时最近工具调用仍能被模型正确续接。

### Workflow User

用户运行多 step workflow，希望 Run 模式能在长执行链路中保持状态可恢复、协议完整和上下文可控。

### Runtime Developer

开发者需要通过 logs 和 metadata 判断压缩是否真正有效，以及压缩失败的原因。

## Functional Requirements

### FR-CCV2-01: Accurate Compression State

系统必须区分：

- `attempted`: 上下文超过阈值，压缩流程被尝试。
- `effective`: 压缩后 token 或消息数实际下降。
- `no-op`: 触发压缩但没有有效减少。
- `failed`: 压缩过程失败并回退到安全上下文。

### FR-CCV2-02: Compression Result Metadata

每次压缩必须返回：

- before tokens / messages
- after tokens / messages
- target tokens
- dropped messages
- summarized tool groups
- stripped reasoning count
- summary tokens
- target exceeded flag
- no-op 或 failed 原因

### FR-CCV2-03: Protocol-Safe Tool-Call Handling

压缩后的 messages 必须满足 OpenAI tool-call protocol：

- 不允许孤立 `tool` message。
- 不允许保留带 `tool_calls` 的 assistant 却删除对应 tool result。
- 不允许 tool result 对应到不存在的 `tool_call_id`。
- 旧 tool-call group 要么完整保留，要么整组变为普通摘要消息。

### FR-CCV2-04: Tool-Call Group Classification

系统必须识别并分类完整 tool-call group：

- current loop group
- recent group
- historical group
- incomplete or invalid group

不同类别必须使用不同压缩策略。

### FR-CCV2-05: Provider-Aware Reasoning Handling

系统必须按模型和 provider 处理 `reasoning_content`：

- DeepSeek thinking tool-call 的当前和最近 group 保留 reasoning。
- DeepSeek 非 tool-call 历史 reasoning 可剥离。
- Kimi 默认 historical reasoning 可不保留。
- Kimi `thinking.keep: "all"` 或等价配置启用时，压缩前应记录该配置对上下文成本的影响。
- 旧 tool-call group 若被整组摘要，则不再发送原 `tool_calls`、`tool` messages 和 `reasoning_content`。

### FR-CCV2-06: Deterministic Structural Compression

系统必须先执行不依赖 LLM 的结构压缩：

- 保留当前 loop。
- 保留最近 N 轮用户交互。
- 保留最近 M 个 tool-call group。
- 剥离安全的 historical reasoning。
- 将旧 tool result 截断为 preview 或整组摘要。
- 复用已有 `CONVERSATION_SUMMARY`。

### FR-CCV2-07: LLM Rolling Summary

当结构压缩后仍超过目标预算，或需要语义保真时，系统应调用 LLM 生成或更新 `CONVERSATION_SUMMARY`。

摘要必须覆盖：

- user goal
- current status
- confirmed constraints
- key decisions
- files and paths
- tool results
- completed actions
- pending tasks
- risks and blockers
- do not repeat

### FR-CCV2-08: LLM Summary Fallback

LLM 摘要失败、超时或返回不合格内容时，系统必须回退到 deterministic summary，不得阻塞主请求。

### FR-CCV2-09: Cross-Mode Integration

Chat、Agent、Run/Workflow 必须接入同一套核心压缩 pipeline。入口可以提供 mode-specific context，但不得复制三套压缩逻辑。

### FR-CCV2-10: Plan Mode Compatibility

Plan 模式必须继续注入当前 plan state、plan markdown 和 remarks，但历史消息压缩必须走通用 V2 pipeline。Plan context 本身的限长和摘要可作为后续增强，不替代通用历史压缩。

### FR-CCV2-11: Observability

logs 必须准确表达：

- compression attempted
- compression effective
- compression no-op
- compression failed with fallback

UI context usage 必须显示真实压缩后的 usage。

### FR-CCV2-12: Test Fixtures

必须新增 synthetic fixtures，覆盖：

- 300+ tool-call group
- 其中多数 assistant 带 `reasoning_content`
- 大 tool args
- 大 tool results
- mixed user/assistant/tool history

测试不得使用真实用户历史内容。

### FR-CCV2-13: Active Tool-Loop Compression Trigger

系统必须在历史压缩后重新计算上下文大小。如果上下文仍接近或超过目标预算，系统必须判断是否需要压缩当前用户请求内部已经完成的旧工具调用组。

触发条件不应只依赖工具调用数量，而应依赖：

- 历史压缩已经执行或评估。
- 历史压缩后仍接近或超过目标预算。
- 当前 loop 中存在可安全摘要的已完成旧 tool-call group。

### FR-CCV2-14: Active Tool-Loop Protocol Safety

当前工具链压缩必须按完整 tool-call group 处理：

- assistant `tool_calls` 和匹配 tool results 要么一起保留。
- 要么整组替换成普通 assistant `TOOL_LOOP_SUMMARY`。
- 不允许保留孤立 tool result。
- 不允许保留带 `tool_calls` 的 assistant 却删除对应 tool result。
- 不允许压缩未闭合的 tool-call group。

### FR-CCV2-15: Final Fallback

历史压缩和当前工具链压缩后，如果上下文仍然过大，系统必须进入明确 final fallback：

- 先尝试更激进但协议安全的确定性压缩。
- 如果仍无法满足预算，则停止自动工具循环并返回清晰状态，例如 `CONTEXT_STILL_TOO_LARGE`。
- 不允许在已知超预算时继续发送请求直到 provider 报错。

## Non-Functional Requirements

### NFR-CCV2-01: Protocol Safety

压缩不能生成 provider 拒绝的非法 message 序列。协议校验失败时必须回退到更保守策略。

### NFR-CCV2-02: Reliability

LLM summary 是可选增强，失败时不能影响主对话请求。

### NFR-CCV2-03: Performance

结构压缩必须在本地快速完成。LLM summary 应支持超时和最大输入大小限制。

### NFR-CCV2-04: Privacy and Data Minimization

发送给 summary LLM 的输入应只包含需要摘要的历史片段，不应额外扩大数据面。

### NFR-CCV2-05: Backward Compatibility

现有 `ConversationMessage` 历史必须能正常加载。缺少 V2 metadata 的旧消息必须走默认策略。

### NFR-CCV2-06: Configurability

recent rounds、recent tool groups、summary token target、LLM summary timeout 等参数必须可配置或集中定义。

## Requirements Coverage

| Requirement | Coverage |
|:---|:---|
| FR-CCV2-01, FR-CCV2-02, FR-CCV2-11 | Accurate telemetry and logs |
| FR-CCV2-03, FR-CCV2-04 | Protocol-aware group model and validator |
| FR-CCV2-05 | Provider-aware reasoning policy |
| FR-CCV2-06 | Deterministic structural compression |
| FR-CCV2-07, FR-CCV2-08 | LLM rolling summary |
| FR-CCV2-09, FR-CCV2-10 | Cross-mode integration |
| FR-CCV2-12 | Regression fixtures |
| FR-CCV2-13, FR-CCV2-14, FR-CCV2-15 | Active tool-loop compression and final fallback |

## Acceptance Criteria

### AC-1: Effective Compression Detection

Given a context over threshold  
When compression is attempted but output tokens equal input tokens  
Then logs must show no effective reduction  
And metadata must set `effective = false`.

### AC-2: Tool Protocol Validity

Given a history with assistant tool calls and tool results  
When compression completes  
Then every remaining tool message must have a valid preceding assistant tool call  
And every remaining assistant tool call must have matching tool results.

### AC-3: Historical Reasoning Reduction

Given historical non-tool assistant messages with `reasoning_content`  
When compression runs  
Then old reasoning can be stripped  
And visible assistant content remains.

### AC-4: Old Tool Group Summary

Given old complete tool-call groups with large reasoning, args, and results  
When they fall outside the recent preservation window  
Then they can be replaced by ordinary summary messages  
And no original `tool_calls`, `tool` messages, or `reasoning_content` from those groups remain in the outgoing request.

### AC-5: DeepSeek/Kimi Recent Group Safety

Given recent thinking-model tool-call groups  
When compression runs  
Then current and recent groups preserve provider-required fields.

### AC-6: LLM Summary Fallback

Given LLM summary fails or times out  
When compression still needs to reduce history  
Then deterministic summary is used  
And the main LLM request proceeds.

### AC-7: Cross-Mode Coverage

Given Chat, Agent, and Run histories with equivalent tool-call pressure  
When each path builds LLM context  
Then all three use V2 compression and emit comparable metadata.

### AC-8: Active Tool-Loop Compression

Given a first-session first-turn task produces hundreds of complete function-call groups  
When historical compression is not enough  
Then Runtime compresses older completed current-loop tool groups into `TOOL_LOOP_SUMMARY`  
And preserves recent and incomplete groups.

### AC-9: Final Fallback

Given historical compression and active tool-loop compression still leave context above the safe target  
When Runtime prepares the next LLM request  
Then Runtime uses a deterministic final fallback or stops with a clear `CONTEXT_STILL_TOO_LARGE` state  
And does not blindly send an over-budget request.

## Open Questions

- Should Kimi `thinking.keep: "all"` be a user-visible setting, a provider preset, or an automatic mode inferred from model name?
- Should rolling summary be persisted in `messages.json`, stored as a synthetic system message, or generated transiently per request?
- Should old tool-call summaries be persisted back to conversation history, or remain request-local only?
- What default LLM should be used for summary if the main model is already near context limit?

## Source Notes

- DeepSeek thinking mode requires `reasoning_content` to be passed back for tool-call turns in subsequent requests: https://api-docs.deepseek.com/guides/thinking_mode
- Kimi thinking docs define `thinking.keep`; default historical reasoning is ignored, while `keep: "all"` preserves it and consumes context: https://platform.kimi.ai/docs/guide/use-kimi-k2-thinking-model
