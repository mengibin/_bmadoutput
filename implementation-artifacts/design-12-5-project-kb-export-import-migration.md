# Design: Project KB Export / Import Migration

**Story:** `12-5-project-kb-export-import-migration.md`  
**设计原则:** 真相源优先、迁移副本语义、归档结构版本化、导入失败可完整回滚、对现有 Project KB 最小侵入

---

## 设计目标

1. 支持把项目知识库导出为可迁移归档，归档至少包含 `source/`、`extracted/`、`notes/`、`manifest.json` 与完整性校验数据。
2. 支持把知识归档导入到目标项目，并在目标项目内重建索引，而不是复用源项目的派生索引。
3. 导入失败时能把目标项目恢复到导入前状态，避免 manifest / source / extracted / index 部分写入。
4. 导入后的知识必须与源项目完全解耦，后续编辑互不影响。
5. 导出与导入结果要能直接反馈到现有 `Knowledge` 页面，并写入统一知识运维事件，供 Story 12.6 消费。

## 非目标（本 Story 不做）

1. 不做远端同步、云备份或跨设备自动分发。
2. 不做 archive 内部文件级可视化浏览器。
3. 不做“只导入 archive 中部分文件”的细粒度选择器。
4. 不保证跨大版本 archive 向前兼容；本 Story 只要求从 `archiveSchemaVersion=1.0` 开始版本化。

## 当前 Cross-Story 约束（2026-03-13）

1. Story 12.3 已确定 project knowledge 的真相源在 Runtime 私有目录，`index.sqlite` 是派生层，不应作为迁移可信输入。
2. 当前 renderer-safe payload 已移除 `originalPath`；archive 也不得把宿主机原始路径暴露为可迁移元数据。
3. 当前 `ProjectKbService` 已承载导入文件、导入目录、重建索引与检索逻辑；12.5 需要尽量复用其提取与重建能力。
4. Story 12.6 将消费 export / import 迁移事件，因此 12.5 需要从第一版开始写结构化 ops 记录。
5. `notes/` 在当前代码中尚未形成 UI 能力，但 archive 结构要先保留该目录作为格式稳定锚点，即使为空也要存在。

## Brownfield 约束与复用点

1. **现有 Project KB 目录结构已稳定**  
   当前 live 目录为：

   ```text
   runtime-store/projects/<projectId>/knowledge/
     source/
     extracted/
     index.sqlite
     manifest.json
     ops-log.ndjson
   ```

2. **现有 ProjectKbService 已能 rebuild**  
   当前 `rebuildIndex()` 已具备逐 source 重提取、分块、写 SQLite、更新 manifest 的能力；12.5 应复用这条逻辑，而不是做第二套“迁移专用索引器”。

3. **现有 RuntimeStore 已有 ZIP 与安全路径处理经验**  
   `runtimeStore.ts` 已使用 `AdmZip` 并具备 safe path resolve 相关 helper；archive 读写可沿用同类约束，避免 zip-slip。

4. **现有 KnowledgePage 已有项目级运维入口**  
   12.5 的 UI 最适合落在已存在的 `KnowledgePage.tsx` 顶部 action 区，而不是另起一个设置页入口。

---

## 改动范围

| 文件 | 改动类型 | 说明 |
|------|----------|------|
| `crewagent-runtime/electron/stores/runtimeStore.ts` | MODIFY | 新增 `notes/` 路径初始化、全局 KB ops log helper、archive 临时目录 helper |
| `crewagent-runtime/electron/services/projectKbService.ts` | MODIFY | 新增 export / import archive、staging + rollback、workspace rebuild 复用层 |
| `crewagent-runtime/electron/services/projectKbService.test.ts` | MODIFY | 增加 export/import/rollback/copy-semantic 回归 |
| `crewagent-runtime/electron/main.ts` | MODIFY | 新增 archive 导出/导入 IPC 与对话框 |
| `crewagent-runtime/electron/preload.ts` | MODIFY | 暴露 archive IPC |
| `crewagent-runtime/electron/electron-env.d.ts` | MODIFY | 补充 archive 导出/导入结果类型 |
| `crewagent-runtime/src/stores/appStore.ts` | MODIFY | 新增 export/import archive action 与结果状态 |
| `crewagent-runtime/src/pages/KnowledgePage/KnowledgePage.tsx` | MODIFY | 增加 `Export Archive` / `Import Archive` 入口与摘要反馈 |
| `crewagent-runtime/src/pages/KnowledgePage/KnowledgePage.css` | MODIFY | archive action / summary 样式 |

---

## 归档格式设计

### 1) ZIP 目录布局

```text
<archive>.zip
  manifest.json
  checksums.json
  source/
    ...
  extracted/
    ...
  notes/
    ...
```

说明：
- `index.sqlite` 不进入 archive，因为它是派生层，导入后必须重建。
- `notes/` 永远存在；如果源项目没有 notes 文件，则导出为空目录。
- `checksums.json` 存放 archive 内成员级 hash；archive 整体 SHA-256 通过导出结果回传，并以内嵌 zip comment 的方式随单文件归档一起携带。

### 2) `manifest.json`

```json
{
  "archiveSchemaVersion": "1.0",
  "kind": "crewagent.project-kb-archive",
  "exportedAt": "2026-03-13T10:00:00.000Z",
  "sourceProject": {
    "projectId": "proj-123",
    "projectName": "CrewAgent Demo"
  },
  "summary": {
    "sourceFiles": 12,
    "extractedFiles": 12,
    "notesFiles": 0,
    "totalBytes": 3145728
  },
  "sources": [
    {
      "sourceId": "src-001",
      "fileName": "spec.pdf",
      "originType": "file",
      "storedPath": "source/src-001-spec.pdf",
      "extractedPath": "extracted/src-001.md",
      "mime": "application/pdf",
      "sizeBytes": 12345,
      "importedAt": "2026-03-13T09:50:00.000Z",
      "updatedAt": "2026-03-13T09:55:00.000Z",
      "status": "ready",
      "processingStage": "completed",
      "extractionMethod": "multimodal",
      "chunkCount": 8,
      "errorCode": null,
      "errorMessage": null
    }
  ]
}
```

约束：
- 不包含 `originalPath`。
- `embedding.configFingerprint` 与 ready embedding 状态不进入 archive 的可信契约；导入后统一按目标环境重建。

### 3) `checksums.json`

```json
{
  "algorithm": "sha256",
  "files": [
    {
      "path": "source/src-001-spec.pdf",
      "sha256": "abc123...",
      "sizeBytes": 12345
    },
    {
      "path": "extracted/src-001.md",
      "sha256": "def456...",
      "sizeBytes": 6789
    }
  ]
}
```

---

## 关键设计决策

1. **archive 迁移 truth-source，不迁移 `index.sqlite`**  
   目标项目的 embedding 配置、provider 能力、fingerprint 可能不同，因此导入后只能 rebuild，不能把旧向量静默当成 ready。

2. **导入采用副本语义，sourceId 必须重绑**  
   导入成功后，目标项目中的 source record 要分配新的 `sourceId` 与新的 `storedPath/extractedPath`，避免与源项目共享逻辑身份或发生 manifest 冲突。

3. **导入去重以内容 hash 为准，不以文件名为准**  
   同名不同内容不能误判为重复；同内容不同文件名可以作为 `skipped` 返回。

4. **导入工作目录与 live 目录分离，完成后原子替换**  
   12.5 不在 live `knowledge/` 上边改边 rebuild，而是在 `knowledge.__importing__<jobId>/` 工作目录中完成 merge 与 rebuild，成功后再 swap。

5. **`notes/` 作为格式锚点先落地**  
   本 Story 允许 `notes/` 为空，也不强制当前 UI 提供 notes 编辑能力；但 archive schema 从第一版开始就固定保留该目录。

6. **迁移事件进入统一 KB ops**  
   `project.export.completed`、`project.import_archive.completed`、`project.import_archive.failed`、`project.import_archive.rollback_completed` 必须写入 ops 记录，便于 Story 12.6 展示。

---

## 核心流程设计

### A. 导出归档（AC-1）

1. Renderer 在 `KnowledgePage` 触发 `Export Archive`。
2. Main 打开 `showSaveDialog()`，默认文件名建议：

   ```text
   <projectName>-knowledge-2026-03-13.zip
   ```

3. `ProjectKbService.exportArchive(projectRoot, outputPath)` 执行：
   - `ensureInitialized(projectRoot)`
   - 读取 live manifest
   - 准备 staging 目录：

     ```text
     runtime-store/_project-kb-imports/<jobId>/export/
     ```

   - 复制 `source/`、`extracted/`、`notes/` 到 staging
   - 生成 sanitized `manifest.json`
   - 生成 `checksums.json`
   - 压缩为 zip
   - 计算 archive SHA-256，并写入 zip comment，保持导出物为单个 `.zip`
4. 返回结果：

```ts
type ProjectKnowledgeExportResult = {
  success: boolean
  archivePath?: string
  archiveSha256?: string
  sourceFiles?: number
  extractedFiles?: number
  notesFiles?: number
  error?: string
}
```

5. 写入 ops 事件 `project.export.completed`。

### B. 导入归档（AC-2 / AC-3 / AC-4）

1. Renderer 触发 `Import Archive`。
2. Main 打开 `showOpenDialog()`，限制 `*.zip`。
3. Service 验证 archive：
   - ZIP 结构存在 `manifest.json`、`checksums.json`、`source/`、`extracted/`、`notes/`
   - 路径无 traversal / absolute path
   - `archiveSchemaVersion === 1.0`
   - zip comment 中的 archive SHA-256 与单文件归档本身一致
   - 每个成员的 `sha256` 与 `checksums.json` 一致
4. 准备工作目录：

```text
runtime-store/projects/<projectId>/knowledge.__importing__<jobId>/
```

5. 如果 live `knowledge/` 已存在，则先完整复制到 working root；否则基于默认空 knowledge 结构初始化 working root。
6. 将 archive 内容 merge 到 working root：
   - 为每个导入 source 分配新的 `sourceId`
   - 新的 `storedPath`、`extractedPath` 使用新 `sourceId`
   - `originalPath` 统一置空，不进入目标 manifest
   - 基于 source file hash 去重，重复内容记为 `skipped`
7. 在 working root 上执行 rebuild：
   - 复用 `ProjectKbService` 的低层 workspace rebuild helper
   - 重建新的 `index.sqlite`
   - 更新 `manifest.embedding` 为目标环境当前状态
8. rebuild 成功后：
   - `knowledge/` 重命名为 `knowledge.__backup__<jobId>/`
   - `knowledge.__importing__<jobId>/` 原子重命名为 `knowledge/`
   - 删除 backup
9. 任何阶段失败：
   - 若尚未 swap，只删除 working root，live root 保持不变
   - 若已 swap 但后续清理失败，则回滚 backup -> live
   - 返回 `rollbackCompleted=true`

### C. 导入 summary

```ts
type ProjectKnowledgeArchiveImportSummary = {
  imported: number
  skipped: number
  failed: number
  rebuiltSources: number
  rollbackCompleted: boolean
}
```

---

## UI 设计

### 1) KnowledgePage 顶部动作区

在现有 `Import Files / Import Folder` 旁新增：

- `Export Archive`
- `Import Archive`

### 2) 导出结果反馈

成功后显示：

```text
Archive exported.
12 source files · 12 extracted files · SHA-256 copied.
```

### 3) 导入结果反馈

成功后显示：

```text
Archive imported.
Imported 8 · Skipped 2 · Rebuilt 8
```

失败且回滚成功时显示：

```text
Import failed. Target knowledge was restored to its previous state.
```

---

## 测试策略

### 单元/集成测试

1. export 生成 zip，且 archive 内包含 `manifest.json/checksums.json/source/extracted/notes`。
2. export manifest 不包含 `originalPath`。
3. import 成功后目标 manifest 中 sourceId 已 rebased。
4. import 重复 archive 时，重复内容按 hash 进入 `skipped`。
5. import 成功后 rebuild 结果存在，且 `index.sqlite` 不是直接复用源 archive 文件。
6. import 某阶段失败时，rollback 后 live knowledge root 与导入前字节级一致。
7. 导入后修改目标项目中的知识文件，不影响源项目目录内容。

### 手动验证

1. 在项目 A 导入若干知识文件。
2. 导出 archive 到本地。
3. 在项目 B 导入 archive。
4. 确认项目 B 的 Knowledge 列表出现文件，检索可用。
5. 修改项目 B 的一个 source/extracted 文件，确认项目 A 不受影响。
6. 构造损坏 zip 或篡改 checksum，确认导入失败且回滚完成。
