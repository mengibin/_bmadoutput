# Design: First-Class `media.extract` + Binary Fallback Policy

**Story:** `mde-1-2-first-class-media-extract-binary-fallback-policy.md`  
**设计原则:** first-class 工具优先、二进制读安全降级、结果契约稳定

---

## 设计目标

1. 让 LLM 能直接发现并调用 `media.extract`，不依赖 `fs.read` 引导。
2. 对二进制输入的 `fs.read` 统一返回结构化提示，杜绝乱码文本输出。
3. 固化 `media.extract` 的输入/输出契约与错误码，便于 MDE-1.3 批处理复用。
4. 复用 MDE-1.1 能力校验，确保不支持模型时 fail-fast。

---

## 范围与非范围

### In Scope (MDE-1.2)
- `fileSystemToolHost` 工具可见列表增加 first-class `media.extract`。
- `media.extract` 参数解析与基础校验：
  - `path` (required)
  - `mode?` (`extract` | `read`, default `extract`)
  - `instruction` (`mode=extract` required, `mode=read` optional)
  - `schema?`
  - `schemaIntent?`
  - `page?`
  - `strict?`
  - `documentTypeHint?`
- `media.extract` 执行前接入 MDE-1.1 capability guard。
- `fs.read` 检测二进制/图片/PDF 时返回结构化 fallback 提示。
- 兼容 OpenAI-compatible 工具名编码差异（`media.extract` ↔ `media-extract`）并避免路由丢失。
- PDF 渲染缺失依赖时返回可诊断错误（缺失模块名 + pip 包建议），不再仅返回笼统失败消息。
- 单测覆盖工具注册、guard 分支、binary fallback 分支。

### Out of Scope (MDE-1.2)
- 拖拽文件夹递归 ingestion 与批量调度（MDE-1.3）。
- schema 意图生成优化与批量输出聚合（MDE-1.3）。
- OFD 解码或转换流水线（Phase-2）。

---

## 关键决策

### 决策 1：`media.extract` 在工具注册层直接暴露
- 原因：减少 LLM 通过 `fs.read` 误读二进制的路径依赖。
- 结果：`getVisibleTools()` 始终可返回 `media.extract`（受现有 tool policy 约束）。

### 决策 2：`fs.read` 对二进制 fail-safe，不返回正文
- 原因：二进制强行按文本读取会产生乱码，误导 LLM。
- 结果：返回结构化提示对象，明确建议改用 `media.extract`。

### 决策 3：`media.extract` 结果使用统一 envelope
- 原因：减少后续批处理与 UI 侧分支复杂度。
- 结果：成功/失败统一为 `{ ok, ... }` + `{ error: { code, message, details? } }`。

### 决策 4：工具名解码按“本轮工具表映射”而非前缀硬编码
- 原因：仅按 `fs-/ui-/python-` 前缀硬编码解码会漏掉 `media-extract`，导致 `media.extract` 调用失败并误走其他工具。
- 结果：在 LLM 适配层用 `encodedToolName -> originalToolName` 映射统一解码；同时在 ToolHost dispatch 层做已知别名兜底。

---

## 数据契约

### `media.extract` 入参

```ts
interface MediaExtractArgs {
  path: string
  mode?: 'extract' | 'read'
  instruction?: string
  schema?: Record<string, unknown>
  schemaIntent?: string
  page?: number
  strict?: boolean
  documentTypeHint?: string
}
```

### `media.extract` 返回

```ts
type MediaExtractResult =
  | {
      ok: true
      mode?: 'extract'
      mime: string
      path: string
      page?: number
      data: Record<string, unknown>
      schemaUsed?: Record<string, unknown>
      schemaSource?: 'provided' | 'generated'
      confidence?: number
      warnings?: string[]
    }
  | {
      ok: true
      mode: 'read'
      mime: string
      path: string
      content: string
      source: 'pdf_text' | 'vision' | 'mixed'
      pages?: Array<{ page: number; text: string }>
      confidence?: number
      warnings?: string[]
    }
  | {
      ok: false
      error: {
        code:
          | 'LLM_MULTIMODAL_NOT_SUPPORTED'
          | 'MEDIA_UNSUPPORTED_FORMAT'
          | 'MEDIA_DECODE_FAILED'
          | 'MEDIA_SCHEMA_INVALID'
          | 'MEDIA_EXTRACTION_FAILED'
        message: string
        details?: unknown
      }
    }
```

### `fs.read` binary fallback 结构

```ts
interface FsReadBinaryHint {
  ok: false
  error: {
    code: 'FS_READ_BINARY_FALLBACK'
    message: string
    details: {
      path: string
      mime?: string
      suggestedTool: 'media.extract'
      suggestedArgs: { path: string; instruction: string }
    }
  }
}
```

---

## 处理流程

### A. Tool Discovery
1. `getVisibleTools()` 合并系统工具定义。
2. 注入 `media.extract` 的 function schema。
3. LLM 请求前对工具名做 provider 兼容编码；响应时按本轮工具映射回写原始名称（含流式/非流式）。
4. 返回给 LLM 供直接 function-call。

### B. `media.extract` 执行链路
1. 参数 JSON 解析与字段校验。
2. 路径解析与沙箱校验（仅 `@project/@pkg/@state`）。
3. capability guard 预检（MDE-1.1）。
4. `mode=read` 且为 PDF 时优先尝试本地文本层提取（嵌入式 Python `pypdf`）；成功则直接返回全文。
5. PDF 本地文本失败时，使用嵌入式 Python `pypdfium2` 将 PDF 页渲染为 PNG，再走多模态。
6. PDF 多模态请求统一发送 `image_url`，不发送 `input_file`（`application/pdf`）。
7. 若渲染失败且为 `PYTHON_MODULE_MISSING`，返回可诊断错误详情（`module/pipPackage/suggestion`）。
8. 输出标准 envelope（成功/失败统一）。

### C. `fs.read` binary fallback
1. `fs.read` 在读取前进行文件类型判定（扩展名 + 二进制 sniff）。
2. 若为二进制/图片/PDF：
   - 不返回 `content` 文本
   - 返回 `FS_READ_BINARY_FALLBACK` 结构化提示
3. 若为文本：
   - 走既有 `fs.read` 正常路径（含截断策略）

---

## 模块与文件改动（设计级）

| 文件 | 改动类型 | 说明 |
|------|----------|------|
| `crewagent-runtime/electron/services/fileSystemToolHost.ts` | MODIFY | 注册 `media.extract` + dispatch + read/extract 分流 + PDF 文本/渲染降级 + binary fallback |
| `crewagent-runtime/electron/services/fileSystemToolHost.test.ts` | MODIFY | 覆盖 tool visibility / extract / fallback 分支 |
| `crewagent-runtime/electron/services/llmAdapter.ts` | MODIFY | 建立工具名编码/解码映射，修复 `media-extract` 回写为 `media.extract`（流式与非流式） |
| `crewagent-runtime/electron/services/llmAdapter.test.ts` | MODIFY | 新增 `media-extract` 解码回归测试 |
| `crewagent-runtime/electron/services/toolHost.ts` | MODIFY | 如需扩展 tool result 类型定义 |
| `crewagent-runtime/electron/services/pythonService.ts` | REUSE | 提供 bundled Python 路径 (`getBundledPythonPath`) |
| `crewagent-runtime/electron/services/multimodalCapabilityService.ts` | REUSE | 复用 guard，不重复实现 |

---

## 错误与日志约定

### 错误码
- `LLM_MULTIMODAL_NOT_SUPPORTED`
- `MEDIA_UNSUPPORTED_FORMAT`
- `MEDIA_DECODE_FAILED`
- `MEDIA_SCHEMA_INVALID`
- `MEDIA_EXTRACTION_FAILED`
- `FS_READ_BINARY_FALLBACK`

### 日志字段
- `runId`
- `toolName`
- `sourceFile`
- `page`
- `provider`
- `model`
- `status`
- `errorCode`
- `durationMs`

约束：不得记录 `apiKey` 或二进制原文。

---

## 测试设计

### Unit
1. `getVisibleTools()` 包含 `media.extract` 及完整参数 schema。
2. `media.extract` 缺少必须字段时返回参数错误。
3. unsupported model 返回 `LLM_MULTIMODAL_NOT_SUPPORTED`。
4. `fs.read` 对图片/PDF 返回 `FS_READ_BINARY_FALLBACK` 提示。
5. `fs.read` 文本文件路径行为保持不变。
6. `media-extract`/`fs-read` 等 provider 兼容名字能够正确回写并路由到点号工具名。
7. PDF 渲染缺失 `pypdfium2` 时，`MEDIA_DECODE_FAILED` details 包含 `module/pipPackage/suggestion`。

### Integration
1. 使用支持模型调用 `media.extract`，返回标准 envelope。
2. 使用不支持模型调用 `media.extract`，确认无 provider outbound 请求。

### Regression
1. 历史 `fs.read` 文本读取测试全通过。
2. 非多模态工具（如 `fs.list`/`fs.search`）行为不变。

---

## 交付边界（与 MDE-1.3 对齐）

- MDE-1.2 交付“工具直连能力 + fallback 策略 + 契约稳定化”。
- MDE-1.3 在此基础上增加“拖拽批处理 + schema 编排 + 聚合输出”。
