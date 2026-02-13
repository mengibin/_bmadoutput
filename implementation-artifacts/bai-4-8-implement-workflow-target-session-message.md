# Story BAI-4.8: Implement Workflow Target Session Message Path

Status: done

> Epic: `_bmad-output/epics-builder-ai.md`  
> Tech Spec: `_bmad-output/tech-spec/builder-ai-implementation-spec.md`

## Story

As a **Builder User**,  
I want workflow target sessions to run tool-loop based suggestions and staged validation,  
So that workflow-level AI optimization uses the same workbench backend path.

## Acceptance Criteria

### AC-1: Workflow message flow is executable
- **Given** `targetType=workflow` session
- **When** sending a message
- **Then** backend returns summary-only assistant response and optional staged+validated changeSet.

### AC-2: Working copy semantics preserved
- **Given** message generates changes
- **When** user has not applied
- **Then** workflow changes exist only in session working copy/overlay and do not mutate business table.

### AC-3: Apply compatibility
- **Given** validated workflow changes
- **When** user manually calls apply
- **Then** apply contract remains unchanged (`confirmSource=ui_manual_apply`).

## Design

### Summary
- 在 `/messages` 中新增 workflow 分支，复用 tool-loop + stage/validate 主链路。
- 前端对话区保持 summary-only；有变更时返回 `normalizedChangeSet/errors/warnings`。
- 不新增路由，保持 manual apply 契约不变。

### API / Contracts
- No new route.
- `messages` response shape unchanged.

### Interface Freeze

#### Workflow Message Output Contract（冻结）

模型最终消息 JSON（非工具事件）约定：

- chat 回合：
  - `{"responseType":"chat","assistantSummary":"..."}`
- change 回合：
  - `{"responseType":"change","assistantSummary":"...","changeSet":{...}}`
  - `changeSet` 必须满足 `crewagent.builder.ai.change-set.v1`

#### Server Processing Flow（冻结）

1. `_create_ai_workbench_message_impl` 对 `target_type=workflow` 分发到 workflow handler。
2. 组装 `ComposeInput(entrypoint="workflow", targetType="workflow", mode=session.mode)`。
3. 运行 `run_tool_loop(...)` 并收集 runtimeTrace。
4. 解析最终 JSON：
   - `responseType=chat` -> `hasChange=false`
   - `responseType=change + changeSet` -> `stage_change_set_record` + `validate_staged_change_set`
5. 返回 `AiSessionMessageOut`（保持现有字段）。

#### Error Codes（冻结）

- 输出非 JSON 或契约缺失：`AI_BAD_RESPONSE`
- `responseType=change` 但缺少 `changeSet`：`AI_BAD_RESPONSE`
- workflow 不存在：`WORKFLOW_NOT_FOUND`

### Errors / Edge Cases
- Missing workflow context should be explicit 400 error.
- Unsupported workflow mutation shape should fail validation, not silently drop.

### Design Decisions (Frozen)

1. workflow 目标不复用 step 的 `suggested/assets` 草稿结构，统一改为 `changeSet` 契约。
2. 是否写业务表由 apply 决定；messages 阶段只做 stage/validate。
3. apply 后 session 仍保持 active（沿用已冻结语义）。

## Technical Scope

- `crewagent-builder-backend/app/routers/packages.py`
- `crewagent-builder-backend/app/services/tool_loop_service.py`
- `crewagent-builder-backend/app/services/semantic_tool_service.py`
- `crewagent-builder-backend/app/services/working_copy_service.py`
- `crewagent-builder-backend/tests/test_ai_workbench_change_api.py`
- `crewagent-builder-backend/tests/test_ai_workbench_multi_target_api.py`（新增）

## Tasks / Subtasks

- [x] Task 1: 新增 workflow message handler 与分发逻辑（AC: 1）
- [x] Task 2: 实现 workflow chat/change 契约解析与 stage/validate（AC: 1,2）
- [x] Task 3: apply 契约回归（`confirmSource` / active session）（AC: 3）
- [x] Task 4: workflow target API 测试补齐（AC: 1,2,3）

## Test Plan

- workflow session message -> chat-only roundtrip
- workflow session message -> hasChange + validate 成功
- apply 后 revision 更新且会话保持 active

## Design Review Notes

- 评审日期：2026-02-12
- 结论：实现完成并回归通过，状态推进至 `done`

## Code Review Closure Notes (2026-02-12)

- 修复：workflow 分支在上下文回退失败时返回 `AI_TARGET_CONTEXT_REQUIRED`，不再吞错为通用构建失败。
- 补测：新增 workflow target `hasChange + stage/validate` 回归用例，锁定 A4 路径。
