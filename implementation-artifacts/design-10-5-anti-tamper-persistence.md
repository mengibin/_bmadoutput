# Design: Story 10.5 Anti-Tamper Persistence

**Story:** `10-5-anti-tamper-persistence.md`

---

## Design Goals
1. Persist trial/license timestamps in a tamper-resistant location.
2. Detect time rollback consistently.

---

## Storage Strategy
- macOS: Keychain or hidden file under `~/Library/Application Support/...`
- Windows: Registry or `%APPDATA%/...` hidden file

---

## Flow
1. On startup, read `firstRunAt` and `lastActiveAt`.
2. If `lastActiveAt > now`, flag tamper.
3. Update `lastActiveAt` on successful run.

---

## Affected Files
- `crewagent-runtime/electron/services/licenseService.ts`
- `crewagent-runtime/electron/stores/runtimeStore.ts`

---

## Test Checklist
1. Normal run updates timestamps.
2. Rollback time -> blocked.
3. Deleting app settings does not bypass if stored externally.
