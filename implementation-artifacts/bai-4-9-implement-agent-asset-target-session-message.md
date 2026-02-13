# Story BAI-4.9: Implement Agent and Asset Target Session Message Path

Status: done

> Epic: `_bmad-output/epics-builder-ai.md`  
> Tech Spec: `_bmad-output/tech-spec/builder-ai-implementation-spec.md`

## Story

As a **Builder User**,  
I want agent and asset target sessions to use the same tool-loop + working-copy pipeline,  
So that all target types behave consistently in AI Workbench.

## Acceptance Criteria

### AC-1: Agent target path available
- **Given** `targetType=agent` session
- **When** sending message
- **Then** backend returns summary-only output and optional staged+validated changeSet.

### AC-2: Asset target path available
- **Given** `targetType=asset` session
- **When** sending message
- **Then** backend returns summary-only output and optional staged+validated changeSet.

### AC-3: Schema/validation guardrails consistent
- **Given** malformed agent/asset changes
- **When** running validate
- **Then** it returns structured errors with stable codes.

## Design

### Summary
- 在 `/messages` 增加 `agent` 与 `asset` 两个 target 分支。
- 统一使用 `responseType + assistantSummary + changeSet` 契约，不新增独立 endpoint。
- 继续沿用 stage/validate/manual apply 与 summary-only 对话策略。

### API / Contracts
- Reuse existing `messages` API and `AiSessionMessageOut`.

### Interface Freeze

#### Agent / Asset Output Contract（冻结）

- chat 回合：
  - `{"responseType":"chat","assistantSummary":"..."}`
- change 回合：
  - `{"responseType":"change","assistantSummary":"...","changeSet":{...}}`
  - `changeSet.targetType` 对应 `agent` 或 `asset`

#### Server Processing Flow（冻结）

1. `session.target_type=agent|asset` 进入对应 message handler。
2. 组装 `ComposeInput`（entrypoint 分别为 `agent` / `asset`）。
3. 执行 `run_tool_loop(...)`。
4. 解析输出：
   - chat -> `hasChange=false`
   - change -> stage + validate，返回 `normalizedChangeSet/errors/warnings`

#### Guardrails（冻结）

- asset 内容读取：Prompt 只给路径，内容必须通过 `builder.asset.read` 工具按需读取。
- 模型不可调用 apply；若调用则 `AI_TOOL_FORBIDDEN`。

### Errors / Edge Cases
- Missing agent/asset context fields should fail fast with typed 400.
- Invalid target id should map to business not-found code.

### Design Decisions (Frozen)

1. Agent/Asset 与 Workflow/Step 共用一套消息响应结构，避免前端分裂处理。
2. 非法目标或上下文错配统一走业务错误码，不回落为 500。
3. 本 Story 不改 apply/cancel 路由，仅扩展 messages 可达性。

## Technical Scope

- `crewagent-builder-backend/app/routers/packages.py`
- `crewagent-builder-backend/app/services/semantic_tool_service.py`
- `crewagent-builder-backend/app/services/working_copy_service.py`
- `crewagent-builder-backend/tests/test_ai_workbench_change_api.py`
- `crewagent-builder-backend/tests/test_semantic_tool_service.py`
- `crewagent-builder-backend/tests/test_ai_workbench_multi_target_api.py`（新增）

## Tasks / Subtasks

- [x] Task 1: 新增 agent message handler 与分发（AC: 1）
- [x] Task 2: 新增 asset message handler 与分发（AC: 2）
- [x] Task 3: 统一 changeSet 校验错误映射（AC: 3）
- [x] Task 4: 补齐 agent/asset 回归测试（AC: 1,2,3）

## Test Plan

- agent target chat-only + hasChange path
- asset target chat-only + hasChange path
- malformed payload/invalid target 的错误码稳定

## Design Review Notes

- 评审日期：2026-02-12
- 结论：实现完成并回归通过，状态推进至 `done`

## Code Review Closure Notes (2026-02-12)

- 补测：新增 agent chat-only（A5）与 asset change（A8）消息路径回归，完善非 step 目标覆盖面。
- 补测：新增 `AI_BAD_RESPONSE` / `AI_TOOL_FORBIDDEN` 在 `/messages` 链路的错误契约断言。
