# Story 3.15: Generate v1.1 `agents.json` (Schema-Ready)

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **Creator**,  
I want the Builder to generate `agents.json` using the v1.1 agent schema (`metadata/persona/prompts/menu/tools`),  
so that Runtime can load personas/prompts/tool policy consistently and schema validation passes.

## Acceptance Criteria

1. **Given** I have agents in ProjectBuilder  
   **When** the Builder generates `agents.json` for export  
   **Then** it produces a v1.1 manifest object (not a legacy array) that validates against `agents.schema.json`  
   **And** it has `schemaVersion: "1.1"` and a non-empty `agents[]` list with required fields (`id/metadata/persona`, no extra fields)

2. **Given** no tool policy is configured  
   **When** the Builder generates `agents.json`  
   **Then** it includes a default tool policy per agent (`tools.fs.enabled: true`, `tools.mcp.enabled: false`, `tools.mcp.allowedServers: []`)

3. **Given** the project has no agents configured  
   **When** the Builder tries to generate `agents.json`  
   **Then** it shows an actionable error (“请先创建至少 1 个 Agent”) and blocks export/preview generation

## Design

### Summary

- Tech Spec: `_bmad-output/tech-spec.md`
- `agents.json` 为导出派生文件（ZIP root），供 Runtime 读取 persona/prompts/menu/tools（multi-workflow 包共享）
- 生成器以“纯函数”实现（不落库）：从 project 的 `agentsJson` 归一化出 v1.1 manifest，并输出 pretty JSON
- 生成结果必须通过 schema：`crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/agents.schema.json`（`additionalProperties:false`，`agents.minItems:1`）

### UX / UI

- ProjectBuilder（`/builder/[projectId]`）增加只读的 `agents.json (preview)`（可折叠）：
  - Pretty JSON + Copy（样式/交互与 `bmad.json (preview)` 一致）
  - Schema errors（阻断生成与后续导出）与 warnings（不阻断）
  - 无 agents 时显示错误：“请先创建至少 1 个 Agent”
- 说明：整包导出（ZIP 下载）在 Story 3.16；本 story 只负责生成/预览 `agents.json`。

### API / Contracts

- 不新增后端接口；复用现有数据源：
  - `GET /packages/{projectId}`（读取 `agentsJson` 存量）
  - `PUT /packages/{projectId}/agents`（Story 3.10 已实现 v1.1 agent 编辑与保存）
- Contract：
  - `agents.json` → `crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/agents.schema.json`
  - 输出路径：zip root `agents.json`（由 `bmad.json.entry.agents` 引用；multi-workflow 示例同样为 root）
- 校验实现建议（前端）：使用 `Ajv2020` 编译 `agents.schema.json`，将 `validate.errors` 格式化为用户可读文本（instancePath + message）

### Data / Storage

- 输入来源：project 的 `agentsJson`（DB 字段，可能为以下形态）：
  1) v1.1 manifest：`{ schemaVersion, agents: [...] }`
  2) legacy array：`[{ name, role, ... }]`
  3) 空/旧默认值：`""` / `"[]"`
- 输出：v1.1 `agents.json`（string，pretty JSON），并确保满足 schema（不输出 schema 未定义字段）
- 生成器（建议新增前端 lib）：
  - `buildAgentsManifestV11({ agentsJsonRaw }): { manifest, warnings, errors }`
  - `formatAgentsManifestV11(manifest): string`（`JSON.stringify(..., null, 2)`）
- 归一化/默认值策略（MVP，目标是“可导出且不破坏引用”）：
  - 若输入为 v1.1 manifest：
    - `schemaVersion` 缺失/不合法：warning + 回退为 `"1.1"`
    - agent 必填字段缺失：尽可能补默认（`metadata.icon="🧩"`, `persona.communication_style="direct"`, `persona.principles=["TBD"]`）；无法补齐则 error
    - `tools` 缺失：补齐默认 `{ fs:{enabled:true}, mcp:{enabled:false, allowedServers:[]} }`
    - `maxReadBytes/maxWriteBytes`：默认不写入（由 Runtime 全局默认合并；后续再做可配置）
  - 若输入为 legacy array：
    - 按确定性规则生成 `agentId`（基于 name，冲突则 `-2/-3...`），并补齐 v1.1 必填结构（metadata/persona/tools）
  - 若无 agents：error（阻断 preview/export）

### Errors / Edge Cases

- JSON 解析失败：错误（阻断；提示“agentsJson 格式不合法，请修复后重试”）
- `agents` 为空：错误（schema `minItems: 1`，阻断；提示创建 Agent）
- 重复 agentId：错误（阻断；提示冲突 id；MVP 不自动改 id，避免破坏 workflow node `agentId` 引用）
- agentId 不合法（不匹配 `^[A-Za-z0-9][A-Za-z0-9._:-]*$`）：错误（阻断；提示需要手工修复/重建 Agent）
- schema 校验失败：错误（阻断；展示具体字段路径与原因）
- tools 缺失：不阻断，自动补齐默认值（见 Data/Storage）

### Test Plan

- Unit（纯函数）：覆盖输入三种形态（manifest/legacy/empty）、默认值补齐、重复/非法 id 错误、schema 校验错误输出（Ajv）
- Integration（手动）：ProjectBuilder 打开 `agents.json (preview)`：
  - 有 agents 时：schema error 为空；Copy 可用
  - 无 agents 时：显示可操作错误（引导创建 agent）

## Tasks / Subtasks

- [x] 1) 实现 agents.json 生成器（纯函数；从 project `agentsJson` 序列化/归一化为 v1.1）
- [x] 2) Schema 校验：使用 `agents.schema.json` 校验输出并给出可读错误
- [x] 3) UI：在 ProjectBuilder 增加 `agents.json (preview)`（含 Copy/warnings/errors）

## References

- `_bmad-output/epics.md`（Epic 3 / Story 3.15）
- `_bmad-output/tech-spec.md`（Package Spec v1.1）
- `crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/agents.schema.json`
- `crewagent-runtime/spec/bmad-package-spec/v1.1/templates/agents.json`
- `crewagent-builder-backend/app/schemas/workflow_package.py`（AgentsManifestV11 校验）

## Dev Agent Record

### Agent Model Used

GPT-5.2 (Codex CLI)

### Debug Log References

- Frontend:
  - `./node_modules/.bin/tsc -p tsconfig.tests.json`
  - `node --test .tmp-tests/tests/*.test.js`
  - `npm -C crewagent-builder-frontend run lint`
  - `npm -C crewagent-builder-frontend run build`

### Completion Notes List

- 新增 v1.1 `agents.json` 生成/校验（纯函数 + AJV），并在 ProjectBuilder 提供 `agents.json (preview)`（Copy + errors/warnings）。
- Code review 修复：禁止 UI 层自动改写非法/重复 `agentId`；AJV 校验延迟编译；补充单测与 `npm test`；增加 schema 同步脚本与预览 UX 修正。

### File List

- `_bmad-output/implementation-artifacts/3-15-generate-v1-1-agents-json-schema-ready.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `crewagent-builder-frontend/.gitignore`
- `crewagent-builder-frontend/eslint.config.mjs`
- `crewagent-builder-frontend/package.json`
- `crewagent-builder-frontend/scripts/sync-bmad-spec.mjs`
- `crewagent-builder-frontend/src/app/builder/[projectId]/page.tsx`
- `crewagent-builder-frontend/src/lib/agents-manifest-v11.ts`
- `crewagent-builder-frontend/src/lib/bmad-spec/v1.1/agents.schema.json`
- `crewagent-builder-frontend/tests/agents-manifest-v11.test.ts`
- `crewagent-builder-frontend/tsconfig.tests.json`

### Change Log

- Builder：增加 v1.1 `agents.json` 生成器 + AJV schema 校验，并在 ProjectBuilder 展示可复制预览与错误提示。
- Review fixes：修复 `agentId` 稳定性风险（不自动改写 id）；优化校验加载；补全测试与脚本；修正预览区在错误场景下的提示。
