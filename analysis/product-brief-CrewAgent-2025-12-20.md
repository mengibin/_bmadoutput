---
stepsCompleted: [1, 2, 3, 4, 5]
inputDocuments: 
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/analysis/research/technical-architecture-decision-research-2025-12-20.md'
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/analysis/brainstorming-session-2025-12-19.md'
workflowType: 'product-brief'
lastStep: 6
project_name: 'CrewAgent'
user_name: 'Mengbin'
date: '2025-12-20'
---

# Product Brief: CrewAgent

**Date:** 2025-12-20
**Author:** Mengbin

---

---

## Executive Summary

**CrewAgent** is a **Universal Agent Operating System** for industry verticals—essentially a "Meta-BMAD" framework for domain experts. It solves the fragmentation between "Static Documentation" and "Dynamic Engineering Tools" by providing a platform where **workflows can be visually defined** (Platform side) and **locally executed** (Client side).

Unlike rigid SaaS solutions, CrewAgent separates the **Definition** (The "Brain" & "Flow") from the **Runtime** (The "Hands"). This allows an industry expert to package their know-how into a "Definition Bundle," which a standard client can execute using the user's own LLM and local software license. It transforms the Engineer's role from a "Manual Data Porter" to a "Supervisor of Digital Experts."

---

## Core Vision

### Problem Statement
**The "Last Mile" Disconnect in Engineering.**
Despite advanced CAD/CAE tools, the final delivery of engineering work relies on static documents (Word/PDF). Engineers waste 50%+ of their time serving as "human middleware"—manually running calculations, retrieving standards, and copying data into reports. This "Data Bifurcation" between the *Calculated Truth* (in tools) and the *Delivered Truth* (in docs) creates inefficiency, liability, and a lack of traceability.

### Why Existing Solutions Fall Short
*   **Generic AI (ChatGPT)**: Cannot access local, specialized tools (SolidWorks, ERP) or proprietary standards.
*   **Vertical SaaS**: Expensive, hard to customize, and data privacy concerns (cloud processing).
*   **Custom Scripts**: Hard to maintain, fragile, and require coding skills that domain experts lack.

### Proposed Solution
**A Decoupled "Bionic-Agent" Architecture:**
1.  **The Platform (Definition Layer)**: A No-Code Visual Builder where experts define "Agents" and "Workflows" (e.g., "Gear Design Flow"). This generates a standard **Definition Package** (prompts, RAG references, step logic).
2.  **The Client (Runtime Layer)**: A lightweight, universal desktop shell. It loads the "Definition Package," connects to the user's **Local LLM API Key**, and drives **Local Tools** via standard drivers (MCP).

### Key Differentiators
1.  **Definition-Driven Capability**: The client is a generic runner; the *intelligence* makes it specialized. One day it's a "Mechanical Designer," the next an "Audit Bot."
2.  **Zero-Integration Cost**: Uses **Model Context Protocol (MCP)** to standardize tool connections, removing the need for custom bespoke integrations.
3.  **Privacy & Control**: "Your Key, Your Tools, Your Local Data." The logic runs where the data lives.

## Target Users

### Primary Users

#### The Creator: "Implementation Engineer" (Simin, 28)
*   **Role**: Agent Architect / Digitalization Specialist.
*   **Context**: Tech-savvy bridge between domain experts and the software.
*   **Pain Point**: "The Chief Engineer knows *what* to do, but he can't code. I need to translate his 'Expertise' into an 'Agent', but writing Python scripts for every step is too slow."
*   **Motivation**: Wants to be the **"Agent Builder"**—using the Platform to quickly package expert knowledge into a product.
*   **Key Interaction**: Interviews Expert -> Uses Visual Builder -> Publishes `.bmad` package.

#### The Consumer: "Mechanical Design Engineer" (David, 25)
*   **Role**: Project Executor.
*   **Pain Point (Data)**: "I have to follow complex ISO/GB standards I haven't memorized. I spend 4 hours copying calculation results into Word docs."
*   **Pain Point (Project)**: "I have 50 files for one project (Calcs, Specs, Drawings). I lose track of versions. If I update a calculation, I forget to update the referencing data in the Spec document."
*   **Success**: "The software manages my project coherence. All documents are linked and consistent. I load the 'Gear Agent', it guides me, runs the calc tool, and generates the report."

### Secondary Users

#### The Source: "Domain Expert" (Chief Wang, 55)
*   **Role**: Technical Director / Knowledge Source.
*   **Interaction**: Doesn't use the software directly, but provides the *Logic* and *Standards* to Simin.
*   **Win**: "Finally, my design standards are enforced across the entire project lifecycle."

### User Journey

#### The "Simin" Journey (Definition)
1.  **Consult**: Simin sits with Chief Wang to map out the "Gearbox Design Process".
2.  **Define**: Simin logs into **CrewAgent Platform**, drags nodes ("Check Spec", "Run Calc", "Gen Report"), and uploads Wang's PDF standards.
3.  **Publish**: Simin exports the "Gearbox-App-v1.bmad" package.

#### The "David" Journey (Execution)
1.  **Setup**: David opens **CrewAgent Client**, creates "Project X", and loads "Gearbox-App-v1.bmad".
2.  **Execute**: The Agent asks for "torque input". David types "500Nm".
3.  **Drive**: The Agent runs the local `GearCalc.exe` (via MCP), gets the diameter.
4.  **Manage**: The Agent saves the calculation file to the Project Folder and updates the "Design Spec.docx" with the new diameter.
5.  **Deliver**: David reviews the linked documents and exports the package.

## Success Metrics

### User Success (The Efficiency Metrics)

#### For Creators (Speed & Agility)
*   **"1-Week Launch"**: Enable building a complete V1 Industry Agent package within 5 working days using the Visual Builder + LLM assistance.
*   **"24-Hour Turnaround"**: Capability to modify and redeploy workflow logic within 1 day based on Expert feedback.

#### For Consumers (Relief)
*   **"80% Doc Reduction"**: Compress documentation/reporting tasks from **1 Week -> 1 Day**.
*   **"Daily Driver"**: Engineers open CrewAgent daily as their primary "Work Orchestrator".

### Business Objectives (The Value Model)

#### Hybrid Revenue Stream
*   **Base Fee**: Enterprise Platform fee (Setup).
*   **Seat License**: Recurring SaaS fee for Client Runtime.
*   **App Sales**: Sales of specific "Industry Packages" (Content).

#### Key Performance Indicators (KPIs)
*   **Time-to-Value**: Speed for a new Implementation Engineer to ship their first package.
*   **Active Usage**: Daily Active Users (DAU) among core design teams in deployed enterprises.

## MVP Scope

### Pilot Scenario (The "Meta-Pilot")
**Scenario**: **"BMAD Software Development Lifecycle (SDLC)"**
To validate the platform's capability to handle complex, specialized workflows, we will implement **The BMAD Method itself** as the first "Industry App".
*   **Input**: User Requirement (e.g., "Build a Snake Game").
*   **Flow**:
    1.  **Product Manager Agent**: Analyzes requirements -> Writes PRD.
    2.  **Architect Agent**: visualizes system -> Writes `architecture.md`.
    3.  **Project Manager Agent**: Breaks down PRD -> Creates `story.md`.
    4.  **Developer Agent**: Reads Story -> Writes Code & Tests (via Terminal Driver).
*   **Output**: A fully functional software repository.

### Core Features

#### Platform (The Visual Builder)
*   **Flow Editor**: Drag-and-drop orchestration of complex, multi-agent chains (Reqs -> Arch -> Code).
*   **Context Management**: "Project-Level Memory" capability, allowing downstream Agents (Dev) to read artifacts created by upstream Agents (Arch).
*   **Package Export**: One-click generation of the `.bmad` Industry Package.

#### Client (The Runtime Shell)
*   **Agent Runner**: A GUI that loads `.bmad` packages and executes the workflow steps interactively.
*   **Workspace Manager**: Manages the "Project Folder," keeping track of all generated artifacts (PRDs, Drawings, Code).
*   **Standard Drivers**: Built-in **FileSystem MCP** (File I/O) and **Terminal MCP** (Command Execution) to support the software dev pilot.

### Out of Scope for MVP
*   **Cloud Services**: No cloud sync, user account systems, or online marketplace. (Local First).
*   **Plugin Marketplace**: Drivers must be manually installed or built-in; no "App Store" yet.
*   **Real-time Collaboration**: Single-player mode only.

### MVP Success Criteria
*   **"The Snake Game Test"**: Can a non-coder use CrewAgent (loaded with the BMAD Package) to generate a working Snake Game from a single prompt, without leaving the CrewAgent interface?
*   **Platform Viability**: Successfully exporting the BMAD workflow from the Visual Builder without hand-coding the YAML files.




