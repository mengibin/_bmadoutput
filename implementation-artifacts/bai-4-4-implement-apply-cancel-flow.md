# Story BAI-4.4: Implement Apply/Cancel Flow with Return Navigation

Status: done

> Epic: `_bmad-output/epics-builder-ai.md`  
> Tech Spec: `_bmad-output/tech-spec/builder-ai-implementation-spec.md`  
> Prototype: `_bmad-output/pencil-new-work.pen`

## Story

As a **Builder User**,  
I want apply/cancel behavior to be explicit and safe,  
So that editing context is not lost.

## Acceptance Criteria

### AC-1: Cancel 安全返回且不写入
- **Given** user has not manually applied changes
- **When** clicking `Cancel and return`
- **Then** frontend calls HTTP `cancel` contract and returns to source page
- **And** semantics are `cancel session + discard overlay` with no business mutation

### AC-2: Apply 手动确认门禁
- **Given** validation is passed
- **When** clicking `Apply changes`
- **Then** request includes `changeSetId + revisionBase + confirmSource=ui_manual_apply`
- **And** apply is never auto-triggered by model loop

### AC-3: mismatch/失败可恢复
- **Given** apply returns warning/error
- **When** response is rendered
- **Then** keep current conversation and preview context
- **And** provide actions to refresh/regenerate or retry

### AC-4: apply 成功后不退出会话
- **Given** apply succeeded
- **When** frontend updates state
- **Then** session remains `active` and user stays in Workbench
- **And** user can continue next-round chat on new baseline

## Interface Freeze

- `POST /projects/{projectId}/ai/sessions/{sessionId}/apply`
- `POST /projects/{projectId}/ai/sessions/{sessionId}/cancel`

## Tasks / Subtasks

- [x] Task 1: 前端动作栏对齐 `apply/cancel` 新契约
- [x] Task 2: 去除 `discard` HTTP 命名依赖
- [x] Task 3: 对齐 `confirmSource=ui_manual_apply` 提交参数
- [x] Task 4: 补齐 apply/cancel/mismatch 回归测试

## Test Plan

- Cancel 调用 `cancel` API 且不产生业务写入
- Apply 请求包含 `confirmSource=ui_manual_apply`
- mismatch warning 不丢失上下文并可继续操作
- apply 成功后不自动跳转返回，且可继续发送下一轮消息
