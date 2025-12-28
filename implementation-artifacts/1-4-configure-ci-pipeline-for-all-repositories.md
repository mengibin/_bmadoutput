# Story 1.4: Configure CI Pipeline for All Repositories

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a Developer,  
I want to set up GitHub Actions CI workflows for all three repositories,  
so that code is automatically built and checked on every push / PR.

## Acceptance Criteria

1. **Given** the three repositories exist on GitHub  
   **When** I create `.github/workflows/ci.yml` in each repo  
   **Then** pushes and pull requests trigger build and lint/test checks  
   **And** failing checks block PR merges.

## Design

### Summary

- 每个仓库各自新增 `.github/workflows/ci.yml`，并统一 **job id = `ci`**，以便在 Branch Protection 里配置 required checks。
- CI 触发条件：`push` + `pull_request`（所有分支），执行“安装依赖 → lint → build/test”的最小质量门禁。
- 前端/Runtime 通过 `actions/setup-node@v4` + `node-version-file: .nvmrc` 保持 Node 版本一致；后端通过 `actions/setup-python@v5` 固定 `python-version: "3.11"`。
- 后端 lint 采用 `ruff check .`：将 `ruff` 写入 `crewagent-builder-backend/requirements.txt`，并在 `crewagent-builder-backend/pyproject.toml` 增加最小 `tool.ruff` 配置（至少 `ignore = ["E501"]`）。
- Runtime 增加 `npm run build:ci`（仅 `tsc && vite build`），CI 使用 `build:ci` 避免 `electron-builder` 打包/签名。

### UX / UI

- N/A（纯 CI / GitHub Actions 配置，无 UI 交互）

### API / Contracts

- N/A

### Data / Storage

- N/A（仅使用 GitHub Actions 的 cache：npm/pip 缓存，不引入持久化数据）

### Errors / Edge Cases

- **Node 版本不一致**：`setup-node` 使用 `.nvmrc`；若未来需要精确到 patch，可将 `.nvmrc` 由 `20` 改为 `20.9.0`（并同步文档/engine）。
- **`npm ci` 依赖 lockfile**：确保各仓库存在 `package-lock.json`（后续改用 pnpm/yarn 需同步 CI）。
- **Next.js build 环境变量**：未来若构建依赖 secrets（如 API Key），需在 GitHub Secrets 配置并在 workflow 中注入；当前基础工程应保持 `npm run build` 在无 secrets 下可通过。
- **后端 ruff 缺失/规则过严**：CI 直接跑 `ruff check .`，必须在依赖与配置到位后再启用；初期建议忽略 `E501`，避免阻塞。
- **Runtime 打包依赖/签名**：CI 使用 `build:ci`，避免 `electron-builder` 在 Linux runner 上因签名/打包依赖失败导致误报。
- **分支保护 required checks 名称**：以 job id `ci` 为准；首次运行 workflow 后再在仓库设置里选择对应 check。

### Test Plan

- **CI 命令验证（本地）**：
  - Frontend：`npm ci && npm run lint && npm run build`
  - Backend：`pip install -r requirements.txt && ruff check . && pytest`
  - Runtime：`npm ci && npm run lint && npm run build:ci`
- **GitHub Actions 验证**：
  1. 在三个仓库各推送一次（或开 PR）触发 workflow，确认 check run 名称出现 `ci`。
  2. 故意引入一个 lint 或测试失败（例如改坏断言）确认 CI 失败。
  3. 在仓库设置开启 Branch Protection：Require status checks to pass before merging，并将 required checks 设为 `ci`。
  4. 再开一个 PR：确认 CI 失败时无法 merge；修复后 CI 通过才可 merge。

## Tasks / Subtasks

- [x] 1) CI baseline & constraints（AC: 1）
  - [x] 确认每个仓库的 workflow 位置为仓库根目录：`.github/workflows/ci.yml`
  - [x] 对齐版本约束：
    - Node：`>= 20.9.0`（前端/Runtime 已有 `.nvmrc` + `package.json#engines`）
    - Python：`3.11`（后端）
  - [x] 确认每个仓库的最小质量门禁（命令级别）：
    - Frontend：`npm ci` → `npm run lint` → `npm run build`
    - Backend：`pip install -r requirements.txt` → `ruff check .` → `pytest`
    - Runtime：`npm ci` → `npm run lint` → `npm run build:ci`（避免打包）

- [x] 2) Builder Frontend CI（Next.js）（AC: 1）
  - [x] 新增 `crewagent-builder-frontend/.github/workflows/ci.yml`
  - [x] Job: `ci`（用于 branch protection 的 required check 名称）
  - [x] Steps：
    - `actions/checkout@v4`
    - `actions/setup-node@v4`（`node-version-file: .nvmrc`，启用 npm cache）
    - `npm ci`
    - `npm run lint`
    - `npm run build`

- [x] 3) Builder Backend CI（FastAPI）（AC: 1）
  - [x] 新增 `crewagent-builder-backend/.github/workflows/ci.yml`
  - [x] Job: `ci`
  - [x] Steps：
    - `actions/checkout@v4`
    - `actions/setup-python@v5`（`python-version: "3.11"`，启用 pip cache）
    - `python -m pip install --upgrade pip`
    - （若未引入）为后端增加 lint：`ruff`（建议写入 `crewagent-builder-backend/requirements.txt`，并在 `crewagent-builder-backend/pyproject.toml` 配置 `tool.ruff`，至少 `ignore = ["E501"]`）
    - `pip install -r requirements.txt`
    - `ruff check .`
    - `pytest`

- [x] 4) Runtime Client CI（Electron + Vite）（AC: 1）
  - [x] 新增 `crewagent-runtime/.github/workflows/ci.yml`
  - [x] Job: `ci`
  - [x] Steps：
    - `actions/checkout@v4`
    - `actions/setup-node@v4`（`node-version-file: .nvmrc`，启用 npm cache）
    - `npm ci`
    - `npm run lint`
    - `npm run build:ci`（仅 `tsc && vite build`，避免 `electron-builder` 打包/签名）
  - [x] 若现有脚本没有 `build:ci`：补充 `package.json` scripts（本 Story 内完成）

- [x] 5) Merge blocking（Branch Protection）（AC: 1）
  - [x] 在 GitHub 仓库设置中启用分支保护：Require status checks to pass before merging
  - [x] 将 required checks 设为 `ci`（与 workflow job name 对齐）
  - [x] 用一个测试 PR 验证：CI 失败时无法 merge

## Dev Notes

- GitHub Actions 只会从**仓库根目录**的 `.github/workflows/*.yml` 读取工作流；本 Story 的前提是三个工程分别作为独立仓库存在于 GitHub。
- Runtime 的 `npm run build` 往往包含 `electron-builder` 打包步骤，CI 建议只做“编译/构建校验”，打包发布可放到 Release workflow（后续 Story）。

### Project Structure Notes

- 前端仓库：`crewagent-builder-frontend/`
- 后端仓库：`crewagent-builder-backend/`
- Runtime 仓库：`crewagent-runtime/`

### References

- `_bmad-output/epics.md`（Epic 1 / Story 1.4）
- `_bmad-output/architecture.md`（技术栈；多仓库策略）
- `crewagent-builder-frontend/package.json`
- `crewagent-builder-frontend/.nvmrc`
- `crewagent-builder-backend/requirements.txt`
- `crewagent-runtime/package.json`
- `crewagent-runtime/.nvmrc`

## Dev Agent Record

### Agent Model Used

GPT-5.2 (Codex CLI)

### Debug Log References

### Completion Notes List

1. 已新增三仓库 GitHub Actions CI：`ci` job，命令门禁与 Story 设计一致。
2. Backend：引入 `ruff`（依赖 + 配置），修复 `migrations/env.py` 的 ruff unused-import（F401）以确保 CI 可过。
3. Runtime：新增 `npm run build:ci`（仅 `tsc && vite build`），避免 CI 打包/签名问题；本地 lint + build:ci 已验证通过。
4. 已在三个 GitHub 仓库完成分支保护：required check 设为 `ci`，CI 失败阻止合并。
5. Code review：通过（保留 3 个 LOW 改进建议：CI 触发范围/并发控制/跨平台矩阵）。

### File List

- crewagent-builder-frontend/.github/workflows/ci.yml
- crewagent-builder-backend/.github/workflows/ci.yml
- crewagent-runtime/.github/workflows/ci.yml
- crewagent-runtime/package.json
- crewagent-builder-backend/requirements.txt
- crewagent-builder-backend/pyproject.toml
- crewagent-builder-backend/migrations/env.py
- _bmad-output/implementation-artifacts/1-4-configure-ci-pipeline-for-all-repositories.md
- _bmad-output/implementation-artifacts/sprint-status.yaml
