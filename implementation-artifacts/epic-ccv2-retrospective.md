# Epic Retrospective: CCV2 Runtime Context Compression V2

**Date:** 2026-05-16  
**Epic:** `CCV2-1`  
**Scope:** `CCV2-1.1 ~ CCV2-1.4`

## 1. Objective

回顾 Runtime Context Compression V2 的交付质量、协议安全边界和剩余风险，把本轮从“压缩触发了但没真正变小”到“协议安全、provider-aware、可摘要、可兜底”的改造经验固化下来，作为后续长上下文和长工具链任务的基线。

## 2. Status Snapshot

- `CCV2-1.1` Compression Observability and Protocol Foundation: `done`
- `CCV2-1.2` Protocol-Aware Structural Compression: `done`
- `CCV2-1.3` LLM Summary and Cross-Mode Integration: `done`
- `CCV2-1.4` Active Tool-Loop Compression and Final Fallback: `done`

## 3. What Was Delivered

1. 压缩结果从单一 `compressed` 扩展为 `attempted`、`effective`、`status` 和完整 metadata，避免 logs 把 no-op 误报成有效压缩。
2. 增加 tool-call group classifier、protocol validator 和 protocol repair，确保压缩后不会产生孤立 `tool` message 或半截 tool-call group。
3. 增加 DeepSeek/Kimi/provider-aware reasoning policy，默认不让旧 `reasoning_content` 长期膨胀上下文，同时保留近期和当前 tool-call group 的 provider 安全需求。
4. 将旧的完整 tool-call group 转成普通 assistant `TOOL_CALL_HISTORY_SUMMARY`，不再携带 `tool_calls`、`tool_call_id`、`reasoning_content`。
5. 增加 `CONVERSATION_SUMMARY` 解析、校验、确定性 fallback 和可选 LLM summary service，用一个滚动语义摘要承接被压缩历史。
6. Chat、Agent、Run/Workflow、Chat Tool Loop 共用 V2 pipeline 和统一 telemetry，不再把 Plan Chat 当成特殊根因处理。
7. 增加 active tool-loop compression：当第一轮长任务没有旧历史可压时，可以压缩当前用户 turn 内已经完成的旧工具调用组。
8. 增加 final fallback：如果历史压缩、当前工具链压缩和更激进 fallback 后仍然超预算，Runtime 返回 `CONTEXT_STILL_TOO_LARGE`，不盲发超预算模型请求。
9. 修复 Chat MCP forced retry 的额外请求路径：追加强制 MCP system prompt 后也会走最终预算检查，避免第二次 LLM retry 绕过兜底。

## 4. Current Compression Strategy

Runtime 现在按以下顺序处理上下文：

1. 先判断是否接近上下文阈值。
2. 如果需要压缩，先压旧历史，不动当前正在进行的工具调用链。
3. 历史里的旧工具调用必须整组处理：要么完整保留，要么变成普通摘要消息。
4. 旧的 `reasoning_content` 默认不长期带入上下文；近期和当前工具调用按 provider policy 保留。
5. 被移除或摘要的旧历史会尽量写入 `CONVERSATION_SUMMARY`，让模型还能看到目标、状态、文件、决策和待办。
6. 如果旧历史压完还是太大，再压当前工具链中已经完成的旧工具调用组，生成 `TOOL_LOOP_SUMMARY`。
7. 如果仍然太大，执行更激进但协议安全的 fallback。
8. 如果最后还是超预算，停止自动 loop，返回 `CONTEXT_STILL_TOO_LARGE`，不继续请求模型。

## 5. What Worked Well

1. 先做 telemetry 和 protocol foundation 是正确的，否则后续摘要工具调用组会缺少安全边界。
2. provider-aware reasoning policy 明确了 DeepSeek/Kimi 的默认行为，避免把普通用户暴露到 `thinking.keep` 之类内部配置。
3. `CONVERSATION_SUMMARY` 和 `TOOL_LOOP_SUMMARY` 分工清楚：前者承接旧对话历史，后者承接当前长工具链。
4. active tool-loop compression 解决了“第一轮就触发大量 function_call，没有旧历史可压”的极端情况。
5. code-review 中反复补上的 final request guard 有价值，尤其是 Run/Workflow 最终组装请求、Chat skill prefix、Chat MCP forced retry 这些容易绕过主压缩点的路径。

## 6. Gaps / Risks

1. `CONVERSATION_SUMMARY` 当前以确定性 fallback 为主，真正 LLM summary 的质量还需要真实长会话样本继续校准。
2. token estimator 仍是估算值，不能完全等同每个 provider 的真实 tokenizer；final guard 只能降低风险，不能替代 provider 端限制。
3. MCP forced retry 如果追加提示后超预算，目前选择停止而不是再尝试一次压缩。这符合“不盲发超预算请求”，但可能让部分临界请求提前停止。
4. Plan mode 自身注入的 plan context 仍未做独立摘要或限长；本轮解决的是通用历史压缩和当前工具链压缩，不是 plan 文档压缩。
5. `TOOL_LOOP_SUMMARY` 是 request-local，不持久写回 conversation storage；如果后续需要跨重启保留当前工具链摘要，需要单独 story。
6. 压缩摘要是事实性 preview，不等价于完整可审计日志。完整工具调用审计仍应依赖原始 logs/history 存储。

## 7. Decisions Carried Forward

1. Tool-call protocol group 只能整组保留或整组摘要，不能删除半截。
2. 历史 `reasoning_content` 默认不长期带入上下文；只有显式内部 keep-all 策略才保留。
3. `CONVERSATION_SUMMARY` 和 `TOOL_LOOP_SUMMARY` 都是普通消息，不携带 tool protocol 字段。
4. LLM summary 是增强能力，失败时必须 deterministic fallback，不能阻塞主请求。
5. 所有最终发送给模型的请求都必须在组装完成后再做一次预算和协议检查，不能只相信压缩阶段的结果。
6. Plan mode 的问题应建立在通用压缩基线之上，不做 plan-only 特例修复。

## 8. Verification Evidence

- `npm test -- --run electron/services/chatToolLoop.test.ts`
  - 28 tests passed.
- `npm test -- --run electron/core/context-compression/context-compression.test.ts electron/services/chatContextBuilder.test.ts electron/services/agentContextBuilder.test.ts electron/services/chatToolLoop.test.ts electron/services/executionEngine.test.ts`
  - 149 tests passed.
- `npm run build:ci`
  - Passed.
  - Existing warnings: npm unknown user config warnings, `jspreadsheet-ce` eval warning, Vite large chunk warnings, and `mcpInstallService` mixed dynamic/static import warning.
- `git diff --check`
  - Passed.

## 9. Evidence Links

- PRD: `_bmad-output/prd-runtime-context-compression-v2.md`
- Architecture: `_bmad-output/architecture/runtime-context-compression-v2-architecture.md`
- Epic: `_bmad-output/epics-runtime-context-compression-v2.md`
- Story CCV2-1.1: `_bmad-output/implementation-artifacts/ccv2-1-1-compression-observability-and-protocol-foundation.md`
- Story CCV2-1.2: `_bmad-output/implementation-artifacts/ccv2-1-2-protocol-aware-structural-compression.md`
- Story CCV2-1.3: `_bmad-output/implementation-artifacts/ccv2-1-3-llm-summary-and-cross-mode-integration.md`
- Story CCV2-1.4: `_bmad-output/implementation-artifacts/ccv2-1-4-active-tool-loop-compression-and-final-fallback.md`

## 10. Follow-Up Candidates

1. Plan context compression: 对 plan markdown、remarks、execution state 做独立限长和摘要。
2. Real LLM summary calibration: 用合成和匿名化长会话 fixture 校准 `CONVERSATION_SUMMARY` 质量。
3. Provider tokenizer calibration: 为 DeepSeek/Kimi/OpenAI-compatible 增加更接近真实 tokenizer 的预算估算。
4. Persistent tool-loop summary: 如果需要跨重启恢复超长当前工具链摘要，另开 story 设计持久化边界。
5. User-facing stop recovery: 当返回 `CONTEXT_STILL_TOO_LARGE` 时，前端可给出更明确的恢复动作和重试建议。
