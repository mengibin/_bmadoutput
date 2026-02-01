# Epic 10: Runtime License Verification (Hardware-Bound)

**Goal**: 在 Runtime 端实现硬件绑定 + 离线激活许可体系，并提供 Builder 离线发卡工具；为未来 SaaS 在线发卡预留接口与扩展点。

**PRD Reference**: `_bmad-output/prd-runtime-license-verification.md`

**Scope**:
- Runtime 硬件绑定 + Builder 离线发卡（不包含 SaaS 发卡）

---

## Epic Overview

当前 Runtime 缺乏有效授权机制，无法限制试用期、无法防止激活码被共享，且无法满足离线客户需求。本 Epic 通过：
- **硬件指纹** 生成 Machine ID
- **非对称签名** 的离线 License Key 验证
- **试用期** 与 **功能锁定**

实现可控、可售、可离线的授权体系。

---

## Design Decisions (from PRD)

1. **Fingerprint**: 使用 `node-machine-id` 生成 Machine ID。
2. **License Payload**: 包含 `machineId`, `issuedAt`, `expiresAt`, `type`, `customerName?`。
3. **Crypto**: Phase 1 使用 RSA/Ed25519，私钥仅在 Builder，公钥内置于 Runtime。
4. **Offline Support**: Runtime 离线可运行已激活许可证。
5. **Tamper Resistance**: 首次运行时间 + 最后活跃时间持久化，检测时间回拨。

---

## Story 10.1: Runtime Trial Gate + First Run Timestamp

**Priority**: Phase 1  
**Status**: `planned`

### Goal
为 Runtime 增加 15 天试用期控制，过期后锁定核心功能，仅保留激活入口与基础设置/导出功能。

### Acceptance Criteria
1. 首次启动记录不可篡改的 `firstRunAt`。
2. UI 显示试用期剩余天数。
3. 试用期过期后，拦截 Agent/Workflow 执行入口。
4. 保留激活界面与基础设置/导出功能可用。

### Files to Modify (draft)
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.tsx`
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx`

---

## Story 10.2: Machine ID Generation + UI Display

**Priority**: Phase 1  
**Status**: `ready-for-dev`

### Goal
在激活页面显示 Machine ID，并支持复制。

### Acceptance Criteria
1. 自动采集硬件指纹并生成 Machine ID。
2. UI 显示 Machine ID，并提供一键复制。
3. 失败时显示可读错误提示。

### Files to Modify (draft)
- `crewagent-runtime/electron/services/licenseService.ts` (NEW)
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx`

---

## Story 10.3: Offline License Verification (Public Key)

**Priority**: Phase 1  
**Status**: `ready-for-dev`

### Goal
支持输入 License Key，使用公钥验签，校验 Machine ID 和过期时间，验证通过后解锁功能。

### Acceptance Criteria
1. 输入 License Key 并验签。
2. 解密 Payload，校验 `machineId` 与本机一致。
3. 校验 `expiresAt` 是否过期。
4. 校验时间回拨（`lastActiveAt > now` 判为无效）。
5. 通过后持久化激活状态并解锁。

### Files to Modify (draft)
- `crewagent-runtime/electron/services/licenseService.ts` (NEW)
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx`

---

## Story 10.4: Builder Offline License Generator

**Priority**: Phase 1  
**Status**: `ready-for-dev`

### Goal
在 Builder 中提供 License 生成器，输入 Machine ID 输出 License Key。

### Acceptance Criteria
1. Builder 提供 License Generator 工具入口。
2. 输入 Machine ID 后生成签名 License Key。
3. 支持设置有效期（永久或固定天数）。

### Files to Modify (draft)
- `crewagent-builder-frontend/...` (新增页面/工具)
- `crewagent-builder-backend/...` (签名逻辑 + 私钥存储)

---

## Story 10.5: Anti-Tamper Persistence

**Priority**: Phase 1  
**Status**: `ready-for-dev`

### Goal
防止用户篡改试用期或激活状态。

### Acceptance Criteria
1. `firstRunAt` / `lastActiveAt` 持久化到系统级位置或隐藏文件。
2. 检测时间回拨时触发锁定。

---

## Epic Risks & Mitigations

| Risk | Mitigation |
|:--|:--|
| 时间回拨绕过 | 记录 `lastActiveAt` 并检测回拨 |
| 私钥泄露 | 私钥仅在 Builder/Server，代码中不落地 |
| 离线客户安装复杂 | UI 给予 Machine ID 复制 + 清晰提示 |

---

## Validation Checklist

- Story 10.1 ~ 10.5 可在离线环境使用
- 试用期到期后核心功能锁定
- License Key 验证包含 Machine ID 与过期校验
