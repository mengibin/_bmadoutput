# Story 3.10: ProjectBuilder Agent Management (v1.1 Required Fields + Stable `agentId`)

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **Creator**,  
I want to create and edit agents at the project level with v1.1-required fields and stable ids,  
so that workflows can reference agents via `agentId` and schema validation won’t fail later.

## Acceptance Criteria

1. **Given** I am in ProjectBuilder  
   **When** I create an agent  
   **Then** it is saved with a stable `agentId` and appears in the agents list  
   **And** required v1.1 fields are present (via user input or defaults), including at least:  
   - `metadata.title`  
   - `metadata.icon`  
   - `persona.principles`

2. **Given** an agent exists  
   **When** I edit and save it  
   **Then** the updates persist and are reflected across workflows in this project

3. **Given** I assign an agent to a node in a workflow editor  
   **When** I save and reload  
   **Then** the node keeps the assignment by `agentId` (not by free-text name)

## Design

### Summary

- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-project-builder.md`
- Agents 数据升级为 v1.1 `agents.json` schema（manifest 结构 + 每个 agent 的必填字段），并在 ProjectBuilder 以表单创建/编辑。
- 生成稳定 `agentId`（符合 schema pattern）并持久化；rename 不改变 id；workflows / nodes 通过 `agentId` 绑定 agent。
- legacy 兼容：支持从旧 `agentsJson`（数组）与旧 node `agent`（自由文本）自动迁移/映射到 `agentId`。

### UX / UI

- ProjectBuilder（`/builder/[projectId]`）Agents 面板：
  - 列表展示：`icon` + `metadata.title`（或 `metadata.name`）+ `persona.role`；并显示只读 `id`（`agentId`）。
  - 操作：`新建 Agent` / `编辑`（MVP 不做删除，避免 workflow 引用悬空；删除可在后续 story）。
  - 新建/编辑弹窗字段（最小满足 v1.1）：
    - `metadata.name`（必填，1–100）
    - `metadata.title`（必填，1–100，默认= `persona.role` 或 `metadata.name`）
    - `metadata.icon`（必填，默认 `🧩`）
    - `persona.role`（必填，1–200）
    - `persona.identity`（必填，多行文本，默认可从旧 `persona` 迁移）
    - `persona.communication_style`（必填，1–200）
    - `persona.principles`（必填：至少 1 条；UI 用 textarea “每行一条”，保存为 `string[]`）
    - `tools`（高级折叠区，默认写入，满足 schema）：
      - `tools.fs.enabled: true`
      - `tools.mcp.enabled: false`（可选：允许填 `allowedServers`）
    - `agentId`：
      - 新建时自动生成（只读展示）
      - 编辑时只读（rename 不改变 id）
- WorkflowEditor（`/editor/[projectId]/[workflowId]`）节点绑定 agent：
  - Node settings 将 `agent` 输入改为下拉选择（来源：Project agents；展示 title/name，值为 `agentId`）。
  - Node data 保存 `agentId`（不再保存自由文本 name）；节点卡片显示选中 agent 的 `icon/title`（或 id）。

### API / Contracts

- 沿用 Project 级 agents API（共享给多个 workflows）：
  - `GET /packages/{projectId}`：返回 `agentsJson`（string，内容为 v1.1 manifest JSON，或 legacy 数组）
  - `PUT /packages/{projectId}/agents`
    - Request：`{ agents: AgentsManifestV11 }`
    - Success：返回 project detail（同 `GET /packages/{projectId}` 的 `data` 结构）
    - 失败：400 `VALIDATION_ERROR`（字段缺失/长度超限/agentId 不合法/重复等），404 `PACKAGE_NOT_FOUND`
- 兼容策略（MVP）：
  - 后端在 `GET /packages/{projectId}` 时若检测到 legacy `agentsJson`（数组），可原样返回；前端需同时支持两种解析。
  - 在 `PUT /agents` 写入时统一落库为 v1.1 manifest（便于后续导出与 schema 校验）。

### Data / Storage

- 继续使用 `workflow_packages.agents_json`（project 级共享）存储 agents manifest：
  - v1.1 结构（推荐落库结构）：
    - `{ schemaVersion: "1.1", agents: Agent[] }`
  - 每个 Agent 的必填字段（来自 `crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/agents.schema.json`）：
    - `id`
    - `metadata.name/title/icon`
    - `persona.role/identity/communication_style/principles`
    - `tools`（建议始终写入默认值）
- `agentId` 生成规则（稳定）：
  - 以 `metadata.name` slugify（kebab-case，保证首字符为字母/数字）生成候选 id
  - 若冲突则追加 `-2/-3/...`（或短后缀），直到唯一
  - 创建后 id 不变（rename 仅改 metadata，不改 id）

### Errors / Edge Cases

- Agents 表单校验：
  - 缺失必填/超长：前端即时提示 + 后端 400 `VALIDATION_ERROR`（details 指向字段）
  - `agentId` 冲突：前端创建时自动避冲突；后端兜底校验重复 id
  - principles 为空：阻止保存并提示“至少 1 条原则”
- Node agent 迁移：
  - 若旧 node 存 `agent`（name 字符串）：加载时按优先级尝试映射：
    1) `agent.id` 精确匹配
    2) `metadata.name`（case-insensitive）
    3) `metadata.title`（case-insensitive）
  - 映射失败：置空并在 editor 顶部提示“存在无法映射的 agent 引用”

### Test Plan

- Backend（pytest）：
  - `PUT /packages/{projectId}/agents`：缺失字段/超长/重复 id → 400；越权 → 404 `PACKAGE_NOT_FOUND`
  - `GET /packages/{projectId}`：可返回 v1.1 manifest（string），且与保存一致
- Frontend（手工冒烟）：
  - 新建 agent（仅填最少字段）→ `agentId` 自动生成且稳定；刷新后仍存在
  - 编辑 agent（改 title/icon/principles）→ 列表与 editor 下拉都更新，但 `agentId` 不变
  - 在 workflow node 选择 agent → 保存/刷新后仍保持（按 `agentId`）
  - legacy 项目：旧 agents 数组/旧 node agent(name) 可自动映射；失败则有明确提示

## Tasks / Subtasks

- [x] 1) 前端：Agent 管理 UI 支持 v1.1 必填字段（含默认值策略）
- [x] 2) 前端：生成稳定 `agentId` 并持久化
- [x] 3) 前端：node 绑定从 `agent(name)` 切换为 `agentId`
- [x] 4) 后端：存储 project 级 agents（供多个 workflow 共享）

## References

- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-project-builder.md`
- `_bmad-output/epics.md`（Epic 3 / Story 3.10）
- `crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/agents.schema.json`

## Dev Agent Record

### Agent Model Used

GPT-5.2 (Codex CLI)

### Debug Log References

- Backend:
  - `crewagent-builder-backend/.venv/bin/pytest -q`
- Frontend:
  - `npm run lint`
  - `npm run build`

### Completion Notes List

- ProjectBuilder Agents 面板支持创建/编辑 v1.1 agents（含 `metadata.title/icon`、`persona.principles` 等必填字段与默认值策略），并写入 v1.1 manifest（`{ schemaVersion, agents[] }`）。
- 创建时基于 `metadata.name` 自动生成稳定 `agentId`；编辑时保持 `agentId` 不变（rename 不改 id）。
- WorkflowEditor 节点绑定改为保存 `agentId`，并支持从旧的 `agent` 文本自动映射（id/name/title 优先级）；映射失败会提示 warning。
- 后端 `PUT /packages/{projectId}/agents` 升级为 v1.1 manifest 校验与落库（含 `agentId` pattern + 去重）。
- Code Review 修复：避免解析丢数据、阻止在 `agentsJson` 非法时覆写、legacy agent 映射会触发自动保存、补齐后端负例校验测试、抽取共享 `agentId` 工具函数。

### File List

- `_bmad-output/implementation-artifacts/3-10-project-builder-agent-management-v1-1-required-fields.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `crewagent-builder-backend/app/schemas/workflow_package.py`
- `crewagent-builder-backend/app/services/package_service.py`
- `crewagent-builder-backend/tests/test_packages.py`
- `crewagent-builder-frontend/src/app/builder/[projectId]/page.tsx`
- `crewagent-builder-frontend/src/app/editor/[projectId]/[workflowId]/page.tsx`
- `crewagent-builder-frontend/src/lib/utils.ts`

### Change Log

- 前端：新增/编辑 v1.1 Agent（含稳定 `agentId` 生成与保存）并在 ProjectBuilder 列表展示。
- 前端：WorkflowEditor 节点 agent 绑定切换为 `agentId`，并兼容旧数据映射与提示。
- 后端：agents 更新接口与存储升级为 v1.1 manifest，并更新测试用例。
- Code Review：加强 agentsJson 解析/保存安全性与 legacy 迁移持久化；补齐后端负例测试覆盖。
