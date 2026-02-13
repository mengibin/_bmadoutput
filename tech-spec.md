---
project_name: "CrewAgent"
date: "2025-12-23"
updated: "2026-02-04"
sources:
  - "crewagent-runtime/spec/bmad-package-spec/v1.1/README.md"
  - "crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/*.json"
  - "crewagent-runtime/spec/bmad-package-spec/v1.1/templates/*"
  - "crewagent-runtime/spec/bmad-package-spec/v1.1/examples/*"
  - "crewagent-runtime/spec/tech-spec-subworkflow.md"
---

# Technical Specification — `.bmad` Package Spec v1.2 (Upgrade Plan)

本技术规范用于指导 **CrewAgent Runtime（Electron）** 加载与执行 `.bmad` 工作流包，并作为 Builder 导出/Runtime 导入的契约（Contract）。
需求侧（Runtime 的行为约束与验收）参见：[`_bmad-output/prd.md`](prd.md) 的 “Appendix A — Runtime Client Detailed Spec (MVP)”。

与 Runtime 实现相关的补充文档（已归档到 `_bmad-output/`）：
- `_bmad-output/architecture/runtime-architecture.md`
- `_bmad-output/tech-spec/runtime-spec.md`
- `_bmad-output/tech-spec/agent-menu-command-contract.md`
- `_bmad-output/tech-spec/prompt-composer-examples.md`
- `_bmad-output/tech-spec/llm-conversation-protocol-openai.md`

## 0. 目标与原则

- 目标：像 Cursor 一样——运行时硬实现工具能力（读写/patch/搜索/沙箱/日志），工作流推进与分支选择主要交给 LLM。
- 原则：用结构化图数据保证 Builder/Runtime **可校验、可恢复**；用 `workflow.md` frontmatter 保持 **Document-as-State**。

> 本文将 `crewagent-runtime/spec/bmad-package-spec/v1.1/README.md` 及其配套的 schemas/templates/examples 合并为一份可执行的规范文档。

## 0.1 v1.2 升级范围（Subworkflow + Portable Skills）

- **Subworkflow**：新增子流程引用与 call/return 语义，支持调用栈恢复。
- **Run State 拆分**：新增 `@state/run.md` 记录 `activeWorkflowId` 与 `callStack`，每个 workflow 独立状态文件。
- **Agent Skills**：新增 `skills` 结构，明确能力与包内脚本导入，工具权限与角色绑定。
- **进度监控**：支持层级化进度树与当前路径高亮。

---

## 1. 包结构（ZIP 根目录）

```text
{name}.bmad  (zip)
  bmad.json                 # 包清单/入口（必选）
  agents.json               # Agent 定义（必选，可被多个 workflow 共享）
  assets/                   # 可选：图片/静态模板/附件（共享）

  # 单 workflow（兼容模式）
  workflow.graph.json
  workflow.md
  steps/
    ...

  # 多 workflow（推荐模式）
  workflows/
    <workflow-id>/
      workflow.graph.json
      workflow.md
      steps/
        ...
```

## 2. 文件作用与区别

- `bmad.json`：包元数据与入口路径（workflow/graph/agents）；可选提供 `workflows[]` 作为多流程索引（便于 Runtime 直接列出可选流程）。
- `workflow.graph.json`：**权威图真源**（nodes/edges/条件/默认分支）。Builder 以此渲染画布；Runtime 以此校验状态跳转。
- `workflow.md`：给人/LLM看的入口文档；Frontmatter 记录运行状态（Document-as-State）。正文建议包含 steps 索引链接，便于 LLM 快速定位 step 文件。
- `steps/*.md`：每个节点一个文件（Step/Decision/Merge/End/Subworkflow 也都可以是一个 node），写清本步目标、产物、变量、以及“如何更新 workflow.md”。
- `agents.json`：Agent persona 清单（对齐 BMAD 信息结构：`metadata`/`persona`/`critical_actions`/`prompts`/`menu`/`tools`/`skills`）。

## 3. 分支（if/branch）语义

- 分支通过 `workflow.graph.json` 的多出边实现：每条边可带 `conditionText`（给 LLM 看）与可选 `conditionExpr`（给 Runtime 评估）。
- `decision` 节点（建议由 Builder/Runtime 额外校验）：
  - 至少 2 条出边；
  - 至少 1 条出边标记 `isDefault: true`（当变量不明确时的兜底）。

## 4. 状态（workflow.md frontmatter）约定

建议至少包含：
- `currentNodeId`：当前节点（恢复/展示必需）
- `stepsCompleted`：已完成节点集合（用于 UI 进度/审计）
- `variables`：分支判断变量（由 LLM 写入）
- `decisionLog`：每次从哪个 decision 走了哪条边（便于解释与回放）

CrewAgent 采用 Cursor 风格：**LLM 通过普通文件工具更新 frontmatter**。Runtime 只做硬约束：
- 文件沙箱（只能在 `jobRoot` 内读写）
- 原子落盘（tmp→rename）
- frontmatter YAML 可解析校验
- 状态跳转合法性校验（新 `currentNodeId` 必须是图允许的后继）

## 5. 校验（Schemas）与超出 Schema 的规则

导入 `.bmad` 时建议校验：
- `bmad.json`：`schemas/bmad.schema.json`
- `workflow.graph.json`：`schemas/workflow-graph.schema.json`
- `agents.json`：`schemas/agents.schema.json`
- `workflow.md` frontmatter：`schemas/workflow-frontmatter.schema.json`（需先解析 YAML）
- `step-xx.md` frontmatter：`schemas/step-frontmatter.schema.json`（需先解析 YAML）

JSON Schema 无法表达的规则（建议 Builder/Runtime 额外校验）：
- 图连通性（从 entry 可达 end）
- decision 节点的 default 分支存在性
- `workflow.md` steps index 引用的文件存在性（或由 graph 真源生成）

---

## 6. 模板（Templates）

### 6.1 `templates/workflow.md`

```md
---
schemaVersion: "1.1"
workflowType: "example"
runId: ""
project_name: ""
user_name: ""
date: ""
updatedAt: ""
currentNodeId: "" # 推荐：初始化为 graph.entryNodeId
stepsCompleted: []
variables: {}
decisionLog: []
artifacts: []
---

# Workflow

## Goal

用一句话描述该工作流要解决什么问题。

## How To Execute（LLM）

1. 读取本文件 frontmatter：`stepsCompleted/currentNodeId/variables`
2. 根据 `workflow.graph.json` 找到当前节点的后继边
3. 读取当前节点对应的 step 文件（`steps/*.md`），按其中指令完成产出
4. 用文件工具更新本文件 frontmatter（记录 stepsCompleted/currentNodeId/variables/decisionLog/artifacts）
5. 重复直到进入 `end` 节点并完成总结

## Steps Index

> Builder 应根据 `workflow.graph.json` 生成/维护本索引（便于 LLM 快速打开 step 文件）。

- [step-01-xxx](steps/step-01-xxx.md)
- [decide-02-xxx](steps/step-02-xxx.md)
- [end-99](steps/end-99.md)
```

### 6.2 `templates/step.md`

```md
---
schemaVersion: "1.1"
nodeId: "step-01"
type: "step" # step | decision | merge | end | subworkflow
title: ""
agentId: "" # from agents.json
inputs: []
outputs: []
setsVariables: []
transitions:
  - to: "step-02"
    label: "next"
    isDefault: true
    conditionText: ""
---

# Step

## Goal

清晰描述本节点要完成的目标。

## Instructions

写给 LLM 的执行指令（工具使用建议、要读哪些文件、要生成哪些产物、如何记录变量等）。

## Completion (Document-as-State)

完成本 step 后，更新 `workflow.md` frontmatter（建议用 `fs.apply_patch`）：

- 追加本 `nodeId` 到 `stepsCompleted`（避免重复）
- 将 `currentNodeId` 更新为下一节点（从 `transitions` 里选择）
- 如产生分支变量，写入 `variables`
- 如发生决策，从 `decision` 节点写入 `decisionLog`
- 如生成文件产物，写入 `artifacts`
- 更新 `updatedAt`（ISO 时间）
```

### 6.3 `templates/agents.json`

```json
{
  "schemaVersion": "1.1",
  "agents": [
    {
      "id": "agent-id",
      "metadata": {
        "name": "PersonaName",
        "title": "Agent Title",
        "icon": "🧩",
        "module": "optional-module",
        "description": "Optional short description.",
        "sourceId": "_bmad/<module>/agents/<agent>.md"
      },
      "persona": {
        "role": "Role",
        "identity": "Identity / responsibility in one paragraph.",
        "communication_style": "How the agent communicates.",
        "principles": ["Principle 1", "Principle 2"]
      },
      "critical_actions": ["Optional: must-do items before/while executing steps."],
      "prompts": [
        {
          "id": "extra-prompt-id",
          "content": "Optional reusable prompt snippet.",
          "description": "Optional description."
        }
      ],
      "menu": [
        {
          "trigger": "example-trigger",
          "description": "Optional: show entrypoints or shortcuts (BMAD-style).",
          "exec": "steps/step-01-xxx.md"
        }
      ],
      "tools": {
        "fs": { "enabled": true, "maxReadBytes": 524288, "maxWriteBytes": 1048576 },
        "mcp": { "enabled": false, "allowedServers": [] }
      },
      "skills": {
        "capabilities": {
          "filesystem": { "enabled": true },
          "python": { "enabled": true },
          "browser": { "enabled": false }
        },
        "imports": [
          { "name": "policy_checker", "source": "assets/skills/compliance_tools.py" },
          { "source": "assets/skills/finance-tools/" }
        ]
      }
    }
  ]
}
```

---

## 6.4 v1.2 扩展要点（规范增量）

### 6.4.1 Graph Node 扩展（Subworkflow）

- `workflow.graph.json` 节点新增字段：
  - `subworkflowRef`: 指向 `bmad.json.workflows[].id`
  - `passContext`: 是否把父流程 artifacts / variables 注入子流程（可选）

### 6.4.2 Run State 拆分

- 新增 `@state/run.md`（run 级）：
  - `activeWorkflowId`
  - `callStack: Array<{ workflowId, nodeId }>`
- workflow 状态拆分：
  - `runs/<runId>/state/workflows/<workflowId>/workflow.md`
- `@state/workflow.md` 保持兼容，作为“当前激活 workflow”的别名。

### 6.4.3 Agent Skills（Portable Skills）

- `agents.json` 支持 `skills`：
  - `capabilities`: filesystem/python/browser
  - `imports`: 单文件脚本或 `SKILL.md` 包
- Runtime 将脚本函数转译为 Function Calling Schema，按角色可见性暴露工具。

## 7. 示例（Examples）

### 7.1 hello-branch（单 workflow + 分支）

- `examples/hello-branch/bmad.json`
```json
{
  "schemaVersion": "1.1",
  "name": "hello-branch",
  "version": "0.1.0",
  "description": "Minimal example: micro-file workflow with a decision branch (greenfield vs brownfield).",
  "createdAt": "2025-12-23T00:00:00Z",
  "entry": {
    "workflow": "workflow.md",
    "graph": "workflow.graph.json",
    "agents": "agents.json",
    "assetsDir": "assets/"
  }
}
```

- `examples/hello-branch/workflow.graph.json`
```json
{
  "schemaVersion": "1.1",
  "entryNodeId": "step-01-intake",
  "nodes": [
    {
      "id": "step-01-intake",
      "type": "step",
      "file": "steps/step-01-intake.md",
      "title": "Intake",
      "agentId": "analyst"
    },
    {
      "id": "decide-02-project-type",
      "type": "decision",
      "file": "steps/step-02-decide-project-type.md",
      "title": "Decide project type",
      "agentId": "pm"
    },
    {
      "id": "step-03a-greenfield",
      "type": "step",
      "file": "steps/step-03a-greenfield.md",
      "title": "Greenfield plan",
      "agentId": "architect"
    },
    {
      "id": "step-03b-brownfield",
      "type": "step",
      "file": "steps/step-03b-brownfield.md",
      "title": "Brownfield plan",
      "agentId": "architect"
    },
    {
      "id": "end-99",
      "type": "end",
      "file": "steps/end-99.md",
      "title": "Finish",
      "agentId": "summarizer"
    }
  ],
  "edges": [
    { "from": "step-01-intake", "to": "decide-02-project-type", "label": "next" },
    {
      "from": "decide-02-project-type",
      "to": "step-03a-greenfield",
      "label": "greenfield",
      "conditionText": "variables.projectType == 'greenfield'",
      "conditionExpr": { "==": [{ "var": "projectType" }, "greenfield"] }
    },
    {
      "from": "decide-02-project-type",
      "to": "step-03b-brownfield",
      "label": "brownfield",
      "isDefault": true,
      "conditionText": "variables.projectType == 'brownfield'",
      "conditionExpr": { "==": [{ "var": "projectType" }, "brownfield"] }
    },
    { "from": "step-03a-greenfield", "to": "end-99", "label": "next" },
    { "from": "step-03b-brownfield", "to": "end-99", "label": "next" }
  ]
}
```

- `examples/hello-branch/workflow.md`
```md
---
schemaVersion: "1.1"
workflowType: "hello-branch"
runId: ""
project_name: ""
user_name: ""
date: ""
updatedAt: ""
currentNodeId: "step-01-intake"
stepsCompleted: []
variables: {}
decisionLog: []
artifacts: []
---

# Hello Branch Workflow

## Goal

通过一个最小示例展示：**micro-file step 文件 + decision 分支 + frontmatter 状态更新**。

## Steps Index

- [step-01-intake](steps/step-01-intake.md)
- [step-02-decide-project-type](steps/step-02-decide-project-type.md)
- [step-03a-greenfield](steps/step-03a-greenfield.md)
- [step-03b-brownfield](steps/step-03b-brownfield.md)
- [end-99](steps/end-99.md)
```

打包命令（在示例目录下执行）：

```bash
zip -r hello-branch.bmad bmad.json workflow.graph.json workflow.md agents.json steps assets
```

### 7.2 multi-workflows（多 workflow + 多 agents）

打包命令：

```bash
zip -r multi-workflows.bmad bmad.json agents.json workflows assets
```

---

## 8. JSON Schemas（Appendix）

> 这些 schemas 来自 `crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/`，用于 Runtime 导入时校验。

### 8.1 `schemas/bmad.schema.json`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://crewagent.local/schemas/bmad.schema.json",
  "title": "CrewAgent .bmad Package Manifest",
  "type": "object",
  "required": ["schemaVersion", "name", "version", "entry"],
  "properties": {
    "schemaVersion": {
      "type": "string",
      "description": "Spec version for this package format.",
      "pattern": "^1\\.1(\\.\\d+)?$"
    },
    "name": { "type": "string", "minLength": 1 },
    "version": {
      "type": "string",
      "description": "SemVer string.",
      "pattern": "^\\d+\\.\\d+\\.\\d+(-[0-9A-Za-z.-]+)?(\\+[0-9A-Za-z.-]+)?$"
    },
    "description": { "type": "string" },
    "author": { "type": "string" },
    "createdAt": { "type": "string", "format": "date-time" },
    "entry": {
      "type": "object",
      "required": ["workflow", "graph", "agents"],
      "properties": {
        "workflow": { "type": "string", "default": "workflow.md" },
        "graph": { "type": "string", "default": "workflow.graph.json" },
        "agents": { "type": "string", "default": "agents.json" },
        "assetsDir": { "type": "string", "default": "assets/" }
      },
      "additionalProperties": false
    },
    "workflows": {
      "type": "array",
      "description": "Optional workflow index for multi-workflow packages. When present, Runtime can list/select workflows without scanning the zip.",
      "items": {
        "type": "object",
        "required": ["id", "workflow", "graph"],
        "properties": {
          "id": {
            "type": "string",
            "description": "Stable workflow identifier (also used in UI).",
            "pattern": "^[A-Za-z0-9][A-Za-z0-9._:-]*$"
          },
          "displayName": { "type": "string" },
          "description": { "type": "string" },
          "workflow": { "type": "string" },
          "graph": { "type": "string" },
          "tags": {
            "type": "array",
            "items": { "type": "string" },
            "default": []
          }
        },
        "additionalProperties": false
      },
      "default": []
    },
    "runtime": {
      "type": "object",
      "properties": {
        "minRuntimeVersion": { "type": "string" },
        "requiresNetwork": { "type": "boolean", "default": false }
      },
      "additionalProperties": false
    },
    "integrity": {
      "type": "object",
      "properties": {
        "sha256": {
          "type": "string",
          "description": "Optional SHA-256 of the ZIP bytes (hex).",
          "pattern": "^[a-fA-F0-9]{64}$"
        }
      },
      "additionalProperties": false
    }
  },
  "additionalProperties": false
}
```

### 8.2 `schemas/workflow-graph.schema.json`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://crewagent.local/schemas/workflow-graph.schema.json",
  "title": "CrewAgent Workflow Graph",
  "type": "object",
  "required": ["schemaVersion", "entryNodeId", "nodes", "edges"],
  "properties": {
    "schemaVersion": {
      "type": "string",
      "pattern": "^1\\.1(\\.\\d+)?$"
    },
    "entryNodeId": { "$ref": "#/$defs/nodeId" },
    "nodes": {
      "type": "array",
      "minItems": 1,
      "items": { "$ref": "#/$defs/node" }
    },
    "edges": {
      "type": "array",
      "minItems": 0,
      "items": { "$ref": "#/$defs/edge" }
    },
    "metadata": {
      "type": "object",
      "additionalProperties": true
    }
  },
  "additionalProperties": false,
  "$defs": {
    "nodeId": {
      "type": "string",
      "description": "Stable node identifier. Recommended: step-01, decide-02, merge-03, end-99.",
      "pattern": "^[A-Za-z0-9][A-Za-z0-9._:-]*$"
    },
    "relativePath": {
      "type": "string",
      "description": "Relative path inside the .bmad zip (POSIX-style preferred).",
      "minLength": 1
    },
    "nodeType": {
      "type": "string",
      "enum": ["step", "decision", "merge", "end", "subworkflow"]
    },
    "node": {
      "type": "object",
      "required": ["id", "type", "file"],
      "properties": {
        "id": { "$ref": "#/$defs/nodeId" },
        "type": { "$ref": "#/$defs/nodeType" },
        "file": { "$ref": "#/$defs/relativePath" },
        "title": { "type": "string" },
        "description": { "type": "string" },
        "agentId": { "type": "string" },
        "inputs": {
          "type": "array",
          "items": { "type": "string" }
        },
        "outputs": {
          "type": "array",
          "items": { "type": "string" }
        }
      },
      "additionalProperties": false
    },
    "edge": {
      "type": "object",
      "required": ["from", "to", "label"],
      "properties": {
        "id": { "type": "string" },
        "from": { "$ref": "#/$defs/nodeId" },
        "to": { "$ref": "#/$defs/nodeId" },
        "label": { "type": "string", "minLength": 1 },
        "isDefault": { "type": "boolean", "default": false },
        "conditionText": {
          "type": "string",
          "description": "Human/LLM readable condition (e.g., variables.projectType == 'greenfield')."
        },
        "conditionExpr": {
          "type": "object",
          "description": "Optional machine-evaluable condition (recommended: JSONLogic object).",
          "additionalProperties": true
        }
      },
      "additionalProperties": false
    }
  }
}
```

### 8.3 `schemas/agents.schema.json`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://crewagent.local/schemas/agents.schema.json",
  "title": "CrewAgent Agents Manifest",
  "type": "object",
  "required": ["schemaVersion", "agents"],
  "properties": {
    "schemaVersion": {
      "type": "string",
      "pattern": "^1\\.1(\\.\\d+)?$"
    },
    "agents": {
      "type": "array",
      "minItems": 1,
      "items": { "$ref": "#/$defs/agent" }
    }
  },
  "additionalProperties": false,
  "$defs": {
    "agentId": {
      "type": "string",
      "pattern": "^[A-Za-z0-9][A-Za-z0-9._:-]*$"
    },
    "nonEmptyString": {
      "type": "string",
      "minLength": 1
    },
    "toolPolicy": {
      "type": "object",
      "properties": {
        "fs": {
          "type": "object",
          "properties": {
            "enabled": { "type": "boolean", "default": true },
            "maxReadBytes": { "type": "integer", "minimum": 1 },
            "maxWriteBytes": { "type": "integer", "minimum": 1 }
          },
          "additionalProperties": false
        },
        "mcp": {
          "type": "object",
          "properties": {
            "enabled": { "type": "boolean", "default": false },
            "allowedServers": {
              "type": "array",
              "items": { "type": "string" }
            }
          },
          "additionalProperties": false
        }
      },
      "additionalProperties": false
    },
    "agentMetadata": {
      "type": "object",
      "required": ["name", "title", "icon"],
      "properties": {
        "name": { "$ref": "#/$defs/nonEmptyString", "description": "Character/persona name (e.g., 'Amelia')." },
        "title": { "$ref": "#/$defs/nonEmptyString", "description": "Agent title (e.g., 'Developer Agent')." },
        "icon": { "$ref": "#/$defs/nonEmptyString", "description": "Emoji or short icon string." },
        "module": { "type": "string", "description": "Optional grouping/module slug (e.g., 'bmm')." },
        "description": { "type": "string" },
        "sourceId": { "type": "string", "description": "Optional original BMAD id/path (e.g., '_bmad/bmm/agents/dev.md')." }
      },
      "additionalProperties": false
    },
    "persona": {
      "type": "object",
      "required": ["role", "identity", "communication_style", "principles"],
      "properties": {
        "role": { "$ref": "#/$defs/nonEmptyString" },
        "identity": { "$ref": "#/$defs/nonEmptyString" },
        "communication_style": { "$ref": "#/$defs/nonEmptyString" },
        "principles": {
          "description": "Either a multi-line string or a list of principle strings.",
          "oneOf": [
            { "$ref": "#/$defs/nonEmptyString" },
            { "type": "array", "minItems": 1, "items": { "$ref": "#/$defs/nonEmptyString" } }
          ]
        }
      },
      "additionalProperties": false
    },
    "prompt": {
      "type": "object",
      "required": ["id", "content"],
      "properties": {
        "id": { "$ref": "#/$defs/nonEmptyString" },
        "content": { "$ref": "#/$defs/nonEmptyString" },
        "description": { "type": "string" }
      },
      "additionalProperties": false
    },
    "menuItem": {
      "type": "object",
      "required": ["description"],
      "properties": {
        "trigger": { "type": "string", "description": "Single trigger (kebab-case recommended)." },
        "triggers": { "type": "array", "items": { "type": "object", "additionalProperties": true } },
        "description": { "$ref": "#/$defs/nonEmptyString" },
        "workflow": { "type": "string" },
        "exec": { "type": "string" },
        "action": { "type": "string" },
        "data": { "type": "string" },
        "validate-workflow": { "type": "string" },
        "ide-only": { "type": "boolean" },
        "web-only": { "type": "boolean" }
      },
      "additionalProperties": true
    },
    "agent": {
      "type": "object",
      "required": ["id", "metadata", "persona"],
      "properties": {
        "id": { "$ref": "#/$defs/agentId" },
        "metadata": { "$ref": "#/$defs/agentMetadata" },
        "persona": { "$ref": "#/$defs/persona" },
        "critical_actions": {
          "type": "array",
          "items": { "$ref": "#/$defs/nonEmptyString" }
        },
        "prompts": {
          "type": "array",
          "items": { "$ref": "#/$defs/prompt" }
        },
        "menu": {
          "type": "array",
          "items": { "$ref": "#/$defs/menuItem" }
        },
        "systemPrompt": {
          "type": "string",
          "description": "Optional compiled system prompt. If omitted, Runtime may render from metadata/persona/critical_actions."
        },
        "userPromptTemplate": { "type": "string" },
        "discussion": { "type": "boolean" },
        "webskip": { "type": "boolean" },
        "conversational_knowledge": {
          "type": "array",
          "items": { "type": "object", "additionalProperties": true }
        },
        "tools": { "$ref": "#/$defs/toolPolicy" }
      },
      "additionalProperties": false
    }
  }
}
```

### 8.4 `schemas/workflow-frontmatter.schema.json`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://crewagent.local/schemas/workflow-frontmatter.schema.json",
  "title": "CrewAgent workflow.md Frontmatter",
  "type": "object",
  "required": ["schemaVersion", "workflowType", "stepsCompleted", "currentNodeId"],
  "properties": {
    "schemaVersion": {
      "type": "string",
      "pattern": "^1\\.1(\\.\\d+)?$"
    },
    "workflowType": { "type": "string", "minLength": 1 },
    "runId": { "type": "string" },
    "project_name": { "type": "string" },
    "user_name": { "type": "string" },
    "date": { "type": "string" },
    "updatedAt": { "type": "string", "format": "date-time" },
    "currentNodeId": {
      "type": "string",
      "description": "Current node id. Empty string allowed before first step starts."
    },
    "stepsCompleted": {
      "type": "array",
      "items": { "type": "string" },
      "default": []
    },
    "variables": {
      "type": "object",
      "description": "Decision variables used by branching conditions.",
      "additionalProperties": true,
      "default": {}
    },
    "decisionLog": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["from", "to", "label"],
        "properties": {
          "from": { "type": "string" },
          "to": { "type": "string" },
          "label": { "type": "string" },
          "reason": { "type": "string" },
          "decidedAt": { "type": "string", "format": "date-time" }
        },
        "additionalProperties": false
      },
      "default": []
    },
    "artifacts": {
      "type": "array",
      "items": { "type": "string" },
      "default": []
    }
  },
  "additionalProperties": true
}
```

### 8.5 `schemas/step-frontmatter.schema.json`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://crewagent.local/schemas/step-frontmatter.schema.json",
  "title": "CrewAgent step-xx.md Frontmatter",
  "type": "object",
  "required": ["schemaVersion", "nodeId", "type"],
  "properties": {
    "schemaVersion": {
      "type": "string",
      "pattern": "^1\\.1(\\.\\d+)?$"
    },
    "nodeId": {
      "type": "string",
      "pattern": "^[A-Za-z0-9][A-Za-z0-9._:-]*$"
    },
    "type": {
      "type": "string",
      "enum": ["step", "decision", "merge", "end", "subworkflow"]
    },
    "title": { "type": "string" },
    "agentId": { "type": "string" },
    "inputs": {
      "type": "array",
      "items": { "type": "string" },
      "default": []
    },
    "outputs": {
      "type": "array",
      "items": { "type": "string" },
      "default": []
    },
    "setsVariables": {
      "type": "array",
      "items": { "type": "string" },
      "default": []
    },
    "transitions": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["to", "label"],
        "properties": {
          "to": { "type": "string" },
          "label": { "type": "string" },
          "isDefault": { "type": "boolean", "default": false },
          "conditionText": { "type": "string" }
        },
        "additionalProperties": false
      },
      "default": []
    }
  },
  "additionalProperties": true
}
```
