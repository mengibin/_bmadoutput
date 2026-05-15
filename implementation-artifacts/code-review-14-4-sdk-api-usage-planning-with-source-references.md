# Code Review: Story 14.4 - SDK API Usage Planning with Source References

**Date:** 2026-05-14
**Reviewer:** AI Code Reviewer (BMAD Method)
**Story File:** `14-4-sdk-api-usage-planning-with-source-references.md`

---

## Scope

- 本次 review 仅审计 Story 14.4 的 `sdk_wiki.plan_api_usage` shared contract、`SdkWikiService` planning scaffold、ToolHost schema/normalization、main dispatch、prompt guidance 和相关测试。
- 不评估工作区已有无关 dirty changes，也不重新审计 14.1~14.3 已批准内容。

---

## Summary

| Metric | Value |
|--------|-------|
| HIGH Issues | 0 |
| MEDIUM Issues | 0 open / 1 resolved |
| LOW Issues | 0 |
| Targeted Tests | PASS - `cd crewagent-runtime && npm test -- electron/services/sdkWikiService.test.ts` |
| SDK Wiki ToolHost Focused Tests | PASS - `cd crewagent-runtime && npm test -- electron/services/fileSystemToolHost.test.ts -t "SDK Wiki"` |
| Build | PASS - `cd crewagent-runtime && npm run build:ci` |
| Review Outcome | Approved |

---

## Resolved Findings

### M1 - Relation page matches do not contribute their `apis` to the plan - RESOLVED

**Location**
`crewagent-runtime/electron/services/sdkWikiService.ts:513`

**Problem**
Story 14.4 AC-2 explicitly says matching `api`、`workflow` 或 `relation` 页面时，`plan_api_usage` should produce source-referenced API recommendations. The implementation searches all page types and stores relation pages as candidates, but only workflow candidates are later read for frontmatter `apis`:

```ts
const workflowCandidates = this.sortPlanCandidates(candidates)
  .filter((candidate) => candidate.page.pageType === 'workflow')
  .slice(0, 3)
for (const candidate of workflowCandidates) {
  ...
  for (const target of toStringArray(workflowPage.frontmatter.apis)) {
```

Relation page candidates are never processed. They are not converted into `requiredApis`, and they are not used as seeds for `expandRelations` because `seedApiIds` only includes `pageType === 'api'`.

**Impact**
An SDK Wiki Pack can validly represent a task through `wiki/relations/*.md` with `apis: [...]` frontmatter, which the builder rules also allow. If a user intent matches such a relation page but not its individual API pages or workflow pages, `plan_api_usage` can return `requiredApis: []` and `missingInformation: no_match` even though the Wiki contains a valid relation-backed API chain. This violates AC-2 and makes planning weaker exactly for the relation pages introduced by the SDK Wiki structure.

**Recommended Fix**
Generalize the frontmatter `apis` extraction from workflow-only to evidence pages that can carry API refs:

- process `workflow` and `relation` candidates for `frontmatter.apis`;
- resolve each API id/symbol to an existing API page before adding it to candidates;
- add a regression test where an intent only matches a relation page, and the relation page `apis` produce `requiredApis` with source refs.

**Resolution**
The planning service now reads `frontmatter.apis` from both `workflow` and `relation` evidence pages. Workflow pages still provide `taskType`; relation pages contribute only API evidence. Resolved API targets are added as existing API page candidates with relation/workflow reasons, and unresolved targets still become `missing_relation_target`.

Regression coverage was added with a relation-only fixture where the intent matches `relation.load-chain` and no workflow evidence is present. The test verifies that relation page `apis` produce `requiredApis` with source refs and no missing information.

---

## Acceptance Criteria Status

| AC | Status | Notes |
|----|--------|-------|
| AC-1 Internal planning tool | PASS | `sdk_wiki.plan_api_usage` contract, service method, ToolHost schema, and main dispatch exist. |
| AC-2 Source-referenced API recommendations | PASS | API, workflow, relation-backed evidence all resolve only existing API pages and preserve source refs. |
| AC-3 Source-referenced plan steps | PASS | Existing required API steps carry source refs or missing source-ref entries. |
| AC-4 Missing knowledge instead of invention | PASS | No-match path returns `missingInformation` and empty plan in covered scenario. |
| AC-5 Runtime LLM boundary | PASS | Implementation is deterministic and does not call LLM/bridge. |
| AC-6 ToolHost and prompt availability | PASS | ToolHost and registry guidance expose `plan_api_usage`. |
| AC-7 Tests | PASS | Positive API/workflow, relation-only, no-match, ToolHost, TypeScript/build verification are covered. |

---

## Verification Notes

- `npm test -- electron/services/sdkWikiService.test.ts` passed: 38 tests.
- `npm test -- electron/services/fileSystemToolHost.test.ts -t "SDK Wiki"` passed: 2 tests.
- Targeted ESLint passed for touched runtime files.
- `npm exec tsc -- --noEmit` passed.
- `npm run build:ci` passed.
- Full `fileSystemToolHost.test.ts` still has existing unrelated terminal/node/shell failures; SDK Wiki focused tests pass.

---

## Conclusion

Story 14.4 is approved. The review blocker is resolved, relation-backed planning is covered by regression tests, and the targeted verification set passes.
