# Story BAI-3.2: Implement `builder.change.validate`

Status: done

> Epic: `_bmad-output/epics-builder-ai.md`  
> Tech Spec: `_bmad-output/tech-spec/builder-ai-implementation-spec.md`  
> Architecture: `_bmad-output/architecture/builder-ai-llm-interaction-architecture.md`

## Story

As a **System**,  
I want to validate staged changeSet overlay with schema + BMAD rules + impact boundaries,  
So that unsafe changes are blocked before manual apply.

## Acceptance Criteria

### AC-1: validate 输入基于 staged 记录
- **Given** a staged change set exists in session overlay
- **When** calling `builder.change.validate`
- **Then** validate reads staged payload by `sessionId + changeSetId`
- **And** returns `valid/errors/warnings` with stable structure

### AC-2: schema-first short circuit
- **Given** schema validation fails
- **When** validation pipeline runs
- **Then** BMAD/reference/impact/security phases do not execute
- **And** response includes `schemaFailed=true`

### AC-3: Validate-Only（不写业务对象）
- **Given** any validate request
- **When** `builder.change.validate` completes
- **Then** workflow/step/agent/asset business entities are not mutated
- **And** metadata write to `ai_change_sets.validation_result_json` is allowed
- **And** `meta.allowWrite=false`

### AC-4: 错误码可定位
- **Given** reference/scope/security violations
- **When** validation fails
- **Then** errors include stable `code/message/path/hints`

## Interface Freeze

### Tool Contract

```json
{
  "name": "builder.change.validate",
  "input": {
    "sessionId": "as_123",
    "changeSetId": "cs_456"
  },
  "output": {
    "valid": false,
    "schemaFailed": false,
    "errors": [
      {
        "code": "AGENT_NOT_FOUND",
        "message": "agentId=xxx not found",
        "path": "operations[0].payload.agentId",
        "hints": ["请使用已存在的 agentId"]
      }
    ],
    "warnings": [],
    "meta": {
      "allowWrite": false
    }
  }
}
```

## Design Decisions (Frozen)

1. validate 只消费 staged change set，不再接收模型直接传入 raw `changeSet` 作为主路径。
2. pipeline 顺序固定：`schema -> BMAD -> reference -> impact -> security/path`。
3. `revision` 比对不在本 Story 处理（归属 BAI-3.3/BAI-3.4）。
4. validate 结果持久化到 `ai_change_sets` 元数据，供 apply gate 使用。

## Tasks / Subtasks

- [x] Task 1: 改造 validate 输入契约为 `sessionId + changeSetId`
- [x] Task 2: 对齐 schema-first pipeline 与短路行为
- [x] Task 3: 对齐 `allowWrite=false` 语义与元数据写入说明
- [x] Task 4: 更新 semantic tool contract 与测试

## Test Plan

- 无 staged change set 时返回 `AI_NO_STAGED_CHANGES`
- schema 失败时 `schemaFailed=true` 且后续阶段不执行
- reference/scope/security 错误返回可定位 path
- validate 之后业务 revision 不变（允许 `ai_change_sets` 元数据更新）
