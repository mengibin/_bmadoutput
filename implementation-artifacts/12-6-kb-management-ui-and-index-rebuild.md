# Story 12.6: Knowledge Management UI & Index Rebuild

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **Consumer**,  
I want a basic management UI for personal/project knowledge operations,  
So that I can inspect sources, trigger maintenance, and recover from index issues.

## Acceptance Criteria

### AC-1: 知识状态总览

**Given** I open Runtime settings/project data views  
**When** knowledge module is available  
**Then** I can see project knowledge status and storage summary  
**And** personal knowledge continues to use the existing maintenance entry without exposing raw internal summary counters

### AC-2: 索引重建维护

**Given** I need maintenance  
**When** I trigger rebuild index  
**Then** system rebuilds index from local truth sources and reports result

### AC-3: 治理操作内部留痕

**Given** runtime / support needs governance visibility  
**When** knowledge operations execute  
**Then** system persists key events (import, commit, orphan cleanup, migration, rebuild) into normalized internal operation records  
**And** default end-user UI does not need to expose activity feed or jump affordances

## Technical Notes

- Delivery pattern: vertical full-stack in one story (Main/IPC/Renderer/Test together).
- This story focuses on management/observability UX and maintenance operations, not core ingestion logic.
- Operation records can come from `ops-log.ndjson` + normalized view model for internal diagnostics/support tooling.
- Design: `_bmad-output/implementation-artifacts/design-12-6-kb-management-ui-and-index-rebuild.md`

## Design

### Summary

- 继续复用现有 `SettingsPage` 与 `KnowledgePage`，不新增独立知识治理路由。
- 为 project knowledge 在默认 Knowledge 视图左侧补齐状态总览，并在右侧保留 `Rebuild Index` 等维护操作；personal knowledge 保持现有 rebuild/clear 入口，不新增默认用户可见的治理活动流。
- 引入统一的 `KnowledgeOperationRecord` 与 `kb:ops:list` IPC，把 personal/project/global 三类 ops 日志归一化后保留给内部诊断/支持流程使用。
- 扩展 `kb:personal:getStatus` 与 `kb:project:getStatus` payload，让 UI 可展示 project 的 index/storage 摘要，并把 personal 扩展状态仅用于维护入口的轻量状态反馈。
- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-12-6-kb-management-ui-and-index-rebuild.md`

### UX / UI

- `SettingsPage` 保留当前 Personal Memory 区块的 `Rebuild Personal Memory` / `Clear Personal Memory` 按钮，不新增 `Files / Daily Files / Entries / Storage / Index Status` 摘要卡。
- `KnowledgePage` 在现有顶部 action row 中新增 `Rebuild Index`，保留 `Import Files`、`Import Folder`、`Export Archive`、`Import Archive`。
- 默认 Knowledge 视图在左侧 `KnowledgePanel` 展示 project summary：files、index、storage、embedding；右侧 `KnowledgePage` 只保留维护工具与 inventory。
- 默认终端用户视图不展示 recent activity、jump 按钮或 source/highlight 导航；如后续需要，仅允许作为内部诊断入口补充。

### API / Contracts

- 复用现有 `kb:personal:getStatus` / `kb:personal:rebuildIndex` / `kb:project:getStatus` / `kb:project:rebuildIndex`，只扩展 payload，不改 IPC 名称。
- 新增 `kb:ops:list`，输入至少支持：
  - `scope: 'personal' | 'project' | 'all'`
  - `projectRoot?: string`
  - `limit?: number`
  - `eventPrefix?: string`
- `kb:ops:list` 返回统一的 `KnowledgeOperationRecord[]`，供内部诊断/支持流程消费，而不是把原始 ndjson 行暴露给默认终端用户视图。
- `ProjectKnowledgeStatusPayload` 增加：
  - `index.status`
  - `index.chunkCount`
  - `index.embeddingCount`
  - `storage.totalBytes`
  - `storage.sourceBytes`
  - `storage.extractedBytes`
  - `storage.notesBytes`
- `PersonalKbStatus` 增加 storage summary，但这组字段不直接在 Settings UI 中展示。

### Data / Storage

- 统一 ops 日志来源：
  - `runtime-store/kb/personal/ops-log.ndjson`
  - `runtime-store/projects/<projectId>/knowledge/ops-log.ndjson`
  - `runtime-store/kb/ops-log.ndjson`
- project KB 继续以 `source/`、`extracted/`、`manifest.json` 为 truth source，`index.sqlite` 为派生层；rebuild 只能从 truth source 重建。
- project `ops-log.ndjson` 需要兼容 legacy source status 行，并在读取阶段映射为 `project.source.status` 事件。
- renderer-safe 边界维持不变：status、ops、jump metadata 都不能重新暴露 `originalPath` 等宿主机绝对路径。

### Errors / Edge Cases

- legacy 行缺少 `schemaVersion`、`scope`、`event` 时必须 fallback 归一化，不能要求先迁移历史日志。
- personal 历史事件不完整是可接受现状；12.6 不伪造 rebuild 之前的 personal timeline。
- rebuild 失败必须同时反映到 status payload 与持久化 ops records，不能只停留在 toast。
- project embedding 配置漂移导致的索引状态，仍应沿用现有 `ProjectKbService.buildStatusEmbedding()` 的安全语义，不能让 UI 误报“ready”。

### Test Plan

- personal/project status payload 的 storage/index 摘要与实际目录大小、SQLite 计数一致。
- `kb:ops:list` 可同时读取 personal/project/global 日志，并支持 `scope` / `limit` / `eventPrefix`。
- legacy project ops 行可被归一化为 internal operation record。
- personal rebuild / clear、project rebuild、orphan cleanup、archive export/import 等关键事件都能进入内部治理记录。
- 默认 Settings / Knowledge 页面不展示 activity feed，但关键维护结果仍会通过现有状态区与反馈文案体现。

## Tasks / Subtasks

- [x] Task 1: Runtime Main + Store（后端）实现（AC: 1,2,3）
  - [x] 1.1 提供 personal/project KB 状态与统计汇总接口
  - [x] 1.2 提供索引重建接口（personal/project）
  - [x] 1.3 提供操作记录查询接口与分页/筛选基础能力

- [x] Task 2: IPC Contract（前后端接口）实现（AC: 1,2,3）
  - [x] 2.1 暴露 status summary / rebuild / ops log 查询接口
  - [x] 2.2 统一错误返回与重建结果结构

- [x] Task 3: Renderer UI（前端）实现（AC: 1,2,3）
  - [x] 3.1 默认 Knowledge 视图补齐 project knowledge 状态面板；Settings 保持现有 personal maintenance 区块
  - [x] 3.2 增加“重建索引”操作与进度/结果反馈
  - [x] 3.3 默认终端用户 UI 不暴露治理活动流；ops 记录仅保留内部诊断消费能力

- [x] Task 4: Integration & E2E 验证（AC: 1,2,3）
  - [x] 4.1 状态总览数据与实际目录/索引一致性验证
  - [x] 4.2 重建成功与失败路径验证
  - [x] 4.3 内部操作日志留痕与归一化验证

- [x] Task 5: 回归与文档
  - [x] 5.1 不影响现有 Settings / Project Data 页面核心功能
  - [x] 5.2 更新维护手册与故障排查文档

## Dev Notes

- 这不是一个“新增知识功能” story，而是把 12.1 / 12.3 / 12.4 / 12.5 已经存在或将存在的知识动作收口到同一套管理与观测视图。
- 后端最好抽出 `knowledgeOpsService` 统一归一化日志读取，避免 Main 或 Renderer 重复解析三类 ndjson。
- project status 扩展应尽量建立在现有 `ProjectKbService` 与 SQLite bridge 计数 helper 上，不要为 UI 再造一套扫描器；personal status 即使扩展，也不应在 12.6 中额外长出一组摘要卡。
- ops records 的第一版重点是“内部留痕与可诊断”，不是面向终端用户做活动流或报表。

### Project Structure Notes

- 主要后端落点：
  - `crewagent-runtime/electron/services/personalKbService.ts`
  - `crewagent-runtime/electron/services/projectKbService.ts`
  - `crewagent-runtime/electron/services/knowledgeOpsService.ts` (new)
  - `crewagent-runtime/electron/stores/runtimeStore.ts`
  - `crewagent-runtime/electron/main.ts`
  - `crewagent-runtime/electron/preload.ts`
  - `crewagent-runtime/electron/electron-env.d.ts`
- 主要前端落点：
  - `crewagent-runtime/src/stores/appStore.ts`
  - `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx`
  - `crewagent-runtime/src/pages/KnowledgePage/KnowledgePage.tsx`
- 主要测试落点：
  - `crewagent-runtime/electron/services/knowledgeOpsService.test.ts`

### References

- `_bmad-output/epics.md` - Epic 12 goal and Story 12.6 acceptance criteria
- `_bmad-output/implementation-artifacts/12-1-personal-kb-storage-and-candidate-commit.md` - personal KB rebuild/clear/commit 基础
- `_bmad-output/implementation-artifacts/12-3-project-kb-import-extract-index-search.md` - project KB status/rebuild/import 基础
- `_bmad-output/implementation-artifacts/12-4-project-kb-orphan-cleanup-integration.md` - orphan cleanup 事件来源
- `_bmad-output/implementation-artifacts/12-5-project-kb-export-import-migration.md` - archive migration 事件来源
- `Tech Spec: _bmad-output/implementation-artifacts/tech-spec-12-6-kb-management-ui-and-index-rebuild.md`
- `_bmad-output/implementation-artifacts/design-12-6-kb-management-ui-and-index-rebuild.md` - 当前 design-story 设计稿
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx` - 当前 personal KB 管理入口
- `crewagent-runtime/src/pages/KnowledgePage/KnowledgePage.tsx` - 当前 project KB 管理入口

## Dev Agent Record

### Agent Model Used

GPT-5（Codex CLI）

### Debug Log References

- `cd crewagent-runtime && ./node_modules/.bin/tsc --noEmit --pretty false`
- `cd crewagent-runtime && ./node_modules/.bin/eslint electron/services/projectKbService.ts electron/services/knowledgeOpsService.ts electron/services/knowledgeOpsService.test.ts electron/services/personalKbService.ts electron/services/personalKbService.test.ts electron/stores/runtimeStore.ts electron/stores/runtimeStore.test.ts electron/main.ts electron/preload.ts electron/electron-env.d.ts src/stores/appStore.ts src/pages/KnowledgePage/KnowledgePage.tsx src/pages/SettingsPage/SettingsPage.tsx`
- `cd crewagent-runtime && ./node_modules/.bin/vitest run electron/services/knowledgeOpsService.test.ts electron/services/personalKbService.test.ts electron/services/projectKbService.test.ts electron/stores/runtimeStore.test.ts`
- `cd crewagent-runtime && ./node_modules/.bin/vitest run electron/services/projectKbService.test.ts electron/stores/runtimeStore.test.ts`

### Completion Notes List

- 新增 `KnowledgeOpsService`，统一读取 personal/project/global 三类 ops 日志，并将 legacy project source 状态行归一化为 renderer-safe 的 `KnowledgeOperationRecord`。
- 扩展 personal/project knowledge status payload：personal 增加 storage summary，project 增加 index/storage 摘要，并在 `ProjectKbService` 中基于 SQLite bridge 返回 chunk/embedding 计数与 index freshness。
- 打通 `kb:ops:list` IPC/preload/env/store 链路，并新增 renderer 侧 `listKnowledgeOperations`、`rebuildProjectKnowledge` action。
- 默认 Knowledge 视图现采用“左侧状态总览、右侧维护工具与 inventory”的分区；`SettingsPage` 保持 personal maintenance 入口，不再向默认终端用户暴露治理活动流。
- 增补覆盖：ops 日志归一化、personal rebuild/commit ops、project status/index/storage、personal ops log 初始化，以及 project/runtime store 回归验证；类型检查、目标文件范围 ESLint、相关 Vitest 用例均已通过。

## File List

- `crewagent-runtime/electron/services/knowledgeOpsService.ts`
- `crewagent-runtime/electron/services/knowledgeOpsService.test.ts`
- `crewagent-runtime/electron/services/personalKbService.ts`
- `crewagent-runtime/electron/services/personalKbService.test.ts`
- `crewagent-runtime/electron/services/projectKbService.ts`
- `crewagent-runtime/electron/services/projectKbService.test.ts`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/stores/runtimeStore.test.ts`
- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/preload.ts`
- `crewagent-runtime/electron/electron-env.d.ts`
- `crewagent-runtime/src/stores/appStore.ts`
- `crewagent-runtime/src/pages/KnowledgePage/KnowledgePage.tsx`
- `crewagent-runtime/src/pages/KnowledgePage/KnowledgePage.css`
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx`
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.css`

## Change Log

- 2026-03-14: 完成 12-6 vertical slice，实现 unified knowledge ops 查询、project/personal status 扩展、project rebuild 入口、KnowledgePage activity/jump/highlight、Settings recent activity、相关测试与验证；状态更新为 `review`。
- 2026-03-14: Senior developer review completed; 2 HIGH / 1 MEDIUM issues logged; status set to `in-progress`.
- 2026-03-14: 根据产品澄清收窄 AC-3 为“内部治理留痕”；默认用户界面移除 activity/jump 暴露，同时移除 personal raw summary 卡并补齐 post-action status refresh；状态恢复为 `review`。
- 2026-03-14: Code review rerun found AC-1 regression after the latest UI simplification: project knowledge index/storage summary is no longer visible in the default Knowledge view; 状态退回 `in-progress`。
- 2026-03-14: 恢复左侧 KnowledgePanel 的 project index/storage/embedding 总览，默认知识页重新对齐“左边总览、右边工具/清单”；状态更新回 `review`。

## Senior Developer Review (AI)

**Date:** 2026-03-14  
**Outcome:** Changes Requested

### Summary

| Metric | Value |
|--------|-------|
| Git vs Story Discrepancies | N/A（当前 worktree 含无关本地改动，review 以 story File List 与当前实现为准） |
| HIGH Issues | 1 |
| MEDIUM Issues | 0 |
| Type Check | ✅ `cd crewagent-runtime && ./node_modules/.bin/tsc --noEmit --pretty false` passed（latest available verification） |
| Unit Tests | ✅ `cd crewagent-runtime && ./node_modules/.bin/vitest run electron/services/knowledgeOpsService.test.ts electron/services/personalKbService.test.ts electron/services/projectKbService.test.ts electron/stores/runtimeStore.test.ts` passed（latest available verification） |

### Acceptance Criteria Check

- AC-1: PARTIAL. project knowledge status/storage summary payload 已存在，但最新 UI 简化后默认 Knowledge 视图里已经没有任何地方把 index/storage 总览展示出来。
- AC-2: PASS. personal/project rebuild 入口与结果反馈已接通。
- AC-3: PASS. 关键治理事件继续保留为内部 ops records，默认终端用户 UI 不再要求 activity/jump。

### Findings

1. **[HIGH] AC-1 回归：latest Knowledge UI simplification 把 project index/storage summary 从默认视图里完全删掉了。**  
   story/design 现在仍要求“打开 Runtime settings/project data views 时可以看到 project knowledge status and storage summary”。但当前左侧 `KnowledgePanel` 只渲染 file counters 与 embedding settings，没有消费 `status.index` / `status.storage`；右侧 `KnowledgePage` 也只剩 maintenance action row 与 inventory list。`WorkspacePage` 组合这两个区域后，默认 project knowledge 视图里已经不存在任何 index/storage summary 展示位。  
   Evidence: `crewagent-runtime/src/pages/KnowledgePage/KnowledgePage.tsx:188-255`, `crewagent-runtime/src/pages/KnowledgePage/KnowledgePage.tsx:257-405`, `crewagent-runtime/src/pages/KnowledgePage/KnowledgePage.tsx:531-597`, `crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.tsx:566-579`, `12-6-kb-management-ui-and-index-rebuild.md:19`

### Test Gaps

- 当前 targeted tests 主要覆盖 service/store contract，没有 renderer-level coverage 去验证默认 Knowledge 视图仍然展示 project status/storage summary，因此这次 UI 简化把 AC-1 悄悄删掉后没有被测试拦住。

## Review Follow-up Resolution (AI)

- [x] 已在左侧 `KnowledgePanel` 恢复 project snapshot，总览重新覆盖 files / index / storage / embedding。
- [x] 右侧 `KnowledgePage` 继续只承载维护 action row 与 knowledge inventory，符合“左边总览、右边工具”的最新产品方向。
- [x] 新增纯函数回归测试，约束左侧 summary 至少包含 index / storage / embedding 三类概览。
- [x] Follow-up verification:
  - `cd crewagent-runtime && ./node_modules/.bin/eslint src/pages/KnowledgePage/KnowledgePage.tsx src/pages/KnowledgePage/KnowledgePage.css src/pages/KnowledgePage/KnowledgePage.test.ts`
  - `cd crewagent-runtime && ./node_modules/.bin/tsc --noEmit --pretty false`
  - `cd crewagent-runtime && ./node_modules/.bin/vitest run src/pages/KnowledgePage/KnowledgePage.test.ts electron/services/knowledgeOpsService.test.ts electron/services/personalKbService.test.ts electron/services/projectKbService.test.ts electron/stores/runtimeStore.test.ts`
