---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8]
workflowComplete: true
inputDocuments:
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/prd-builder-AI.md'
  - '/Users/mengbin/code/GPT/CrewAgent/crewagent-runtime/spec/PACKAGE_BUILDER_PROMPT.md'
  - '/Users/mengbin/code/GPT/CrewAgent/crewagent-runtime/spec/LLM_COMPLETE_SPEC.md'
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/architecture.md'
workflowType: 'architecture'
lastStep: 8
project_name: 'CrewAgent Builder AI'
user_name: 'Mengbin'
date: '2026-02-08'
---

# Architecture Decision Document

_本文按 BMAD-method 架构文档方式整理：先做上下文与约束分析，再沉淀关键架构决策（AD-*），最后落到可实现的 API、数据模型、时序、迁移与验证标准。_

## 0. Document Map

- PRD：`_bmad-output/prd-builder-AI.md`
- Epics：`_bmad-output/epics-builder-ai.md`
- Tech Spec：`_bmad-output/tech-spec/builder-ai-implementation-spec.md`
- Delivery Roadmap：`_bmad-output/implementation-artifacts/builder-ai-delivery-roadmap.md`
- Sprint Status：`_bmad-output/implementation-artifacts/sprint-status-builder-ai.yaml`

## 1. Project Context Analysis

### 1.1 Requirements Overview（来自 PRD）

本次 Builder AI 的核心能力可归纳为 5 组：

| Capability Area | 关键 FR | 架构含义 |
|:---|:---|:---|
| Unified Workbench | FR18-FR22b, FR36-FR38 | 需要统一会话模型、目标上下文恢复、统一预览与提交协议 |
| Multi-Entry AI | FR23-FR27, FR32-FR35 | 需要入口化 Prompt 组装器与上下文裁剪器 |
| Structured Suggestion | FR1-FR14, FR28-FR31 | 需要强约束输出契约与领域校验流水线 |
| User-Level LLM Profile | FR15-FR17 | 需要用户级 provider/baseUrl/model/apiKey 持久化与鉴权 |
| Reliability/Security | NFR3-NFR13 | 需要读写权限分离、原子提交、审计与并发控制 |

### 1.2 Existing Technical Baseline（代码现实）

当前 Builder 关键数据均在数据库文本/JSON 字段中：

- `workflow_packages`: `agents_json`, `assets_json`, `artifacts_json`
- `package_workflows`: `workflow_md`, `graph_json`, `step_files_json`

当前 AI 仅 Step 入口，且应用改动由前端分多请求执行，不是后端单事务原子提交。

### 1.3 Architectural Constraints

1. 不能假设 Builder 具备 Runtime 的文件系统挂载模型（`@pkg/@project`）  
2. 需要兼容现有 `/packages` 路由体系，逐步演进  
3. 必须避免 LLM 直接写 DB 造成不可控副作用  
4. 要支持 step/asset、step/agent、workflow/step 的耦合读取

---

## 2. Key Architectural Decisions (ADRs)

### AD-01: LLM 与系统采用 ToolCall 循环，而非单轮生成

- **Decision**: AI 会话采用 `LLM -> tool calls -> tool results -> LLM` 循环，直到输出可校验建议。
- **Rationale**: 与 BMAD 的“按需取证”一致，避免一次性塞入大上下文。
- **Consequence**: 需要 Tool Host、循环控制器、修复回路（Repair Loop）。

### AD-02: Identity 分层为 Core Identity + Domain Persona

- **Decision**: 将原 `Layer-0 Identity` 拆分：
  - `Layer-0A Core Identity`（平台助手身份）
  - `Layer-0B Domain Persona`（行业应用架构师身份）
- **Rationale**: `PACKAGE_BUILDER_PROMPT.md` 强调行业专家角色，不能被统一 Builder 口吻覆盖。
- **Consequence**: 需要 `domain_profile` 来源（项目元数据或首次问答收集）。

### AD-03: 读宽写严（Read-Wide, Write-Guarded）

- **Decision**:
  - 读取允许跨对象扩展（在权限范围内）
  - 写入必须通过 `changeSet` -> `validate` -> `confirm` -> `apply`
- **Rationale**: step 与 assets/agent/workflow 有真实耦合，不应硬性禁读。
- **Consequence**: 写路径集中到后端 Apply Orchestrator，LLM 不直接写库。

### AD-04: 写入采用 Two-Phase Commit（业务层）

- **Decision**:
  - 阶段 A（Plan）：生成变更计划与预览
  - 阶段 B（Apply）：后端再校验、并发检查、单事务提交
- **Rationale**: 当前前端分请求写入容易出现部分成功、部分失败。
- **Consequence**: 需要新增 `changeSet` 模型与 apply 接口。

### AD-05: Builder 采用 DB 语义工具，不直接暴露 fs.write

- **Decision**: 在 Builder AI 中提供 `builder.*` 语义工具，内部映射到服务层/数据库。
- **Rationale**: Builder 数据不是文件系统真源，语义工具更稳、可控、可审计。
- **Consequence**: Tool Host 需实现 tool schema、权限判定、调用日志。

### AD-06: 并发控制采用 Revision 乐观锁

- **Decision**: 对 workflow/agents/assets 引入 revision，apply 时强制校验。
- **Rationale**: 避免多窗口或多人并发覆盖。
- **Consequence**: apply 接口可能返回 `409 CONFLICT`，前端需提示重载。

### AD-07: 用户级 LLM 配置优先，组织策略可覆盖

- **Decision**: 默认优先级 `OrgPolicy > UserProfile > SystemDefault`。
- **Rationale**: 满足个性化同时保持组织治理能力。
- **Consequence**: Profile API 要返回“不可用原因”，而非仅 bool。

---

## 3. Target Architecture

```text
[Entry Surface]
  Workflow Editor / Step Inspector / Agent Editor / Assets Panel
            |
            v
[AI Workbench Route]
  - Conversation Pane
  - Change Preview Pane
  - Impact + Validation Status
  - Apply / Cancel
            |
            v
[AI Gateway]
  - Session Service
  - Context Builder
  - Prompt Composer
  - LLM Adapter
  - Semantic Tool Host (builder.*)
  - Validator Pipeline
  - Apply Orchestrator (transaction + revision check)
            |
            +--> [Profile Service]
            +--> [Project/Workflow Service]
            +--> [Audit/Event Store]
```

---

## 4. System Prompt Architecture

### 4.1 Layer Model

1. `Layer-0A Core Identity`
2. `Layer-0B Domain Persona`
3. `Layer-1 Global Guardrails`
4. `Layer-2 Output Contract`
5. `Layer-3 Entry Strategy`
6. `Layer-4 Mode Strategy`
7. `Layer-5 Context Digest`
8. `Layer-6 Apply Policy`

### 4.2 Domain Persona Assembly（来自 BMAD 思路）

`Layer-0B` 输入建议：

- `industry_domain`
- `target_users`
- `core_pain_points`
- `expected_outcomes`
- `quality_style`

示例（片段）：

```text
You are an industry application architect for {industry_domain}.
Primary users: {target_users}.
Your goal is to convert domain intent into executable BMAD-compliant workflow assets.
```

### 4.3 Entry/Mode Matrix

| Target | Create | Optimize | Special Rule |
|:---|:---|:---|:---|
| Workflow | 生成 workflow/frontmatter/graph 策略 | 优化结构/变量/分支策略 | 需输出影响节点与引用变化 |
| Step | 生成 Goal/Instructions/Completion | 优化 step 内容与引用建议 | 可联动 assets 但必须列 impact |
| Agent | 生成 schema 合法 agent | 优化 persona/prompts/tools | 默认不改 `agentId` |
| Asset | 生成内容与路径建议 | 优化内容并检查引用 | 路径与扩展名受策略约束 |

### 4.4 Prompt Composer Pseudocode

```ts
function composeSystemInstruction(input: ComposeInput): string[] {
  return [
    buildCoreIdentity(input.platformPolicy),
    buildDomainPersona(input.domainProfile),
    buildGlobalGuardrails(input.securityPolicy, input.bmadConstraints),
    buildOutputContract(input.targetType),
    buildEntryStrategy(input.targetType),
    buildModeStrategy(input.mode),
    buildContextDigest(input.contextEnvelope),
    buildApplyPolicy(input.writePolicy),
  ].filter(Boolean);
}
```

### 4.5 Schema Validation Guardrails in System Instruction

- 系统指令必须显式要求模型执行“schema-first”自检，再输出结果。  
- 自检至少覆盖：
  - 必填字段完整（required）
  - 字段类型正确（type）
  - 枚举值合法（enum）
  - 不输出未定义扩展字段（no unsupported extras）
- 若模型无法确定 schema 合法性，应先继续用 read tools 补充上下文，不允许猜测字段。  
- 若当前结果未通过自检，优先返回修复建议与修改概要，不返回错误结构化 payload。

---

## 5. LLM Interaction Protocol (Builder Side)

### 5.1 Conversation Loop

```text
CreateSession
  -> BuildInitialContext
  -> ComposePrompt
  -> UserInstruction(chat)
  -> CallLLM
      -> LLM Planning
      -> if tool_calls: ExecuteTools(仅读 + Stage Overlay) -> AppendResults -> CallLLM
      -> else: Parse -> StageDraft -> Validate -> PreviewReady
  -> AssistantReply(summary-only)
  -> User continues
      -> BuildDeltaContext(from latest overlay) -> ComposePrompt -> CallLLM
  -> User Manual Confirm Apply
      -> ValidateChangeSet -> CheckRevision -> CommitTransaction
```

### 5.2 Stop Conditions

- `tool_calls` 为空且结构化输出合法：进入预览
- 连续修复超过阈值（默认 2）：返回人工修复提示
- provider/权限不可用：返回可操作错误码

### 5.3 Assistant Output Policy（会话展示）

- 聊天区只展示“修改概要信息”：变更对象、变更类型、影响范围、风险提示、下一步建议。
- 聊天区不回传完整文件全文（避免长文本污染对话和误操作）。
- 文件级完整内容、逐行对比、可编辑预览仅在 Preview/Diff Pane 展示。
- 每一轮都基于“最新临时草稿态”输出摘要，不基于过期快照。

---

## 6. Read/Write Strategy and Permission Model

### 6.1 Read Policy（宽）

- 允许读取：目标对象 + 关联摘要
- 支持按需扩读：引用路径、邻接节点、被引用对象
- 对大内容做窗口化返回（preview + metadata）

### 6.2 Write Policy（严）

- LLM 不直接写 DB
- LLM 可通过工具写入“会话临时草稿区”（Draft Overlay），用于多轮迭代
- 会话临时草稿区不等于业务数据，不影响正式 workflow/step/agent/asset
- 未 Apply 前，读写都基于“最新 overlay”：读取优先级 `overlay > base`
- 预览阶段基于 `base + overlay` 计算结果，diff 比较基线固定为 `base`（原始数据）
- 只有用户点击确认后，才允许触发 `builder.change.apply`
- `changeSet` 必须包含 `impact`
- Apply 前执行：
  - schema 校验
  - 领域校验
  - revision 校验
  - 安全校验
- 全部通过后单事务提交

### 6.3 Why This Works for DB Storage

当前 Builder 的 workflow/step/agents/assets 均为 DB 字段存储，  
语义工具可以直接调用现有 service layer，而不是模拟文件系统 patch。

---

## 7. Context Architecture

### 7.1 Initial Context Envelope

```json
{
  "session_meta": {
    "projectId": 1,
    "workflowId": 2,
    "targetType": "step",
    "targetId": "step-01",
    "mode": "optimize"
  },
  "domain_profile": {},
  "normative_baseline": {},
  "target_snapshot": {},
  "dependency_digest": {},
  "tool_capabilities": {},
  "revision_info": {}
}
```

### 7.2 Minimal Context by Target

- Workflow: frontmatter 摘要 + graph 摘要 + node/agent 索引
- Step: step 内容 + 入出边 + variables keys + 相关 assets
- Agent: agent 定义 + agent 列表 + 引用节点列表
- Asset: 目标文件 + 被引用 step 列表 + 文件类型约束

### 7.3 Context Budget

- `response_budget`: 30%
- `context_budget`: 70%
- 裁剪顺序：
  1. 保留目标对象
  2. 保留结构摘要
  3. 其他对象降级为路径 + 摘要

---

## 8. Semantic Tool Contracts (Builder.*)

### 8.1 Read Tools

| Tool | Input | Output |
|:---|:---|:---|
| `builder.context.get` | target metadata | initial context envelope |
| `builder.workflow.read` | `projectId/workflowId` | workflow snapshot |
| `builder.step.read` | `projectId/workflowId/nodeId` | step snapshot |
| `builder.agent.read` | `projectId/agentId` | agent snapshot |
| `builder.asset.read` | `projectId/path` | asset content + metadata |
| `builder.refs.find` | object locator | inbound/outbound references |

### 8.2 Write-Gated Tools

| Tool | Role |
|:---|:---|
| `builder.change.stage` | 写入/更新会话级临时草稿（overlay），支持多轮增量修改 |
| `builder.change.validate` | 对“当前 overlay 视图”做校验，返回错误、warning 与修复建议 |
| `builder.change.apply` | 仅在用户手动确认后执行原子提交（把已验证草稿落库） |
| `builder.change.discard` | 丢弃会话临时草稿，回到基线状态 |

**执行约束（强制）**

1. `builder.change.apply` 必须由“人工确认路径”触发，不能由 LLM 自动决策触发。  
2. Tool loop 默认不暴露 apply 能力给模型；apply 由 Workbench UI 的 `Apply` 按钮调用后端接口。  
3. Validate 可多次调用，且仅更新校验结果与草稿元数据，不写业务对象。  
4. 任何时刻失败都可 `discard`，保证正式数据不被半成品污染。  
5. `builder.change.stage` 与 `builder.change.validate` 必须作用于同一个 session overlay，保证多轮指令连续性。  

### 8.3 Example `changeSet`

```json
{
  "targetType": "step",
  "mode": "optimize",
  "changeSet": {
    "steps": [
      {
        "nodeId": "step-01",
        "workflowId": 2,
        "stepMarkdown": "...",
        "reason": "补齐 Completion 并增加 policy 引用"
      }
    ],
    "assets": {
      "upsert": [
        { "path": "assets/policies/expense-policy.md", "content": "..." }
      ],
      "delete": []
    }
  },
  "impact": {
    "updated": ["step:step-01", "asset:assets/policies/expense-policy.md"],
    "risks": [],
    "requiresConfirmation": true
  }
}
```

---

## 9. API Architecture

### 9.1 User LLM Profile APIs

- `GET /users/me/llm-profile`
- `PUT /users/me/llm-profile`
- `POST /users/me/llm-profile/test`

### 9.2 AI Session APIs

- `POST /projects/{projectId}/ai/sessions`
- `GET /projects/{projectId}/ai/sessions/{sessionId}`
- `POST /projects/{projectId}/ai/sessions/{sessionId}/messages`
- `POST /projects/{projectId}/ai/sessions/{sessionId}/apply`
- `POST /projects/{projectId}/ai/sessions/{sessionId}/cancel`

### 9.3 Error Model

统一错误结构：

```json
{
  "error": {
    "code": "AI_VALIDATION_FAILED",
    "message": "step frontmatter 缺少 nodeId",
    "hints": ["请保持 nodeId 不变", "补齐 frontmatter.required 字段"]
  }
}
```

---

## 10. Data Architecture and Migration

### 10.1 Existing Tables（复用）

- `workflow_packages`
- `package_workflows`
- `users`

### 10.2 New Tables（建议）

1. `user_llm_profiles`
2. `ai_sessions`
3. `ai_messages`
4. `ai_change_sets`
5. `ai_audit_events`（可选）

### 10.3 Revision Fields（建议）

- `package_workflows.revision`（int）
- `workflow_packages.agents_revision`（int）
- `workflow_packages.assets_revision`（int）

### 10.4 Migration Strategy

1. 先加表/字段，不切流量
2. 上线双写（旧 step-draft + 新 session 兼容）
3. 前端切换到 workbench
4. 观察稳定后下线旧路径

---

## 11. Validation Pipeline

### 11.1 Validation Stages

1. JSON schema validation
2. BMAD rule validation
3. Relationship/impact validation
4. Security/path validation
5. Concurrency validation

### 11.2 BMAD Rule Examples

- step frontmatter 必填字段完整
- agentId 必须存在于 `agents_json`
- assets 路径必须在允许范围
- workflow graph 不得产生非法边

---

## 12. Security, Compliance, and Observability

### 12.1 Security Controls

- API key 加密存储
- PII/secret 日志脱敏
- tool 调用按 user/project 权限隔离

### 12.2 Audit Events

- `ai.session.created`
- `ai.prompt.composed`
- `ai.tool.called`
- `ai.validation.failed`
- `ai.change.applied`

### 12.3 Metrics

- 请求成功率、平均耗时、P95
- Repair Loop 次数
- Apply 转化率
- `409 revision conflict` 频率

---

## 13. BMAD-Style Implementation Roadmap

### Phase 1（Architecture Baseline）

1. Prompt Composer 分层实现（含 Layer-0B）
2. Initial Context Envelope 与 Context Builder
3. `changeSet` 契约与校验器

### Phase 2（Service & API）

1. `user_llm_profiles` 持久化
2. `ai_sessions` 系列 API
3. `builder.*` 语义工具与 Tool Host

### Phase 3（UI Integration）

1. AI Workbench 页面
2. 四类入口接入（Workflow/Step/Agent/Asset）
3. Apply 冲突处理与恢复流程

### Phase 4（Hardening）

1. 审计与监控
2. 并发压测与回归
3. 兼容接口下线

---

## 14. Acceptance Criteria for Architecture Readiness

1. 能从任一入口进入统一会话并完成预览与应用闭环
2. 能在不直接写 DB 的前提下完成跨对象建议
3. 能在 revision 冲突时安全失败并提示恢复
4. 能使用用户级 LLM 配置并被组织策略覆盖
5. 所有写入均可追溯到会话、建议与提交记录

---

## 15. Final Architecture Conclusions

1. Builder AI 的正确实现路径是“ToolCall 循环 + 语义工具”，不是单次文本生成。  
2. 身份层必须行业化：`Core Identity + Domain Persona`。  
3. 读权限应支持耦合关系，写权限必须走两阶段受控提交。  
4. 初始上下文应标准化为 Envelope，支持按需扩读。  
5. 在当前 Builder 的数据库模型下，此架构可渐进落地，无需先改存储形态。  
