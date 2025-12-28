# Story 3.14: Generate v1.1 `bmad.json` Manifest

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **Creator**,  
I want the Builder to generate `bmad.json` following **Package Spec v1.1**,  
so that the exported `.bmad` package has a clear entrypoint and metadata.

## Acceptance Criteria

1. **Given** a ProjectBuilder project exists and contains **multiple workflows**  
   **When** the Builder generates `bmad.json`  
   **Then** it includes `schemaVersion/name/version/createdAt/entry/workflows`（v1.1）  
   **And** `entry.workflow/entry.graph/entry.agents` reference the generated files in the ZIP（以 package root 为基准路径）  
   **And** `workflows[]` lists all workflows in the project, each with `id/workflow/graph`（可选带 `displayName/description/tags`）  
   **And** `entry` points to the default workflow（若无 default，则按确定性规则选一个）

## Design

### Summary

- Tech Spec: `_bmad-output/tech-spec.md`
- 目标：从 ProjectBuilder 导出 **整包（package）**；不提供 “workflow 单独导出”。
- 生成 v1.1 `bmad.json`（满足 `bmad.schema.json`），并采用多 workflow 推荐目录结构：`workflows/<workflowId>/...`
- 产物为“派生文件”：用于预览与后续导出（Story 3.16），不写入 DB

### UX / UI

- 位置：ProjectBuilder（`/builder/[projectId]`）增加只读的 `bmad.json (preview)` 区块（可折叠）。
- 内容：
  - Pretty JSON（`JSON.stringify(manifest, null, 2)`）
  - Copy 按钮（复制 JSON）
  - Warnings（如“未设置默认 workflow，已选择 … 作为入口”）
  - Errors（如 workflows 为空/重复/不合法 id）会阻断预览生成（显示红色提示）
- 说明：下载 `.bmad` 的按钮与流程在 Story 3.16 处理；3.14 仅做 manifest 生成与预览。

### API / Contracts

- API：不新增后端接口；复用现有 ProjectBuilder 数据加载：
  - `GET /packages/{projectId}`（取 project name 等）
  - `GET /packages/{projectId}/workflows`（取 workflows 列表与 `isDefault`）
- Contract（必须满足，`additionalProperties: false`）：
  - Schema：`crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/bmad.schema.json`
  - 例子：`crewagent-runtime/spec/bmad-package-spec/v1.1/examples/multi-workflows/bmad.json`

### Data / Storage

- `bmad.json` 为派生文件（不落库），建议实现纯函数生成器：
  - `buildBmadManifestV11({ projectName, workflows, createdAt? }) -> { manifest, warnings, errors }`
  - `createdAt`：导出时（Story 3.16）传入 `new Date().toISOString()`；预览可使用 memoized 时间戳，避免每次渲染变动。
- 字段映射（MVP）：
  - `schemaVersion: "1.1"`
  - `name`: `project.name`
  - `version: "0.1.0"`（后续再做可配置）
  - `createdAt`: ISO string
  - `entry`：
    - `workflow: "workflows/<workflowId>/workflow.md"`
    - `graph: "workflows/<workflowId>/workflow.graph.json"`
    - `agents: "agents.json"`
    - `assetsDir: "assets/"`
  - `workflows[]`：对每个 workflow 生成：
    - `id`: `String(workflow.id)`（MVP；满足 schema `pattern` 且稳定）
    - `displayName`: `workflow.name`
    - `workflow/graph`: 同上路径规则
    - `tags: []`（可选）
- Default workflow 选择（写入 `entry`）：
  - 若存在 `isDefault=true`：选 `workflow.id` 最小的那一个（确定性）；若多 default，写 warning。
  - 若不存在 default：选 `workflow.id` 最小的那一个并写 warning。

### Errors / Edge Cases

- workflows 为空：错误（无法生成 `bmad.json`）。
- workflowId 重复：错误（阻断生成）。
- workflowId 不符合 schema pattern：错误（阻断生成；MVP 使用数字 ID 可规避）。
- projectName 为空：错误或回退为 `"Untitled"`（需在实现中二选一并保持一致）。
- 多个 default：不阻断；选择确定性的 entry，并提示 warning。

### Test Plan

- Unit（纯函数）：覆盖 default 选择、路径拼接、warnings/errors、createdAt 注入。
- Integration（手动）：ProjectBuilder 预览 `bmad.json`，对照 `bmad.schema.json` 必填字段与路径。
- Regression：在 Story 3.16 的整包导出中，确保 `bmad.json.entry.*` 与 `workflows[]` 路径与 ZIP 内真实文件一致。

## Tasks / Subtasks

- [x] 1) 实现 `bmad.json` 生成函数（与导出 pipeline 解耦；输入：project + workflows 列表 + default 选择）
- [x] 2) UI：在 ProjectBuilder 的导出预览中展示 `bmad.json (preview)`（便于验收）
- [x] 3) 为 Story 3.16 提供可复用的格式化输出（用于导出时写入 ZIP 根目录 `bmad.json`）

## References

- `_bmad-output/epics.md`（Epic 3 / Story 3.14）
- Tech Spec: `_bmad-output/tech-spec.md`
- `crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/bmad.schema.json`
- `crewagent-runtime/spec/bmad-package-spec/v1.1/examples/multi-workflows/bmad.json`

## Dev Agent Record

### Agent Model Used

GPT-5.2 (Codex CLI)

### Debug Log References

- Frontend:
  - `npm -C crewagent-builder-frontend run lint`
  - `npm -C crewagent-builder-frontend run build`

### Completion Notes List

- 新增 v1.1 `bmad.json` 生成器（纯函数）：默认 workflow 选择（含 warning）、多 workflow 索引与路径拼接。
- ProjectBuilder 增加 `bmad.json (preview)`（含 Copy、warnings/errors），用于后续整包导出（Story 3.16）。

### File List

- `_bmad-output/implementation-artifacts/3-14-generate-v1-1-bmad-json-manifest.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `crewagent-builder-frontend/src/app/builder/[projectId]/page.tsx`
- `crewagent-builder-frontend/src/lib/bmad-manifest-v11.ts`

### Change Log

- Builder 生成并预览 v1.1 `bmad.json`（multi-workflow 结构 + entry/workflows 索引）；提供格式化函数供后续 ZIP 导出写入根目录。
