# Story 10.1: Runtime Trial Gate + First Run Timestamp

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **Consumer**,
I want Runtime to enforce a 15-day trial with clear remaining days and lock core execution when expired,
So that licensing can be enforced before commercial activation.

## Acceptance Criteria

### 1. First Run Timestamp
- **Given** I launch Runtime for the first time
- **When** the app initializes settings
- **Then** it records an immutable `firstRunAt` timestamp in a durable store
- **And** the value is reused on subsequent launches (not reset)

### 2. Trial Countdown Banner
- **Given** the trial period is active
- **When** I open the Runtime UI
- **Then** I see "Trial remaining: X days" in a visible location

### 3. Trial Expired Lock
- **Given** the trial period has ended
- **When** I try to run an Agent or Workflow
- **Then** the action is blocked and I am redirected to the activation screen
- **And** all functionality is locked except activation

### 4. Time Rollback Detection (basic)
- **Given** Runtime has a stored `lastActiveAt`
- **When** system time is earlier than `lastActiveAt`
- **Then** Runtime treats it as tampering and forces activation
- **And** the lock persists until activation (even if time is corrected)

## Design

### Summary
- 首次启动写入不可变 `firstRunAt`，后续仅更新 `lastActiveAt`。
- 试用期 15 天，剩余天数按 `ceil((trialEndAt - now) / dayMs)` 计算，最小值为 0。
- 若检测到系统时间早于 `lastActiveAt` / `firstRunAt`，视为回拨并强制进入激活流程，锁定持久化。
- 全局锁定：屏蔽所有功能与导航，仅保留激活入口。
- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-10-1-runtime-trial-gate-first-run.md`

### UX / UI
- Workspace（Works/Conversation）顶部显示试用 Banner：`Trial remaining: X days`。
- 试用过期/回拨：全局 BlockingNotice 覆盖，按钮「去激活」跳转 Settings 激活区。
- Settings 顶部显示同样的试用 Banner；锁定状态下仅显示激活入口与提示。

### API / Contracts
```typescript
interface TrialState {
  firstRunAt: number
  lastActiveAt: number
  tamperDetected?: boolean
}

interface RuntimeSettings {
  trialState?: TrialState
}
```

### Data / Storage
- `trialState` 持久化在 `app.getPath('userData')/runtime-store/settings.json`。
- `firstRunAt` 一旦写入不再覆盖；`lastActiveAt` 在启动与周期刷新时更新。

### Errors / Edge Cases
- settings 缺失/损坏：以当前时间初始化 `firstRunAt`/`lastActiveAt`。
- `now < lastActiveAt` 或 `now < firstRunAt`：标记 `tamperDetected` 并强制激活。
- 试用过期仍尝试运行：阻断 `startConversationRun()` 并跳转 Settings。
- 删除 settings 会重置试用（强防篡改由 10.5 处理）。
- 回拨触发后即使系统时间恢复正常也保持锁定，直到激活成功。

### Test Plan
- 首次启动写入 `firstRunAt`，重启后保持不变。
- 试用期内 Banner 显示剩余天数，边界天数计算正确。
- 试用过期/回拨时运行入口被阻断并跳转激活页。

## Technical Components / Changes

| File | Change |
|:-----|:-------|
| `crewagent-runtime/electron/stores/runtimeStore.ts` | **MOD**: store `firstRunAt`, `lastActiveAt` |
| `crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.tsx` | **MOD**: block run entry points |
| `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx` | **MOD**: show trial banner + activation entry |

## Tasks / Subtasks

- [x] 1) Trial 状态持久化与回拨检测（AC: 1,4）
  - [x] 1.1 在 `RuntimeSettings` 增加 `trialState` 类型（`firstRunAt/lastActiveAt/tamperDetected`）
  - [x] 1.2 启动时初始化 `firstRunAt`，仅首次写入；更新 `lastActiveAt`
  - [x] 1.3 检测 `now < lastActiveAt/firstRunAt` 时写入 `tamperDetected = true`（持久化）

- [x] 2) Renderer 侧读取与试用状态计算（AC: 1,2,3,4）
  - [x] 2.1 `settings:get` 返回包含 `trialState`
  - [x] 2.2 `appStore.initialize()` 保存 `trialState`，并提供 `trialRemainingDays/expired/locked` 的计算结果

- [x] 3) 全局锁定与导航屏蔽（AC: 3,4）
  - [x] 3.1 AppShell/Sidebar：锁定时禁用所有导航项与交互
  - [x] 3.2 全局 BlockingNotice 覆盖层，仅保留“去激活”入口

- [x] 4) 运行入口阻断 + 激活引导（AC: 3,4）
  - [x] 4.1 `startConversationRun()` 前检测锁定，阻断并跳转 Settings 激活区
  - [x] 4.2 Works/Workspace 显示锁定提示（不可绕过）

- [x] 5) 试用 Banner 显示（AC: 2）
  - [x] 5.1 Workspace 顶部显示 `Trial remaining: X days`
  - [x] 5.2 Settings 顶部显示 `Trial remaining: X days`

- [x] 6) 测试与验证
  - [x] 6.1 RuntimeStore 单测：首次写入、回拨锁定持久化
  - [x] 6.2 手动验证：试用期、过期锁定、回拨后锁定不自动恢复

## Dev Notes

- `trialState` 暂存 `settings.json`，后续 10.5 再迁移到系统级持久化。
- 锁定逻辑必须**全局统一**，避免只拦截某个入口导致绕过。
- 试用天数使用 `ceil` 计算（首日显示 15 天）。
- Renderer 每 60s 刷新 trial 状态，Main IPC 统一校验并阻断 runs/chat。

### Project Structure Notes

- 状态读写集中在 `RuntimeStore`，UI 侧只读并派生 `remainingDays/locked`。
- `BlockingNotice` 已存在，可复用为全屏阻断层。

### References

- `_bmad-output/prd-runtime-license-verification.md`
- `_bmad-output/implementation-artifacts/10-runtime-license-verification.md`
- `_bmad-output/implementation-artifacts/tech-spec-10-1-runtime-trial-gate-first-run.md`

## Dev Agent Record

### Agent Model Used

GPT-5 (Codex CLI)

### Debug Log References

- `npm -C crewagent-runtime test -- runtimeStore.test.ts`（警告：npm user config `python/install`）
- `npm -C crewagent-runtime test`（警告：npm user config `python/install`，以及 MessageList SSR `useLayoutEffect` 警告）

### Completion Notes List

- 实现 `trialState` 持久化（`firstRunAt/lastActiveAt/tamperDetected`）并在时间回拨时持久锁定。
- Renderer 侧计算 `remaining/expired/locked`，并在运行入口阻断启动。
- 全局锁定覆盖层 + Sidebar 禁用非激活导航；Settings 仅保留激活入口。
- Workspace / Settings 显示试用期 Banner。
- 添加 RuntimeStore 单测覆盖首次写入与回拨锁定持久化。
- Main IPC 增加 trial 刷新与 runs/chat 全面阻断，Renderer 周期刷新锁定状态。
- 手动验证已完成（试用期/过期锁定/回拨持久锁定通过）。
- 试用期 Banner 移动到 Sidebar（Settings 区域），Workspace/Settings 页面不再显示。

### Manual Verification

- 试用期内显示剩余天数 Banner。
- 试用过期后运行入口被阻断并跳转到激活页。
- 系统时间回拨后保持锁定，恢复时间不自动解除。

### File List

- `_bmad-output/implementation-artifacts/10-1-runtime-trial-gate-first-run.md`
- `_bmad-output/implementation-artifacts/tech-spec-10-1-runtime-trial-gate-first-run.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/preload.ts`
- `crewagent-runtime/electron/services/agentSessionContract.ts`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/stores/runtimeStore.test.ts`
- `crewagent-runtime/src/stores/appStore.ts`
- `crewagent-runtime/src/components/layout/AppShell.tsx`
- `crewagent-runtime/src/components/layout/AppShell.css`
- `crewagent-runtime/src/components/layout/Sidebar.tsx`
- `crewagent-runtime/src/pages/WorksPage/WorksPage.tsx`
- `crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.tsx`
- `crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.css`
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx`
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.css`
- `crewagent-runtime/src/vite-env.d.ts`

### Change Log

- Runtime：新增试用期持久化、回拨锁定与全局激活阻断。
- Runtime：补充试用状态刷新与 runs/chat IPC 统一锁定。
- UI：试用期 Banner 移动到 Sidebar（Settings 区域），Workspace/Settings 页面不再显示。
- UI：AppShell 定时刷新试用状态以更新锁定覆盖层。
- Tests：补充 trialState 初始化/刷新与回拨锁定持久化单测。

## Dependencies
- Epic 10 (License Verification)

## Verification Plan
1. Fresh install -> first run timestamp created
2. Trial active -> banner visible
3. Trial expired -> run blocked, activation shown
4. System time rollback -> forced activation and persistent lock
