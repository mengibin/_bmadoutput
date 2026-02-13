# Story BAI-1.3: Implement LLM Profile Test API

Status: done

> Epic: `_bmad-output/epics-builder-ai.md`  
> Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-bai-1-user-llm-profile-foundation.md`

## Story

As a **Builder User**,  
I want to test provider connectivity before using AI,  
So that I can quickly fix endpoint/model/key issues.

## Acceptance Criteria

### AC-1: Test endpoint availability
- **Given** user authenticated
- **When** calling `POST /users/me/llm-profile/test`
- **Then** endpoint returns structured result (`ok/code/message/hints`)

### AC-2: Structured failure reasons
- **Given** provider is unreachable / unauthorized / timeout
- **When** test endpoint is called
- **Then** returns actionable reason and hints

### AC-3: 持久化健康状态
- **Given** test uses persisted profile
- **When** test succeeds or fails
- **Then** `health_status` and `last_tested_at` are updated

## Design Decisions (Frozen)

1. `provider` 仅允许 `disabled|openai-compatible`。
2. `provider=disabled` 不发起外部请求，返回受控不可用结果。
3. `openai-compatible` 使用最小 `chat/completions` 探测。

## Tasks / Subtasks

- [x] Task 1: 更新 test schema（去除 `mock`）
- [x] Task 2: 更新服务侧 provider 分支逻辑
- [x] Task 3: 更新错误码映射与 hints
- [x] Task 4: 补充 profile test API 回归测试

## Test Plan

- 未登录 `401`
- `provider=disabled` 返回受控结果（不发起外部探测）
- `provider=openai-compatible`：无效 key / 不可达 / 超时分别返回对应错误码
- 保存模式下更新 `last_tested_at/health_status`
