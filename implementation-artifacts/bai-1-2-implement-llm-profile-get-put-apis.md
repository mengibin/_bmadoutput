# Story BAI-1.2: Implement LLM Profile GET/PUT APIs

Status: done

> Epic: `_bmad-output/epics-builder-ai.md`  
> Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-bai-1-user-llm-profile-foundation.md`

## Story

As a **Builder User**,  
I want to view and update my own LLM profile,  
So that AI Workbench reads my settings.

## Acceptance Criteria

### AC-1: GET `/users/me/llm-profile`
- **Given** user authenticated
- **When** GET endpoint is called
- **Then** returns current user's profile
- **And** if profile does not exist, creates default profile and returns it

### AC-2: PUT `/users/me/llm-profile`
- **Given** user authenticated
- **When** required fields are invalid for selected provider
- **Then** API returns actionable validation errors

### AC-3: 密钥不明文返回
- **Given** profile has api key
- **When** GET endpoint returns payload
- **Then** plaintext api key is never returned (masked only)

## Interface Freeze

### Frozen Endpoint
- `GET /users/me/llm-profile`
- `PUT /users/me/llm-profile`

### Frozen PUT Request Schema

```json
{
  "provider": "disabled|openai-compatible",
  "baseUrl": "string | null",
  "model": "string | null",
  "apiKey": "string | null",
  "timeoutSeconds": 60,
  "contextWindow": 100000
}
```

### Frozen Validation Matrix
- `provider=disabled`：`baseUrl/model/apiKey` 可为空
- `provider=openai-compatible`：`baseUrl/model` 必填，`timeoutSeconds>0`

## Tasks / Subtasks

- [x] Task 1: 更新 schema 契约（去除 `mock`）
- [x] Task 2: 更新 GET/PUT 输入校验与错误码
- [x] Task 3: 更新 API 文档与前端契约类型
- [x] Task 4: 补充 GET/PUT 回归测试

## Test Plan

- 未登录访问返回 `401`
- 首次 GET 自动创建默认 profile
- PUT 非法 provider / 缺 baseUrl / 缺 model / timeout<=0 返回结构化错误
- GET/PUT 不返回明文 key
