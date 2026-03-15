# Tech-Spec: Story 12.6 Knowledge Management UI & Index Rebuild

**Created:** 2026-03-14
**Status:** Ready for Development
**Source Story (if applicable):** _bmad-output/implementation-artifacts/12-6-kb-management-ui-and-index-rebuild.md

## Overview

### Problem Statement

Runtime 目前已经分散具备 personal KB rebuild/clear、project KB import/rebuild/export/import archive、orphan cleanup 与 archive migration 等能力，但这些能力的状态、维护入口和运维事件仍分布在不同页面与不同日志格式里。结果是维护动作虽然存在，但默认用户界面与内部治理留痕之间边界不清，难以回答三个关键问题：

1. 当前 project knowledge 是否健康，索引与存储是否处于可用状态。
2. rebuild、import、orphan cleanup、migration 等动作是否成功，以及失败发生在哪个对象上。
3. 哪些治理事件应该保留为内部诊断记录，而不应该直接暴露给普通终端用户。

### Solution

在不新增独立治理路由的前提下，扩展现有 SettingsPage 与 KnowledgePage：

- SettingsPage 继续承载 personal knowledge 维护入口，不新增 recent activity，也不新增 raw summary 卡。
- KnowledgePage 继续承载 project knowledge 管理入口，新增 status summary 与 `Rebuild Index`，不新增默认用户可见的 activity panel。
- 引入 `KnowledgeOpsService` 统一读取 personal/project/global 三类 ops 日志，并兼容 legacy project `ops-log.ndjson` 行格式。
- 扩展 personal/project status payload，让 renderer 可直接显示 index 与 storage 摘要。
- 保留 `kb:ops:list` / `KnowledgeOperationRecord` 作为 internal diagnostics contract，而不是默认终端用户功能。

### Scope (In/Out)

**In Scope**

- 扩展 `kb:personal:getStatus` 与 `kb:project:getStatus` payload
- 新增 `kb:ops:list` IPC 与统一的 `KnowledgeOperationRecord`
- personal/project rebuild 结果写入结构化 ops 事件
- KnowledgePage 新增 `Rebuild Index` 与 status cards
- SettingsPage 保持轻量 maintenance 入口
- 单元/集成测试覆盖 legacy 兼容、status 汇总与 internal ops metadata

**Out of Scope**

- 新建知识治理路由或独立运维页面
- 默认终端用户可见的 activity feed / jump UI
- 知识正文浏览器、全文检索器或报表系统
- 自动修复策略或后台自愈任务
- 历史 personal 内存事件的补录/伪造回填
- 多人协作审计、权限体系或云端运维

## Context for Development

### Codebase Patterns

- `RuntimeStore` 负责 runtime-store 目录布局、manifest/ops-log 初始化、renderer-safe payload 输出；`getProjectKnowledgeStatusPayload()` 已经承担 `originalPath` 去敏的边界。
- `ProjectKbService` 是 project knowledge 的主服务层，已集中实现 `getStatus()`、`rebuildIndex()`、`exportArchive()`、`importArchive()`，并通过 SQLite bridge 提供 `count_chunks` / `count_embeddings` 这类衍生统计。
- `PersonalKbService` 当前只暴露 `getStatus()`、`rebuildIndex()`、`clearAll()`，并把事件写入 runtime log，而不是独立 personal ops log。
- Electron IPC contract 里，project knowledge 走 `ipcOk` / `ipcErr` 风格并返回 payload；preload 与 `electron-env.d.ts` 负责把 contract 暴露给 renderer。
- renderer 使用 Zustand `appStore` 统一封装 IPC，`KnowledgePage` 直接从 store 消费 `projectKnowledgeStatus`，并依赖 `kb:project:progress` 刷新页面。
- `KnowledgePage` 已有 import/export/import-archive action row 与 inventory list；`SettingsPage` 已有 personal rebuild/clear 与 orphan project list。因此 12.6 应增量扩展，而不是复制一套新页面。

### Files to Reference

- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/services/personalKbService.ts`
- `crewagent-runtime/electron/services/projectKbService.ts`
- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/preload.ts`
- `crewagent-runtime/electron/electron-env.d.ts`
- `crewagent-runtime/src/stores/appStore.ts`
- `crewagent-runtime/src/pages/KnowledgePage/KnowledgePage.tsx`
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx`
- `crewagent-runtime/src/components/OrphanProjectList.tsx`
- `_bmad-output/implementation-artifacts/design-12-6-kb-management-ui-and-index-rebuild.md`
- `_bmad-output/implementation-artifacts/12-4-project-kb-orphan-cleanup-integration.md`
- `_bmad-output/implementation-artifacts/12-5-project-kb-export-import-migration.md`

### Technical Decisions

1. **不新增路由，只扩展现有页面**
   - Personal knowledge 维护仍留在 `SettingsPage`
   - Project knowledge 维护仍留在 `KnowledgePage`
   - 避免信息架构分裂与重复入口

2. **新增统一的运维事件读取层**
   - 新建 `knowledgeOpsService.ts`
   - 从三类日志读取并归一化：
     - `runtime-store/kb/personal/ops-log.ndjson`
     - `runtime-store/projects/<projectId>/knowledge/ops-log.ndjson`
     - `runtime-store/kb/ops-log.ndjson`
   - 对 legacy project source status 行做 fallback 映射

3. **扩展 status contract，而不是新建平行接口**
   - `PersonalKbStatus` 增加 `storage`
   - `ProjectKnowledgeStatusPayload` 增加 `index` 与 `storage`
   - 继续复用 `kb:personal:getStatus`、`kb:project:getStatus`、`kb:personal:rebuildIndex`、`kb:project:rebuildIndex`

4. **renderer 默认页面不消费 ops records，但内部 contract 保留**
   - 新增 `kb:ops:list`
   - 返回 `KnowledgeOperationRecord[]`
   - 过滤、limit、eventPrefix 在 main/service 侧完成

5. **jump metadata 允许保留，但不属于默认用户界面能力**
   - `jump.kind = project-source | project-maintenance | project-activity | personal-file`
   - metadata 继续存在于 internal contract，供未来支持工具或非默认诊断入口使用

6. **project rebuild 必须写结构化 started/completed/failed 事件**
   - default UI 不展示 activity feed，但 rebuild 失败仍必须进入持久化 ops 记录
   - 12.6 需要把 rebuild 变成 first-class governance event

7. **Settings 中不展示 personal 原始摘要卡**
   - personal status 扩展用于按钮状态、错误反馈与轻量维护提示
   - 不新增 `Files / Daily Files / Entries / Storage / Index Status` 直出卡片，避免把内部结构泄露为 UI 负担

## Implementation Plan

### Tasks

- [ ] Task 1: 扩展 runtime-store 数据模型与状态汇总
  - [ ] 为 personal KB 增加 storage 统计 helper（`totalBytes`、`coreFileCount`、`dailyFileCount`）
  - [ ] 为 project KB 增加 storage/index 汇总 helper（目录大小、chunkCount、embeddingCount）
  - [ ] 保持 project status payload 对 renderer 不暴露 `originalPath`

- [ ] Task 2: 增加统一 knowledge ops 读取能力
  - [ ] 新建 `crewagent-runtime/electron/services/knowledgeOpsService.ts`
  - [ ] 定义 `KnowledgeOperationRecord`、scope/level/jump 模型
  - [ ] 兼容 legacy project source status 行到 `project.source.status`
  - [ ] 增加 `knowledgeOpsService.test.ts`

- [ ] Task 3: 把关键维护动作写成结构化 ops 事件
  - [ ] `PersonalKbService.rebuildIndex()` 写 `personal.rebuild.started/completed/failed`
  - [ ] `PersonalKbService.clearAll()` 与 commit 相关路径写 personal scope 事件
  - [ ] `ProjectKbService.rebuildIndex()` 写 `project.rebuild.started/completed/failed`
  - [ ] 复用 12.4 / 12.5 已写入的 orphan / archive 事件

- [ ] Task 4: 扩展 IPC / preload / renderer type contract
  - [ ] `main.ts` 增加 `kb:ops:list`
  - [ ] `preload.ts` 暴露 `kbOpsList`
  - [ ] `electron-env.d.ts` 补充 ops list 与扩展后的 status 类型
  - [ ] `appStore.ts` 暴露 project rebuild public action 与 ops 查询 action

- [ ] Task 5: 实现 renderer UI
  - [ ] `SettingsPage.tsx` 保持 personal maintenance 入口，不展示 raw summary/activity
  - [ ] `KnowledgePage.tsx` 增加 status cards 与 `Rebuild Index`
  - [ ] 默认终端用户页面不消费 `kb:ops:list`

- [ ] Task 6: 回归验证
  - [ ] 验证 personal/project status 与实际目录/SQLite 统计一致
  - [ ] 验证 rebuild 成功/失败都反映到 status + persisted ops records
  - [ ] 验证 orphan / migration / archive / import / rebuild 事件写入完整
  - [ ] 验证默认 Settings / Knowledge 页面不暴露 activity/jump UI

### Acceptance Criteria

- [ ] AC 1: 打开 Runtime settings/project knowledge 页面时，可以看到 project knowledge 的状态与存储摘要；personal knowledge 仍使用现有维护入口且不暴露 raw internal counters
- [ ] AC 2: 用户可在现有页面触发 personal/project rebuild，系统从 local truth source 重建索引并回传结果，同时写入治理留痕
- [ ] AC 3: import、commit、orphan cleanup、migration、rebuild 等关键事件被保留为内部治理记录；默认终端用户 UI 不要求查看或跳转这些记录

## Additional Context

### Dependencies

- Node `fs` / `path`
- 现有 `ProjectKbService` SQLite bridge（`count_chunks` / `count_embeddings`）
- 现有 `kb:project:progress` 广播机制
- `zustand` store 与当前 renderer 刷新模式
- 12.4 orphan cleanup ops 事件
- 12.5 archive export/import ops 事件

### Testing Strategy

- 单元测试：
  - `KnowledgeOpsService` 日志归一化、legacy fallback、eventPrefix/limit 过滤
  - personal/project status storage/index 汇总 helper
- 集成测试：
  - `kb:ops:list` 读取 personal/project/global 多源日志
  - rebuild/import/archive/orphan 事件进入统一 internal ops 流
  - project status payload 中 `chunkCount` / `embeddingCount` 与 SQLite 统计一致
- 手动验证：
  - Settings 中 personal rebuild/clear 后只出现 maintenance 反馈，不出现 activity feed / raw summary cards
  - KnowledgePage 中 rebuild 成功/失败路径反馈正确，状态卡同步刷新
  - 默认终端用户视图不显示 jump 按钮或 recent activity

### Notes

- 12.6 的核心不是新增知识能力，而是把 12.1 / 12.3 / 12.4 / 12.5 已存在的动作收口成可维护、可诊断、但不默认暴露给终端用户的治理留痕。
- `KnowledgeOpsService` 应优先做“薄归一化层”，不要把默认用户 UI 拼装逻辑塞回 `main.ts` 或 renderer。
- project status 的 storage/index 汇总尽量复用现有 manifest、目录扫描与 SQLite bridge，不要引入第二套独立知识扫描器。

## Traceability

- Story: _bmad-output/implementation-artifacts/12-6-kb-management-ui-and-index-rebuild.md
- Design: _bmad-output/implementation-artifacts/design-12-6-kb-management-ui-and-index-rebuild.md
