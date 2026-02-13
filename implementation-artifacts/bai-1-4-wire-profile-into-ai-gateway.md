# Story BAI-1.4: Wire Profile into AI Gateway

Status: done

> Epic: `_bmad-output/epics-builder-ai.md`  
> Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-bai-1-user-llm-profile-foundation.md`

## Story

As a **System**,  
I want AI calls to resolve provider config from user profile,  
So that provider behavior is user-scoped in the AI Workbench session flow.

## Acceptance Criteria

### AC-1: 用户级配置生效
- **Given** two users with different profile configs
- **When** each calls AI Workbench session/message APIs
- **Then** requests use their own provider settings

### AC-2: 新链路统一接入
- **Given** AI request enters from Workbench flow
- **When** backend resolves provider
- **Then** provider resolution is applied in unified session/tool-loop gateway
- **And** no business path depends on legacy `step-draft`

### AC-3: 禁用配置阻断
- **Given** user profile provider is `disabled`
- **When** AI request is sent
- **Then** request is blocked with explicit error code

## Interface Freeze

### Frozen Service Contract

```python
resolve_effective_llm_config(
    db: Session,
    settings: Settings,
    *,
    user_id: int,
) -> ResolvedLlmConfig
```

### Frozen `ResolvedLlmConfig` Shape
- `provider: "disabled" | "openai-compatible"`
- `base_url: str | None`
- `model: str | None`
- `api_key: str | None`（仅内存使用）
- `timeout_seconds: int`
- `blocked: bool`
- `block_code: str | None`
- `block_message: str | None`

## Tasks / Subtasks

- [x] Task 1: 统一 session/message gateway 的 profile 解析入口
- [x] Task 2: 删除 `step-draft` 作为 profile 生效主路径的文档与依赖
- [x] Task 3: 更新 resolver 的 provider 枚举（去除 `mock`）
- [x] Task 4: 补充“多用户隔离 + disabled 阻断”集成测试

## Test Plan

- 两用户分别调用 Workbench 会话接口，provider 与行为隔离
- `provider=disabled` 返回明确阻断错误
- legacy `step-draft` 不再作为 profile 生效验收路径
