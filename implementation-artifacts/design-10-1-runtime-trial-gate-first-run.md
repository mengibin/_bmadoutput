# Design: Story 10.1 Runtime Trial Gate + First Run Timestamp

**Story:** `10-1-runtime-trial-gate-first-run.md`  
**Design principle:** Minimal UI change, runtime-side enforcement

---

## Design Goals
1. Enforce 15-day trial with clear countdown.
2. Block core execution when trial expires.
3. Detect basic time rollback.

---

## UX / UI

### Trial Banner + Activation Entry (Placement Update)
- Location: **Settings 卡片中的版本号（例如 v0.2.2）右侧**，与版本号同一行并靠右显示。
- Layout: `版本号`（左） + `Trial Badge`（中） + `Activate` 入口（右）。
- Copy: "Trial remaining: X days"
- Style:
  - Badge: pill 样式、浅色背景、渐变边框、字体加粗。
  - Activate: 轻量按钮（ghost/outline），与 badge 视觉平衡。
  - 当 <3 天时，badge 使用更高对比度（warning 主题）。

示意（文本）：
```
[Settings  ...]      v0.2.2   [Trial: 12 days]   [Activate]
```

### Expired State
- Replace Run/Agent entry actions with a blocking notice.
- Show activation CTA button leading to activation screen.

---

## Data Model
```
TrialState {
  firstRunAt: number
  lastActiveAt: number
}
```

---

## Runtime Logic
1. On startup, load `firstRunAt` and `lastActiveAt` from durable store.
2. If `firstRunAt` is missing, initialize it.
3. If `lastActiveAt > now`, mark trial invalid (tamper).
4. Compute remaining days = 15 - floor((now - firstRunAt)/86400s).
5. When remaining <= 0, block execution.

---

## Affected Files
- `crewagent-runtime/electron/stores/runtimeStore.ts` (persist trial timestamps)
- `crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.tsx` (block execution)
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx` (banner + activation link)

---

## Test Checklist
1. First launch -> `firstRunAt` saved.
2. Trial active -> banner shows remaining days.
3. Trial expired -> run blocked.
4. Time rollback -> activation forced.
