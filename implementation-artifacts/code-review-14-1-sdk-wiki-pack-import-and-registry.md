# Code Review: Story 14.1 - SDK Wiki Pack Import and Registry

**Date:** 2026-05-14
**Reviewer:** AI Code Reviewer (BMAD Method)
**Story File:** `14-1-sdk-wiki-pack-import-and-registry.md`

---

## Scope

- 本次 review 仅审计 Story 14.1 的 SDK Wiki import/registry 实现。
- 不评估工作区已有无关 dirty changes：`package*.json`、`src/pages/WorksPage/*`、builder/docs 相关文件。

---

## Summary

| Metric | Value |
|--------|-------|
| HIGH Issues | 0 open (1 resolved) |
| MEDIUM Issues | 0 open (2 resolved) |
| LOW Issues | 0 |
| Targeted Tests | ✅ `cd crewagent-runtime && npm run test -- electron/services/sdkWikiService.test.ts electron/stores/runtimeStore.test.ts` |
| Targeted Lint | ✅ `cd crewagent-runtime && npx eslint electron/services/sdkWikiService.ts electron/services/sdkWikiService.test.ts electron/stores/runtimeStore.ts electron/stores/runtimeStore.test.ts electron/main.ts electron/preload.ts shared/sdkWikiTypes.ts src/vite-env.d.ts --ext ts,tsx --report-unused-disable-directives --max-warnings 0` |
| Build | ✅ `cd crewagent-runtime && npm run build:ci` |
| Review Outcome | Approved |

---

## Resolved Findings

### H1 - Manifest-listed `wiki/*.md` pages can bypass frontmatter validation

**What changed**
- `validatePages()` now receives `index/manifest.json.files`, validates every manifest-listed `wiki/**/*.md`, and requires `id/type/title` frontmatter on each page.
- `index/pages.json` references must now also appear in `index/manifest.json.files`.
- Added regression coverage for manifest-listed pages outside `index/pages.json` and pages indexed outside the manifest file list.

**Assessment**
AC-3 page frontmatter coverage now matches the tech spec contract.

---

### M1 - Archive entries containing `..` are normalized instead of rejected

**What changed**
- `normalizeZipEntryPath()` now rejects raw archive path segments equal to `..` before normalization.
- Added a regression test using an archive entry name mutated to `foo/../evil.txt`.

**Assessment**
Archive path handling now satisfies the explicit traversal rejection requirement.

---

### M2 - Directory imports can preserve symlinks that escape the registered pack root

**What changed**
- Import staging now scans the copied/extracted pack and rejects any symlink before manifest/hash/page validation.
- Added regression coverage for a directory pack where a `wiki/` page is replaced by a symlink to an external file.

**Assessment**
Registered SDK Wiki assets can no longer preserve symlinks from imported directory packs.

---

## Acceptance Criteria Status

| AC | Status | Notes |
|----|--------|-------|
| AC-1 Directory/archive import | ✅ | Directory/archive happy paths work; archive path traversal and parent segments are rejected. |
| AC-2 Required structure/manifest validation | ✅ | Required dirs, `sdk-wiki.json`, `index/manifest.json`, schema/version/id/entry are covered. |
| AC-3 Index/hash/page validation | ✅ | Index files, hashes, manifest-listed wiki frontmatter, and page index consistency are covered. |
| AC-4 RuntimeStore registration | ✅ | Commit path, registry write, alias, list payload, install metadata are implemented. |
| AC-5 Failure safety | ✅ | Validation failures leave registry unchanged; symlink and archive path failures are rejected before registration. |

---

## Test Notes

- Follow-up targeted Vitest suite passes: 2 files, 105 tests.
- Targeted ESLint passes for all 14.1 touched runtime files.
- `npm run build:ci` passes; remaining warnings are existing npm/Vite warnings.
- Full repo lint/test were not rerun during this follow-up. Previous dev-story run reported unrelated full-lint warning in `src/components/workflow/WorkflowGraphView.tsx:52` and an unrelated flaky full-test failure in `electron/services/fileSystemToolHost.test.ts`.

---

## Conclusion

No blocking findings remain for Story 14.1. The story can proceed from `review` once the team accepts the follow-up changes.
