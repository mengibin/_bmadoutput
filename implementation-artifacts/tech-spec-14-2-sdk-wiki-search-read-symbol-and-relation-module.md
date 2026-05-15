# Tech-Spec: Story 14.2 SDK Wiki Search, Read, Symbol, and Relation Module

**Created:** 2026-05-14
**Status:** Ready for Development
**Source Story:** `_bmad-output/implementation-artifacts/14-2-sdk-wiki-search-read-symbol-and-relation-module.md`

## Overview

### Problem Statement

Story 14.1 can import and register SDK Wiki Packs, but Runtime cannot yet expose that knowledge to the main LLM except as installed metadata. Story 14.2 adds read-only SDK Wiki abilities so the LLM can discover relevant pages, read source-grounded content on demand, resolve API symbols, and expand indexed relations without loading an entire SDK Wiki into the prompt.

### Solution

Add a read-only `SdkWikiIndexReader` behind `SdkWikiService` query methods. Wire those methods into `FileSystemToolHost` as internal tools:

- `sdk_wiki.list_sdks`
- `sdk_wiki.search_pages`
- `sdk_wiki.read_page`
- `sdk_wiki.resolve_symbol`
- `sdk_wiki.expand_relations`

Add a compact SDK Wiki registry system message through the existing extra-system-message injection path. The context message advertises available SDK Wikis and tool usage, but never embeds full page content.

## Context for Development

### Existing Runtime Surface

- `RuntimeStore` owns SDK Wiki private storage and registry helpers from Story 14.1.
- `SdkWikiService` currently exposes import/list and has path/hash/page validation helpers.
- `FileSystemToolHost` already exposes internal tools and routes `project_kb.retrieve` through a callback.
- `toolHost.ts` owns the `ToolResult` union; SDK Wiki result shapes must be added there.
- `main.ts` already instantiates `sdkWikiService`.
- `ExecutionEngine` already injects extra system messages for skill registry context; SDK Wiki compact registry should follow the same pattern.

### Observed Index Shape

Local SAM/Abaqus fixtures use:

```ts
type PagesIndex = Array<{
  id: string
  type: 'api' | 'workflow' | 'concept' | 'relation'
  title: string
  path: string
  summary?: string
  module?: string
  risk?: string
}>

type StringToPageIds = Record<string, string[]>

type RelationsIndex = Record<string, {
  requires?: string[]
  related?: string[]
  apis?: string[]
}>
```

The reader should tolerate missing optional fields, but required ids/paths/types should fail with structured index errors.

## Contracts

### Shared Types

Extend `shared/sdkWikiTypes.ts` or a small adjacent shared file with:

```ts
type SdkWikiQueryErrorCode =
  | 'SDK_WIKI_INVALID_ARGS'
  | 'SDK_WIKI_NOT_INSTALLED'
  | 'SDK_WIKI_VERSION_REQUIRED'
  | 'SDK_WIKI_PAGE_NOT_FOUND'
  | 'SDK_WIKI_SYMBOL_NOT_FOUND'
  | 'SDK_WIKI_INDEX_INVALID'
  | 'SDK_WIKI_READ_FAILED'

interface SdkWikiPageRef {
  sdkId: string
  version: string
  pageId: string
  pageType: 'api' | 'workflow' | 'concept' | 'relation'
  title: string
  path: string
  summary?: string
  score?: number
  sourceRefs?: string[]
}

interface SdkWikiToolError {
  code: SdkWikiQueryErrorCode
  message: string
  details?: unknown
}
```

Tool result examples:

```ts
type SdkWikiSearchPagesResult =
  | { ok: true; mode: 'search_pages'; items: SdkWikiPageRef[]; query: string; truncated?: boolean }
  | { ok: false; error: SdkWikiToolError }

type SdkWikiReadPageResult =
  | {
      ok: true
      mode: 'read_page'
      page: SdkWikiPageRef
      frontmatter: Record<string, unknown>
      content: string
      bytes: number
      truncated: boolean
    }
  | { ok: false; error: SdkWikiToolError }
```

### Tool Schemas

`sdk_wiki.list_sdks`:

```json
{
  "type": "object",
  "properties": {
    "sdkId": { "type": "string" },
    "version": { "type": "string" }
  },
  "additionalProperties": false
}
```

`sdk_wiki.search_pages`:

```json
{
  "type": "object",
  "properties": {
    "sdkId": { "type": "string" },
    "version": { "type": "string" },
    "query": { "type": "string" },
    "pageTypes": {
      "type": "array",
      "items": { "type": "string", "enum": ["api", "workflow", "concept", "relation"] }
    },
    "topK": { "type": "integer", "minimum": 1, "maximum": 12 }
  },
  "required": ["query"],
  "additionalProperties": false
}
```

`sdk_wiki.read_page`:

```json
{
  "type": "object",
  "properties": {
    "sdkId": { "type": "string" },
    "version": { "type": "string" },
    "pageId": { "type": "string" },
    "mode": { "type": "string", "enum": ["content", "full"] },
    "maxBytes": { "type": "integer", "minimum": 1 }
  },
  "required": ["sdkId", "pageId"],
  "additionalProperties": false
}
```

`sdk_wiki.resolve_symbol`:

```json
{
  "type": "object",
  "properties": {
    "sdkId": { "type": "string" },
    "version": { "type": "string" },
    "symbol": { "type": "string" },
    "topK": { "type": "integer", "minimum": 1, "maximum": 12 }
  },
  "required": ["symbol"],
  "additionalProperties": false
}
```

`sdk_wiki.expand_relations`:

```json
{
  "type": "object",
  "properties": {
    "sdkId": { "type": "string" },
    "version": { "type": "string" },
    "pageId": { "type": "string" },
    "symbol": { "type": "string" },
    "depth": { "type": "integer", "minimum": 1, "maximum": 2 }
  },
  "additionalProperties": false
}
```

## Implementation Details

### `SdkWikiIndexReader`

Suggested location: `electron/services/sdkWikiIndexReader.ts`.

Responsibilities:

- Load `pages.json`, `symbols.json`, `terms.json`, and `relations.json`.
- Normalize page entries and build `pageId -> page` maps.
- Resolve symbol/term map entries to page refs.
- Read Markdown by page id only.
- Parse frontmatter with `gray-matter`.
- Enforce root containment with `path.resolve` + `path.relative`.
- Return no absolute paths.

No cache is required for MVP. If a cache is added, key it by `sdkId@version` and keep invalidation simple by rebuilding on process restart or import success.

### `SdkWikiService`

Add query methods:

- `listSdks(params?)`
- `searchPages(params)`
- `readPage(params)`
- `resolveSymbol(params)`
- `expandRelations(params)`
- `formatSdkWikiRegistrySystemMessage()`

Version selection rules:

- Exact `sdkId + version` selects one pack.
- `sdkId` with one ready version selects that version.
- `sdkId` with multiple ready versions and no `version` returns `SDK_WIKI_VERSION_REQUIRED`.
- No `sdkId` searches all ready SDKs where meaningful; `read_page` should require `sdkId`.

Bounded output defaults:

- `topK`: default 5, max 12.
- Search snippet/summary: max 500 chars.
- `read_page` default max: 12,000 bytes.
- `read_page` hard max: 48,000 bytes.

Search heuristic:

1. exact page id/title match;
2. exact symbol match from `symbols.json`;
3. exact/lowercase term match from `terms.json`;
4. substring match against title, id, path, summary;
5. de-duplicate by `sdkId@version:pageId` and sort by score.

### ToolHost

Add `SdkWikiToolResult` to `toolHost.ts` and include it in `ToolResult`.

Add optional callback to `FileSystemToolHostOptions`:

```ts
onSdkWikiToolCall?: (payload: {
  toolName: 'sdk_wiki.list_sdks' | 'sdk_wiki.search_pages' | 'sdk_wiki.read_page' | 'sdk_wiki.resolve_symbol' | 'sdk_wiki.expand_relations'
  args: Record<string, unknown>
  context: ExecuteContext
}) => Promise<SdkWikiToolResult>
```

In `FileSystemToolHost`:

- Add schemas to `getVisibleTools()`.
- Add dispatch branch before fs tools.
- Validate required primitive fields and clamp numeric bounds before callback.
- Return `TOOL_NOT_AVAILABLE` or `E_INTERNAL` if callback is missing.

### Main Wiring

`main.ts` should pass `onSdkWikiToolCall` to `FileSystemToolHost` and route to `sdkWikiService`.

Keep IPC import/list handlers from 14.1 unchanged.

### Compact Registry Context

Use a helper similar to `formatSkillRegistrySystemMessage`.

Content should include:

- installed SDK id/version/name/language/page count;
- allowed tool names;
- guidance: use `search_pages`, `resolve_symbol`, `read_page`, and `expand_relations` before SDK API claims;
- warning: do not invent APIs when no SDK Wiki result exists.

Injection route:

- Prefer `ExecutionEngine` extra system messages so chat/agent/run can share the same behavior.
- Keep `PromptComposer` generic unless implementation finds a cleaner local pattern.

## Tests

### Service Tests

- compact list returns installed SDK metadata without absolute paths;
- search exact page id/title/summary;
- search term index;
- search symbol index;
- read page returns frontmatter/content/sourceRefs;
- read page truncates and reports `truncated`;
- read page unknown id returns `SDK_WIKI_PAGE_NOT_FOUND`;
- corrupted `pages.json` returns `SDK_WIKI_INDEX_INVALID`;
- indexed path escape is rejected;
- resolve symbol hit/no_match;
- expand relations resolves related/requires and reports missingTargets;
- multi-version ambiguity returns `SDK_WIKI_VERSION_REQUIRED`.

### ToolHost Tests

- SDK Wiki tools are visible when ready SDK Wikis exist;
- list/search/read/resolve/expand dispatch to callback with normalized args;
- invalid args return structured tool errors;
- callback missing returns structured error;
- result payloads contain no absolute RuntimeStore path.

### Context Tests

- compact registry system message appears when ready SDK Wikis exist;
- message does not include full Markdown page bodies;
- message includes guidance to use SDK Wiki tools before SDK API claims.

## Acceptance Criteria Mapping

| AC | Implementation |
|----|----------------|
| AC-1 | Compact registry system message via ExecutionEngine extra system messages |
| AC-2 | `listSdks` service method + `sdk_wiki.list_sdks` tool |
| AC-3 | `searchPages` over pages/symbols/terms indexes |
| AC-4 | `readPage` via index page id and root-contained Markdown read |
| AC-5 | `resolveSymbol` over `symbols.json` |
| AC-6 | `expandRelations` over `relations.json` with resolved refs/missingTargets |
| AC-7 | Structured errors, read-only service, ToolHost fallback behavior |

## Notes

- Do not implement `sdk_wiki.plan_api_usage` in 14.2.
- Do not add embedding/BM25/SQLite.
- Do not expose absolute RuntimeStore paths.
- Do not use recursive page discovery as a substitute for the imported indexes.
