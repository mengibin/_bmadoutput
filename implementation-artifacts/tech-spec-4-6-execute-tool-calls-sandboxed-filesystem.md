# Tech Spec: Story 4.6 - Execute Tool Calls (Sandboxed FileSystem)

**Created:** 2026-01-03  
**Status:** Ready for Development  
**Source Story:** `_bmad-output/implementation-artifacts/4-6-execute-tool-calls-sandboxed-filesystem.md`

## Overview

### Problem Statement

The Runtime must safely execute LLM-requested filesystem tool calls (`fs.*`) while enforcing:

- Strict mount roots (`@project`, `@pkg`, `@state`) with correct read/write permissions.
- Strong sandboxing against `..` traversal and **symlink escapes**.
- Token-safe “narrow reads” (windowing + truncation) and configurable read/write limits.
- Atomic writes (tmp → rename) for all write-like operations.
- Audit logging for every tool call (including rejected calls) to `@state/logs/execution.jsonl`.

Story 4.5 introduced the `ToolHost` interface and the ExecutionEngine ToolCalls loop, but ToolHost has no concrete implementation yet.

### Solution

Implement a **local, sandboxed filesystem ToolHost** in Electron Main:

- Provide OpenAI-compatible `tools[]` definitions for `fs.list/fs.read/fs.search/fs.write/fs.apply_patch`.
- Execute tool calls with a mount-aware sandbox resolver:
  - `@project` → `projectRoot` (RW)
  - `@pkg` → extracted package path (RO)
  - `@state` → run-scoped state dir (`ensureRunStateDir(projectRoot, runId)`) (RW)
- Return tool results as a stable **ToolResult union** (string-safe for `JSON.stringify()` embedding).
- Emit audit events with duration + inputs/outputs to `@state/logs/execution.jsonl`.

### Scope (In / Out)

**In scope**
- Implement `fs.*` tools (list/read/search/write/apply_patch) and mount sandboxing.
- Enforce `@pkg` read-only and block all non-mounted / escaping paths.
- Enforce `agent.tools.fs.maxReadBytes` / `maxWriteBytes` (defaults applied if unset).
- Add unit tests for sandbox behavior and ToolResult shapes (no real network).

**Out of scope / Deferred**
- Schema validation + graph transition guard for `@state/workflow.md` updates (Story 4.7).
- Run initialization/copying initial `workflow.md` into run state (Story 4.8).
- MCP tool execution (Story 4.10).

## Context for Development

### Codebase Patterns

- Electron Main services live under `crewagent-runtime/electron/services/`.
- Tests use `vitest` and run in Node (`npm -C crewagent-runtime test`).
- OpenAI-compatible message/tool shapes are defined in:
  - `crewagent-runtime/electron/services/openaiProtocol.ts`
- The ToolCalls loop expects `ToolHost.executeToolCall(...)` to return a JSON-serializable `ToolResult` and the engine backfills it as:
  - `role="tool"`, `tool_call_id=<id>`, `content=JSON.stringify(toolResult)`
  - see `crewagent-runtime/electron/services/executionEngine.ts`.

### Files to Reference

- Story: `_bmad-output/implementation-artifacts/4-6-execute-tool-calls-sandboxed-filesystem.md`
- Protocol (tool result shapes + mounts): `_bmad-output/tech-spec/llm-conversation-protocol-openai.md`
- ToolHost interface: `crewagent-runtime/electron/services/toolHost.ts`
- Engine tool execution contract: `crewagent-runtime/electron/services/executionEngine.ts`
- Package path + run state dir helpers:
  - `crewagent-runtime/electron/stores/runtimeStore.ts` (`getPackage().path`, `ensureRunStateDir(...)`)
- Prompt contract expects mount paths only:
  - `crewagent-runtime/electron/services/prompt-templates/system-base-rules.md`

### Technical Decisions

1. **`@state` must be run-scoped**
   - Use `RuntimeStore.ensureRunStateDir(projectRoot, runId)` as the `@state` mount root.
   - Do **not** reuse `RuntimeStore.readFile/writeFile` for tool execution (it currently maps `@state` differently and does not enforce symlink-escape protection).

2. **Sandboxing must be symlink-safe**
   - Path traversal is blocked at the alias parsing layer (`..` segments rejected).
   - Symlink escape is blocked via `realpath`-based prefix checks.

3. **Limits are enforced at execution time**
   - LLM prompt includes ToolPolicy text, but the runtime must enforce actual byte limits regardless of model compliance.

## Contracts

### ToolHost API (Execution-time context)

Story 4.5’s `ToolHost.executeToolCall` does not currently include `agentId`, but Story 4.6 must enforce `agent.tools.fs.maxReadBytes/maxWriteBytes`.

**Decision:** extend the execution context to include `agentId` (the effective agent for the current node), and update ExecutionEngine to pass it.

Proposed API (minimal change):

```ts
export interface ToolHost {
  getVisibleTools(context: { packageId: string; agentId: string }): OpenAIFunctionTool[]
  executeToolCall(params: {
    toolCallId: string
    name: string
    argumentsJson: string
    context: { projectRoot: string; packageId: string; runId: string; agentId: string }
    signal?: AbortSignal
  }): Promise<ToolResult>
}
```

### ToolResult Shapes

Follow `_bmad-output/tech-spec/llm-conversation-protocol-openai.md` §6.

```ts
export interface ToolError {
  code:
    | 'ENOENT'
    | 'E_SANDBOX_VIOLATION'
    | 'E_READ_LIMIT'
    | 'E_WRITE_LIMIT'
    | 'E_INVALID_FRONTMATTER'
    | 'E_SCHEMA_VALIDATION'
    | 'E_INVALID_TRANSITION'
    | 'E_PRECONDITION_FAILED'
    | 'E_INTERNAL'
  message: string
  details?: Record<string, unknown>
}

export type ToolResult =
  | FsReadResult
  | FsListResult
  | FsSearchResult
  | FsWriteResult
  | FsApplyPatchResult
```

### OpenAI `tools[]` Definitions

Expose function tools with stable names (must match prompts and engine history):

- `fs.list`
- `fs.read`
- `fs.search`
- `fs.write`
- `fs.apply_patch`

Parameter schemas should be strict enough to help the model, but tolerant of extra fields.

Recommended parameter schemas (sketch):

```ts
// fs.list
{
  type: 'object',
  properties: { path: { type: 'string' } },
  required: ['path'],
  additionalProperties: false,
}

// fs.read
{
  type: 'object',
  properties: {
    path: { type: 'string' },
    startLine: { type: 'integer', minimum: 1 },
    endLine: { type: 'integer', minimum: 1 },
  },
  required: ['path'],
  additionalProperties: false,
}

// fs.search
{
  type: 'object',
  properties: {
    path: { type: 'string' },
    pattern: { type: 'string' },
    before: { type: 'integer', minimum: 0 },
    after: { type: 'integer', minimum: 0 },
    maxMatches: { type: 'integer', minimum: 1 },
  },
  required: ['path', 'pattern'],
  additionalProperties: false,
}

// fs.write
{
  type: 'object',
  properties: {
    path: { type: 'string' },
    content: { type: 'string' },
    ifMatchSha256: { type: 'string' },
  },
  required: ['path', 'content'],
  additionalProperties: false,
}

// fs.apply_patch (updateFrontmatter only in this story)
{
  type: 'object',
  properties: {
    path: { type: 'string' },
    patch: {
      type: 'object',
      properties: {
        operation: { const: 'updateFrontmatter' },
        update: { type: 'object' },
        ifMatchSha256: { type: 'string' },
      },
      required: ['operation', 'update'],
      additionalProperties: true,
    },
  },
  required: ['path', 'patch'],
  additionalProperties: false,
}
```

## Sandbox & Mount Resolution

### Alias Parsing Rules

- Input paths are always “mount alias paths” (e.g. `@project/src/index.ts`).
- Reject:
  - missing/unknown mount (`@foo/...`)
  - empty mount (`@project` is allowed and resolves to mount root)
  - NUL bytes
  - absolute paths (`/etc/passwd`, `C:\...`)
  - any `..` segment after normalization (prevents traversal)

### Symlink-safe Resolution Algorithm

Given `aliasPath` and `mountRootAbs`:

1. Normalize alias to forward slashes and strip leading `./` and `/`.
2. Split `@mount` and `relativePath`.
3. Compute candidate absolute path: `candidateAbs = path.resolve(mountRootAbs, relativePath)`.
4. Compute `mountRootReal = realpath(mountRootAbs)` (directory must exist).
5. Compute `candidateReal`:
   - If `candidateAbs` exists: `candidateReal = realpath(candidateAbs)`
   - If it does not exist (writes): resolve `realpath(dirname(candidateAbs))` and join basename, then validate parent is inside mount.
6. Allow only if `candidateReal` is within `mountRootReal` (prefix check using `path.relative` and rejecting `..` / absolute).

### Mount Permissions

- `@project`: read/write
- `@pkg`: read-only (any write or patch returns `E_SANDBOX_VIOLATION`)
- `@state`: read/write

## Tool Semantics

### `fs.list(path)`

- Lists direct children of a directory.
- Skips hidden entries (`.` prefix) and symbolic links (avoid accidental escape vectors).
- Truncation: if entries exceed `maxEntries` (recommend default 200), return:
  - `{ ok: true, path, entriesPreview, truncated: true, hint }`

### `fs.read(path, startLine?, endLine?)`

- Reads file content as UTF-8.
- If `startLine/endLine` provided:
  - 1-based, inclusive.
  - If `endLine < startLine`, return `E_INTERNAL`.
- Limit enforcement:
  - Use `maxReadBytes` from agent policy (fallback default: 50_000 bytes).
  - If full read would exceed limit, return truncated result with `contentPreview` and a hint to use `startLine/endLine` or `fs.search`.
- Always return:
  - `bytes`: total file size (from `stat.size`)
  - `sha256`: hash of the full file bytes (streaming hash to avoid memory blow-up)

### `fs.search(path, pattern, before?, after?, maxMatches?)`

- Searches within a directory tree (recursive).
- Defaults:
  - `before=1`, `after=1`, `maxMatches=50`
- Skips directories: `.git`, `node_modules`, and hidden directories by default.
- Pattern semantics:
  - Treat `pattern` as a plain substring match (MVP).
  - (Optional) If `pattern` is wrapped like `/.../i`, interpret as regex.
- Returns structured matches:
  - `path` is mount alias
  - `line` is 1-based
  - `text` is the full line (trim trailing newline)
  - include `before/after` context arrays when requested
- Truncation:
  - Stop after `maxMatches` and set `truncated=true` with basic `stats`.

### `fs.write(path, content, ifMatchSha256?)`

- Allowed mounts: `@project`, `@state` only.
- Enforce `maxWriteBytes` from agent policy (fallback default: 100_000 bytes).
- Optional optimistic concurrency:
  - If `ifMatchSha256` is set, compute current file sha256; if mismatch return `E_PRECONDITION_FAILED`.
- Atomic write:
  - Ensure parent directory exists.
  - Write to `.${basename}.tmp.<uuid>` in the same directory.
  - `fs.rename` over the target (same filesystem).
- Return `{ ok: true, path, bytesWritten, sha256After }`.

### `fs.apply_patch(path, patch)`

MVP scope: **only** supports `patch.operation === "updateFrontmatter"` (for `@state/workflow.md`).

- Allowed mounts: `@state` only for `updateFrontmatter` in this story.
- Parse the target Markdown using `gray-matter`.
- Apply updates:
  - `stepsCompleted.append`: append unique strings
  - `variables.set`: shallow merge into `variables`
  - `decisionLog.append`: append objects (no schema validation here)
  - `artifacts.append`: append unique strings
  - `currentNodeId.set`: overwrite
  - `updatedAt.set`: overwrite
- If YAML parsing fails: return `E_INVALID_FRONTMATTER`.
- If `ifMatchSha256` present and does not match current file: `E_PRECONDITION_FAILED`.
- Write back atomically (same temp+rename strategy as `fs.write`).
- Return `{ ok: true, path, sha256Before, sha256After }`.

## Audit Logging

### Location

Append JSONL events to:

`@state/logs/execution.jsonl` → `<runStateDir>/logs/execution.jsonl`

Use `RuntimeStore.ensureRunStateDir(projectRoot, runId)` to ensure the run state dir exists.

### Event Shape (recommended)

Each tool execution emits exactly one event (even on failures):

```json
{
  "ts": "ISO8601",
  "kind": "tool.exec",
  "toolCallId": "call-1",
  "toolName": "fs.read",
  "agentId": "sm",
  "input": { "path": "@project/foo.md", "startLine": 1, "endLine": 20 },
  "output": { "ok": true, "...": "..." },
  "durationMs": 12
}
```

Notes:
- Do not log full file contents; log sizes/hashes and truncated previews only.
- Rejected sandbox calls must include enough detail to debug (mount + normalized path + reason).

## Implementation Plan

### Tasks

- [ ] Add a concrete ToolHost implementation (e.g. `electron/services/fileSystemToolHost.ts`) for `fs.*`.
- [ ] Extend `ToolHost.executeToolCall` context to include `agentId` and update `ExecutionEngineImpl` to pass the effective agent.
- [ ] Implement mount resolver + symlink-safe sandbox checks.
- [ ] Implement `fs.list` + truncation.
- [ ] Implement `fs.read` with windowing + truncation + streaming sha256.
- [ ] Implement `fs.search` with context lines + truncation.
- [ ] Implement `fs.write` with limits + atomic write + (optional) ifMatchSha256.
- [ ] Implement `fs.apply_patch` (`updateFrontmatter` only) + atomic write + ifMatchSha256.
- [ ] Implement audit JSONL logging for each tool call (including rejected ones).
- [ ] Add vitest coverage for sandbox, limits, and ToolResult shapes.

### Acceptance Criteria (Implementation Checklist)

- [ ] Mount enforcement: only `@project/@pkg/@state` accepted; unknown roots rejected with `E_SANDBOX_VIOLATION`.
- [ ] Sandbox: `..` traversal and symlink escapes are rejected with `E_SANDBOX_VIOLATION`.
- [ ] Permissions: `@pkg` writes are rejected; only `@project/@state` are writable.
- [ ] `fs.read` supports windowing and token-safe truncation; returns `sha256`.
- [ ] `fs.search` returns structured matches with optional context and truncation.
- [ ] `fs.write/fs.apply_patch` enforce write limits and are atomic (tmp → rename).
- [ ] Tool results always match ToolResult union and are `JSON.stringify()` safe.
- [ ] Every tool call is audit-logged with input/output + duration to `@state/logs/execution.jsonl`.

## Additional Context

### Dependencies

- Node: `fs`, `path`, `crypto` (sha256)
- Frontmatter parsing: `gray-matter` (already used by RuntimeStore)

### Testing Strategy

Add `vitest` tests in `crewagent-runtime/electron/services/`:

- Sandbox:
  - traversal: `@project/../evil.txt` → `E_SANDBOX_VIOLATION`
  - symlink escape: create a symlink under `@project` pointing outside and ensure read/write reject
- Permissions:
  - `fs.write('@pkg/x', ...)` → `E_SANDBOX_VIOLATION`
- Read limits:
  - large file returns `truncated: true` and hint
- Atomic writes:
  - write creates final file with expected content; no partial files on success
- apply_patch:
  - updates frontmatter fields and preserves body
  - `ifMatchSha256` mismatch → `E_PRECONDITION_FAILED`

Run:

- `npm -C crewagent-runtime test -- electron/services`

## Traceability

- Story: `_bmad-output/implementation-artifacts/4-6-execute-tool-calls-sandboxed-filesystem.md`
- Protocol: `_bmad-output/tech-spec/llm-conversation-protocol-openai.md`
- Related:
  - Story 4.5 engine loop: `_bmad-output/implementation-artifacts/4-5-call-llm-api-openai-toolcalls-local-llm.md`
  - Story 4.7 validation: `_bmad-output/epics.md` (Story 4.7)
  - Story 4.8 run init: `_bmad-output/epics.md` (Story 4.8)
