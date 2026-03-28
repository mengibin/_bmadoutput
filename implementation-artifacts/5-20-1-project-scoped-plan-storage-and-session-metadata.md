# Story 5-20-1: Project-Scoped Plan Storage and Session Metadata

## Overview
**Epic**: 5 – Runtime Chat Plan Mode
**Priority**: High
**Status**: `done`

## Goal
为 Chat 会话建立项目级 Plan 工件存储基础和会话元数据模型，使 Plan 模式能够被初始化、恢复、删除和清理。

## Business Value
- **项目可清理**：Plan 数据归属明确，后续孤立项目清理可直接按项目删除。
- **会话可恢复**：用户重进 conversation 后能找回计划状态和工件路径。
- **后续 Story 基座**：没有这层持久化，就无法实现预览、编辑、执行和进度。

## Acceptance Criteria

### 1. 项目级 Plan 工件初始化
- **Given** 当前会话是 `chat`
- **When** 用户首次打开 `Plan` 开关
- **Then** 系统在以下目录创建 Plan 工件：

```text
@state/projects/<projectId>/plans/<conversationId>/plan.md
@state/projects/<projectId>/plans/<conversationId>/plan.state.json
@state/projects/<projectId>/plans/<conversationId>/remarks.json
```

- **And** `plan.md` 初始为空草稿
- **And** `remarks.json` 初始为空数组
- **And** `plan.state.json` 初始为 `version = 1`、`status = drafting`、`tasks = []`

### 2. 会话元数据持久化
- **Given** Plan 工件已创建
- **When** 会话元数据保存
- **Then** `ConversationMetadata` 中保存：
  - `chatSubmode = 'plan'`
  - `planStatus`
  - `planPath`
  - `planStatePath`
  - `planRemarksPath`

### 3. 恢复
- **Given** 应用重启或项目重新打开
- **When** 用户重新进入该 Chat 会话
- **Then** Plan 子模式、Plan 状态和工件路径被恢复。

### 4. Conversation 级清理
- **Given** 会话已启用 Plan
- **When** 用户删除该 conversation
- **Then** 只删除对应 `plans/<conversationId>/` 目录
- **And** 不影响同一项目内其他 conversation 的 Plan 工件。

### 5. 项目级清理兼容
- **Given** 项目被删除或进入孤立项目清理流程
- **When** 项目 runtime 数据被清理
- **Then** `@state/projects/<projectId>/...` 下的 Plan 数据可整体删除。

## Technical Components / Changes

1. **RuntimeStore**
   - 新增 Plan 工件路径构造方法
   - 新增 `ensureConversationPlanArtifacts()`
   - 新增 `deleteConversationPlanArtifacts()`
2. **ConversationMetadata**
   - 扩展 `chatSubmode` 与 Plan 相关路径字段
3. **IPC**
   - 新增 `plan:ensureArtifacts`
4. **appStore**
   - 扩展 `Conversation`
   - 增加 Plan 元数据更新动作

## Delivery Scope by Layer

### Backend Scope

- 扩展 `ConversationMetadata` 结构。
- 新增项目级 Plan 工件路径生成逻辑。
- 新增 Plan 工件初始化方法。
- 新增 conversation 删除时的 Plan 工件清理逻辑。
- 保证项目级清理流程可覆盖 Plan 工件命名空间。

### Frontend Scope

- 扩展 `Conversation` 前端类型。
- 为 Chat 会话增加 `chatSubmode` 与 Plan 元数据映射。
- 在会话加载时恢复 Plan 相关字段。
- 在开启/关闭 Plan 时触发 metadata 持久化。

### Integration Scope

- 开启 Plan 开关后，前端能拿到初始化后的工件路径。
- 重进会话后，前后端对 Plan 元数据的理解一致。
- 删除 conversation 后，前端列表和磁盘状态一致。

## Dependencies
- 依赖现有 Story 5-8 Conversation 持久化能力。
- 为 Story 5-20-2 / 5-20-3 / 5-20-4 提供底座。

## Related Artifacts
- PRD: `_bmad-output/prd-runtime-chat-plan-mode.md`
- Epic: `_bmad-output/epics-runtime-chat-plan-mode.md`
- Design: `_bmad-output/implementation-artifacts/design-5-20-chat-plan-mode.md`
- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-5-20-chat-plan-mode.md`

## Verification Plan

### Manual Verification
1. 新建 Chat 会话，打开 Plan，确认 3 个工件文件创建成功。
2. 关闭应用并重新打开，确认会话中 Plan 状态恢复。
3. 删除当前 conversation，确认只删除该 conversation 的 Plan 子目录。
4. 用同项目多个会话验证隔离性。

### Automated Tests
- RuntimeStore 单测：Plan 工件初始化与删除。
- Store/IPC 单测：Conversation metadata 持久化和恢复。

## Risks / Mitigations

| Risk | Description | Mitigation |
|:---|:---|:---|
| R1 | `projectId` 与实际 runtime 目录键不一致 | 统一复用现有 runtime 的稳定 `projectId` 解析逻辑 |
| R2 | 删除 conversation 时遗漏 Plan 子目录 | 在删除 conversation 的单测中强制检查目录删除 |
| R3 | 旧会话缺少新字段导致恢复异常 | 所有新字段使用可选字段并设置默认值 |

## Acceptance Checklist

- [x] 首次打开 Plan 开关后创建 3 个工件文件
- [x] 工件路径采用 `@state/projects/<projectId>/plans/<conversationId>/...`
- [x] `plan.md` 初始为空草稿
- [x] `remarks.json` 初始为空数组
- [x] `plan.state.json` 初始为 `version = 1`、`status = drafting`、`tasks = []`
- [x] 会话 metadata 成功保存 `chatSubmode` 与 Plan 路径
- [x] 重启应用后会话能恢复 Plan 状态
- [x] 删除 conversation 只删除该 conversation 的 Plan 目录
- [x] 项目级清理流程可覆盖整个项目命名空间

## Sequencing Notes

- 本 Story 完成前，不建议进入 Plan 面板 UI 实现。
- 本 Story 是 5-20 全部后续 Story 的前置门槛。

## Tasks / Subtasks
- [x] Backend-1 扩展 `ConversationMetadata` 与 `Conversation` 类型定义
- [x] Backend-2 增加项目级 Plan 路径生成与初始化逻辑
- [x] Backend-3 增加 `plan.state.json` / `remarks.json` 默认内容生成
- [x] Backend-4 增加 conversation 删除联动的 Plan 目录清理
- [x] Frontend-1 增加 `chatSubmode` 和 Plan metadata 的 store 映射
- [x] Frontend-2 增加 Plan 开关触发 metadata 更新
- [x] Integration-1 验证初始化后前端能读取到 artifact paths
- [x] Test-1 补 RuntimeStore 初始化/删除单测
- [x] Test-2 补 Conversation 恢复与清理集成测试

### Review Follow-ups (AI)

- [x] [AI-Review][MEDIUM] `plan:ensureArtifacts` 仅允许对真实存在且当前 `activeType = 'chat'` 的 conversation 初始化，避免生成脱离 conversation index 的孤立 Plan 目录（`crewagent-runtime/electron/stores/runtimeStore.ts`, `crewagent-runtime/electron/main.ts`）
- [x] [AI-Review][MEDIUM] 重新开启 Plan Mode 时不再把已有 `plan.state.json.status` 错误重置为 `drafting`，而是回填当前持久化状态（`crewagent-runtime/electron/stores/runtimeStore.ts`, `crewagent-runtime/src/pages/WorksPage/WorksPage.tsx`）
- [x] [AI-Review][MEDIUM] 补齐自动化证据：覆盖 metadata 持久化/恢复、非法 conversation 初始化拦截，以及 orphan project 清理覆盖 Plan 命名空间（`crewagent-runtime/electron/stores/runtimeStore.test.ts`, `crewagent-runtime/src/stores/appStore.test.ts`）

## File List

- `_bmad-output/implementation-artifacts/5-20-1-project-scoped-plan-storage-and-session-metadata.md`
- `_bmad-output/implementation-artifacts/code-review-5-20-chat-plan-mode.md`
- `_bmad-output/implementation-artifacts/validation-report-story-5-20-1.md`
- `crewagent-runtime/shared/planMode.ts`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/stores/runtimeStore.test.ts`
- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/preload.ts`
- `crewagent-runtime/electron/electron-env.d.ts`
- `crewagent-runtime/src/stores/appStore.ts`
- `crewagent-runtime/src/stores/appStore.test.ts`
- `crewagent-runtime/src/hooks/useConversationWorkspace.ts`
- `crewagent-runtime/src/pages/WorksPage/WorksPage.tsx`
- `crewagent-runtime/src/vite-env.d.ts`

## Dev Agent Record

### Agent Model Used

GPT-5 Codex

### Debug Log References

- `crewagent-runtime`: `npm test -- electron/stores/runtimeStore.test.ts src/stores/appStore.test.ts`
- `crewagent-runtime`: `npx tsc --noEmit`

### Completion Notes List

- 新增 `shared/planMode.ts` 统一定义 Plan 工件路径、默认状态、remarks 与状态解析。
- `plan.md` 初始化逻辑已更新为空白草稿，不再预写默认模板正文。
- `RuntimeStore.ensureConversationPlanArtifacts()` 现已校验 conversation 存在且当前为 chat，并返回持久化的 `planStatus`。
- conversation metadata 已扩展 `chatSubmode / planStatus / planPath / planStatePath / planRemarksPath`，前后端均可持久化与恢复。
- conversation 删除会清理对应 `plans/<conversationId>/`；orphan project 清理会删除整个项目 runtime 根目录，从而覆盖 Plan 命名空间。
- 新增 `appStore` 恢复/持久化测试与 `RuntimeStore` Plan 生命周期测试，补齐 review 指出的证据缺口。

## Change Log

- 2026-03-24: 完成 Story 5-20-1 的项目级 Plan 工件、metadata 扩展、IPC/store 接线与删除清理；状态更新为 `review`。
- 2026-03-24: Senior Developer Review 首轮发现 3 个 MEDIUM 问题；已记录为 `Review Follow-ups (AI)` 并回退整改。
- 2026-03-24: 修复 review follow-ups（conversation 校验、状态保持、测试补强），复跑 `npm test -- electron/stores/runtimeStore.test.ts src/stores/appStore.test.ts` 与 `npx tsc --noEmit` 通过；状态恢复为 `review`。
- 2026-03-25: 根据更新后的产品规则，将 `plan.md` 默认内容改为空白草稿，并新增初始化内容断言测试；复跑 `./node_modules/.bin/vitest run electron/stores/runtimeStore.test.ts src/stores/appStore.test.ts` 与 `./node_modules/.bin/tsc --noEmit --pretty false` 通过；状态保持为 `review`。
- 2026-03-28: 按 BMAD sprint closing 约定关闭 Story；结合 Senior Developer Review `Approved` 结论，将状态更新为 `done`。

## Senior Developer Review (AI)

**Date:** 2026-03-24  
**Outcome:** Approved

### Summary

| Metric | Value |
|--------|-------|
| Git vs Story Discrepancies | 0（仅评估 Story 5-20-1 范围内实现） |
| HIGH Issues | 0 |
| MEDIUM Issues | 0 |
| LOW Issues | 0 |
| Tests | ✅ `npm test -- electron/stores/runtimeStore.test.ts src/stores/appStore.test.ts` passed |
| Type Check | ✅ `npx tsc --noEmit` passed |

### Review History

- 2026-03-24: Changes Requested（M1: conversation/type 校验缺失；M2: Plan status 被错误重置；M3: metadata/recovery/cleanup 自动化证据不足）
- 2026-03-24: Approved（上述问题已修复并补齐测试）
