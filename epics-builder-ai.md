---
stepsCompleted: [1, 2, 3, 9]
inputDocuments:
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/prd-builder-AI.md'
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/architecture/builder-ai-llm-interaction-architecture.md'
workflowType: 'epics'
lastStep: 9
project_name: 'CrewAgent Builder AI'
user_name: 'Mengbin'
date: '2026-02-08'
---

# CrewAgent Builder AI - Epic Breakdown

## Overview

本文件将 `prd-builder-AI` 与 `builder-ai-llm-interaction-architecture` 拆解为可落地的 Epic/Story，覆盖：

- 统一 AI Workbench
- 用户级 LLM 配置
- ToolCall 循环
- 语义工具（DB 语义）
- changeSet 三阶段提交（stage/validate/manual apply）
- 观测与治理

执行顺序与门禁见：`_bmad-output/implementation-artifacts/builder-ai-delivery-roadmap.md`  
状态跟踪见：`_bmad-output/implementation-artifacts/sprint-status-builder-ai.yaml`

## Requirements Inventory（Builder AI）

### Functional Requirements

- FR15-FR17：用户级 LLM Profile 配置与连通性测试
- FR18-FR22b, FR36-FR38：统一 AI 工作台（会话 + 预览 + 应用）
- FR23-FR27：入口化上下文与 Prompt 组装
- FR32-FR35：Workflow Create/Optimize
- FR1-FR14：Step Create/Optimize + assets patch
- FR28-FR31：Agent Create/Optimize

### Non-Functional Requirements

- NFR3-NFR5b：安全、最小上下文、用户密钥隔离
- NFR6-NFR8：可用性、可靠性、可观测
- NFR9-NFR13：统一体验、无损返回、配置效率

## FR Coverage Map

| FR/NFR | Epic |
|:---|:---|
| FR15-FR17, NFR5b, NFR12 | Epic BAI-1 |
| FR23-FR27, FR36-FR38, NFR5 | Epic BAI-2 |
| FR1-FR14, FR28-FR35, NFR6-NFR7 | Epic BAI-3 |
| FR18-FR22b, NFR9-NFR11, NFR13 | Epic BAI-4 |
| NFR3-NFR4, NFR8 + rollout concerns | Epic BAI-5 |

## Epic List

### Epic BAI-1: User-Level LLM Profile Foundation

**Goal**: 完成用户级 LLM 配置持久化、读写和连通性验证，替代全局环境变量直读。  
**FRs covered**: FR15, FR16, FR17, NFR5b, NFR12
**Implementation Spec**: `_bmad-output/implementation-artifacts/tech-spec-bai-1-user-llm-profile-foundation.md`
**Status Snapshot (2026-02-11)**: `done`（与 `_bmad-output/implementation-artifacts/sprint-status-builder-ai.yaml` 对齐）

**Semantic Freeze (BAI-1)**
- provider 枚举：`disabled | openai-compatible`（不再使用 `mock`）
- org policy：`allow-all | disable-all | allow-openai-compatible-only`
- profile 生效主链路：`/packages/{projectId}/ai/sessions/{sessionId}/messages`
- legacy `step-draft` 不再作为 BAI-1 验收路径

**Stories**

### Story BAI-1.1: Add `user_llm_profiles` Data Model

As a **Platform Engineer**,  
I want to persist LLM profile by user,  
So that each user can use independent provider settings.

**Acceptance Criteria**

**Given** an authenticated user  
**When** profile row does not exist  
**Then** system can create default profile (`provider=disabled`) without affecting other users.

**And** key fields include `provider/base_url/model/api_key_encrypted/timeout_seconds/context_window`.

### Story BAI-1.2: Implement Profile APIs (`GET/PUT`)

As a **Builder User**,  
I want to view and update my own LLM profile,  
So that AI Workbench reads my settings.

**Acceptance Criteria**

**Given** `PUT /users/me/llm-profile`  
**When** required fields are invalid for selected provider  
**Then** API returns actionable validation errors.

**And** `GET` never returns plaintext api key (masked only).

### Story BAI-1.3: Implement Profile Connection Test API

As a **Builder User**,  
I want to test provider connectivity before using AI,  
So that I can quickly fix endpoint/model/key issues.

**Acceptance Criteria**

**Given** `POST /users/me/llm-profile/test`  
**When** provider is unreachable or unauthorized  
**Then** API returns a structured failure reason (`code/message/hints`).

### Story BAI-1.4: Wire Profile into AI Gateway

As a **System**,  
I want AI calls to resolve provider config from user profile,  
So that provider behavior is user-scoped.

**Acceptance Criteria**

**Given** two users with different profile configs  
**When** each calls AI session  
**Then** requests use their own provider settings.

### Story BAI-1.5: Org Policy Override Hook

As an **Admin Policy Layer**,  
I want to override user profile when organization policy forbids external providers,  
So that governance is enforceable.

**Acceptance Criteria**

**Given** org policy forbids external calls  
**When** user profile selects openai-compatible  
**Then** AI session is blocked with explicit reason.

---

### Epic BAI-2: Prompt Composer + Context Builder + Semantic Tool Host

**Goal**: 实现 ToolCall 循环底座、分层 Prompt 组装、初始上下文包和语义工具（含临时草稿 stage）。  
**FRs covered**: FR23-FR27, FR36-FR38, NFR5
**Implementation Spec**: `_bmad-output/implementation-artifacts/tech-spec-bai-2-prompt-composer-context-tool-loop.md`
**Status Snapshot (2026-02-11)**: `done`（与 `_bmad-output/implementation-artifacts/sprint-status-builder-ai.yaml` 对齐）

**Semantic Freeze (BAI-2)**
- Prompt Composer 统一使用分层 `compose_messages`，并包含 schema-first guardrails（required/type/enum/no extra fields）。
- Tool loop 为 function-call 多轮执行（非单次调用）。
- 模型侧工具白名单仅允许 `read + builder.change.stage + builder.change.validate + builder.change.discard`，`builder.change.apply` 禁止暴露。
- 上下文预算策略：非目标对象降级为 `path + summary`，目标对象内容保持完整。
- Step 场景 Prompt：asset 正文不内联，只给 `referenced_asset_paths`，内容通过 `builder.asset.read` 按需读取。
- 历史上下文采用“最近 6 条原文 + 更早 1 条摘要”。

**Stories**

### Story BAI-2.1: Build Prompt Composer Layering Engine

As a **System**,  
I want to assemble system instructions by layers,  
So that entry/mode/domain variations are controlled and testable.

**Acceptance Criteria**

**Given** different targetType/mode inputs  
**When** composing prompts  
**Then** output includes `Layer-0A` + `Layer-0B` + Guardrails + Contract + Entry + Mode + Context + ApplyPolicy.

**And** Guardrails 明确要求 “schema-first 自检” 后再输出结构化建议（required/type/enum/no extra fields）。

### Story BAI-2.2: Build Domain Persona Resolver

As a **System**,  
I want domain persona to be dynamic per project/workflow context,  
So that AI role is not a generic builder assistant.

**Acceptance Criteria**

**Given** project has domain metadata  
**When** creating a session  
**Then** Layer-0B persona reflects industry/target users/pain points.

### Story BAI-2.3: Build Initial Context Envelope

As a **System**,  
I want a standard initial context envelope,  
So that all entrypoints share deterministic context structure.

**Acceptance Criteria**

**Given** targetType in {workflow, step, agent, asset}  
**When** session starts  
**Then** envelope includes `session_meta/domain_profile/normative_baseline/target_snapshot/dependency_digest/tool_capabilities/revision_info`.

### Story BAI-2.4: Implement Semantic Read Tools (`builder.*`)

As a **System**,  
I want DB-semantic read tools instead of raw fs writes,  
So that tool calls are aligned with Builder storage model.

**Acceptance Criteria**

**Given** tool call `builder.step.read`  
**When** node exists and user authorized  
**Then** tool returns step snapshot with metadata and revision.

### Story BAI-2.5: Implement ToolCall Loop Controller

As a **System**,  
I want LLM-tool loop execution with stop conditions and repair loop,  
So that suggestions are generated from evidence, not guesswork.

**Acceptance Criteria**

**Given** LLM returns tool calls  
**When** tool results are appended  
**Then** loop continues until no tool calls or max iterations reached.

**And** loop 仅允许读工具与 `stage/validate/discard`，`apply` 不暴露给模型自动调用。

### Story BAI-2.6: Context Budget + Trimming Policy

As a **System**,  
I want predictable context budgeting,  
So that request size remains stable and controllable.

**Acceptance Criteria**

**Given** oversized context  
**When** composing request  
**Then** non-target content is reduced to path + summary while target content remains intact.

---

### Epic BAI-3: changeSet Contract, Validation, Staging, and Atomic Apply

**Goal**: 建立“先暂存后提交”的后端写入闭环，支持跨对象影响面控制。  
**FRs covered**: FR1-FR14, FR28-FR35, NFR6, NFR7
**Status Snapshot (2026-02-11)**: `done`（与 `_bmad-output/implementation-artifacts/sprint-status-builder-ai.yaml` 对齐）

**Semantic Freeze (BAI-3)**
- 写入闭环固定为 `stage -> validate -> manual apply`。
- `builder.change.validate` 为 validate-only：不写 workflow/step/agent/asset 业务对象（允许变更集元数据更新）。
- `builder.change.apply` 必须由 UI 手动触发（`confirmSource=ui_manual_apply`），模型侧不可调用。
- single-writer MVP 下 revision 不一致走 warning-first：`AI_REVISION_BASE_MISMATCH`，保证可恢复。
- workflow delete 在 AI apply 链路禁用（validate/apply 均受控拦截）。
- apply 成功后进行事务提交与 revision 更新，同时重置 working copy，不关闭会话（session 保持 active）。

**Stories**

### Story BAI-3.1: Define `changeSet` JSON Schema

As a **System**,  
I want a strict contract for AI suggestions,  
So that validation and apply can be deterministic.

**Acceptance Criteria**

**Given** invalid shape response  
**When** parsing suggestion  
**Then** API rejects with `AI_BAD_RESPONSE` + hints.

### Story BAI-3.2: Implement `builder.change.validate`

As a **System**,  
I want to validate staged changeSet with schema + BMAD rules + impact boundaries,  
So that unsafe changes are blocked before apply.

**Acceptance Criteria**

**Given** step references nonexistent agentId  
**When** validating changes  
**Then** validation fails with explicit reference error.

### Story BAI-3.3: Implement `builder.change.apply` with Transaction

As a **System**,  
I want all user-confirmed staged changes committed atomically,  
So that partial writes are avoided.

**Acceptance Criteria**

**Given** changeSet touching step + asset  
**When** apply runs  
**Then** both succeed or both rollback.

### Story BAI-3.4: Add Revision Safety Contract (Single-Writer MVP)

As a **System**,  
I want revision mismatch to be visible and recoverable in workbench flow,  
So that stale preview risk is explicit before/after manual apply.

**Acceptance Criteria**

**Given** revision mismatch in single-writer MVP  
**When** applying changeSet  
**Then** API returns structured mismatch warning/details and UI shows recovery actions without breaking current page flow.

### Story BAI-3.5: Decommission Legacy `ai/step-draft` Optimizer

As a **System**,  
I want to retire old step-draft optimizer path,  
So that Step AI generation uses one unified Workbench flow.

**Acceptance Criteria**

**Given** any legacy `ai/step-draft` call  
**When** request reaches backend  
**Then** API returns structured deprecation error and no optimizer mutation path is executed.

---

### Epic BAI-4: Unified AI Workbench Frontend Integration

**Goal**: 落地统一 AI 页面并接入四类入口。  
**FRs covered**: FR18-FR22b, FR36-FR38, NFR9-NFR11, NFR13
**Status Snapshot (2026-02-12)**: `done`（`BAI-4.1~4.5` 已完成，`BAI-4.6` 已取消，`BAI-4.7~4.10` 已完成开发回归）

**Semantic Freeze (BAI-4)**
- 会话区统一为 chat-only 体验：对话展示 summary-only，不展示完整文件正文。
- 预览区与对话区解耦：chat-only 回合不刷新 diff；有结构化变更时才刷新 preview/diff。
- diff 基线固定为 `base vs latest overlay`。
- Apply/Cancel 契约冻结：HTTP 使用 `cancel` 命名，cancel 语义为“结束会话并丢弃 overlay，不写业务对象”。
- Apply 成功后留在当前 Workbench 会话，不强制跳转退出，可继续下一轮聊天。
- 四入口后端语义冻结：`/ai/sessions/{sessionId}/messages` 需支持 `workflow|step|agent|asset`，不再仅限 step。
- BAI-4.7~4.10 code-review 闭环（2026-02-12）：context mismatch/required 请求不写入会话消息；`AI_TARGET_CONTEXT_REQUIRED` 契约已落地；前端已移除 step-only 发送限制。

**Stories**

### Story BAI-4.1: Create AI Workbench Route and Shell

As a **Builder User**,  
I want a dedicated AI workbench route,  
So that all AI operations happen in one consistent page.

**Acceptance Criteria**

**Given** entry query params (`targetType/targetId/mode`)  
**When** opening workbench  
**Then** page shows resolved target header and mode badge.

### Story BAI-4.2: Implement Conversation Pane

As a **Builder User**,  
I want multi-turn chat with status and retry,  
So that I can iteratively refine suggestions.

**Acceptance Criteria**

**Given** transient provider failure  
**When** retrying  
**Then** same session continues without losing latest preview.

**And** 对话区仅展示“修改概要”，不展示完整文件全文。

### Story BAI-4.3: Implement Change Preview Pane

As a **Builder User**,  
I want to preview structured updates before applying,  
So that I understand affected objects.

**Acceptance Criteria**

**Given** suggestion includes cross-object updates  
**When** preview rendered  
**Then** impact list includes each affected workflow/step/agent/asset.

**And** diff 基线固定为原始数据（base），对比目标为当前会话最新 overlay。

### Story BAI-4.4: Implement Apply/Cancel Flow with Return Navigation

As a **Builder User**,  
I want apply/cancel behavior to be explicit and safe,  
So that editing context is not lost.

**Acceptance Criteria**

**Given** Cancel clicked  
**When** returning to source page  
**Then** original unsaved edits remain intact.

**And** Apply 只能由用户手动按钮触发，Cancel 会丢弃会话 overlay 草稿。

### Story BAI-4.5: Connect Four Entrypoints

As a **Builder User**,  
I want Workflow/Step/Agent/Asset to open same AI workbench,  
So that interaction model is consistent.

**Acceptance Criteria**

**Given** any AI entrypoint  
**When** navigating to workbench  
**Then** session initializes with correct target context.

### Story BAI-4.6: Session Restore on Refresh (Canceled / De-scoped)

> 决策（2026-02-11）：该 Story 取消，不进入当前交付范围。  
> 原因：当前产品策略为“改完即走”，不保留/恢复 session refresh 状态，避免引入额外会话持久化复杂度。

### Story BAI-4.7: Expand Session Message Contract for Multi-Target

As a **Backend Engineer**,  
I want `/messages` request schema and server contract to support workflow/agent/asset contexts,  
So that non-step targets can use one unified session message endpoint.

**Acceptance Criteria**

**Given** targetType in `{workflow, agent, asset}`  
**When** calling `POST /packages/{projectId}/ai/sessions/{sessionId}/messages`  
**Then** API accepts corresponding target context payload and does not reject by `AI_TARGET_NOT_SUPPORTED`.

### Story BAI-4.8: Implement Workflow Target Message Path

As a **Builder User**,  
I want workflow target sessions to generate/validate staged changes through tool loop,  
So that workflow-level AI optimization can run in the same workbench.

**Acceptance Criteria**

**Given** an active workflow-target session  
**When** sending message  
**Then** assistant summary, staged changeSet, and validation result are returned with existing contract shape.

### Story BAI-4.9: Implement Agent/Asset Target Message Path

As a **Builder User**,  
I want agent and asset target sessions to run through the same backend flow,  
So that all entry targets behave consistently in workbench.

**Acceptance Criteria**

**Given** active `agent` or `asset` target session  
**When** sending message  
**Then** backend returns summary-only chat output and optional staged/validated changes with unified error contract.

### Story BAI-4.10: Four-Target Backend E2E Regression and Contract Lock

As a **Release Owner**,  
I want end-to-end regression coverage for workflow/step/agent/asset target messages,  
So that future changes cannot silently break non-step paths.

**Acceptance Criteria**

**Given** all four targets  
**When** running backend API regression suite  
**Then** each target can complete `chat -> (optional) stage -> validate` flow and error codes stay deterministic.

---

### Epic BAI-5: Security, Observability, and Rollout Hardening

**Goal**: 让新架构可审计、可观测、可灰度上线。  
**FRs covered**: NFR3, NFR4, NFR8 + rollout

**Stories**

### Story BAI-5.1: Security Redaction and Secret Handling

As a **Security Engineer**,  
I want sensitive fields redacted in logs and responses,  
So that keys and secrets are never exposed.

**Acceptance Criteria**

**Given** request/response logs  
**When** inspecting events  
**Then** api key fields are masked and cannot be recovered.

### Story BAI-5.2: Audit Event Pipeline

As an **Operator**,  
I want key AI lifecycle events recorded,  
So that every apply operation is traceable.

**Acceptance Criteria**

**Given** successful apply  
**When** audit queried  
**Then** event links sessionId, suggestionId, userId, and changed objects.

### Story BAI-5.3: Metrics and Alerting Baseline

As an **Operator**,  
I want metrics for latency/error/conflict/apply conversion,  
So that rollout health is visible.

**Acceptance Criteria**

**Given** production traffic  
**When** observing dashboards  
**Then** P95 latency, failure codes, and apply conversion are available.

### Story BAI-5.4: Canary Rollout + Legacy Decommission Plan

As a **Release Manager**,  
I want staged rollout with safe feature-flag rollback (without reviving legacy optimizer path),  
So that migration risk is controlled.

**Acceptance Criteria**

**Given** canary enabled  
**When** severe issue occurs  
**Then** traffic can switch to guarded mode (`disable apply` / `workbench read-only`) without data loss.

---

## Definition of Done（Initiative Level）

1. 四类入口都能进入统一 workbench 并完成 “建议 -> 暂存草稿 -> 预览 -> 手动应用”
2. apply 全部走后端原子事务，杜绝前端多请求部分成功
3. revision mismatch 可检测并可恢复（单用户阶段先以提示/契约为主）
4. 用户级 LLM 配置真实生效，且支持组织策略覆盖
5. 审计与核心指标可用，支持上线后问题定位
