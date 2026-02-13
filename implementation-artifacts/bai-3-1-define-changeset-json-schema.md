# Story BAI-3.1: Define `changeSet` JSON Schema

Status: done

> Epic: `_bmad-output/epics-builder-ai.md`  
> Tech Spec: `_bmad-output/tech-spec/builder-ai-implementation-spec.md`  
> Architecture: `_bmad-output/architecture/builder-ai-llm-interaction-architecture.md`

## Story

As a **System**,  
I want a strict contract for AI suggestions,  
So that validation and apply can be deterministic.

## Acceptance Criteria

### AC-1: 非法 shape 可被拒绝
- **Given** LLM 返回不符合 `changeSet` 结构
- **When** suggestion 进入解析/校验流程
- **Then** 返回 `AI_BAD_RESPONSE`，并包含明确 `hints`（字段路径 + 期望类型）

### AC-2: 覆盖四类对象的统一变更表达
- **Given** workflow/step/agent/asset 的新增、修改、删除场景
- **When** 构造 `changeSet`
- **Then** 结构可统一表达对象级操作、目标定位与影响面摘要

### AC-3: 版本化与可扩展策略明确
- **Given** 后续 schema 演进需求
- **When** 新增字段或操作类型
- **Then** 通过 `schemaVersion` 与兼容规则保证向后兼容

## Technical Scope

- 定义 `changeSet` 核心 JSON Schema（含 `$id`、`schemaVersion`、`operations[]`、`impact`、`revisions`）。
- 定义后端 Pydantic/Typed schema 与 JSON Schema 导出能力。
- 在 AI suggestion 解析入口接入 schema 校验（仅校验，不执行 apply）。
- 定义统一错误映射：schema parse error -> `AI_BAD_RESPONSE`。

## Design Decisions (Frozen)

1. 统一操作模型冻结为 `operations[]`，每条操作显式包含 `op`, `targetType`, `target`, `payload`，禁止分散式 `steps/assets/agents` 并列写法作为主契约。
2. 删除操作最小必填为：`op="delete" + targetType + target`；删除操作禁止携带业务 `payload`。
3. 移动操作暂不纳入 BAI-3.1（避免过早引入路径重排复杂度），统一通过 `delete + upsert` 表达；`move` 预留到后续 minor 版本。
4. `impact` 最小字段冻结为：`objects[]`, `counts`, `riskFlags[]`, `requiresConfirmation`，用于前端预览和审批 gating。
5. `revisions` 采用顶层聚合表达，字段冻结为 `workflowRevision/agentsRevision/assetsRevision`；按 operation 内嵌 revision 在 BAI-3.4 再评估。
6. `schemaVersion` 使用 `major.minor` 字符串；当前冻结 `1.0`。major 不兼容升级直接拒绝，minor 兼容扩展仅允许进入 `extensions`。
7. Schema 默认 `additionalProperties: false`（严格模式）；仅 `extensions` 字段允许开放键值扩展。
8. 后端校验输出必须可映射为统一错误：`AI_BAD_RESPONSE` + `hints[{path, expected, actual}]`。

## Interface Freeze

### Frozen `changeSet` Envelope

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "crewagent.builder.ai.change-set.v1",
  "schemaVersion": "1.0",
  "targetType": "workflow|step|agent|asset",
  "mode": "create|optimize",
  "operations": [],
  "impact": {},
  "revisions": {
    "workflowRevision": 10,
    "agentsRevision": 3,
    "assetsRevision": 7
  },
  "extensions": {}
}
```

### Frozen Operation Contract

- 公共字段：
  - `op`: `upsert|delete`
  - `targetType`: `workflow|step|agent|asset`
  - `target`: object（按 `targetType` 的定位键）
  - `payload`: object（仅 `upsert` 必填）
  - `reason`: string（可选，供审计展示）
- `target` 最小定位：
  - workflow: `workflowId`
  - step: `workflowId + nodeId`
  - agent: `agentId`
  - asset: `path`

### Frozen `impact` Contract

```json
{
  "objects": ["step:step-01", "asset:assets/policies/a.md"],
  "counts": {
    "workflow": 0,
    "step": 1,
    "agent": 0,
    "asset": 1
  },
  "riskFlags": ["cross-object", "high-churn"],
  "requiresConfirmation": true
}
```

### Frozen Validator Contract

```python
validate_change_set_schema(
    *,
    raw_change_set: Mapping[str, Any],
) -> dict[str, Any]
```

返回结构：

```json
{
  "ok": false,
  "normalizedChangeSet": null,
  "errors": [
    {
      "code": "AI_BAD_RESPONSE",
      "path": "operations[0].target.workflowId",
      "expected": "integer",
      "actual": "null",
      "message": "workflowId is required for step target"
    }
  ]
}
```

### Frozen Error Codes

- `AI_BAD_RESPONSE`（shape/type/required 校验失败）
- `AI_CHANGESET_SCHEMA_UNSUPPORTED`（major version 不兼容）
- `AI_CHANGESET_CONTRACT_VIOLATION`（op 与 payload 组合非法）

### Frozen Compatibility Rules

1. 仅 `schemaVersion=1.x` 进入校验；major 不是 `1` 直接返回 `AI_CHANGESET_SCHEMA_UNSUPPORTED`。
2. `schemaVersion=1.x` 下，`extensions` 内新增字段允许透传；核心字段新增必须走 minor 升级并保持 backward compatible。
3. 旧版分散结构（`steps/assets/...`）不作为正式输出契约；若出现，先由 adapter 归一化为 `operations[]` 再进入 validator，并返回 deprecation warning。

## Tasks / Subtasks

- [x] Task 1: 定义 schema 草案与字段语义（AC: 1,2,3）
- [x] Task 2: 增加 schema 校验入口与错误映射（AC: 1）
- [x] Task 3: 增加契约测试（合法/非法/边界）并固化快照（AC: 1,2,3）

## Test Plan

- 合法 `changeSet`（四类对象混合操作）可通过 schema 校验。
- 缺失必填字段、类型错误、未知操作类型能返回 `AI_BAD_RESPONSE` + hint path。
- `schemaVersion` 变更时，旧版本兼容规则有明确测试覆盖。

## File List

- `crewagent-builder-backend/app/schemas/ai_changeset.py`
- `crewagent-builder-backend/app/services/change_set_schema_service.py`
- `crewagent-builder-backend/tests/test_change_set_schema_service.py`

## Design Completion

- 设计冻结项已完成，`BAI-3.1` 已满足 `ready-for-dev` 进入条件。

## Dev Agent Record

### Completion Notes

- 修复 schema 字符串字段的隐式 `str()` 转换问题：`nodeId/agentId/path/schemaVersion/reason` 现在要求正确字符串类型，不再接受 `None/int` 等并静默转换。
- 新增回归测试覆盖非字符串标识字段，确保返回 `AI_BAD_RESPONSE` 且可定位字段路径。

### Validation

- `cd crewagent-builder-backend && .venv/bin/ruff check app/schemas/ai_changeset.py app/services/change_set_schema_service.py tests/test_change_set_schema_service.py`
- `cd crewagent-builder-backend && .venv/bin/pytest -q tests/test_change_set_schema_service.py`
