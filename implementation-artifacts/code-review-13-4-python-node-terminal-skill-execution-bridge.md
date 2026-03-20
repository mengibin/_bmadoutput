# Code Review: Story 13.4 - Python/Node/Terminal Skill Execution Bridge

**Date:** 2026-03-19  
**Reviewer:** AI Code Reviewer (BMAD Method)  
**Story File:** `13-4-python-node-terminal-skill-execution-bridge.md`

---

## Scope

- 本次为 review follow-up 完成后的 re-review。
- 复审重点是 `node.run` / `npm.install` 的执行边界、skill workspace 并发语义，以及新增的 `python.run module` / `node.run` / `npm.install` 回归覆盖。

---

## Summary

| Metric | Value |
|--------|-------|
| HIGH Issues | 0 open (1 resolved) |
| MEDIUM Issues | 0 open (1 resolved) |
| LOW Issues | 0 |
| Targeted Tests | ✅ `cd crewagent-runtime && npx vitest run electron/services/fileSystemToolHost.test.ts electron/services/skillWorkspaceService.test.ts` |
| Type Check | ✅ `cd crewagent-runtime && npx tsc --noEmit --pretty false` |
| Lint | ✅ `cd crewagent-runtime && npx eslint electron/services/fileSystemToolHost.ts electron/services/fileSystemToolHost.test.ts electron/services/skillActivation.ts electron/services/toolHost.ts electron/services/skillWorkspaceService.ts electron/services/skillWorkspaceService.test.ts electron/services/terminalService.ts` |
| Review Outcome | Approved |

---

## Resolved Findings

### R1 - `node.run` / `npm.install` 不再继承完整宿主环境变量

**What changed**  
- `executeBundledNodeProcess()` 改为复用 Story 5.19 的 `buildExecutionEnv(undefined, policy)`，不再把完整 `process.env` 传给 bundled Node 子进程。
- `SkillWorkspaceService.npmInstall()` 也改成从同一套最小基础环境出发，再叠加 npm registry / proxy 配置。

**Evidence**  
- `node.run` env 隔离： [fileSystemToolHost.ts](/Users/mengbin/code/GPT/CrewAgent/crewagent-runtime/electron/services/fileSystemToolHost.ts#L1972)
- `npm.install` env 隔离： [skillWorkspaceService.ts](/Users/mengbin/code/GPT/CrewAgent/crewagent-runtime/electron/services/skillWorkspaceService.ts#L190)
- 共享 env helper： [terminalService.ts](/Users/mengbin/code/GPT/CrewAgent/crewagent-runtime/electron/services/terminalService.ts#L134)
- 回归测试： [fileSystemToolHost.test.ts](/Users/mengbin/code/GPT/CrewAgent/crewagent-runtime/electron/services/fileSystemToolHost.test.ts#L896)

**Assessment**  
- 通过 `process.env` 重新暴露宿主 secrets 的回归已修复。
- `node.run` / `npm.install` 与 `terminal.run` / `shell.exec` 现在使用同一条 env 隔离基线。

---

### R2 - skill workspace 不再通过删除共享 `source-shadow` 破坏并发执行

**What changed**  
- `SkillWorkspaceService.ensureWorkspace()` 不再 `rm -rf` 共享 shadow 目录后重建，而是为每次 ensure 创建新的 `source-shadow/<uuid>/`。
- 这样相同 skill 的并发 run 拿到的是不同 shadow 路径，不会互相删除执行中的 source tree。

**Evidence**  
- 新的 shadow 生成逻辑： [skillWorkspaceService.ts](/Users/mengbin/code/GPT/CrewAgent/crewagent-runtime/electron/services/skillWorkspaceService.ts#L80)
- 回归测试： [skillWorkspaceService.test.ts](/Users/mengbin/code/GPT/CrewAgent/crewagent-runtime/electron/services/skillWorkspaceService.test.ts#L118)

**Assessment**  
- 共享 `source-shadow` 的 destructive rebuild 问题已解除。
- 当前实现更接近设计里“workspace 可复用，但执行 shadow 可刷新”的目标。

---

## Acceptance Criteria Status

| AC | Status | Notes |
|----|--------|-------|
| AC-1 Python and Node execution | ✅ | `python.run` 支持 `module`；`node.run` 支持 `code` / `file` / `module`；声明过的 `@skill` 文件可执行 |
| AC-2 npm dependency install | ✅ | `npm.install` 写入 isolated workspace；缺包时 `node.run` 触发一次 auto-install retry |
| AC-3 Bundled runtime bridge | ✅ | 继续复用 bundled Python / bundled Node/npm / terminal bridge；错误保持结构化 |

---

## Residual Risk / Test Gap

- 这次 follow-up 已经补上 env isolation 和 fresh shadow 的回归。
- 仍未补真正的多进程并发执行集成测试；当前并发风险主要通过 workspace service 的 path-level regression 证明已缓解。
- Story 5.19 依赖的跨平台实机验证仍是单独跟踪项，但不再构成 13.4 的代码级阻塞。

---

## Conclusion

- No blocking findings remain for Story 13.4.
- Story 13.4 can be closed as `done`.
