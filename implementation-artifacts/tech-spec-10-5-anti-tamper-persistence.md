# Tech Spec: Story 10.5 Anti-Tamper Persistence

## 1. Overview
Persist trial/license timestamps in a tamper-resistant location and detect system time rollback.

## 2. Storage Strategy
- macOS: Keychain or hidden file under `~/Library/Application Support/CrewAgent/license.dat`
- Windows: Registry or `%APPDATA%/CrewAgent/license.dat`

## 3. Data
```
AntiTamperState {
  firstRunAt: number
  lastActiveAt: number
}
```

## 4. Flow
1. On startup, read anti-tamper state.
2. If `lastActiveAt > now`, mark tamper and block execution.
3. Update `lastActiveAt` on successful run.
4. Do not overwrite `firstRunAt` once set.
5. On startup, re-verify persisted `licenseKey`; clear `licenseState` on failure.

## 5. Files & Changes

| File | Change |
|:-----|:-------|
| `crewagent-runtime/electron/services/licenseService.ts` | **MOD**: read/write anti-tamper state |
| `crewagent-runtime/electron/stores/runtimeStore.ts` | **MOD**: consume tamper state |
| `crewagent-runtime/electron/main.ts` | **MOD**: re-verify persisted license state on startup |

## 6. Verification
1. Normal run updates `lastActiveAt`.
2. Rollback time -> tamper mode.
3. Delete app settings -> still detects if stored externally.
4. Tampered/invalid license -> persisted licenseState cleared.
