# Tech Spec: Story 4.9 - Save All Artifacts to ProjectRoot (Project-First)

**Created:** 2026-01-08  
**Status:** Ready for Development  
**Source Story:** `_bmad-output/implementation-artifacts/4-9-save-all-artifacts-to-project-folder.md`

## Overview

### Problem Statement

Runtime 的目标是 **Project-First**：所有用户可见产物默认写入 ProjectRoot（通常是 `ProjectRoot/artifacts/`），并在 `@state/workflow.md` 的 `artifacts[]` 中记录产物路径（Project 内相对路径）。  

当前实现中，`fs.write` 负责把文件写入 `@project/...`，并把工具执行写入 `@state/logs/execution.jsonl`；但对于 “产物清单（artifacts[]）” 的维护缺少强制一致性，容易出现：

- 文件已写入 `@project/artifacts/...`，但 `@state/workflow.md.artifacts[]` 未同步（UI 无法可靠展示产物列表）。
- 依赖 `artifacts[]` 的恢复/上下文注入不稳定（容易漏上下文）。

### Solution

在 Electron Main 的 `FileSystemToolHost.fsWrite` 成功写入文件后，若目标位于 `@project/artifacts/...` 且是新文件创建：

1) 将路径转换为 Project 内相对路径（例如 `artifacts/foo.md`）  
2) 使用 `@state/workflow.md` 的 `updateFrontmatter` 逻辑，把该路径追加到 `artifacts[]`（去重）  
3) 保证“写文件 + 记录 artifacts”在语义上要么都成功，要么整体失败（避免半成功状态）

## Scope (In / Out)

**In scope**
- `fs.write` 创建 `@project/artifacts/...` 新文件后，自动追加记录到 `@state/workflow.md.artifacts[]`。
- 记录值为 Project 内相对路径（strip `@project/`）。
- 单元测试覆盖核心行为与失败模式。

**Out of scope**
- 改动 LLM 协议或新增工具（继续复用 `fs.write` / `fs.apply_patch`）。
- UI 增加独立的 “Artifacts 面板”（如需增强，走后续 UI story）。

## Implementation Plan

### Code Touch Points

- `crewagent-runtime/electron/services/fileSystemToolHost.ts`
  - 在 `fsWrite()` 成功写入后执行 artifact 记录逻辑。
  - 复用 `fsApplyPatch()` 的 `updateFrontmatter` 分支来更新 `@state/workflow.md`（保留 schema + transition 校验）。
- `crewagent-runtime/electron/services/fileSystemToolHost.test.ts`
  - 新增测试用例覆盖：append、dedupe、非 artifacts 目录不记录、失败回滚语义。

### Acceptance Checklist

- [ ] `fs.write` 创建 `@project/artifacts/...` 新文件后，`@state/workflow.md.artifacts[]` 追加对应的 project-relative 路径。
- [ ] 重复写同一路径不会造成 `artifacts[]` 重复。
- [ ] 写入 `@project` 但不在 `@project/artifacts/` 下时，不写入 `artifacts[]`。
- [ ] 执行日志（`@state/logs/execution.jsonl`）中能看到 `fs.write` 事件，包含写入路径（用于 UI 展示）。

## Notes / Traceability

- Story: `_bmad-output/implementation-artifacts/4-9-save-all-artifacts-to-project-folder.md`
- Architecture: `_bmad-output/architecture/runtime-architecture.md`（Project-First + artifacts 约定）
- Protocol: `_bmad-output/tech-spec/llm-conversation-protocol-openai.md`

