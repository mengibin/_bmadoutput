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
- FR-MNG-01: System must create a dedicated Project Folder for each new job/execution.
- FR-MNG-02: System must save all generated artifacts within the Project Folder.
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
- Database: PostgreSQL via Supabase
- ORM: SQLAlchemy + Alembic (Backend)
- Auth: Username/Password via Supabase Auth
- Deployment: Vercel (Frontend), Railway (Backend), GitHub Releases (Runtime)

**LLM-as-Engine Architecture:**
- Runtime does NOT implement traditional workflow engine.
- Build "Thin Shell": Prompt Composer, Tool Executor, State Manager, LLM Adapter.

### FR Coverage Map

| FR ID | Epic | Description |
|:---|:---|:---|
| FR-DEF-01~05 | Epic 3 | Workflow definitions, Step Chaining, Agent Manifest, Package Export, JSON Validation |
| FR-BLD-01~05 | Epic 3 | Node Graph, Graph-First Editing, Agent Forms, Prompt Templates, One-Click Export |
| FR-RUN-01~06 | Epic 4 | Package Load, Frontmatter State, Document-as-State, Agent Injection, Context Injection, Pause/Resume |
| FR-INT-01~04 | Epic 4 | Stdio MCP, stdout/stderr capture, FileSystem MCP, Sandboxed Access |
| FR-MNG-01~04 | Epic 5 | Project Folder, Artifact Storage, Execution Log, State UI |
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
**FRs covered**: Architecture Auth Requirements (Supabase Auth)
**Deliverables**:
- Login page
- Registration flow
- JWT authentication
- Supabase Auth verification in FastAPI backend

---

### Epic 3: Workflow Visual Builder
**Goal**: Creators (Simin) can design workflows and agents through a visual interface, export as `.bmad` packages.
**FRs covered**: FR-DEF-01~05, FR-BLD-01~05
**Deliverables**:
- React Flow canvas with workflow nodes
- Step/Agent configuration forms
- Markdown auto-generation from graph
- `.bmad` package export

---

### Epic 4: Workflow Execution Engine
**Goal**: Consumers (David) can load and execute `.bmad` workflows locally.
**FRs covered**: FR-RUN-01~06, FR-INT-01~04, NFR-SEC-01~02, NFR-USAB-01~02
**Deliverables**:
- Package loader (ZIP extraction)
- LLM Adapter (OpenAI + Ollama)
- MCP Drivers (Stdio, FileSystem)
- State Manager (Frontmatter R/W)
- Prompt Composer

---

### Epic 5: Observability, Settings & Recovery
**Goal**: Consumers can view execution progress, manage settings/drivers, and recover from failures.
**FRs covered**: FR-MNG-01~04, NFR-REL-01~02, NFR-SEC-01, NFR-USAB-01~02
**Deliverables**:
- Project folder management
- Execution log UI
- Workflow state visualization
- Crash recovery (resume from Frontmatter)
- Runtime settings (LLM provider/model, API keys)
- MCP driver management (enable/disable)

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
**Then** my account is created via Supabase Auth
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

### Story 2.3: Integrate Supabase Auth in Backend

As a **Developer**,
I want to integrate Supabase Auth verification in the FastAPI backend,
So that API endpoints are protected.

**Acceptance Criteria:**

**Given** a request comes with a JWT token in the Authorization header
**When** the backend validates the token with Supabase
**Then** authenticated requests proceed to the handler
**And** unauthenticated requests receive 401 Unauthorized

**Given** a request has a missing/expired/invalid JWT token
**When** the backend validates the token with Supabase
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
**Then** a `.bmad` ZIP file is generated containing `workflow.md`, step files, and `agents.json`
**And** the package is validated against the JSON Schema before download

---

## Epic 4: Workflow Execution Engine

**Goal**: Consumers can load and execute `.bmad` workflows locally with project folder management.

### Story 4.1: Load .bmad Package into Runtime

As a **Consumer**,
I want to load a `.bmad` package into the Runtime Client,
So that I can execute the workflow.

**Acceptance Criteria:**

**Given** I have a `.bmad` file
**When** I drag-and-drop or click "Open Package" in Runtime
**Then** the ZIP is extracted to a temporary location
**And** the package is validated against JSON Schema
**And** I see the workflow preview

**Given** the `.bmad` file is corrupted, not a ZIP, or cannot be extracted
**When** I try to open it in Runtime
**Then** I see a clear error message explaining the failure
**And** no project state is created/modified
**And** the failure is recorded in the execution log

**Given** the `.bmad` file is extractable but fails JSON Schema validation
**When** the Runtime validates the package
**Then** I see validation errors (actionable and field-specific where possible)
**And** the workflow is not loaded for execution

---

### Story 4.2: Parse Frontmatter to Determine Current Step

As a **Runtime**,
I want to parse YAML Frontmatter from `workflow.md`,
So that I know which step to execute next.

**Acceptance Criteria:**

**Given** a workflow is loaded
**When** the Runtime reads `workflow.md` Frontmatter
**Then** `stepsCompleted` array is parsed
**And** the next incomplete step is identified

---

### Story 4.3: Compose Complete Prompt for LLM

As a **Runtime**,
I want to assemble the complete prompt (Agent Persona + Step Instructions + Context),
So that the LLM receives full context.

**Acceptance Criteria:**

**Given** a step is about to execute
**When** the Prompt Composer assembles the prompt
**Then** it includes Agent's System Prompt, Step's User Prompt, and prior artifacts
**And** placeholder variables are substituted

---

### Story 4.4: Call LLM API (OpenAI / Ollama)

As a **Runtime**,
I want to send the composed prompt to the configured LLM,
So that I get an AI response.

**Acceptance Criteria:**

**Given** the prompt is assembled and API Key is configured
**When** the LLM Adapter sends the request
**Then** a response is received (or error is handled)
**And** tool call requests are extracted if present

---

### Story 4.5: Execute MCP Tool Calls (Stdio Driver)

As a **Runtime**,
I want to execute CLI tools via Stdio MCP Driver,
So that the AI can run local commands.

**Acceptance Criteria:**

**Given** the LLM requests a tool call with `mcp://stdio/run_command`
**When** the Stdio Driver spawns the child process
**Then** stdout and stderr are captured
**And** output is returned to the LLM for next iteration

---

### Story 4.6: Execute MCP Tool Calls (FileSystem Driver)

As a **Runtime**,
I want to read/write files via FileSystem MCP Driver,
So that the AI can create and modify project files.

**Acceptance Criteria:**

**Given** the LLM requests a tool call with `mcp://fs/write_file`
**When** the FileSystem Driver processes the request
**Then** the file is written to the Project Folder
**And** paths outside Project Folder are rejected (Sandbox)

**Given** the LLM requests a file write outside the Project Folder
**When** the FileSystem Driver validates the requested path
**Then** the request is rejected with a sandbox violation error
**And** no file is written outside the Project Folder
**And** the rejection (reason + blocked path) is surfaced in the Runtime UI and execution log

---

### Story 4.7: Update Frontmatter on Step Completion

As a **Runtime**,
I want to update `stepsCompleted` in Frontmatter when a step finishes,
So that execution state is persisted.

**Acceptance Criteria:**

**Given** a step has completed successfully
**When** the State Manager updates `workflow.md`
**Then** the step ID is added to `stepsCompleted` array
**And** the file is saved to disk

---

### Story 4.8: Create Execution Project Folder

As a **Consumer**,
I want a new project folder created for each execution,
So that each run has an isolated workspace.

**Acceptance Criteria:**

**Given** I click "Start Execution" on a loaded package
**When** the Runtime initializes the job
**Then** a new Project Folder is created under the OS-appropriate app data directory (e.g., `%LOCALAPPDATA%\\CrewAgent\\projects\\{job-id}\\` on Windows)
**And** the extracted package files are copied there
**And** all file operations are sandboxed to this Project Folder by default

---

### Story 4.9: Save All Artifacts to Project Folder

As a **Runtime**,
I want all generated files to be saved within the Project Folder,
So that outputs are organized and persisted.

**Acceptance Criteria:**

**Given** the AI generates a new file via tool call
**When** the FileSystem Driver writes the file
**Then** the file is saved under the current Project Folder
**And** the artifact is listed in the Execution Log

---

## Epic 5: Observability, Settings & Recovery

**Goal**: Consumers can view execution progress, manage settings, and recover from failures.

> **Design Decision (BMad Pattern)**: Workflow dependencies are handled via:
> - `stepsCompleted` Frontmatter array for state tracking
> - Prerequisite file checks only at workflow load time
> - LLM handles implicit data dependencies at runtime

### Story 5.1: View Current Workflow State

As a **Consumer**,
I want to see the current workflow state in the Runtime UI,
So that I know the execution progress.

**Acceptance Criteria:**

**Given** a workflow is loaded or executing
**When** I look at the main panel
**Then** I see which steps are completed, in-progress, and pending
**And** the current step is highlighted

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

---

### Story 5.3: Resume Execution After Crash

As a **Consumer**,
I want to resume execution from the last saved state after a crash,
So that I don't lose progress.

**Acceptance Criteria:**

**Given** the Runtime crashed or was closed during execution
**When** I reopen the Runtime and load the same project folder
**Then** the workflow resumes from the last completed step (per Frontmatter)
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
**When** I select Provider (OpenAI / Azure / Ollama) and enter Endpoint URL / Model Name
**Then** the configuration is saved
**And** the LLM Adapter uses these settings for API calls

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
**Then** I see a list of available drivers (Stdio, FileSystem, etc.)
**And** I can toggle each driver on/off

---

## Design Notes

### Dependency Handling (BMad Pattern)

CrewAgent follows **BMad's "Document-as-State"** pattern:

1. **State Tracking**: `stepsCompleted: [1, 2, 3]` in Frontmatter
2. **Prerequisites**: Checked only at workflow load (not per-step)
3. **Execution Order**: Sequential, determined by LLM reading state
4. **Data Dependencies**: Implicit - LLM reads prior artifacts as needed
5. **No Parallel Execution** (MVP)

This keeps the runtime simple while preserving LLM flexibility.


