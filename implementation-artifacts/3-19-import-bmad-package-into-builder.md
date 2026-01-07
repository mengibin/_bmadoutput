# Story 3.19: Import .bmad Package into Builder

Status: done

> **Tech Spec:** `_bmad-output/implementation-artifacts/tech-spec-3-19-import-bmad-package-into-builder.md`


## Story

As a **Creator**,
I want to import an existing `.bmad` package into the Builder,
So that I can edit and extend workflows created by others or migrate packages between projects.

## Business Value

- **Collaboration**: Enables team members to share and iterate on workflow designs
- **Reusability**: Allows importing community or pre-built packages as starting points
- **Round-trip Editing**: Completes the Creator-Consumer loop by allowing exported packages to be re-imported

## Acceptance Criteria

### AC-1: Package Upload UI
**Given** I am in the ProjectBuilder view
**When** I click "Import Package" button
**Then** a file picker dialog opens that filters for `.bmad` files
**And** I can select a `.bmad` file from my filesystem

### AC-2: Package Extraction & Validation
**Given** I have selected a `.bmad` file for import
**When** the Builder processes the file
**Then** it extracts the ZIP contents to a temporary location
**And** it validates `bmad.json` against the v1.1 schema
**And** it validates `agents.json` against the v1.1 schema
**And** it validates `workflow.graph.json` against the v1.1 schema
**And** it validates step file frontmatters against the v1.1 schema
**And** if validation fails, it shows actionable error messages with file path and schema pointer

### AC-3: Project Creation from Package
**Given** the `.bmad` package passes validation
**When** the import completes
**Then** a new project is created with the package name (from `bmad.json.name`)
**And** all workflows from the package are loaded into the project
**And** all agents from `agents.json` are loaded at project level
**And** all assets from `assets/` are preserved with their paths
**And** I am redirected to the ProjectBuilder view for the new project

### AC-4: Conflict Handling for Existing Projects
**Given** a project with the same name already exists
**When** I import a package
**Then** the Builder prompts me with options:
  - Create with a new name (auto-suffix like "-imported")
  - Merge into existing project (future enhancement, may be deferred)
  - Cancel import

### AC-5: Multi-Workflow Package Support
**Given** the `.bmad` package contains multiple workflows (`bmad.json.workflows[]`)
**When** the import completes
**Then** all workflows are imported into the project
**And** each workflow's graph, steps, and agent assignments are preserved
**And** the `entry` workflow is marked as default

---

## Import Specification

> 本节明确 Builder 导入 `.bmad` 包时的解析规则与数据映射，遵循 **Package Spec v1.1**。

### 权威规范引用

- 包格式规范: `crewagent-runtime/spec/bmad-package-spec/v1.1/README.md`
- JSON Schemas: `crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/`

### 必选文件 (Required Files)

| 文件 | Schema | 必选 | 说明 |
|:---|:---|:---:|:---|
| `bmad.json` | `bmad.schema.json` | ✅ | 包清单与入口定义 |
| `agents.json` | `agents.schema.json` | ✅ | Agent 定义（项目级共享） |
| `workflow.graph.json` 或 `workflows/*/workflow.graph.json` | `workflow-graph.schema.json` | ✅ | 工作流图结构（权威真源） |
| `workflow.md` 或 `workflows/*/workflow.md` | `workflow-frontmatter.schema.json` | ✅ | 工作流入口文档 |
| `steps/*.md` 或 `workflows/*/steps/*.md` | `step-frontmatter.schema.json` | ✅ | 步骤文件（每节点一个） |

### 可选文件 (Optional Files)

| 文件/目录 | 说明 |
|:---|:---|
| `assets/` | 静态资源（图片、模板、附件） |
| `bmad.json.integrity.sha256` | 可选完整性校验 |

### 导入解析顺序

```
1. 解压 ZIP → 临时内存位置
2. 读取 bmad.json
   ├── 校验 → bmad.schema.json
   ├── 提取 name, version, description, author
   └── 确定模式: 单 workflow (entry.*) 或多 workflow (workflows[])
3. 解析 agents.json
   ├── 校验 → agents.schema.json
   └── 提取 agents[] → 映射到 Builder Agent 模型
4. 遍历 workflows（单/多）
   ├── 读取 workflow.graph.json
   │   ├── 校验 → workflow-graph.schema.json
   │   └── 提取 nodes[], edges[], entryNodeId
   ├── 读取 workflow.md
   │   ├── 解析 YAML frontmatter
   │   └── 校验 → workflow-frontmatter.schema.json
   └── 遍历 steps/*.md
       ├── 解析 YAML frontmatter
       ├── 校验 → step-frontmatter.schema.json
       └── 验证 frontmatter.nodeId ∈ graph.nodes[].id
5. 复制 assets/ (如存在)
6. 创建项目记录（数据库）
```

### 数据字段映射

**bmad.json → Project**
| bmad.json 字段 | Builder Project 字段 |
|:---|:---|
| `name` | `workflow_packages.name` |
| `entry.*` / `workflows[]` | `workflow_packages.workflows_json` + `package_workflows.*` |
| `entry.workflow` / `workflows[].workflow` | `workflow_packages.workflow_md` + `package_workflows.workflow_md` |
| `entry.graph` / `workflows[].graph` | `workflow_packages.graph_json` + `package_workflows.graph_json` |
| `entry.stepsDir` / `workflows[].stepsDir` | 解析 step 文件路径（导入时用于定位） |

**agents.json → Project agents**
| agents.json 字段 | Builder 存储 |
|:---|:---|
| `{ schemaVersion, agents[] }` | `workflow_packages.agents_json`（项目级共享） |

**assets/** → Project assets**
| assets 路径 | Builder 存储 |
|:---|:---|
| `assets/**` | `workflow_packages.assets_json`（保持原路径） |

---

## UI References

> **Design Document:** `_bmad-output/implementation-artifacts/design-3-19-import-bmad-package-ui.md`

![ImportPackageModal](import_package_modal_1767614046988.png)

![ValidationErrorDisplay](validation_error_display_1767614101820.png)

![ConflictDialog](conflict_resolution_dialog_1767614121698.png)

---

## Dev Agent Record

### Agent Model Used
GPT-5.2 (Codex CLI)

### Debug Log References
- `crewagent-builder-backend/.venv/bin/pytest -q crewagent-builder-backend/tests`
- `npm -C crewagent-builder-frontend run lint`
- `npm -C crewagent-builder-frontend run build`

### Completion Notes List
- Implemented server-side v1.1 schema validation for `bmad.json`, `agents.json`, `workflow.graph.json`, `workflow.md` frontmatter, and step frontmatter.
- Implemented conflict rename + retry flow (409 `PKG_NAME_CONFLICT` with suggested name).
- Preserved single-workflow `steps/` key paths and validated root `steps/*.md` agent references.
- Updated `finance-operations.bmad` example workflow frontmatter to satisfy required fields (`stepsCompleted`, `currentNodeId`).

### File List
- crewagent-builder-backend/app/main.py
- crewagent-builder-backend/app/services/package_import_service.py
- crewagent-builder-backend/requirements.txt
- crewagent-builder-backend/tests/test_step_integration.py
- crewagent-builder-frontend/src/app/dashboard/page.tsx
- crewagent-builder-frontend/src/app/login/page.tsx
- crewagent-builder-frontend/src/components/StepEditorPanel.tsx
- crewagent-runtime/spec/bmad-package-spec/v1.1/examples/finance-operations.bmad
- crewagent-runtime/spec/bmad-package-spec/v1.1/examples/finance-operations/workflows/ar-collections/workflow.md
- crewagent-runtime/spec/bmad-package-spec/v1.1/examples/finance-operations/workflows/expense-compliance/workflow.md
- crewagent-runtime/spec/bmad-package-spec/v1.1/examples/finance-operations/workflows/variance-analysis/workflow.md
