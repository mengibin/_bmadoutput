# Story BAI-4.7: Expand Session Message Contract for Multi-Target

Status: done

> Epic: `_bmad-output/epics-builder-ai.md`  
> Tech Spec: `_bmad-output/tech-spec/builder-ai-implementation-spec.md`

## Story

As a **Backend Engineer**,  
I want `/ai/sessions/{sessionId}/messages` contract to accept workflow/agent/asset contexts,  
So that non-step targets can use one unified session messages endpoint.

## Acceptance Criteria

### AC-1: Request schema supports all target contexts
- **Given** `targetType` is `workflow|agent|asset`
- **When** request body is validated
- **Then** API accepts target-specific context payload and does not require `stepNode/stepContext`.

### AC-2: Backward compatibility for step target
- **Given** existing step message request
- **When** client calls `/messages`
- **Then** behavior and response contract stay backward compatible.

### AC-3: Unified error contract
- **Given** target context is missing or malformed
- **When** backend rejects the request
- **Then** it returns structured error code/message/hints (no generic 500).

## Design

### Summary
- 扩展 `/messages` 入参为“多 target 上下文联合模型”，不改 URL、不改响应结构。
- 路由层按 `session.target_type` 进行上下文选择与校验（以 session 为准，不信任客户端 target 推断）。
- 保持输出契约与 SSE runtime event 不变（前端无需重构消费层）。

### API / Contracts
- Route unchanged:
  - `POST /packages/{projectId}/ai/sessions/{sessionId}/messages`

### Interface Freeze

#### Request Schema（冻结）

`AiSessionMessageRequest` 增加以下可选上下文字段：

- `workflowContext?: AiWorkflowDraftContext`
- `stepNode?: AiStepNodeDraft`（保留）
- `stepContext?: AiStepDraftContext`（保留）
- `agentContext?: AiAgentDraftContext`
- `assetContext?: AiAssetDraftContext`

#### Target Context Matrix（冻结）

- `session.target_type = workflow`
  - 允许：`workflowContext`
  - 忽略：`stepNode/stepContext/agentContext/assetContext`
- `session.target_type = step`
  - 允许：`stepNode/stepContext`
  - 忽略：`workflowContext/agentContext/assetContext`
- `session.target_type = agent`
  - 允许：`agentContext`
  - 忽略：其余 context
- `session.target_type = asset`
  - 允许：`assetContext`
  - 忽略：其余 context

#### Error Contract（冻结）

- 缺少目标必需上下文（且无法从 DB/working copy 回退）：
  - `AI_TARGET_CONTEXT_REQUIRED`
- 客户端传入上下文与 session 目标明显冲突：
  - `AI_TARGET_CONTEXT_MISMATCH`
- 未支持目标（兜底）：
  - `AI_TARGET_NOT_SUPPORTED`

### Errors / Edge Cases
- 兼容旧前端：即使只传 `content`，只要后端可从 session + DB 还原目标快照也应继续执行。
- 新前端：优先传目标 context 以减少后端二次查询与歧义。
- 所有 context 错误都走 4xx 业务错误，禁止抛 500。

### Design Decisions (Frozen)

1. `/messages` 的目标类型只来自 `AiSession.target_type`，不从请求体推断。
2. step 现有请求和响应协议保持向后兼容。
3. 本 Story 仅冻结“契约与路由校验”，目标具体生成逻辑在 BAI-4.8/4.9 落地。

## Technical Scope

- `crewagent-builder-backend/app/schemas/ai_workbench.py`
- `crewagent-builder-backend/app/routers/packages.py`
- `crewagent-builder-backend/tests/test_ai_workbench_change_api.py`
- `crewagent-builder-backend/tests/test_ai_step_draft.py`
- `crewagent-builder-backend/tests/test_ai_workbench_multi_target_api.py`（新增）

## Tasks / Subtasks

- [x] Task 1: 扩展 `AiSessionMessageRequest` 多 target context 模型（AC: 1）
- [x] Task 2: 在 `/messages` 落地 target-context matrix 与错误码映射（AC: 1,3）
- [x] Task 3: 保持 step 兼容（现有 payload 无需改动）（AC: 2）
- [x] Task 4: 增加多 target 契约测试（AC: 1,2,3）

## Test Plan

- step 请求回归通过（无行为变化）
- workflow/agent/asset 请求可通过 schema 校验
- context 缺失/错配时返回预期错误码

## Design Review Notes

- 评审日期：2026-02-12
- 结论：实现完成并回归通过，状态推进至 `done`

## Code Review Closure Notes (2026-02-12)

- 修复：`/messages` 在 target/context 校验失败时不再提前写入 user message，避免无效请求污染会话历史。
- 修复：明确落地 `AI_TARGET_CONTEXT_REQUIRED` 错误码（含 details），与 Interface Freeze 契约保持一致。
- 回归：新增 context mismatch / context required 的消息不落库断言。
