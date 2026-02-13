# Story BAI-1.1: Add `user_llm_profiles` Data Model

Status: done

> Epic: `_bmad-output/epics-builder-ai.md`  
> Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-bai-1-user-llm-profile-foundation.md`

## Story

As a **Platform Engineer**,  
I want to persist LLM profile by user,  
So that each user has isolated provider settings.

## Acceptance Criteria

### AC-1: 用户级 profile 持久化
- **Given** an authenticated user
- **When** profile row does not exist
- **Then** system can create default profile (`provider=disabled`) without affecting other users

### AC-2: 表结构字段完整
- **Then** key fields include:
  - `provider/base_url/model/api_key_encrypted/timeout_seconds/context_window`
  - `health_status/last_tested_at/created_at/updated_at`

### AC-3: 约束与索引
- **Then** `user_id` is unique
- **And** `provider` enum is restricted to `disabled|openai-compatible`
- **And** `health_status` enum is restricted to `unknown|ok|failed`

## Design Decisions (Frozen)

1. `user_llm_profiles` 与 `users` 使用 1:1 关系，`user_id` 唯一约束。
2. profile 懒创建：首次访问 profile API 时自动创建默认行（`provider=disabled`）。
3. `api_key` 仅保存密文（`api_key_encrypted`），不落库明文。
4. 当前语义移除 `mock` provider，不再写入或返回 `mock`。

## Interface Freeze

### Frozen DB Contract (`user_llm_profiles`)

| Column | Type | Null | Default | Notes |
|:---|:---|:---:|:---|:---|
| `id` | integer | N | pk | 自增主键 |
| `user_id` | integer | N | - | FK users.id, unique |
| `provider` | varchar(32) | N | `disabled` | `disabled|openai-compatible` |
| `base_url` | varchar(500) | Y | `null` | provider 相关 |
| `model` | varchar(200) | Y | `null` | provider 相关 |
| `api_key_encrypted` | text | Y | `null` | 仅密文 |
| `timeout_seconds` | integer | N | `60` | > 0 |
| `context_window` | integer | Y | `null` | 可选 |
| `health_status` | varchar(32) | N | `unknown` | `unknown|ok|failed` |
| `last_tested_at` | datetime(tz) | Y | `null` | 最后测试时间 |
| `created_at` | datetime(tz) | N | now | 创建时间 |
| `updated_at` | datetime(tz) | N | now | 更新时间 |

## Tasks / Subtasks

- [x] Task 1: 对齐 migration 与 enum（移除 `mock`）
- [x] Task 2: 对齐 ORM model 与 validator
- [x] Task 3: 对齐 profile service 默认值与兼容逻辑
- [x] Task 4: 补充 migration/ORM/service 回归测试

## Test Plan

- Migration 升级/回滚测试通过
- 首次 `GET /users/me/llm-profile` 自动创建 `provider=disabled`
- 任何 `provider=mock` 历史输入被拒绝或迁移为受控错误
