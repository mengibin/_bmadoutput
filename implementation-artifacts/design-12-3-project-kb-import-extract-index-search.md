# Design: Project KB Import / Extract / Index / Retrieval Tool

**Story:** `12-3-project-kb-import-extract-index-search.md`  
**设计原则:** 项目级严格隔离、Runtime 私有存储、知识固化优先、LLM 工具消费优先、处理状态透明、沿用现有 runtime 工作区 UI、Embedding 可选启用（OpenAI-style interface only）

---

## 设计目标

1. 在每个项目的 Runtime 私有目录下初始化独立的 `knowledge/` 空知识库。
2. 支持从 `Knowledge` 面板导入文件或文件夹，并提供逐文件处理进度与失败原因。
3. 将导入内容固化为 runtime-private 的知识资产，形成稳定的项目知识来源。
4. 向 Runtime ToolHost 暴露 project-scoped retrieval tool，供 LLM 按项目检索已固化知识。
5. 在 UI 中明确展示：
   - 知识文件总数
   - 知识文件列表
   - 每个文件的处理状态
6. 当用户尚未配置 embedding 时，在 `Knowledge` 面板内提供内联启用与配置入口。

---

## 非目标（本 Story 不做）

1. 不做人类手动检索/搜索界面。
2. 不做人类知识内容浏览器，不提供文档正文阅读体验。
3. 不做结果引用、snippet 展示、source trace 的人类检索结果列表。
4. 不做单条删除、批量清空、重建索引等治理动作；这些留到后续 story。
5. 不做跨项目迁移/导入导出包。
6. 不引入外部向量库或外部 RAG 服务。

---

## Brownfield 约束与复用点

1. **现有 runtime UI 已有稳定壳层**  
   `Sidebar.tsx` 提供项目一级导航；`WorkspacePage.tsx` 提供 `workspace-panel + workspace-main` 结构；12.3 应沿用这套结构，而不是新造独立页面布局语言。

2. **当前工作区左侧面板宽度与交互习惯已固定**  
   `WorkspacePage.css` 中 `workspace-panel.is-open` 为 `360px`。`Knowledge` 应使用同一宽度节奏，把“文件总数 / 导入动作 / 文件列表”放进这一侧面板。

3. **`SettingsPage` 是全局配置中心**  
   当前 `llm` / `multimodalLlm` 都由 `SettingsPage.tsx + appStore.ts` 管理。`embeddingLlm` 应落在同一设置体系下，不应新造 project-local settings。

4. **多模态提取链路已存在**  
   `FileSystemToolHost.mediaExtract()` 已覆盖 PDF / 图片的 readable extraction。12.3 只需要提炼可复用 helper，并坚持“多模态可用时优先多模态”的策略。

5. **Runtime Store 已有原子写模式**  
   `runtimeStore` 已具备 `writeJsonAtomic()` / `writeTextAtomic()` 模式。`manifest.json`、`extracted/*.md` 与 knowledge 状态更新应复用这一模式。

6. **Embedding 配置与 Project KB 状态需要跨主进程 / 渲染层分层处理**  
   `embeddingLlm` 是全局 runtime setting，但 Project KB manifest 还需要记录“当前索引是按哪份 embedding 配置生成的”。同时，主进程内部可保留 `originalPath` 用于审计/重导入诊断，renderer 只能收到剥离后的 renderer-safe status payload。

7. **现有 Runtime 已有 ToolHost 体系可复用**  
   `fileSystemToolHost.ts` / `toolHost.ts` 已承载系统工具调用。Project KB 的检索入口应接入同一套 tool surface，而不是在 Renderer 里实现一套人类搜索逻辑。

---

## 设计决策

1. **Knowledge 模块以“系统检索 + 运营可见性”为主，不做人类检索体验**  
   12.3 的核心不是“人怎么搜”，而是“项目知识有没有被成功固化”以及“LLM 是否能通过 project-scoped retrieval tool 正确拿到这些知识”。

2. **沿用 runtime 现有双区结构，但 embedding onboarding 放在左侧 panel**  
   `Knowledge` 路由进入后使用：
   - 左侧 `workspace-panel`：知识总览、导入入口、embedding 配置卡、文件列表
   - 右侧 `workspace-main`：空态说明、库存统计、处理状态列表

3. **文件导入与文件夹导入拆成两个明确动作**  
   不做一个混合模糊入口。UI 明确提供：
   - `Import Files`
   - `Import Folder`
   
   这样和用户心智一致，也更利于 Electron `dialog` 分别处理。

4. **Embedding 是全局设置，但 `Knowledge` 面板允许内联配置**  
   `embeddingLlm` 的权威存储仍在全局 `Settings`。  
   但在 `Knowledge` 面板中，可以直接显示：
   - 开关：`embedding is enable`
   - interfaceType / baseUrl / model / apiKey / timeout

   本 story 的 embedding UI 范围仅要求 OpenAI-style interface：
   - `OpenAI`
   - `OpenAI Compatible`
   
   保存动作仍写入全局设置，而不是写入 project-local knowledge config。

5. **Embedding 向量必须识别配置漂移，不允许静默复用旧索引**  
   Project KB manifest 需要记录 embedding config fingerprint（例如 provider + baseUrl + model）。  
   当当前全局 embedding 配置与 manifest 中记录的 fingerprint 不一致时：
   - `Knowledge` 状态不能继续显示 embedding ready
   - retrieval 必须在使用前触发 rebuild / invalidation，而不是直接拿旧向量参与 hybrid

6. **若已配置 embedding，则开关默认开启**  
   `Knowledge` 面板只需要读取设置并镜像其状态。  
   用户打开 `Knowledge` 时，如果 `embeddingLlm` 已有效存在，则默认展示为已启用状态。

7. **导入后 UI 只展示三个核心维度**  
   - 总量：多少个知识文件
   - 清单：有哪些知识文件
   - 状态：ready / processing / failed 以及失败原因

8. **索引是系统能力，不是人类阅读界面**  
   `index.sqlite` 与可选 embeddings 的主要消费者是 Runtime ToolHost。UI 只展示导入与处理状态，不提供搜索框、检索结果列表、正文预览或人工消费式知识阅读区。

---

## UI 结构

### 1) 顶层导航

`Sidebar.tsx` 的 project nav 新增：

- `Files`
- `Knowledge`
- `Works`

`Knowledge` 与 `Files / Works` 同级。

### 2) Workspace 布局

`WorkspacePage.tsx` 的 panel 类型扩展为：

```ts
type WorkspacePanel = 'files' | 'knowledge' | 'works' | null
```

进入 `Knowledge` 后，布局为：

```text
Sidebar
  └─ WorkspacePage
      ├─ workspace-panel (360px)
      │   └─ KnowledgePanel
      └─ workspace-main
          └─ KnowledgeContent
```

### 3) 左侧 `KnowledgePanel`（360px）

左侧面板承担结构化管理信息，不承担复杂编辑，也不承担知识正文消费。

包含：

1. 头部摘要
   - 标题：`Knowledge`
   - 副标题：`N files`

2. 导入动作
   - `Import Files`
   - `Import Folder`

3. 状态统计
   - `Total`
   - `Ready`
   - `Processing`
   - `Failed`

4. Embedding 开关与配置卡片
   - 开关：`embedding is enable`
   - 接口类型：`OpenAI | OpenAI Compatible`
   - `baseUrl`
   - `model`
   - `apiKey`
   - `timeout`
   - 保存反馈

5. 文件列表
   每个列表项至少显示：
   - 文件名 / 文件夹来源名
   - 导入时间
   - 状态 badge：`ready | processing | failed`
   - 若失败则显示一行错误摘要

### 4) 右侧 `KnowledgeContent`

#### 空态

显示：

- 空态标题：强调“先固化项目知识”
- 简短说明：这里用于沉淀项目资料，并查看处理状态
- 导入入口说明：通过左侧面板执行 `Import Files / Import Folder`

#### 已导入态

显示：

1. 统计摘要
   - `Total`
   - `Ready`
   - `Processing`
   - `Failed`

2. 知识文件列表
   - 文件名 / 文件夹来源名
   - 状态 badge：`ready | processing | failed`
   - 提取方式与失败原因摘要

3. 导入提示
   - 继续导入时仍通过左侧面板执行 `Import Files / Import Folder`

---

## 存储与数据模型

### 1) Runtime 目录布局

```text
<runtime-userData>/runtime-store/projects/<projectId>/knowledge/
  source/
    <sourceId>-<safeName>.<ext>
  extracted/
    <sourceId>.md
  index.sqlite
  manifest.json
  ops-log.ndjson
```

说明：

- `source/` 存放复制后的原始知识文件
- `extracted/` 存放标准化文本层
- `index.sqlite` 作为后台索引载体，为后续系统使用保留
- `manifest.json` 负责 UI 直接消费的元数据与状态

### 2) `manifest.json`

建议结构：

```json
{
  "version": "1.0",
  "projectId": "abc123",
  "initializedAt": "2026-03-11T09:00:00.000Z",
  "updatedAt": "2026-03-11T09:10:00.000Z",
  "embedding": {
    "enabled": true,
    "provider": "openai-compatible",
    "model": "text-embedding-3-small",
    "lastIndexedAt": "2026-03-11T09:09:00.000Z",
    "lastError": null,
    "configFingerprint": "{\"provider\":\"openai-compatible\",\"baseUrl\":\"https://api.deepseek.com\",\"model\":\"text-embedding-3-small\"}"
  },
  "summary": {
    "totalFiles": 7,
    "readyFiles": 5,
    "processingFiles": 1,
    "failedFiles": 1
  },
  "sources": [
    {
      "sourceId": "src-001",
      "fileName": "spec.pdf",
      "originType": "file",
      "storedPath": "source/src-001-spec.pdf",
      "originalPath": "/Users/me/Desktop/spec.pdf",
      "mime": "application/pdf",
      "sizeBytes": 12345,
      "importedAt": "2026-03-11T09:02:00.000Z",
      "status": "ready",
      "extractedPath": "extracted/src-001.md",
      "chunkCount": 8,
      "processingStage": "completed",
      "errorCode": null,
      "errorMessage": null
    }
  ]
}
```

状态约束：

- `status`: `ready | processing | failed`
- `processingStage`: `queued | copying | extracting | normalizing | indexing | completed | failed`
- 失败文件保留 `errorCode` / `errorMessage`

补充说明：

- `originalPath` 只保留在主进程内部 manifest，用于诊断和审计；不通过 `kb:project:getStatus` 暴露到 renderer
- `embedding.configFingerprint` 用于判断当前索引是否与全局 embedding 配置一致，防止 stale vectors 静默参与 hybrid retrieval

### 3) `index.sqlite`

本 story 中，`index.sqlite` 是后台索引资产，也是 Project KB retrieval tool 的查询基础。  
UI 不直接依赖检索结果，只依赖 `manifest.json` 汇总状态。

推荐最低职责：

- 记录已处理 chunk
- 记录可选 embeddings
- 提供 project-scoped retrieval query 的后端数据源

---

## IPC / Tool 合同

建议新增：

```ts
openProjectKnowledgeFilesDialog()
openProjectKnowledgeFolderDialog()
kbProjectImportFiles(payload: { projectRoot: string; filePaths: string[] })
kbProjectImportFolder(payload: { projectRoot: string; directoryPath: string })
kbProjectGetStatus(payload: { projectRoot: string }): ProjectKnowledgeStatusPayload
onProjectKnowledgeImportProgress(callback)
```

同时在 Runtime ToolHost 暴露内部可调用入口：

```ts
projectKnowledgeRetrieve(payload: {
  projectRoot: string
  query: string
  topK?: number
})
```

约束：

- 仅检索 `ready` sources
- 返回 chunk 文本 + source metadata
- 由 LLM 工具层调用，不在 Renderer 提供人工搜索入口
- `kb:project:getStatus` 返回 renderer-safe payload，不包含 `originalPath`

设置相关：

- 复用现有 `saveSettings`
- 新增或扩展 `embeddingLlm`

`Knowledge` 空态中的 embedding 保存，应调用同一条全局设置保存链路。

---

## 核心流程

### A. 项目知识库初始化

位置：`runtimeStore.createProject()` / `runtimeStore.openProject()`

1. 先执行现有 `ensureProjectRuntimeDirs(projectRoot)`
2. 再执行 `ensureProjectKnowledgeDirs(projectRoot)`，创建：
   - `knowledge/`
   - `knowledge/source/`
   - `knowledge/extracted/`
   - 空的 `manifest.json`
   - 空的 `index.sqlite`
   - 空的 `ops-log.ndjson`
3. `getProjectKnowledgeStatus(projectRoot)` 返回摘要与文件列表

### B. 文件导入

入口一：`Import Files`

- `dialog.showOpenDialog`
- `properties: ['openFile', 'multiSelections']`

入口二：`Import Folder`

- `dialog.showOpenDialog`
- `properties: ['openDirectory']`

文件夹导入行为：

1. 扫描目录
2. 按 allowlist 过滤支持的文件类型
3. 逐文件进入统一导入流水线
4. 文件夹本身不作为知识实体，目录内文件才作为知识文件进入 `manifest`

### C. 单文件处理流水线

1. Copy 到 `knowledge/source/`
2. Extract
   - `.md` / `.txt`: 直接读取
   - `.docx`: `adm-zip` + `word/document.xml`
   - `.pdf` / 图片：复用 `mediaExtract` helper
3. Normalize 到 `extracted/`
4. 写 `manifest` 状态
5. 背景更新 `index.sqlite`
6. 若 embedding 开启，则执行 embedding 流程

### D. Embedding 配置与开关

#### 默认规则

1. 若全局没有有效 `embeddingLlm`，则 `embedding is enable` 默认为关闭
2. 若全局已有有效 `embeddingLlm`，则 `embedding is enable` 默认为开启

#### 在 `Knowledge` 空态中打开开关

1. 展开嵌入式配置表单
2. 用户填写 interfaceType / baseUrl / model / apiKey / timeout
3. 点击保存后，调用与 `Settings` 相同的全局保存逻辑
4. 保存成功后：
   - `Settings` 中应可见同一配置
   - `Knowledge` 面板中开关保持开启
5. 如果当前项目已有 vectors，且新配置与旧 fingerprint 不一致，则状态先显示为 not ready，随后在 rebuild 完成后恢复为 ready

#### 在 `Knowledge` 中关闭开关

建议行为：

- 关闭仅表示“当前不启用 embedding 流程”
- 配置可保留，避免每次重新填写
- 若 manifest 中存在旧 fingerprint，关闭开关不会删除向量，但 UI 不应把它们视为当前有效的 embedding ready 状态

### E. LLM Retrieval Tool 调用

1. LLM 在项目上下文内触发 project knowledge retrieval tool
2. ToolHost 将请求路由到 `projectKnowledgeRetrieve`
3. Runtime 读取当前项目 `index.sqlite`
4. 若 manifest 中 embedding fingerprint 与当前全局配置不一致，则先触发 rebuild / invalidation
5. 仅在 `ready` sources 范围内执行检索
6. 返回：
   - chunk text
   - sourceId / fileName
   - chunk index / relevance score（若有）
7. LLM 基于返回结果继续推理；这一过程不要求 Renderer 展示人工搜索界面

---

## 改动范围

| 文件 | 改动类型 | 说明 |
|------|----------|------|
| `crewagent-runtime/src/components/layout/Sidebar.tsx` | MODIFY | 新增 `Knowledge` 一级导航 |
| `crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.tsx` | MODIFY | 扩展 `knowledge` panel |
| `crewagent-runtime/src/pages/KnowledgePage/KnowledgePage.tsx` | NEW | Knowledge 左侧面板与右侧内容区 |
| `crewagent-runtime/src/pages/KnowledgePage/KnowledgePage.css` | NEW | Knowledge 页面样式，跟随现有 workspace 节奏 |
| `crewagent-runtime/src/stores/appStore.ts` | MODIFY | 新增 knowledge 状态、文件列表、embedding 设置读写；renderer 侧状态类型剥离 `originalPath` |
| `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx` | MODIFY | 增加 `embeddingLlm` 全局配置 |
| `crewagent-runtime/electron/stores/runtimeStore.ts` | MODIFY | knowledge 路径、manifest/index、embedding setting 持久化、renderer-safe status payload |
| `crewagent-runtime/electron/services/projectKbService.ts` | NEW | 文件/文件夹导入编排、提取、索引、状态同步、stale embedding drift rebuild |
| `crewagent-runtime/electron/services/fileSystemToolHost.ts` | MODIFY | 暴露 project knowledge retrieval tool，供 LLM 调用 |
| `crewagent-runtime/electron/main.ts` | MODIFY | knowledge 文件/文件夹导入 IPC 与进度广播 |
| `crewagent-runtime/electron/preload.ts` | MODIFY | 暴露 knowledge IPC |

---

## 测试策略

### 1) `runtimeStore` 单测

- 新建/打开项目时自动创建 `knowledge/` 目录结构
- `manifest.json` / `index.sqlite` 路径位于正确 project runtime root
- `embeddingLlm` 设置可持久化与回读

### 2) `projectKbService` 单测

- 文件导入成功并产生 source 记录
- 文件夹导入会扫描并导入受支持文件
- 某个文件失败时其它文件仍成功
- 状态正确写入 `ready / processing / failed`
- 多模态可用时图片与扫描型 PDF 优先走多模态路径
- project-scoped retrieval query 只返回 `ready` sources
- embedding config 漂移后，旧 vectors 会被识别为 stale，并在 retrieval 前自动 rebuild

### 3) ToolHost / retrieval 测试

- LLM 调用 project knowledge retrieval tool 时能命中当前项目索引
- 返回结果包含 chunk 文本和 source metadata
- 跨 project 不串读

### 4) Renderer / store 测试

- `Knowledge` 左侧面板显示总数与文件列表
- 空态提示通过左侧面板执行 `Import Files` / `Import Folder`
- 左侧 `KnowledgePanel` 在未配置 embedding 时显示 `embedding is enable` 开关与配置区
- 若全局已配置 embedding，则开关默认开启
- 已导入态显示文件状态与失败原因
- renderer store 中的 knowledge status 不包含 `originalPath`

### 5) 回归验证

- 不影响现有 `Files` / `Works` 行为
- 不破坏现有 `Settings` 的 LLM / Multimodal 配置路径
- 不破坏现有 `media.extract` 工具能力

---

## 开发建议顺序

1. 先补 `runtimeStore` 的 knowledge path / manifest / status / embedding setting helpers
2. 再实现 `projectKbService` 的逐文件处理、索引与 retrieval query
3. 接着把 project knowledge retrieval tool 接到 `fileSystemToolHost`
4. 然后实现文件导入与文件夹导入 IPC
5. 再补 `SettingsPage` 的 `embeddingLlm` 配置
6. 最后实现 `KnowledgePage` 的空态、文件列表、状态显示与内联 embedding onboarding，并补回归测试与文档
