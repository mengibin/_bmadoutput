# Story 12.5: Project KB Export / Import Migration

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **Consumer**,  
I want to export and import project knowledge archives,  
So that reusable knowledge can be migrated across projects or devices.

## Acceptance Criteria

### AC-1: 导出归档

**Given** I export project knowledge  
**When** export completes  
**Then** archive contains `source/`, `extracted/`, `notes/`, `manifest.json`, and integrity checks  
**And** UI provides export path/result feedback

### AC-2: 导入恢复与索引重建

**Given** I import a knowledge archive into target project  
**When** archive passes validation  
**Then** data is restored into target knowledge directory  
**And** target index is rebuilt  
**And** UI shows import summary (imported files, skipped files, rebuild result)

### AC-3: 失败回滚

**Given** import fails at any stage  
**When** rollback executes  
**Then** target project knowledge returns to pre-import state  
**And** failure is logged with reason  
**And** UI clearly indicates rollback completed

### AC-4: 复制语义

**Given** import succeeds  
**When** I edit imported content later  
**Then** source and target projects evolve independently (copy semantics)

## Technical Notes

- Delivery pattern: vertical full-stack in one story (Main/IPC/Renderer/Test together).
- Use staged directory + atomic swap for import safety.
- Integrity checks should include archive-level and file-level hashes.
- Design: `_bmad-output/implementation-artifacts/design-12-5-project-kb-export-import-migration.md`

## Design

> Required before development. Keep this story in `ready-for-design` until the linked design is accepted and the archive contract below is treated as implementation input.

### UX / UI

- 项目级入口落在现有 `KnowledgePage` action row，不新建设置页入口。
- 在现有 `Import Files / Import Folder` 旁新增 `Export Archive` 与 `Import Archive`。
- 导出成功后需要显示 archive path / counts / checksum 结果。
- 导入成功后需要显示 `imported / skipped / rebuilt` 摘要；失败但回滚完成时要明确告诉用户目标项目已恢复。

### API / Contracts

- 新增 project KB archive 导出/导入 IPC，但现有 project status/rebuild 通道保持不变。
- 导出结果需要返回 archive 路径、SHA-256 与 summary。
- 导入结果需要返回 imported/skipped/failed/rebuiltSources/rollbackCompleted。
- archive manifest 必须剔除 `originalPath`，保持 renderer-safe 和迁移安全边界一致。

### Data / Storage

- archive ZIP 必须包含 `manifest.json`、`checksums.json`、`source/`、`extracted/`、`notes/`。
- `index.sqlite` 不进入 archive；导入后统一按目标环境 rebuild。
- 导入使用 working root + atomic swap，不在 live `knowledge/` 上边改边 rebuild。
- 导入后的 source record 必须 rebased 到新的 `sourceId`，保证 copy semantics，不与源项目共享逻辑身份。

### Errors / Edge Cases

- archive schema 版本不匹配、校验和错误、zip-slip 风险、缺文件，都必须在导入前拒绝。
- 即使 archive 中 extracted 文件完整，也不允许跳过目标项目 rebuild。
- 同内容重复导入按 hash 进入 `skipped`，不能只凭文件名判重。
- 若导入过程任一阶段失败，必须回滚到导入前的 target knowledge 状态。

### Test Plan

- export archive 结构和 checksums 正确。
- import 成功后 target 能 rebuild，并且不复用 source `index.sqlite`。
- 导入冲突文件按内容 hash 去重进入 `skipped`。
- 导入失败时 rollback 完成且 target 恢复到导入前状态。
- 导入后编辑 target knowledge 不影响 source project。

## Tasks / Subtasks

- [x] Task 1: Runtime Main + Store（后端）实现（AC: 1,2,3,4）
  - [x] 1.1 实现 knowledge archive 导出（含 checksum/metadata）
  - [x] 1.2 实现导入验证与 staged 写入
  - [x] 1.3 实现导入成功后的索引重建
  - [x] 1.4 实现失败回滚与错误日志
  - [x] 1.5 确保导入为副本语义（不共享可变引用）

- [x] Task 2: IPC Contract（前后端接口）实现（AC: 1,2,3）
  - [x] 2.1 暴露 export/import 命令与结果结构
  - [x] 2.2 返回导入 summary（成功/跳过/失败计数）
  - [x] 2.3 返回 rollback 执行状态

- [x] Task 3: Renderer UI（前端）实现（AC: 1,2,3）
  - [x] 3.1 提供导出入口（路径选择 + 结果提示）
  - [x] 3.2 提供导入入口（文件选择 + 结果摘要展示）
  - [x] 3.3 导入失败时显示回滚完成状态

- [x] Task 4: Integration & E2E 验证（AC: 1,2,3,4）
  - [x] 4.1 正常导出/导入链路验证
  - [x] 4.2 损坏归档导入触发回滚验证
  - [x] 4.3 导入后编辑不影响源项目验证

- [x] Task 5: 回归与文档
  - [x] 5.1 不影响已有 orphan/rebind 流程
  - [x] 5.2 更新迁移格式说明与操作手册

## Dev Notes

- 这个 story 的核心不是“复制文件”本身，而是把 project KB 的 truth-source 安全迁移到另一个项目，同时确保派生索引在目标环境中重建。
- `ProjectKbService` 已经具备 import/rebuild 低层能力；archive import 应优先抽出 workspace 级 helper 来复用，而不是平行维护第二套索引器。
- 当前 `runtimeStore.ts` 已有 `AdmZip` 和安全路径解析经验，导入验证应直接复用同类防 traversal 约束。
- `notes/` 虽然当前 UI 未使用，但在 archive schema 中要先稳定保留，避免后续格式反复变更。

### Project Structure Notes

- 主要后端落点：
  - `crewagent-runtime/electron/services/projectKbService.ts`
  - `crewagent-runtime/electron/stores/runtimeStore.ts`
  - `crewagent-runtime/electron/main.ts`
  - `crewagent-runtime/electron/preload.ts`
  - `crewagent-runtime/electron/electron-env.d.ts`
- 主要前端落点：
  - `crewagent-runtime/src/stores/appStore.ts`
  - `crewagent-runtime/src/pages/KnowledgePage/KnowledgePage.tsx`
  - `crewagent-runtime/src/pages/KnowledgePage/KnowledgePage.css`
- 主要测试落点：
  - `crewagent-runtime/electron/services/projectKbService.test.ts`

### References

- `_bmad-output/epics.md` - Epic 12 goal and Story 12.5 acceptance criteria
- `_bmad-output/implementation-artifacts/12-3-project-kb-import-extract-index-search.md` - 现有 project KB 目录结构、rebuild、renderer-safe payload 约束
- `_bmad-output/implementation-artifacts/12-4-project-kb-orphan-cleanup-integration.md` - orphan cleanup 不应被 archive 迁移破坏
- `_bmad-output/implementation-artifacts/design-12-5-project-kb-export-import-migration.md` - 当前 design-story 设计稿
- `crewagent-runtime/electron/services/projectKbService.ts` - 现有 import/rebuild/retrieve 实现
- `crewagent-runtime/src/pages/KnowledgePage/KnowledgePage.tsx` - 当前项目级知识运营入口

## Dev Agent Record

### Agent Model Used

GPT-5（Codex CLI）

### Debug Log References

- `cd crewagent-runtime && npx vitest run electron/services/projectKbService.test.ts`
- `cd crewagent-runtime && ./node_modules/.bin/tsc --noEmit --pretty false`

### Completion Notes List

- `ProjectKbService` 现已支持 project knowledge archive 导出/导入，归档结构固定为 `manifest.json`、`checksums.json`、`source/`、`extracted/`、`notes/`，并显式排除 `index.sqlite`；archive-level SHA-256 以 zip comment 方式随单个 `.zip` 一起携带和校验。
- archive import 现已使用 working root + atomic swap，并在失败时回滚到导入前的 live knowledge 状态；成功/失败/rollback 会写入全局 KB ops log。
- archive import 会按 source file 内容 hash 去重，并为新导入 source 重新分配 `sourceId` 与目标路径，保证 copy semantics。
- `KnowledgePage` 现已补齐 `Export Archive` 与 `Import Archive` 入口，并展示成功摘要以及失败后的 rollback 提示。
- 补充回归覆盖：archive 结构/checksum、内容 hash 去重 + 副本语义、损坏归档回滚恢复。

## File List

- `crewagent-runtime/electron/services/projectKbService.ts`
- `crewagent-runtime/electron/services/projectKbService.test.ts`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/preload.ts`
- `crewagent-runtime/electron/electron-env.d.ts`
- `crewagent-runtime/src/stores/appStore.ts`
- `crewagent-runtime/src/pages/KnowledgePage/KnowledgePage.tsx`
- `crewagent-runtime/src/pages/KnowledgePage/KnowledgePage.css`
- `crewagent-runtime/src/vite-env.d.ts`

## Change Log

- 2026-03-13: 完成 12-5 vertical slice，实现 project KB archive export/import、单文件 zip 内嵌 archive hash、staged import + rollback、content-hash dedupe、copy semantics、KnowledgePage archive 入口与结果提示；状态更新为 `review`。
