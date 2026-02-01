# Tech-Spec: Story 10.1 Runtime Trial Gate + First Run Timestamp

**Created:** 2026-01-31
**Status:** Ready for Development
**Source Story (if applicable):** _bmad-output/implementation-artifacts/10-1-runtime-trial-gate-first-run.md

## Overview

### Problem Statement
Runtime 目前缺少试用期与时间防篡改控制，无法限制 15 天游试用与过期锁定，也无法在系统时间回拨时阻止绕过。

### Solution
在 Runtime 启动时持久化 `firstRunAt` 与 `lastActiveAt`，并基于 `firstRunAt + 15 天` 计算剩余天数（`ceil`）；当试用过期或检测到时间回拨时，进入**持久锁定**状态（直到激活成功），并屏蔽所有功能，仅允许进入激活页面。

### Scope (In/Out)
**In Scope**
- 首次启动写入不可变 `firstRunAt`，后续复用
- 每次启动更新 `lastActiveAt`
- 15 天试用倒计时与 UI Banner
- 试用过期/时间回拨时阻断运行并跳转激活入口

**Out of Scope**
- Machine ID 生成（Story 10.2）
- License Key 验签与激活（Story 10.3）
- 系统级持久化/强防篡改（Story 10.5）

## Context for Development

### Codebase Patterns
- Electron 主进程通过 `RuntimeStore` 在 `app.getPath('userData')/runtime-store` 下持久化 JSON 文件（`settings.json` 等）。
- Renderer 通过 `ipcRenderer.getSettings()` 拉取设置并写入 `appStore.initialize()`。
- 运行入口在 `ConversationView` 中调用 `startConversationRun()`，内部使用 `ipcRenderer.createRun()`。
- 通用阻断 UI 使用 `BlockingNotice` 组件。

### Files to Reference
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/src/stores/appStore.ts`
- `crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.tsx`
- `crewagent-runtime/src/pages/WorksPage/WorksPage.tsx`
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx`
- `crewagent-runtime/src/components/common/BlockingNotice.tsx`

### Technical Decisions
- `trialState` 存放在 `settings.json`（后续可迁移到独立 license store）。
- 时间均使用 `Date.now()` 的 epoch ms。
- 计算方式：
  - `trialEndAt = firstRunAt + 15 * 24 * 60 * 60 * 1000`
  - `remainingDays = max(0, ceil((trialEndAt - now) / dayMs))`（首日显示 15 天）
- 回拨检测：`now < lastActiveAt` 或 `now < firstRunAt` 视为篡改，设置 `tamperDetected = true` 并进入**持久锁定**（即使时间改回也不自动恢复）。
- 锁定策略：全局阻断 + 仅保留激活入口（Settings 中的 Activation 区域）。

## Implementation Plan

### Tasks
- [ ] Task 1: RuntimeStore 初始化/加载 settings 时建立 `trialState`（firstRunAt/lastActiveAt/tamperDetected）并持久化。
- [ ] Task 2: `settings:get` 返回包含 `trialState`；`appStore.initialize()` 保存并提供给 UI。
- [ ] Task 3: 全局锁定策略：过期/回拨时屏蔽所有功能与导航，仅保留激活入口。
- [ ] Task 4: Workspace/Conversation 运行入口检测试用状态；过期/回拨时阻断 `startConversationRun()` 并跳转 Settings 激活区。
- [ ] Task 5: Settings 页面显示试用倒计时 Banner，并提供激活入口/提示。

### Acceptance Criteria
- [ ] AC1: 首次启动写入不可变 `firstRunAt`，后续启动复用。
- [ ] AC2: 试用期内 UI 显示 “Trial remaining: X days”。
- [ ] AC3: 试用结束后进入全局锁定，所有功能不可用，仅允许进入激活页面。
- [ ] AC4: 若系统时间早于 `lastActiveAt`，视为回拨并强制激活且锁定持久化。

## Additional Context

### Dependencies
- Epic 10: Runtime License Verification
- Story 10.2 / 10.3 / 10.5

### Testing Strategy
- 单元测试：验证 `firstRunAt` 不被覆盖、`lastActiveAt` 更新、回拨检测与持久锁定逻辑。
- 手动验证：首次启动、试用期内倒计时、过期全局锁定、回拨后锁定不自动恢复。

### Notes
- 若后续引入 Dev Build bypass，需要与 10.2/10.3 的约定保持一致。

## Traceability
- Story: _bmad-output/implementation-artifacts/10-1-runtime-trial-gate-first-run.md
- Design: _bmad-output/implementation-artifacts/10-1-runtime-trial-gate-first-run.md#Design
