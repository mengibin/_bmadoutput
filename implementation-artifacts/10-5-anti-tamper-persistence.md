# Story 10.5: Anti-Tamper Persistence

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. Design complete. -->

## Story

As a **Vendor**,
I want Runtime to resist time rollback and reinstallation tricks,
So that trial and license enforcement cannot be easily bypassed.

## Acceptance Criteria

### 1. Durable Timestamps
- **Given** Runtime starts
- **Then** it reads `firstRunAt` and `lastActiveAt` from a durable location
- **And** it updates `lastActiveAt` on each successful run

### 2. Tamper Detection
- **Given** `lastActiveAt` exists
- **When** system time is earlier than `lastActiveAt`
- **Then** Runtime marks license state as invalid and blocks execution

### 3. Storage Location
- **Given** the system is macOS or Windows
- **Then** the timestamp is stored in a protected location (Keychain/Registry or hidden file)

### 4. License Integrity Hardening
- **Given** Runtime loads persisted `licenseState`
- **When** the `licenseKey` signature or payload fails verification
- **Then** Runtime treats the license as invalid, clears the stored payload, and keeps trial restrictions enforced

## Design

### Data
```
AntiTamperState {
  firstRunAt: number
  lastActiveAt: number
}
```

## Technical Components / Changes

| File | Change |
|:-----|:-------|
| `crewagent-runtime/electron/services/licenseService.ts` | **MOD**: read/write tamper state |
| `crewagent-runtime/electron/stores/runtimeStore.ts` | **MOD**: consume tamper state + merge trial state |
| `crewagent-runtime/electron/main.ts` | **MOD**: app 启动时重新验证持久化 licenseState |

## Dependencies
- Story 10.1 (Trial Gate)
- Story 10.3 (License Verification)

## Verification Plan
1. Normal run updates lastActiveAt
2. Rollback system time -> run blocked
3. Delete local settings -> tamper state still detected (if stored externally)

> 设计文档：`_bmad-output/implementation-artifacts/design-10-5-anti-tamper-persistence.md`  
> 技术规格：`_bmad-output/implementation-artifacts/tech-spec-10-5-anti-tamper-persistence.md`

---

## Tasks / Subtasks

- [x] 1) Anti-Tamper 持久化存储（AC: 1,3）
  - [x] 1.1 `licenseService` 添加 anti-tamper 读写与路径策略（macOS/Windows）
  - [x] 1.2 `runtimeStore.refreshTrialState()` 读取/写入 anti-tamper state

- [x] 2) 回拨检测与锁定持久化（AC: 2）
  - [x] 2.1 `lastActiveAt` 回拨检测保持 `tamperDetected` 持久化
  - [x] 2.2 删除 settings 后仍通过 anti-tamper state 阻断

- [x] 3) 测试与验证
  - [x] 3.1 RuntimeStore anti-tamper persistence 单测
  - [x] 3.2 `vitest` 运行（runtimeStore.test.ts）

- [x] 4) License 完整性校验与清理（AC: 4）
  - [x] 4.1 启动时重新验证 `licenseKey`，失败则清理持久化状态

---

## Dev Agent Record

### Agent Model Used

GPT-5 (Codex CLI)

### Debug Log References

- `npm -C crewagent-runtime test -- runtimeStore.test.ts`（通过；npm user config `python/install` 警告）
- `npm -C crewagent-runtime test -- runtimeStore.test.ts`（通过；npm user config `python/install` 警告）
- `npm -C crewagent-runtime test`（通过；npm user config `python/install` 警告；MessageList `useLayoutEffect` SSR 警告）
- `npm -C crewagent-runtime test -- runtimeStore.test.ts`（通过；npm user config `python/install` 警告）

### Completion Notes List

- 新增 anti-tamper 读写接口，使用 `appData/CrewAgent/license.dat` 作为持久化位置。
- `refreshTrialState()` 合并外部 anti-tamper state 与 settings，防止删除 settings 绕过。
- 新增单测验证 anti-tamper 文件在删除 settings 后仍可恢复状态。
- App 启动时重新验证已持久化的 `licenseKey`，失败即清理 `licenseState` 并继续按试用规则限制。

### Manual Verification

- 已执行（2026-02-01）：验证测试通过。

---

## File List

- `_bmad-output/implementation-artifacts/10-5-anti-tamper-persistence.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `_bmad-output/implementation-artifacts/tech-spec-10-5-anti-tamper-persistence.md`
- `crewagent-runtime/electron/services/licenseService.ts`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/stores/runtimeStore.test.ts`
- `crewagent-runtime/electron/main.ts`

---

## Change Log

- Runtime：增加 anti-tamper 外部持久化与回拨检测合并逻辑；补充单测覆盖；启动时重新验证持久化 licenseState 并在失败时清理。
