# Story BAI-3.3: Implement Manual `apply` with Transaction

Status: done

> Epic: `_bmad-output/epics-builder-ai.md`  
> Tech Spec: `_bmad-output/tech-spec/builder-ai-implementation-spec.md`  
> Architecture: `_bmad-output/architecture/builder-ai-llm-interaction-architecture.md`

## Story

As a **System**,  
I want all user-confirmed validated staged changes committed atomically,  
So that partial writes are avoided.

## Acceptance Criteria

### AC-1: 单事务原子提交
- **Given** changeSet touches multiple object types
- **When** manual apply runs
- **Then** all operations commit in one transaction
- **And** any failure causes full rollback

### AC-2: 手动确认门禁
- **Given** apply is requested
- **When** `confirmSource != ui_manual_apply`
- **Then** apply is rejected with `AI_APPLY_REQUIRES_CONFIRMATION`

### AC-3: revision mismatch warning-first
- **Given** `revisionBase` differs from current revision
- **When** apply runs in single-writer MVP
- **Then** apply continues and returns `warnings[].code=AI_REVISION_BASE_MISMATCH`
- **And** no HTTP 409 hard block in current version

### AC-4: workflow delete 禁用
- **Given** operation is `targetType=workflow` + `op=delete`
- **When** apply runs
- **Then** reject with `AI_APPLY_UNSUPPORTED`

## Interface Freeze

### HTTP Apply Contract
- `POST /projects/{projectId}/ai/sessions/{sessionId}/apply`

Request:

```json
{
  "changeSetId": "cs_456",
  "confirmSource": "ui_manual_apply",
  "revisionBase": {
    "workflowRevision": 10,
    "agentsRevision": 3,
    "assetsRevision": 7
  }
}
```

Response (mismatch example):

```json
{
  "data": {
    "applied": true,
    "warnings": [
      {
        "code": "AI_REVISION_BASE_MISMATCH",
        "field": "workflowRevision",
        "provided": 10,
        "current": 11,
        "blocking": false
      }
    ]
  },
  "error": null
}
```

## Design Decisions (Frozen)

1. apply 不向模型 loop 暴露，只能走 UI 手动确认。
2. apply 输入固定为 `sessionId + changeSetId + confirmSource + revisionBase?`。
3. apply 前重验 validated change set；无效即拒绝。
4. revision mismatch 当前为 warning-first，可恢复。

## Tasks / Subtasks

- [x] Task 1: 增加 `confirmSource` 门禁校验
- [x] Task 2: revision mismatch 从 conflict 改为 warning-first
- [x] Task 3: 增加 workflow delete 防御拒绝
- [x] Task 4: 更新 apply 契约测试（事务、回滚、warning）

## Test Plan

- valid + manual confirm -> apply success
- confirmSource 缺失/非法 -> `AI_APPLY_REQUIRES_CONFIRMATION`
- revision mismatch -> applied=true + warnings
- workflow delete -> `AI_APPLY_UNSUPPORTED`
- 中途失败 -> 全量回滚

## Post-Completion Stabilization (2026-02-11)

### Additional Fixes Delivered

1. Step apply 路径一致性：
   - Step upsert/apply 不再依赖固定 `steps/{nodeId}.md`。
   - 优先使用 `payload.stepPath`，并兼容 graph `node.file` 与现存 step 文件匹配路径。
   - 对同 `nodeId` 的重复 step 文件执行去重，避免 apply 后路径漂移。
2. 生成 frontmatter 的 YAML 安全性：
   - 修复 list 字段序列化语法（`inputs/outputs/setsVariables` 采用合法 YAML 结构）。
   - 所有字符串字段统一安全转义，避免双引号/冒号污染 frontmatter。

### Regression Verification

- `tests/test_change_set_apply_service.py`：apply 路径匹配与去重回归通过。
- `tests/test_ai_step_draft.py`：frontmatter 转义与可解析性回归通过。
