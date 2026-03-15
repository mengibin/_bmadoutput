# Sprint Change Proposal：Runtime 知识库能力增量

日期：2026-03-09  
项目：CrewAgent  
触发方式：Correct Course（Implementation 阶段需求变更导航）  
变更模式：Batch（一次性审阅）

---

## 1. 问题摘要（Issue Summary）

### 1.1 触发问题

当前 Runtime 缺少可治理的本地知识库能力，导致：

- 用户长期偏好无法稳定跨会话复用；
- 项目参考资料（PDF/Word/图片等）缺少统一导入与检索闭环；
- 项目目录删除/迁移后，聊天与知识残留数据缺乏统一语义的清理治理；
- 同类项目复用与跨设备迁移缺少标准路径。

### 1.2 已确认的关键决策

1. 个人知识库：MVP 仅在 `chat` 模式启用。  
2. 项目知识库存储：放在 Runtime 私有目录（`runtime-store/projects/<projectId>/knowledge/`），不放项目根目录。  
3. “清空”语义：定义为“孤儿项目清理”（项目目录不存在但 Runtime 数据仍在），复用既有 orphan 检测链路。  
4. 项目知识需支持迁移（导出/导入，含完整性校验与失败回滚）。

---

## 2. 影响分析（Impact Analysis）

### 2.1 Epic 影响

- 现有 **Epic 7（上下文构建）** 需扩展：增加 chat-only 个人知识注入逻辑与模式隔离保障。  
- 现有 **Story 8.4（孤儿项目恢复）** 能力可复用，但需承接知识目录统计与删除覆盖范围。  
- 当前 `epics.md` 无“知识库”专项 Epic，建议新增独立 Epic（建议编号 `Epic 12`）承载此批需求。

### 2.2 Story 影响

- 需要新增一组可落地故事覆盖：存储结构、导入索引、检索注入、孤儿清理联动、迁移能力、管理入口。
- 不建议将全部需求塞入已有单故事（如 8.4），会导致 AC 过载和回归面失控。

### 2.3 文档冲突/缺口

- `prd.md` 尚未把“个人知识库/项目知识库”列为链接子 PRD，主文档可追溯性不足。  
- `architecture.md` 尚无知识库子系统边界、数据流和故障语义（回滚/重建索引）。  
- `epics.md` 与 `sprint-status.yaml` 尚未纳入知识库故事，实施入口缺失。

### 2.4 技术影响

- `RuntimeStore`：新增 `knowledge/` 生命周期管理；孤儿检测统计项扩展。  
- `chat` 上下文组装：新增个人知识检索注入；`agent/run` 严格跳过。  
- 导入与索引流水线：复用现有多模态提取能力并补 manifest/完整性校验。  
- IPC/UI：新增知识管理入口与迁移操作入口（导出/导入）。

---

## 3. 推荐方案（Recommended Approach）

### 3.1 路径选择

采用 **Direct Adjustment（直接增量）**，不回滚既有实现：

- 先完成增量架构设计（Knowledge Base Delta）；
- 再新增 Epic/Stories；
- 再进入标准 story 生命周期（design/dev/review）。

### 3.2 方案理由

- 现有 orphan 管理链路已成熟（detect/rebind/delete/ignore），可直接复用，风险可控。  
- 需求边界已清晰（personal chat-only、project runtime-private、orphan cleanup 语义明确）。  
- 通过新增独立 Epic 可以避免污染历史故事 AC，减少回归耦合。

### 3.3 风险与控制

- 风险：Chat 上下文膨胀。  
  控制：限额注入 + 命中片段优先。  
- 风险：误把个人知识带入 `agent/run`。  
  控制：模式硬隔离 + 审计日志。  
- 风险：迁移包导入失败导致脏状态。  
  控制：导入事务化 + 回滚。  
- 风险：孤儿删除误操作。  
  控制：二次确认 + 明确影响范围展示。

---

## 4. 详细变更提案（Detailed Change Proposals）

### 4.1 PRD 主文档补链（`_bmad-output/prd.md`）

**OLD**

- Linked Requirement Documents 仅包含：
  - Builder AI
  - Runtime License
  - Runtime Multimodal

**NEW**

- 在 Linked Requirement Documents 增加：
  - `prd-runtime-personal-knowledge-base.md`
  - `prd-runtime-project-knowledge-base.md`

**Rationale**

- 让主 PRD 成为完整索引入口，保证后续架构与实现流程可追溯。

---

### 4.2 增量架构补充（`_bmad-output/architecture.md`）

**OLD**

- 无“知识库子系统”专章，无模式隔离与迁移/回滚语义。

**NEW**

- 新增章节：`Runtime Knowledge Base Delta`
  - 组件边界：`PersonalKBService`、`ProjectKBService`、`KnowledgeIndex`、`KnowledgeMigration`  
  - 存储边界：
    - personal: `@state/kb/personal/*`
    - project: `runtime-store/projects/<projectId>/knowledge/*`  
  - 模式规则：仅 chat 读取 personal；agent/run 禁止 personal 注入。  
  - 孤儿联动：复用 orphan 检测链路，删除覆盖聊天 + knowledge。  
  - 失败语义：索引可重建、迁移可回滚、导入失败不污染目标项目。

**Rationale**

- 先明确边界再拆故事，避免实现阶段反复返工。

---

### 4.3 新增知识库 Epic（`_bmad-output/epics.md`）

**OLD**

- `epics.md` 无知识库专属 Epic（当前以 Epic 1~10 + MDE 为主）。

**NEW（建议新增 Epic 12）**

`Epic 12: Runtime Knowledge Base (Personal + Project, Local First)`

建议拆分故事：

1. `12-1-personal-kb-storage-and-candidate-commit`  
2. `12-2-chat-only-personal-kb-retrieval-and-injection`  
3. `12-3-project-kb-import-extract-index-search`  
4. `12-4-project-kb-orphan-cleanup-integration`  
5. `12-5-project-kb-export-import-migration`  
6. `12-6-kb-management-ui-and-index-rebuild`

**Rationale**

- 将“个人/项目/迁移/孤儿治理”拆成独立可验收单元，降低耦合风险。

---

### 4.4 Sprint 跟踪对齐（`_bmad-output/implementation-artifacts/sprint-status.yaml`）

**OLD**

- 无 `epic-12` 与知识库故事状态项。

**NEW**

- 增加：
  - `epic-12: backlog`
  - `12-1...: backlog`
  - `12-2...: backlog`
  - `12-3...: backlog`
  - `12-4...: backlog`
  - `12-5...: backlog`
  - `12-6...: backlog`
  - `epic-12-retrospective: optional`

**Rationale**

- 让实施阶段工具（`sprint-status/create-story`）可正确路由到知识库工作流。

---

### 4.5 既有 8.4 的关系说明（不直接改 AC）

**OLD**

- 8.4 已实现 orphan 检测与清理（聊天维度已覆盖）。

**NEW**

- 不强改 8.4 AC；在 Epic 12 下新增“知识目录联动”故事（12-4）承接。  
- 8.4 作为基础设施复用点，不作为知识库需求主承载故事。

**Rationale**

- 避免历史故事范围膨胀，减少对已评审资产的破坏性修改。

---

## 5. 实施交接（Implementation Handoff）

### 5.1 变更范围分类

**Moderate（中等变更）**：需要 backlog 重组与架构增量，但无需回滚既有实现。

### 5.2 交接对象与职责

- PM/Architect：
  - 确认本提案；
  - 更新 `architecture.md` 增量章节；
  - 在 `epics.md` 增加 Epic 12 与故事切分。  
- SM：
  - 更新 `sprint-status.yaml`；
  - 触发 `create-story/design-story` 进入开发节奏。  
- DEV：
  - 按故事顺序实现并通过 `code-review` 质量门。

### 5.3 完成判定

- 主 PRD 链接完整；
- 架构增量文档完成并评审通过；
- Epic/Story 已进入 sprint-status；
- 至少首个知识库故事进入 `ready-for-design`。

---

## 附录 A：Checklist 执行结果（摘要）

- [x] 触发问题明确且与当前实施阶段相关。  
- [x] 核心文档已可用（PRD、架构、epics、sprint-status）。  
- [x] 识别到受影响 artifact 并给出 old/new 对照。  
- [x] 给出实施路由和责任人。  
- [!] 待用户批准后执行文档落地（架构增量、epics/sprint-status 更新）。
