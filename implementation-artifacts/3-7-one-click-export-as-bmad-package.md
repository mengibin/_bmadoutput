# Story 3.7: One-Click Export as .bmad Package

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **Creator**,  
I want to export my workflow as a `.bmad` package with one click,  
so that I can distribute it to Consumers.

## Acceptance Criteria

1. **Given** my workflow project is complete  
   **When** I click "Export Package"  
   **Then** a `.bmad` ZIP file is generated containing `workflow.md`, step files, and `agents.json`  
   **And** the package is validated (basic JSON/schema checks) before download

> Follow-up: Full **Package Spec v1.1** export is implemented as **Story 3.8–3.17** (ProjectBuilder foundation → v1.1 generators → zip → schema validation + errors).

## Design

### Summary

- Editor Header 增加 `Export Package` 按钮。
- 点击导出：
  - 若存在未保存改动：先触发 `Save now`（并等待成功）。
  - 从已保存的 `workflowMd / agentsJson / stepFilesJson` 生成 ZIP：
    - `workflow.md`
    - `agents.json`
    - `step-xx.md`（来自 stepFilesJson）
  - 生成一个内存 “manifest” 并用 JSON Schema 校验（Ajv）：
    - `workflowMd: string`
    - `agents: AgentDefinition[]`
    - `stepFiles: Record<string, string>`
  - 校验通过后触发浏览器下载：`${projectName}.bmad`

### UX / UI

- 按钮：`Export Package`
  - Disabled 条件：未加载完成 / 正在保存 / 保存失败 / 校验失败
  - 失败：显示明确错误（schema 校验/zip 生成/下载失败）

### Test Plan

- 手工冒烟：
  - 编辑后直接点 Export → 自动先保存 → 下载 `.bmad`
  - 解压 zip：包含 `workflow.md`、`agents.json`、`step-01.md` 等
- 工程校验：
  - Frontend：`npm run lint`、`npm run build`

## Tasks / Subtasks

- [x] 1) 前端：实现导出 ZIP（AC: 1）
  - [x] 安装 `jszip`
  - [x] 组装文件并生成 blob 下载

- [x] 2) 前端：Schema 校验（AC: 1）
  - [x] 安装 `ajv`
  - [x] 定义 schema 并在导出前校验

- [x] 3) 校验
  - [x] `npm run lint`
  - [x] `npm run build`

## References

- `_bmad-output/epics.md`（Epic 3 / Story 3.7）
- `_bmad-output/prd.md`（FR-DEF-04/05）

## Dev Agent Record

### Agent Model Used

GPT-5.2 (Codex CLI)

### Debug Log References

- Frontend:
  - `npm run lint`
  - `npm run build`

### Completion Notes List

- Editor 增加 `Export Package`：自动保存后将 `workflow.md`、`agents.json`、step files 打包为 `.bmad`（ZIP）并下载。
- 导出前用 Ajv 做 JSON Schema 校验，并额外校验 `workflow.md` 引用的 step files 存在。

### File List

- `_bmad-output/implementation-artifacts/3-7-one-click-export-as-bmad-package.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `crewagent-builder-frontend/package-lock.json`
- `crewagent-builder-frontend/package.json`
- `crewagent-builder-frontend/src/app/editor/[workflowId]/page.tsx`
