# PRD: Runtime Multimodal Data Extraction

> **Parent Document**: [Product Requirements Document (CrewAgent)](prd.md)
> **Traceability**: FR-MULTI-01 / FR-MULTI-02 / FR-MULTI-03 / FR-MULTI-04 in `prd.md`

## 1. 概述

本 PRD 定义 Runtime 的通用多模态数据提取能力，重点支持：

1. 通过聊天面板拖拽文件/文件夹导入图片与固定版式文档；
2. 按用户意图提取结构化数据并输出 JSON；
3. 支持 `schema`（JSON Schema）由用户提供或由 LLM 按意图生成。

本能力优先通过 Runtime 工具层（`media.extract`）实现，并与后续全局多模态协议改造保持兼容。

---

## 2. 用户故事 (User Stories)

- 作为业务用户，我希望从一个目录中的图片/PDF文档批量提取关键信息，减少人工录入。
- 作为审核人员，我希望提取结果能回溯到源文件和页码，便于复核与审计。
- 作为应用开发者，我希望在 Runtime 中以统一工具接口完成多模态抽取，而不依赖场景特定脚本。
- 作为 Runtime 使用者，我希望 LLM 能直接调用 `media.extract` 处理二进制文档，不必先走 `fs.read`。

---

## 3. 问题定义 (Problem Definition)

当前 Runtime 以文本上下文为核心，面对图片/固定版式文档时缺少统一的数据提取能力，导致：

1. 文档信息抽取依赖手工 OCR 或临时脚本；
2. 多模态能力未作为一等工具路径暴露给 LLM；
3. 抽取输出格式不稳定，难以纳入自动化流程。

需要提供标准化、可追踪、可校验的多模态抽取能力。

---

## 4. 功能范围 (Scope)

### In Scope

- 支持文件类型（MVP）：`.pdf`, `.png`, `.jpg`, `.jpeg`, `.webp`。
- 支持输入入口：聊天面板拖拽单文件或文件夹。
- 支持混合输入（同一目录中多种格式并存）。
- 支持输出结果：
  - `extracted-data.json`：结构化提取数据。
- 支持通过 JSON Schema 约束输出结构。

### Out of Scope (MVP)

- 复杂版式的高保真重排版编辑。
- 针对某一行业票据的专用规则引擎。
- 云端文档知识库与跨项目数据索引。

---

## 5. 验收标准 (Acceptance Criteria)

### AC-1: 文件发现与排序

- 系统支持从聊天面板拖拽导入文件或文件夹。
- 文件夹导入后可递归发现图片与 PDF 文件。
- 默认按文件名升序处理（可扩展排序策略）。
- 输出中保留源文件相对路径，保证可追溯。

### AC-2: 结构化输出文件

- 生成 `extracted-data.json` 作为标准输出物。
- JSON 结果按处理顺序稳定输出，便于下游系统消费。

### AC-3: Schema 驱动结构化提取

- `media.extract` 支持传入 `schema`（JSON Schema）。
- 当未提供 `schema` 时，系统可根据用户意图由 LLM 生成 schema 草案并用于提取。
- 输出 `extracted-data.json` 必须满足 schema 校验（strict 模式）。
- 每条记录至少包含：
  - `sourceFile`
  - `page`（图片默认 1）
  - `data`（业务字段对象）

### AC-4: 多模态能力路径

- 提供 `media.extract` 工具，支持图片/PDF 的多模态理解。
- `media.extract` 必须作为 function call 直接暴露给 LLM（first-class tool）。
- 对图片/PDF 等二进制文档，LLM 应优先直接调用 `media.extract`，无需先调用 `fs.read`。
- `media.extract` 支持 `instruction + schema` 的联合约束。
- 当模型不支持多模态输入时，返回明确错误码（如 `LLM_MULTIMODAL_NOT_SUPPORTED`）。

### AC-5: 多模态 LLM 配置入口

- Runtime 设置页必须提供多模态 LLM 配置区域（provider/baseUrl/model/apiKey/timeout）。
- 运行前执行能力校验：若当前配置模型不支持多模态，需在调用前提示或返回明确错误码。
- 允许用户切换“文本模型”和“多模态模型”配置（可同 provider，不强制同 model）。

### AC-6: `fs.read` 兼容 fallback

- `fs.read` 不作为二进制文档主路径，仅用于文本读取与兼容场景。
- 若误对图片/PDF调用 `fs.read`，不得返回乱码文本。
- 可返回结构化提示对象（含 `mime`、`path`、建议下一步调用 `media.extract`）。

### AC-7: 通用场景验证

给定目录 `@project/docs-multimodal/`，包含：

- 5 张图片文件（png/jpg/webp）；
- 3 个 PDF 文件（共 6 页）；

系统执行后必须生成：

- `@project/artifacts/extracted-data.json`

并满足：

- JSON 记录数等于所有可解析页面数；
- 每条记录都有 `sourceFile`、`page`、`data` 字段；
- 结构化输出通过 schema 校验（strict 模式）。

---

## 6. 技术设计 (Technical Design)

### 6.1 工具接口：`media.extract`

建议参数：

```ts
interface MediaExtractArgs {
  path: string;                 // @project/... alias path
  instruction: string;          // 提取目标说明
  schema?: object;              // JSON Schema（可选）
  schemaIntent?: string;        // 当 schema 缺失时用于生成 schema 的意图描述
  page?: number;                // PDF 可选页码
  strict?: boolean;             // 严格 schema 校验
  documentTypeHint?: string;    // 可选提示，如 "report", "receipt", "form"
}
```

建议返回：

```ts
interface MediaExtractResult {
  ok: boolean;
  mime?: string;
  path?: string;
  page?: number;
  data?: Record<string, unknown>;
  schemaUsed?: Record<string, unknown>;
  schemaSource?: 'provided' | 'generated';
  confidence?: number;
  warnings?: string[];
  error?: { code: string; message: string; details?: unknown };
}
```

### 6.2 处理流水线

1. 聊天面板拖拽输入文件/文件夹，Runtime 建立候选文件列表；
2. LLM 直接调用 `media.extract` 作为主处理路径；
3. 按文件类型分支处理：
   - 图片：直接 `media.extract`；
   - PDF：逐页渲染后 `media.extract`；
4. 若未提供 schema，则根据 `instruction/schemaIntent` 由 LLM 生成 schema；
5. 聚合结构化结果，写入 `extracted-data.json`。

### 6.3 多模态模型配置设计

- 在 Runtime Settings 中新增 `Multimodal LLM` 配置项：
  - `provider`
  - `baseUrl`
  - `model`
  - `apiKey`
  - `timeout`
- 支持 capability 探测或白名单校验，避免误用纯文本模型执行 `media.extract`。
- 当多模态配置缺失或不支持时，统一返回 `LLM_MULTIMODAL_NOT_SUPPORTED`。

### 6.4 OFD 扩展（Phase 2）

- 支持 `.ofd` 输入（先转 PDF/图片再抽取）。
- OFD 不进入 MVP 强制范围，但接口设计需预留格式扩展能力。

### 6.5 工具调用策略

- 默认策略：
  - 二进制/固定版式文档（图片、PDF、OFD） -> 直接 `media.extract`
  - 纯文本文件 -> `fs.read`
- `fs.read` 在多模态场景仅作为 fallback，不应成为主流程依赖。
- 系统提示词与工具文档需明确该策略，降低模型误用概率。

---

## 7. 非功能需求 (Non-Functional Requirements)

- **NFR-MULTI-01 可追溯性**：每条记录必须回链到源文件和页码。
- **NFR-MULTI-02 稳定性**：单文件失败不应中断整批处理。
- **NFR-MULTI-03 可观测性**：日志记录每个文件/页的状态、耗时、错误码。
- **NFR-MULTI-04 安全性**：遵守 `@project/@pkg/@state` 沙箱边界。

---

## 8. 验证计划 (Validation Plan)

### 8.1 数据集

- 使用真实或脱敏样本，覆盖不同版式与清晰度。
- 包含清晰、轻度模糊、遮挡、旋转页面等情况。

### 8.2 指标

- Schema 合规率：>= 98%
- 页面抽取成功率：>= 95%
- 单页失败率：<= 5%

### 8.3 回归用例

- 图片与 PDF 混合目录；
- 聊天面板拖入单文件/多文件/文件夹；
- 非结构化内容图片（输出 warning）；
- 空目录与损坏文件；
- 模型不支持多模态时的报错与降级路径。

---

## 9. 路线图 (Roadmap)

| 阶段 | 范围 | 目标 |
| :--- | :--- | :--- |
| Phase 1 | `media.extract` + 拖拽输入 + JSON输出 | 跑通 `extracted-data.json` |
| Phase 2 | 全局多模态消息协议改造 + OFD 支持 | 聊天主链路原生支持图文/固定版式上下文 |
| Phase 3 | 领域模板化抽取 | 可配置模板与质量评估体系 |
