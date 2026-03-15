# Design: Project KB Orphan Cleanup Integration

**Story:** `12-4-project-kb-orphan-cleanup-integration.md`  
**设计原则:** `projectId` 稳定映射、清理范围显式、删除整棵 runtime root、重绑定不搬迁数据、结果可观测

---

## 设计目标

1. 在 orphan 检测结果中明确展示“知识残留也会被清理”的语义，而不是只显示笼统数据大小。
2. 为 orphan 条目补充 project knowledge 摘要，至少包含 knowledge included、source count、ready count、knowledge bytes。
3. 保持删除实现为删除 `runtime-store/projects/<projectId>/` 整棵 runtime root，避免聊天与知识分开清理导致残留不一致。
4. 保持 rebind 为“恢复 projectRoot 绑定”，而不是复制或迁移 runtime 数据；已有 conversations / knowledge 直接继续可用。
5. 删除与重绑定返回可供 Renderer 直接展示的 outcome summary，并写入统一 KB ops 事件，供 Story 12.6 的管理 UI 消费。

## 非目标（本 Story 不做）

1. 不做 orphan data 的细粒度选择删除，不支持“只删聊天不删知识”。
2. 不做已删除 orphan data 的回收站或恢复能力。
3. 不做 orphan knowledge 内容浏览器或手动检索 UI。
4. 不做 project knowledge 索引重建、导入导出或迁移逻辑；这些分别属于 Story 12.5 / 12.6。

## 当前 Cross-Story 约束（2026-03-13）

1. Story 8.4 已经落地 orphan 主链路：`detect/rebind/delete/ignore`，12.4 只能扩展语义，不能重做交互骨架。
2. Story 12.3 已经把 project knowledge 固化到 `runtime-store/projects/<projectId>/knowledge/`，其中 `manifest.json`、`source/`、`extracted/`、`index.sqlite`、`ops-log.ndjson` 都属于 orphan cleanup 范围。
3. 当前 runtime 数据目录是按 `projectId` 挂在 `runtime-store/projects/<projectId>/` 下，不是按旧 `projectRoot` 路径命名；rebind 必须复用既有 runtime root。
4. Renderer-safe 边界已确定：project KB 面向前端的 payload 不能重新暴露 `originalPath` 等宿主机路径细节。
5. Story 12.6 会消费 orphan delete / rebind 事件，因此 12.4 应从现在开始写入结构化知识运维事件，而不是只靠 UI 临时 toast。

## Brownfield 约束与复用点

1. **现有 orphan UI 已稳定存在**  
   `OrphanProjectList.tsx` 已经直接调用 `projects:getOrphans / deleteOrphan / rebindOrphan / ignoreOrphan`，12.4 应延续这一入口，不必先重构到 appStore。

2. **现有 orphan 检测已经按 projectId 找 runtime root**  
   `RuntimeStore.detectOrphanProjects()` 当前使用 `getProjectRuntimeRootById(projectId)` 统计目录大小与 conversation 数量，这正是 knowledge residual 正确落位的基础。

3. **当前 delete 实现已经删除整棵 runtime root**  
   `deleteOrphanData()` 已直接 `rmSync(runtimeRoot, { recursive: true, force: true })`。12.4 不应该拆成多段删除，只需把“knowledge 也被包含”显式化并返回 summary。

4. **当前 rebind 顺序已经基本正确**  
   `rebindProject()` 会先写入新的 `.crewagent.json`，再调用 `ensureProjectRuntimeDirs()` 和 `ensureProjectKnowledgeInitialized()`；这是保证 runtime root 继续锚定旧 `projectId` 的关键顺序，不能打乱。

5. **当前 Project KB 状态语义可以复用**  
   `manifest.summary.totalFiles / readyFiles / failedFiles` 已是最廉价的 orphan knowledge 摘要来源，无需重新扫描 knowledge 内容级数据。

---

## 改动范围

| 文件 | 改动类型 | 说明 |
|------|----------|------|
| `crewagent-runtime/electron/stores/runtimeStore.ts` | MODIFY | 扩展 orphan payload、删除/重绑结果结构、knowledge 摘要统计、全局 KB ops 记录 |
| `crewagent-runtime/electron/stores/runtimeStore.test.ts` | MODIFY | 增加 orphan detection/delete/rebind 对 knowledge residual 的回归测试 |
| `crewagent-runtime/electron/main.ts` | MODIFY | IPC 返回结构扩展为 outcome summary |
| `crewagent-runtime/electron/preload.ts` | MODIFY | 暴露扩展后的 orphan IPC 返回 |
| `crewagent-runtime/electron/electron-env.d.ts` | MODIFY | 更新 OrphanProject / delete / rebind 类型 |
| `crewagent-runtime/src/components/OrphanProjectList.tsx` | MODIFY | 增加 knowledge included 文案、结果提示、重绑恢复提示 |
| `crewagent-runtime/src/pages/SettingsPage/SettingsPage.css` | MODIFY | orphan knowledge 摘要与状态样式 |

---

## 数据模型设计

### 1) Orphan 条目扩展

```ts
interface OrphanProjectKnowledgeSummary {
  included: true
  totalFiles: number
  readyFiles: number
  failedFiles: number
  totalBytes: number
  hasManifest: boolean
  hasIndex: boolean
}

interface OrphanProject extends ProjectMetadata {
  isOrphan: true
  conversationCount: number
  totalSizeBytes: number
  isRemovable: boolean
  cleanupScope: {
    conversations: true
    knowledge: true
  }
  knowledge: OrphanProjectKnowledgeSummary
}
```

说明：
- `totalSizeBytes` 继续表示整棵 runtime root 大小，包含 conversations / runs / state / knowledge。
- `knowledge.totalBytes` 是其中 knowledge 子树大小，用于 UI 显示“知识残留已纳入清理范围”，但不参与二次累加。
- 若 `knowledge/manifest.json` 缺失，则 `totalFiles/readyFiles/failedFiles` 退化为 `0`，但 `included=true` 仍成立。

### 2) 删除结果结构

```ts
type DeleteOrphanDataResult = {
  success: boolean
  result?: {
    deletedAt: string
    conversationCount: number
    knowledgeFiles: number
    knowledgeBytes: number
    totalSizeBytes: number
  }
  error?: string
}
```

### 3) 重绑定结果结构

```ts
type RebindProjectResult = {
  success: boolean
  project?: ProjectMetadata
  config?: ProjectConfig
  recovery?: {
    conversationCount: number
    knowledgeFiles: number
    knowledgeReadyFiles: number
  }
  error?: string
  errorCode?: 'PROJECT_ID_MISMATCH'
  conflictProjectId?: string
}
```

### 4) 全局 KB ops 事件（供 Story 12.6）

新增全局日志文件：

```text
<userData>/runtime-store/kb/ops-log.ndjson
```

本 Story 只要求写入两类事件：

```json
{
  "schemaVersion": "1.0",
  "scope": "project",
  "event": "project.orphan.deleted",
  "projectId": "proj-123",
  "createdAt": "2026-03-13T10:00:00.000Z",
  "summary": {
    "conversationCount": 6,
    "knowledgeFiles": 12,
    "knowledgeBytes": 1048576,
    "totalSizeBytes": 2097152
  }
}
```

```json
{
  "schemaVersion": "1.0",
  "scope": "project",
  "event": "project.orphan.rebound",
  "projectId": "proj-123",
  "createdAt": "2026-03-13T10:05:00.000Z",
  "summary": {
    "conversationCount": 6,
    "knowledgeFiles": 12,
    "knowledgeReadyFiles": 10
  }
}
```

---

## 核心流程设计

### A. Orphan 检测（AC-1）

1. `RuntimeStore.detectOrphanProjects()` 继续遍历 `projects.json` 中的 metadata。
2. 对缺失 `projectRoot` 的项目：
   - 使用 `getProjectRuntimeRootById(projectId)` 找到 runtime root；
   - 继续计算 `conversationCount` 与 `totalSizeBytes`；
   - 额外读取 `knowledge/manifest.json` 与 `knowledge/index.sqlite`，构造 `knowledge` 摘要；
   - 返回 `cleanupScope = { conversations: true, knowledge: true }`。
3. 若 knowledge 子树不存在：
   - `knowledge.included` 仍为 `true`；
   - `knowledge.totalFiles = 0`；
   - `knowledge.totalBytes = 0`。
4. Renderer 在 orphan 条目上显式展示：
   - `Chats: N`
   - `Knowledge: M files`
   - `Cleanup scope: chats + knowledge`

### B. 删除 orphan data（AC-2）

1. Renderer 二次确认文案改为：
   - `Delete all stored data for "<project>"? This removes chats and project knowledge.`
2. Main 继续调用 `runtimeStore.deleteOrphanData(projectId)`。
3. Store 在删除前先抓取 summary：
   - conversationCount
   - knowledgeFiles
   - knowledgeBytes
   - totalSizeBytes
4. 仅当 `projectRoot` 依旧不存在时才允许删除。
5. 删除动作仍然只做一件事：`rmSync(runtimeRoot)`。
6. 删除成功后：
   - 从 `projects.json` 移除 project metadata；
   - 清理 orphan cache；
   - 写入 `runtime-store/kb/ops-log.ndjson`；
   - 返回 summary 给 Renderer。
7. Renderer 刷新 orphan list，并显示删除结果摘要，而不是只静默消失。

### C. Rebind orphan project（AC-3）

1. Renderer 选择新的 `projectRoot`。
2. Store 继续先检查：
   - 新路径存在且可访问；
   - 如目标目录已有其他 `projectId`，沿用现有 force-confirm 逻辑。
3. 成功路径中，必须保持当前顺序：
   - 写入新目录下 `.crewagent.json`，确保 `projectId` 沿用旧值；
   - 再调用 `ensureProjectRuntimeDirs(newProjectRoot)`；
   - 再调用 `ensureProjectKnowledgeInitialized(newProjectRoot)`。
4. 因为 runtime root 是 `projects/<projectId>/`：
   - conversations、knowledge、runs、state 都无需搬迁；
   - rebind 只是让新的 projectRoot 重新指向原有 runtime root。
5. 返回 `recovery` 摘要给 Renderer，直接显示：
   - `Recovered 6 chats and 12 knowledge files`
6. 同步写入全局 KB ops 事件 `project.orphan.rebound`。

---

## UI 设计

### 1) Orphan 列表行文案

每个 orphan item 在原有 `Last opened / Chats / Size` 基础上新增：

- `Knowledge: 12 files · 10 ready`
- `Cleanup scope: chats + knowledge`

若 knowledge manifest 缺失但知识目录存在，则显示：

- `Knowledge residual detected`

### 2) 删除确认文案

从泛化删除提示改为明确范围提示：

```text
Delete all stored data for "Foo"?
This removes orphan chats and project knowledge from Runtime private storage.
```

### 3) 删除/重绑结果反馈

- 删除成功：`Removed 6 chats and 12 knowledge files.`
- 重绑成功：`Project re-bound. 6 chats and 12 knowledge files are available again.`

---

## 测试策略

### 单元/集成测试

1. `detectOrphanProjects()` 在存在 `knowledge/manifest.json` 时返回 knowledge 摘要。
2. knowledge 目录缺失时仍返回 `cleanupScope.knowledge = true` 且摘要为 0。
3. `deleteOrphanData()` 删除整棵 runtime root 后，conversations 与 knowledge 同时消失。
4. `deleteOrphanData()` 在 projectRoot 重新出现时必须拒绝执行。
5. `rebindProject()` 成功后，旧 runtime root 中的 knowledge manifest 仍能被读取。
6. `rebindProject()` 必须返回 recovery summary。
7. orphan delete / rebind 均会写入全局 KB ops 事件。

### 手动验证

1. 创建项目并导入 project knowledge。
2. 在 Finder 中删除原 project root。
3. 打开 Settings -> Project Data，确认 orphan 条目显示 knowledge included。
4. 执行 `Delete Data`，确认 orphan 消失且再次打开 Runtime 不再看到该项目残留。
5. 重复流程，但改为 `Rebind Folder`，确认项目重新可打开且 Knowledge 页仍显示原文件列表。
