# Tech-Spec: Story 14.4 SDK API Usage Planning with Source References

**Created:** 2026-05-14
**Status:** Implemented; Approved
**Source Story:** `_bmad-output/implementation-artifacts/14-4-sdk-api-usage-planning-with-source-references.md`

## Overview

### Problem Statement

Story 14.1~14.3 已经提供 SDK Wiki Pack 导入、查询、读取、关系展开和 Settings 管理入口。Runtime 现在可以回答单个 API 的问题，但还缺少一个面向任务 intent 的 planning scaffold。没有该 scaffold 时，主 LLM 需要自行串联 search/read/relations，容易漏掉 source refs 或在缺知识时编造 API。

### Solution

新增内部工具 `sdk_wiki.plan_api_usage`。该工具不调用 LLM，只基于已导入 SDK Wiki index、page frontmatter、source refs 和 relations 生成确定性 planning scaffold：

- 根据 `intent` 搜索 workflow/API/concept/relation pages；
- 从匹配 API、workflow/relation `apis` frontmatter 和 relation expansion 中收集 existing API pages；
- 为每个 recommended API 携带 `sourceRefs`；
- 生成小步 `executionPlan`，每步只引用 existing page refs；
- 缺少相关页面或 source refs 时返回 `missingInformation`；
- 返回 deterministic `confidence`，供主 LLM 判断是否需要进一步 search/read。

## Scope

In scope:

- `SdkWikiToolName` 增加 `sdk_wiki.plan_api_usage`；
- shared payload/result types；
- `SdkWikiService.planApiUsage()`；
- `FileSystemToolHost` visible tool schema and argument normalization；
- `main.ts` SDK Wiki tool dispatch；
- registry system message guidance；
- focused service and ToolHost tests。

Out of scope:

- LLM calls inside SDK Wiki service；
- final SDK script generation；
- SDK/MCP/SAM execution；
- risk confirmation/audit gate；
- SAM-specific adapter logic；
- UI changes beyond existing Settings management。

## Contracts

### Input

```ts
export interface SdkWikiPlanApiUsagePayload extends SdkWikiVersionSelector {
  sdkId: string
  intent: string
  modelState?: Record<string, unknown>
  constraints?: {
    preferSmallSteps?: boolean
    requireSourceRefs?: boolean
  }
  topK?: number
}
```

Rules:

- `sdkId` and `intent` are required.
- `version` is optional only when there is a single installed version for the selected SDK.
- `modelState` must be an object if provided; 14.4 does not do SDK-specific state inference.
- `constraints.requireSourceRefs` defaults to true.
- `constraints.preferSmallSteps` defaults to true.
- `topK` uses existing SDK Wiki clamp semantics, max 12.

### Output

```ts
export interface SdkWikiPlannedApi {
  page: SdkWikiPageRef
  reasons: string[]
  sourceRefs: string[]
}

export interface SdkWikiPlanStep {
  id: string
  title: string
  action: string
  apiRefs: SdkWikiPageRef[]
  sourceRefs: string[]
}

export interface SdkWikiMissingInformation {
  kind: 'no_match' | 'missing_source_refs' | 'missing_relation_target'
  message: string
  query?: string
  pageId?: string
  target?: string
}

export type SdkWikiPlanApiUsageResult =
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

## Planning Algorithm

1. Validate `sdkId`, `version`, `intent`, `modelState`, `constraints`, and `topK`.
2. Select one installed SDK Wiki using existing version resolution semantics.
3. Build query variants from intent:
   - full intent;
   - dotted/code-like symbols found in intent;
   - whitespace/punctuation-separated tokens;
   - bounded CJK n-grams for Chinese task text.
4. Search pages for these variants with existing `SdkWikiIndexReader.searchPages`.
5. Score and dedupe page refs by `pageId`.
6. Identify evidence pages:
   - workflow matches can provide `taskType` and listed `apis`;
   - API matches become candidate required APIs;
   - relation matches can provide listed `apis`;
   - concept matches can add evidence but not become required APIs.
7. Expand relations for candidate APIs to pull direct `requires` / `apis` / `related` API pages.
8. Build `requiredApis` from existing API page refs only.
9. Build `executionPlan`:
   - if `preferSmallSteps` is true, create one step per required API;
   - otherwise group required APIs into one source-referenced step.
10. Validate source refs:
   - if `requireSourceRefs` is true and a recommended API lacks source refs, add `missing_source_refs`;
   - steps inherit source refs from referenced API refs.
11. Return `missingInformation` instead of invented API names when there are no matches or relation targets are missing.

## Technical Decisions

- **TD-01**: `plan_api_usage` is read-only and deterministic; no hidden LLM or external bridge calls.
- **TD-02**: `requiredApis` contains only `pageType === 'api'` pages that exist in `index/pages.json`.
- **TD-03**: Workflow pages can provide `taskType` and ordering hints, but not arbitrary generated steps.
- **TD-04**: Relation targets that cannot be resolved become `missingInformation`, not synthetic API refs.
- **TD-05**: The tool returns bounded output; top-level API count is capped by `topK`.
- **TD-06**: Confidence is heuristic: no APIs = 0, matched APIs raise confidence, workflow/relation/source refs raise it further, missing information lowers it.
- **TD-07**: ToolHost argument normalization rejects non-object `modelState` and non-object `constraints`.

## Files to Modify

- `crewagent-runtime/shared/sdkWikiTypes.ts`
- `crewagent-runtime/electron/services/sdkWikiService.ts`
- `crewagent-runtime/electron/services/sdkWikiService.test.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.test.ts`
- `crewagent-runtime/electron/main.ts`

## Test Plan

Service tests:

- planning for a pressure-load style intent returns existing API refs such as Pressure/Surface/StaticStep when fixture relations support them;
- relation-only planning returns existing API refs from `wiki/relations/*.md` `apis` frontmatter;
- every returned API and step has source refs when fixture pages provide them;
- unknown intent returns `missingInformation` and no invented APIs;
- invalid args return `SDK_WIKI_INVALID_ARGS`.

ToolHost tests:

- visible tools include `sdk_wiki.plan_api_usage`;
- arguments are trimmed and normalized;
- `topK` is clamped;
- invalid `constraints` / missing `intent` is rejected before service dispatch.

Verification:

- focused service test;
- focused ToolHost test;
- targeted ESLint for touched runtime files;
- TypeScript/build verification.

## Traceability

- Story: `_bmad-output/implementation-artifacts/14-4-sdk-api-usage-planning-with-source-references.md`
- PRD: `_bmad-output/prd-runtime-sdk-wiki-and-tool-risk.md`
- Architecture: `_bmad-output/architecture/runtime-sdk-wiki-and-tool-risk-architecture.md`
- Requirements: `_bmad-output/project-planning-artifacts/runtime-sdk-wiki-generic-requirements/02-sdk-llm-wiki-module.md`
