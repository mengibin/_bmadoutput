# Code Review: Story 4.4 – Compose Complete Prompt for LLM

**Date:** 2026-01-01  
**Reviewer:** AI Code Reviewer (BMAD Method)  
**Story File:** `4-4-compose-complete-prompt-for-llm.md`

---

## Summary

| Metric | Value |
|--------|-------|
| Story vs Implementation Discrepancies | 1 |
| HIGH Issues | 0 |
| MEDIUM Issues | 0 |
| LOW Issues | 3 |
| Build | ✅ `npm -C crewagent-runtime run build:ci` passed |
| Unit Tests | ✅ `npm -C crewagent-runtime run test` passed |
| Lint | ❌ `npm -C crewagent-runtime run lint` fails (unrelated issues) |

---

## 🟢 LOW ISSUES

### L1. `safeRelativePath()` silently strips leading slashes
- **Current:** `normalizeSlashPath()` removes leading `/`, so accidental absolute paths become “relative-looking” and are aliased into `@pkg/...` or `@project/...`.
- **Evidence:** `crewagent-runtime/electron/services/promptComposer.ts:74`
- **Impact:** Can mask upstream bugs (e.g. passing `/Users/...` becomes `Users/...`), producing confusing `@pkg/Users/...` paths.
- **Fix:** Consider treating leading `/` as invalid (fail fast) or explicitly document/rename the helper to reflect normalization behavior.

### L2. Invalid graph/step paths degrade to placeholder strings instead of failing fast
- **Current:** When `safeRelativePath()` fails, composer emits `@pkg/<invalid-graph-path>` / `@pkg/<invalid-step-path>`.
- **Evidence:**
  - `crewagent-runtime/electron/services/promptComposer.ts:267`
  - `crewagent-runtime/electron/services/promptComposer.ts:299`
- **Impact:** Execution can proceed with unusable navigation instructions; errors surface later as “file not found” tool results rather than a clear precondition failure.
- **Fix:** Throw a structured error when required paths are invalid (graphPathInPkg / stepFilePathInPkg), consistent with “block on missing/invalid graph/step” story edge cases.

### L3. `STATE_SUMMARY` is an extension beyond the protocol spec (document it)
- **Current:** `RUN_DIRECTIVE` always appends a `STATE_SUMMARY` block after the protocol section.
- **Evidence:**
  - `crewagent-runtime/electron/services/promptComposer.ts:271`
  - `crewagent-runtime/electron/services/promptComposer.ts:344`
- **Impact:** If any downstream parser/tooling assumes user messages contain only `RUN_DIRECTIVE` / `NODE_BRIEF` / `USER_INPUT`, this extra block could break strict parsing.
- **Fix:** Document `STATE_SUMMARY` as an allowed extension in `_bmad-output/tech-spec/llm-conversation-protocol-openai.md`, or emit it as a separate user message with a clear header boundary.

---

## Acceptance Criteria Verification

| AC | Status | Evidence |
|----|--------|----------|
| AC1: Prompt Assembly layers (rules/policy/persona + step reference + variables) | ✅ | Layering in `compose()`: `crewagent-runtime/electron/services/promptComposer.ts:122`; builders: `crewagent-runtime/electron/services/promptComposer.ts:260`, `crewagent-runtime/electron/services/promptComposer.ts:294` |
| AC2: Strict shells (`RUN_DIRECTIVE` / `NODE_BRIEF` / `USER_INPUT`) | ✅ | `buildRunDirective()`: `crewagent-runtime/electron/services/promptComposer.ts:260`; `buildNodeBrief()`: `crewagent-runtime/electron/services/promptComposer.ts:294`; `formatUserInput()`: `crewagent-runtime/electron/services/promptComposer.ts:402` |
| AC3: No absolute path leakage in injected fields | ✅ | Redaction helper: `crewagent-runtime/electron/services/promptComposer.ts:88`; tests: `crewagent-runtime/electron/services/promptComposer.test.ts:143`, `crewagent-runtime/electron/services/promptComposer.test.ts:175` |

---

## Notes (Implementation Strengths)

- Prompt layering matches the docs (`Base Rules → Tool Policy → Persona → RUN_DIRECTIVE → NODE_BRIEF → USER_INPUT`).
- Unit tests cover ordering, persona override, node transitions, and path redaction.
- `dataContext` and variables injection are sanitized before being surfaced to the model.

---

## Next Actions

- [ ] [AI-Review][LOW] Decide whether `safeRelativePath()` should reject leading `/` (or rename to reflect normalization).
- [ ] [AI-Review][LOW] Fail fast on invalid `graphPathInPkg` / `stepFilePathInPkg` rather than emitting placeholder paths.
- [ ] [AI-Review][LOW] Document `STATE_SUMMARY` in the conversation protocol (or emit it as a separate user message).

