---
stepsCompleted: [1, 2, 3]
inputDocuments:
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/prd.md'
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/architecture.md'
workflowType: 'epics'
lastStep: 3
project_name: 'CrewAgent'
user_name: 'Mengbin'
date: '2025-12-21'
---

# CrewAgent - Epic Breakdown

## Overview

This document provides the complete epic and story breakdown for CrewAgent, decomposing the requirements from the PRD and Architecture into implementable stories.

## Requirements Inventory

### Functional Requirements

**Definition Capabilities (The Package Format):**
- FR-DEF-01: System must support Workflow definitions as Markdown files (`.md`) with YAML Frontmatter for state tracking.
- FR-DEF-02: System must support Step Chaining: A main `workflow.md` can reference sub-step files.
- FR-DEF-03: System must support Agent Manifest as a structured file (CSV or JSON) defining agent personas.
- FR-DEF-04: System must export complete workflow definitions as a portable `.bmad` package (ZIP bundle).
- FR-DEF-05: System must validate imported `.bmad` packages against a JSON Schema before loading.

**Creation Capabilities (The Visual Builder):**
- FR-BLD-01: Creators can visually render a Markdown Workflow as an interactive Node Graph.
- FR-BLD-02: Creators can edit Workflow steps via Graph-First Editing (auto-generates Markdown).
- FR-BLD-03: Creators can define Agent Personas via a form-based UI.
- FR-BLD-04: Creators can configure Prompt Templates (System Prompt, User Prompt) within each Agent definition.
- FR-BLD-05: Creators can one-click export the current project as a `.bmad` package.

**Execution Capabilities (The Runtime Engine):**
- FR-RUN-01: Consumers can load a `.bmad` package into the local Runtime Client.
- FR-RUN-02: Runtime must execute by reading Frontmatter state (`stepsCompleted`) and determining the next step.
- FR-RUN-03: Runtime must update Frontmatter in-place when a step completes (Document-as-State).
- FR-RUN-04: Runtime must inject the correct Agent Persona as LLM System Prompt.
- FR-RUN-05: Runtime must support Context Injection (passing artifacts from previous steps).
- FR-RUN-06: Runtime must support Pause/Resume from last saved Frontmatter state.

**Integration Capabilities (MCP Drivers):**
- FR-INT-01: Runtime must support Stdio MCP Driver for spawning local CLI tools.
- FR-INT-02: Runtime must capture stdout and stderr from spawned tools.
- FR-INT-03: Runtime must support FileSystem MCP Driver for reading/writing files.
- FR-INT-04: Runtime must enforce Sandboxed File Access (scoped to Project Folder).

**Management Capabilities (Project & State):**
- FR-MNG-01: System must support selecting/opening a user **Project Folder** (`ProjectRoot`), and create a dedicated **Run Folder** (private RuntimeStore) for each new execution.
- FR-MNG-02: System must save user-visible artifacts within `ProjectRoot` by default (e.g., `ProjectRoot/artifacts/`), while keeping package cache, run state, and logs in a private RuntimeStore by default.
- FR-MNG-03: System must maintain an Execution Log recording each tool call and result.
- FR-MNG-04: Consumers can view the current Workflow State in the Client UI.

### Non-Functional Requirements

**Reliability & Availability:**
- NFR-REL-01: Graceful Recovery - resume from last saved Frontmatter state after crash.
- NFR-REL-02: Configurable Timeout for all Tool Calls (default 5 minutes).

**Security & Privacy:**
- NFR-SEC-01: LLM API Key Storage is user-configurable (plaintext allowed).
- NFR-SEC-02: Sandboxed Execution - child processes restricted to Project Folder.

**Usability (Offline Mode):**
- NFR-USAB-01: Runtime Client fully operational offline after initial license activation.
- NFR-USAB-02: Client must support local LLM endpoints (Ollama, LM Studio).

### Additional Requirements (from Architecture)

**Starter Template Required:**
- Architecture specifies: Next.js (Frontend), FastAPI (Backend), Electron-Vite (Runtime).
- Epic 1 Story 1 must initialize repositories using these starters.

**Technical Infrastructure:**
- Database: PostgreSQL（开发：本地 Docker；生产：托管 PostgreSQL）
- ORM: SQLAlchemy + Alembic (Backend)
- Auth: Username/Password（系统自研：FastAPI + PostgreSQL）
- Deployment: Vercel (Frontend), Railway (Backend), GitHub Releases (Runtime)

**LLM-as-Engine Architecture:**
- Runtime does NOT implement traditional workflow engine.
- Build "Thin Shell": Prompt Composer, Tool Executor, State Manager, LLM Adapter.

### FR Coverage Map

| FR ID | Epic | Description |
|:---|:---|:---|
| FR-DEF-01~04 | Epic 3 | Workflow definitions, Step Chaining, Agent Manifest, Package Export |
| FR-DEF-05 | Epic 4 | Validate imported `.bmad` packages via JSON Schema |
| FR-BLD-01~05 | Epic 3 | Node Graph, Graph-First Editing, Agent Forms, Prompt Templates, One-Click Export |
| FR-RUN-01~06 | Epic 4 | Package Load, Frontmatter State, Document-as-State, Agent Injection, Context Injection, Pause/Resume |
| FR-INT-01~04 | Epic 4 | Stdio MCP, stdout/stderr capture, FileSystem MCP, Sandboxed Access |
| FR-MNG-01~04 | Epic 5 | ProjectRoot + RuntimeStore runs, Artifact output, Execution Log, State UI |
| NFR-REL-01~02 | Epic 5 | Graceful Recovery, Tool Timeout |
| NFR-SEC-01 | Epic 5 | API Key Storage |
| NFR-SEC-02 | Epic 4 | Sandboxed Execution |
| NFR-USAB-01~02 | Epic 4, Epic 5 | Offline Mode, Local LLM Support |

## Epic List

### Epic 1: Project Initialization & Infrastructure
**Goal**: Developers can set up all three repositories and run basic "hello world" apps.
**FRs covered**: Architecture Starter Templates
**Deliverables**: 
- `crewagent-builder-frontend/` (Next.js) initialized
- `crewagent-builder-backend/` (FastAPI) initialized
- `crewagent-runtime/` (Electron) initialized

---

### Epic 2: User Authentication & Account Management
**Goal**: Users can register and login on the Builder platform and the backend can enforce authenticated access.
**FRs covered**: Architecture Auth Requirements (Custom Auth)
**Deliverables**:
- Login page
- Registration flow
- JWT authentication
- 自研 Auth（FastAPI）+ PostgreSQL 用户表
- FastAPI 中 JWT 校验与受保护路由

---

### Epic 3: Workflow Visual Builder
**Goal**: Creators (Simin) can design workflows and agents through a visual interface, export as `.bmad` packages.
**FRs covered**: FR-DEF-01~04, FR-BLD-01~05
**Deliverables**:
- React Flow canvas with workflow nodes
- Step/Agent configuration forms
- Markdown auto-generation from graph
- `.bmad` package export

---

### Epic 4: Workflow Execution Engine
**Goal**: Consumers (David) can load and execute `.bmad` workflows locally.
**FRs covered**: FR-DEF-05, FR-RUN-01~06, FR-INT-01~04, NFR-SEC-01~02, NFR-USAB-01~02
**Deliverables**:
- ProjectManager（选择/打开 ProjectRoot；确保 `ProjectRoot/artifacts/`）
- RuntimeStore + RunManager（包缓存 + run state/logs，默认不污染项目；对 LLM 暴露 `@project/@pkg/@state`）
- Package loader（ZIP → RuntimeStore/packages 缓存 + v1.1 JSON Schema validation）
- GraphStore（`workflow.graph.json`）+ transition guard
- State Manager（`@state/workflow.md` Frontmatter R/W + 原子落盘）
- ToolHost（builtin `fs.*`：mount sandbox + `@pkg` 只读；MCP Stdio 可选/后续）
- Prompt Composer + LLM Adapter + ExecutionEngine（ToolCalls loop）

---

### Epic 5: Observability, Settings & Recovery
**Goal**: Consumers can view execution progress, manage settings/drivers, and recover from failures.
**FRs covered**: FR-MNG-01~04, NFR-REL-01~02, NFR-SEC-01, NFR-USAB-01~02
**Deliverables**:
- ProjectRoot 最近列表 + Run 列表/Resume
- Execution log UI
- Workflow state visualization
- Crash recovery (resume from Frontmatter)
- Runtime settings (LLM provider/model, API keys)
- Tool policy / MCP driver management (enable/disable)

---

### Epic 8: Optimization & Enhancement
**Goal**: Optimize user experience, improve system usability, and enhance runtime efficiency.
**FRs covered**: N/A (non-functional improvements)
**Deliverables**:
- Default package initialization
- Performance optimizations
- UX improvements
- Quality of life features

---

## Epic 1: Project Initialization & Infrastructure

**Goal**: Developers can set up all three repositories and run basic "hello world" apps.

### Story 1.1: Initialize Builder Frontend Repository

As a **Developer**,
I want to initialize `crewagent-builder-frontend` using Next.js starter,
So that I can begin building the Visual Workflow Builder.

**Acceptance Criteria:**

**Given** a clean development environment
**When** I run `npx create-next-app@latest crewagent-builder-frontend --typescript --tailwind --eslint --app --src-dir`
**Then** the Next.js project is created with TypeScript, Tailwind CSS, ESLint, and App Router
**And** I can run `npm run dev` and see the default Next.js page at `localhost:3000`

---

### Story 1.2: Initialize Builder Backend Repository

As a **Developer**,
I want to initialize `crewagent-builder-backend` with FastAPI structure,
So that I can begin building the API and database layer.

**Acceptance Criteria:**

**Given** Python 3.11+ and pip are installed
**When** I create the project with FastAPI, SQLAlchemy, and Alembic
**Then** the project structure matches the Architecture document
**And** I can run `uvicorn app.main:app --reload` and see the FastAPI docs at `/docs`

---

### Story 1.3: Initialize Runtime Client Repository

As a **Developer**,
I want to initialize `crewagent-runtime` using electron-vite starter,
So that I can begin building the Desktop Runtime Client.

**Acceptance Criteria:**

**Given** Node.js 18+ is installed
**When** I run `npm create electron-vite@latest crewagent-runtime` and select the **React** template
**Then** the Electron + Vite + React project is created with TypeScript
**And** I can run `npm run dev` and see the Electron window with default content

---

### Story 1.4: Configure CI Pipeline for All Repositories

As a **Developer**,
I want to set up GitHub Actions CI workflows for all three repositories,
So that code is automatically tested on every push.

**Acceptance Criteria:**

**Given** the three repositories exist on GitHub
**When** I create `.github/workflows/ci.yml` in each repo
**Then** pushes trigger build and lint checks
**And** failing checks block PR merges

---

## Epic 2: User Authentication & Account Management

**Goal**: Users can register and login on the Builder platform, and the backend can protect APIs via authenticated access.

### Story 2.1: User Registration on Builder

As a **User**,
I want to register for an account on the Builder platform,
So that I can save and manage my Workflow definitions.

**Acceptance Criteria:**

**Given** I am on the Builder login page
**When** I click "Register" and enter email, username, and password
**Then** my account is created via the Builder Backend (FastAPI) and stored in PostgreSQL
**And** I receive a JWT token and am authenticated
**And** I am redirected to the Dashboard

---

### Story 2.2: User Login on Builder

As a **User**,
I want to login to the Builder platform,
So that I can access my saved Workflows.

**Acceptance Criteria:**

**Given** I have a registered account
**When** I enter my email and password on the login page
**Then** I receive a JWT token and am authenticated
**And** I can access protected routes (Dashboard, Editor)

---

### Story 2.3: Implement JWT Auth in Backend

As a **Developer**,
I want to implement JWT authentication in the FastAPI backend,
So that API endpoints are protected.

**Acceptance Criteria:**

**Given** a request comes with a JWT token in the Authorization header
**When** the backend validates the token signature and claims
**Then** authenticated requests proceed to the handler
**And** unauthenticated requests receive 401 Unauthorized

**Given** a request has a missing/expired/invalid JWT token
**When** the backend validates the token signature and claims
**Then** the request is rejected with 401 Unauthorized
**And** the response contains a structured JSON error body (code + message)
**And** the protected handler is not executed

---

## Epic 3: Workflow Visual Builder

**Goal**: Creators can design workflows and agents through a visual interface, export as `.bmad` packages.

### Story 3.1: Create New Workflow Project

As a **Creator**,
I want to create a new Workflow project in the Builder,
So that I can start designing my workflow.

**Acceptance Criteria:**

**Given** I am logged into the Builder Dashboard
**When** I click "New Workflow" and enter a project name
**Then** a new project is created with default `workflow.md` and `agents.json`
**And** I am taken to the Editor page for this project

---

### Story 3.2: Add and Edit Step Nodes in Graph

As a **Creator**,
I want to add and edit Step nodes on the React Flow canvas,
So that I can visually define workflow steps.

**Acceptance Criteria:**

**Given** I am in the Workflow Editor
**When** I drag a "Step" from the palette onto the canvas
**Then** a new Step node appears with default properties
**And** I can double-click to edit the step's name, instructions, and agent assignment

---

### Story 3.3: Connect Steps to Form Execution Order

As a **Creator**,
I want to connect Steps with edges to define execution order,
So that the workflow has a clear execution path.

**Acceptance Criteria:**

**Given** multiple Step nodes exist on the canvas
**When** I drag from one node's output port to another's input port
**Then** an edge (connection) is created between them
**And** the execution order is reflected in the generated Markdown

---

### Story 3.4: Define Agent Personas via Form

As a **Creator**,
I want to define Agent personas through a form-based UI,
So that each step has a specialized AI assistant.

**Acceptance Criteria:**

**Given** I click "Manage Agents" in the Editor
**When** I fill out the Agent form (name, role, communication_style, persona)
**Then** the Agent is added to `agents.json`
**And** it appears in the Agent dropdown for Step assignment

---

### Story 3.5: Configure Agent Prompt Templates

As a **Creator**,
I want to configure System Prompt and User Prompt templates for each Agent,
So that the AI understands each step's task.

**Acceptance Criteria:**

**Given** I am editing an Agent
**When** I enter a System Prompt template (with `{{placeholders}}`)
**Then** the template is saved to the Agent definition
**And** it will be used during Runtime execution

---

### Story 3.6: Auto-Generate Markdown from Graph

As a **Creator**,
I want the system to automatically generate Markdown files from my graph edits,
So that I don't need to write Markdown manually.

**Acceptance Criteria:**

**Given** I have made changes to the workflow graph
**When** I save the project or navigate away
**Then** `workflow.md` is regenerated with YAML Frontmatter and step references
**And** step files (`step-01.md`, etc.) are created/updated based on nodes

---

### Story 3.7: One-Click Export as .bmad Package

As a **Creator**,
I want to export my workflow as a `.bmad` package with one click,
So that I can distribute it to Consumers.

**Acceptance Criteria:**

**Given** my workflow project is complete
**When** I click "Export Package"
**Then** a `.bmad` ZIP file is generated containing:
  - `workflow.md`
  - `steps/*.md`
  - `agents.json`
**And** the Builder runs basic pre-export validation (e.g., JSON parse + referenced files exist) and blocks download with actionable errors if validation fails

> 注：完整 **Package Spec v1.1** 合规导出拆分为 Story 3.8–3.17：
> - Story 3.8–3.11：ProjectBuilder 基础能力（多 workflows/多 agents/多 node types + edge label/default + v1.1 必填字段）
> - Story 3.12–3.17：按“先生成 → 再打包 → 再校验”顺序逐步完成 v1.1 导出

---

### Story 3.8: ProjectBuilder Shell (Full-Screen + Navigation)

As a **Creator**,
I want the Builder to provide a full-screen ProjectBuilder shell that shows workflows and agents for a project,
So that I can manage my project at a glance and navigate into a workflow editor.

**Acceptance Criteria:**

**Given** I open the Builder for a project
**When** the page loads
**Then** I see a **ProjectBuilder** view in full-width layout (no max-width container) with:
  - a Workflows panel (list + open)
  - an Agents panel (list)

**Given** the project has workflows
**When** I click a workflow in the list
**Then** I enter that workflow’s editor view and can see its canvas

---

### Story 3.9: ProjectBuilder Multi-Workflow Management (Create/Select)

As a **Creator**,
I want to create and manage multiple workflows within a single project,
So that I can build multi-workflow packages and keep workflows organized.

**Acceptance Criteria:**

**Given** I am in ProjectBuilder
**When** I create a workflow (name required)
**Then** the workflow appears in the workflows list
**And** it is persisted and visible after refresh

**Given** a project contains multiple workflows
**When** I open workflow A then workflow B
**Then** each workflow loads and saves its own graph/content independently

---

### Story 3.10: ProjectBuilder Agent Management (v1.1 Required Fields + Stable `agentId`)

As a **Creator**,
I want to create and edit agents at the project level with v1.1-required fields and stable ids,
So that workflows can reference agents via `agentId` and schema validation won’t fail later.

**Acceptance Criteria:**

**Given** I am in ProjectBuilder
**When** I create an agent
**Then** it is saved with a stable `agentId` and appears in the agents list
**And** required v1.1 fields are present (via user input or defaults), including at least:
  - `metadata.title`
  - `metadata.icon`
  - `persona.principles`

**Given** an agent exists
**When** I edit and save it
**Then** the updates persist and are reflected across workflows in this project

**Given** I assign an agent to a node in a workflow
**When** I save and reload
**Then** the node keeps the assignment by `agentId` (not by free-text name)

---

### Story 3.11: Workflow Editor Node Types + Edge Labels + Default Branch

As a **Creator**,
I want the workflow editor to support multiple node types and configurable edges,
So that the graph is v1.1-ready (edge labels + decision default branch + stable node ids).

**Acceptance Criteria:**

**Given** I am in a workflow editor
**When** I view the palette
**Then** the palette shows node types (at least: `step`, `decision`, `merge`, `end`, `subworkflow`)
**And** I can drag a node type into the canvas to create a node of that type

**Given** I create or edit an edge between nodes
**When** I set an edge label
**Then** the label is saved with the edge
**And** if I do not set a label, the system uses a deterministic default label

**Given** a decision node has multiple outgoing edges
**When** I select one outgoing edge as default
**Then** that choice is saved and can later be exported as `isDefault: true`

**Given** nodes exist on the canvas
**When** I save and reload
**Then** each node keeps a stable `nodeId` (not derived from topological ordering)

---

### Story 3.12: Generate v1.1 `workflow.md` + `steps/<nodeId>.md`

As a **Creator**,
I want the Builder to generate `workflow.md` and step files in **Package Spec v1.1** format（见 `_bmad-output/tech-spec.md`），
So that export artifacts (`workflow.graph.json`, `bmad.json`) can reference stable, spec-aligned files.

**Acceptance Criteria:**

**Given** I edit the workflow graph in Builder and save
**When** the Builder generates workflow documents
**Then** `workflow.md` contains v1.1 frontmatter（至少含 `schemaVersion/workflowType/currentNodeId/stepsCompleted`）
**And** step files are generated as `steps/<nodeId>.md`（文件名使用 graph `node.id`，不按拓扑顺序重编号）
**And** each step file frontmatter is v1.1-aligned（至少含 `schemaVersion/nodeId/type`，且 `nodeId == graph node.id`）

---

### Story 3.13: Generate v1.1 `workflow.graph.json` (nodes/edges + node.file)

As a **Creator**,
I want the Builder to generate `workflow.graph.json` following **Package Spec v1.1**,
So that Runtime can validate transitions and load step files deterministically.

**Acceptance Criteria:**

**Given** the Builder has a saved graph (nodes/edges)
**When** the Builder generates `workflow.graph.json`
**Then** it includes `schemaVersion/entryNodeId/nodes/edges`
**And** each node includes `id/type/file` where `file` points to the step file path in the package
**And** each edge includes `from/to/label`
**And** when a node has multiple outgoing edges, one edge is marked `isDefault: true`

---

### Story 3.14: Generate v1.1 `bmad.json` Manifest (entry/workflows)

As a **Creator**,
I want the Builder to generate `bmad.json` following **Package Spec v1.1**,
So that the exported `.bmad` package has a clear entrypoint and metadata.

**Acceptance Criteria:**

**Given** a workflow project exists
**When** the Builder generates `bmad.json`
**Then** it includes `schemaVersion/name/version/createdAt/entry`
**And** `entry.workflow/entry.graph/entry.agents` reference the generated files in the zip
**And** the manifest includes `workflows[]` listing all workflows in the package
**And** `entry` points to the default workflow (deterministic fallback if none)

---

### Story 3.15: Generate v1.1 `agents.json` (Schema-Ready)

As a **Creator**,
I want the Builder to generate `agents.json` using the v1.1 agent schema (`metadata/persona/prompts/menu/tools`),
So that Runtime can load personas/prompts/tool policy consistently and schema validation passes.

**Acceptance Criteria:**

**Given** I manage agents in ProjectBuilder and save
**When** the Builder generates `agents.json`
**Then** it has `schemaVersion: "1.1"` and an `agents[]` list with required fields
**And** default tool policy is present (`tools.fs` enabled; `tools.mcp` disabled unless configured)

---

### Story 3.16: Export v1.1 `.bmad` ZIP Bundle (structure + download)

As a **Creator**,
I want to export a v1.1-compliant `.bmad` ZIP containing all required files in the correct paths,
So that Runtime can import it without additional transforms.

**Acceptance Criteria:**

**Given** v1.1 artifacts have been generated (`bmad.json/agents.json/workflow.graph.json/workflow.md/steps/`)
**When** I click "Export Package (v1.1)"
**Then** the downloaded `.bmad` zip contains those files at the expected paths
**And** it blocks export if required artifacts are missing

---

### Story 3.17: Validate v1.1 `.bmad` Export with Schemas and Actionable Errors

As a **Creator**,
I want the Builder to validate my v1.1 `.bmad` export against the official schemas and show actionable errors,
So that I can fix issues before downloading/importing the package.

**Acceptance Criteria:**

**Given** I click "Export Package (v1.1)"
**When** the Builder validates the generated export payload using schemas from `crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/`
**Then** it validates:
  - `bmad.json` → `bmad.schema.json`
  - `workflow.graph.json` → `workflow-graph.schema.json`
  - `agents.json` → `agents.schema.json`
  - `workflow.md` frontmatter（YAML→JSON）→ `workflow-frontmatter.schema.json`
  - `steps/*.md` frontmatter（YAML→JSON）→ `step-frontmatter.schema.json`
**And** validation failures block the download and show actionable error messages (file path + schema pointer)

---

### Story 3.18: ProjectBuilder Package Assets Management (Templates)

As a **Creator**,
I want to manage package-level assets (e.g., templates) and reference them from workflow steps,
So that the exported `.bmad` contains everything needed by Runtime without relying on external files.

**Acceptance Criteria:**

**Given** I am in ProjectBuilder
**When** I create/edit/delete files under the package `assets/` namespace
**Then** the assets are persisted and listed with stable paths (e.g., `assets/templates/*.md`)
**And** I can copy/insert asset paths into workflow step `inputs/outputs` or step instructions

**Given** package assets exist
**When** I export the package (v1.1)
**Then** the ZIP contains `assets/**` at the expected paths
**And** `bmad.json.entry.assetsDir == "assets/"` continues to be valid

---

### Story 3.19: Import .bmad Package into Builder

As a **Creator**,
I want to import an existing `.bmad` package into the Builder,
So that I can edit and extend workflows created by others or migrate packages between projects.

**Acceptance Criteria:**

**Given** I am in the ProjectBuilder view
**When** I click "Import Package" and select a `.bmad` file
**Then** the Builder extracts and validates the package against v1.1 schemas
**And** a new project is created with the package name and all workflows/agents/assets loaded
**And** validation failures show actionable error messages (file path + schema pointer)

**Given** a project with the same name already exists
**When** I import a package
**Then** the Builder prompts me to rename or cancel

**Given** the `.bmad` package contains multiple workflows
**When** the import completes
**Then** all workflows are imported with their graphs, steps, and agent assignments preserved

> 详细规格见：`_bmad-output/implementation-artifacts/3-19-import-bmad-package-into-builder.md`

---

## Epic 4: Workflow Execution Engine


**Goal**: Consumers can load and execute `.bmad` workflows locally（Project-First：产物写入 ProjectRoot；state/logs 存入 RuntimeStore）。

**Dev Must Read（强制，避免实现漂移）**
- `_bmad-output/architecture/runtime-architecture.md`（Runtime 总览与硬约束）
- `_bmad-output/architecture/entrypoints-agent-vs-workflow.md`（Agent-First vs Workflow-First、停点判断、内部 loop 语义）
- `_bmad-output/tech-spec/agent-menu-command-contract.md`（Agent Menu → CommandRouter 契约）
- `_bmad-output/tech-spec/prompt-composer-examples.md`（PromptComposer 组装方式与示例）
- `_bmad-output/tech-spec/llm-conversation-protocol-openai.md`（OpenAI-compatible ToolCalls；tool 结果如何回填 messages）
- `_bmad-output/implementation-artifacts/runtime/dev-guide-create-story-micro.md`（开发/验收 Golden Path）
- `_bmad-output/implementation-artifacts/runtime/create-story-micro-ideal-trace.md`（ExecutionEngine 内部循环“对照脚本”）

**Epic 4 Definition of Done（建议）**
- 能用示例包 `crewagent-runtime/spec/bmad-package-spec/v1.1/examples/create-story-micro/` 跑通 1 次 end-to-end（对照上述 dev guide + ideal trace）。

### Story 4.1: Load `.bmad` Package (v1.1) into Runtime

As a **Consumer**,
I want to load a `.bmad` package into the Runtime Client,
So that I can execute the workflow.

**Acceptance Criteria:**

**Given** I have a `.bmad` file (ZIP)
**When** I drag-and-drop or click "Open Package" in Runtime
**Then** the ZIP is extracted into RuntimeStore package cache（例如：`<RuntimeStoreRoot>/packages/<packageId>/`，以 packageId 去重）
**And** Runtime reads `bmad.json` and validates it against `schemas/bmad.schema.json`
**And** Runtime resolves `entry.workflow/entry.graph/entry.agents` and validates all referenced files exist
**And** Runtime validates `workflow.graph.json` / `agents.json` against their schemas
**And** I see a workflow preview (name/version + step list derived from graph)
**And** the extracted package is mounted as `@pkg` and treated as read-only（不允许写入）

**Given** `bmad.json.workflows[]` is present (multi-workflow package)
**When** I open the package
**Then** Runtime lists available workflows and allows me to select one
**And** the selected workflow’s `workflow/graph` paths are used for preview and execution

**Given** the `.bmad` file is corrupted, not a ZIP, or cannot be extracted
**When** I try to open it in Runtime
**Then** I see a clear error message explaining the failure
**And** no ProjectRoot files are created/modified and no run is created
**And** the failure is recorded in the execution log

**Given** the `.bmad` file is extractable but fails v1.1 schema validation
**When** the Runtime validates the package
**Then** I see validation errors (actionable and field-specific where possible, including file path + schema location)
**And** the workflow is not loaded for execution

---

### Story 4.2: Load Workflow Graph & Step Files

As a **Runtime**,
I want to load `workflow.graph.json` as the authoritative workflow structure,
So that I can render steps and validate transitions.

**Acceptance Criteria:**

**Given** a workflow is selected from `bmad.json`
**When** the Runtime loads `workflow.graph.json`
**Then** it validates the graph against `schemas/workflow-graph.schema.json`
**And** it verifies `entryNodeId` exists and all edges reference existing nodes
**And** it verifies each node’s `file` exists inside the package (relative path)
**And** it parses each `steps/*.md` frontmatter and validates against `schemas/step-frontmatter.schema.json`
**And** `steps/*.md` 的 `nodeId` 必须与 graph 的 `node.id` 一致，否则导入失败并提示差异

---

### Story 4.3: Parse Frontmatter to Determine Current Node

As a **Runtime**,
I want to parse YAML Frontmatter from `@state/workflow.md`,
So that I can resume execution and determine the current node.

**Acceptance Criteria:**

**Given** a run is created or resumed
**When** the Runtime reads `@state/workflow.md` Frontmatter
**Then** it validates frontmatter against `schemas/workflow-frontmatter.schema.json`
**And** it reads `currentNodeId/stepsCompleted/variables/decisionLog/artifacts`
**And** if `currentNodeId` is empty, the runtime treats the workflow as “not started yet” and uses `workflow.graph.json.entryNodeId` as the first node
**And** if `currentNodeId` does not exist in the graph, the runtime blocks execution and shows a fix hint

---

### Story 4.4: Compose Complete Prompt for LLM

As a **Runtime**,
I want to assemble the complete prompt (Agent Persona + Step Instructions + Context + Tool Policy),
So that the LLM receives full context and stays within runtime guardrails.

**Acceptance Criteria:**

**Given** a node is about to execute
**When** the Prompt Composer assembles the prompt
**Then** it includes:
  - Agent persona/prompt template from `agents.json` (by `agentId`)
  - Current node step instructions from `steps/*.md`
  - Context injection（最小摘要，不直接塞大文件全文；鼓励 `fs.search → fs.read(window)` 按需取证）
  - Tool policy（fs/mcp enable + 限额）derived from `agents.json.tools`
  - Mounts & rules（`@project/@pkg/@state`；`@pkg` 只读；默认产物写 `@project/artifacts/...`）
**And** it emits machine-parseable user shells aligned with `llm-conversation-protocol-openai.md`:
  - `RUN_DIRECTIVE`（start/continue/resume）
  - `NODE_BRIEF`（stepFile、outputsMap、allowedNext）
  - `USER_INPUT`（当需要用户输入时）
**And** `variables` from `@state/workflow.md` are provided to the LLM（结构化 JSON 或占位符 substitution；并保证不会泄露真实路径如 RuntimeStoreRoot）

---

### Story 4.5: Call LLM API (OpenAI ToolCalls / Local LLM)

As a **Runtime**,
I want to send the composed prompt to the configured LLM and run a ToolCalls-based execution loop,
So that the workflow can progress with tool assistance.

**Acceptance Criteria:**

**Given** the prompt is assembled and a provider is configured
**When** the LLM Adapter sends the request
**Then** a response is received (or error is handled gracefully without crashing)
**And** the request/response uses an OpenAI-compatible `messages/tools/tool_calls/tool` protocol（DeepSeek 等 OpenAI-compatible Provider 可复用同一协议；不依赖 `developer` role）
**And** when assistant returns `tool_calls`, Runtime:
  - executes tools by `tool_calls[i].id`
  - appends `role="tool"` messages with `tool_call_id` and `content=JSON.stringify(result)`
  - continues the internal loop until the assistant returns no tool calls
**And** the “stop point” semantics match `_bmad-output/architecture/entrypoints-agent-vs-workflow.md`:
  - no tool calls + not complete → `WaitingUser`
  - complete condition met → `Completed`
**And** the user can Pause/Stop execution without corrupting `@state/workflow.md` state

---

### Story 4.6: Execute Tool Calls (Sandboxed FileSystem)

As a **Runtime**,
I want to read/write files via a sandboxed FileSystem tool host,
So that the AI can create and modify project files safely.

**Acceptance Criteria:**

**Given** the LLM requests a file tool call (e.g., `fs.read` / `fs.write` / `fs.apply_patch` / `fs.search`)
**When** the runtime FileSystem tool host processes the request
**Then** the operation only applies within the allowed mount roots: `@project/@pkg/@state`
**And** `@pkg` is read-only; any write to `@pkg/...` is rejected
**And** writes are only allowed under `@project/...` or `@state/...`
**And** any path traversal / symlink escape that resolves outside the mounted real paths is rejected (Sandbox)
**And** `fs.read` supports narrow reads (range/window) and size limits (default: read preview + `truncated=true` for large files) to control tokens
**And** `fs.search` returns structured matches (`path/line/text` + optional context; supports truncation) so LLM can “先定位再精读”
**And** `fs.write/fs.apply_patch` enforce write limits + atomic writes (tmp → rename), especially for `@state/workflow.md`
**And** all tool results follow a stable JSON shape (ToolResult union) and are safe to embed into `role="tool"` messages (see `llm-conversation-protocol-openai.md`)
**And** every tool call (including rejected ones) is audit-logged with inputs/outputs + duration to `@state/logs/execution.jsonl`

**Given** the LLM requests a file write outside `@project`/`@state` (or tries to write `@pkg`)
**When** the runtime validates the requested path
**Then** the request is rejected with a sandbox violation error
**And** no file is written outside the allowed mount roots
**And** the rejection (reason + blocked path) is surfaced in the Runtime UI and execution log

---

### Story 4.7: Validate Graph Transition & Update Frontmatter

As a **Runtime**,
I want to validate state transitions against `@pkg/workflow.graph.json` and persist state in `@state/workflow.md`,
So that execution is correct, auditable, and recoverable.

**Acceptance Criteria:**

**Given** a step has completed successfully
**When** `@state/workflow.md` frontmatter changes
**Then** the runtime validates:
  - YAML frontmatter is parseable and matches `schemas/workflow-frontmatter.schema.json`
  - `currentNodeId` is either empty (not started) or exists in graph
  - `currentNodeId` change is allowed by an outgoing edge from the previous node (graph transition guard)
  - `stepsCompleted` does not regress (no removal of completed nodes)
**And** invalid updates are rejected with a message listing allowed next nodes
**And** accepted updates are saved atomically (tmp → rename)
**And** state updates are recoverable without relying on chat history（恢复只依赖 `@state/workflow.md` + `@pkg/workflow.graph.json`）

---

### Story 4.8: Create Run Folder (RuntimeStore) & Initialize State

As a **Consumer**,
I want a new run folder created for each execution,
So that each run has isolated state/logs while writing artifacts into my ProjectRoot.

**Acceptance Criteria:**

**Given** I click "Start Execution" on a loaded package
**When** the Runtime initializes the job
**Then** a new Run Folder is created under RuntimeStore (e.g., `<RuntimeStoreRoot>/projects/<projectId>/runs/<runId>/`)
**And** `@state/` is mapped to the **run-scoped** state root (`.../runs/<runId>/state/`) and includes:
  - `@state/workflow.md` (copied from package `workflow.md` and initialized with `runId`/metadata)
  - `@state/logs/execution.jsonl` (created if missing)
**And** the Runtime persists a **run metadata record** in a project-scoped `runsIndex` (e.g., `runId`, `workflowId`, `status`, `startedAt`, `lastUpdatedAt`)
**And** `@pkg/` is mapped to the extracted package cache (read-only) and is NOT copied into the run folder
**And** `@project/` is mapped to the user-selected ProjectRoot and `@project/artifacts/` is created if missing
**And** all `fs.*` operations are sandboxed to the resolved real paths of `@project/@pkg/@state`
**And** `@state` file operations require a `runId` context (explicit parameter or active-run binding); missing context must be rejected

---

### Story 4.9: Save All Artifacts to ProjectRoot (Project-First)

As a **Runtime**,
I want all user-visible generated files to be saved within the user ProjectRoot by default,
So that outputs are organized and persisted.

**Acceptance Criteria:**

**Given** the AI generates a new file via tool call
**When** the runtime file tool host writes the file
**Then** the file is saved under `@project/...` (default `@project/artifacts/...`)
**And** the artifact path (project-relative, e.g. `artifacts/...`) is appended into `@state/workflow.md.artifacts`
**And** the artifact is listed in the Execution Log UI (backed by `@state/logs/execution.jsonl`)

---

### Story 4.10: Execute MCP Tool Calls (Stdio Driver)

As a **Runtime**,
I want to execute CLI tools via a Stdio MCP Driver,
So that the AI can run local commands and capture outputs.

**Acceptance Criteria:**

**Given** MCP is enabled for the current Agent (per `agents.json` agent policy: `tools.mcp.enabled`)
**When** the LLM requests a tool call with `mcp://stdio/run_command`
**Then** the Stdio Driver spawns the child process inside the `@project` sandbox (cwd restricted to ProjectRoot)
**And** stdout and stderr are captured (with size limits)
**And** the result (exit code + output) is returned to the LLM for next iteration

**Given** MCP is disabled (globally or for the current Agent)
**When** the LLM requests any `mcp://...` tool call
**Then** the request is rejected with a clear “MCP disabled” error
**And** no process is spawned

---

### Story 4.11: Agent-First Session (Menu) & Command Routing Contract

As a **Consumer**,
I want to start from an Agent Session (menu-driven) and route my input deterministically,
So that Runtime behaves like BMAD/Cursor: “choose agent → show menu → user input triggers workflow/exec/action”.

**Acceptance Criteria:**

**Given** I have loaded a v1.1 package that contains `agents.json`
**When** I enter an Agent Session (select an agent)
**Then** I see the agent persona + its `menu[]` rendered
**And** when I type user input, Runtime resolves it to a structured command per `_bmad-output/tech-spec/agent-menu-command-contract.md`:
  - numeric selection / exact trigger / fuzzy match / clarification when multiple candidates
  - `workflow` starts a WorkflowRun (v1.1 micro-file + graph only; classic workflow.yaml must error clearly)
  - `exec` starts ScriptRun (or redirects to WorkflowRun when exec points to workflow.md)
  - `action` dispatches to builtin or prompt action
**And** after routing to a run, ExecutionEngine follows the same internal loop/stop semantics as Workflow-First

---

## Epic 5: Observability, Settings & Recovery

**Goal**: Consumers can view execution progress, manage settings, and recover from failures.

> **Design Decision (BMad Pattern)**: Workflow dependencies are handled via:
> - `@state/workflow.md` Frontmatter state（`currentNodeId` + `stepsCompleted` + `variables/decisionLog`）
> - Prerequisite file checks only at workflow load time
> - LLM handles implicit data dependencies at runtime

### Story 5.0: Runtime UI Shell & Navigation (IA First)

As a **Consumer**,
I want a consistent Runtime application shell (navigation + core layout),
So that all later features (import/run/progress/log/settings) fit into a coherent UI and don’t require rework.

**Acceptance Criteria:**

**Given** I open the Runtime app
**When** no Project is selected yet
**Then** the left sidebar shows **Start** and **Settings** only (Settings pinned at the bottom)
**And** Start lets me **New Project** or **Open Project**
**And** New Project creates `ProjectRoot/artifacts/` and `ProjectRoot/.crewagent.json`

**Given** I open a Project
**When** the Project loads
**Then** the left sidebar shows the **Project name/icon**
**And** below it shows **Files** and **Works**
**And** Start is no longer visible while a project is active

**Given** I open a Project
**When** the Project has no `activePackageId` in `.crewagent.json` **or** the referenced package is missing locally
**Then** the Runtime shows a **blocking error overlay**
**And** it prevents entering **Works/Execution** until the package is re-imported

**Given** I open **Files**
**When** I click a file
**Then** I can view and edit the file in a **file tab**
**And** I can save changes back to `@project`
**And** file viewing is **extensible** (future file types via plugins)

**Given** I open **Works**
**When** I create a conversation
**Then** I can choose entry type: **Agent / Workflow / Chat**
**And** within a conversation I can **switch type**
**And** I can see how many workflows were started from that conversation
**And** I can select a workflow to continue its execution

**Given** I open **Settings**
**When** I configure Runtime
**Then** I can manage:
  - **Package info & cache** (active package summary, list/remove/clear, re-validate, import `.bmad`)
  - **LLM Provider** (OpenAI / OpenAI-compatible like DeepSeek / Azure / Ollama), model, endpoint, API key, test connection
  - **Theme** (system/light/dark; optional high-contrast)

**Given** I trigger an execution-related action inside a conversation
**When** the UI updates
**Then** it reflects the canonical state derived from `@state/workflow.md` + `@pkg/workflow.graph.json` (not from chat memory)

**Given** I am inside a Project
**When** I click **Files** or **Works** in the left nav
**Then** a **slide-out panel** opens from the left
**And** the main workspace stays visible on the right as **tabs**
**And** the **first tab is fixed** as **Conversation/Chat** (cannot be closed)
**And** opening files creates **additional tabs** for file viewing/editing (multiple tabs allowed)

**References**
- UX spec: `_bmad-output/ux-design-specification.md`（Runtime：Explain-Plan-Approve / State is Visible / Safety by Default）
- Runtime entrypoints: `_bmad-output/architecture/entrypoints-agent-vs-workflow.md`

---

### Story 5.10: Files Page (Project Explorer)

As a **Consumer**,
I want to view and manage files in my Project,
So that I can verify generated artifacts, edit configurations, and organize workflow files.

**Acceptance Criteria:**

**Given** a loaded Project
**When** I navigate to **Files**
**Then** a **slide-out panel** opens with a file tree rooted at **ProjectRoot** (default focus `@project/artifacts/` when present)
**And** selecting a file opens a **file tab** after the fixed Conversation tab

**Given** I open a markdown file
**When** it is previewed
**Then** it renders as Markdown and provides an **Open in OS** action
**And** unsupported types show a “No preview available” state with **Open in OS**

**Given** I perform file actions
**When** I create/rename/delete files or folders
**Then** operations are scoped to **ProjectRoot** only and path traversal is rejected
**And** file viewing is **extensible** via a viewer registry by file extension

**Dependencies**
- Story 5.0 (UI Shell & Navigation)

---

### Story 5.1: View Current Workflow State

As a **Consumer**,
I want to preview the workflow graph in the Runtime UI before I start,
So that I understand what will happen when I click Start.

**Acceptance Criteria:**

**Given** I am in a Project and have selected a workflow conversation (before starting a run)
**When** I view the workflow graph
**Then** I see the workflow flow graph (nodes + edges) so I can understand the overall process
**And** the graph does not expose per-step Markdown content/details in this story

*Note*: Overlaying execution status (completed / in-progress / pending) and highlighting the current node should be implemented after run-scoped `@state` is available (see Story 4-8 / 4-3).

---

### Story 5.2: View Execution Log

As a **Consumer**,
I want to view a log of all tool calls and their results,
So that I can debug issues.

**Acceptance Criteria:**

**Given** a workflow has been executing
**When** I click "View Log"
**Then** I see a chronological list of tool calls, inputs, and outputs
**And** errors are highlighted in red
**And** logs are backed by an append-only JSONL file under RuntimeStore (e.g., `@state/logs/execution.jsonl`)
**And** each tool call entry includes `tool_call_id`, tool name, args (sanitized), result (ToolResult JSON), and duration
**And** the log can optionally retain provider raw response fields for compatibility (e.g., OpenAI-compatible providers that require echoing extra fields)

---

### Story 5.3: Resume Execution After Crash

As a **Consumer**,
I want to resume execution from the last saved state after a crash,
So that I don't lose progress.

**Acceptance Criteria:**

**Given** the Runtime crashed or was closed during execution
**When** I reopen the Runtime and open the same ProjectRoot
**Then** the Runtime can locate the latest run for that ProjectRoot (from RuntimeStore) and offer Resume
**And** on resume, the workflow continues from `@state/workflow.md.currentNodeId` / `stepsCompleted` (per Frontmatter)
**And** no duplicate work is performed

---

### Story 5.4: Configure Tool Call Timeout

As a **Consumer**,
I want to configure a timeout for tool calls,
So that stuck tools don't block execution forever.

**Acceptance Criteria:**

**Given** I am in Runtime Settings
**When** I set "Tool Timeout" to 5 minutes
**Then** any tool call exceeding 5 minutes is terminated
**And** the user is prompted to retry or skip

---

### Story 5.5: Configure LLM Provider and Model

As a **Consumer**,
I want to select my LLM provider and model,
So that I can use my preferred AI service.

**Acceptance Criteria:**

**Given** I am in Runtime Settings
**When** I select Provider (OpenAI / OpenAI-compatible / Azure / Ollama) and enter Endpoint URL / Model Name
**Then** the configuration is saved
**And** the LLM Adapter uses these settings for API calls
**And** for OpenAI-compatible providers (e.g., DeepSeek), the Runtime uses the OpenAI `chat.completions` ToolCalls protocol (see `_bmad-output/tech-spec/llm-conversation-protocol-openai.md`)

---

### Story 5.6: Configure LLM API Key in Runtime

As a **Consumer**,
I want to configure my LLM API Key in the Runtime Client,
So that Workflows can call cloud AI services when needed.

**Acceptance Criteria:**

**Given** I am in Runtime Settings
**When** I enter an API Key for the selected provider (e.g., OpenAI/Azure) and save
**Then** the key is saved to local config (plaintext allowed)
**And** the Runtime can make LLM API calls using this key

**Given** the selected provider does not require an API key (e.g., local Ollama)
**When** I configure the provider and model
**Then** the Runtime does not require an API key to execute workflows

**Given** the API key is missing or invalid
**When** the Runtime attempts to call the provider
**Then** the Runtime surfaces a clear error message
**And** I am guided to fix the configuration (without crashing the workflow)

---

### Story 5.7: View and Manage MCP Drivers

As a **Consumer**,
I want to see which MCP Drivers are available and enable/disable them,
So that I can control what tools the AI can use.

**Acceptance Criteria:**

**Given** I am in Runtime Settings
**When** I view the "MCP Drivers" section
**Then** I see:
  - Builtin tools（例如 `fs.*`，用于 `@project/@state` 文件操作，MVP 必选）
  - MCP drivers（例如 Stdio，按需启用）
**And** I can toggle MCP drivers on/off (global)
**And** when a MCP driver is disabled, all `mcp://...` calls are rejected with a clear error

---

### Story 5.8: Persist Conversations & Messages (RuntimeStore)

As a **Consumer**,
I want conversations and message history to persist per project,
So that I can reopen a project and continue previous chats.

**Acceptance Criteria:**

**Given** I create a conversation in **Works**
**When** the conversation is created
**Then** a conversation metadata entry is persisted under RuntimeStore for the project (e.g., `conversationsIndex`)
**And** the metadata includes `{ conversationId, entryType, createdAt, lastActiveAt, workflowRuns[] }`

**Given** I send or receive messages inside a conversation
**When** messages are produced
**Then** they are appended to a conversation log file under RuntimeStore (e.g., `<RuntimeStoreRoot>/projects/<projectId>/conversations/<conversationId>/messages.jsonl`)
**And** messages include `role`, `content`, `createdAt`, and tool-call fields when present

**Given** I reopen the Runtime and open the same ProjectRoot
**When** I open **Works** and select a conversation
**Then** the conversation history is loaded from RuntimeStore and displayed
**And** I can continue the chat and append new messages

**Given** I delete a conversation
**When** the deletion is confirmed
**Then** its metadata and message log are removed from RuntimeStore
**And** no conversation data is written into ProjectRoot



### Story 5.11: Enhanced Chat Interface Foundation (Streaming & Iceberg Model)

As a **Consumer**,
I want a modern chat experience with streaming responses and a clean "Iceberg" view that hides internal thinking details,
So that I can focus on the results without being overwhelmed by logs.

**Acceptance Criteria:**

**Given** the LLM is generating a response
**When** chunks are received
**Then** the UI updates progressively (Streaming)
**And** "Thinking" blocks or "Tool Call" blocks are collapsed by default (showing only discrete animated indicators)
**And** I can click indicators to view details (or be directed to the Log tab)

**Given** I am in a chat
**When** Markdown content is rendered
**Then** it uses a dedicated `MessageMarkdown` renderer isolated from file viewers
**And** it supports standard Markdown (code blocks, lists, bold/italic) with chat-optimized styling

---

### Story 5.12: Interactive Embedded Widgets (Forms & Selection)

As a **Consumer**,
I want to interact with structured widgets (forms, lists, confirmations) directly in the chat stream,
So that I can provide precise inputs to the Agent (e.g. filling missing info, selecting execution items, providing feedback).

**Acceptance Criteria:**

**Given** the Agent needs structured user input (completing info, selecting items, feedback)
**When** it triggers an interactive widget request (via specific ToolCall or structured message)
**Then** the Chat UI renders the corresponding widget instead of plain text:
  - **Form Widget**: Based on JSON Schema (for generic info gathering)
  - **Selection Widget**: List of items with checkboxes/radio buttons (for "Select items to execute")
  - **Action Proposal Widget**: A plan/summary with "Approve" or "Modify" options (for "Next recommended actions")
  - **Multi-Item Feedback Widget**: List of items with per-item action options (for "Review each suggestion")
  - **Action Selection Widget**: List of actions with "Continue" button (for "Choose next workflow/step")

**Given** I interact with a widget
**When** I submit my choice/input
**Then** the data is validated against the schema
**And** the result is sent back to the Agent as a structured response
**And** the widget enters a "Read-Only/Submitted" state to prevent re-submission

---

### Story 5.13: Real-Time LLM Streaming Backend (SSE + IPC)

As a **Consumer**,
I want the runtime backend to stream LLM responses token-by-token,
So that the chat UI updates in real time without waiting for the full response.

**Acceptance Criteria:**

**Given** the selected LLM provider supports streaming (OpenAI / OpenAI-compatible)
**When** the runtime requests `chat.completions` with `stream: true`
**Then** the backend parses SSE chunks and emits IPC events in order:
  - `llm:stream-start` (with stable `messageId`)
  - `llm:stream-chunk` (with text `delta` and optional `partType`)
  - `llm:stream-end` once the final message is assembled

**Given** the assistant requests tool calls
**When** tool execution begins/ends
**Then** the backend emits `llm:stream-tool-start` and `llm:stream-tool-end`
**And** tool end includes `{ result, duration }`

**Given** the provider does not support streaming or a stream error occurs
**When** the runtime retries or falls back
**Then** the request completes via the non-stream path
**And** the UI still receives a final assistant message (no regressions)

**Given** a run is aborted or canceled
**When** streaming is in progress
**Then** the stream stops cleanly and no dangling IPC listeners remain

---

### Story 5.14: Tool Policy & ChatMode Alignment

*(Existing story - details in implementation-artifacts/5-14-tool-policy-and-chatmode-alignment.md)*

---

### Story 5.15: Python Script Execution Capability

As a **Consumer**,
I want the Agent to be able to execute Python scripts,
So that I can leverage Python for complex calculations, data processing, and deterministic logic.

**Acceptance Criteria:**

**Given** the Agent needs to perform a computation or data operation
**When** it calls the `python.run` tool with `code` or `file` argument
**Then** the Runtime executes the script using the **bundled Python** environment (not host system Python)
**And** returns `stdout`, `stderr`, and `exitCode` as the tool result

**Given** the Settings page
**When** I view the Engine section
**Then** I can configure "Python Execution Timeout" (default: 60 seconds)
**And** scripts exceeding this timeout are killed with an error

**Given** the Agent generates Python code
**When** it submits via `python.run(code="...")`
**Then** the code is written to a temp file, executed, and the temp file is cleaned up
**And** no user confirmation is required (auto-execution)

**Given** the Runtime installation package
**When** it is installed on a fresh macOS/Windows machine
**Then** it includes a bundled portable Python (3.11+) in `resources/python/`
**And** `python.run` works without requiring the user to install Python separately

**Feasibility Analysis:**

| Component | Current State | Integration Effort |
|:----------|:-------------|:-------------------|
| `ToolHost` interface | Extensible via `getVisibleTools()` + `executeToolCall()` | Low |
| `FileSystemToolHost` | Handles `fs.*` tools; can add `python.run` in same pattern | Low |
| `ExecutionEngine` | Already routes all tools via `ToolHost`; no changes needed | None |
| `RuntimeSettings` | Already has `engine.maxTurns`; can add `pythonTimeout` | Low |
| `SettingsPage` | Already renders engine settings; add one input field | Low |

**Integration Points (Runtime Only):**

| File | Change |
|:-----|:-------|
| `fileSystemToolHost.ts` | Add `python.run` to `getVisibleTools()` and `dispatch()` |
| `pythonService.ts` | **NEW**: Utility to resolve bundled Python path |
| `runtimeStore.ts` | Add `pythonTimeout` to `RuntimeSettings` |
| `SettingsPage.tsx` | Add "Python Timeout" input field |
| `system-base-rules.md` | Add Python usage instructions |
| `electron-builder.json5` | Configure `extraResources` to bundle Python |

**Risk Analysis:**

| Risk | Level | Mitigation |
|:-----|:------|:-----------|
| Large bundle size (~50MB) | Medium | Accept for v1 |
| Missing packages | Medium | LLM sees error and informs user |
| Security (arbitrary code) | High | Same risk as `fs.write`; Accept for v1 |
| Infinite loop | Low | Timeout handling |

**Design Decisions:**
1. **Bundled Python**: Portable Python 3.11+ in `resources/python/`
2. **Script Mode**: Fresh process per call (no REPL state)
3. **Auto-Execution**: No user confirmation required
4. **Timeout**: Configurable, default 60 seconds

> 详细规格见：`_bmad-output/implementation-artifacts/5-15-python-script-execution.md`
> 技术规格见：`_bmad-output/implementation-artifacts/tech-spec-5-15-python-script-execution.md`

---

### Story 5.16: Python Auto-Install Missing Libraries

As a **Consumer**,
I want the system to automatically install missing Python libraries when my code fails with `ModuleNotFoundError`,
So that I don't need to manually manage package installations.

**Acceptance Criteria:**

**Given** the Agent calls `python.run(code="import pandas; ...")`
**When** the script fails with `ModuleNotFoundError: No module named 'pandas'`
**Then** Runtime detects the missing module, runs `pip install pandas`, and retries the script

**Given** the Settings page → Python section
**Then** I can configure: Auto-install On/Off, PyPI mirror URL, Max retries per run

**Given** the Runtime installation
**Then** it includes small common libraries: requests, openpyxl, python-docx, PyYAML, beautifulsoup4 (~15MB)
**And** large libraries (numpy, pandas) are installed on-demand when first imported

**Platform Support:**
- **v1**: macOS Intel (x86_64)
- **Future**: macOS Apple Silicon (arm64), Windows (x64), Linux (x64)

> 详细规格见：`_bmad-output/implementation-artifacts/5-16-python-third-party-libraries.md`

---

### Story 5.17: File Drag-to-Chat Attachment

As a **Consumer**,
I want to drag files from the Files panel into the chat input area,
So that I can reference project files in my conversation with the Agent and provide context more efficiently.

**Acceptance Criteria:**

**Given** I am viewing the Files panel in a Project
**When** I drag a file node from the file tree
**Then** the drag operation starts with file metadata (path, name)

**Given** I am in a conversation with the chat input visible
**When** I drag a file over the chat input area
**Then** the input area visually indicates it can receive the drop
**And** when I drop the file, it is added to the attached files list

**Given** I have attached files to the chat input
**When** I view the chat input area
**Then** the attached files appear as tags above the text input
**And** each tag shows the file name with a remove button

**Given** I have attached files and typed a message
**When** I send the message
**Then** the message includes file metadata references (path, name)
**And** the LLM can use `fs.read` to access file content if needed
**And** the attached files list is cleared after sending

> 详细规格见：`_bmad-output/implementation-artifacts/5-17-file-drag-to-chat-attachment.md`

---

---

## Epic 8: Optimization & Enhancement

**Goal**: Optimize user experience, improve system usability, and enhance runtime efficiency.

### Story 8.1: First Launch Experience & Default Package Initialization

As a **Consumer**,
I want the Runtime to show a welcome/splash screen on first launch and automatically import a default `.bmad` package if bundled,
So that I can immediately start using the system with a guided first experience.

**Acceptance Criteria:**

**Given** the Runtime starts for the first time (first launch)
**When** the application loads
**Then** a welcome/splash screen is displayed
**And** after the splash screen completes, the setting `skipSplashScreen` is automatically set to `true`

**Given** the Runtime starts on subsequent launches
**When** the setting `skipSplashScreen` is `true`
**Then** the splash screen is skipped and the app goes directly to the Start page

**Given** the Runtime starts for the first time (empty RuntimeStore)
**When** a default package exists in `resources/default-package/`
**Then** it is automatically imported

**Given** no default package exists in `resources/default-package/`
**When** the Runtime starts
**Then** initialization continues normally (current flow)

**Given** the RuntimeStore already contains packages
**When** the Runtime starts
**Then** the default package check is skipped

**Implementation Notes:**

| Component | Change |
|:----------|:-------|
| `RuntimeSettings` | Add `skipSplashScreen: boolean` setting |
| `RuntimeStore` | Add `tryInitializeDefaultPackage()` method |
| `App.tsx` or `SplashScreen` | Show splash on first launch, auto-set skip after |
| `resources/default-package/` | Optional directory for bundled `.bmad` files |

> 详细规格见：`_bmad-output/implementation-artifacts/8-1-default-package-initialization.md`

---

### Story 8.2: Project Exit to Start Page

As a **Consumer**,
I want to exit the current Project and return to the Start page,
So that I can switch to a different project or close my current work session.

**Acceptance Criteria:**

**Given** I am inside a Project (viewing conversations, files, or settings)
**When** I click an "Exit Project" or "Close Project" button
**Then** I am returned to the Start page (Home/Welcome screen)
**And** the project state is preserved for future access

**Given** I am on the Start page
**When** I view recent projects
**Then** I can see the project I just exited and can re-open it

**Implementation Notes:**

| Component | Change |
|:----------|:-------|
| `appStore` | Add `closeProject()` action |
| Project UI | Add exit button (e.g., in sidebar or header) |

---

### Story 8.3: Settings Tab Reorder

As a **Consumer**,
I want the Settings tabs to be ordered by importance and frequency of use,
So that I can quickly access the most commonly used settings.

**Acceptance Criteria:**

**Given** I am on the Settings page
**When** I view the tab list
**Then** the tabs are ordered as:
  1. **Package** - Package management (import/remove)
  2. **LLM** - LLM provider/model configuration
  3. **MCP** - MCP server management
  4. **Tool** - Tool policy settings
  5. Remaining tabs ordered by importance (Engine, Python, Node, Appearance, etc.)

**Implementation Notes:**

| Component | Change |
|:----------|:-------|
| `SettingsPage.tsx` | Reorder tabs array |

---

### Story 8.4: Project Data Management & Orphan Recovery

As a **Consumer**,
I want to manage project data stored in RuntimeStore and recover orphan data when project folders are moved,
So that I don't lose chat history when reorganizing my files and can clean up unused data.

**Acceptance Criteria:**

**Given** the Runtime starts
**When** it detects orphan project data (original projectRoot path no longer exists)
**Then** the user is notified of orphan data in Settings or Start page

**Given** I view orphan project data in Settings
**When** I see the orphan project list
**Then** each orphan entry displays:
  - **项目名称** (from `.crewagent.json.projectName` or folder name)
  - **原路径** (truncated with tooltip for full path)
  - **最后打开时间**
  - **聊天数量**
  - **数据大小** (e.g., "2.3 MB")

**Given** I view orphan project data
**When** I click "重新绑定文件夹"
**Then** I can select a new folder to associate with the historical data
**And** the project mapping is updated
**And** all conversations and run history are preserved

**Given** I view orphan project data
**When** I click "删除数据"
**Then** a confirmation dialog appears
**And** upon confirmation, the orphan data is removed from RuntimeStore

**Given** I view orphan project data from removable storage
**When** I click "忽略(外接设备)"
**Then** the orphan is temporarily hidden until next restart

**Given** I move a project folder to a new location
**When** I open the project from the new location
**Then** if `.crewagent.json` contains a persistent `projectId`, the historical data is automatically matched
**And** I see all previous conversations

**UI Mockup:**

```
┌─────────────────────────────────────────────────────────────┐
│ 项目数据管理                                                 │
├─────────────────────────────────────────────────────────────┤
│ ⚠️ 发现 2 个孤儿项目数据                                     │
│                                                             │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 📁 my-crewagent-project                                 │ │
│ │    原路径: /Users/mengbin/old-folder/my-crewagent-...   │ │
│ │    最后打开: 2026-01-20 | 聊天: 5 条 | 大小: 2.3 MB      │ │
│ │    [重新绑定文件夹]  [删除数据]                          │ │
│ └─────────────────────────────────────────────────────────┘ │
│                                                             │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 📁 finance-workflow                                     │ │
│ │    原路径: /Volumes/USB/projects/finance-workflow       │ │
│ │    最后打开: 2026-01-15 | 聊天: 12 条 | 大小: 5.1 MB     │ │
│ │    [重新绑定文件夹]  [删除数据]  [忽略(外接设备)]         │ │
│ └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

**Implementation Notes:**

| Component | Change |
|:----------|:-------|
| `ProjectMetadata` | Add `projectId`, `projectName`, `conversationCount`, `totalSizeBytes` |
| `.crewagent.json` | Add persistent `projectId` (UUID) field |
| `RuntimeStore` | Add `detectOrphanProjects()`, `rebindProject()`, `deleteOrphanData()` |
| `SettingsPage` | Add "Project Data Management" tab/section |
| Start Page | Show orphan data warning badge (optional) |

---

### Story 8.5: Smart Conversation Naming

As a **Consumer**,
I want conversations to be automatically named based on content,
So that I can easily identify and find my conversation history.

**Problem:**

Multiple Workflow conversations show the same truncated name (e.g., "Workflow: Supply Chain Deci..."), making it hard to distinguish between them.

**Solution: Smart Naming**

使用 LLM 根据对话内容自动生成有意义的名称。

**Acceptance Criteria:**

**Given** I start a new conversation and send my first message
**When** the first assistant reply completes
**Then** the system uses LLM to generate a short, descriptive title (≤30 chars)
**And** the title reflects the key topic or intent of the conversation
**And** replaces the default "Workflow: xxx" or "New Conversation" name

**Given** I continue an existing conversation
**When** the topic changes significantly (optional enhancement)
**Then** the system may suggest updating the title

**Given** I want to manually rename a conversation
**When** I click on the conversation name
**Then** I can edit it inline
**And** the custom name overrides the auto-generated name

**Given** I hover over a truncated conversation name
**When** the tooltip appears
**Then** I see the full conversation title

**Smart Naming Examples:**

| First Message Content | Generated Title |
|:----------------------|:----------------|
| "帮我分析这个供应链决策..." | "供应链决策分析" |
| "设计一个V型带轮..." | "V带轮设计计算" |
| "报销流程的合规检查" | "报销合规检查" |
| "Create user story for login" | "Login Feature Story" |

**Implementation Notes:**

| Component | Change |
|:----------|:-------|
| `ConversationMetadata` | Add `generatedTitle`, `customTitle` fields |
| LLM call after first exchange | Generate title (simple prompt, fast model) |
| Conversation list UI | Show generated/custom title, inline edit |
| Conversation item | Add tooltip for full name |

**Prompt Example:**

```
Generate a short title (max 30 chars) for this conversation based on the user's first message:

User: {first_user_message}

Requirements:
- Concise and descriptive
- Capture the main topic/intent
- No quotes or special characters
```

---

### Story 8.6: Gemini LLM Provider Support

As a **Consumer**,
I want to use Google Gemini as my LLM provider,
So that I can leverage Gemini's capabilities for workflow execution.

**Acceptance Criteria:**

**Given** I am on the Settings → LLM page
**When** I select "Gemini" from the provider dropdown
**Then** I see Gemini-specific configuration fields:
  - API Key
  - Model selection (gemini-pro, gemini-1.5-pro, etc.)
  - Base URL (optional, for custom endpoints)

**Given** I have configured Gemini as my provider
**When** I execute a workflow or chat with an agent
**Then** the request is sent to the Gemini API
**And** tool calls work correctly (function calling)

**Given** Gemini API returns an error
**When** the error occurs
**Then** a clear error message is shown to the user

**Implementation Notes:**

| Component | Change |
|:----------|:-------|
| `LLMProvider` type | Add `'gemini'` option |
| `LlmAdapter` | Add Gemini API support (OpenAI-compatible or native) |
| Settings UI | Add Gemini provider option and fields |

**Note:** Gemini supports OpenAI-compatible API format, so may reuse existing `OpenAICompatibleLlmAdapter` with different base URL.

---

## Design Notes


### Dependency Handling (BMad Pattern)

CrewAgent follows **BMad's "Document-as-State"** pattern:

1. **State Tracking**: `@state/workflow.md` frontmatter（`currentNodeId` + `stepsCompleted` + `variables/decisionLog/artifacts`，Spec v1.1）
2. **Prerequisites**: Checked only at workflow load (not per-step)
3. **Execution Order**: Graph-driven (`workflow.graph.json` as source of truth). LLM chooses the next edge; runtime validates transitions.
4. **Data Dependencies**: Implicit - LLM reads prior artifacts as needed
5. **No Parallel Execution** (MVP)

This keeps the runtime simple while preserving LLM flexibility.
