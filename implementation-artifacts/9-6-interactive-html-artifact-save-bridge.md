# Story 9.6: Interactive HTML Artifact Save Bridge

Status: done

<!-- Note: Implementation, review follow-up fixes, and re-review are complete. Focused tests and build pass; lint remains blocked only by a pre-existing unrelated Fast Refresh warning. -->

## Overview

Enable workflow-generated HTML artifacts under `@project` to provide richer user interaction and persist user-produced output files for downstream workflow steps.

This story follows the Interactive HTML Artifact specification:

- `crewagent-runtime/docs/interactive-html-artifact-spec.md`

## User Story

As a **workflow author**,
I want a step to copy a predefined interactive HTML asset into `@project`, load nearby JSON input data, and let the user save output files from that HTML,
So that the next workflow step can read structured or document output produced by a richer UI than a plain chat form.

## Acceptance Criteria

### AC-1: Project HTML Preview Can Request Saves

**Given** a user opens an HTML file from `@project` in Runtime preview  
**When** the HTML sends a `window.parent.postMessage` message with type `crewagent.interactive.save`  
**Then** Runtime accepts the message only from the active preview iframe.

### AC-2: Output Path Is Scoped To HTML Directory

**Given** the HTML requests a save with `outputPath`  
**When** Runtime resolves the path  
**Then** the output is written relative to the current HTML file's directory  
**And** the resolved output path cannot escape that directory or its subdirectories.

### AC-3: Flexible Output File Types

**Given** the HTML sends `content` and `encoding`  
**When** Runtime saves the output  
**Then** Runtime supports `utf-8` text output and `base64` binary output  
**And** it does not restrict the output extension to JSON.

### AC-4: Unsafe Paths Are Rejected

Runtime rejects output paths that:

- are empty or directory-only
- contain `..`
- start with `/`
- contain protocol-style prefixes such as `file://`, `http://`, or `https://`
- contain `@project`
- attempt to overwrite the source HTML file

### AC-5: Save Result Is Returned To HTML

**Given** Runtime finishes a save request  
**Then** Runtime posts a `crewagent.interactive.saveResult` message back to the iframe  
**And** the result includes the original `requestId`, success flag, output path, and error if failed.

## Design Specification

### Protocol

HTML save request:

```js
window.parent.postMessage(
  {
    type: "crewagent.interactive.save",
    version: 1,
    requestId: crypto.randomUUID(),
    outputPath: "./submission.json",
    encoding: "utf-8",
    content: JSON.stringify(data, null, 2)
  },
  "*"
);
```

Runtime save result:

```json
{
  "type": "crewagent.interactive.saveResult",
  "version": 1,
  "requestId": "same-request-id",
  "ok": true,
  "outputPath": "./submission.json"
}
```

### Data Flow

```text
@project HTML iframe
  -> postMessage(crewagent.interactive.save)
  -> HtmlPreview verifies iframe source/origin
  -> window.ipcRenderer.saveInteractiveArtifact(...)
  -> ipcMain files:saveInteractiveArtifact
  -> RuntimeStore.writeInteractiveHtmlArtifact(...)
  -> file written under current HTML directory
  -> saveResult posted back to iframe
```

### Security Boundary

- HTML never receives `ipcRenderer`.
- HTML cannot directly write files.
- HTML may choose `outputPath`, but Runtime treats it as relative to the HTML file directory and validates it before writing.
- Only `@project` HTML files are accepted as interactive artifact sources.

## Tasks / Subtasks

- [x] 1. Document the Interactive HTML Artifact contract.
- [x] 2. Add RuntimeStore save method with path validation.
- [x] 3. Add Electron IPC route and preload bridge.
- [x] 4. Add HTML preview iframe `postMessage` listener and save result callback.
- [x] 5. Add RuntimeStore tests for allowed save, traversal rejection, `@project` path rejection, source overwrite rejection, and non-project HTML rejection.

## Verification Plan

### Automated

- `npm test -- --run electron/stores/runtimeStore.test.ts src/components/files/fileViewers.test.tsx`
- `npm run build:ci`

### Manual

1. Create an HTML artifact under `@project/artifacts/manual/`.
2. In that HTML, call `fetch("./input.json")` and render data.
3. Send `crewagent.interactive.save` with `outputPath: "./submission.json"`.
4. Verify Runtime writes `@project/artifacts/manual/submission.json`.
5. Try `../escape.json` and verify Runtime rejects it and posts a failed `saveResult`.

## Dev Agent Record

### Implementation Notes

- Implemented the save bridge as a dedicated file IPC rather than reusing generic `files:write`, so path scoping is enforced server-side.
- The iframe message listener validates message source and preview origin before forwarding to IPC.
- Output extension remains unrestricted; save handling supports `utf-8` and `base64`.

### Review Follow-up

- Added strict base64 validation so malformed binary payloads are rejected before any file write.
- Added segment-level protocol marker rejection for `outputPath` values such as `./outputs/https://example/result.txt`.
- Added renderer-level bridge tests for active iframe save forwarding, save result callback, and outside-source message rejection.
- Added real filesystem identity checks so source HTML overwrite is rejected even through case-variant paths.
- Added realpath containment checks so symlinked output directories cannot escape the HTML directory boundary.
- Bound async save result delivery to the original source iframe instead of the current iframe ref after preview changes.
- Re-reviewed Story 9-6 after follow-up fixes; no new implementation findings were found.

### Verification Results

- ✅ `npm test -- --run electron/stores/runtimeStore.test.ts src/components/files/fileViewers.test.tsx` (92 tests)
- ✅ Re-review verification: `npm test -- --run electron/stores/runtimeStore.test.ts src/components/files/fileViewers.test.tsx` (92 tests)
- ✅ `npm run build:ci`
- ❌ `npm run lint` still fails on a pre-existing `src/components/workflow/WorkflowGraphView.tsx` Fast Refresh warning unrelated to Story 9-6.

### File List

- `crewagent-runtime/docs/interactive-html-artifact-spec.md`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/stores/runtimeStore.test.ts`
- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/preload.ts`
- `crewagent-runtime/electron/electron-env.d.ts`
- `crewagent-runtime/src/vite-env.d.ts`
- `crewagent-runtime/src/components/files/fileViewers.tsx`
- `crewagent-runtime/src/components/files/fileViewers.test.tsx`
- `_bmad-output/implementation-artifacts/code-review-9-6-interactive-html-artifact-save-bridge.md`
