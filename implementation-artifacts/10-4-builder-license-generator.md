# Story 10.4: Builder Offline License Generator

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. Design complete. -->

## Story

As an **Admin/Reseller**,
I want a Builder tool that generates offline license keys from Machine ID,
So that I can issue licenses without SaaS dependencies.

## Acceptance Criteria

### 1. Generator UI
- **Given** I open Builder tools
- **When** I select License Generator
- **Then** I can input Machine ID, customer name, and expiry

### 2. Key Generation
- **Given** valid inputs
- **When** I click Generate
- **Then** a signed License Key is produced

### 3. Copy/Export
- **Given** a License Key is generated
- **Then** I can copy it (and optionally export to a file)

## Design

### UX / UI
- Simple form + Generate button
- Output area with copy button

## Technical Components / Changes

| File | Change |
|:-----|:-------|
| `crewagent-builder-frontend/...` | **MOD/NEW**: License Generator UI |
| `crewagent-builder-backend/...` | **MOD/NEW**: sign payload with private key |
| `crewagent-builder-backend/config` | **MOD**: private key storage strategy |

## Dependencies
- Story 10.3 (License Verification)

## Verification Plan
1. Generate key -> produces non-empty signed token
2. Verify key in Runtime -> activation succeeds

> 设计文档：`_bmad-output/implementation-artifacts/design-10-4-builder-license-generator.md`  
> 技术规格：`_bmad-output/implementation-artifacts/tech-spec-10-4-builder-license-generator.md`

---

## Tasks / Subtasks

- [x] 1) 后端许可证生成与签名服务（AC: 2）
  - [x] 1.1 新增请求/响应 schema 与 payload 组装
  - [x] 1.2 读取私钥（ENV/文件）并用 Ed25519 签名
  - [x] 1.3 新增 `/license/generate` 端点（需认证）并返回 licenseKey

- [x] 2) Builder Tools UI 与 License Generator 页面（AC: 1,3）
  - [x] 2.1 Dashboard 增加 License Generator 入口
  - [x] 2.2 页面表单支持 Machine ID / Customer / Expiry
  - [x] 2.3 输出区提供 Copy 与 Download

- [x] 3) 测试与校验
  - [x] 3.1 后端生成接口测试（签名校验）
  - [x] 3.2 前端测试运行（npm test）

---

## Dev Agent Record

### Agent Model Used

GPT-5 (Codex CLI)

### Debug Log References

- `pytest`（通过；passlib `crypt` deprecation warning）
- `pytest tests/test_license_generator.py`（通过；passlib `crypt` deprecation warning）
- `npm -C crewagent-builder-frontend test`（通过；npm user config `python/install` 警告）

### Completion Notes List

- 新增 License 生成服务：使用 Ed25519 私钥对 payload 签名，输出 `payload.signature` 格式。
- 新增 `/license/generate` 后端接口（认证保护），并提供私钥 ENV/文件配置策略。
- Dashboard 增加 Builder Tools 入口；新增 License Generator 页面表单与输出区（复制/下载）。
- 新增后端接口测试，验证 payload 与签名可被公钥校验。
- License 字符串编码调整为标准 base64（与 10.3 规范一致）。

### Manual Verification

- 已完成：在 Builder License Generator 页面生成 License Key，并验证复制与下载流程正常。

---

## File List

- `_bmad-output/implementation-artifacts/10-4-builder-license-generator.md`
- `crewagent-builder-backend/.env.example`
- `crewagent-builder-backend/.gitignore`
- `crewagent-builder-backend/app/config.py`
- `crewagent-builder-backend/app/main.py`
- `crewagent-builder-backend/app/routers/license.py`
- `crewagent-builder-backend/app/schemas/license.py`
- `crewagent-builder-backend/app/services/license_service.py`
- `crewagent-builder-backend/config/README.md`
- `crewagent-builder-backend/requirements.txt`
- `crewagent-builder-backend/tests/test_license_generator.py`
- `crewagent-builder-frontend/src/app/dashboard/page.tsx`
- `crewagent-builder-frontend/src/app/tools/license-generator/page.tsx`

---

## Change Log

- Builder Backend：新增 License 生成与签名服务、`/license/generate` 接口与私钥配置。
- Builder Frontend：新增 License Generator 工具入口与表单/复制/下载输出区。
