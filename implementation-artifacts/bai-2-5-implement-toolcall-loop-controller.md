# Story BAI-2.5: Implement ToolCall Loop Controller

Status: done

> Epic: `_bmad-output/epics-builder-ai.md`  
> Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-bai-2-prompt-composer-context-tool-loop.md`

## Story

As a **System**,  
I want LLM-tool loop execution with stop conditions and repair loop,  
So that suggestions are generated from evidence, not guesswork.

## Acceptance Criteria

### AC-1: ToolCall 循环执行
- **Given** LLM returns tool calls
- **When** tool results are appended
- **Then** loop continues until no tool calls or max iterations reached

### AC-2: 白名单冻结
- **Given** loop is running in assistant tool-call mode
- **When** model calls tools
- **Then** only `read + builder.change.stage + builder.change.validate + builder.change.discard` are allowed
- **And** `builder.change.apply` is forbidden and returns `AI_TOOL_FORBIDDEN`

### AC-3: 停止条件与修复回路
- **Given** invalid output or non-converging loop
- **When** max limits are reached
- **Then** returns structured error (`AI_TOOL_LOOP_LIMIT_EXCEEDED` / `AI_BAD_RESPONSE` / `AI_VALIDATION_FAILED`)

### AC-4: 聊天输出摘要化
- **Given** model produces long content
- **When** response is sent to conversation panel
- **Then** output is `summary-only` and full content is kept in preview/diff channel

## Design Decisions (Frozen)

1. loop controller 只负责编排，不直接写业务实体。
2. tool loop 仅操作会话 overlay（stage/validate/discard），apply 只能手动触发。
3. 消息构造统一使用 `compose_messages(...)`。
4. tool result 统一写入 `role=tool`，保留 `tool_call_id/name`。

## Tasks / Subtasks

- [x] Task 1: 更新 tool allowlist（加 stage/discard，禁 apply）
- [x] Task 2: 对齐 `AI_TOOL_FORBIDDEN` 错误契约
- [x] Task 3: 对齐 `compose_messages` 多轮链路
- [x] Task 4: 补齐 loop 单测与集成测试

## Test Plan

- 多轮 tool loop 在上限内收敛
- 模型调用 `builder.change.apply` 返回 `AI_TOOL_FORBIDDEN`
- 同一 session 多轮 `stage/validate/discard` 连续生效
- 聊天消息仅输出摘要字段
