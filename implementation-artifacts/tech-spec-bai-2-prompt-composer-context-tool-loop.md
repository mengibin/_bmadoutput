# Tech Spec: Epic BAI-2 - Prompt Composer + Context Builder + Semantic Tool Host

**Epic**: `BAI-2`（`_bmad-output/epics-builder-ai.md`）  
**Status**: Done（2026-02-11）  
**Scope**: BAI-2.1 ~ BAI-2.6  
**Related Docs**:
- `_bmad-output/prd-builder-AI.md`
- `_bmad-output/architecture/builder-ai-llm-interaction-architecture.md`
- `_bmad-output/tech-spec/builder-ai-implementation-spec.md`

---

## 1. 目标与边界

### 1.1 目标

为 Builder AI 建立可复用的“提示词组装 + 上下文组装 + ToolCall 循环 + 语义工具 + 上下文预算控制”底座，支撑后续会话与变更提交流程。
并冻结 BAI-2 边界：本 Epic 负责多轮取证与临时草稿（stage）能力，不负责自动 apply。

覆盖需求：

- FR23-FR27：入口化上下文与 Prompt 组装
- FR36-FR38：统一 AI 工作台能力底座
- NFR5：最小上下文、受控读写、稳定可追踪

### 1.2 非目标（本 Epic 不做）

- 不实现 `changeSet.apply` 原子提交（BAI-3）
- 不实现自动 apply（模型不可直接触发 apply）
- 不实现完整前端 Workbench 页面（BAI-4）
- 不实现组织策略 UI（只使用现有后端策略能力）

---

## 2. 现状与差距

### 2.1 当前实现现状

- 早期 `app/services/ai_step_service.py` 以单入口 step draft prompt 为主，当前需统一迁移到会话化 tool loop。
- Prompt 仍以固定字符串拼接为主，缺少分层模型和可测试组合。
- 无通用 context envelope 构建器。
- 无 `builder.*` 语义读工具与统一 ToolCall 循环控制器。
- 无上下文预算与裁剪策略。

### 2.2 核心差距

1. 不同入口（workflow/step/agent/asset）无法共享一致组装协议。
2. LLM 取证方式依赖“预塞上下文”，没有按需读工具闭环。
3. 大上下文场景缺少可预测裁剪，稳定性与成本不可控。

---

## 3. 模块设计（Epic BAI-2）

## 3.1 Prompt Composer（BAI-2.1）

新增服务：`app/services/prompt_composer_service.py`

核心方法：

```python
compose_system_instruction(input: ComposeInput) -> list[PromptLayer]
compose_messages(input: ComposeInput, user_message: str) -> list[dict[str, str]]
```

分层顺序（固定）：

1. `Layer-0A Core Identity`
2. `Layer-0B Domain Persona`
3. `Layer-1 Global Guardrails`
4. `Layer-2 Output Contract`
5. `Layer-3 Entry Strategy`
6. `Layer-4 Mode Strategy`
7. `Layer-5 Context Digest`
8. `Layer-6 Apply Policy`

要求：

- 层顺序确定、可快照测试。
- 每层可单测校验是否命中（按 `targetType/mode`）。
- Guardrails 必须包含 schema-first 约束（required/type/enum/no extra fields）。

## 3.2 Domain Persona Resolver（BAI-2.2）

新增服务：`app/services/domain_persona_service.py`

```python
resolve_domain_profile(db, *, project_id: int, workflow_id: int | None) -> DomainProfile
build_domain_persona_layer(profile: DomainProfile) -> PromptLayer
```

优先级：

1. 项目级领域元数据（若已存在）
2. workflow 语义线索（名称、变量、资产命名）
3. 安全默认值（非行业专属）

要求：

- 不返回“统一 Builder 助手”空泛身份。
- persona 输出包含 `industry_domain/target_users/core_pain_points/expected_outcomes/quality_style`。

## 3.3 Initial Context Envelope（BAI-2.3）

新增服务：`app/services/context_envelope_service.py`

```python
build_initial_context_envelope(
    db,
    *,
    user_id: int,
    project_id: int,
    workflow_id: int | None,
    target_type: str,
    target_id: str,
    mode: str,
) -> ContextEnvelope
```

固定结构：

- `session_meta`
- `domain_profile`
- `normative_baseline`
- `target_snapshot`
- `dependency_digest`
- `tool_capabilities`
- `revision_info`

要求：

- schema 稳定，支持后续会话恢复。
- 最小化默认内容，按目标对象做差异化填充。

## 3.4 Semantic Read Tools（BAI-2.4）

新增服务：`app/services/semantic_tool_service.py`

第一批工具（只读）：

- `builder.context.get`
- `builder.workflow.read`
- `builder.step.read`
- `builder.agent.read`
- `builder.asset.read`
- `builder.refs.find`

要求：

- 全部走后端 service 层与鉴权，不暴露直接 DB/FS 写入。
- 每个工具返回统一结构：`ok/data/error` + `revision` + `meta`。

## 3.5 ToolCall Loop Controller（BAI-2.5）

新增服务：`app/services/tool_loop_service.py`

```python
run_tool_loop(
    *,
    session_state: SessionState,
    llm_adapter: LlmAdapter,
    tool_host: SemanticToolHost,
    max_iterations: int = 8,
    max_repairs: int = 2,
) -> LoopResult
```

循环规则：

1. 调用 LLM
2. 若有 tool_calls：执行工具并回填 tool results
3. 无 tool_calls 时解析结构化输出并进入 validator
4. 超过上限返回可恢复错误码

白名单（冻结）：

- 允许：`builder.context.get/workflow.read/step.read/agent.read/asset.read/refs.find`
- 允许：`builder.change.stage`、`builder.change.validate`、`builder.change.discard`
- 禁止：`builder.change.apply`（仅 UI 手动确认路径可触发）

## 3.6 Context Budget + Trimming（BAI-2.6）

新增服务：`app/services/context_budget_service.py`

```python
trim_context_with_budget(
    *,
    envelope: ContextEnvelope,
    model: str,
    context_token_budget: int,
    reserve_for_response: int,
) -> TrimmedEnvelope
```

裁剪策略：

1. 目标对象永不裁剪（`target_snapshot`）
2. 依赖对象先摘要化再截断
3. 非目标扩展对象降级为 `path + summary + hash`

---

## 4. 接口契约（冻结草案）

## 4.1 ComposeInput

```json
{
  "entrypoint": "workflow|step|agent|asset",
  "targetType": "workflow|step|agent|asset",
  "mode": "create|optimize",
  "domainProfile": {},
  "contextEnvelope": {},
  "writePolicy": {
    "allowApply": false,
    "requiresValidation": true
  }
}
```

## 4.2 Context Envelope（最小字段）

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

## 4.3 Semantic Tool Call Envelope

```json
{
  "tool": "builder.step.read",
  "arguments": {
    "projectId": 1,
    "workflowId": 2,
    "nodeId": "step-01"
  }
}
```

统一返回：

```json
{
  "ok": true,
  "data": {},
  "error": null,
  "meta": {
    "revision": 12
  }
}
```

---

## 5. 错误码（BAI-2 新增）

- `AI_PROMPT_COMPOSE_ERROR`
- `AI_DOMAIN_PROFILE_INCOMPLETE`
- `AI_CONTEXT_BUILD_ERROR`
- `AI_TOOL_NOT_ALLOWED`
- `AI_TOOL_FORBIDDEN`
- `AI_TOOL_EXECUTION_ERROR`
- `AI_TOOL_LOOP_LIMIT_EXCEEDED`
- `AI_CONTEXT_BUDGET_EXCEEDED`

---

## 6. Story 级实施拆分

## 6.1 BAI-2.1（Prompt Composer）

1. 新建 `prompt_composer_service.py`
2. 实现层级组装与顺序测试
3. 提供统一 `compose_messages`

## 6.2 BAI-2.2（Domain Persona）

1. 新建 `domain_persona_service.py`
2. 建立 `DomainProfile` 与 fallback 规则
3. 与 Composer 集成 `Layer-0B`

## 6.3 BAI-2.3（Initial Context）

1. 新建 `context_envelope_service.py`
2. 产出标准 envelope schema
3. 覆盖四类 targetType 构建分支

## 6.4 BAI-2.4（Semantic Read Tools）

1. 新建 `semantic_tool_service.py`
2. 实现六个 `builder.*` 只读工具
3. 补鉴权与审计字段

## 6.5 BAI-2.5（Tool Loop）

1. 新建 `tool_loop_service.py`
2. 实现 stop conditions 与 repair loop
3. 输出 `LoopResult` 供 session 层消费

## 6.6 BAI-2.6（Budget/Trim）

1. 新建 `context_budget_service.py`
2. 实现 token budget + trim report
3. 与 Prompt Composer/Loop 串联

---

## 7. 验收矩阵

1. Prompt 层次完整且顺序稳定（快照可复现）
2. 不同项目/工作流可生成不同 Domain Persona
3. 四入口都能生成标准 Context Envelope
4. LLM 可通过 `builder.*` 工具按需取证
5. Tool loop 在上限内收敛，异常有明确错误码
6. 超大上下文可被可预测裁剪，目标对象不丢失
7. 聊天回复遵循 `summary-only`（不在 conversation 中返回完整文件全文）
8. 模型侧 loop 不暴露 apply，出现 apply 调用必须返回 `AI_TOOL_FORBIDDEN`

---

## 8. DoD（Epic BAI-2）

1. BAI-2.1 ~ BAI-2.6 story 文档齐备并进入开发状态流转
2. 后端核心服务接口冻结并有单测计划
3. 与 Epic BAI-3 的 `changeSet` 校验接口边界清晰
4. `sprint-status-builder-ai.yaml` 状态同步更新
5. Prompt Composer 的 schema-first guardrails 已冻结并可测试
6. Tool loop 白名单与 apply 禁调策略已冻结并可测试
