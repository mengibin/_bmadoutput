# Story BAI-4.2: Implement Conversation Pane

Status: done

> Epic: `_bmad-output/epics-builder-ai.md`  
> Tech Spec: `_bmad-output/tech-spec/builder-ai-implementation-spec.md`  
> Prototype: `_bmad-output/pencil-new-work.pen`

## Story

As a **Builder User**,  
I want multi-turn chat with status and retry,  
So that I can iteratively refine staged suggestions.

## Acceptance Criteria

### AC-1: 多轮会话
- **Given** user enters AI Workbench with `targetType/targetId/mode`
- **When** creating/restoring session and sending messages
- **Then** conversation shows ordered user/assistant messages with status

### AC-2: 统一会话接口
- **Given** send/retry actions
- **When** frontend calls backend
- **Then** it uses `/projects/{projectId}/ai/sessions` and `/projects/{projectId}/ai/sessions/{sessionId}/messages`
- **And** does not call legacy `step-draft` APIs

### AC-3: summary-only 输出
- **Given** assistant returns long content
- **When** rendering conversation
- **Then** pane shows summary-only fields (`assistantSummary`) and not full file text

### AC-4: 失败可恢复
- **Given** timeout/429/5xx
- **When** user retries
- **Then** same `sessionId` continues and preview context is preserved

## Tasks / Subtasks

- [x] Task 1: conversation pane 接口切换到 session/messages
- [x] Task 2: 去除 step-draft/stream 客户端依赖
- [x] Task 3: summary-only 渲染与类型契约对齐
- [x] Task 4: 补齐前端回归测试

## Test Plan

- send/retry 全部走 session/messages API
- 对话区不展示完整文件全文
- retry 不丢失 session 与 preview 状态
