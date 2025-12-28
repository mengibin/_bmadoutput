# Story 3.17: Validate v1.1 `.bmad` Export with Schemas and Actionable Errors

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **Creator**,  
I want the Builder to validate my v1.1 `.bmad` export against the official schemas and show actionable errors,  
so that I can fix issues before downloading/importing the package.

## Acceptance Criteria

1. **Given** I click "Export Package (v1.1)" (ProjectBuilder)  
   **When** the Builder validates the in-memory export payload (the files that will be written into the ZIP)  
   **Then** it validates JSON files with v1.1 schemas (draft 2020-12):  
   - `bmad.json` → `bmad.schema.json`  
   - `agents.json` → `agents.schema.json`  
   - for each workflow in `bmad.json.workflows[]`: `workflows/<workflowId>/workflow.graph.json` → `workflow-graph.schema.json`

2. **Given** the export contains Markdown files  
   **When** the Builder validates frontmatter  
   **Then** it extracts YAML frontmatter (`--- ... ---`) and validates only the frontmatter JSON (not the Markdown body):  
   - `workflows/<workflowId>/workflow.md` frontmatter（YAML→JSON）→ `workflow-frontmatter.schema.json`  
   - `workflows/<workflowId>/steps/*.md` frontmatter（YAML→JSON）→ `step-frontmatter.schema.json`

3. **Given** schema or frontmatter validation fails  
   **When** I attempt export  
   **Then** the download is blocked and Builder shows actionable errors grouped by file path, including:  
   - `filePath` (zip path)  
   - schema pointer (`schemaPath`) + instance pointer (`instancePath`)  
   - a human-friendly message (Chinese preferred; keep original if needed)

4. **Given** schema/frontmatter validation passes  
   **When** I export  
   **Then** the download proceeds (Story 3.16)

## Design

### Summary

- Tech Spec: `_bmad-output/tech-spec.md`（Package Spec v1.1）
- 使用 `ajv/dist/2020`（Ajv2020；draft 2020-12）加载 v1.1 schemas，并在导出前对“即将写入 ZIP 的内容”执行校验。
- schemas 来源：以 `crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/` 为唯一来源；前端通过 `npm run sync:bmad-spec` 同步到 `crewagent-builder-frontend/src/lib/bmad-spec/v1.1/`（避免跨包运行时读取 runtime 目录）。
- frontmatter 校验：从 Markdown 中提取 YAML frontmatter（仅校验 frontmatter，不校验正文），YAML→JSON 后再用 Ajv 校验。
- 错误展示：按 `filePath` 分组；每条至少包含 `schemaPath` + `instancePath` + message，并对常见 schema 失败提供可操作提示（见下文）。

### UX / UI

- Export 按钮点击后流程：
  1) 先跑 Story 3.16 的“基础一致性校验”（缺文件/空图/环/非法 id/`node.file` 指向不存在等）  
  2) 再跑本 story 的 schema/frontmatter 校验  
  3) 若失败：阻断下载，并展示错误面板（按文件分组，支持复制错误文本、展开更多详情、重试）

### API / Contracts

- 不新增后端接口；复用 Story 3.16 导出所需的读取接口与数据结构。
- 校验的“输入”是导出前在内存中组装的 `filesByPath`（zip 内路径 → 文件内容字符串）。

### Data / Storage

- N/A（导出校验为派生操作；不新增 DB 字段；不落盘；失败仅影响当次导出流程）

### Implementation Notes

- Ajv 复用：沿用现有 Builder 的 Ajv2020 初始化与错误格式化风格，避免重复实现与 UX 不一致：
  - `crewagent-builder-frontend/src/lib/agents-manifest-v11.ts`（Ajv2020 + 错误列表格式化）
  - `crewagent-builder-frontend/src/app/editor/[projectId]/[workflowId]/page.tsx`（`formatAjvErrors`）
- schemas（前端本地副本）应包含全部 v1.1 校验所需文件，并由 `sync:bmad-spec` 脚本同步：
  - `bmad.schema.json`
  - `agents.schema.json`
  - `workflow-graph.schema.json`
  - `workflow-frontmatter.schema.json`
  - `step-frontmatter.schema.json`
- 建议抽一个纯函数模块，供导出（3.16）与预览/调试复用：
  - `validateExportBundleV11({ filesByPath }): { ok, errors, warnings }`
  - `filesByPath` 的 key 使用 zip 内路径（例如 `workflows/1/workflow.md`），这样错误可以直接定位到最终 ZIP 路径。
- frontmatter 解析建议使用通用 YAML 解析库（例如 `yaml`）而不是手写解析器，避免数组/嵌套结构解析不全导致“假通过/假失败”。
- 关键输入路径（与 Story 3.16 产物一致）：
  - `bmad.json`
  - `agents.json`
  - `workflows/<workflowId>/workflow.graph.json`
  - `workflows/<workflowId>/workflow.md`
  - `workflows/<workflowId>/steps/<nodeId>.md`
- 建议新增/调整的前端模块（命名可微调，但需保持“导出/校验”职责清晰）：
  - `crewagent-builder-frontend/src/lib/export/validate-export-bundle-v11.ts`（纯函数；输入 `filesByPath`）
  - `crewagent-builder-frontend/src/lib/export/frontmatter.ts`（frontmatter 提取 + YAML parse；返回 `data | error`）
  - `crewagent-builder-frontend/src/lib/export/ajv-validators-v11.ts`（Ajv2020 validator cache：bmad/agents/graph/workflow-frontmatter/step-frontmatter）
  - UI 入口：`crewagent-builder-frontend/src/app/builder/[projectId]/page.tsx`（Export 按钮流程里调用校验）
- Ajv errors → UI 的结构化输出建议：
  - `filePath`（zip path）
  - `kind`: `schema` | `frontmatter`
  - `instancePath` / `schemaPath`（Ajv）
  - `message`（面向用户）
  - `hint?`（可选：基于 keyword 的增强提示）

### Error Categories

- Schema 校验失败（Ajv errors）
- frontmatter 解析失败（无 frontmatter / YAML 语法错误）
- 一致性校验失败（复用 Story 3.16 的校验结果；统一在同一错误面板呈现）

### Actionable Error Hints (Examples)

对常见 Ajv errors 做“提示增强”（仍保留原始 `schemaPath/instancePath`）：

- `additionalProperties`: 指出多余字段名（例如“存在未允许字段: foo，请删除或改为 schema 支持字段”）
- `required`: 指出缺失字段名（例如“缺少必填字段: metadata.icon”）
- `minItems`: 指出最小数量（例如“agents 不能为空（minItems=1）”）
- `pattern`: 指出不合法格式（例如 `workflowId/nodeId/agentId` pattern 不匹配）

### Errors / Edge Cases

- schema 文件不同步（前端缺少最新 schema）：
  - 开发态：运行 `npm run sync:bmad-spec` 同步 schemas
  - 运行态：错误提示应指向“请更新 Builder 版本/重新构建”，避免让用户去手动拷贝 schema 文件
- frontmatter 缺失：错误需明确是“缺少 `---` frontmatter”还是“frontmatter 解析失败”

### Test Plan

- 正例：
  - 正常项目导出（multi-workflow）→ schema/frontmatter 全部通过 → 下载成功
- 负例（建议覆盖以下至少 1 个用例/类目）：
  - `bmad.json`：缺少 `entry/workflows` 必填字段、`version` 非 semver、存在额外字段触发 `additionalProperties:false`
  - `agents.json`：`agents=[]`（`minItems=1`）、agent 缺少 `metadata.icon`、存在多余字段
  - `workflow.graph.json`：`nodes=[]`、`node.file` 指向不存在文件、edge `label` 为空、nodeId pattern 不合法
  - `workflow.md`：无 frontmatter、YAML 语法错误、缺少 `workflowType/currentNodeId/stepsCompleted`
  - `steps/*.md`：无 frontmatter、缺少 `nodeId/type`、`type` 非枚举值、`transitions` item 缺字段
- Unit：将 `validateExportBundleV11` 输入为一份“已知正确”的 files map（可来自 `crewagent-runtime/spec/bmad-package-spec/v1.1/examples/`），断言 `ok=true`；再对每类负例断言错误可定位且可读。

## Tasks / Subtasks

- [x] 1) 补齐前端 v1.1 schemas：更新 `crewagent-builder-frontend/scripts/sync-bmad-spec.mjs` 并同步缺失 schema 文件
- [x] 2) 引入 frontmatter YAML 解析能力（`yaml`）并实现 `extract + parse YAML → JSON`（仅用于导出校验路径）
- [x] 3) 实现 `validateExportBundleV11`：Ajv2020 校验 `bmad.json/agents.json/workflow.graph.json` + frontmatter 校验
- [x] 4) 错误格式化：按文件分组 + schemaPath/instancePath + 常见错误提示增强
- [x] 5) UI：导出前校验、阻断下载、错误面板展示（与 Story 3.16 的导出按钮/错误展示联动）
- [x] 6) 单测：已知正确示例通过 + 关键负例覆盖

## References

- Tech Spec: `_bmad-output/tech-spec.md`
- `_bmad-output/epics.md`（Epic 3 / Story 3.17）
- `crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/`（v1.1 JSON Schemas）
- `_bmad-output/implementation-artifacts/3-16-export-v1-1-bmad-zip-bundle.md`（导出结构与路径约定）

## Dev Agent Record

### Agent Model Used

GPT-5.2 (Codex CLI)

### Debug Log References

- Frontend:
  - `npm -C crewagent-builder-frontend test`
  - `npm -C crewagent-builder-frontend run lint`
  - `npm -C crewagent-builder-frontend run build`

### Completion Notes List

- 前端补齐 v1.1 schemas 同步：扩展 `sync:bmad-spec` 覆盖 `bmad/workflow-frontmatter/step-frontmatter` 等缺失 schema。
- 新增导出校验模块：Ajv2020（draft 2020-12）+ `ajv-formats`，并用 `yaml` 解析 Markdown frontmatter（只校验 frontmatter）。
- 导出按钮集成校验：导出前对 `filesByPath` 执行 schema/frontmatter 校验；失败则阻断下载并按文件路径分组展示（含 instancePath/schemaPath + 复制全部）。
- 补充单测：覆盖“正常导出通过/缺失 frontmatter/required 缺字段/additionalProperties”。

### File List

- `_bmad-output/implementation-artifacts/3-17-validate-v1-1-export-with-schemas.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `crewagent-builder-frontend/package.json`
- `crewagent-builder-frontend/package-lock.json`
- `crewagent-builder-frontend/scripts/sync-bmad-spec.mjs`
- `crewagent-builder-frontend/src/app/builder/[projectId]/page.tsx`
- `crewagent-builder-frontend/src/lib/bmad-zip-v11.ts`
- `crewagent-builder-frontend/src/lib/export/ajv-validators-v11.ts`
- `crewagent-builder-frontend/src/lib/export/frontmatter.ts`
- `crewagent-builder-frontend/src/lib/export/validate-export-bundle-v11.ts`
- `crewagent-builder-frontend/src/lib/bmad-spec/v1.1/bmad.schema.json`
- `crewagent-builder-frontend/src/lib/bmad-spec/v1.1/step-frontmatter.schema.json`
- `crewagent-builder-frontend/src/lib/bmad-spec/v1.1/workflow-frontmatter.schema.json`
- `crewagent-builder-frontend/tests/validate-export-bundle-v11.test.ts`
