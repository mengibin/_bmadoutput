# Tech Spec: Story 4.8 - Create Run Folder (RuntimeStore) & Initialize State

**Created:** 2026-01-03  
**Status:** Ready for Development  
**Source Story:** `_bmad-output/implementation-artifacts/4-8-create-execution-project-folder.md`

## Overview

### Problem Statement

当前 Runtime 的“Start Workflow / Start Agent”在 Renderer 侧仅生成 `runId` 并把 run 写入 `appStore`，但并没有在 **RuntimeStore** 中创建真正的 run 文件夹，也没有初始化 run-scoped 的 `@state/workflow.md` 与审计日志文件。因此：

- `@state` mount 无法做到“每次执行一个隔离的 state/logs”（Story 4.6/4.7/5.3 依赖）。
- Crash/Resume 等能力无法只靠 `@state/workflow.md` 恢复（Epic 4/5 目标）。
- 多 workflow 包（`bmad.json.workflows[]`）启动时缺少统一的“workflowRef → workflow.md/graph”解析与校验。

### Solution

在 Electron Main 增加一个专职的 `RunManager`（新文件 `crewagent-runtime/electron/services/runManager.ts`），负责：

1) 解析/校验 `workflowRef`（支持多 workflow 与 entry fallback）  
2) 创建 run 文件夹结构（run-scoped `state/` + `state/logs/`）  
3) 从选定 workflow 的 `workflow.md` 拷贝为 run state，并注入/重置必要 frontmatter（`runId/activeAgentId/workflowRef/updatedAt/...`）  
4) 维护 project 级 `runsIndex.json`（用于列出/恢复最近 run）  
5) 通过 IPC 暴露 `runs:create`、`runs:list`，并让 UI 在点击 Start 时调用它（不再在 Renderer 自行生成 runId）

### Scope (In / Out)

**In scope**
- Main 侧实现 `RunManager.createRun()` 与 `RunManager.listRuns()`。
- 生成 run-scoped `@state/workflow.md`（从选定 workflow 的 `workflow.md` 拷贝并注入字段）。
- 创建/确保 `@state/logs/execution.jsonl` 存在（空文件即可）。
- 写入/更新 `<RuntimeStoreRoot>/projects/<projectId>/runsIndex.json`（原子写）。
- 通过 IPC 暴露 run 创建与列表能力，并在 `PackagePage` 启动流程时调用。
- 多 workflow 解析与校验（严格遵守 Story 4.8 的规则）。

**Out of scope / Deferred**
- 启动 ExecutionEngine 进行实际跑流程（后续故事/集成点）。
- Run 的 pause/resume UI 交互（Epic 5 相关）。
- 对 `@state/workflow.md` 的 graph transition 校验（Story 4.7）。
- Conversation/Message 持久化绑定 run（Story 5.8）。

## Context for Development

### Codebase Patterns

- Electron Main IPC handler 在 `crewagent-runtime/electron/main.ts`（`ipcMain.handle(...)`）。
- Renderer 通过 `crewagent-runtime/electron/preload.ts` 暴露的 `window.ipcRenderer.*` 调用 IPC。
- 包元数据与 workflow/graph/agents 路径已在 `RuntimeStore.importPackage()` 时解析为 `PackageMetadata.workflows[]`（支持多 workflow）。
- Run-scoped state 目录已有 helper：`RuntimeStore.ensureRunStateDir(projectRoot, runId)`。
- Frontmatter 解析库已在 Main 使用：`gray-matter`（`ExecutionEngine`/`FileSystemToolHost`/`RuntimeStore` 均使用）。

### Files to Reference

- Story: `_bmad-output/implementation-artifacts/4-8-create-execution-project-folder.md`
- Package Spec（多 workflow 语义）：`crewagent-runtime/spec/bmad-package-spec/v1.1/README.md`
- RuntimeStore（package/workflow/state/run-state-dir）：`crewagent-runtime/electron/stores/runtimeStore.ts`
- ToolHost 审计日志落盘：`crewagent-runtime/electron/services/fileSystemToolHost.ts`
- IPC wiring patterns：`crewagent-runtime/electron/main.ts`, `crewagent-runtime/electron/preload.ts`
- UI 启动入口（当前为占位 runId）：`crewagent-runtime/src/pages/PackagePage/PackagePage.tsx`

### Technical Decisions

1) **多 workflow 解析规则（严格）**
- 若 `bmad.json.workflows[]` 非空：`workflowRef` 必填且必须命中某个 `id`。
- 若 `bmad.json.workflows[]` 缺失或为空：允许 `workflowRef` 为空，回退到 entry；若提供了 `workflowRef`，必须等于“entry 对应的派生 workflow id”（可直接使用 `RuntimeStore.getPackage(packageId).workflows[0].id` 作为 derived id）。

2) **`runsIndex.json` 作为 Run 列表真源（JSON 数组）**
- 位置：`<RuntimeStoreRoot>/projects/<projectId>/runsIndex.json`
- 内容：`RunMetadata[]`（按创建时间追加；读取时可排序）
- 更新采用原子写（tmp → rename），避免崩溃导致半写入。

3) **Run 初始化时的 frontmatter 注入策略**
- 以 package 内选定 workflow 的 `workflow.md` 作为模板。
- 写入 run-scoped `@state/workflow.md` 时：
  - 强制写入：`runId`, `activeAgentId`, `workflowRef`, `updatedAt`
  - 初始化/重置（保证新 run 干净）：`stepsCompleted: []`, `variables: {}`, `decisionLog: []`, `artifacts: []`
  - `currentNodeId`：建议写入空字符串 `""`（RuntimeStore.getWorkflowState 会 fallback 到 graph.entryNodeId）

4) **Run 初始 phase**
- `runsIndex.json` 与 UI `RunPhase` 对齐：`'idle' | 'running' | 'waiting-user' | 'completed' | 'failed'`
- `createRun()` 返回的初始 `phase` 设为 `waiting-user`（UI 进入 Runs 工作区等待用户输入/启动 pump）。

## Implementation Plan

### Tasks

- [ ] 新增 `crewagent-runtime/electron/services/runManager.ts`
  - [ ] 定义 `RunConfig`/`RunMetadata`/错误模型
  - [ ] `createRun(config)`：创建 run folder、初始化 `@state/workflow.md`、写 `runsIndex.json`
  - [ ] `listRuns(projectRoot)`：读取并返回该 project 的 runs 列表
  - [ ] `getRunStatePath(projectRoot, runId)`：返回 run state 目录（用于 ToolHost mount）

- [ ] Electron Main IPC
  - [ ] `ipcMain.handle('runs:create', ...)`：调用 RunManager.createRun
  - [ ] `ipcMain.handle('runs:list', ...)`：调用 RunManager.listRuns

- [ ] Preload API
  - [ ] 在 `crewagent-runtime/electron/preload.ts` 暴露 `createRun(payload)` / `listRuns(payload)`

- [ ] UI：Start Workflow / Start Agent 改为调用 createRun
  - [ ] `crewagent-runtime/src/pages/PackagePage/PackagePage.tsx`：移除 Renderer 自生成 `runId` 的逻辑
  - [ ] 使用 IPC 返回的 `RunMetadata` 填充 `appStore.setActiveRun(...)`

- [ ] Tests（vitest）
  - [ ] 新增 `crewagent-runtime/electron/services/runManager.test.ts`
  - [ ] 覆盖：多 workflow 解析、路径存在性校验、workflow.md frontmatter 注入、runsIndex 原子写、logs 文件创建

### Acceptance Criteria (Checklist)

- [ ] 点击 Start（或调用 `RunManager.createRun`）会在 `<RuntimeStoreRoot>/projects/<projectId>/runs/<runId>/` 创建 run。
- [ ] `@state/workflow.md` 存在且来自选定 workflow 的 `workflow.md`，并注入：`runId/activeAgentId/workflowRef/updatedAt`，且 `stepsCompleted/variables/decisionLog/artifacts` 为新 run 的初始值。
- [ ] `@state/logs/execution.jsonl` 存在（空文件即可）。
- [ ] `runsIndex.json` 被创建/追加，并包含 `{ runId, projectId, packageId, workflowRef, activeAgentId, phase, createdAt, lastUpdatedAt }`。
- [ ] 多 workflow 校验遵循 Story 4.8：存在 `workflows[]` 时必须显式选择；无 `workflows[]` 时允许 workflowRef 为空并回退 entry。
- [ ] `createRun` 对无效输入返回结构化错误（可用于 UI toast/log）。

## Additional Context

### Dependencies

- `gray-matter`：解析/写回 `workflow.md` frontmatter
- Node `crypto.randomUUID()`：生成 `runId`

### Testing Strategy

- 单元测试优先（vitest）：
  - 使用临时目录模拟 `projectRoot` 与 “package 解压根”（可直接用 spec 下的 example 包内容作为 fixture 拷贝）。
  - mock `RuntimeStore`（只需要 `getPackage()` / `ensureRunStateDir()` / `getAgentDefinition()`）以避免依赖 Electron `app.getPath('userData')`。
- 手动验证（可选）：
  - 在 UI 的 Package 页面点击 Start Workflow，确认 Runs 页面能读取 progress（`runs:getProgress`）且 run state 文件存在。

### Notes

- `RuntimeStore.importPackage()` 当前对 `workflows[]` 场景会 run-time 校验 graph，但不会强制校验每个 workflow 的 `workflow.md` 是否存在；Story 4.8 的 createRun 需要补齐该校验。
- ToolHost（Story 4.6）已经会在写审计日志时创建 `state/logs/`，但 Story 4.8 仍应在 run 初始化时创建空日志文件，以满足“启动即具备审计落盘路径”的一致性。

## Traceability

- Story: `_bmad-output/implementation-artifacts/4-8-create-execution-project-folder.md`
- Design: `_bmad-output/architecture/runtime-architecture.md`

