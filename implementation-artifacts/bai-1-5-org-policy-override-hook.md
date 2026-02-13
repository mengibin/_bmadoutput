# Story BAI-1.5: Org Policy Override Hook

Status: done

> Epic: `_bmad-output/epics-builder-ai.md`  
> Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-bai-1-user-llm-profile-foundation.md`

## Story

As an **Admin Policy Layer**,  
I want to override user profile when organization policy forbids external providers,  
So that governance is enforceable.

## Acceptance Criteria

### AC-1: 策略覆盖生效
- **Given** org policy forbids external calls
- **When** user profile selects `openai-compatible`
- **Then** AI session is blocked with explicit reason

### AC-2: Profile API 可见有效可用性
- **Given** user requests profile
- **When** GET profile is called
- **Then** response includes effective availability derived from org policy

### AC-3: 统一策略实现入口
- **Given** AI call path and profile API path
- **When** policy check is executed
- **Then** both paths share the same resolver logic

## Interface Freeze

### Frozen Config Key
- `LLM_ORG_POLICY_MODE`

### Frozen Policy Modes
| Mode | disabled | openai-compatible |
|:---|:---:|:---:|
| `allow-all` | ✅ | ✅ |
| `disable-all` | ✅ | ❌ |
| `allow-openai-compatible-only` | ✅ | ✅ |

说明：`disabled` 始终允许，表示“不使用外部 LLM”。

## Tasks / Subtasks

- [x] Task 1: 更新 policy mode 定义（移除 `allow-mock-only`）
- [x] Task 2: 更新 profile API `effectiveAvailability` 逻辑
- [x] Task 3: 更新 AI gateway policy gate
- [x] Task 4: 补充 org policy 回归测试

## Test Plan

- `allow-openai-compatible-only` 下 openai-compatible 允许
- `disable-all` 下 openai-compatible 阻断
- GET profile 返回可用性状态与原因
