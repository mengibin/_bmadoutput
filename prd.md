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
*   **OS Strategy**: **macOS (Priority #1)** and **Windows** (Priority #2).
    *   *Constraint*: Must run on typical Engineering Workstations (often restricted environments). Linux support is deprioritized.
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
*   **Client Runtime**: macOS support (priority), Windows support, Local FileSystem Driver, Terminal Driver.
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
*   **FR-MNG-01**: System must support selecting/opening a user **Project Folder** (`ProjectRoot`), and create a dedicated **Run Folder** (private RuntimeStore) for each new execution.
*   **FR-MNG-02**: System must save user-visible artifacts (code, docs, intermediates) within `ProjectRoot` by default (e.g., `ProjectRoot/artifacts/`), while keeping package cache, run state, and logs in a private RuntimeStore by default.
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

## Appendix A — Runtime Client Detailed Spec (MVP)

> 本附录将 `_bmad-output/tech-spec/runtime-spec.md` 的内容并入 PRD，作为 Runtime 需求/约束的更细化版本（Cursor 风格：工具硬实现，流程由 LLM 驱动；ToolCalls 优先，MCP 预留）。
>
> `.bmad` 包格式（schemas/templates/examples）的权威技术规范见：[`_bmad-output/tech-spec.md`](tech-spec.md)。

### A.1 `.bmad` 包（v1.1）约定

#### A.1.1 ZIP 结构（最小可跑）

```text
{name}.bmad  (zip)
  bmad.json                 # 包清单/入口（必选）
  workflow.graph.json       # 图结构（必选，权威真源）
  workflow.md               # 入口文档 + Frontmatter 状态（必选）
  steps/                    # micro-file steps（必选）
    step-01-*.md
    decide-*.md
    end-*.md
  agents.json               # Agent persona + prompts + tool policy（必选）
  assets/                   # 可选：图/模板/静态文件

  # 可选：多 workflow（一个包多个流程）
  workflows/<workflow-id>/
    workflow.graph.json
    workflow.md
    steps/
      ...
```

#### A.1.2 Step 来源与顺序（micro-file + graph）

- **来源**：节点文件在 `steps/*.md`（每个 node 一个文件，step/decision/merge/end 都可建模成 node）。
- **权威顺序/跳转**：以 `workflow.graph.json` 为准（Builder 渲染画布；Runtime 校验 `currentNodeId` 的合法跳转）。
- **索引（给 LLM 快速打开）**：`workflow.md` 正文建议包含 steps 索引链接，但它不是“真源”，只是可读索引。

#### A.1.3 `workflow.md` Frontmatter（Document-as-State）

必须字段（MVP/分支友好）：
- `schemaVersion: "1.1"`（与包 spec 对齐）
- `workflowType: string`
- `currentNodeId: string`
- `stepsCompleted: string[]`（已完成 nodeId 集合）
- `variables: object`（分支变量，默认 `{}`）
- `decisionLog: Array<{from,to,label,reason?,decidedAt?}>`（默认 `[]`）

建议字段（增强可恢复与可审计）：
- `runId: string`（uuid）
- `project_name`, `user_name`, `date`
- `inputDocuments: string[]`（上下文注入入口）
- `artifacts: string[]`（runtime 记录输出物路径）
- `updatedAt: string`（ISO datetime）

### A.2 Runtime 组件拆分（Electron）

#### A.2.1 Main Process（能力与安全边界）

- `ProjectManager`：打开/切换 `ProjectRoot`（用户工程目录），确保默认产物目录（如 `ProjectRoot/artifacts/`）存在。
- `RuntimeStore`：应用私有目录根（推荐 Electron `app.getPath('userData')`）；保存包缓存、run state、logs（用户不可见、不污染项目）。
- `PackageManager`：导入 `.bmad`（zip 解压到 RuntimeStore/packages/）、JSON Schema 校验、生成包清单（`bmad.json`）；如 `bmad.json.workflows[]` 存在，则在 UI 提供“选择要运行的 workflow”（默认使用 `entry` 指定的入口）。
- `RunManager`：创建/恢复 Run（一次执行实例），在 RuntimeStore/projects/<projectId>/runs/<runId>/ 下初始化 `@state/workflow.md` 与日志目录。
- `WorkflowStateStore`：frontmatter 读写（解析/校验/原子写 tmp→rename）。
- `GraphStore`：加载 `workflow.graph.json`，提供 node/edge 查询与跳转合法性校验（`currentNodeId -> nextNodeId`）。
- `StepIndex`：为 UI/LLM 生成“可读索引”（可从 graph + steps 生成；也可回填到 `workflow.md` 正文）。
- `AgentRegistry`：加载 `agents.json`，提供 persona/prompt 模板。
- `PromptComposer`：拼 system/user prompt（包含 tool 规范 + 沙箱规则）。
- `LLMAdapter`：先实现 OpenAI ToolCalls；本地模型后续兼容（可先不做）。
- `ToolHost`：内置 ToolCalls（文件/patch/搜索/日志写入），并在写入时执行沙箱与校验（frontmatter YAML 可解析；可选：状态跳转必须符合 graph）。
- `ExecutionEngine`：对话编排器（LLM ↔ ToolCalls）。不替 LLM 决策走哪条分支；Runtime 只做“硬边界”（工具/沙箱/校验/日志）与“合法性校验”（next node 必须是图允许的后继）。
- `LogStore`：JSONL 追加写（执行记录/工具调用/错误），供 UI 展示。

#### A.2.2 Renderer（体验与可视化）

- Package 导入/历史
- Workflow 预览（steps 列表 + 进度）
- Chat 流式输出
- ToolCalls 面板（显示执行的读写/patch）
- Artifacts 面板（生成文件列表）
- Settings（provider/model/apiKey/timeout）

### A.3 执行模型：ToolCalls 优先（MVP 主干）

#### A.3.1 Cursor 兼容执行循环（推荐默认）

1) Runtime 打开 run：把三类 mount 暴露给 LLM（仅暴露 alias，不泄露真实路径）：
   - `@project/...` → ProjectRoot（读写，默认产物写 `@project/artifacts/...`）
   - `@pkg/...` → `.bmad` 解包内容（只读）
   - `@state/...` → 本次 run 的私有状态（读写，用户不可见），核心文件 `@state/workflow.md`
2) Runtime 发送初始指令（system 级约束 + tool policy），要求 LLM：
   - 读取 `@state/workflow.md` frontmatter 的 `currentNodeId/stepsCompleted/variables`
   - 根据 `@pkg/workflow.graph.json` 的出边决定 next node，并读取对应 `@pkg/steps/<node-file>.md`
   - 完成本步产出后，用普通文件工具更新 `@state/workflow.md` frontmatter（追加 `stepsCompleted`、更新 `currentNodeId/variables/decisionLog/updatedAt/artifacts`），并把产物写到 `@project/...`（默认 `@project/artifacts/`）
3) 进入 `chatLoop`：
   - LLM 输出（可能包含 toolCalls）
   - ToolHost 执行 toolCalls → 把结果回给 LLM
4) 结束条件（由 LLM/用户决定）：LLM 明确告知完成，或用户点击 Pause/Stop。
5) UI 通过监听/重读 `@state/workflow.md` frontmatter 展示进度（current node 高亮、stepsCompleted 进度、artifacts 列表等）。

#### A.3.2 严格模式（可选）：专用状态更新工具

默认模式：LLM 直接用 `fs.apply_patch/fs.write` 更新 `@state/workflow.md`；Runtime 负责原子落盘、YAML 校验、跳转合法性校验。  
可选增强：提供 `runtime.update_state(...)` / `runtime.complete_node(...)` 由 Runtime 负责读-改-写与校验（作为开关不影响 Cursor 兼容性）。

### A.4 安全与可靠性（必须从一开始做）

#### A.4.1 文件沙箱（NFR-SEC-02）

- 所有 fs 工具只能访问 mount roots：`@project/@pkg/@state`
  - Runtime 负责把 mount alias 映射到真实路径，并在 `realpath` 后做前缀校验，防 `..` 和 symlink escape
  - `@pkg` 强制只读；任何写入直接拒绝
- 单次读取大小限制（例如 512KB/文件）
- 写入默认走 `apply_patch`；`write` 只允许写到 `@project` 或 `@state`
- 对包含 YAML frontmatter 的写入（尤其 `@state/workflow.md`）建议在落盘前解析校验，失败则拒绝并返回可修复错误

#### A.4.2 超时与资源限制（NFR-REL-02）

- ToolCalls 默认超时 5min（可配置）。
- stdout/stderr（未来 MCP）需要截断与大小上限。

#### A.4.3 可恢复（NFR-REL-01）

- crash 后只需重新加载 run：
  - 从 `@state/workflow.md` 的 `currentNodeId/stepsCompleted/variables` 恢复 next node
  - 从 `@state/logs/execution.jsonl` 恢复审计线索（不要求恢复完整对话）

### A.5 MCP（后续接入的重工具通道）

- 本地文件/轻操作：先用 ToolCalls（内置 ToolHost）
- 复杂外部工具：后续接入严格 MCP（stdio/jsonrpc）

### A.6 MVP 里程碑（建议）

1) 导入 `.bmad` → 选择 Project → 创建 run（RuntimeStore）→ 展示 steps 列表  
2) 实现 `fs.read/fs.write/fs.apply_patch` ToolCalls（含沙箱与 frontmatter 校验）  
3) 让 LLM 跑通 step-01 → 写入 artifact → 更新 frontmatter  
4) 增加 `fs.search`、完善日志与 UI  
5) （可选）严格模式状态更新工具  
6) 之后再引入 MCP（stdio driver）与审批/权限  

### A.7 子流程（Sub-Workflow）支持（v1.2+）

建议通过 node 类型 + `callStack` 实现 call/return；并做循环依赖检测与最大嵌套深度限制。

## Document Completion

**PRD Status**: ✅ Complete
**Completed By**: Mengbin
**Completion Date**: 2025-12-23

This document serves as the **Capability Contract** for CrewAgent. All downstream UX Design, Architecture, and Development work should trace back to the requirements defined herein.

**Recommended Next Steps:**
1. **Create Architecture** (`/bmad-bmm-workflows-create-architecture`) - Define system design based on technical constraints.
2. **Create Epics & Stories** (`/bmad-bmm-workflows-create-epics-and-stories`) - Break down FRs into implementable work units.
3. **Create UX Design** (`/bmad-bmm-workflows-create-ux-design`) - Design the Visual Builder and Client UI.


