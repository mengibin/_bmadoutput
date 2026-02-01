---
workflowType: "prd"
lastStep: 11
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11]
date: "2026-01-29"
project_name: "CrewAgent Builder：LLM 辅助 Step/Agent/assets 创建与优化"
---

# 产品需求文档（PRD）- CrewAgent Builder：LLM 辅助 Step/Agent/assets 创建与优化

**Author:** TBD  
**Date:** 2026-01-29  

---

## Executive Summary（执行摘要）

### Vision（愿景）
让工作流设计者在 CrewAgent Builder 中，以「逐个对象（Step / Agent / assets）」的方式，借助 LLM 快速生成/优化符合 BMAD v1.1 规范的内容，并在一个独立的 AI 工作台页面（Chat + 变更预览 + 应用）内完成沟通与落地，显著降低流程搭建门槛与返工成本。

### Problem（要解决的问题）
1. **Step 编写成本高**：Step 的 Goal/Instructions/Completion（Document-as-State）需要细化到可执行、可校验；手写慢且容易遗漏。
2. **规范容易写错**：`assets/`、`artifacts/` 路径、Python 路径、frontmatter 字段、Completion 状态更新等，容易出错导致导出/运行失败。
3. **资产管理割裂**：Step 常需要政策/模板/脚本等静态参考，缺少「从 Step 需求出发、自动生成/更新 assets 并建立引用」的闭环。
4. **Agent 定义不成体系**：Agent persona、prompts、工具策略等需要结构化表达且与工作流匹配；缺少“生成/优化 + 校验 + 预览应用”的一体化入口。
5. **增量优化缺少抓手**：现有 Step/Agent/assets 在调试/迭代中需要不断“局部变更”，希望每次只影响选中对象（必要时连带 assets），而不是重写整包。

### Target Users（目标用户）
- 工作流/行业包设计者：产品经理、业务专家、解决方案架构师、AI 运营、开发者（负责把业务流程落成可执行工作流包）。
- 使用场景：新建工作流、把现有 Step 规范化、补齐 Completion、补资产引用、提升可执行性与可维护性。

### Differentiator（差异化）
- **逐 Step 增量建议**：一次只针对一个 Step 生成/优化，适配编辑器工作流（B 模式）。
- **结构化输出 + 可预览应用**：LLM 输出必须结构化（建议 + assets patch + notes），Builder 侧提供预览与显式确认再应用。
- **规范优先、可校验**：围绕 BMAD v1.1 的硬约束做 guardrails，减少“看起来对但导出/运行失败”的幻觉输出。
- **assets 闭环**：让 LLM 同时产出必要的 `assets/`（policy/template/script）并在 Step 中引用，形成“Step ↔ assets”协同。

---

## Success Criteria（成功标准）

### Outcome Metrics（结果指标）
1. **效率**：用户创建一个可导出/可运行的 Step（含必要 assets）的平均时间降低 ≥ 50%。
2. **质量**：AI 生成/优化后，导出校验（v1.1）一次通过率 ≥ 90%（以“无 error 级别问题”为准）。
3. **采用**：编辑器中 AI 功能的月活使用率（MAU 使用过 ≥1 次）≥ 30%（上线后 4-6 周）。
4. **返工**：因规范错误（路径/frontmatter/缺段落）导致的人工修复次数降低 ≥ 40%。

### Release Gate（上线门槛）
- 支持“逐 Step 创建/优化”完整闭环：生成 → 预览 → 应用 → 保存 → 导出校验可通过（示例工作流）。
- 至少提供一种可用 LLM 模式（可为 mock/本地/自建 OpenAI-compatible）与可禁用开关。
- 不引入高风险的数据泄露路径（见 NFR-安全）。

---

## Product Scope（范围）

### MVP（最小可用）
1. **AI 工作台（新页面）**：在使用 AI 创建/优化 Step/Agent/assets 时打开独立页面，包含 Chat 窗口 + 变更预览 + Apply/Cancel。
2. **逐 Step Create/Optimize**：用户提供 Prompt（Create 必填；Optimize 可选），生成/优化当前 Step 的建议内容（Goal/Instructions/Completion + frontmatter 相关字段），可预览/应用。
3. **逐 Agent Create/Optimize**：生成/优化 Agent 定义（persona/prompts/tools 等），可预览/应用，保证 `agents.json` schema 可通过校验。
4. **assets 建议与应用**：LLM 可返回 `assets/` 的 upsert/delete 建议；用户确认后，Builder 完成创建/更新/删除并刷新。
5. **基础校验**：对建议做基础规范校验（如 schema、assets 路径合法、输出字段类型等）；失败时给出可读错误并允许重试。
6. **记录与可追溯**：保留一次建议的 notes（至少在 AI 工作台页面展示；持久化可后置）。

### Out of Scope（不做）
- 自动生成整包/多 workflow 的“一键全量生成”（A 模式）。
- 自动修改多个 Step/跨 Step 重构（除非用户明确选择批量能力）。
- 自动执行/运行工作流并基于运行日志闭环优化（可作为后续迭代）。
- 复杂的差异对比器（MVP 可只做“当前 vs 建议”的文本预览）。

### Next（后续迭代方向）
- LLM 输出变更可视化 diff（逐段落/逐字段）。
- “校验失败 → 自动生成修复建议并一键应用”的修复模式。
- 支持更多资产类型与关联（schemas、prompts、脚本目录结构建议等）。
- 组织级策略（禁止外发、只允许某些模型/端点、脱敏规则）。

---

## User Journeys（用户旅程）

### Journey 0：进入 AI 工作台（新页面）
1. 用户在 Builder 中从某个入口点击“AI 创建/优化…”
   - Step 入口：Workflow Editor 中选中节点 → AI Create/Optimize
   - Agent 入口：Agent 列表/Agent 编辑器 → AI Create/Optimize
   - assets 入口：Assets 管理区或 Step 引用处 → AI Create/Optimize
2. 系统通过路由跳转到一个新的页面（AI 工作台，单页应用内新 route，非新标签页），页面包含：
   - Chat：用户与 AI 多轮对话（补充信息/澄清范围/确认假设）
   - 变更预览：展示 AI 产出的结构化变更（对 Step/Agent/assets 的改动内容与清单）
3. 用户在工作台内确认：
   - Apply：将变更写回对应对象并返回/刷新原编辑页面
   - Cancel：不写入任何变更，返回原页面

### Journey 1：生成一个新 Step（Create）
1. 用户在编辑器中选中一个节点（Step/Decision/Merge/End/Subworkflow）。
2. 点击 AI Create Step（入口可在 Inspector 或节点操作菜单）。
3. 系统打开 AI 工作台（新页面），用户输入 Prompt 并与 AI 对话澄清需求。
4. 工作台展示建议：Step 内容 + 可选 assets patch + notes，并提供预览。
5. 用户点击 Apply：系统将建议写入当前 Step，并应用 assets 变更；保存后通过导出校验。

### Journey 2：优化当前 Step（Optimize）
1. 用户选中一个已有 Step（可能缺 Completion 或 instructions 粗糙）。
2. 点击 AI Optimize Step（Prompt 可选，例如“补齐 Document-as-State 规则，并把政策放到 assets/”）。
3. 系统打开 AI 工作台（新页面），AI 基于当前 Step + 上下游/变量上下文给出增量改写建议。
4. 预览 → Apply → 保存。

### Journey 3：生成/优化 Agent（Create/Optimize）
1. 用户从 Agent 列表进入：
   - Create：点击“AI Create Agent”，输入目标职责与沟通风格等
   - Optimize：选择已有 Agent → 点击“AI Optimize Agent”，补齐 persona/prompts/tools 等
2. 系统打开 AI 工作台（新页面），用户与 AI 多轮澄清 Agent 的职责边界与输出风格。
3. 工作台展示建议：Agent 结构化定义（含 schemaVersion/persona/prompts/tools/menu 等）与 notes，并提供预览。
4. Apply 后写回 `agents.json`（并确保 agentId 唯一与 schema 可校验）。

### Journey 4：AI 建议创建/更新 assets
1. AI 识别到 Step 需要政策/模板/脚本等静态资料。
2. 返回 `assets/` 的 upsert（可包含新文件路径与内容）。
3. 预览资产清单，用户确认后应用。
4. Step 中应出现对 assets 的引用（例如 `assets/policies/...`），并符合规范。

### Journey 5：失败与恢复
- AI 返回不合规（例如非法 assets 路径、缺关键段落）→ Builder 阻止应用并显示原因 → 用户可继续优化或手改。
- 资产更新冲突/不存在等 → Builder 给出明确提示并允许重试或手动处理。

---

## Domain Requirements（领域/规范要求）

1. Step 必须符合 BMAD v1.1 的 step.md 结构要求：
   - YAML frontmatter 至少包含：`schemaVersion`, `nodeId`, `type`, `title`, `agentId`, `inputs`, `outputs`（若规范要求）。
   - 至少包含 `## Goal` 与 `## Instructions`；Completion 推荐采用 `## Completion (Document-as-State)` 语义（标题可由实现映射，但内容必须表达状态更新规则）。
2. assets 使用规范：
   - 所有静态参考文件必须放在 `assets/` 下（如 `assets/policies/...`、`assets/templates/...`、`assets/scripts/...`）。
   - Step 中引用 assets 必须使用 `assets/...` 的包内相对路径（并与导出/运行工具路径规则一致）。
3. artifacts 使用规范：
   - 动态生成产物必须落在 `artifacts/...` 目录语义下（并可映射到运行时 `@project/artifacts/...`）。
4. 路径禁忌（防错要求）：
   - `@project/...` 与 `@pkg/...` 只允许出现在工具调用参数中；不得出现在 Python 代码里（如有 Python 片段必须使用相对路径）。
5. Agent 定义规范（BMAD v1.1）
   - AI 生成/优化的 Agent 必须满足 `agents.json` v1.1 schema（含 `schemaVersion`, `agents[].id/metadata/persona` 等必填字段）。
   - `agentId`（即 `agents[].id`）必须符合 ID pattern，并在项目内唯一；MVP 中 **Optimize 不允许修改 agentId**（避免批量更新引用）。
   - Step/Graph 引用的 `agentId` 若不存在于 `agents.json`，前端必须给出明确告警（并阻止导出校验通过）。
6. LLM Provider 配置规范（Builder AI）
   - Provider 需要支持至少三种状态：`disabled`（禁用）、`mock`（离线/演示）、`openai-compatible`（OpenAI 兼容接口）。
   - `openai-compatible` 至少需要以下配置项：`baseUrl`、`model`、`apiKey`（可选：当 endpoint 不需要鉴权时允许为空）、`timeoutSeconds`。
   - 推荐的服务端环境变量命名（便于部署与排障）：`AI_PROVIDER`、`AI_BASE_URL`、`AI_MODEL`、`AI_API_KEY`、`AI_TIMEOUT_SECONDS`。
   - 机密配置（如 `apiKey`）不得在前端明文持久化；默认以服务端环境变量/服务端配置为准。
   - 当 provider 未配置或不满足必填项时，前端必须明确提示“不可用原因”（例如缺少 baseUrl/model），并禁止发起请求。

---

## Innovation Analysis（创新点分析）

- 用 LLM 做“逐 Step 的结构化建议生成”，不是自由写作：
  - 输出必须能直接映射到 Step 的字段与段落；
  - assets 变更必须以 patch 形式列出；
  - Builder 必须作为守门员执行校验与预览确认，降低幻觉风险。
- 该功能的价值来自“可应用、可校验、可回滚”的增量变更，而非一次性生成长文。

---

## Project-Type Requirements（项目类型要求）

1. 作为 Builder 编辑体验的一部分，必须在“选中节点 → Inspector 编辑”流程中可用。
2. 必须适配多种节点类型（至少 step/decision/merge/end/subworkflow），并保证类型变更不会破坏 workflow.graph 的约束（例如 decision 默认分支等）。
3. 必须与现有导出校验流程兼容：AI 产生的内容应可通过 v1.1 校验与打包导出。

---

## Functional Requirements（功能需求）

### A. Step 创建与优化
- FR1：用户可以对选中的节点发起 **Create**（输入 Prompt）并获得 Step 建议内容。
- FR2：用户可以对选中的节点发起 **Optimize**（Prompt 可选）并获得改进建议内容。
- FR3：建议内容必须覆盖：`type/title/agentId/inputs/outputs/setsVariables（如适用）/Goal/Instructions/Completion`。
- FR4：用户必须能在应用前预览建议内容，并选择 Apply 或 Cancel。
- FR5：Apply 后只影响当前选中节点对应的 Step 内容（除非 assets patch 被确认应用）。

### B. assets 建议与应用
- FR6：AI 可以返回 assets patch：`upsert`（创建或更新）与 `delete`（删除）。
- FR7：用户在 Apply 时必须明确同意应用 assets patch（MVP 可与 Apply 同按钮，但必须在预览中清晰展示将变更的 assets 清单）。
- FR8：assets patch 中的路径必须被校验为 `assets/` 下的安全文件路径；不合法则禁止应用并提示原因。
- FR9：应用 assets patch 后，资产列表应立即刷新，且 Step 中引用路径保持一致。

### C. 规范与校验（Guardrails）
- FR10：系统必须在应用前执行最低限度的结构校验（字段类型、必填内容、assets 路径合法）。
- FR11：当校验失败时，必须给出可读错误并允许用户继续优化或手动编辑，不得破坏现有 Step 内容。
- FR12：系统必须支持“禁用 AI 功能”（例如在组织策略或用户设置中）。

### D. 可用性与反馈
- FR13：AI 请求必须有 loading 状态与失败提示；失败不影响当前编辑内容。
- FR14：建议返回的 notes 必须在预览中展示（用于说明假设、风险、需要确认的信息）。

### E. LLM Provider 配置
- FR15：系统必须提供 LLM provider 的配置入口（至少支持服务端配置），包含：`provider`、`baseUrl`、`model`、`apiKey`（如需要）、`timeoutSeconds`。
- FR16：前端必须展示当前 provider 可用性状态（可用/禁用/未配置）及失败原因，并在不可用时禁用 Generate/Optimize。
- FR17：系统应提供“连接测试/健康检查”能力（MVP 可选），用于验证 baseUrl/model 配置是否可正常返回。

### F. AI 工作台（新页面：Chat + 变更预览）
- FR18：当用户从 Step/Agent/assets 的 AI 入口进入时，系统必须通过路由跳转打开一个独立的 AI 工作台页面（同站内新 route，默认同标签页）。
- FR19：AI 工作台页面必须包含两块核心区域：**Chat 对话区** 与 **变更预览区**（展示本次将修改的 Step/Agent/assets 内容与清单）。
- FR20：Chat 必须支持多轮对话：AI 可以追问缺失信息；用户补充后，预览区应更新为新的建议版本（以“最新建议”为准）。
- FR21：工作台必须提供 Apply/Cancel：Apply 将变更写回项目并回到原编辑页面；Cancel 不写入任何变更。
- FR22：工作台必须展示本次建议的影响范围（例如“将修改：step-xx；将 upsert：assets/policies/...；将更新：agentId=...”）。
- FR22b：AI 工作台 route 必须可携带“入口上下文”（例如 projectId + targetType + targetId + mode），以便页面刷新后仍能恢复当前会话与预览（MVP 可仅恢复目标定位与草稿，不强制持久化完整对话）。

### G. Context Prompt 构造（按入口）
- FR23：系统必须根据入口类型与动作选择对应的 Context Prompt 模板：`Step.Create`、`Step.Optimize`、`Agent.Create`、`Agent.Optimize`、`Asset.Create`、`Asset.Optimize`。
- FR24：Step 入口的上下文至少包含：当前节点信息（type/title/agentId/inputs/outputs/setsVariables）、当前 Step 内容（Optimize 必填）、上下游边信息、workflow variables keys、可用 agent 列表、被引用的 assets 路径（可选包含内容）。
- FR25：Agent 入口的上下文至少包含：当前 Agent 定义（Optimize 必填）、项目内现有 agents 列表（避免重复/冲突）、该 agent 在 workflow nodes 中的引用概览（影响面）、工具策略约束与 schema 要求。
- FR26：assets 入口的上下文至少包含：目标 assets 路径与类型（policy/template/script 等）、现有内容（Optimize 必填）、引用它的 Step 列表/片段（影响面）、路径与扩展名约束。
- FR27：Context Prompt 必须遵循最小化原则，并在工作台展示“本次上下文摘要”（用户能看到将发送哪些对象/文件，MVP 可不支持细粒度勾选，但必须可见）。

### H. Agent 创建与优化
- FR28：系统必须支持 AI Create Agent：基于用户输入生成一个 v1.1 schema 合法的 Agent（persona/prompts/tools/menu 等），可预览并应用到 `agents.json`。
- FR29：系统必须支持 AI Optimize Agent：在保留 `agentId` 不变的前提下优化 persona/prompts/tools 等字段，并可预览/应用。
- FR30：系统必须在应用前校验 agentId 唯一性与 schema 合法性；校验失败时不得写入，并提供可操作的错误信息。
- FR31：工作台必须展示该 Agent 的引用范围（例如被哪些节点引用），帮助用户评估修改影响。

---

## Non-Functional Requirements（非功能需求）

### Performance（性能）
- NFR1：生成/优化请求在正常网络条件下的 P95 响应时间 ≤ 20s（MVP 可放宽但需有超时提示与重试）。
- NFR2：UI 不得因长文本预览导致明显卡顿（预览区域可做折叠/分页/截断策略）。

### Security & Privacy（安全与隐私）
- NFR3：必须支持“只允许自建/本地 OpenAI-compatible 端点”或“完全禁用外部调用”的部署策略。
- NFR4：不得在客户端/日志中泄露密钥、token 或敏感配置。
- NFR5：对发送给 LLM 的上下文必须遵循最小化原则（仅限当前目标对象：Step/Agent/assets 相关内容与必要引用）。

### Reliability（可靠性）
- NFR6：AI 服务不可用时，Builder 的核心编辑/保存/导出流程不受影响。
- NFR7：对 assets 的批量变更必须具备失败可恢复性（部分失败需明确提示）。

### Observability（可观测性）
- NFR8：需要记录 AI 请求的基础事件（成功/失败、耗时、模式 create/optimize、是否包含 assets patch、是否被应用）。

### Usability（可用性）
- NFR9：用户必须能在 1-2 次操作内完成“生成 → 预览 → 应用”闭环。
- NFR10：错误信息必须面向非工程用户可理解（优先描述“哪里不符合规范/如何修复”）。
- NFR11：AI 工作台页面应支持无损返回：用户取消/返回后，原编辑页面的未保存修改不得丢失（避免 AI 打断编辑流）。

---

## Risks / Open Questions（风险与待确认）

1. AI 输出的 Completion（Document-as-State）规则，是否需要强制包含某些固定语句/结构（以适配运行时）？
2. assets patch 的删除操作是否需要二次确认（防误删）？
3. 上下文喂给 LLM 的边界：是否允许带上相邻节点 Step 内容？还是只给 graph 元信息？
4. 组织策略：是否需要“禁止外发 assets 内容/只给路径不给内容”的模式？
5. 成本控制：是否需要每次请求的 token/字数上限，以及超限策略？
6. AI 工作台对话是否需要持久化（用于审计/回溯/复用）？若需要，存储范围与脱敏策略是什么？
7. 预览区是否需要支持“部分应用”（例如只应用 Step，不应用 assets patch；或仅应用 persona，不动 prompts）？

---

## Next Steps（下一步建议）
- 在该 PRD 基础上进入：Architecture（接口、数据结构、存储与鉴权策略）→ Epics & Stories（按 FR/NFR 拆解到可交付迭代）。
