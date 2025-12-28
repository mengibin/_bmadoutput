# Story 3.18: ProjectBuilder Package Assets Management (Templates)

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **Creator**,  
I want to manage package-level assets (e.g., templates) and reference them from workflow steps,  
so that the exported `.bmad` contains everything needed by Runtime without relying on external files.

## Acceptance Criteria

1. **Given** I am in ProjectBuilder  
   **When** I create/edit/delete files under the package `assets/` namespace  
   **Then** the assets are persisted and listed with stable paths (e.g., `assets/templates/*.md`)

2. **Given** assets exist in the project  
   **When** I am editing a workflow step node  
   **Then** I can copy/insert asset paths into the step metadata (`inputs/outputs`) or step instructions text

3. **Given** package assets exist  
   **When** I export the package (v1.1)  
   **Then** the ZIP contains `assets/**` at the expected paths  
   **And** `bmad.json.entry.assetsDir == "assets/"` remains valid

## Design

### Summary

- Tech Spec: `_bmad-output/tech-spec.md`（Package Spec v1.1 / `assets/`）
- MVP：仅支持**文本类** assets（`.md/.txt/.json/.yaml/.yml`），存储在 Builder Backend（DB）并在 v1.1 导出时写入 ZIP 根目录 `assets/**`（供 Runtime 以 `@pkg/assets/**` 只读访问）。
- 资产路径与导出安全校验对齐：复用导出管线对 `assets/**` 的安全规则，避免双写与规则漂移（`crewagent-builder-frontend/src/lib/bmad-zip-v11.ts`）。

### UX / UI

- ProjectBuilder（`/builder/[projectId]`）新增 “Assets” 区块：
  - 列表：按 path 排序展示（例如 `assets/templates/story-template.md`），每项提供：
    - `Copy (zip path)`：复制 `assets/...`
    - `Copy (runtime path)`：复制 `@pkg/assets/...`
    - `Edit` / `Delete`
  - 新建/编辑弹窗（Modal）：
    - `path`：必须以 `assets/` 开头；仅允许 `[A-Za-z0-9._/-]`（禁止空段、`..`、反斜杠、绝对路径）
    - `content`：文本编辑（textarea），UTF-8
    - 校验提示：不合法 path/扩展名/超限时，明确提示并阻断保存
- WorkflowEditor（`/editor/[projectId]/[workflowId]`）在 Step Node 编辑面板增加 “Insert Asset”：
  - 选择一个 asset 后提供快捷动作：
    - `Copy @pkg path`（`@pkg/assets/...`）
    - `Insert into Instructions`：在 instructions 末尾追加一行 `@pkg/assets/...`（最小实现，不要求光标插入）
    - `Add to inputs`：向 inputs 列表追加 `assets/...`（一行一个）
    - `Add to outputs`：不推荐（assets 为只读）；如提供则需要明确 warning（默认不提供该按钮）

### API / Contracts

> 约定：与现有 `/packages` API 一致，返回 `{ data, error }`；需要登录（JWT）。

- `GET /packages/{projectId}/assets`
  - 返回 `{ data: { assetsJson: string }, error: null }`（`assetsJson` 为 JSON object string：`{ "<path>": "<content>" }`）
- `POST /packages/{projectId}/assets`
  - Body：`{ path: string, content: string }`（create-only；若已存在同 path → 409）
  - 返回：同 GET（或返回单个 asset 也可，但需前后端一致）
- `PUT /packages/{projectId}/assets`
  - Body：`{ path: string, content: string }`（update-only；若不存在 → 404）
- `DELETE /packages/{projectId}/assets`
  - Body：`{ path: string }`（delete；若不存在 → 404）

服务端校验（必须做，前端校验仅作为 UX）：

- `path` 规则：
  - 必须以 `assets/` 开头
  - 必须是相对路径（不能以 `/` 开头）
  - 规范化：`\` → `/`，去除前导 `./`
  - 禁止：空段、`.`、`..`、`\u0000`
  - 允许字符集：`^[A-Za-z0-9._/-]+$`
  - 扩展名白名单：`.md .txt .json .yaml .yml`
- 限制（MVP，可在 config 中集中定义）：
  - 单文件最大：`256 KiB`
  - 每项目最大总量：`2 MiB`
  - 最大文件数：`200`

### Data / Storage

- 后端：在 `workflow_packages` 增加列 `assets_json`（Text，默认 `{}`），存放 JSON object string：`{ "assets/..": "..." }`。
- 读取时解析为 dict，写回时 `json.dumps(..., ensure_ascii=False)`；key 为规范化后的 path。
- 说明：MVP 不支持二进制；后续若支持，可将 value 升级为 `{ encoding, content, mime }` 结构并保持向后兼容。

### Errors / Edge Cases

- 401：未登录（沿用现有 auth 行为）
- 404：项目不存在 / asset 不存在
- 409：create 时 path 已存在（提示“路径已存在，请改名或使用编辑”）
- 400：path 不合法 / 扩展名不支持 / JSON 序列化失败
- 413：单文件或总量超限（提示当前限制与建议）
- 删除影响：若 workflow step 中引用了已删除的资产，Builder 不做强制阻断（允许用户自负），但在 Editor 的 Insert UI 中可提示“引用不存在资产”的轻量 warning（可选）。

### Test Plan

- Backend (unit/integration):
  - path 校验：`assets/../x`、`/assets/x`、包含空段/`\0`、非法字符 → 400
  - 扩展名限制：`.png/.pdf` → 400
  - 超限：单文件/总量/文件数 → 413
  - CRUD：create/update/delete + 404/409 行为
- Frontend:
  - ProjectBuilder：新建/编辑/删除后列表刷新；Copy 两种 path 生效
  - WorkflowEditor：Insert into instructions/inputs 生效（对 selected node data 的更新）
  - Export：当 assets 存在时，导出 ZIP 包含 `assets/**`（可用 `JSZip.loadAsync` 读回断言）

## Tasks / Subtasks

- [x] 1) Backend：`workflow_packages` 增加 `assets_json` + migration；实现 assets CRUD endpoints（含校验与限额）
- [x] 2) Frontend：ProjectBuilder 增加 “Assets” 区块（列表/新建/编辑/删除/复制 `assets/...` 与 `@pkg/assets/...`）
- [x] 3) Frontend：WorkflowEditor 增加 “Insert Asset” 快捷入口（copy/insert 到 instructions、add 到 inputs）
- [x] 4) 导出集成：导出前拉取 assets 并传入 `buildBmadExportFilesV11({ assets })`；导出 ZIP 包含 `assets/**`
- [x] 5) 测试：后端 CRUD + 前端导出 ZIP 读回断言（含 assets）

## References

- `_bmad-output/epics.md`（Epic 3 / Story 3.18）
- Tech Spec: `_bmad-output/tech-spec.md`
- `crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/bmad.schema.json`（`entry.assetsDir`）
- `crewagent-runtime/spec/bmad-package-spec/v1.1/examples/create-story-micro/assets/story-template.md`（assets 使用示例）

## Dev Agent Record

### Agent Model Used

GPT-5.2 (Codex CLI)

### Debug Log References

- Backend:
  - `python -m pytest -q`
- Frontend:
  - `npm -C crewagent-builder-frontend test`
  - `npm -C crewagent-builder-frontend run lint`
  - `npm -C crewagent-builder-frontend run build`

### Completion Notes List

- Backend：新增 `workflow_packages.assets_json`（Text，默认 `{}`），并实现 `/packages/{projectId}/assets` CRUD（含 path 规则 + 扩展名白名单 + 单文件/总量/文件数限额）。
- Frontend：ProjectBuilder 新增 Assets 面板（列表/新建/编辑/删除/复制 zip path 与 `@pkg` runtime path）；WorkflowEditor 新增 Insert Asset（Copy @pkg / Insert into Instructions / Add to inputs）。
- 导出集成：ProjectBuilder 导出 v1.1 时将 assets 写入 ZIP 根目录 `assets/**`，并与既有导出安全校验规则对齐。
- 测试：后端 assets CRUD 测试覆盖；前端 test/lint/build 通过（导出库单测已覆盖 assets 写入 ZIP）。

### File List

- `_bmad-output/implementation-artifacts/3-18-project-builder-package-assets-management.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `crewagent-builder-backend/app/models/workflow_package.py`
- `crewagent-builder-backend/app/routers/packages.py`
- `crewagent-builder-backend/app/schemas/workflow_package.py`
- `crewagent-builder-backend/app/services/package_service.py`
- `crewagent-builder-backend/migrations/versions/2d4e6f8a0c1b_add_assets_json_to_workflow_packages.py`
- `crewagent-builder-backend/tests/test_package_assets.py`
- `crewagent-builder-frontend/src/app/builder/[projectId]/page.tsx`
- `crewagent-builder-frontend/src/app/editor/[projectId]/[workflowId]/page.tsx`
- `crewagent-builder-frontend/src/lib/api-client.ts`
- `crewagent-builder-frontend/src/lib/assets-v11.ts`
