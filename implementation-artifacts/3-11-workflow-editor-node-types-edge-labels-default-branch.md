# Story 3.11: Workflow Editor Full-Screen + Node Types + Edge Labels + Variables + Artifacts

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **Creator**,  
I want the workflow editor to be full-screen and v1.1-ready (node types, edges, variables, artifacts),  
so that I can design complex workflows and later export step/workflow files that match the v1.1 spec.

## Acceptance Criteria

1. **Given** I open a workflow in WorkflowEditor  
   **When** I view the page layout  
   **Then** the editor uses the full viewport width (no max-width container)  
   **And** the canvas is the primary area and does not get squeezed by side panels (panels can collapse or resize)

2. **Given** I am in a workflow editor  
   **When** I view the palette  
   **Then** the palette shows node types (at least: `step`, `decision`, `merge`, `end`, `subworkflow`)  
   **And** I can drag a node type into the canvas to create a node of that type

3. **Given** I select a node  
   **When** I edit node settings  
   **Then** I can configure v1.1-aligned fields (at least: `nodeId`, `type`, `title`, `agentId`, `inputs`, `outputs`, `setsVariables`)  
   **And** those fields persist after save + refresh

4. **Given** I create or edit an edge between nodes  
   **When** I set an edge label (and optional `conditionText`)  
   **Then** the label is saved with the edge  
   **And** if I do not set a label, the system uses a deterministic default label

5. **Given** a decision node has multiple outgoing edges  
   **When** I select one outgoing edge as default  
   **Then** that choice is saved and can later be exported as `isDefault: true`

6. **Given** nodes exist on the canvas  
   **When** I save and reload  
   **Then** each node keeps a stable `nodeId` (not derived from topological ordering)  
   **And** the `nodeId` format is validated against v1.1 (pattern) with clear error messages

7. **Given** I am editing a workflow  
   **When** I open workflow-level settings  
   **Then** I can manage “workflow global variables”  
   **And** variables persist per-workflow (intended to map to `workflow.md` frontmatter `variables`)

8. **Given** I am in ProjectBuilder or WorkflowEditor  
   **When** I open the project-level artifacts manager  
   **Then** I can create/rename directories under `artifacts/` for this project  
   **And** when configuring a node’s `outputs`, the UI helps me choose/validate paths like `artifacts/<dir>/<file>` (runtime path is `@project/artifacts/...`)

## Design

### Summary

- 本 story 目标：把 WorkflowEditor 的“画布/编辑模型”升级为 v1.1-ready，同时补齐后续 Step 引用所需的 **Variables** 与 **Artifacts** 基础能力。
- Editor 全屏布局 + 画布优先（可折叠/可拖拽调整的侧边栏），避免画布被挤压。
- Node/Edge 能表达 v1.1 step frontmatter 所需信息：`type/nodeId/title/agentId/inputs/outputs/setsVariables` + `transitions(label/isDefault/conditionText)`。
- Workflow 级提供全局变量（参考 v1.1 `workflow.md` frontmatter `variables`），支持后续 step 指令引用 `variables.<key>`。
- Project 级提供 artifacts 目录管理（参考 step 的 `outputs: ["artifacts/..."]`），为后续 step 产物路径选择/校验做支撑。
- 参考 v1.1 micro 示例（`step-01-select-story.md`）：
  - `outputs: ["artifacts/create-story/target.md"]`
  - `setsVariables: ["storyKey","epicNum","storyNum","storyTitle","sprintStatusPath"]`
  - Step 内容中用 `@project/artifacts/...` 与 `variables.<key>` 做引用

### UX / UI

- 布局（WorkflowEditor）：
  - 全屏宽度（避免外层 `max-width` 容器限制），高度以 viewport 为基准。
  - 画布居中且占最大空间；左右侧栏（Palette / Inspector）支持：
    - collapse（收起为 icon rail）
    - resize（拖拽调整宽度）
  - 目标：在常见 13–16 寸屏幕下画布可视面积显著增加，拖拽/连线不受遮挡。

- Palette：
  - Node types：`step/decision/merge/end/subworkflow`（最低集）
  - 拖入画布后，默认生成：
    - 稳定 `nodeId`（可编辑但需校验）
    - `type`
    - 基础字段默认值（如空 `inputs/outputs/setsVariables`）

- Inspector（右侧）建议 Tabs：
  - **Node**：选中节点的编辑
    - 通用字段：`nodeId`（pattern 校验）、`title`、`agentId`（下拉取 project agents）
    - v1.1 对齐字段：`inputs`、`outputs`、`setsVariables`
    - decision：出边列表（每条边可编辑 `label` / `conditionText`），并设置 default
    - subworkflow：选择 target workflow（引用 workflow id）
  - **Workflow**：workflow-level settings
    - `variables`：Key/Value 列表（MVP：string value；高级模式允许 JSON value）
    - 显示引用提示：`variables.<key>`
  - **Artifacts**（可选：也可放在 ProjectBuilder 主页面）：
    - 展示 project `artifacts/` 目录树（目录级别即可）
    - 支持：新建目录 / 重命名目录（MVP 不做删除，避免已有 outputs 引用悬空；删除可在后续 story）

### Data / Storage

- Graph model（存于 `graph_json`）：
  - Node：
    - `type`（`step/decision/merge/end/subworkflow`）
    - `nodeId`（稳定，后续导出 `steps/<nodeId>.md`）
    - `data` 承载 v1.1 step 字段：`title/agentId/inputs/outputs/setsVariables`
  - Edge：
    - `label`（用于导出 transitions.label）
    - `conditionText`（可选）
    - `isDefault`（用于导出 transitions.isDefault；或 decision node 指向默认 edgeId）

- Workflow variables（per workflow）：
  - 存储为 workflow 级 settings（推荐映射到 `workflow_md` frontmatter `variables`，以便后续导出/校验）
  - `setsVariables` 的编辑体验：
    - 从已有 variables key 下拉选择
    - 新增 key 时允许“一键加入 workflow variables”（避免引用丢失）

- Project artifacts directories（per project）：
  - 存储为 project 级数据（例如 `workflow_packages` 新字段 `artifacts_json`/`project_settings_json`；实现时以现有存储风格为准）
  - `outputs` 校验：
    - 路径必须以 `artifacts/` 开头（保存时校验 + 友好提示）
    - 目录不存在时提示创建（或自动创建目录记录）

### Deterministic Defaults

- edge.label 默认策略（当用户未填写）：
  - 单出边：`next`
  - 多出边：`branch-1`, `branch-2`, ...（基于 edge 创建顺序，保持稳定）
- decision default：
  - 默认第一条出边为 default（用户可改）

### Notes

- 本 story 只负责“编辑与存储模型 + UX”；v1.1 `workflow.md` + `steps/<nodeId>.md` + `workflow.graph.json` 的导出生成在后续导出相关 story（如 Story 3.13/3.14）。

## Test Plan

- Layout：
  - Editor 全屏宽度生效；缩放窗口后画布仍为主要区域；侧栏可收起/可调整宽度
- Node/Edge：
  - palette 拖入不同 node type → 保存/刷新后 type 不丢
  - edge label/conditionText 可编辑并持久化；label 缺省时默认策略稳定
  - decision 多出边时可设置 default 并持久化
  - nodeId 稳定且 pattern 校验生效（非法输入阻止保存并提示）
- Variables：
  - Workflow variables 可增改并持久化；节点 `setsVariables` 支持选择/新增并与 workflow variables 同步
- Artifacts：
  - project artifacts 目录可新建/重命名并持久化
  - node outputs 支持选择/校验 `artifacts/...` 路径（提示 `@project/artifacts/...`）

## Tasks / Subtasks

- [x] 1) 前端：WorkflowEditor 全屏布局 + 画布面积提升（侧栏可收起/可 resize）
- [x] 2) 前端：palette 支持多 node types + 默认 node data（含稳定 `nodeId` 生成）
- [x] 3) 前端：Inspector/Node settings（v1.1 step 字段：`inputs/outputs/setsVariables`）
- [x] 4) 前端：edge label / conditionText 编辑与持久化（含 deterministic default）
- [x] 5) 前端：decision default 分支选择与持久化
- [x] 6) 前端：Workflow settings → 全局 variables 管理（与 `setsVariables` 联动）
- [x] 7) 前端：Project artifacts 目录管理 UI（目录树 + 新建/重命名）
- [x] 8) 后端：graph_json / workflow_md / project settings 存储支持新增字段（variables + artifacts directories）

## References

- `_bmad-output/epics.md`（Epic 3 / Story 3.11）
- `crewagent-runtime/spec/bmad-package-spec/v1.1/examples/create-story-micro/steps/step-01-select-story.md`
- `crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/step-frontmatter.schema.json`
- `crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/workflow-frontmatter.schema.json`
- `crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/workflow-graph.schema.json`
