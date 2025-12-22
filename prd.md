---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11]
inputDocuments: 
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/analysis/product-brief-CrewAgent-2025-12-20.md'
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/analysis/research/technical-architecture-decision-research-2025-12-20.md'
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/analysis/brainstorming-session-2025-12-19.md'
documentCounts:
  briefs: 1
  research: 1
  brainstorming: 1
  projectDocs: 0
workflowType: 'prd'
lastStep: 11
workflowComplete: true
project_name: 'CrewAgent'
user_name: 'Mengbin'
date: '2025-12-20'
---

# Product Requirements Document - CrewAgent

**Author:** Mengbin
**Date:** 2025-12-20

## Executive Summary

**CrewAgent** is a **Universal Agent Operating System** for industry verticals—essentially a "Meta-BMAD" framework. It solves the **"Last Mile Disconnect"** in engineering by decoupling **Agent Definition** (Platform) from **Runtime Execution** (Client). It transforms the engineer's role from a "Manual Data Porter" to a "Supervisor of Digital Experts."

### What Makes This Special

*   **Decoupled "Bionic-Agent" Architecture**: The "Brain" (Logic/Prompts) lives in the Cloud Definition, while the "Hands" (Tool Execution) run on the Local Client. This ensures capability without compromising data privacy.
*   **Standardized Integration (via MCP)**: Unlike fragile, version-specific plugin ecosystems, CrewAgent uses the **Model Context Protocol (MCP)** to standardize how Agents drive local tools (CAD, ERP, Terminals). This abstracts integration complexity.
*   **Template-First Definition**: To avoid "Visual Spaghettification," the platform prioritizes a **Template-First** experience. Experts start with proven industry patterns (e.g., "Parameter Validation Flow") and customize them, rather than building complex logic node-by-node from scratch.
*   **Privacy & Control**: "Your Key, Your Tools, Your Local Data." The logic processes run locally where the sensitive design data resides.

## Project Classification

**Technical Type:** Hybrid (SaaS Platform + Desktop Client)
**Domain:** Engineering Productivity (Industrial Software)
**Complexity:** High (Visual Orchestration, Local Process Control, MCP Integration)
**Project Context:** Greenfield - Creating a new platform from scratch.

This classification indicates a high need for **robust architecture** (handling state between Cloud/Local) and **strong UX patterns** (simplifying complex logic for non-coders).

## Success Criteria

### User Success
*   **For Creators (Simin)**:
    *   **"Template-First Velocity"**: 90% of new agents start from a high-quality template.
    *   **"1-Week Launch"**: Enable building a complete V1 Industry Agent package within 5 working days using the Visual Builder + LLM assistance.
*   **For Consumers (David)**:
    *   **"80% Doc Reduction"**: Compress documentation/reporting tasks from **1 Week -> 1 Day**.
    *   **"Project Coherence"**: 100% of generated documents maintain correct cross-references (no version mismatch).

### Business Success
*   **Hybrid Revenue Stream**: Successful validation of the "Platform Fee + Seat License + App Sales" model with first pilot customers.
*   **Engagement**: >60% Daily Active Users (DAU) among design teams in deployed pilots.
*   **Ecosystem Trust**: Successful deployment of "Signed Packages" running securely in enterprise environments without IT blocking.

### Technical Success
*   **Standardized Integration**: Zero custom code required for 3 major tools (FileSystem, Terminal, CSV/Excel) using standard MCP.
*   **Platform Viability**: Successfully exporting complex BMAD workflows from the Visual Builder without hand-coding YAML.

## Product Scope

### MVP - Minimum Viable Product (The "Meta-Pilot")
**Scenario**: **"BMAD Software Development Lifecycle (SDLC)"**
To validate the platform's capability, we will implement **The BMAD Method itself** as the first "Industry App".
*   **Core Features**: Valid Visual Builder (Platform) + Agent Runner (Client).
*   **Capabilities**: Requirements Analysis -> Architecture Design -> Code Generation.
*   **Boundaries**: Local-first execution, Standard MCP drivers only, Single-player mode.

### Growth Features (Post-MVP)
*   **Marketplace**: Public sharing of Industry Packages.
*   **Cloud Operations**: Team collaboration, cloud-hosted runtime options.
*   **Advanced Drivers**: Complex CAD/CAE integrations (SolidWorks, Ansys).

### Vision (Future)
*   **"The App Store for Engineering Knowledge"**: A thriving ecosystem where domain experts monetize their knowledge as "Digital Agents," and engineers access a "Bionic Workforce" on demand.

## User Journeys

### The "Simin" Journey (Creator) - Template Velocity
*   **Scene**: Simin needs to digitize a "Gear Design Process" but is overwhelmed by the blank canvas.
*   **Action**: She selects the **"Mechanical Calc Template"** from the library. It comes with pre-wired nodes for "Standard Check", "Calc Run", and "Report Gen".
*   **Refinement**: She replaces the generic "Calc Node" with her company's `Gear.exe` using the **Terminal Driver** config, and updates the prompt.
*   **Result**: She publishes "Gear-App-v1.bmad" in 2 hours, not 2 weeks.

### The "David" Journey (Consumer) - Project Coherence
*   **Scene**: David starts "Project X" but has 3 different Excel sheets called "Final_v2".
*   **Action**: He opens CrewAgent, creates "Project X", and loads Simin's package.
*   **Execution**: The Agent guides him: "Input Torque?". He types it. The Agent runs the calc, saves `result.json` to the **managed project folder**, and auto-fills `Design_Spec.docx`.
*   **Result**: David exports the final package. The Agent guarantees that the numbers in the Word doc match the calculation result.

### Journey Requirements Summary
*   **Template Library**: Integrated UI for choosing starting points.
*   **Driver Config UI**: Simple form to bind local executables (path, args) to nodes.
*   **Project Context Manager**: Runtime system that creates a dedicated folder per project and tracks file versions.

## Domain-Specific Requirements

### Engineering Reliability & Trust
Since CrewAgent operates in high-precision domains, "Hallucination" is not just an annoyance—it's a critical failure mode. The platform must enforce reliability patterns.

### Key Domain Concerns
*   **Validation**: Engineering outputs must be verifiable. We cannot blindly trust LLM-generated parameters.

### Compliance Requirements
*   **Mandatory Self-Check**: All Agents must include a "Validation Step" before Final Output.
    *   *Example*: If an agent calculates a gear diameter, it must run a reverse-check formula or compare against a standard range *before* showing the result to the user.

### Implementation Considerations
*   **"Guardrail Nodes"**: The Visual Builder should provide specific "Validator Nodes" (e.g., "Range Check", "Regex Match", "Script Validator") that can be placed after any Generation Node.

## Innovation & Novel Patterns

### Detected Innovation Areas
*   **The "Decoupled Bionic" Pattern**: Separating "Cloud Brain" (Definition) from "Local Hands" (Runtime). This hybrid architecture allows SaaS-level intelligence to drive air-gapped, local industrial tools safely.
*   **"Meta-Agent" Definition**: Encapsulating expert logic into a transferable file format (`.bmad`) rather than hard-coded scripts, enabling a "No-Code" agent creation experience for domain experts.

### Validation Approach
*   **"The Snake Game Test"**: Validating the Meta-Agent concept by having it orchestrate a complete SDLC from a single prompt.
*   **Metric**: "1-Week Launch" proves the efficiency of the definition model over traditional custom coding.

### Risk Mitigation
*   **Manual Override**: If the Agent logic fails or hallucinates, the user is already in the Runtime Client (a desktop app) and can manually take control of the local tools, ensuring no dead-ends.

## Technical Constraints & Architecture

### Desktop Client (Runtime)
*   **OS Strategy**: **Windows (Priority #1)** and **Linux** (Priority #2).
    *   *Constraint*: Must run on typical Engineering Workstations (often restricted environments). macOS support is deprioritized.
*   **Framework**: Electron (or Tauri if performance dictates) for cross-platform Node.js access.
*   **Offline Mode**: Essential requirement. Client must function with zero internet access after initial license check.

### SaaS Platform (Builder)
*   **Authentication**: **Simple Username/Password** only.
    *   *Constraint*: No external OAuth (Google/GitHub) to avoid enterprise firewall issues or third-party dependency in restricted networks.
*   **Core UI**: React Flow for Visual Node Graph.

### Integration Layer
*   **Protocol**: **Model Context Protocol (MCP)** over DBus/Stdio.
*   **Execution Model**: Local Child Processes. The Agent spawns tools directly on the host machine, ensuring speed and data privacy.

## Project Scoping & Phased Development

### MVP Strategy & Philosophy

**MVP Approach:** "Pilot-Driven Platform" (Vertical Tracer Bullet)
**Philosophy**: Build *only* the features needed to support the **BMAD SDLC Pilot**. If the platform cannot support the creation and execution of the "Snake Game" agent, it is not ready.

### MVP Feature Set (Phase 1)

**Core User Journeys Supported:**
*   **The "Simin" Journey**: Creating the SDLC Agent using a basic Visual Builder.
*   **The "David" Journey**: Executing the SDLC Agent to generate code (Snake Game) using local drivers.

**Must-Have Capabilities:**
*   **Visual Builder (Basic)**: Drag-and-drop flow editing, Node configuration.
*   **Client Runtime**: Windows support, Local FileSystem Driver, Terminal Driver.
*   **Definition Format**: `.bmad` package export/import with JSON Schema validation.

### Post-MVP Features

**Phase 2: Growth (The "Company Ready")**
*   **Multi-Tenancy**: Organization concepts, Team sharing.
*   **Security**: Package Signing (Enterprise Trust).
*   **Linux Support**: Beta release for specific engineering workstations.

**Phase 3: Expansion (The "Ecosystem")**
*   **Marketplace**: Public sharing of Industry Packages.
*   **3rd Party Drivers**: CAD (SolidWorks), ERP (SAP) integration drivers.
*   **Cloud Sync**: Optional cloud backup for projects.

### Risk Mitigation Strategy

**Technical Risks:**
*   **Complex Graph Logic**: *Mitigation*: Start with "Linear Flows" only for MVP to ensure stability before adding branching/looping complexity.
*   **Local Tool Failure**: *Mitigation*: Strong "Stderr Capturing" in the Terminal Driver to report tool errors back to the Agent context.

**Market Risks:**
*   **"Too Generic"**: *Mitigation*: The "Pilot-First" focus ensures we solve at least ONE specific problem (SDLC) extremely well, proving the model.

## Functional Requirements

### Definition Capabilities (The Package Format)
*   **FR-DEF-01**: System must support Workflow definitions as **Markdown files (`.md`)** with **YAML Frontmatter** for state tracking.
*   **FR-DEF-02**: System must support **Step Chaining**: A main `workflow.md` can reference sub-step files (e.g., `steps/step-01.md`).
*   **FR-DEF-03**: System must support **Agent Manifest** as a structured file (CSV or JSON) defining agent personas (name, role, style, principles).
*   **FR-DEF-04**: System must export complete workflow definitions as a portable **`.bmad` package** (ZIP bundle containing workflow files, agent definitions, and assets).
*   **FR-DEF-05**: System must validate imported `.bmad` packages against a JSON Schema before loading.

### Creation Capabilities (The Visual Builder)
*   **FR-BLD-01**: Creators can visually render a Markdown Workflow as an interactive **Node Graph** (read-only visualization of `.md` structure).
*   **FR-BLD-02**: Creators can edit Workflow steps via **Graph-First Editing**: Users manipulate the visual graph (add/delete/connect nodes, configure properties via forms), and the system **auto-generates** the underlying Markdown files. No direct Markdown editing required.
    *   *Rationale*: This makes the tool accessible to domain experts who don't know Markdown syntax.
*   **FR-BLD-03**: Creators can define **Agent Personas** via a form-based UI (populating the `agent-manifest.csv`).
*   **FR-BLD-04**: Creators can configure **Prompt Templates** (System Prompt, User Prompt) within each Agent definition.
*   **FR-BLD-05**: Creators can one-click export the current project as a `.bmad` package.

### Execution Capabilities (The Runtime Engine)
*   **FR-RUN-01**: Consumers can load a `.bmad` package into the local Runtime Client.
*   **FR-RUN-02**: Runtime must execute by **reading Frontmatter state** (`stepsCompleted`) and determining the next step.
*   **FR-RUN-03**: Runtime must **update Frontmatter in-place** when a step completes (Document-as-State).
*   **FR-RUN-04**: Runtime must inject the correct **Agent Persona** (from manifest) as LLM System Prompt based on current step configuration.
*   **FR-RUN-05**: Runtime must support **Context Injection** (passing artifacts from previous steps to current step's prompt).
*   **FR-RUN-06**: Runtime must support **Pause/Resume**: User can close the Client and resume from the last saved Frontmatter state.

### Integration Capabilities (MCP Drivers)
*   **FR-INT-01**: Runtime must support **Stdio MCP Driver** for spawning local CLI tools (e.g., `python`, `node`, `git`).
*   **FR-INT-02**: Runtime must capture `stdout` and `stderr` from spawned tools and inject them back into the Agent context.
*   **FR-INT-03**: Runtime must support **FileSystem MCP Driver** for reading/writing files within the Project Folder.
*   **FR-INT-04**: Runtime must enforce **Sandboxed File Access**: All file operations must be scoped to the Project's designated folder.

### Management Capabilities (Project & State)
*   **FR-MNG-01**: System must create a dedicated **Project Folder** for each new job/execution.
*   **FR-MNG-02**: System must save all generated artifacts (code, docs, intermediates) within this Project Folder.
*   **FR-MNG-03**: System must maintain an **Execution Log** recording each tool call and its result for debugging.
*   **FR-MNG-04**: Consumers can view the current **Workflow State** (which step is active, what's completed) in the Client UI.

## Non-Functional Requirements

### Reliability & Availability
*   **NFR-REL-01**: Runtime must support **Graceful Recovery**: If the Client crashes, the next launch must resume from the last saved Frontmatter state with no data loss.
*   **NFR-REL-02**: All Tool Calls must have a **Configurable Timeout** (default 5 minutes) to prevent infinite hangs on unresponsive local tools.

### Security & Privacy
*   **NFR-SEC-01**: **LLM API Key Storage** is user-configurable. The Client will support plaintext configuration files; secure storage (OS vault) is optional and not enforced.
*   **NFR-SEC-02**: **Sandboxed Execution**: Spawned child processes must be restricted to the Project Folder by default; no access to parent directories or system paths unless explicitly configured by the user.

### Usability (Offline Mode)
*   **NFR-USAB-01**: The Runtime Client must be **fully operational offline** after initial license activation. No "phone home" or cloud dependency is required for execution.
*   **NFR-USAB-02**: The Client must support connecting to **local LLM endpoints** (e.g., Ollama, LM Studio) for completely air-gapped operation.

---

## Document Completion

**PRD Status**: ✅ Complete
**Completed By**: Mengbin
**Completion Date**: 2025-12-21

This document serves as the **Capability Contract** for CrewAgent. All downstream UX Design, Architecture, and Development work should trace back to the requirements defined herein.

**Recommended Next Steps:**
1. **Create Architecture** (`/bmad-bmm-workflows-create-architecture`) - Define system design based on technical constraints.
2. **Create Epics & Stories** (`/bmad-bmm-workflows-create-epics-and-stories`) - Break down FRs into implementable work units.
3. **Create UX Design** (`/bmad-bmm-workflows-create-ux-design`) - Design the Visual Builder and Client UI.









