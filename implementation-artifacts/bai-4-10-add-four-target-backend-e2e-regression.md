# Story BAI-4.10: Add Four-Target Backend E2E Regression and Contract Lock

Status: done

> Epic: `_bmad-output/epics-builder-ai.md`  
> Tech Spec: `_bmad-output/tech-spec/builder-ai-implementation-spec.md`

## Story

As a **Release Owner**,  
I want end-to-end regression coverage for workflow/step/agent/asset message paths,  
So that future iterations do not silently break non-step target behavior.

## Acceptance Criteria

### AC-1: Four-target message regression
- **Given** workflow/step/agent/asset sessions
- **When** running API regression
- **Then** each target can complete `message -> (optional) stage -> validate` successfully.

### AC-2: Error contract lock
- **Given** target mismatch, malformed context, or validation failures
- **When** endpoint returns errors
- **Then** code/message/hints structure is stable and asserted by tests.

### AC-3: Session lifecycle lock
- **Given** apply and cancel flows
- **When** tests run after multi-target changes
- **Then** apply keeps session active; cancel clears working copy and closes session.

## Design

### Summary
- Add API contract tests for all targets.
- Freeze critical response fields for `messages/apply/cancel`.
- Guard against regressions by locking key business errors.

### API / Contracts
- Existing endpoints only; no path changes.

### Regression Matrix Freeze

#### Case Group A: Message happy path

- A1: step target chat-only
- A2: step target change + validate
- A3: workflow target chat-only
- A4: workflow target change + validate
- A5: agent target chat-only
- A6: agent target change + validate
- A7: asset target chat-only
- A8: asset target change + validate

#### Case Group B: Error contract

- B1: target/context mismatch -> `AI_TARGET_CONTEXT_MISMATCH`
- B2: required context missing -> `AI_TARGET_CONTEXT_REQUIRED`
- B3: bad response shape -> `AI_BAD_RESPONSE`
- B4: tool apply called in loop -> `AI_TOOL_FORBIDDEN`

#### Case Group C: Lifecycle

- C1: apply success -> session still `active`
- C2: cancel success -> session `cancelled` + working copy cleared
- C3: apply mismatch revision -> warning `AI_REVISION_BASE_MISMATCH` (non-blocking)

### Design Decisions (Frozen)

1. 多 target 用例进入同一测试套件，避免分散到不同文件导致漏测。
2. 关键错误码做精确断言（code + status + core message fragment）。
3. E2E 回归完成后，BAI-4 才能重新进入 `done`。

## Technical Scope

- `crewagent-builder-backend/tests/test_ai_workbench_change_api.py`
- `crewagent-builder-backend/tests/test_ai_step_draft.py`
- Optional new suite:
  - `crewagent-builder-backend/tests/test_ai_workbench_multi_target_api.py`
- `_bmad-output/implementation-artifacts/sprint-status-builder-ai.yaml`

## Tasks / Subtasks

- [x] Task 1: 覆盖四 target 的 message 主路径（AC: 1）
- [x] Task 2: 覆盖错误契约稳定性（AC: 2）
- [x] Task 3: 回归 apply/cancel 生命周期（AC: 3）
- [x] Task 4: CI 测试分组和执行说明更新（AC: 1,2,3）

## Test Plan

- 四 target 全路径回归通过
- 错误契约快照稳定
- apply/cancel 语义不回退

## Design Review Notes

- 评审日期：2026-02-12
- 结论：实现完成并回归通过，状态推进至 `done`

## Code Review Closure Notes (2026-02-12)

- 关闭问题：补齐 Case Group A/B 缺口（workflow change、agent chat-only、asset change、required context、bad response、tool forbidden）。
- 关闭问题：确认错误请求不会写入会话消息表，保证回归测试与线上行为一致。
- 回归结果：`tests/test_ai_step_draft.py`（25 passed）、`tests/test_ai_workbench_change_api.py`（7 passed）。
