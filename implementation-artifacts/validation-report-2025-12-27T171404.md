# Validation Report

**Document:** `_bmad-output/implementation-artifacts/3-17-validate-v1-1-export-with-schemas.md`  
**Checklist:** `_bmad/bmm/workflows/4-implementation/create-story/checklist.md`  
**Date:** `2025-12-27T17:14:04+0800`

## Summary

- Overall: 6/12 passed (50%)
- Critical Issues: 0

## Section Results

### Story Format & Completeness

Pass Rate: 5/7 (71%)

✓ PASS - Clear story title + Status present  
Evidence: Header + status (`3-17-validate-v1-1-export-with-schemas.md:1-5`)

✓ PASS - Story statement includes role/action/benefit  
Evidence: “As a Creator… so that…” (`3-17-validate-v1-1-export-with-schemas.md:9-11`)

✓ PASS - Acceptance criteria are explicit and verifiable  
Evidence: Validates 5 artifact types + blocks download on errors (`3-17-validate-v1-1-export-with-schemas.md:15-23`)

✓ PASS - References include spec + official schema source  
Evidence: Epics + schemas path + tech spec referenced (`3-17-validate-v1-1-export-with-schemas.md:54-58`)

✓ PASS - Tasks map to AC with clear breakdown  
Evidence: Tasks cover frontmatter parse + schema validation + error formatting + UI blocking (`3-17-validate-v1-1-export-with-schemas.md:47-52`)

⚠ PARTIAL - Design notes “Ajv 已引入” but does not anchor implementation to existing Builder v1.1 validators/helpers  
Evidence: Mentions Ajv but no reuse pointer (`3-17-validate-v1-1-export-with-schemas.md:29-34`) vs existing Ajv2020 patterns (`crewagent-builder-frontend/src/lib/agents-manifest-v11.ts:1-96`, `crewagent-builder-frontend/src/app/editor/[projectId]/[workflowId]/page.tsx:7-15`)  
Impact: Medium — risk of duplicated Ajv setup + inconsistent error formatting across Builder pages.

⚠ PARTIAL - Design lacks a concrete UX flow definition for “block download + show errors” (placement, grouping, copy/export, retry)  
Evidence: Design has only Summary/Categories/Test Plan; no UI flow (`3-17-validate-v1-1-export-with-schemas.md:25-45`)  
Impact: Low — acceptable for `ready-for-design`, but should be tightened in `design-story`.

### Technical & Disaster-Prevention Coverage

Pass Rate: 1/5 (20%)

⚠ PARTIAL - Missing explicit JSON Schema draft + Ajv2020 requirement (draft 2020-12 schemas)  
Evidence: Schemas declare draft 2020-12 (`crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/bmad.schema.json:2`) and Builder already uses Ajv2020 (`crewagent-builder-frontend/src/lib/agents-manifest-v11.ts:1-4`) but story only says “Ajv” (`3-17-validate-v1-1-export-with-schemas.md:29`)  
Impact: Medium — using the wrong Ajv class/options can produce false validation results.

⚠ PARTIAL - Does not call out strict schema failure modes (`additionalProperties:false`, `minItems`, required fields) for better UX/error messages  
Evidence: `additionalProperties:false` is common in v1.1 JSON files (`bmad.schema.json:31-32,79`; `workflow-graph.schema.json:28,63-85`; `agents.schema.json:12-18`) but story has no guidance beyond “show schema pointer” (`3-17-validate-v1-1-export-with-schemas.md:31-34`)  
Impact: Medium — errors will be hard to act on without tailored “what to change” hints.

⚠ PARTIAL - Schema import/source-of-truth strategy for frontend is underspecified (runtime path vs frontend copies)  
Evidence: AC points to runtime schemas directory (`3-17-validate-v1-1-export-with-schemas.md:16-22`) while frontend currently keeps v1.1 schemas under `crewagent-builder-frontend/src/lib/bmad-spec/v1.1/` and imports from there (`crewagent-builder-frontend/src/lib/agents-manifest-v11.ts:4`, `crewagent-builder-frontend/src/app/editor/[projectId]/[workflowId]/page.tsx:11`)  
Impact: Medium — risk of build-time path issues or schema drift across repos.

✓ PASS - Error output requirements are actionable (file path + schemaPath + instancePath + message)  
Evidence: AC requires actionable errors (`3-17-validate-v1-1-export-with-schemas.md:23`) and Design specifies schemaPath + instancePath (`3-17-validate-v1-1-export-with-schemas.md:31-34`)

⚠ PARTIAL - Test plan is too coarse; missing concrete negative/edge cases (YAML parse failures, missing frontmatter, additionalProperties, minItems, invalid ids)  
Evidence: Only 1 negative + 1 positive scenario (`3-17-validate-v1-1-export-with-schemas.md:42-45`)  
Impact: Medium — easy to ship a validator that works only for “happy path” data.

## Failed Items

None.

## Partial Items

1. 未将实现锚定到现有 Builder v1.1 Ajv/validator 代码，可能导致重复实现与 UX 不一致。  
2. 未明确 JSON Schema draft 2020-12 与 Ajv2020 使用要求。  
3. 未明确前端 schemas 的同步/来源策略（runtime 目录 vs `src/lib/bmad-spec/v1.1/`）。  
4. 未显式强调 `additionalProperties:false` / `minItems` 等强约束与典型错误提示策略。  
5. Test plan 缺少关键负例/边界覆盖。

## Recommendations

1. Must Fix: 在 Design/Tasks 中明确使用 `ajv/dist/2020`（Ajv2020）并复用现有错误格式化风格；同时明确 schemas 在前端的落点与同步方式（建议统一放在 `crewagent-builder-frontend/src/lib/bmad-spec/v1.1/`）。  
2. Should Improve: 明确 UI 交互：导出按钮点击后先 validate→失败则阻断下载并按文件分组展示（支持复制/展开更多错误详情）。  
3. Consider: 把 “导出前校验” 抽成一个 `validateExportV11(payload)` 的 lib 模块，供预览/导出/后续更多校验复用。

