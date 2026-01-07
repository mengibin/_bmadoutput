# Validation Report: Story 4.4 - Compose Complete Prompt for LLM

**Date**: 2026-01-01  
**Story**: 4.4 - Compose Complete Prompt for LLM  
**Status**: ✅ **APPROVED**（可进入 `design-story`，有少量建议）

---

## Summary

Story 4.4 现在的结构与约束清晰，已对齐架构文档与 `prompt-composer-examples.md` 的分层输出方式，并补齐了安全要求（mount alias、禁止真实路径泄露）。适合进入 `design-story` 做最终收敛（尤其是 RUN_DIRECTIVE/NODE_BRIEF 的字段细节与错误契约）。

---

## Validation Checklist

### ✅ Story Structure
- [x] Title + `Status` 字段符合现有 story 结构
- [x] User Story（role/action/benefit）完整
- [x] Acceptance Criteria 可验证且覆盖核心需求（分层、协议块、安全）
- [x] Design / Tasks / References 具备可执行性与可追踪性

### ✅ Technical Completeness
- [x] 明确 Prompt 分层顺序（Rules → Policy → Persona → Directive → Brief → Input）
- [x] 明确数据来源（agents.json / graph / step / @state/workflow.md）
- [x] 明确 persona 继承规则（`effectiveAgentId = node.agentId ?? run.activeAgentId`）
- [x] 明确 machine-parseable shells 需要对齐协议文档
- [x] 明确安全底线（只允许 `@project/@pkg/@state`；禁止绝对路径）

### ✅ Alignment with Project
- [x] 对齐 `_bmad-output/architecture/runtime-architecture.md` 中 PromptComposer 定义
- [x] 对齐 `_bmad-output/tech-spec/prompt-composer-examples.md`（推荐模型行为：先 `fs.read` state/graph/step）
- [x] 对齐 Epic 4 Story 4.4 的验收项（prompt 组成 + shells + variables 注入与不泄露真实路径）

---

## Findings

### Strengths
1. **防灾导向明确**：强调 mount alias + 不泄露真实路径，避免隐私与安全事故。
2. **不强行塞全文**：明确“通过 `@pkg/<node.file>` + `fs.read` 获取 step 指令”，与 examples 一致，避免 token 爆炸。
3. **可集成性好**：把 `RuntimeStore.getWorkflowState/loadWorkflow/getAgentDefinition` 作为单一数据源，减少重复解析。

### Recommendations (Minor)

#### 1) 明确 graph 路径来源（避免硬编码）
在 `design-story` 中固定：RUN_DIRECTIVE 里的 `graph: @pkg/<graphRelPath>` 必须来自 workflow 选择结果（manifest/entry/workflows[]），禁止写死 `@pkg/workflow.graph.json`。

#### 2) 明确 NODE_BRIEF 的最小字段集合
建议在 `design-story` 明确 `NODE_BRIEF` 至少包含：
- `currentNodeId`（含 node type）
- `step file`（`@pkg/<node.file>`）
- `inputs/outputs`（若来自 step frontmatter 的映射规则，需定义格式）
- `allowed next`（含 label/default/conditionText）

#### 3) 增加“绝对路径泄露”测试门槛
建议在单测中对 `messages[].content` 做扫面断言：
- 不包含 `/Users/`、`C:\\`、`runtime-store/` 等关键片段
- 仅允许出现 `@project/@pkg/@state` 前缀路径

---

## Verdict

**APPROVED**：Story 4.4 已达到 “ready-for-design” 的质量门槛；下一步建议直接运行 `design-story 4.4`，把协议字段与错误契约收敛到最终可开发形态。

---

## Next Steps
1. `design-story 4.4`：细化 RUN_DIRECTIVE/NODE_BRIEF 字段与格式（对齐协议文档）。
2. `dev-story 4.4`：基于现有 `PromptComposer` 实现补齐 graph 路径、allowedNext 生成、以及不泄露真实路径的强校验。

