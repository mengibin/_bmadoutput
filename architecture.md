---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8]
workflowComplete: true
inputDocuments:
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/prd.md'
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/analysis/product-brief-CrewAgent-2025-12-20.md'
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/analysis/research/technical-architecture-decision-research-2025-12-20.md'
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/analysis/brainstorming-session-2025-12-19.md'
workflowType: 'architecture'
lastStep: 8
project_name: 'CrewAgent'
user_name: 'Mengbin'
date: '2025-12-21'
---

# Architecture Decision Document

_This document builds collaboratively through step-by-step discovery. Sections are appended as we work through each architectural decision together._

## Project Context Analysis

### Requirements Overview

**Functional Requirements (20 FRs across 5 capability areas):**

| Capability Area | FRs | Architectural Implication |
|:---|:---:|:---|
| **Definition** | 5 | Markdown Parser, YAML Frontmatter R/W, JSON Schema Validation, ZIP Archiver |
| **Creation** | 5 | React Flow (Graph Editor), Markdown Code Generator |
| **Execution** | 6 | Lightweight "Orchestrator", State Machine, LLM API Client |
| **Integration** | 4 | Child Process Manager, MCP Stdio Protocol, Sandboxed FS |
| **Management** | 4 | Project Folder Manager, Execution Log, State UI |

**Non-Functional Requirements (NFRs that drive architecture):**
*   **Reliability**: Crash recovery = State MUST persist to disk (Frontmatter).
*   **Security**: Sandboxed Execution = Child processes need `cwd` restriction.
*   **Usability (Offline)**: Must support local LLM (Ollama) = LLM adapter layer must be abstracted.

### Technical Constraints & Dependencies

*   **OS**: Windows (P1), Linux (P2). macOS deprioritized.
*   **Auth**: Simple Username/Password.
*   **Client Framework**: Electron (or Tauri if performance dictates).
*   **Integration Protocol**: MCP over Stdio.
*   **Offline Mode**: Mandatory.

### Scale & Complexity

*   **Complexity Level**: HIGH (Hybrid SaaS + Desktop)
*   **Primary Domain**: Full-Stack (Web Builder + Desktop Runtime)
*   **Estimated Components**: 
    *   SaaS: Visual Builder (React Flow), Auth, Package Export
    *   Desktop: Runtime Client, MCP Drivers, State Manager

### Cross-Cutting Concerns Identified

1.  **State Synchronization**: Builder <-> Package <-> Runtime must all agree on `.bmad` format.
2.  **Sandbox Security**: All tool executions must be scoped to Project Folder.
3.  **LLM Abstraction**: Must support OpenAI API, Azure OpenAI, and local LLMs (Ollama).

---

## Key Architectural Decision: LLM-as-Engine

> **Decision**: The Runtime will **NOT** implement a traditional "workflow engine" or state machine in application code. Instead, **the LLM itself IS the engine**.

**Rationale (BMad Pattern):**
In BMad, there is no Python or Node.js code "executing" the workflow. The LLM reads the Markdown, parses Frontmatter, decides the next step, and issues Tool Calls. We replicate this pattern.

**What we DO build (the "Thin Shell"):**

| Module | Responsibility |
|:---|:---|
| **Prompt Composer** | Assembles the complete prompt (System Prompt + Step Instructions + Context) |
| **Tool Executor** | Receives Tool Call requests from LLM, dispatches to MCP Drivers |
| **State Manager** | Reads/Writes Frontmatter in `workflow.md` to persist state |
| **LLM Adapter** | Abstracts LLM API calls (supports OpenAI, Ollama, etc.) |

**What we do NOT build:**
*   Complex state machine logic.
*   Explicit "step transition" graph in code.
*   Workflow DSL compiler.

**Implication**: The system's "intelligence" is entirely in the Markdown workflow definitions and the LLM's reasoning. The Client is a "dumb shell" that connects the LLM to local tools.

---

## Starter Template Evaluation

### Primary Technology Domains

CrewAgent is a **Hybrid** project requiring TWO distinct codebases:

1.  **Web Builder (SaaS)**: A visual workflow editor accessed via browser.
2.  **Runtime Client (Desktop)**: A local application that executes workflows.

### Selected Starters

#### 1. Web Builder: Next.js + React Flow + Tailwind

**Selected Starter:** `create-next-app` with TypeScript template

**Initialization Command:**
```bash
npx create-next-app@latest crewagent-builder --typescript --tailwind --eslint --app --src-dir
```

**Architectural Decisions Provided:**
| Aspect | Decision |
|:---|:---|
| **Framework** | Next.js 14+ (App Router) |
| **Language** | TypeScript |
| **Styling** | Tailwind CSS |
| **Routing** | App Router (file-based) |
| **Build Tool** | Turbopack (dev) / Webpack (prod) |

**Additional Dependencies to Install:**
*   `@xyflow/react` (React Flow for node graph editor)
*   `yaml` (YAML parsing for Frontmatter)
*   `jszip` (Package creation)
*   `ajv` (JSON Schema validation)

---

#### 2. Runtime Client: electron-vite + React + Tailwind

**Selected Starter:** `electron-vite` with React + TypeScript template

**Initialization Command:**
```bash
npm create @electron-vite@latest crewagent-runtime -- --template react-ts
```

**Architectural Decisions Provided:**
| Aspect | Decision |
|:---|:---|
| **Desktop Framework** | Electron 28+ |
| **Build Tool** | Vite (extremely fast HMR) |
| **Renderer** | React + TypeScript |
| **Styling** | Tailwind CSS (add manually) |
| **IPC** | Electron IPC for main <-> renderer |

**Additional Dependencies to Install:**
*   `yaml` (YAML parsing for Frontmatter)
*   `openai` or `ai` (LLM API client)
*   `@anthropic-ai/sdk` (if Claude support needed)

---

### Monorepo Consideration

> **Decision**: Keep the two projects as **separate repositories** for MVP.

**Rationale**:
*   Simplifies CI/CD pipelines (different build targets).
*   Avoids monorepo tooling complexity (Turborepo/Nx learning curve).
*   MVP scope is limited; no need for shared code yet.

**Future**: If significant code sharing emerges (e.g., `.bmad` parsing library), extract to a shared npm package.

---

## Core Architectural Decisions

### Updated Technology Stack (Python Backend)

> **Adjustment**: Backend changed from Node.js to **Python (FastAPI)** per user preference.

| Layer | Technology | Purpose |
|:---|:---|:---|
| **Builder Frontend** | Next.js 14+ (App Router) + Tailwind | Visual Workflow Editor UI |
| **Builder Backend** | **Python 3.11+ / FastAPI** | Auth, Package CRUD, API |
| **Runtime Client** | Electron + Vite + React | Desktop Workflow Executor |

---

### Data Architecture

| Decision | Choice | Rationale |
|:---|:---|:---|
| **Database (Builder)** | PostgreSQL (via Supabase) | Managed service, built-in Auth, generous free tier |
| **ORM** | SQLAlchemy + Alembic | FastAPI standard, mature, excellent migration support |
| **Client Local Storage** | Plain JSON Files | Minimal complexity for API keys, execution logs, preferences |

---

### Authentication & Security

| Decision | Choice | Rationale |
|:---|:---|:---|
| **Auth Method** | Username/Password (Supabase Auth) | PRD constraint: no OAuth |
| **API Security** | JWT Tokens (Supabase-issued) | Standard, stateless |
| **Client API Keys** | User-configurable, plaintext allowed | PRD NFR-SEC-01 |

---

### API & Communication

| Decision | Choice | Rationale |
|:---|:---|:---|
| **API Style** | REST (FastAPI auto-docs) | Simple, OpenAPI spec generated automatically |
| **Package Download** | REST Endpoint returning `.bmad` file | Client calls `GET /packages/{id}/download` |
| **Error Handling** | HTTP Status Codes + JSON Error Body | Standard REST conventions |

---

### Infrastructure & Deployment

| Decision | Choice | Rationale |
|:---|:---|:---|
| **Frontend Hosting** | Vercel | Optimized for Next.js, auto-HTTPS, free tier |
| **Backend Hosting** | Railway or Fly.io | Easy Python deployment, auto-scale |
| **Database Hosting** | Supabase | Managed PostgreSQL, includes Auth |
| **Client Distribution** | GitHub Releases + Auto-Update | Electron standard practice |

---

### Project Structure (3 Repositories)

```
crewagent/
├── crewagent-builder-frontend/   # Next.js
├── crewagent-builder-backend/    # FastAPI
└── crewagent-runtime/            # Electron
```

> **Rationale**: Separate repos simplify CI/CD. Backend and Frontend could be monorepo later if needed.

---

## Implementation Patterns & Consistency Rules

### Naming Conventions

| Category | Convention | Example |
|:---|:---|:---|
| **Database Tables** | snake_case, plural | `users`, `workflow_packages` |
| **Database Columns** | snake_case | `created_at`, `user_id` |
| **API Endpoints** | kebab-case, plural | `/api/v1/workflow-packages` |
| **Python Functions** | snake_case (PEP 8) | `get_user_by_id()` |
| **Python Classes** | PascalCase | `WorkflowPackage` |
| **TypeScript Variables** | camelCase | `workflowId`, `getUserData()` |
| **React Components** | PascalCase | `<WorkflowEditor />`, `<NodePanel />` |
| **File Names (Python)** | snake_case | `user_service.py` |
| **File Names (React)** | PascalCase for components | `WorkflowEditor.tsx` |

---

### API Response Format

**Standard Response Wrapper:**
```json
{
  "data": { ... },
  "error": null
}
```

**Error Response:**
```json
{
  "data": null,
  "error": {
    "code": "AUTH_001",
    "message": "Invalid credentials"
  }
}
```

**HTTP Status Codes:**
*   `200` - Success
*   `201` - Created
*   `400` - Bad Request (validation error)
*   `401` - Unauthorized
*   `404` - Not Found
*   `500` - Internal Server Error

---

### Data Exchange Formats

| Format | Convention |
|:---|:---|
| **JSON Fields (API)** | camelCase (auto-converted via Pydantic `alias_generator`) |
| **Date/Time** | ISO 8601, UTC (`"2025-12-21T10:00:00Z"`) |
| **Boolean** | `true` / `false` (not 1/0) |
| **Null Handling** | Explicit `null`, never omit key |

---

### Testing Conventions

| Category | Convention |
|:---|:---|
| **Python Tests** | Co-located: `user_service.py` → `user_service_test.py` |
| **React Tests** | Co-located: `WorkflowEditor.tsx` → `WorkflowEditor.test.tsx` |
| **Test Framework (Python)** | pytest |
| **Test Framework (React)** | Vitest + React Testing Library |

---

### Error Handling Patterns

**Python (FastAPI):**
*   Use custom `HTTPException` with `code` and `message`.
*   Log errors with context before raising.
*   Never expose stack traces to API responses.

**TypeScript (React/Electron):**
*   Use try/catch with typed error objects.
*   Display user-friendly messages, log full errors to console/file.
*   Use React Error Boundaries for component failures.

---

### Enforcement Guidelines

**All AI Agents MUST:**
1.  Follow naming conventions exactly as documented above.
2.  Use the standard API response wrapper format.
3.  Write co-located tests for all new modules.
4.  Use ISO 8601 UTC for all timestamps.

---

## Project Structure & Directory Layout

### 1. Builder Frontend (`crewagent-builder-frontend/`)

**Technology**: Next.js 14 (App Router) + Tailwind CSS + React Flow

```
crewagent-builder-frontend/
├── README.md
├── package.json
├── next.config.js
├── tailwind.config.js
├── tsconfig.json
├── .env.local.example
├── .gitignore
├── src/
│   ├── app/
│   │   ├── globals.css
│   │   ├── layout.tsx
│   │   ├── page.tsx                    # Landing / Dashboard
│   │   ├── login/
│   │   │   └── page.tsx
│   │   ├── dashboard/
│   │   │   └── page.tsx
│   │   └── editor/
│   │       └── [workflow_id]/
│   │           └── page.tsx            # Workflow Editor (React Flow)
│   ├── components/
│   │   ├── ui/                         # Button, Input, Modal, etc.
│   │   ├── workflow/                   # WorkflowCanvas, NodePanel, EdgeTypes
│   │   └── forms/                      # AgentConfigForm, StepConfigForm
│   ├── lib/
│   │   ├── api-client.ts               # FastAPI client (fetch wrapper)
│   │   ├── auth.ts                     # Auth utilities
│   │   └── utils.ts
│   └── types/
│       └── index.ts
└── public/
    └── assets/
```

---

### 2. Builder Backend (`crewagent-builder-backend/`)

**Technology**: Python 3.11+ / FastAPI + SQLAlchemy + Alembic

```
crewagent-builder-backend/
├── README.md
├── requirements.txt
├── pyproject.toml
├── .env.example
├── alembic.ini
├── .gitignore
├── app/
│   ├── __init__.py
│   ├── main.py                         # FastAPI entry point
│   ├── config.py                       # Settings (pydantic-settings)
│   ├── database.py                     # SQLAlchemy engine & session
│   ├── models/                         # SQLAlchemy ORM Models
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── workflow_package.py
│   ├── schemas/                        # Pydantic Request/Response Schemas
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── workflow_package.py
│   ├── routers/                        # API Endpoints
│   │   ├── __init__.py
│   │   ├── auth.py
│   │   ├── users.py
│   │   └── packages.py
│   ├── services/                       # Business Logic
│   │   ├── __init__.py
│   │   ├── auth_service.py
│   │   └── package_service.py
│   └── utils/
│       ├── __init__.py
│       └── security.py                 # JWT, password hashing
├── migrations/                         # Alembic migrations
│   └── versions/
└── tests/
    ├── __init__.py
    ├── conftest.py                     # pytest fixtures
    ├── test_auth.py
    └── test_packages.py
```

---

### 3. Runtime Client (`crewagent-runtime/`)

**Technology**: Electron + Vite + React + TypeScript

```
crewagent-runtime/
├── README.md
├── package.json
├── electron.vite.config.ts
├── tsconfig.json
├── tailwind.config.js
├── .env.example
├── .gitignore
├── src/
│   ├── main/                           # Electron Main Process
│   │   ├── index.ts                    # Main entry (BrowserWindow)
│   │   ├── ipc-handlers.ts             # IPC message handlers
│   │   └── mcp-drivers/                # MCP Driver implementations
│   │       ├── index.ts
│   │       ├── stdio-driver.ts         # Child process spawning
│   │       └── filesystem-driver.ts    # Sandboxed file R/W
│   ├── renderer/                       # React UI (Renderer Process)
│   │   ├── index.html
│   │   ├── main.tsx
│   │   ├── App.tsx
│   │   ├── components/
│   │   │   ├── ExecutionPanel.tsx
│   │   │   ├── LogViewer.tsx
│   │   │   └── SettingsModal.tsx
│   │   └── lib/
│   │       ├── llm-adapter.ts          # OpenAI/Ollama abstraction
│   │       ├── state-manager.ts        # Frontmatter R/W
│   │       └── prompt-composer.ts      # Prompt assembly
│   ├── preload/                        # Electron Preload Script
│   │   └── index.ts
│   └── shared/                         # Types shared between main/renderer
│       └── types.ts
├── resources/                          # Icons, assets for packaging
└── tests/
    └── main/
        └── mcp-drivers.test.ts
```

---

### Integration Boundaries

| Interface | Mechanism | Description |
|:---|:---|:---|
| **Frontend ↔ Backend** | REST API (HTTPS) | CRUD for packages, Auth |
| **Backend → DB** | SQLAlchemy ORM | PostgreSQL via Supabase |
| **Runtime → LLM** | OpenAI/Ollama API | Via `llm-adapter.ts` |
| **Runtime → Local Tools** | MCP (Stdio) | Via `mcp-drivers/` |
| **Main ↔ Renderer** | Electron IPC | Via `ipc-handlers.ts` |

---

## Architecture Validation Results

### Coherence Validation ✅

| Check | Result | Notes |
|:---|:---:|:---|
| **Technology Compatibility** | ✅ | Next.js, FastAPI, Electron are all mature, compatible with each other |
| **Pattern Consistency** | ✅ | Naming conventions align across Python (snake_case) and TypeScript (camelCase) |
| **Structure Alignment** | ✅ | 3-repo structure supports clear ownership and independent deployments |

### Requirements Coverage Validation ✅

| Category | Coverage | Mapping |
|:---|:---:|:---|
| **Definition (FR-DEF-01~05)** | 5/5 | Backend: `package_service.py`, Schemas: `workflow_package.py` |
| **Creation (FR-BLD-01~05)** | 5/5 | Frontend: `components/workflow/`, React Flow |
| **Execution (FR-RUN-01~06)** | 6/6 | Runtime: `llm-adapter.ts`, `state-manager.ts`, `prompt-composer.ts` |
| **Integration (FR-INT-01~04)** | 4/4 | Runtime: `mcp-drivers/stdio-driver.ts`, `filesystem-driver.ts` |
| **Management (FR-MNG-01~04)** | 4/4 | Runtime: Project Folder, Execution Log (TBD in implementation) |
| **NFR-REL (Reliability)** | 2/2 | Frontmatter persistence, Tool timeout |
| **NFR-SEC (Security)** | 2/2 | Sandboxed FS, API key storage |
| **NFR-USAB (Offline)** | 2/2 | `llm-adapter.ts` supports Ollama |

### Implementation Readiness ✅

*   **Decision Completeness**: All critical decisions documented with rationale.
*   **Structure Completeness**: Full directory trees for all 3 repositories.
*   **Pattern Completeness**: Naming, API format, testing conventions defined.

### Gap Analysis

| Priority | Gap | Mitigation |
|:---|:---|:---|
| **Nice-to-Have** | `.bmad` Package JSON Schema | Define during Epics/Stories phase |
| **Nice-to-Have** | CI/CD Pipeline YAML | Add as implementation story |

---

## Document Completion

**Architecture Status**: ✅ Complete
**Completed By**: Mengbin
**Completion Date**: 2025-12-21

### AI Agent Implementation Guidelines

1.  **Follow all patterns exactly** as documented in "Implementation Patterns" section.
2.  **Respect project structure** boundaries defined in "Project Structure" section.
3.  **Use LLM-as-Engine** approach: Build the Thin Shell (Prompt Composer, Tool Executor, State Manager), NOT a traditional workflow engine.
4.  **Validate all MCP tool executions** are sandboxed to Project Folder.

### Recommended Next Steps

1.  `/bmad-bmm-workflows-create-epics-and-stories` - Break down FRs into implementable stories.
2.  `/bmad-bmm-workflows-create-ux-design` - Design the Visual Builder and Runtime Client UIs.
3.  **Initialize Repositories**:
    *   `npx create-next-app@latest crewagent-builder-frontend --typescript --tailwind --eslint --app --src-dir`
    *   `npm create @electron-vite@latest crewagent-runtime -- --template react-ts`
    *   Create `crewagent-builder-backend/` with FastAPI template.






