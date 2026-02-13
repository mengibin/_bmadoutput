---
project_name: "CrewAgent Builder AI"
date: "2026-02-08"
updated: "2026-02-10"
sources:
  - "_bmad-output/prd-builder-AI.md"
  - "_bmad-output/architecture/builder-ai-llm-interaction-architecture.md"
  - "crewagent-runtime/spec/PACKAGE_BUILDER_PROMPT.md"
  - "crewagent-runtime/spec/LLM_COMPLETE_SPEC.md"
---

# Technical Implementation Spec — Builder AI Workbench (v1)

## 0. Purpose

本规范用于指导 Builder AI 的实现落地，覆盖：

- 用户级 LLM Profile
- AI Session 与 ToolCall 循环
- Prompt Composer 分层
- DB 语义工具
- `changeSet` 暂存（stage）/校验（validate）/手动应用（manual apply）
- 统一前端 Workbench 接入

该文档是 `epics-builder-ai` 的工程实现契约。

执行与排期参考：
- `_bmad-output/implementation-artifacts/builder-ai-delivery-roadmap.md`
- `_bmad-output/implementation-artifacts/sprint-status-builder-ai.yaml`

## 1. Scope

### In Scope

- `Workflow/Step/Agent/Asset` 四类入口统一 AI Workbench
- `create/optimize` 双模式
- 用户级 provider/baseUrl/model/apiKey 配置
- 后端三阶段写入语义：`stage -> validate -> manual apply`
- revision 安全契约（single-writer MVP：warning-first + recoverability）

### Out of Scope

- 组织策略中心 UI（仅保留 API 覆盖点）
- 全自动批量重构（无需确认）
- 自动 apply（模型不可直接触发）
- 数据库级并发原子锁定（后续版本）
- Runtime 引擎改造

## 2. Current Codebase Mapping

### Backend

- Router: `crewagent-builder-backend/app/routers/packages.py`
- Step AI service: `crewagent-builder-backend/app/services/ai_step_service.py`
- Package services:
  - `crewagent-builder-backend/app/services/package_service.py`
  - `crewagent-builder-backend/app/services/package_workflow_service.py`

### Data Model

- `workflow_packages`:
  - `agents_json`, `assets_json`, `artifacts_json`
- `package_workflows`:
  - `workflow_md`, `graph_json`, `step_files_json`

### Frontend

- Workflow editor current Step AI:
  - `crewagent-builder-frontend/src/app/editor/[projectId]/[workflowId]/page.tsx`
- Profile page:
  - `crewagent-builder-frontend/src/app/profile/page.tsx`

## 3. Target Module Architecture

### Backend Modules (New)

1. `app/models/user_llm_profile.py`
2. `app/models/ai_session.py`
3. `app/models/ai_change_set.py`
4. `app/schemas/ai_session.py`
5. `app/services/ai_session_service.py`
6. `app/services/prompt_composer_service.py`
7. `app/services/semantic_tool_service.py`
8. `app/services/change_set_service.py`
9. `app/routers/ai_sessions.py`
10. `app/routers/llm_profile.py`

### Frontend Modules (New)

1. `src/app/builder/[projectId]/ai-workbench/page.tsx`
2. `src/components/ai/workbench-shell.tsx`
3. `src/components/ai/conversation-pane.tsx`
4. `src/components/ai/change-preview-pane.tsx`
5. `src/lib/ai-workbench-client.ts`

## 4. Data Model Design

### 4.1 `user_llm_profiles`

Fields:

- `id` (pk)
- `user_id` (fk users.id, unique)
- `provider` (`disabled|openai-compatible`)
- `base_url` (nullable)
- `model` (nullable)
- `api_key_encrypted` (nullable)
- `timeout_seconds` (default 60)
- `context_window` (nullable)
- `health_status` (`unknown|ok|failed`)
- `last_tested_at` (nullable)
- `created_at`, `updated_at`

### 4.2 `ai_sessions`

Fields:

- `id` (pk)
- `project_id`
- `workflow_id` (nullable)
- `user_id`
- `target_type` (`workflow|step|agent|asset`)
- `target_id`
- `mode` (`create|optimize`)
- `status` (`active|cancelled|failed`)
- `context_envelope_json`
- `latest_suggestion_json` (nullable)
- `created_at`, `updated_at`

说明：

- `apply` 成功后会话保持 `active`，便于继续同会话多轮优化。
- `applied` 是 changeSet 状态（`ai_change_sets.status`），不是 session 状态。

### 4.3 `ai_messages`

Fields:

- `id` (pk)
- `session_id` (fk)
- `role` (`system|user|assistant|tool`)
- `content`
- `tool_name` (nullable)
- `tool_args_json` (nullable)
- `tool_result_json` (nullable)
- `created_at`

### 4.4 `ai_change_sets`

Fields:

- `id` (pk)
- `session_id` (fk)
- `suggestion_version` (int)
- `change_set_json`
- `validation_result_json`
- `status` (`staged|validated|applied|rejected`)
- `created_at`, `updated_at`

状态流转：

- `staged -> validated -> applied`
- `staged/validated -> rejected`（校验失败或用户明确放弃）

### 4.5 `ai_session_working_files`

Fields:

- `id` (pk)
- `session_id` (fk ai_sessions.id)
- `owner_user_id` (fk users.id)
- `project_id`
- `workflow_id` (nullable)
- `object_type` (`workflow|step|agent|asset`)
- `object_key` (string, e.g. `step:step-collect` / `asset:assets/policies/a.md`)
- `content_json` (effective overlay payload)
- `deleted` (bool, default false)
- `base_hash` (nullable, optimistic snapshot marker)
- `updated_at`

Behavior:

- all read/write tools use effective view (`working copy -> DB baseline`)
- no business mutation before manual apply
- apply success or cancel resets working copy for current session

### 4.6 Revision Fields (Existing Tables)

- `package_workflows.revision` (int, default 1)
- `workflow_packages.agents_revision` (int, default 1)
- `workflow_packages.assets_revision` (int, default 1)

## 5. API Contract

## 5.1 LLM Profile APIs

### GET `/users/me/llm-profile`

Response:

```json
{
  "data": {
    "provider": "openai-compatible",
    "baseUrl": "https://api.example.com",
    "model": "gpt-4o-mini",
    "apiKeyMasked": "sk-***",
    "timeoutSeconds": 60,
    "contextWindow": 100000,
    "healthStatus": "ok",
    "lastTestedAt": "2026-02-08T12:00:00Z"
  },
  "error": null
}
```

### PUT `/users/me/llm-profile`

Request:

```json
{
  "provider": "openai-compatible",
  "baseUrl": "https://api.example.com",
  "model": "gpt-4o-mini",
  "apiKey": "sk-xxx",
  "timeoutSeconds": 60,
  "contextWindow": 100000
}
```

### POST `/users/me/llm-profile/test`

Request:

```json
{
  "provider": "openai-compatible",
  "baseUrl": "https://api.example.com",
  "model": "gpt-4o-mini",
  "apiKey": "sk-xxx",
  "timeoutSeconds": 60
}
```

Response:

```json
{
  "data": {
    "ok": false,
    "code": "AI_PROVIDER_ERROR",
    "message": "401 unauthorized",
    "hints": ["检查 API Key 是否有效", "确认 endpoint 与 model 匹配"]
  },
  "error": null
}
```

## 5.2 AI Session APIs

### POST `/projects/{projectId}/ai/sessions`

Request:

```json
{
  "workflowId": 12,
  "targetType": "step",
  "targetId": "step-01",
  "mode": "optimize",
  "userPrompt": "补齐 completion 并关联 policy"
}
```

Response:

```json
{
  "data": {
    "sessionId": "as_123",
    "status": "active",
    "contextEnvelope": {},
    "latestSuggestion": null
  },
  "error": null
}
```

### GET `/projects/{projectId}/ai/sessions/{sessionId}`

Response includes:

- session metadata
- latest suggestion
- validation state
- messages (paginated)

### POST `/projects/{projectId}/ai/sessions/{sessionId}/messages`

Request:

```json
{
  "content": "不要新增文件，只优化现有 step"
}
```

Response:

```json
{
  "data": {
    "assistantSummary": "建议更新了 2 个 step 和 1 个 asset 引用（可先预览差异后再 apply）",
    "latestSuggestion": {},
    "validation": {}
  },
  "error": null
}
```

说明：

- 聊天面板仅返回 `summary-only` 信息（`assistantSummary`）。
- 变更全文、逐字段 diff 仅在 Preview/Diff 面板展示，不回填到对话正文。
- Step 场景 Prompt 默认只注入 `referenced_asset_paths`，不内联 asset 正文；正文需由 `builder.asset.read` 按需读取。
- 多轮历史上下文采用“最近 6 条原文 + 更早 1 条摘要（本地规则）”。

### POST `/projects/{projectId}/ai/sessions/{sessionId}/apply`

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

Response:

```json
{
  "data": {
    "applied": true,
    "sessionStatus": "active",
    "newRevision": {
      "workflowRevision": 11,
      "agentsRevision": 3,
      "assetsRevision": 8
    },
    "warnings": []
  },
  "error": null
}
```

Apply semantics:

- apply commits transaction + bumps revision + marks changeSet `applied`
- working copy is reset to new baseline
- session remains `active` (conversation continues)

Revision mismatch（warning-first）response:

```json
{
  "data": {
    "applied": true,
    "newRevision": {
      "workflowRevision": 11,
      "agentsRevision": 4,
      "assetsRevision": 8
    },
    "warnings": [
      {
        "code": "AI_REVISION_BASE_MISMATCH",
        "message": "workflowRevision mismatch",
        "field": "workflowRevision",
        "provided": 10,
        "current": 11,
        "blocking": false
      }
    ]
  }
}
```

### POST `/projects/{projectId}/ai/sessions/{sessionId}/cancel`

Cancels current session and discards unsubmitted overlay.

- No workflow/step/agent/asset business mutation.
- Session status becomes `cancelled`.
- Session working copy is reset (overlay discarded).

## 6. Semantic Tool Contract (for LLM)

## 6.1 Read Tools

### `builder.context.get`

Input:

```json
{ "sessionId": "as_123" }
```

Output:

```json
{
  "session_meta": {},
  "domain_profile": {},
  "normative_baseline": {},
  "target_snapshot": {},
  "dependency_digest": {},
  "tool_capabilities": {},
  "revision_info": {}
}
```

### `builder.step.read`

Input:

```json
{ "projectId": 1, "workflowId": 2, "nodeId": "step-01" }
```

Output:

```json
{
  "nodeId": "step-01",
  "markdown": "...",
  "frontmatter": {},
  "revision": 10
}
```

Read semantics:

- return effective object view with working copy precedence
- fallback to persisted DB baseline when no overlay exists

### `builder.refs.find`

Input:

```json
{
  "projectId": 1,
  "locator": { "type": "asset", "id": "assets/policies/a.md" }
}
```

Output:

```json
{
  "inbound": [{ "type": "step", "workflowId": 2, "nodeId": "step-03" }],
  "outbound": []
}
```

## 6.2 Staging/Validation/Manual Apply Tools

### `builder.change.stage`

Input:

```json
{ "sessionId": "as_123", "changeSet": {} }
```

Output:

```json
{
  "changeSetId": "cs_456",
  "status": "staged",
  "warnings": []
}
```

### `builder.change.validate`

Input:

```json
{ "sessionId": "as_123", "changeSetId": "cs_456" }
```

Output:

```json
{
  "valid": false,
  "errors": [
    { "code": "AGENT_NOT_FOUND", "message": "agentId=xxx not found", "path": "changeSet.steps[0].agentId" }
  ],
  "warnings": [],
  "status": "rejected",
  "meta": { "allowWrite": false }
}
```

### `builder.change.apply`

Input:

```json
{
  "sessionId": "as_123",
  "changeSetId": "cs_456",
  "confirmSource": "ui_manual_apply",
  "revisionBase": {}
}
```

Output:

```json
{
  "applied": true,
  "newRevision": {},
  "warnings": []
}
```

### `builder.change.discard`

Input:

```json
{ "sessionId": "as_123", "changeSetId": "cs_456" }
```

Output:

```json
{
  "discarded": true
}
```

Tool loop policy:

- Model-exposed whitelist: read tools + `builder.change.stage` + `builder.change.validate` + `builder.change.discard`
- `builder.change.apply` is UI manual action only; model-side call must return `AI_TOOL_FORBIDDEN`

## 7. Prompt Composer Spec

## 7.1 Layer Order

1. `Layer-0A Core Identity`
2. `Layer-0B Domain Persona`
3. `Layer-1 Global Guardrails`
4. `Layer-2 Output Contract`
5. `Layer-3 Entry Strategy`
6. `Layer-4 Mode Strategy`
7. `Layer-5 Context Digest`
8. `Layer-6 Apply Policy`

## 7.2 Compose Inputs

- targetType / mode / user prompt
- context envelope
- domain profile
- org policy
- tool capability profile
- history window (`latest 6 raw turns + 1 older-summary`)
- referenced asset paths (content resolved by tool read, not inlined)

## 7.3 Compose Output

`OpenAI-compatible messages[]`, where system messages can be merged before provider call if needed.

## 8. Validation Pipeline Spec

Pipeline stages:

1. JSON parsing and shape validation
2. BMAD constraints validation
3. reference integrity validation
4. impact boundary validation
5. security/path validation
6. revision comparison (apply only, warning-first in single-writer MVP)

Failure handling:

- return structured error with hints
- optional repair loop (max 2) in session mode

## 9. Apply Orchestrator Spec

Algorithm:

1. load session + validated changeSet overlay
2. verify manual confirm source (`confirmSource=ui_manual_apply`)
3. compare `revisionBase` with current revisions and collect `AI_REVISION_BASE_MISMATCH` warnings
4. begin DB transaction
5. apply workflow/step/agents/assets mutations
6. bump revisions
7. persist audit event
8. commit transaction
9. reset session working copy overlay
10. keep session `active`

If any step fails:

- rollback
- return `AI_APPLY_FAILED` with root cause

## 10. Frontend Workbench Spec

## 10.1 Route and State

Route:

- `/builder/[projectId]/ai-workbench`

Query:

- `workflowId`
- `targetType`
- `targetId`
- `mode`
- `source` (optional return page marker)

Client state:

- session metadata
- message list
- latest suggestion
- validation status
- apply state

## 10.2 UX Behavior

- entry from four surfaces uses same page
- explicit target badge + mode badge
- conversation pane is summary-only (no full file payload in chat body)
- preview and impact always visible
- apply disabled until validation passes
- cancel returns without mutation

## 11. Security and Observability

Security:

- encrypt/decrypt api key only server side
- redact secret fields in logs
- enforce user/project authorization on every tool

Observability:

- `ai.session.created`
- `ai.message.sent`
- `ai.tool.called`
- `ai.validation.failed`
- `ai.apply.succeeded` / `ai.apply.failed`
- latency, revision-mismatch warning, and tool-forbidden metrics

## 12. Testing Strategy

### 12.1 Backend Tests

- unit: prompt composer layering
- unit: changeSet validator rules
- integration: session create/message/apply
- integration: revision mismatch returns warning and remains recoverable
- integration: model-side `builder.change.apply` returns `AI_TOOL_FORBIDDEN`
- integration: cancel discards overlay and does not mutate workflow/step/agent/asset
- security: key masking/redaction

### 12.2 Frontend Tests

- route restoration
- preview rendering for four target types
- apply/cancel behavior
- revision warning handling UX
- summary-only conversation rendering

### 12.3 E2E Scenarios

1. Step optimize + asset upsert + apply
2. Workflow create + validate + apply
3. Agent optimize with unchanged agentId
4. Asset optimize with reference impact

## 13. Rollout Plan

Phase A:

- backend profile + session skeleton + validate-only

Phase B:

- apply orchestrator + workbench UI (step/workflow first)

Phase C:

- agent/asset entrypoints + observability + canary

Phase D:

- clean up legacy `ai/step-draft` references (deprecated/decommissioned)
