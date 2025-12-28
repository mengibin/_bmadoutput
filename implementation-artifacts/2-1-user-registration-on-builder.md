# Story 2.1: User Registration on Builder

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **User**,  
I want to register for an account on the Builder platform,  
so that I can save and manage my Workflow definitions.

## Acceptance Criteria

1. **Given** I am on the Builder login page  
   **When** I click "Register" and enter email, username, and password  
   **Then** my account is created via the Builder Backend (FastAPI) and stored in PostgreSQL  
   **And** I receive a JWT token and am authenticated  
   **And** I am redirected to the Dashboard.

## Design

### Summary

- 在 `crewagent-builder-frontend/src/app/login/page.tsx` 实现 Login/Register 切换（Tab 或按钮），注册表单包含 email/username/password。
- 注册请求调用 Builder Backend：`POST /auth/register`，后端写入 PostgreSQL（开发环境可用 Docker 启动本地 Postgres）。
- 注册成功后后端返回 JWT（`accessToken`），前端保存（MVP 可先 localStorage）并 `router.replace("/dashboard")`。
- `crewagent-builder-frontend/src/app/dashboard/page.tsx` 作为跳转目标先做最小占位（后续 Story 再做受保护路由/用户信息展示）。

### UX / UI

- 页面：`/login`
  - Tab：`登录` / `注册`
  - 注册表单字段：
    - Email（必填；基本格式校验）
    - Username（必填；建议 3-32；仅允许字母数字/下划线/短横线）
    - Password（必填；建议最少 8 位）
  - 交互：
    - 提交中：按钮 disabled + loading 文案
    - 成功：保存 token 后直接跳转 `/dashboard`
    - 失败：顶部或字段下方显示错误（不要暴露 key/内部堆栈）
  - 环境未配置（缺失 `NEXT_PUBLIC_API_BASE_URL`）：显示明确提示，并禁用提交

### API / Contracts

- Frontend env：
  - `NEXT_PUBLIC_API_BASE_URL`（例如：`http://localhost:8000`）
- Backend env（示例）：
  - `DATABASE_URL=postgresql+psycopg://crewagent:crewagent@localhost:5432/crewagent`
  - `JWT_SECRET=...`（HS256；非开发环境必须足够强）
  - `JWT_ALGORITHM=HS256`
  - `JWT_EXPIRES_IN_SECONDS=86400`
  - `JWT_ISSUER=crewagent-builder-backend`
  - `JWT_AUDIENCE=crewagent-builder-frontend`
  - `JWT_LEEWAY_IN_SECONDS=0`
- Backend API：
  - `POST /auth/register`
    - Request: `{ "email": string, "username": string, "password": string }`
    - Success (201): `{ "data": { "accessToken": string, "user": { "id": number, "email": string, "username": string } }, "error": null }`
    - Failure:
      - 400：字段校验失败（email 格式、username 规则、密码长度）
      - 409：邮箱或用户名已存在
      - 500：内部错误（不暴露堆栈）
  - Error body（统一结构）：
    - `{ "data": null, "error": { "code": string, "message": string, "details"?: object } }`
    - `details`（可选）：字段级错误（例如 `{ "email": "...", "username": "...", "password": "..." }`）

### Data / Storage

- PostgreSQL `users` 表（后端自研）：
  - `id`（PK）
  - `email`（unique；建议存储为 lower-case）
  - `username`（unique；规则同前端校验）
  - `password_hash`（bcrypt/argon2 等哈希；禁止明文）
  - `created_at`（UTC）

### Errors / Edge Cases

- `NEXT_PUBLIC_API_BASE_URL` 缺失：页面应提示“环境未配置”，避免空白失败。
- 邮箱已注册：提示“该邮箱已注册，请直接登录”。
- 用户名已存在：提示“该用户名已被占用，请更换”。
- 密码弱 / 格式不合法：提示“密码不符合要求”（并在前端做基本校验减少无效请求）。
- 后端不可达 / 网络错误：提示“服务暂不可用，请稍后重试”。
- 数据库未启动/连接失败：后端返回 500（或统一 503），前端提示“服务暂不可用”，后端日志记录根因。

### Test Plan

- 手工冒烟：
  - 成功注册 → 自动跳转 `/dashboard`
  - 已注册邮箱 → 看到清晰错误提示
  - 用户名冲突 → 看到清晰错误提示
  - 缺失 env → 看到清晰配置提示
- 工程校验：
  - Backend：`pytest` 通过（覆盖成功注册/重复邮箱/重复用户名；CI 可用 SQLite，开发本地用 Docker Postgres）
  - Frontend：`npm run lint`、`npm run build` 通过

## Tasks / Subtasks

- [x] 1) Backend：用户表与数据库准备（AC: 1）
  - [x] 使用本地 Docker 启动 PostgreSQL（或提供等价本地安装方式）
  - [x] 扩展 `users` 模型：`email`、`username`、`password_hash`、`created_at`（唯一约束 + 索引）
  - [x] Alembic migration 生成并可执行

- [x] 2) Backend：注册 API + JWT 签发（AC: 1）
  - [x] 新增 `POST /auth/register`（email/username/password）
  - [x] 密码哈希存储（禁止明文），注册成功签发 JWT
  - [x] 统一错误返回（400/409/500），不泄露内部信息
  - [x] pytest 覆盖：成功注册、重复邮箱、重复用户名

- [x] 3) Frontend：注册 UI + 调用后端（AC: 1）
  - [x] `/login` 中提供注册入口（Tab 或按钮）
  - [x] 注册表单字段：email、username、password（required + basic 校验）
  - [x] 调用 `POST ${NEXT_PUBLIC_API_BASE_URL}/auth/register`，成功后保存 token 并跳转 `/dashboard`
  - [x] 提交态与错误态：loading、提交失败提示、字段错误提示

- [x] 4) Dashboard 占位与重定向（AC: 1）
  - [x] 确认注册成功后可到达 `/dashboard`（后续 Story 再做受保护路由/登录态守卫）

- [x] 5) 本地验证（AC: 1）
  - [x] `npm run dev` 本地手工走通注册流程（成功与失败各至少 1 次）
  - [x] `npm run lint` 与 `npm run build` 通过
  - [x] `pytest` 通过

### Review Follow-ups (AI)

- [x] [AI-Review][HIGH] 替换自研密码哈希/JWT 为标准库实现（建议：`passlib[bcrypt]` 或 `argon2-cffi` + `pyjwt`/`python-jose`），并补齐 issuer/audience/clock-skew 等校验（`crewagent-builder-backend/app/utils/security.py:12`）
- [x] [AI-Review][MEDIUM] 增加启动时配置校验：非开发环境禁止使用默认/弱 `JWT_SECRET`（例如 `change-me` 或过短），不满足则 fail-fast（`crewagent-builder-backend/app/config.py:15`）
- [x] [AI-Review][LOW] 在全局异常处理器中记录服务端日志（校验错误/未捕获异常），返回仍保持不泄露内部信息（`crewagent-builder-backend/app/main.py:48`）
- [x] [AI-Review][LOW] 明确 token 存储策略（localStorage vs Cookie）及后续 API 请求携带方式（可在后续 Story 2.2/2.3 落地）（`crewagent-builder-frontend/src/lib/auth.ts:1`）

## Dev Notes

### 关键约束 / 决策

- 本 Story 只实现“注册 + 跳转 + token 签发”，登录页/受保护路由细节在 Story 2.2/2.3 扩展（避免一次性引入过多状态管理复杂度）。
- 账号体系改为系统自研：后端负责密码哈希、JWT 签发与校验；数据库使用本地 Docker PostgreSQL（开发）。
- Token 策略（MVP）：前端 localStorage 保存 `accessToken`；后续请求使用 `Authorization: Bearer <token>`；未来可迁移到 httpOnly Cookie 以降低 XSS 风险。
- 说明：此前有一版基于 Supabase 的注册实现（前端），由于架构调整将被替换/移除。

### Project Structure Notes

- 目标路由位置（App Router）：
  - `crewagent-builder-frontend/src/app/login/page.tsx`
  - `crewagent-builder-frontend/src/app/dashboard/page.tsx`
- 目标工具封装：
  - `crewagent-builder-frontend/src/lib/auth.ts`（已有占位，可用于后续抽象）
  - `crewagent-builder-frontend/src/lib/api-client.ts`（后续与后端交互时使用）

### References

- `_bmad-output/epics.md`（Epic 2 / Story 2.1）
- `_bmad-output/architecture.md`（Auth Method: Custom Auth；前端目录结构）
- `_bmad-output/prd.md`（Authentication constraints: username/password only）
- `crewagent-builder-frontend/.env.local.example`

## Dev Agent Record

### Agent Model Used

GPT-5.2 (Codex CLI)

### Debug Log References

- Backend:
  - `docker compose up -d`
  - `DATABASE_URL=... alembic upgrade head`
  - `./.venv/bin/ruff check .`
  - `./.venv/bin/pytest -q`
- Frontend:
  - `npm install`
  - `npm run lint`
  - `npm run build`
  - `npm run dev`（浏览器手工验证：成功注册 + 重复邮箱错误）

### Completion Notes List

- Backend 新增 `POST /auth/register`：写入 PostgreSQL（Docker）`users` 表，密码使用 `passlib[bcrypt]` 哈希存储，注册成功签发 `PyJWT`（HS256，含 issuer/audience，支持 leeway 校验）。
- Backend 增加统一错误响应：校验错误 400、冲突 409、内部错误 500 均返回 `{ data, error }` 结构。
- Frontend `/login` 注册改为调用后端并保存 token（localStorage），注册成功跳转 `/dashboard`。
- 校验通过：Backend `ruff` + `pytest`；Frontend `npm run lint` + `npm run build`。

### Change Log

- Replace Supabase Auth with custom backend registration + JWT.
- Add local Docker PostgreSQL compose + Alembic migration.
- Add backend tests for registration flow.

### File List

- `_bmad-output/architecture.md`
- `_bmad-output/epics.md`
- `_bmad-output/implementation-artifacts/2-1-user-registration-on-builder.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`

- `crewagent-builder-backend/.env.example`
- `crewagent-builder-backend/README.md`
- `crewagent-builder-backend/requirements.txt`
- `crewagent-builder-backend/app/config.py`
- `crewagent-builder-backend/app/database.py`
- `crewagent-builder-backend/app/main.py`
- `crewagent-builder-backend/app/models/user.py`
- `crewagent-builder-backend/app/routers/auth.py`
- `crewagent-builder-backend/app/schemas/user.py`
- `crewagent-builder-backend/app/services/auth_service.py`
- `crewagent-builder-backend/app/utils/security.py`
- `crewagent-builder-backend/docker-compose.yml`
- `crewagent-builder-backend/migrations/versions/245ca88be4a6_init_tables.py`
- `crewagent-builder-backend/tests/conftest.py`
- `crewagent-builder-backend/tests/test_register.py`
- `crewagent-builder-backend/tests/test_user_model.py`

- `crewagent-builder-frontend/.env.local.example`
- `crewagent-builder-frontend/package-lock.json`
- `crewagent-builder-frontend/package.json`
- `crewagent-builder-frontend/src/app/dashboard/page.tsx`
- `crewagent-builder-frontend/src/app/login/page.tsx`
- `crewagent-builder-frontend/src/app/page.tsx`
- `crewagent-builder-frontend/src/lib/api-client.ts`
- `crewagent-builder-frontend/src/lib/auth.ts`

## Senior Developer Review (AI)

### Review Outcome

- Date: 2025-12-22
- Outcome: Approved
- Issues: 0
- Acceptance Criteria: PASS

### Action Items

- [x] [AI-Review][HIGH] 替换自研密码哈希/JWT 为标准库实现（建议：`passlib[bcrypt]` 或 `argon2-cffi` + `pyjwt`/`python-jose`），并补齐 issuer/audience/clock-skew 等校验（`crewagent-builder-backend/app/utils/security.py:12`）
- [x] [AI-Review][MEDIUM] 增加启动时配置校验：非开发环境禁止使用默认/弱 `JWT_SECRET`（例如 `change-me` 或过短），不满足则 fail-fast（`crewagent-builder-backend/app/config.py:15`）
- [x] [AI-Review][LOW] 在全局异常处理器中记录服务端日志（校验错误/未捕获异常），返回仍保持不泄露内部信息（`crewagent-builder-backend/app/main.py:48`）
- [x] [AI-Review][LOW] 明确 token 存储策略（localStorage vs Cookie）及后续 API 请求携带方式（可在后续 Story 2.2/2.3 落地）（`crewagent-builder-frontend/src/lib/auth.ts:1`）
