# Story 14.4: SDK API Usage Planning with Source References

Status: done

<!-- Created after Story 14.1/14.2/14.3 reached done. Story 14.4 follows the Epic 14 strict sequence and adds source-grounded API planning before risk-gated SDK execution. -->

## Story

作为 **Runtime User**，
我希望 Runtime 能基于已导入 SDK Wiki 为 SDK API 使用生成小步、可追溯的计划草案，
以便主 LLM 在回答或执行 SDK 任务前不会编造不存在的 API。

## Acceptance Criteria

### AC-1: Internal planning tool

**Given** Runtime 已安装至少一个 SDK Wiki Pack
**When** agent 调用 `sdk_wiki.plan_api_usage` 并提供 `sdkId`、`intent`、可选 `modelState` 和 `constraints`
**Then** Runtime 必须返回结构化 planning scaffold
**And** 结果至少包含 `taskType`、`requiredApis`、`executionPlan`、`missingInformation`、`confidence`。

### AC-2: Source-referenced API recommendations

**Given** SDK Wiki 中存在与 intent 匹配的 API、workflow 或 relation 页面
**When** `sdk_wiki.plan_api_usage` 组织计划
**Then** `requiredApis` 中每个推荐 API 必须引用已存在 page
**And** 每个推荐 API 必须携带 `sourceRefs`，如果缺少 source refs 则必须在 `missingInformation` 中明确报告。

### AC-3: Source-referenced plan steps

**Given** Runtime 生成 `executionPlan`
**When** plan step 引用 API 或 workflow page
**Then** 每个 step 必须包含 `sourceRefs` 或对应的 `missingInformation`
**And** step 不得引用 SDK Wiki 中不存在的 API 名称。

### AC-4: Missing knowledge instead of invention

**Given** SDK Wiki 中缺少相关 API、symbol 或 workflow 页面
**When** `sdk_wiki.plan_api_usage` 无法定位足够知识
**Then** Runtime 必须返回空的或降级的 `requiredApis` / `executionPlan`
**And** 必须返回 `missingInformation`
**And** 不得发明 API 名称、参数或执行步骤。

### AC-5: Runtime LLM boundary

**Given** 后续可能存在 SDK Bridge、SAM MCP Server 或外部执行服务
**When** 需要 SDK 语义理解或任务规划
**Then** LLM 推理仍必须发生在 CrewAgent Runtime 主 LLM loop
**And** `SdkWikiService` 只做确定性检索、关系展开和结构化草案生成，不调用 LLM。

### AC-6: ToolHost and prompt availability

**Given** SDK Wiki 工具已配置
**When** Runtime 构造可见工具和 SDK Wiki registry system message
**Then** `sdk_wiki.plan_api_usage` 必须作为内部只读工具暴露给 agent
**And** registry guidance 必须提示 agent 在 SDK API claims 或 SDK 操作前使用规划/检索工具。

### AC-7: Tests

**Given** 14.4 完成开发
**When** 运行目标测试
**Then** 必须覆盖：

- intent 命中 API/relations 时返回 source-referenced plan；
- 缺少匹配页时返回 `missingInformation` 且不发明 API；
- ToolHost 暴露并归一化 `sdk_wiki.plan_api_usage` 参数；
- TypeScript 编译通过。

## Design

### Summary

- 新增内部工具 `sdk_wiki.plan_api_usage`。
- 工具输入是 `sdkId`、可选 `version`、自然语言 `intent`、可选 `modelState` 和 `constraints`。
- `SdkWikiService` 使用 `SdkWikiIndexReader` 的 search/read/relation 能力做确定性 planning scaffold。
- 计划不是最终业务脚本；它是给主 LLM 的 grounding scaffold，主 LLM 仍负责结合用户意图、上下文和读取到的页面内容生成最终回答或代码。
- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-14-4-sdk-api-usage-planning-with-source-references.md`

### Design Decisions

- `plan_api_usage` 要求 `sdkId`，`version` 可选；若同一 SDK 多版本已安装且未传 version，沿用现有 `SDK_WIKI_VERSION_REQUIRED` 语义。
- `intent` 是唯一必需 query。`modelState` 在 14.4 仅透传/保留为未来扩展输入，不做 SDK-specific 推理。
- `constraints.requireSourceRefs !== false` 时，缺 source refs 的推荐项必须产生 missingInformation。
- `constraints.preferSmallSteps` 默认为 true，计划 step 优先按 API/ref 拆成小步。
- 检索策略以已有 index 为准：搜索 intent 及派生 token，收集 workflow/API/relation/concept 页面，再对 API 页面做 relation expansion。
- `requiredApis` 只允许来自已存在 API page；workflow/concept/relation 页面可以作为 evidence，不伪装成 API。
- 置信度是 deterministic heuristic，不代表 LLM 判断：无 API 为 0，有 API 但缺 source refs 降低，有 workflow/relations 支撑提高。

### Result Contract

```ts
type SdkWikiPlanApiUsageResult =
  | {
      ok: true
      mode: 'plan_api_usage'
      sdkId: string
      version: string
      intent: string
      taskType: string
      requiredApis: SdkWikiPlannedApi[]
      executionPlan: SdkWikiPlanStep[]
      missingInformation: SdkWikiMissingInformation[]
      confidence: number
    }
  | { ok: false; error: SdkWikiToolError }
```

### Out of Scope

- 不调用 LLM；
- 不生成最终可执行 SDK 脚本；
- 不执行 SDK/MCP/SAM 操作；
- 不实现 SDK tool risk confirmation gate（Story 14.5）；
- 不 hardcode SAM API 或 SAM workflow。

## Tasks / Subtasks

- [x] Task 1: Shared contracts and tool surface (AC: 1,6)
  - [x] 1.1 Add `sdk_wiki.plan_api_usage` to `SdkWikiToolName`
  - [x] 1.2 Add payload/result shared types
  - [x] 1.3 Add ToolHost visible tool schema and argument normalization
  - [x] 1.4 Add main-process dispatch to `SdkWikiService.planApiUsage`

- [x] Task 2: Deterministic planning service (AC: 1,2,3,4,5)
  - [x] 2.1 Add `SdkWikiService.planApiUsage`
  - [x] 2.2 Build intent query variants without hardcoding SDK names
  - [x] 2.3 Collect matching pages and relation-expanded API pages
  - [x] 2.4 Build required API refs only from existing pages
  - [x] 2.5 Build source-referenced execution plan steps
  - [x] 2.6 Return missingInformation instead of invented APIs

- [x] Task 3: Prompt guidance (AC: 6)
  - [x] 3.1 Update SDK Wiki registry system message to mention `plan_api_usage`

- [x] Task 4: Tests and verification (AC: 7)
  - [x] 4.1 Add service tests for successful source-referenced planning
  - [x] 4.2 Add service tests for missing knowledge
  - [x] 4.3 Update ToolHost tests for visible schema and normalized args
  - [x] 4.4 Run focused tests, lint, and build/TypeScript verification
  - [x] 4.5 Add relation-only planning regression coverage from code review M1

## Dev Notes

### Codebase Findings

- `SdkWikiService` already has read-only query entrypoints and error normalization.
- `SdkWikiIndexReader` already loads pages/symbols/terms/relations and returns `SdkWikiPageRef` with `sourceRefs`.
- `FileSystemToolHost` already centralizes SDK Wiki tool schemas, argument validation, and dispatch through `onSdkWikiToolCall`.
- `main.ts` already dispatches SDK Wiki tool calls to `SdkWikiService`.
- `formatSdkWikiRegistrySystemMessage()` already injects registry guidance into chat/agent/run system messages.

### References

- PRD: `_bmad-output/prd-runtime-sdk-wiki-and-tool-risk.md`
- Architecture: `_bmad-output/architecture/runtime-sdk-wiki-and-tool-risk-architecture.md`
- Epic: `_bmad-output/epics-runtime-sdk-wiki-and-tool-risk.md`
- Development plan: `_bmad-output/implementation-artifacts/epic-14-runtime-sdk-wiki-development-plan.md`
- Requirements: `_bmad-output/project-planning-artifacts/runtime-sdk-wiki-generic-requirements/02-sdk-llm-wiki-module.md`
- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-14-4-sdk-api-usage-planning-with-source-references.md`

## Dev Agent Record

### Agent Model Used

GPT-5 Codex

### Debug Log References

- `npm test -- electron/services/sdkWikiService.test.ts` passed: 1 test file, 38 tests.
- `npm test -- electron/services/fileSystemToolHost.test.ts -t "SDK Wiki"` passed: 2 tests, 91 skipped.
- Full `npm test -- electron/services/fileSystemToolHost.test.ts` still has existing unrelated terminal/node/shell failures; SDK Wiki focused tests pass.
- `npm exec eslint -- shared/sdkWikiTypes.ts electron/services/sdkWikiService.ts electron/services/sdkWikiService.test.ts electron/services/fileSystemToolHost.ts electron/services/fileSystemToolHost.test.ts electron/main.ts --max-warnings 0` passed.
- `npm exec eslint -- electron/services/sdkWikiService.ts electron/services/sdkWikiService.test.ts --max-warnings 0` passed after M1 fix.
- `npm exec tsc -- --noEmit` passed.
- `npm run build:ci` passed; remaining warnings are existing Vite package/chunk warnings.
- `git diff --check -- shared/sdkWikiTypes.ts electron/services/sdkWikiService.ts electron/services/sdkWikiService.test.ts electron/services/fileSystemToolHost.ts electron/services/fileSystemToolHost.test.ts electron/main.ts` passed.
- `git diff --check` passed after M1 fix.

### Completion Notes List

- Added `sdk_wiki.plan_api_usage` shared payload/result contracts and internal tool name.
- Implemented deterministic `SdkWikiService.planApiUsage()` with intent query variants, workflow API hints, relation expansion, source refs, missingInformation, and heuristic confidence.
- Added ToolHost visible schema and argument normalization for `plan_api_usage`.
- Added main-process dispatch and updated SDK Wiki registry prompt guidance.
- Added service coverage for source-referenced pressure planning and missing-knowledge behavior.
- Fixed review M1 by allowing relation page `apis` frontmatter to contribute API candidates during planning.
- Added relation-only planning regression coverage proving `wiki/relations/*.md` evidence can produce source-referenced `requiredApis`.
- Updated ToolHost coverage for `plan_api_usage` visibility, normalization, and invalid args.

### File List

- `_bmad-output/implementation-artifacts/14-4-sdk-api-usage-planning-with-source-references.md`
- `_bmad-output/implementation-artifacts/tech-spec-14-4-sdk-api-usage-planning-with-source-references.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `crewagent-runtime/shared/sdkWikiTypes.ts`
- `crewagent-runtime/electron/services/sdkWikiService.ts`
- `crewagent-runtime/electron/services/sdkWikiService.test.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.test.ts`
- `crewagent-runtime/electron/main.ts`

### Change Log

- 2026-05-14 - Created and designed Story 14.4 for SDK API usage planning with source references; status set to ready-for-dev.
- 2026-05-14 - Implemented `sdk_wiki.plan_api_usage`; status moved to review.
- 2026-05-14 - Fixed code review M1 for relation-backed planning; status moved to done.
