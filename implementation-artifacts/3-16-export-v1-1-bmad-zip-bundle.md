# Story 3.16: Export v1.1 `.bmad` ZIP Bundle (structure + download)

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **Creator**,  
I want to export a v1.1-compliant `.bmad` ZIP containing all required files in the correct paths,  
so that Runtime can import it without additional transforms.

## Acceptance Criteria

1. **Given** v1.1 artifacts have been generated（project 级 agents + 多 workflow 产物 + bmad.json）  
   **When** I click "Export Package (v1.1)"（在 ProjectBuilder）  
   **Then** a `.bmad` zip is downloaded containing **all workflows in the project** with at least:  
   - `bmad.json` (root; multi-workflow index + entry paths)  
   - `agents.json` (root; v1.1 manifest)  
   - `workflows/<workflowId>/workflow.graph.json`（v1.1; `node.file` 以 zip root 为基准）  
   - `workflows/<workflowId>/workflow.md`（v1.1 frontmatter + steps index）  
   - `workflows/<workflowId>/steps/<nodeId>.md`（v1.1 step frontmatter；文件名与 graph node id 一致）  
   **And** the downloaded filename is `${sanitizeFilename(projectName)}.bmad`（空名回退为 `Untitled.bmad`）  
   **And** export is blocked with an actionable error if any required artifact is missing/invalid

## Design

### Summary

- Tech Spec: `_bmad-output/tech-spec.md`（Package Spec v1.1 多 workflow 推荐结构）
- 导出目标是 **整包**（包含 project 内全部 workflows；不提供单 workflow 导出）
- 使用 `JSZip` 在前端组装 zip 并触发下载，写入 v1.1 文件集：
  - root：`bmad.json` / `agents.json`
  - per workflow：`workflows/<workflowId>/workflow.md` / `workflow.graph.json` / `steps/<nodeId>.md`
- 说明：`artifacts/`（运行产物目录，@project 可写）不随包导出；只有 `assets/`（随包分发，@pkg 只读）会进入 ZIP（Story 3.18）
- `workflow.graph.json` 由 Builder graph 生成（Story 3.13），并在导出时将 `node.file` 写为 zip root 路径：`workflows/<workflowId>/steps/<nodeId>.md`

### UX / UI

- ProjectBuilder（`/builder/[projectId]`）增加按钮：`Export Package (v1.1)`
  - Disabled：加载中 / 有阻断性 errors（`bmad.json` 或 `agents.json` 生成失败等）/ 正在导出
  - 点击后：
    - 拉取所有 workflow 详情（`GET /packages/{projectId}/workflows/{workflowId}`）
    - 组装 zip 并下载
  - 失败提示：
    - 用可操作错误阻断下载（包含 workflow 名称/ID + 失败原因）

### API / Contracts

- 不新增后端接口；复用现有读取接口：
  - `GET /packages/{projectId}`（project.name + agentsJson）
  - `GET /packages/{projectId}/workflows`（workflow list）
  - `GET /packages/{projectId}/workflows/{workflowId}`（workflowMd + graphJson + stepFilesJson）
- 产物契约：
  - `bmad.json` → `crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/bmad.schema.json`
  - `agents.json` → `crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/agents.schema.json`
  - `workflows/<workflowId>/workflow.graph.json` → `crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/workflow-graph.schema.json`
  - `workflows/<workflowId>/workflow.md` / `steps/*.md` frontmatter schema 校验在 Story 3.17

### Data / Storage

- N/A（导出为派生操作，不新增 DB 字段；zip 在浏览器内存中生成后下载）
- 输入来源（必须齐全，否则阻断导出）：
  - `agentsJson`：v1.1 manifest（Story 3.10）或可升级的 legacy（Story 3.15）
  - 每个 workflow：
    - `workflowMd`：v1.1（Story 3.12）
    - `stepFilesJson`：key 为 `steps/<nodeId>.md`（Story 3.12）
    - `graphJson`：Builder graph（ReactFlow nodes/edges；用于导出时派生 v1.1 workflow.graph.json，Story 3.13）

### Implementation Notes

- 建议新增前端导出模块（纯函数 + 轻薄 UI 包装）：
  - `buildZipBundleV11({ project, workflows, workflowDetails, assets? }): { blob, filename, errors, warnings }`
  - `sanitizeFilename(name): string`
- 打包路径规则（多 workflow 推荐模式）：
  - `bmad.json`：root
  - `agents.json`：root
  - 对每个 workflow（`id = String(workflowId)`，需与 `bmad.json.workflows[].id` 一致）：
    - 写入 `workflows/<id>/workflow.md`（来自 `workflowMd`）
    - 写入 `workflows/<id>/steps/<nodeId>.md`（来自 `stepFilesJson`；key 已含 `steps/` 前缀 → 直接拼接 `workflows/<id>/`）
    - 生成并写入 `workflows/<id>/workflow.graph.json`：
      - `buildWorkflowGraphV11({ nodes, edges })` 生成基础 v1.1 graph（其 `node.file` 为 `steps/<nodeId>.md`）
      - 导出写入前把每个 `node.file` 改为 `workflows/<id>/steps/<nodeId>.md`（zip root 基准）
- 基础一致性校验（本 story 做，Story 3.17 才做 schema 级别）：
  - `bmad.json`/`agents.json` 生成器返回 errors → 阻断
  - 任一 workflow 的 `workflowMd` 为空 / `stepFilesJson` 解析失败或为空 → 阻断（提示“请在 Editor 保存以生成 v1.1 文件”）
  - `buildWorkflowGraphV11` 返回 errors（空图/断边/环/非法 nodeId）→ 阻断（提示具体 workflow）
  - 校验 zip 路径存在：`bmad.json.entry.workflow/graph/agents` 指向的文件必须存在于 zip
  - 校验每个 `workflow.graph.json.nodes[].file` 指向的 step 文件存在于 zip

### Errors / Edge Cases

- projectName 为空：文件名回退 `Untitled.bmad`（不阻断）
- workflowId 非法：阻断（需满足 `^[A-Za-z0-9][A-Za-z0-9._:-]*$`；数字字符串合法）
- `stepFilesJson` key 不以 `steps/` 开头：阻断（避免写入错误路径）
- 生成 zip 失败 / 浏览器下载被拦截：提示“导出失败，请重试或检查浏览器下载权限”

### Test Plan

- 正例：导出后解压，路径与文件齐全且 steps 文件位于 `workflows/<workflowId>/steps/`
- 负例：构造缺失 steps 或 graph 循环 → 导出被阻断并提示原因
- Unit：用 `JSZip.loadAsync` 读取生成的 blob，断言文件清单与关键文件内容存在（multi-workflow + node.file 前缀正确）

## Tasks / Subtasks

- [x] 1) 前端：实现整包导出（ProjectBuilder 按钮 + 触发下载）
- [x] 2) 前端：实现 v1.1 zip 组装（root + `workflows/<workflowId>/...`；包含所有 workflows）
- [x] 3) 前端：生成并写入 v1.1 `workflow.graph.json`（export-time 修正 `node.file` 为 zip root 路径）
- [x] 4) 基础一致性校验与错误展示（缺文件/解析失败/空图/环/非法 id）
- [x] 5) 单测：导出 zip 结构断言（JSZip 读回校验）

### Review Follow-ups (AI)

- [x] [AI-Review][High] 将 `assets/**` 导出明确归属到 Story 3.18（本 story 仅负责整包导出结构与下载；zip builder 保留可注入 assets 的能力） `_bmad-output/implementation-artifacts/3-18-project-builder-package-assets-management.md`
- [ ] [AI-Review][Medium] 补全/校准本 story 的 File List（当前与 git 实际变更偏差大），确保可审计或清理无关改动 `_bmad-output/implementation-artifacts/3-16-export-v1-1-bmad-zip-bundle.md:141`
- [x] [AI-Review][Medium] 改善导出错误展示（errors/warnings 分行、可折叠、最多显示 N 条+展开），避免长字符串难读 `crewagent-builder-frontend/src/app/builder/[projectId]/page.tsx:1117`
- [x] [AI-Review][Medium] 扩展导出单测覆盖：workflowId 非法、`bmad.json.entry` 指向缺文件、stepFilesJson key 非 `steps/`、assets path 安全等 `crewagent-builder-frontend/tests/bmad-zip-v11.test.ts`
- [x] [AI-Review][Low] 下载 URL 回收改为延迟/after tick，降低极端浏览器下载失败概率 `crewagent-builder-frontend/src/app/builder/[projectId]/page.tsx:69`
- [ ] [AI-Review][Low] 重构 ProjectBuilder 导出相关逻辑为独立组件/hook，并统一格式化缩进，降低维护成本 `crewagent-builder-frontend/src/app/builder/[projectId]/page.tsx:1017`

## References

- `_bmad-output/epics.md`（Epic 3 / Story 3.16）
- `_bmad-output/tech-spec.md`（Package Spec v1.1）

## Dev Agent Record

### Agent Model Used

GPT-5.2 (Codex CLI)

### Debug Log References

- Frontend:
  - `npm -C crewagent-builder-frontend run lint`
  - `npm -C crewagent-builder-frontend test`
  - `npm -C crewagent-builder-frontend run build`

### Completion Notes List

- ProjectBuilder 增加 `Export Package (v1.1)`：导出整包多 workflow 的 `.bmad`（zip root：`bmad.json/agents.json`；per-workflow：`workflows/<id>/...`）。
- 导出时生成 v1.1 `workflow.graph.json`，并将 `node.file` 重写为 zip root 路径 `workflows/<workflowId>/steps/<nodeId>.md`。
- 增加基础一致性校验（缺 workflow/step files、graph 有环/断边、未知 agentId 等）与错误提示；并补充 JSZip 读回的单测断言。

### File List

- `_bmad-output/implementation-artifacts/3-16-export-v1-1-bmad-zip-bundle.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `crewagent-builder-frontend/src/app/builder/[projectId]/page.tsx`
- `crewagent-builder-frontend/src/lib/bmad-zip-v11.ts`
- `crewagent-builder-frontend/tests/bmad-zip-v11.test.ts`

### Change Log

- Builder：新增 v1.1 整包导出（multi-workflow zip 结构 + 下载）。
- Builder：导出时生成 `workflow.graph.json` 并校验/重写 `node.file` 指向 zip 内 step 文件。
- Tests：增加导出 zip 的结构与关键字段断言（JSZip loadAsync）。
