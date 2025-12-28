# Story 3.12: Generate v1.1 `workflow.md` + `steps/<nodeId>.md`

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **Creator**,  
I want the Builder to generate `workflow.md` and step files in **Package Spec v1.1** format,  
so that export artifacts can reference stable, spec-aligned files.

## Acceptance Criteria

1. **Given** I edit the workflow graph in Builder and save  
   **When** the Builder generates `workflow.md`  
   **Then** `workflow.md` uses v1.1 frontmatter（至少包含 `schemaVersion/workflowType/currentNodeId/stepsCompleted`）  
   **And** the Steps Index references `steps/<nodeId>.md`（文件名使用 graph `node.id`，不按拓扑顺序重编号）

2. **Given** the Builder generates step files  
   **When** step files are generated for each node  
   **Then** each file path is `steps/<nodeId>.md`  
   **And** its frontmatter is v1.1-aligned（至少包含 `schemaVersion/nodeId/type`，且 `nodeId == graph node.id`）

> 注：`workflow.graph.json`/`bmad.json`/`agents.json`/ZIP 导出/Schema 校验分别在 Story 3.13–3.17 完成。

## Design

### Summary

- 将 Builder 当前的“按拓扑排序生成 step-01.md/step-02.md”改为“按 nodeId 稳定生成 `steps/<nodeId>.md`”，避免连线变化导致文件名漂移。
- `workflow.md` frontmatter 改为 v1.1 要求字段，并在正文 Steps Index 中引用 `steps/<nodeId>.md`。
- step frontmatter 改为 v1.1 step-frontmatter 结构；正文仍可沿用 Builder 的指令/说明。

### Open Questions

- `workflowType` 默认值：固定（如 `workflow`）还是允许用户配置
- `currentNodeId` 初始化策略：空字符串 vs 自动选择 entry node（入度为 0 的节点）
- 旧格式迁移：是否在保存/导出时自动升级已存在的 `step-01.md` 文件名与 frontmatter

### Test Plan

- 保存后检查 stored 内容：
  - `workflow.md` frontmatter 含 `schemaVersion/workflowType/currentNodeId/stepsCompleted`
  - `stepFilesJson` 的 key 形如 `steps/<nodeId>.md`，且不会因为连线变化而重命名
  - step frontmatter `nodeId` 与 graph node id 一致

## Tasks / Subtasks

- [x] 1) 更新生成逻辑：`workflow.md` v1.1 frontmatter + Steps Index（引用 `steps/<nodeId>.md`）
- [x] 2) 更新生成逻辑：step 文件改为 `steps/<nodeId>.md`，frontmatter 对齐 v1.1
- [x] 3) 兼容旧数据：加载/保存时能处理旧的 `step-01.md`（必要时自动迁移）
- [x] 4) 冒烟：创建项目→新增节点→连线→保存→刷新→文件名与 frontmatter 稳定

## References

- `_bmad-output/epics.md`（Epic 3 / Story 3.12）
- `_bmad-output/tech-spec.md`（Package Spec v1.1）
- `crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/workflow-frontmatter.schema.json`
- `crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/step-frontmatter.schema.json`
- `crewagent-runtime/spec/bmad-package-spec/v1.1/templates/workflow.md`
- `crewagent-runtime/spec/bmad-package-spec/v1.1/templates/step.md`
