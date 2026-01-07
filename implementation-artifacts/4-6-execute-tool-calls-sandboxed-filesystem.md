# Story 4.6: Execute Tool Calls (Sandboxed FileSystem)

Status: review

## Story

As a **Runtime**,
I want to read/write files via a sandboxed FileSystem tool host,
so that the AI can create and modify project files safely.

## Acceptance Criteria

1. **Sandboxed File Access (Mounts)**
   - **Allowed Roots**:
     - `@project` -> `<ProjectRoot>` (Read/Write)
     - `@pkg` -> `<PackageRoot>` (Read-Only)
     - `@state` -> `<RunStateRoot>` (Read/Write)
   - **Enforcement**:
     - All paths must start with a valid mount alias.
     - Path traversal (`..`) that escapes the real path of the mount root MUST be rejected with `E_SANDBOX_VIOLATION`.
     - Symlink resolution (`realpath`) must be checked to prevent escape.
     - Writes to `@pkg` MUST be rejected.

2. **FileSystem Tools Implementation**
   - `fs.list(path)`: List directory contents.
   - `fs.read(path, startLine?, endLine?)`: Read file content with optional windowing. Return `sha256` for integrity.
   - `fs.write(path, content)`: Atomic writes (tmp -> rename).
   - `fs.search(path, pattern)`: Structured matches with context lines (`before`/`after`).
   - `fs.apply_patch(path, patch)`: Support `updateFrontmatter` operation for `@state/workflow.md`.

3. **Unified ToolResult Protocol**
   - All tools return `{ ok: true, ...data } | { ok: false, error: ToolError }`.
   - Results serialized via `JSON.stringify()` for `role="tool"` messages.

4. **Standard Error Codes**
   - `ENOENT` - File not found
   - `E_SANDBOX_VIOLATION` - Path outside allowed mounts
   - `E_READ_LIMIT` - File too large to read fully
   - `E_WRITE_LIMIT` - Content exceeds write limit
   - `E_INVALID_FRONTMATTER` - YAML parse failed
   - `E_SCHEMA_VALIDATION` - Frontmatter doesn't match schema
   - `E_INVALID_TRANSITION` - Graph transition not allowed
   - `E_PRECONDITION_FAILED` - `ifMatchSha256` mismatch
   - `E_INTERNAL` - Unexpected error

5. **Safety & Audit**
   - **Narrow Reads**: Default read first ~50KB; return `truncated: true` with hint if larger.
   - **Configurable Limits**: Respect `agent.tools.fs.maxReadBytes` / `maxWriteBytes`.
   - **Audit Log**: Every tool execution logged to `@state/logs/execution.jsonl`.

6. **Story 4.7 Integration Hook**
   - Writes to `@state/workflow.md` require YAML parsing, schema validation, and graph transition validation (deferred to Story 4.7). This story provides the raw FS layer; Story 4.7 wraps it with validation.

## Technical Context

- **Module**: `crewagent-runtime/electron/services/toolHost.ts`
- **Integration**:
  - Implements `ToolHost` interface defined in Story 4.5.
  - `ExecutionEngine` calls `ToolHost.execute(toolCall)` in its loop.
  - `RuntimeStore.resolvePath(mountAlias)` provides real paths.
- **Protocol**: `_bmad-output/tech-spec/llm-conversation-protocol-openai.md` Section 6.

## Design

### Summary

- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-4-6-execute-tool-calls-sandboxed-filesystem.md`
- Implement a concrete `ToolHost` that exposes OpenAI function tools (`fs.*`) and executes them against a sandboxed mount table (`@project/@pkg/@state`).
- Mount mapping: `@project → projectRoot (RW)`, `@pkg → package root (RO)`, `@state → ensureRunStateDir(projectRoot, runId) (RW)`.
- Sandbox enforcement: reject traversal + prevent symlink escapes via `realpath`-based checks; block all writes to `@pkg`.
- Audit: every tool call (including rejected) is logged to `@state/logs/execution.jsonl` with inputs/outputs + duration.

### UX / UI

- N/A (backend-only). Errors must be surfaced via existing execution log/UI by returning structured `ToolResult` errors and letting the engine/UI render them.

### API / Contracts

- **ToolHost execution context must include effective agent** to enforce `agent.tools.fs.maxReadBytes/maxWriteBytes` at runtime:
  - Extend `ToolHost.executeToolCall(...).context` to include `agentId` and have `ExecutionEngine` pass the effective agent for the current node.
- **Tool names (OpenAI function tools)**:
  - `fs.list`, `fs.read`, `fs.search`, `fs.write`, `fs.apply_patch`
- **Tool argument contracts (minimum)**:
  - `fs.list`: `{ path: string }`
  - `fs.read`: `{ path: string, startLine?: number, endLine?: number }` (1-based inclusive window)
  - `fs.search`: `{ path: string, pattern: string, before?: number, after?: number, maxMatches?: number }`
  - `fs.write`: `{ path: string, content: string, ifMatchSha256?: string }`
  - `fs.apply_patch`: `{ path: string, patch: UpdateFrontmatterOperation }` (MVP: `operation="updateFrontmatter"` only)
- **Tool results**: return the stable ToolResult union from `_bmad-output/tech-spec/llm-conversation-protocol-openai.md` so the engine can safely do `JSON.stringify(result)` for `role="tool"` messages.

### Data / Storage

- No new DB; all persistence is filesystem-based.
- `@state` is **run-scoped**: `<...>/runs/<runId>/state/` (via `RuntimeStore.ensureRunStateDir(projectRoot, runId)`).
- Audit log is JSONL at `@state/logs/execution.jsonl`.
- Writes (`fs.write`, `fs.apply_patch`) must be atomic (tmp → rename in the same directory).
- `fs.read` returns `sha256` of the full file content for integrity / optimistic concurrency.

### Errors / Edge Cases

- `E_SANDBOX_VIOLATION`: unknown mount, traversal (`..`), symlink escape, attempting to write `@pkg`, or any resolved path outside the mount root.
- `ENOENT`: file not found (read/search/list/write target as applicable).
- `E_READ_LIMIT`: file too large to read fully; return `truncated=true` + hint (prefer `fs.search` → `fs.read(window)`).
- `E_WRITE_LIMIT`: content exceeds max write bytes.
- `E_PRECONDITION_FAILED`: `ifMatchSha256` provided but does not match current file.
- `E_INVALID_FRONTMATTER`: `fs.apply_patch` cannot parse YAML frontmatter in `@state/workflow.md`.
- `E_INTERNAL`: unexpected errors (include `details` with safe diagnostics; never leak absolute paths to the LLM).

### Test Plan

- Unit (vitest) for sandbox resolver:
  - traversal rejected: `@project/../evil.txt` → `E_SANDBOX_VIOLATION`
  - symlink escape rejected (symlink under `@project` pointing outside)
  - write-to-`@pkg` rejected
- Unit for tool semantics + shapes:
  - `fs.read` windowing + truncation + sha256
  - `fs.list` truncation + skip hidden/symlink entries
  - `fs.search` structured matches + context + truncation
  - `fs.write` atomic write + limits + `ifMatchSha256`
  - `fs.apply_patch` `updateFrontmatter` + atomic write + `ifMatchSha256` mismatch
- Integration (optional): mock LLM tool_calls in `ExecutionEngine` and verify ToolHost results are backfilled as `role="tool"` strings and that audit JSONL is appended.

## Tasks / Subtasks

- [x] Implement `FileSystemProvider` with Sandbox logic.
- [x] Implement `fs.read` with truncation and `sha256`.
- [x] Implement `fs.list` with truncation.
- [x] Implement `fs.search` with context lines.
- [x] Implement `fs.write` with atomic write.
- [x] Implement `fs.apply_patch` with `updateFrontmatter` support.
- [x] Register tools in `ToolHost` (reuse interface from Story 4.5).
- [x] Add unit tests for Sandbox security and ToolResult shapes.

## Dev Notes

- **Execution Flow**: `ExecutionEngine` (Story 4.5) calls `ToolHost.execute()` for each `tool_calls[i]`.
- **Result Serialization**: All results are `JSON.stringify()`-ed before being placed in `role="tool"` messages.
- **State Updates**: Raw `fs.apply_patch` to `@state/workflow.md` is allowed here; validation (YAML/schema/transition) is in Story 4.7.

## Verification

- `npm -C crewagent-runtime run test`

## References

- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-4-6-execute-tool-calls-sandboxed-filesystem.md`
- Protocol: `_bmad-output/tech-spec/llm-conversation-protocol-openai.md`
- Architecture: `_bmad-output/architecture/runtime-architecture.md`
- Story 4.5: `_bmad-output/implementation-artifacts/4-5-call-llm-api-openai-toolcalls-local-llm.md`
- Story 4.7: `_bmad-output/epics.md` (Validate Graph Transition)
