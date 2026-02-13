# Story BAI-3.4: Add Revision Safety Contract (Single-Writer MVP)

Status: done

> Epic: `_bmad-output/epics-builder-ai.md`  
> Tech Spec: `_bmad-output/tech-spec/builder-ai-implementation-spec.md`  
> Architecture: `_bmad-output/architecture/builder-ai-llm-interaction-architecture.md`

## Story

As a **System**,  
I want revision mismatch to be visible and recoverable during manual apply,  
So that stale preview risk is explicit while keeping MVP simple.

## Acceptance Criteria

### AC-1: mismatch warning 契约稳定
- **Given** `revisionBase` does not match current snapshot
- **When** apply executes
- **Then** returns `AI_REVISION_BASE_MISMATCH` warning with `field/provided/current`

### AC-2: Workbench 同页恢复
- **Given** mismatch warning returned
- **When** frontend renders result
- **Then** user keeps current conversation + preview context
- **And** user can continue optimize or refresh/regenerate in same page

### AC-3: 单用户假设冻结
- **Given** current release model
- **Then** assume single-project single-writer
- **And** DB-level atomic lock/strict conflict block are deferred

## Design Decisions (Frozen)

1. 当前版本不强制 409 conflict。
2. warning-first 语义是上线主契约。
3. 并发原子锁与强阻断放入后续版本独立 story。

## Tasks / Subtasks

- [x] Task 1: 统一 revision compare helper 与 warning payload
- [x] Task 2: 前端同页恢复文案与动作对齐原型
- [x] Task 3: 更新接口文档与测试断言（warning-first）

## Test Plan

- 任一 revision 字段 mismatch 返回 warning（不阻断）
- warning 下 apply 事务行为不退化
- 前端收到 warning 后不丢对话/预览状态
