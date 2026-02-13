# Story BAI-4.3: Implement Change Preview Pane

Status: done

> Epic: `_bmad-output/epics-builder-ai.md`  
> Tech Spec: `_bmad-output/tech-spec/builder-ai-implementation-spec.md`  
> Prototype (mandatory reference): `_bmad-output/pencil-new-work.pen` (`Preview Column` `EFFUb` + `Markdown Diff Panel` `47CBT`)

## Story

As a **Builder User**,  
I want to preview structured updates before applying,  
So that I understand affected objects.

## Acceptance Criteria

### AC-1: 变更文件清单可见且结构化
- **Given** session 已产生 `latestSuggestion`
- **When** 预览区渲染
- **Then** 展示“Changed files”列表（新增/修改标识）和验证摘要（如 `4 files · 1 added · 3 modified`）

### AC-2: 影响范围覆盖跨对象修改
- **Given** suggestion 含 `workflow/step/agent/asset` 跨对象更新
- **When** 渲染 impact
- **Then** 用户可看到每个受影响对象与风险提示，不遗漏任何对象类型

### AC-3: Markdown Diff 可读
- **Given** 选中一个变更项
- **When** 展示差异内容
- **Then** 提供 `Before/After` 对照与增删统计，帮助用户在 Apply 前审阅

### AC-3b: Diff 基线固定
- **Given** 会话存在临时草稿 overlay
- **When** 渲染 diff
- **Then** `Before` 固定为原始 base 数据
- **And** `After` 固定为当前 session 最新 overlay 结果

### AC-4: 与最新建议版本一致
- **Given** 用户继续对话并生成新建议
- **When** 预览刷新
- **Then** 仅显示最新建议版本，不与旧版本混淆

## Design

### Summary
- Tech Spec: `_bmad-output/tech-spec/builder-ai-implementation-spec.md`
- Prototype: `_bmad-output/pencil-new-work.pen`（`EFFUb/URWYM/LQvut + 47CBT/9ldVM`）
- 基于原型 `EFFUb + 47CBT` 实现右侧变更清单与下方 markdown diff 区域。
- 对齐 FR19/FR22：用户在 Apply 前可见影响对象、变更文件与关键差异。
- 对齐 FR14：展示 notes/风险说明，避免“黑盒应用”。

### UX / UI（来自原型）
- 右侧预览卡片：
  - 标题 `Changed files`
  - 验证摘要 badge（原型 `validationBadge`）
  - 文件列表（示例：`M steps/...`, `A assets/...`, `M agents/...`, `M workflows/...`）
  - hint 文案：仅展示变更文件清单，详细差异在下方
- 下方 diff 面板：
  - 标题含目标文件路径（原型 `Markdown diff · ...`）
  - `+/-` 统计 badge（原型 `+14/-6`）
  - `Before/After` 双栏 markdown 内容

### API / Contracts
- 会话消息响应需包含：
  - `latestSuggestion.changeSet`
  - `latestSuggestion.impact`
  - `validation`（passed/errors/hints）
- 前端不直接解析自由文本作为预览来源，必须以结构化字段为准。
- 与 `builder.change.validate` 对齐：
  - 失败时仍渲染预览，但 Apply 按钮禁用并显示失败原因。

### Data / Storage
- 预览前端状态：
  - `changedFiles[]`, `impact[]`, `selectedFile`, `diffModel`, `validationStatus`, `baseSnapshot`, `overlaySnapshot`.
- 仅存储当前会话的最新预览快照；历史版本可后续扩展，不在本故事内实现。

### Errors / Edge Cases
- suggestion 缺失 `changeSet`：显示“预览不可用”并引导继续对话修复。
- diff 体积过大：默认折叠/截断，避免页面卡顿（NFR2）。
- 文件路径不合法或越界：以 validation 错误卡展示，不提供 Apply。
- 跨对象引用错误（例如 step 引用不存在 agent）：明确显示对象级错误。

### Test Plan
- 生成包含 step+asset+agent+workflow 的建议，预览区完整显示四类对象影响。
- 切换不同文件项，diff 面板正确更新对应 `Before/After`。
- validation 失败时预览可见且 Apply 禁用，错误提示可读。
- 新消息返回新建议后，旧版本不残留，列表和 diff 均更新。
- 多轮 stage 后 diff 始终比较 `base vs latest overlay`，不会串到过期中间版本。

## Technical Scope

- 新增变更预览组件：
  - `crewagent-builder-frontend/src/components/ai/change-preview-pane.tsx`
- 预览组件接入 Workbench：
  - `crewagent-builder-frontend/src/components/ai/workbench-shell.tsx`
- 扩展会话响应映射（preview model 适配）：
  - `crewagent-builder-frontend/src/lib/ai-workbench-client.ts`

## Tasks / Subtasks

- [x] Task 1: 按原型实现 Changed files 列表与 validation badge（AC: 1）
- [x] Task 2: 实现 impact 渲染（覆盖 workflow/step/agent/asset）（AC: 2）
- [x] Task 3: 实现 markdown diff 双栏与 +/- 统计（AC: 3）
- [x] Task 4: 实现“最新建议覆盖旧建议”刷新策略（AC: 4）
- [x] Task 5: 补充前端单测（预览渲染、切换、错误态、base/overlay diff）（AC: 1,2,3,3b,4）

## Dev Notes

- Guardrails：
  - 预览区只读，不允许在此直接编辑并绕过 validate/apply 流程。
  - 变更清单排序稳定（建议按对象类型 + 路径）以减少视觉抖动。
  - 影响范围必须完整展示，不允许仅显示目标对象而忽略联动对象。
  - diff 必须以 base 为固定左侧，不随会话中间状态漂移。

### Project Structure Notes
- 预览渲染逻辑应集中在 `change-preview-pane.tsx`，避免散落到 route 页面。
- 差异渲染可复用现有 markdown 组件，但需保证 before/after 并排语义。

## References

- `_bmad-output/epics-builder-ai.md`（Story BAI-4.3）
- `_bmad-output/prd-builder-AI.md`（FR19/FR22）
- `_bmad-output/tech-spec/builder-ai-implementation-spec.md`（10.2 UX Behavior）
- `_bmad-output/architecture/builder-ai-llm-interaction-architecture.md`（8.3 changeSet 示例）
- `_bmad-output/pencil-new-work.pen`（`EFFUb/URWYM/LQvut/47CBT/9ldVM`）

## File List

- `_bmad-output/implementation-artifacts/bai-4-3-implement-change-preview-pane.md`
- `_bmad-output/implementation-artifacts/sprint-status-builder-ai.yaml`
- `crewagent-builder-frontend/src/components/ai/change-preview-pane.tsx`
- `crewagent-builder-frontend/src/components/ai/workbench-shell.tsx`
- `crewagent-builder-frontend/src/lib/ai-workbench-model.ts`
- `crewagent-builder-frontend/tests/ai-workbench-model.test.ts`

## Dev Agent Record

### Agent Model Used

GPT-5 (Codex CLI)

### Debug Log References

- `cd crewagent-builder-frontend && npm run test`
- `cd crewagent-builder-frontend && npm run lint`

### Completion Notes List

- 按原型实现 `Changed files` 列表 + validation badge，并支持条目切换。
- 增加 impact 渲染（objects / counts / riskFlags），覆盖 step/asset 等跨对象提示。
- 实现 markdown diff 面板（Before/After + `+/-` 统计）。
- Workbench 仅保留“当前最新建议”预览快照，新的 suggestion 会覆盖旧数据。
- 增加 `ai-workbench-model` 单测，验证预览模型构建与校验文案逻辑。

### File List

- `_bmad-output/implementation-artifacts/bai-4-3-implement-change-preview-pane.md`
- `_bmad-output/implementation-artifacts/sprint-status-builder-ai.yaml`
- `crewagent-builder-frontend/src/components/ai/change-preview-pane.tsx`
- `crewagent-builder-frontend/src/components/ai/workbench-shell.tsx`
- `crewagent-builder-frontend/src/lib/ai-workbench-model.ts`
- `crewagent-builder-frontend/tests/ai-workbench-model.test.ts`

## Change Log

- Frontend：预览列升级为“文件清单 + impact + markdown diff”三段式结构。
- Frontend：新增预览纯函数模型，统一 changed files、diff 与 impact 的构造逻辑。
- Stabilization（2026-02-11）：修复 Workbench diff 与 Editor 打开 Step 内容不一致的问题：
  - Editor 读取 Step 时优先使用真实 step 文件（不再只基于节点结构重组 markdown）。
  - autosave/save 链路保留 `stepFilePath` 与 graph `node.file`，避免回写到错误路径。
  - node rename 时同步默认 step 文件路径，防止路径孤儿与错位覆盖。
