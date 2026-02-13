# Builder AI 交付路线图（BMAD Method）

**Created:** 2026-02-08  
**Status:** In Progress (BAI-4 Multi-Target Backend Completed, Entering BAI-5 Prep)  
**Input Docs:**  
- `_bmad-output/prd-builder-AI.md`  
- `_bmad-output/architecture/builder-ai-llm-interaction-architecture.md`  
- `_bmad-output/epics-builder-ai.md`  
- `_bmad-output/tech-spec/builder-ai-implementation-spec.md`  

---

## 1. 目标与原则

本路线图用于把 Builder AI 从“文档设计”推进到“可上线能力”，并确保以下原则：

1. **先控风险再扩范围**：先打通用户级 LLM 配置与 validate-only，再开启手动 apply。
2. **读写分离**：读取可跨对象，写入必须通过 `changeSet -> stage -> validate -> manual apply`。
3. **统一入口体验**：Workflow/Step/Agent/Asset 四入口统一到 AI Workbench。
4. **可回退上线**：统一新链路，旧 `step-draft` 走受控停用；通过 feature flag 做分阶段灰度。

---

## 2. 里程碑总览

### Milestone M1：基础能力可用（Profile + Session Skeleton）

**目标**
- 用户级 LLM 配置生效，能创建 AI Session 并完成多轮消息。
- 仅支持建议与校验，不执行真实写入。

**覆盖 Stories**
- BAI-1.1 ~ BAI-1.4
- BAI-2.1, BAI-2.3
- BAI-3.1, BAI-3.2（validate-only）

**退出标准**
- `GET/PUT/TEST /users/me/llm-profile` 全量可用。
- `POST /ai/sessions` + `/messages` 可返回结构化建议。
- `builder.change.validate` 可返回结构化错误与 hints。

---

### Milestone M2：后端闭环可写（Staging + Validate + Manual Apply）

**目标**
- 形成后端原子提交能力，替代前端多请求写入。
- 建立 overlay 草稿语义与 revision 安全契约（单用户阶段先提示可恢复，原子锁定延后）。

**覆盖 Stories**
- BAI-2.4, BAI-2.5, BAI-2.6
- BAI-3.3, BAI-3.4, BAI-3.5

**退出标准**
- `builder.change.stage/validate/apply/discard` 语义完整可用，apply 仅手动触发。
- `builder.change.apply` 在单事务内写入 workflow/step/agent/asset。
- revision mismatch 返回结构化 warning/details，并在 Workbench 可恢复（硬冲突门禁延后）。
- legacy `step-draft` 已受控停用，Step AI 仅走 Workbench 新链路。

---

### Milestone M3：前端统一工作台可用（四入口接入）

**目标**
- 新建 AI Workbench 路由，四类入口统一交互。
- 用户能完成“建议 -> 预览 -> 应用 -> 返回”闭环。

**覆盖 Stories**
- BAI-4.1 ~ BAI-4.5
- BAI-4.6（刷新恢复）已取消（de-scope），不进入当前实施范围

**退出标准**
- Workflow/Step/Agent/Asset 入口都可跳转同一 Workbench 页面。
- 预览区展示影响范围与验证结果。
- Cancel 返回不丢原页面未保存编辑。

---

### Milestone M4：安全与可观测上线（灰度 + 审计）

**目标**
- 完成敏感信息保护、审计、核心指标看板与告警。
- 支持 canary 与 feature flag 回退（不恢复 legacy optimizer）。

**覆盖 Stories**
- BAI-5.1 ~ BAI-5.4

**退出标准**
- 审计链路可追踪 sessionId / changeSetId / userId。
- 关键指标可见：成功率、P95、冲突率、应用转化率。
- canary 回退路径验证通过。

---

## 3. 分阶段实施顺序（建议）

1. **Phase A（1-2 周）**：M1  
2. **Phase B（1-2 周）**：M2  
3. **Phase C（1-2 周）**：M3  
4. **Phase D（1 周）**：M4  

> 注：可根据团队规模并行推进。建议后端先行 M1/M2，前端在 M2 中后段开始 M3。

---

## 4. 并行泳道（Parallel Workstreams）

### Workstream 1：Backend Core
- owner：后端工程师
- 范围：BAI-1, BAI-2, BAI-3, BAI-5（后端部分）
- 关键依赖：数据库 migration、鉴权中间件、组织策略接口

### Workstream 2：Frontend Workbench
- owner：前端工程师
- 范围：BAI-4 + Profile 页优化（FR17）
- 关键依赖：Session API 与 Validate/Apply API 合约稳定

### Workstream 3：Quality & Release
- owner：QA / SRE
- 范围：回归、观测、灰度发布、回滚演练
- 关键依赖：审计事件与 metrics 埋点完成

---

## 5. 版本门禁（Release Gates）

### Gate G1（M1->M2）
- Profile API 覆盖单元测试与集成测试。
- Session 对话含 ToolCall 的最小闭环跑通。

### Gate G2（M2->M3）
- Apply 原子提交稳定，mismatch 提示与回滚路径验证通过。
- 关键错误码定义冻结（对前端可用）。

### Gate G3（M3->M4）
- 四入口 UX 一致性评审通过。
- 取消/返回无损编辑通过回归。

### Gate G4（上线）
- 安全红线通过（密钥脱敏、鉴权、越权测试）。
- observability dashboard 与报警生效。
- canary 观察窗口通过并可控回退。

---

## 6. 关键风险与应对

1. **风险：LLM 返回不稳定导致校验失败率高**  
   **应对**：强化 `Layer-2 Output Contract` + repair loop + 样例库。

2. **风险：跨对象写入导致脏数据**  
   **应对**：必须单事务 apply + stage/validate 门禁 + 失败回滚；数据库级原子锁定在后续版本补齐。

3. **风险：上下文过大导致耗时/成本不可控**  
   **应对**：context budget 策略 + 摘要化非目标对象。

4. **风险：统一新链路切换引发回归**  
   **应对**：feature flag + canary + read-only 降级策略。

---

## 7. 产物清单（交付时需齐套）

1. 数据库迁移脚本（`user_llm_profiles`, `ai_sessions`, `ai_messages`, `ai_change_sets`, revision 字段）
2. API 文档（Profile / Sessions / Apply / Errors）
3. Prompt Composer 规范与测试样例
4. Workbench 页面与四入口接线清单
5. 安全与审计说明（脱敏策略、事件字典）
6. 上线手册（灰度、回退、告警阈值）

---

## 8. 当前推进焦点（2026-02-12）

1. `BAI-1.1 ~ BAI-1.5` 已按新架构口径完成同步与回归：provider 枚举收敛为 `disabled|openai-compatible`，Profile 生效主链路为 session/messages。
2. `BAI-2.1 ~ BAI-2.6 / BAI-3.2 / BAI-3.3 / BAI-3.4 / BAI-3.5 / BAI-4.2 / BAI-4.4` 已完成开发与回归，状态更新为 `done`；`BAI-4` 重新打开补齐后端多 target messages 链路。
3. 主链路已收敛为 `session + tool loop + stage/validate + manual apply`，legacy `step-draft` 已受控退场（`410 AI_STEP_DRAFT_DEPRECATED`）。
4. 本轮新增语义已落地：Prompt 仅放 asset path（内容按需 `builder.asset.read`）、历史注入为“最近 6 条原文 + 更早 1 条摘要”、会话 working copy 生效、apply 后 session 保持 `active`。
5. 后端全量测试通过（`188 passed`），前端测试通过（`34 passed`），lint 无 error（仅历史 warning）。
6. `BAI-4.6`（刷新恢复）已取消（de-scope，不做 session refresh restore）；`BAI-4.7 ~ BAI-4.10` 已完成开发与回归并更新为 `done`（后端 workflow/agent/asset message 路径与四 target 回归已落地），下一阶段进入 Epic `BAI-5`（安全、审计、可观测与灰度上线）。
7. BAI-2 关闭补充（代码审查问题已清零）：
   - Prompt composer 运行链路统一为 `ComposeInput + compose_messages`。
   - Domain persona 已注入 step 主链路（不再 `domainProfile=None`）。
   - Context envelope + budget trim 已在运行时生效，且顶层键校验改为顺序无关。
8. 稳定性热修复已完成并回归：
   - 修复 Step 变更在 Editor autosave 链路中的文件路径漂移（保留 `stepFilePath/node.file`，避免 apply 后显示与实际落库不一致）。
   - 修复 Step frontmatter YAML 生成安全性（字符串转义 + list 语法规范化），避免 `Unable to parse Step file`。
   - 修复 Workbench 新建 session 前 project 级 working copy 清理，避免跨会话残留 overlay 干扰 diff 与编辑视图。
9. BAI-4.7 ~ BAI-4.10 code-review 闭环已完成：
   - `/messages` 目标上下文校验失败不再写入用户消息，避免历史污染。
   - `AI_TARGET_CONTEXT_REQUIRED` 错误码已按契约落地并回归。
   - 前端 Workbench 消息发送已去除 step-only 限制，支持 workflow/step/agent/asset 四目标上下文发送。
