# Story 12.2: Chat-Only Personal KB Retrieval & Injection

Status: done

## Story

As a **Consumer**,  
I want personal memory to augment only chat conversations,  
So that agent/run workflows are not impacted by personal preferences.

## Acceptance Criteria

### AC-1: Chat 模式个人知识检索与注入

**Given** current mode is `chat`  
**When** context is built for an answer  
**Then** system always injects fixed core memory (`SOUL.md`, `USER.md`)  
**And** system additionally injects `MEMORY.md#Pinned`（如存在）  
**And** long-tail memory injection uses query-based top-k retrieval from `MEMORY.md#General`（兼容 legacy non-pinned 条目）and recent non-empty daily memory (`memory/YYYY-MM-DD.md`)  
**And** retrieval must only inject entries with effective `status=active` (exclude `deleted/superseded`)  
**And** when same topic has multiple active generations, only newest effective entry is injected  
**And** personal injection remains transparent to end users (no chat UI badge/toast)

### AC-2: Agent/Run 模式强隔离

**Given** current mode is `agent` or `run`  
**When** context is built  
**Then** personal KB retrieval/injection is skipped  
**And** no user-facing skip indicator is shown in agent/run UI

### AC-3: 可观测日志

**Given** personal knowledge is used or skipped  
**When** runtime logs events  
**Then** logs can distinguish `chat hit injection` vs `chat miss` vs `agent/run skipped`
**And** logs are the primary visibility surface for this story

### AC-4: 关键记忆沉淀规则（双层注入前提）

**Given** a memory is marked as long-term and must always take effect  
**When** memory is persisted from Story 12.1 write path  
**Then** it should be routed into `SOUL.md` / `USER.md` / `MEMORY.md#Pinned`  
**And** daily memory should be treated as long-tail retrieval source, not fixed-injection source

## Technical Notes

- Delivery pattern: vertical full-stack in one story (Main/IPC/Renderer/Test together).
- Personal KB injection must follow dual-layer model in chat mode: fixed core first, retrieval tail second.
- Keep strict mode guard in one place to avoid drift (`chat` allowlist + `agent/run` deny by default).
- Injection resolver must honor Story 12.1 mutation metadata (`memoryId/status/replaces*`) to avoid stale memory revival.
- Long-term memory routing should align with Story 12.1 write path (`SOUL/USER/Pinned`); long-tail reusable memory should land in `MEMORY.md#General`; daily memory stays retrieval-only.
- Story 12.2 依赖 Story 12.1 提供明确的 `Pinned vs General vs daily` 写入契约；读取侧不负责猜测某条 `MEMORY.md` 内容属于 fixed 还是 retrieval。
- Design: `_bmad-output/implementation-artifacts/design-12-2-chat-only-personal-kb-retrieval-and-injection.md`
- Code Review: `_bmad-output/implementation-artifacts/code-review-12-2-chat-only-personal-kb-retrieval-and-injection.md`
- Cross-Story Review: `_bmad-output/implementation-artifacts/code-review-12-1-12-2-personal-kb-layering-contract.md`

## Tasks / Subtasks

- [x] Task 1: Runtime Main 集成（后端）实现（AC: 1,2,3,4）
  - [x] 1.1 在 chat 上下文构建链路接入“双层注入”管线（fixed core + retrieval tail）
  - [x] 1.2 增加模式守卫：`agent/run` 跳过 personal 注入
  - [x] 1.3 实现注入上限策略（fixed 层与 retrieval 层分别预算 + top-k）
  - [x] 1.4 输出注入命中/跳过事件到日志（区分 fixed/retrieval 命中）

- [x] Task 2: Injection 诊断契约（内部）实现（AC: 1,2,3,4）
  - [x] 2.1 统一日志事件字段（`mode/decision/reason/fixedCount/retrievalCount/usedChars/budgetChars`）
  - [x] 2.2 如保留 `usage.personalKb`，仅作内部调试/测试用途，不作为前端展示依据

- [x] Task 3: Renderer UI（前端）约束（AC: 1,2）
  - [x] 3.1 chat UI 不新增“personal memory injected”状态提示
  - [x] 3.2 agent/run UI 不新增“skipped by mode”提示

- [x] Task 4: Integration & E2E 验证（AC: 1,2,3,4）
  - [x] 4.1 chat 模式固定层始终注入（SOUL/USER + optional pinned）并产生日志
  - [x] 4.2 chat 模式长尾检索层按 query 命中（hit/miss 可区分）
  - [x] 4.3 agent/run 模式不触发 personal 注入
  - [x] 4.4 日志事件分类校验（hit vs miss vs skipped）

- [x] Task 5: 记忆沉淀规则对齐（AC: 4）
  - [x] 5.1 明确“长期稳定记忆”写入目标（SOUL/USER/Pinned）与判定准则
  - [x] 5.2 明确 daily 仅作为检索层来源，不参与 fixed 层直注入

- [x] Task 6: 回归与文档
  - [x] 6.1 不影响现有 chat 历史压缩与上下文构建
  - [x] 6.2 更新 story file list 与变更说明

- [x] Task 7: Code Review Follow-ups（Blocking）（AC: 1,4）
  - [x] 7.1 固定层改为基于 active entry 解析 `SOUL.md` / `USER.md`，禁止在结构化记忆存在时直接 raw file 注入
  - [x] 7.2 明确并落地 `MEMORY.md#Pinned` 读取契约与 section-aware 写入能力，保证“长期记忆沉淀”与固定层读取一致
  - [x] 7.3 修复中文固定层 section 标题匹配，不再依赖 JS `\\b`
  - [x] 7.4 补充回归测试：`SOUL/USER` 更新后旧记忆不再注入；中文 section 可被固定层命中

- [x] Task 8: Cross-Story Review Follow-ups（Blocking）（AC: 1,4）
  - [x] 8.1 12.1 已提供明确的 `MEMORY.md#Pinned` / `MEMORY.md#General` 写入分类契约
  - [x] 8.2 注入层改为消费 `Pinned`（fixed）与 `General`（retrieval）的一致结构，不能把所有 `MEMORY.md` 写入都视为 fixed
  - [x] 8.3 recent daily source 选择需排除自动初始化但无 active entry 的空白 daily 文件
  - [x] 8.4 补回归测试：空白今日 daily 不占用 recent retrieval window

## Dev Notes (2026-03-10)

- `main.ts` 在 `callAgentChat` 接入 personal KB 双层注入决策：
  - `chat` 模式生成并注入额外 system 段落；
  - `agent` 模式走 `mode_guard` 跳过；
  - 统一记录日志事件：`kb.personal.injection.hit|miss|skipped`。
- `runs:continue` 增加 run 模式 skip 日志，保证 `agent/run` 都有可观测的 mode guard 证据。
- `usage.personalKb` 统一挂载到返回 usage，仅作为内部诊断字段，不用于前端提示。
- `personalKbService` 增加双层注入单测覆盖（fixed/retrieval、budget truncation、mode guard、kb unavailable）。
- `runtimeStore` 增加检索面 API 单测覆盖（recent daily list、active/search、sectionTitle 与 deleted 过滤）。

## Dev Notes (2026-03-11)

- `personalKbService` 固定层改为优先消费 `getPersonalKbActiveEntries(SOUL.md/USER.md)`，仅在无结构化记忆块时才回退到 legacy plain-text 注入。
- Story 12.1 已补齐 `targetSection` 写入契约：`MEMORY.md` 现按 `Pinned|General` 落盘，不再默认落到 `## Pinned`。
- `resolvePersonalKbMutationTarget` 现在返回 `sectionTitle`，让写入链路和固定层读取共享同一套 section 语义。
- 中文固定层 section 匹配改为 token/substring 规则，覆盖 `关键记忆`、`长期偏好` 等标题。
- 新增回归测试覆盖：结构化 `SOUL/USER` 不再 raw dump、中文 pinned section 命中、`MEMORY.md#Pinned` / `MEMORY.md#General` 写入契约落地。
- `runtimeStore.listRecentPersonalKbDailyTargets()` 现已改为 recent non-empty daily 规则：只返回有 active entry 的 daily 文件，自动初始化的空白今日文件不再占用 retrieval window。
- 新增回归测试覆盖：
  - recent daily 仅按“有 active entry 的 daily 文件”取最近 N 个；
  - 空白今日 daily 不会挤占 recent retrieval window。

## Archived Review Follow-ups (2026-03-11)

- Code review 结论：`1 HIGH / 2 MEDIUM`，当前不满足直接批准条件。
- Must Fix 范围：
  - `SOUL.md` / `USER.md` 固定层不能继续直接读整文件文本，必须走 active-state 过滤；
  - `MEMORY.md#Pinned` 需要真实读取契约与 section-aware 写入基础，不能只有读取约定没有写入落地；
  - 中文固定层 section 标题匹配必须可用。
- 以上问题已在本次 `dev-story 12-2` 中先行修复，但不代表 12.2 当前已可通过；当前 story 仍受下方 cross-story review 阻塞。

## Current Cross-Story Review Follow-ups (2026-03-11)

- 当前结论：12.2 的 cross-story blocker 已关闭，story 可标记为 `done`。
- 已完成内容：
  - 12.1 已提供 `Pinned vs General vs daily` 写入契约；
  - 注入层当前稳定消费 `SOUL.md` / `USER.md` / `MEMORY.md#Pinned` / `MEMORY.md#General` 结构；
  - recent daily 已改为 “最近有 active entry 的 daily 文件” 窗口，空白今日文件不会挤占检索来源。
- 当前统一依赖：
  - `SOUL.md` / `USER.md` / `MEMORY.md#Pinned`：fixed layer；
  - `MEMORY.md#General` + recent non-empty daily：retrieval layer。
- 当前状态：
  - Story 12.2 已满足 AC-1~AC-4，并已完成 review，可标记为 `done`。

## File List

- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/services/personalKbService.ts`
- `crewagent-runtime/electron/services/personalKbService.test.ts`
- `crewagent-runtime/electron/services/chatContextBuilder.ts`
- `crewagent-runtime/electron/services/chatContextBuilder.test.ts`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/stores/runtimeStore.test.ts`
- `_bmad-output/implementation-artifacts/12-2-chat-only-personal-kb-retrieval-and-injection.md`
- `_bmad-output/implementation-artifacts/design-12-2-chat-only-personal-kb-retrieval-and-injection.md`
- `_bmad-output/implementation-artifacts/code-review-12-2-chat-only-personal-kb-retrieval-and-injection.md`
- `_bmad-output/prd-runtime-personal-knowledge-base.md`
- `_bmad-output/epics.md`

## Verification

- `npm --prefix crewagent-runtime test -- electron/services/personalKbService.test.ts electron/services/chatContextBuilder.test.ts electron/stores/runtimeStore.test.ts`
  - 3 个测试文件通过，`84 passed`（2026-03-11）。
- `cd crewagent-runtime && npx tsc --noEmit`
  - TypeScript 编译检查通过（0 errors）。

## Change Log

- 2026-03-10: 完成双层注入主链路开发，状态更新为 `ready-for-review`。
- 2026-03-11: 完成 code review，发现 `H1/M1/M2` 阻断项；已新增 review 报告并将状态回退为 `in-progress`。
- 2026-03-11: 完成 Must Fix 修复与回归测试，状态恢复为 `ready-for-review`。
- 2026-03-11: Cross-story review reopened 12.2; blocked by 12.1 write classification contract and recent non-empty daily retrieval rule, status returned to `in-progress`.
- 2026-03-11: 12.1 写入分类契约已落地；12.2 当前仅剩 recent non-empty daily 检索窗口 blocker。
- 2026-03-11: recent non-empty daily 检索窗口已落地，回归测试补齐，12.2 cross-story blocker 关闭，状态恢复为 `ready-for-review`。
- 2026-03-11: Story 12-2 review approved，BMAD 状态正式收尾为 `done`。
