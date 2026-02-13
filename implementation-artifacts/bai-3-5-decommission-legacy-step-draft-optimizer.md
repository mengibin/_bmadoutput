# Story BAI-3.5: Decommission Legacy Step-Draft Optimizer Path

Status: done

> Epic: `_bmad-output/epics-builder-ai.md`  
> Tech Spec: `_bmad-output/tech-spec/builder-ai-implementation-spec.md`  
> Architecture: `_bmad-output/architecture/builder-ai-llm-interaction-architecture.md`

## Story

As a **System**,  
I want to fully retire legacy `step-draft` optimizer paths,  
So that Step AI uses only the unified Workbench session/tool-loop flow.

## Acceptance Criteria

### AC-1: 旧路径退场
- **Given** any request to legacy `step-draft` endpoint
- **When** backend receives request
- **Then** backend returns structured deprecation response (or endpoint removed per release policy)
- **And** no business mutation occurs

### AC-2: 主路径唯一
- **Given** Step create/optimize use case
- **When** user triggers AI flow
- **Then** path is `ai sessions + messages + tool loop + stage/validate/manual apply`

### AC-3: 前端去依赖
- **Given** Workbench frontend
- **When** sending/retrying AI request
- **Then** no client code calls `/ai/workbench/step-draft` or stream variants

## Design Decisions (Frozen)

1. `step-draft` 不再是主业务链路。
2. Step AI 统一收敛到 session/message/tool-loop。
3. 前后端均清理 `step-draft` 专用分支、类型与测试基线。

## Tasks / Subtasks

- [x] Task 1: 后端下线/停用 legacy `step-draft` 路由与服务耦合
- [x] Task 2: 前端移除 `requestAiStepDraft*` 调用链
- [x] Task 3: 测试迁移到 session/messages 主链路
- [x] Task 4: 更新文档与监控项

## Test Plan

- grep 代码库不再存在 workbench step-draft API 调用
- Step AI 全流程仅走 session/message/tool-loop
- legacy endpoint 命中时返回受控结果且无数据写入
