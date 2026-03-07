# Design: Drag-and-Drop Batch Extraction + Schema Contract + Observability

**Story:** `mde-1-3-drag-and-drop-batch-extraction-schema-contract-observability.md`  
**设计原则:** 批处理确定性、契约优先、失败隔离、可观测闭环

---

## 设计目标

1. 让用户通过聊天拖拽文件/文件夹即可提供文件上下文，并在需要时触发批量多模态提取，无需手工逐文件调用。
2. 拖拽默认只提供文件上下文，不强制触发提取；由 LLM 根据意图选择 `fs.read` 或 `media.extract`。
3. 保证批处理顺序、输出结构稳定，便于下游自动消费；持久化仅作为可选能力。
4. 将 schema 约束前置为核心契约（provided / generated），严格模式下不允许隐式写入无效记录。
5. 建立文件/页级观测数据，支持复盘和故障定位。

---

## 范围与非范围

### In Scope (MDE-1.3)
- 聊天输入附件（文件/文件夹）元数据规范化与后端接收。
- 目录递归发现与支持格式过滤（`.pdf/.png/.jpg/.jpeg/.webp`）。
- 批处理调度（稳定排序）与聚合结构化结果返回。
- 可选持久化输出（`persistArtifact=true` 时写入 `@project/artifacts/extracted-data.json`）。
- `schema` provided / `schemaIntent` generated 双路径编排。
- strict 模式 schema 校验失败的阻断策略（无效记录不写入产物）。
- 文件/页级日志与关键指标字段标准化。

### Out of Scope (MDE-1.3)
- OFD 专项解码与转换流水线（Phase-2）。
- 大规模并行调度优化（多 worker、队列优先级策略）。
- 领域特化模板库（行业票据特征工程）。

---

## 关键决策

### 决策 1：批处理排序强制确定性
- 规则：按 `sourceFile` 相对路径字典序升序；PDF 同文件内按 `page` 升序。
- 结果：同样输入集在同配置下输出顺序稳定，便于 diff 和自动回归。

### 决策 2：记录级失败隔离 + 运行级可控失败
- 记录级错误（格式不支持、单页解码失败）不终止整批，写入错误元信息。
- 运行级前置错误（如 `LLM_MULTIMODAL_NOT_SUPPORTED`）直接 fail-fast，不进入批处理主体。

### 决策 3：输出文件原子写入
- 默认不强制落盘，提取结果通过 IPC 结构化返回。
- 当 `persistArtifact=true` 时，采用临时文件 + rename 提交，避免中断时产生损坏 JSON。
- 结果：消费者读取持久化文件时要么拿到旧版本，要么拿到完整新版本。

### 决策 4：schema 为一等契约
- provided schema 优先；无 provided 时按 `schemaIntent` 生成 schema 并标记来源。
- strict 模式下，校验失败记录不写入有效结果数组，必须返回结构化错误明细。

---

## 数据契约

### 聊天附件输入

```ts
interface ChatAttachment {
  path: string
  name: string
  mimeType?: string
  isDirectory?: boolean
}
```

### 批处理中间任务

```ts
interface BatchExtractionTask {
  sourceFile: string        // alias path, e.g. @project/docs/a.pdf
  mime: string
  page?: number             // pdf page when expanded
  instruction: string
  schema?: Record<string, unknown>
  schemaIntent?: string
  strict?: boolean
  documentTypeHint?: string
}
```

### 输出结果（默认）

```ts
interface ExtractedDataRecord {
  sourceFile: string
  page: number
  data: Record<string, unknown>
  schemaSource?: 'provided' | 'generated'
  schemaUsed?: Record<string, unknown>
  confidence?: number
  warnings?: string[]
}

interface ExtractedDataResult {
  runId: string
  generatedAt: string
  records: ExtractedDataRecord[]
  errors?: Array<{
    sourceFile: string
    page?: number
    code: string
    message: string
    details?: unknown
  }>
  stats: {
    totalTasks: number
    succeeded: number
    failed: number
  }
  artifactPath?: string // only when persistArtifact=true
}
```

---

## 处理流程

### A. 输入归一化
1. Renderer 从 `ChatInput` 发送附件数组。
2. Main 侧校验路径并解析 alias（限定 `@project/@pkg/@state`）。
3. 目录附件递归展开为候选文件列表，过滤支持格式。

### B. 批任务构建
1. 对每个候选文件构建 `BatchExtractionTask`。
2. PDF 若指定页策略则展开到页级任务；否则按默认策略由 `media.extract` 内部处理。
3. 任务列表按确定性规则排序。

### C. Schema 编排
1. 若传入 `schema`：直接使用，`schemaSource=provided`。
2. 若仅有 `schemaIntent`：调用 schema 生成流程并校验，`schemaSource=generated`。
3. strict 模式下为每个任务应用 schema 校验。

### D. 执行与聚合
1. 串行执行（MVP）调用 `media.extract`，收集结果。
2. 对失败任务写入 `errors[]`，不中断其它任务（除前置 fail-fast 场景）。
3. 聚合 `records/errors/stats` 生成 artifact 数据结构。

### E. 持久化与返回
1. 默认直接返回结构化结果给会话层（含统计与失败清单）。
2. 当 `persistArtifact=true` 时，原子写入 `@project/artifacts/extracted-data.json` 并返回 `artifactPath`。

---

## 模块与文件改动（设计级）

| 文件 | 改动类型 | 说明 |
|------|----------|------|
| `crewagent-runtime/src/pages/RunsPage/components/ChatInput.tsx` | MODIFY | 附件拖拽元数据标准化与发送 |
| `crewagent-runtime/src/pages/WorksPage/WorksPage.tsx` | MODIFY | 发送附件 + 批处理提取意图 |
| `crewagent-runtime/electron/services/agentSessionContract.ts` | MODIFY | 扩展会话消息契约（attachments） |
| `crewagent-runtime/electron/main.ts` | MODIFY | 组装批处理流程，调用提取并返回结构化结果（可选写产物） |
| `crewagent-runtime/electron/services/fileSystemToolHost.ts` | REUSE/MODIFY | 复用 `media.extract`，补齐批处理调用所需字段 |
| `crewagent-runtime/electron/stores/runtimeStore.ts` | MODIFY | 产物路径与 run 级元数据写入支持 |
| `crewagent-runtime/electron/services/llmAdapter.ts` | REUSE/MODIFY | schemaIntent 生成 schema 的调用编排 |
| `crewagent-runtime/src/pages/RunsPage/components/ChatInput.test.tsx` | MODIFY | 附件拖拽 payload 回归测试 |
| `crewagent-runtime/electron/services/fileSystemToolHost.test.ts` | MODIFY | 批处理 + strict + schemaSource 测试覆盖 |

---

## 错误与日志约定

### 错误码
- 前置失败：`LLM_MULTIMODAL_NOT_SUPPORTED`
- 记录级失败：`MEDIA_UNSUPPORTED_FORMAT` / `MEDIA_DECODE_FAILED` / `MEDIA_SCHEMA_INVALID` / `MEDIA_EXTRACTION_FAILED`

### 日志字段（每文件/页）
- `runId`
- `sourceFile`
- `page`
- `provider`
- `model`
- `status` (`success` / `failed`)
- `durationMs`
- `errorCode` (失败时)

约束：禁止记录 `apiKey`、原始二进制内容和长文本敏感片段。

---

## 测试设计

### Unit
1. 附件列表归一化（文件/目录/非法路径）。
2. 递归发现与排序稳定性。
3. schemaSource 判断与 strict 校验分支。

### Integration
1. 混合图片+PDF 批处理后结构化结果字段完整且顺序稳定。
2. `persistArtifact=true` 时产物文件存在且结构正确。
3. strict 模式下无效记录不会进入 `records`。
4. 局部失败时批处理继续，`stats/errors` 正确。

### E2E
1. 拖拽目录 -> 聊天发送后由 LLM 决策调用 `fs.read/media.extract`（不默认自动批提取）。
2. 显式批处理请求时返回结构化结果；可选持久化路径正确。
3. 切换不支持模型 -> 返回 `LLM_MULTIMODAL_NOT_SUPPORTED` 且不发起提取调用。
4. 日志审计字段齐全且无敏感信息泄露。

---

## 交付边界

- MDE-1.3 完成后，Epic MDE-1 的核心能力闭环达成：
  - 模型能力守卫（MDE-1.1）
  - first-class `media.extract`（MDE-1.2）
  - 批处理编排 + 产物输出 + 可观测（MDE-1.3）
- 后续扩展聚焦 OFD 与大规模并发优化。
