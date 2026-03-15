# Design: Knowledge Management UI & Index Rebuild

**Story:** `12-6-kb-management-ui-and-index-rebuild.md`  
**设计原则:** 复用现有 Settings / Knowledge 页面、真相源重建、知识运维内部留痕、Renderer-safe payload、最小但完整的维护闭环

---

## 设计目标

1. 在现有 Runtime UI 中补齐 project knowledge 的状态总览，并在 personal knowledge 入口保持轻量维护体验，而不是再新建一套独立信息架构。
2. 为 personal KB 与 project KB 提供显式的 `Rebuild Index` 维护入口，并把结果反馈到 UI。
3. 把 import、commit、orphan cleanup、migration 等关键知识运维动作汇总为可查询的内部记录，供支持/诊断使用。
4. 普通终端用户默认不需要看到治理活动流或 jump 入口，避免把内部留痕直接暴露成产品 UI 负担。
5. 与 Story 12.4 / 12.5 对齐，让 orphan/migration 结果进入统一 ops 留痕，而不是额外长出一套默认用户可见页面。

## 非目标（本 Story 不做）

1. 不做人类全文检索器或知识正文阅读器。
2. 不做复杂权限体系、多人协作审计或云端运维面板。
3. 不做自动修复策略；本 Story 的维护动作以人工触发 rebuild 为主。
4. 不回填 12.6 之前已经丢失的内存级个人日志事件。

## 当前 Cross-Story 约束（2026-03-13）

1. Story 12.1 的 personal KB 真相源仍是 Markdown 文件与 `index.json`；12.6 只能在此基础上补管理与观测。
2. Story 12.3 的 project KB 真相源仍是 `source/`、`extracted/`、`manifest.json`，`index.sqlite` 只是派生层。
3. Story 12.4 已定义 orphan delete / rebind 必须写全局 KB ops 事件。
4. Story 12.5 已定义 archive export / import 也会写 KB ops 事件，并且 `notes/` 目录成为 archive 稳定结构的一部分。
5. 当前 project `ops-log.ndjson` 仍然存在 legacy 行格式（逐 source 状态行）；12.6 必须兼容读取，而不是要求先做日志迁移。

## Brownfield 约束与复用点

1. **SettingsPage 已有 Project Data + Personal Memory 入口**  
   当前 `SettingsPage.tsx` 已包含 `OrphanProjectList` 和 Personal Memory rebuild/clear 按钮，12.6 应扩展这一块，而不是另起新路由。

2. **KnowledgePage 已经承担项目级知识运营入口**  
   当前 `KnowledgePage.tsx` 已有 import actions、inventory list、embedding 配置；12.6 只需补 maintenance/status，不要重做页面。

3. **现有 KB 状态 IPC 已存在**  
   `kb:personal:getStatus`、`kb:personal:rebuildIndex`、`kb:project:getStatus`、`kb:project:rebuildIndex` 都已存在，12.6 优先扩展 payload，而不是改协议名字。

4. **现有 project progress 事件可复用**  
   `kb:project:progress` 已经能驱动 KnowledgePage refresh，rebuild / archive import 成功后仍可沿用这一刷新机制。

---

## 改动范围

| 文件 | 改动类型 | 说明 |
|------|----------|------|
| `crewagent-runtime/electron/stores/runtimeStore.ts` | MODIFY | 增加 personal/global KB ops log 路径、knowledge 存储摘要 helper |
| `crewagent-runtime/electron/services/personalKbService.ts` | MODIFY | personal rebuild/clear/commit 写入 personal ops log |
| `crewagent-runtime/electron/services/projectKbService.ts` | MODIFY | project import/rebuild/export/importArchive 写结构化 ops 事件；扩展 status summary |
| `crewagent-runtime/electron/services/knowledgeOpsService.ts` | NEW | 统一读取 personal/project/global ops log，兼容 legacy 行并归一化给内部诊断流程 |
| `crewagent-runtime/electron/services/knowledgeOpsService.test.ts` | NEW | ops 归一化、筛选与 internal metadata 测试 |
| `crewagent-runtime/electron/main.ts` | MODIFY | 新增 `kb:ops:list` IPC；扩展 status 返回结构 |
| `crewagent-runtime/electron/preload.ts` | MODIFY | 暴露 `kbOpsList`、扩展 status/rebuild 调用 |
| `crewagent-runtime/electron/electron-env.d.ts` | MODIFY | 补充 KB ops 与扩展 status 类型 |
| `crewagent-runtime/src/stores/appStore.ts` | MODIFY | 新增 project rebuild action，并保留内部 ops 查询能力 |
| `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx` | MODIFY | personal KB 维护操作；移除默认用户可见的 activity / raw summary |
| `crewagent-runtime/src/pages/SettingsPage/SettingsPage.css` | MODIFY | personal knowledge maintenance 轻量化样式 |
| `crewagent-runtime/src/pages/KnowledgePage/KnowledgePage.tsx` | MODIFY | project KB 状态卡、rebuild action；移除默认用户可见的 activity panel |
| `crewagent-runtime/src/pages/KnowledgePage/KnowledgePage.css` | MODIFY | management / status layout 样式 |

---

## 统一运维事件模型

### 1) 日志来源

```text
<userData>/runtime-store/kb/personal/ops-log.ndjson
<userData>/runtime-store/projects/<projectId>/knowledge/ops-log.ndjson
<userData>/runtime-store/kb/ops-log.ndjson
```

角色分工：
- `personal/ops-log.ndjson`: personal commit / clear / rebuild
- `projects/<projectId>/knowledge/ops-log.ndjson`: import / rebuild / export / importArchive / source status
- `runtime-store/kb/ops-log.ndjson`: orphan cleanup、跨项目 archive 迁移等全局事件

### 2) 归一化后的内部记录

```ts
type KnowledgeOperationScope = 'personal' | 'project' | 'global'
type KnowledgeOperationLevel = 'info' | 'success' | 'warn' | 'error'

interface KnowledgeOperationRecord {
  opId: string
  scope: KnowledgeOperationScope
  event: string
  level: KnowledgeOperationLevel
  createdAt: string
  projectId?: string
  title: string
  detail?: string
  summary?: Record<string, string | number | boolean | null>
  jump?: {
    kind: 'personal-file' | 'project-source' | 'project-maintenance' | 'project-activity'
    projectRoot?: string
    sourceId?: string
    targetFile?: string
  }
}
```

### 3) Legacy project `ops-log.ndjson` 兼容策略

当前 12.3 的日志行大致为：

```json
{
  "sourceId": "src-001",
  "fileName": "spec.pdf",
  "status": "ready",
  "processingStage": "completed",
  "updatedAt": "2026-03-13T09:00:00.000Z",
  "errorMessage": null
}
```

12.6 的 `KnowledgeOpsService` 必须：

1. 先识别新 envelope：存在 `schemaVersion + event + scope` 时直接使用。
2. 对 legacy 行做 fallback 归一化：
   - `event = project.source.status`
   - `title = <fileName>`
   - `detail = processingStage / errorMessage`
   - `jump.kind = project-source`

说明：
- 12.6 不要求重写历史 `ops-log.ndjson`。
- 12.6 之后新增的 project 事件全部写新 envelope，避免后续继续放大 legacy 格式。

---

## 扩展状态总览模型

### 1) Personal KB status

在当前 `PersonalKbStatus` 基础上增加：

```ts
type PersonalKbStatus = {
  initialized: boolean
  rootPath: string
  files: PersonalKbFileStatus[]
  index: {
    status: 'ready' | 'missing' | 'stale'
    lastIndexedAt?: string
    entryCount?: number
  }
  storage: {
    totalBytes: number
    coreFileCount: number
    dailyFileCount: number
  }
}
```

### 2) Project KB status

在当前 `ProjectKnowledgeStatusPayload` 基础上增加：

```ts
type ProjectKnowledgeStatusPayload = {
  initialized: boolean
  rootPath: string
  summary: {
    totalFiles: number
    readyFiles: number
    processingFiles: number
    failedFiles: number
  }
  embedding: {
    enabled: boolean
    provider?: string
    model?: string
    lastIndexedAt?: string
    lastError?: string | null
  }
  index: {
    status: 'ready' | 'missing' | 'stale'
    chunkCount: number
    embeddingCount: number
  }
  storage: {
    totalBytes: number
    sourceBytes: number
    extractedBytes: number
    notesBytes: number
  }
  sources: ProjectKnowledgeSourceSummary[]
}
```

说明：
- `index.chunkCount` / `embeddingCount` 可直接复用现有 SQLite bridge 的 `count_chunks` / `count_embeddings`。
- `storage.*Bytes` 通过目录大小 helper 计算。

---

## UI 结构设计

### A. SettingsPage: Personal Knowledge 管理

在现有 Personal Memory 区块内保持轻量化，不新增内部状态摘要卡：

1. **维护操作**
   - `Rebuild Personal Memory`
   - `Clear Personal Memory`
   - 沿用当前已有交互与按钮命名，不额外引入新的 personal maintenance 入口

2. **默认不展示治理活动流**
   - personal 文件数、daily 文件数、entries、storage、index status 仅作为内部状态来源，不在 Settings 中单独展示给用户
   - orphan/migration 等 ops 事件保留在 runtime-store / internal contract，不在默认 Settings UI 直出

### B. KnowledgePage: Project Knowledge 管理

保持现有页面骨架不变，只扩展：

1. **顶部 action row**
   - `Import Files`
   - `Import Folder`
   - `Export Archive`
   - `Import Archive`
   - `Rebuild Index`

2. **状态卡区**
   - Files summary
   - Index summary
   - Storage summary
   - Embedding summary

3. **Knowledge Inventory**
   - 沿用现有 source list

---

## 核心流程设计

### A. Personal status + rebuild

1. SettingsPage 初始化时调用 `kb:personal:getStatus`。
2. personal status 仅用于按钮可用态、轻量状态提示与错误反馈，不渲染 `Files / Daily Files / Entries / Storage / Index Status` 摘要卡。
3. 点击 `Rebuild Personal Memory`：
   - 继续复用现有 `kb:personal:rebuildIndex`
   - personalKbService 额外写 `personal/ops-log.ndjson`
   - UI 刷新 status 与维护反馈

### B. Project status + rebuild

1. KnowledgePage 加载时继续调用 `kb:project:getStatus`。
2. status payload 扩展为 `summary + embedding + index + storage + sources`。
3. 点击 `Rebuild Index`：
   - appStore 暴露正式 public action，而不是仅内部 helper
   - 调用 `kb:project:rebuildIndex`
   - 项目 ops 记录新增 `project.rebuild.started/completed/failed`
   - 页面刷新 status

### C. Internal ops record 查询

新增 IPC：

```ts
ipcMain.handle('kb:ops:list', async (_event, payload) => {
  // payload:
  // {
  //   scope: 'personal' | 'project' | 'all',
  //   projectRoot?: string,
  //   limit?: number,
  //   eventPrefix?: string
  // }
})
```

返回：

```ts
type KnowledgeOpsListResult = {
  success: boolean
  records?: KnowledgeOperationRecord[]
  error?: string
}
```

说明：
- `jump` metadata 可以继续保留在 internal contract 中，供后续支持工具或非默认诊断入口使用。
- 12.6 默认终端用户 UI 不消费这些 jump metadata。

---

## 测试策略

### 单元/集成测试

1. `KnowledgeOpsService` 能同时读取 personal/project/global 三类日志。
2. legacy project ops 行能被正确归一化为 `project.source.status`。
3. personal rebuild / clear / commit 会写 personal ops log。
4. orphan delete / rebind 与 archive export / import 事件可在 `kb:ops:list(scope=all)` 中出现。
5. project status payload 包含 index/storage 摘要，且与实际目录/SQLite 计数一致。
6. `Rebuild Index` 成功与失败路径都能反映到 status 与持久化 ops 记录。
7. 默认 Settings / Knowledge 页面不展示 activity feed 或 jump 入口。

### 手动验证

1. 在 Settings 中触发 personal rebuild / clear，确认默认用户界面只保留 maintenance 反馈，不展示 raw summary/activity。
2. 在项目 Knowledge 页导入文件、导出 archive、导入 archive、重建索引，确认状态卡与反馈文案按预期刷新。
3. 检查 personal/project/global `ops-log.ndjson`，确认关键事件已写入并保持归一化读取能力。
4. 触发 orphan delete / rebind，确认事件进入全局 KB ops 留痕。
