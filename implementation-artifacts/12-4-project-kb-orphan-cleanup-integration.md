# Story 12.4: Project KB Orphan Cleanup Integration

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **Consumer**,  
I want orphan project cleanup to include project knowledge residuals,  
So that Runtime private storage remains consistent after project folders are removed.

## Acceptance Criteria

### AC-1: 孤儿检测展示知识残留语义

**Given** a project root no longer exists  
**When** orphan detection runs  
**Then** orphan entries include project metadata plus runtime residual stats (conversation count and total size)  
**And** UI explicitly indicates that knowledge residuals are included in cleanup scope

### AC-2: 删除覆盖聊天与知识

**Given** I execute orphan delete  
**When** deletion is confirmed  
**Then** Runtime removes `runtime-store/projects/<projectId>/` completely  
**And** residual conversations and knowledge files are both removed  
**And** UI refreshes orphan list and shows deletion outcome

### AC-3: 重绑定后知识可恢复

**Given** I execute orphan rebind  
**When** binding succeeds  
**Then** existing conversations and project knowledge remain available under the re-bound project  
**And** UI reflects the recovered project immediately

## Technical Notes

- Delivery pattern: vertical full-stack in one story (Main/IPC/Renderer/Test together).
- Reuse existing orphan chain (`detect/rebind/delete/ignore`) and extend semantics for knowledge coverage.
- Delete operation should remain guarded by "project root missing" precondition.
- Design: `_bmad-output/implementation-artifacts/design-12-4-project-kb-orphan-cleanup-integration.md`

## Design

> Required before development. Keep this story in `ready-for-design` until the linked design is accepted and the story-level guardrails below are treated as implementation constraints.

### UX / UI

- 在 `Settings -> Project Data` 的 orphan 列表中，现有条目需要增加知识残留语义，而不是只显示聊天数和总大小。
- 每个 orphan item 需要显式展示 `Knowledge: <files> files` 与 `Cleanup scope: chats + knowledge`，避免用户误以为删除只清聊天。
- 删除成功后要有面向用户的 outcome summary，例如 “Removed 6 chats and 12 knowledge files”。
- 重绑定成功后要有恢复结果提示，例如 “Recovered 6 chats and 12 knowledge files”。

### API / Contracts

- `projects:getOrphans` 返回的 `OrphanProject` payload 需要扩展 `cleanupScope` 与 `knowledge` 摘要。
- `projects:deleteOrphan` 返回值需要从布尔成功扩展为带 summary 的 result。
- `projects:rebindOrphan` 返回值需要扩展 recovery summary，供 UI 直接展示恢复结果。
- 现有 IPC channel 名保持不变，避免前端入口全面重写。

### Data / Storage

- orphan detection 继续以 `projectId -> runtime-store/projects/<projectId>/` 为唯一 runtime root 映射。
- Project knowledge residual 范围包括 `knowledge/source/`、`knowledge/extracted/`、`knowledge/index.sqlite`、`knowledge/manifest.json`、`knowledge/ops-log.ndjson`。
- 删除语义保持为删除整棵 runtime root，不做 conversations / knowledge 分开清理。
- orphan delete / rebind 要写入全局 KB ops 事件，供 Story 12.6 的活动记录消费。

### Errors / Edge Cases

- 若原 `projectRoot` 又重新存在，`deleteOrphanData` 必须继续拒绝执行。
- 若 knowledge manifest 缺失，但 knowledge 目录仍存在，UI 仍需显示 knowledge included，只是摘要计数回落到 0。
- 若重绑定目标目录已有不同 `projectId`，继续沿用现有 force / conflict 逻辑，不能绕过。
- rebind 只恢复路径绑定，不迁移 runtime 数据；任何试图复制旧 runtime root 的实现都偏离当前契约。

### Test Plan

- orphan detection 返回 knowledge summary 与 cleanup scope。
- orphan delete 后 `runtime-store/projects/<projectId>/` 整棵树消失，聊天与知识同时消失。
- orphan rebind 后既有 conversations 与 `KnowledgePage` 状态都能继续读取。
- orphan delete / rebind 事件进入全局 KB ops log。

## Tasks / Subtasks

- [x] Task 1: Runtime Main + Store（后端）实现（AC: 1,2,3）
  - [x] 1.1 在 orphan stats 中补充 knowledge scope 标识/统计
  - [x] 1.2 确认 `deleteOrphanData` 删除覆盖完整 project runtime root
  - [x] 1.3 rebind 后验证 knowledge 目录可继续访问

- [x] Task 2: IPC Contract（前后端接口）实现（AC: 1,2,3）
  - [x] 2.1 扩展 orphan payload（cleanup scope/knowledge included）
  - [x] 2.2 删除与重绑返回结果增加面向 UI 的状态字段

- [x] Task 3: Renderer UI（前端）实现（AC: 1,2,3）
  - [x] 3.1 orphan 列表增加“包含知识残留”文案与提示
  - [x] 3.2 删除后即时刷新并展示操作结果
  - [x] 3.3 重绑成功后显示恢复状态

- [x] Task 4: Integration & E2E 验证（AC: 1,2,3）
  - [x] 4.1 人工删除 projectRoot 后 orphan 检测覆盖知识残留
  - [x] 4.2 orphan delete 后 conversations + knowledge 同时清除
  - [x] 4.3 rebind 后历史会话与知识检索均可用

- [x] Task 5: 回归与文档
  - [x] 5.1 不破坏现有 8.4 交互与行为
  - [x] 5.2 更新 story file list 与测试说明

## Dev Notes

- 当前最佳复用点是 Story 8.4 的 orphan 主链路与 Story 12.3 的 project knowledge manifest/status 结构；本 story 不应该引入第二套 orphan 管理模型。
- `RuntimeStore.detectOrphanProjects()` 已经基于 `projectId` 统计 runtime root 大小与 conversations；知识摘要应在这个阶段顺便补齐，而不是在 Renderer 端推导。
- `deleteOrphanData()` 当前已删除整棵 runtime root；开发时应保留这一“单点删除”语义，只扩展 summary 和可观测性。
- `rebindProject()` 必须保持“写新 `.crewagent.json` -> ensure runtime dirs -> ensure knowledge initialized”的顺序，避免误创建新 projectId 对应的数据根。

### Project Structure Notes

- 主要后端落点：
  - `crewagent-runtime/electron/stores/runtimeStore.ts`
  - `crewagent-runtime/electron/main.ts`
  - `crewagent-runtime/electron/preload.ts`
  - `crewagent-runtime/electron/electron-env.d.ts`
- 主要前端落点：
  - `crewagent-runtime/src/components/OrphanProjectList.tsx`
  - `crewagent-runtime/src/pages/SettingsPage/SettingsPage.css`
- 主要测试落点：
  - `crewagent-runtime/electron/stores/runtimeStore.test.ts`

### References

- `_bmad-output/epics.md` - Epic 12 goal and Story 12.4 acceptance criteria
- `_bmad-output/implementation-artifacts/8-4-project-data-management.md` - orphan detect/rebind/delete 既有契约
- `_bmad-output/implementation-artifacts/12-3-project-kb-import-extract-index-search.md` - project knowledge runtime-private layout 与 renderer-safe status 边界
- `_bmad-output/implementation-artifacts/design-12-4-project-kb-orphan-cleanup-integration.md` - 当前 design-story 设计稿
- `crewagent-runtime/electron/stores/runtimeStore.ts` - orphan/project runtime root 当前实现
- `crewagent-runtime/src/components/OrphanProjectList.tsx` - 当前 orphan UI 入口

## Dev Agent Record

### Agent Model Used

GPT-5（Codex CLI）

### Debug Log References

- `cd crewagent-runtime && npx vitest run electron/stores/runtimeStore.test.ts src/components/OrphanProjectList.test.ts`
- `cd crewagent-runtime && ./node_modules/.bin/tsc --noEmit --pretty false`

### Completion Notes List

- `RuntimeStore` 现已为 orphan payload 增加 `cleanupScope` 和 `knowledge` 摘要，并在 delete/rebind 成功时返回面向 UI 的 outcome summary。
- orphan delete / rebind 成功后会写入 `runtime-store/kb/ops-log.ndjson`，供 12.6 的统一 knowledge activity 读取。
- `OrphanProjectList` 现已显式展示 `Knowledge: ...` 与 `Cleanup scope: chats + knowledge`，删除确认文案也明确包含 project knowledge。
- 新增回归覆盖：orphan detection 的知识摘要、delete summary + 全局事件、rebind recovery summary，以及 orphan UI 文案 helper。

## File List

- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/stores/runtimeStore.test.ts`
- `crewagent-runtime/electron/electron-env.d.ts`
- `crewagent-runtime/src/components/OrphanProjectList.tsx`
- `crewagent-runtime/src/components/OrphanProjectList.test.ts` (new)
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.css`

## Change Log

- 2026-03-13: 完成 12-4 vertical slice，实现 orphan knowledge summary、delete/rebind outcome summary、全局 KB ops 事件，以及 orphan UI 的知识清理透明提示；状态更新为 `review`。
