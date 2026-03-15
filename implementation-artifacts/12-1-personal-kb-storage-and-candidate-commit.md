# Story 12.1: Personal KB Storage & Candidate Commit

Status: done

## Story

As a **Consumer**,  
I want personal memory to be stored in a structured local directory and written only after explicit confirmation,  
So that long-term preferences are reusable without silent memory pollution.

## Acceptance Criteria

### AC-1: Personal KB 初始化结构

**Given** Runtime starts and personal knowledge has not been initialized  
**When** I start a conversation in any mode (`chat/agent/run`)  
**Then** the system creates personal KB structure under Runtime private state:
- `USER.md`
- `SOUL.md`
- `MEMORY.md`
- `memory/YYYY-MM-DD.md`
- `index.json`
- `manifest.json`

### AC-2: 候选记忆必须显式确认

**Given** I express explicit memory intent (e.g., "记住这个")  
**When** the system receives this intent in `chat/agent/run` mode and runs LLM routing  
**Then** it must request my confirmation before writing  
**And** routing output must be schema-validated and target-file allowlisted before candidate creation  
**And** low-confidence/invalid routing must fall back safely (clarification or rule fallback)  
**And** unconfirmed candidates are not persisted  
**And** confirmation UI must not expose internal file names (e.g. `MEMORY.md` / `memory/*.md`) to end users

### AC-3: 提交写入与元数据

**Given** a candidate is approved  
**When** it is committed  
**Then** `ADD/UPDATE/DELETE` must be executed with system-side mutation semantics (not plain append)  
**And** entries include `source` and `updatedAt` metadata  
**And** entries include stable identity/status metadata (`memoryId`, `status`) for later injection filtering  
**And** write routing must distinguish `MEMORY.md#Pinned`（固定层）vs `MEMORY.md#General`（检索层）vs `memory/YYYY-MM-DD.md`（daily）  
**And** the system must not default all `MEMORY.md` writes to `Pinned`  
**And** index is incrementally updated

### AC-4: 索引重建能力

**Given** index is corrupted or missing  
**When** rebuild is triggered  
**Then** index can be rebuilt from Markdown truth sources

### AC-5: 极简治理操作（清空/重整）

**Given** I am a non-technical user in Settings  
**When** I manage personal knowledge  
**Then** UI should only expose two actions: `清空个人记忆` and `重新整理个人记忆`  
**And** `清空个人记忆` must require secondary confirmation and clear memory safely

### AC-6: System Confirmation 必须前置短路（非 LLM 业务确认）

**Given** a `WIDGET_SUBMIT` from personal KB system confirmation (`origin=system.personal_kb`)  
**When** Main processes this submit event  
**Then** commit/reject must be handled in Main pre-LLM path  
**And** it must not call `callAgentChat` or trigger any model request

### AC-7: 候选提交防重放与幂等

**Given** a `WIDGET_SUBMIT` from personal KB system confirmation  
**When** Main validates and processes the submit  
**Then** it must verify `origin=system.personal_kb` and enforce `candidateId + conversationId + TTL` validation before commit/reject  
**And** if candidate is missing, expired, conversation mismatched, or already finalized, it must reject with deterministic error and no file write  
**And** the same `candidateId` can only be finalized once (idempotent), duplicate submit must not produce duplicate writes

## Technical Notes

- Scope: personal KB foundation for `chat/agent/run` write path (storage + candidate commit); retrieval/injection remains chat-only in Story 12.2.
- Delivery pattern: vertical full-stack in one story (Main/IPC/Renderer/Test together).
- Storage boundary: Runtime private state (`@state/kb/personal/`), not project root.
- Truth source: Markdown files are source of truth; `index.json` is local acceleration metadata, not the truth source. KB MVP 不使用 SQLite。
- UX boundary: Settings 对普通用户仅暴露“清空个人记忆 / 重新整理个人记忆”，不显示文件级技术细节。
- Interaction boundary: personal KB confirmation 属于 system-origin pre-LLM gate，不等同于 LLM 普通 confirmation。
- Security/idempotency boundary: `WIDGET_SUBMIT(origin=system.personal_kb)` 必须做 `candidateId + conversationId + TTL` 校验，并保证一次性提交（no double-write）。
- Routing boundary: 三模式 (`chat/agent/run`) 都可触发“入库路由”；但个人知识注入仅在 chat 模式（Story 12.2）。
- Layering boundary: Story 12.1 负责写入分层契约。对于 `MEMORY.md`，必须明确区分 `Pinned`（固定层）与 `General`（检索层）；Story 12.2 只消费该契约，不负责反推。
- Mutation boundary: `UPDATE/DELETE` 采用软状态变更（`active/superseded/deleted`）并保留审计轨迹，不做直接硬删除。
- Design: `_bmad-output/implementation-artifacts/design-12-1-personal-kb-storage-and-candidate-commit.md`
- Cross-Story Review: `_bmad-output/implementation-artifacts/code-review-12-1-12-2-personal-kb-layering-contract.md`
- Single-Story Review: `_bmad-output/implementation-artifacts/code-review-12-1-personal-kb-storage-and-candidate-commit.md`

## Tasks / Subtasks

- [x] Task 1: Runtime Main + Store（后端）实现（AC: 1,2,3,4,5,6,7）
  - [x] 1.1 创建 `@state/kb/personal/` 目录与基础文件 + `manifest.json`
  - [x] 1.2 实现 LLM 路由（结构化输出）+ 候选记忆生成与“未确认不落盘”门控
  - [x] 1.3 实现确认提交写入（含 `source` / `updatedAt`）
  - [x] 1.4 实现增量索引更新与全量重建
  - [x] 1.5 实现清空个人记忆（`USER.md` + `SOUL.md` + `MEMORY.md` + `memory/*.md` 清空、索引清零）
  - [x] 1.6 实现候选 `TTL`、`conversationId` 绑定校验与提交幂等防重放（同一 `candidateId` 仅可 finalize 一次）

- [x] Task 2: IPC Contract（前后端接口）实现（AC: 1,2,3,4,5,6,7）
  - [x] 2.1 新增 personal-kb 初始化/候选/提交/重建/清空 IPC
  - [x] 2.2 统一错误码与返回结构，支持 UI 可视化提示（新增 `KB_PERSONAL_CANDIDATE_EXPIRED` / `KB_PERSONAL_CANDIDATE_INVALID` / `KB_PERSONAL_CANDIDATE_ALREADY_FINALIZED`）

- [x] Task 3: Renderer UI（前端）实现（AC: 1,2,3,4,5,6,7）
  - [x] 3.1 会话界面增加候选记忆确认交互（适配 `chat/agent/run` 入口，approve/reject）
  - [x] 3.2 Settings/Knowledge 面板仅保留极简治理操作（清空/重整）
  - [x] 3.3 增加“重建索引”与“清空个人记忆”结果反馈（成功/失败）
  - [x] 3.4 候选失效/重复提交时展示可理解错误反馈（不暴露技术细节）

- [x] Task 4: Integration & E2E 验证（AC: 1,2,3,4,5,6,7）
  - [x] 4.1 集成验证：候选 -> 确认 -> 落盘 -> 索引更新链路
  - [x] 4.2 负向验证：拒绝候选不写入
  - [x] 4.3 模式回归：`chat/agent/run` 均可显式入库；`agent/run` 不触发检索注入
  - [x] 4.4 路由异常验证：低置信/超时/非法 JSON 不误写入
  - [x] 4.5 清空验证：二次确认后清空个人记忆并重置索引
  - [x] 4.6 前置短路验证：`origin=system.personal_kb` 的 `WIDGET_SUBMIT` 不触发 LLM 调用
  - [x] 4.7 幂等与防重放验证：重复提交、候选过期、conversation 不匹配均不落盘并返回确定性错误码

- [x] Task 5: 文档与观测
  - [x] 5.1 记录关键日志事件（candidate created/committed/rejected/rebuild/cleared）
  - [x] 5.2 增加候选提交拦截日志（expired/invalid/already_finalized）
  - [x] 5.3 更新 File List 与变更说明，便于 code-review

- [x] Task 6: Cross-Story Review Follow-ups（Blocking）（AC: 3）
  - [x] 6.1 为 `MEMORY.md` 写入契约补齐明确分层语义：`Pinned`（固定层）/ `General`（检索层）/ `daily`
  - [x] 6.2 路由输出或系统提交路径必须携带 `targetSection`（或等价字段），不能只返回 `targetFile`
  - [x] 6.3 修正当前实现“所有 `MEMORY.md` 新增写入默认落到 `Pinned`”的问题
  - [x] 6.4 补回归测试：长期记忆进入 `Pinned`，长尾记忆进入 `General`，12.2 读取结果与写入契约一致

- [x] Task 7: Single-Story Review Follow-ups（Blocking）（AC: 2,3）
  - [x] 7.1 收窄 fallback 对 `USER.md` 的分类条件，避免仅因 `偏好/preference` 关键词就把场景性经验固化为固定层
  - [x] 7.2 `UPDATE/DELETE` 的本地目标解析必须优先使用路由器抽取的 `candidateText`，不能直接拿整句用户输入做主匹配
  - [x] 7.3 补回归测试：覆盖“带偏好关键词但应进入 `MEMORY.md#General`”和“旧值 + 新值同句”的 update/delete 解析场景

## Definition of Done（Blocking）

- [x] Task 1~5 及子任务全部完成并勾选。
- [x] Task 6 的 cross-story blocker 已关闭，`Pinned/General/daily` 写入契约与 12.2 读取契约一致。
- [x] Task 7 的 single-story review blocker 已关闭，fallback 分类与 update/delete target resolve 精度满足当前设计约束。
- [x] AC-1~AC-7 对应单元测试与集成测试通过（含 4.7 幂等/防重放）。
- [x] 验证 `origin=system.personal_kb` 提交短路不触发 LLM 调用。
- [x] 关键日志事件可观测：`candidate.created/committed/rejected`、`candidate.submit_blocked`、`index.rebuild.*`、`cleared`。
- [x] Story 状态恢复为 `ready-for-review` 前，附最新 review follow-up 的最小验证记录（测试结果或命令输出摘要）。

## Dev Notes (2026-03-10)

- 新增 `personalKbService`，实现候选生成、系统确认提交、`candidateId + conversationId + TTL` 校验、幂等 finalize 与错误码拦截。
- `chat:send` / `agent:dispatch` / `runs:continue` 均接入 personal KB 前置短路：
  - `origin=system.personal_kb` 的 `WIDGET_SUBMIT` 不再进入 LLM 循环；
  - 显式记忆意图会直接返回 `confirmation` widget block，待用户确认后写入。
- `RuntimeStore` 新增 personal KB 存储/写入/重建/清空/状态能力，目录位于 Runtime 私有目录 `runtime-store/kb/personal/`。
- Settings 新增极简治理卡片，仅保留 `清空个人记忆` 与 `重新整理个人记忆` 两个动作。
- 新增 mutation target resolve 与 `ADD/UPDATE/DELETE` 执行语义：
  - `ADD` 写入 `status=active`
  - `UPDATE` 先写旧记忆 `status=superseded`，再写新记忆 `status=active`
  - `DELETE` 写入 `status=deleted`
- 候选确认文案不再展示内部目标文件名，避免技术细节暴露给终端用户。

## Dev Notes (2026-03-11)

- `personalKbService` 现已把 `targetSection` 贯穿到 `LLM route -> candidate -> submit -> runtimeStore.applyPersonalKbMutation`，不再只靠 `targetFile`。
- 对 `MEMORY.md`：
  - `fallbackRoute` 会给出 `Pinned` / `General`；
  - LLM route 若缺失 `targetSection`，服务层会做规则归一化/兜底，而不是在提交阶段默认写入 `Pinned`。
- `runtimeStore.applyPersonalKbMutation()` 现已对 `MEMORY.md` 新增/更新做 section 校验：
  - 新增必须显式带 `Pinned|General`；
  - 更新可沿用已有 section，或接收新的 `targetSection` 完成分层迁移；
  - 不再允许 “`MEMORY.md` 无 section 默认进 `Pinned`”。
- 新增回归测试覆盖：
  - generic reusable memory -> `MEMORY.md#General`
  - always-on long-term memory -> `MEMORY.md#Pinned`
  - `MEMORY.md` 无有效 section 时写入失败
- Task 7 已补齐 single-story follow-ups：
  - fallback 仅在稳定身份/长期默认设定时写入 `USER.md`，不再因 `偏好/preference` 关键词把场景性经验误写为固定层；
  - `UPDATE/DELETE` 目标解析现已优先使用路由器抽取的 `candidateText`，整句用户输入仅作为辅助 fallback。
- 新增回归测试覆盖：
  - 带“偏好”关键词的 workflow hint 进入 `MEMORY.md#General`；
  - 路由器抽取旧值后，update target resolve 优先命中 `candidateText`。

## Code-Review Open Questions Resolution (2026-03-10)

1. `UPDATE/DELETE` 如何判定目标？
   - 结论：由系统做本地目标解析（active entry match + 歧义检测），LLM 路由结果仅作为候选意图。
2. 软状态会不会影响后续注入？
   - 结论：Story 12.2 明确注入层仅使用 `status=active`，并处理 `superseded/deleted`。
3. 候选确认是否展示文件名？
   - 结论：不展示，用户只看到“操作类型 + 候选内容”。

## Current Cross-Story Review Follow-ups (2026-03-11)

- 当前结论：12.1 的 cross-story blocker 已关闭，story 可标记为 `done`。
- 已完成内容：
  - 路由/候选/提交链路现已显式携带 `targetSection`；
  - `MEMORY.md` 写入不再默认进入 `Pinned`；
  - `Pinned / General / daily` 的落盘契约已与 12.2 的读取契约对齐。
- 当前统一决议：
  - `SOUL.md` / `USER.md`：固定层核心记忆；
  - `MEMORY.md#Pinned`：长期但不属于 `SOUL/USER` 的固定层记忆；
  - `MEMORY.md#General`：可复用但非固定生效的检索层记忆；
  - `memory/YYYY-MM-DD.md`：daily 检索层记忆。
- 索引契约统一为 `index.json`；Epic 12 的 KB MVP 不再保留 SQLite 作为当前设计前提。
- 12.2 的 `recent non-empty daily` 检索窗口问题已关闭；当前无未关闭的 cross-story blocker。

## Current Single-Story Review Follow-ups (2026-03-11)

- 当前结论：single-story findings 已修复，12.1 可标记为 `done`。
- 已完成内容：
  - fallback 对 `USER.md` 的判定已收窄，不再仅因 `偏好/preference` 关键词把场景性经验误写到固定层；
  - `UPDATE/DELETE` target resolve 现已优先使用路由器抽取的 `candidateText`，整句用户输入仅作为辅助 fallback；
  - 新增回归测试覆盖“偏好关键词 -> General”和“旧值 + 新值同句 -> candidateText 优先解析”；
  - `IDENTITY.md` 已从 personal KB 契约中移除，不再作为初始化结构、路由 allowlist 或清空语义的一部分；legacy `IDENTITY.md` 会在初始化时自动清理。
- 当前统一决议：
  - `USER.md` 只保留稳定身份、称呼、语言与长期默认风格；
  - scene/workflow-specific 可复用经验默认进入 `MEMORY.md#General`；
  - `UPDATE/DELETE` 的目标定位以 `candidateText` 为主，不再直接用整句用户输入做主匹配；
  - personal KB 当前真相源仅包括 `USER.md`、`SOUL.md`、`MEMORY.md` 与 `memory/YYYY-MM-DD.md`。

## File List

- `crewagent-runtime/electron/services/personalKbService.ts` (new)
- `crewagent-runtime/electron/services/personalKbService.test.ts` (new)
- `crewagent-runtime/electron/services/chatContextBuilder.ts`
- `crewagent-runtime/electron/services/chatContextBuilder.test.ts`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/stores/runtimeStore.test.ts`
- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/preload.ts`
- `crewagent-runtime/electron/electron-env.d.ts`
- `crewagent-runtime/shared/conversationTypes.ts`
- `crewagent-runtime/src/pages/RunsPage/components/widgets/widgetSubmission.ts`
- `crewagent-runtime/src/pages/RunsPage/components/widgets/widgetMessageParser.ts`
- `crewagent-runtime/src/stores/appStore.ts`
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx`
- `crewagent-runtime/src/vite-env.d.ts`

## Verification

- `npm --prefix crewagent-runtime test -- electron/services/personalKbService.test.ts electron/stores/runtimeStore.test.ts electron/services/chatContextBuilder.test.ts`
  - 3 个测试文件，84 个测试全部通过（2026-03-11）。
- `cd crewagent-runtime && npx tsc --noEmit`
  - TypeScript 全量检查通过（2026-03-11）。

## Archived Historical Review Follow-ups (AI)

> 说明：本节保留的是历史单-story review 快照。其原始开闭状态已由下方 `Review Follow-up Resolution (AI)` 覆盖；当前真正未关闭的 blocker 只看上方 `Task 6` 和 `Current Cross-Story Review Follow-ups`。

- [x] [AI-Review][HIGH] 历史上的 `index.sqlite` / JSON 契约冲突已归档关闭；当前 KB MVP 统一使用 `index.json`，不再以 SQLite 作为本 story 的设计前提。（`crewagent-runtime/electron/stores/runtimeStore.ts`）
- [x] [AI-Review][HIGH] 个人记忆提交链路原子性问题已修复并归档。（`crewagent-runtime/electron/stores/runtimeStore.ts`, `crewagent-runtime/electron/services/personalKbService.ts`）
- [x] [AI-Review][HIGH] Story 验证记录已重跑并刷新到当前 targeted verification。（`_bmad-output/implementation-artifacts/12-1-personal-kb-storage-and-candidate-commit.md`, `crewagent-runtime/electron/stores/runtimeStore.test.ts`）
- [x] [AI-Review][MEDIUM] allowlist 外 `targetFile` 的安全 fallback 已补齐并归档。（`crewagent-runtime/electron/services/personalKbService.ts`）
- [x] [AI-Review][MEDIUM] recent daily 窗口问题已转入 Story 12.2 / cross-story review 统一跟踪，不再作为 12.1 单独 blocker 追踪。（`crewagent-runtime/electron/stores/runtimeStore.ts`, `crewagent-runtime/electron/stores/runtimeStore.test.ts`）
- [x] [AI-Review][MEDIUM] Story File List 不完整，遗漏了这次改动实际引入的 chat context builder 文件，降低了 code-review 可追踪性。（`_bmad-output/implementation-artifacts/12-1-personal-kb-storage-and-candidate-commit.md`, `crewagent-runtime/electron/services/chatContextBuilder.ts`, `crewagent-runtime/electron/services/chatContextBuilder.test.ts`）

## Review Follow-up Resolution (AI)

> 注：上方历史快照里如仍有旧 checkbox 未改写，以本节的 resolution 状态为准。

- [x] [AI-Review][HIGH] 索引契约已与实现对齐：个人 KB 索引文件统一为 `index.json`，story/design/architecture/PRD 同步修正为 JSON 元数据加速层。
- [x] [AI-Review][HIGH] 个人记忆提交链路已改为“先构建快照、后统一提交”，提交失败会回滚 Markdown / index / manifest，避免重复 side effect。
- [x] [AI-Review][HIGH] story 验证记录已刷新并重跑；最新 targeted verification 见上方 `Verification` 节（当前为 3 个测试文件、84 个测试全部通过，且 `tsc --noEmit` 通过）。
- [x] [AI-Review][MEDIUM] LLM 路由返回 allowlist 之外的 `targetFile` 时，当前实现已记录告警并回退到安全 fallback，而不是静默跳过显式记忆请求。
- [x] [AI-Review][MEDIUM] 移除失效审查项：recent daily 规则已在 Story 12.2 中收敛为 recent non-empty daily，当前不再作为 12.1 的开放缺陷追踪。
- [x] [AI-Review][MEDIUM] Story File List 已补全 `chatContextBuilder.ts` / `chatContextBuilder.test.ts`，review 范围与当前 diff 一致。

## Change Log

- 2026-03-11: Senior developer review completed; 3 HIGH / 3 MEDIUM issues logged; status set to in-progress.
- 2026-03-11: Follow-up fixes completed for review findings; story/docs aligned to `index.json`, atomic snapshot commit added, invalid-target fallback covered by tests, status returned to ready-for-review.
- 2026-03-11: Cross-story review reopened 12.1; write contract must explicitly distinguish `Pinned/General/daily`, status returned to `in-progress`.
- 2026-03-11: `targetSection` / `Pinned vs General vs daily` 写入契约已落地，12.1 cross-story blocker 关闭，状态恢复为 `ready-for-review`。
- 2026-03-11: Single-story review reopened 12.1; fallback classification precision and update/delete target resolve must be corrected before returning to `ready-for-review`.
- 2026-03-11: Task 7 fixes completed; fallback classification and update/delete target resolve precision corrected, targeted verification refreshed, status returned to `ready-for-review`.
- 2026-03-11: `IDENTITY.md` 从 personal KB 契约中移除；legacy 文件初始化自动清理，story verification 与 review records 刷新到最新实现。
- 2026-03-11: Story 12-1 review approved，BMAD 状态正式收尾为 `done`。

## Senior Developer Review (AI)

### Review Outcome

- Date: 2026-03-11
- Outcome: Changes Requested (resolved 2026-03-11)
- Issues: 6 logged, 0 open after follow-up fixes
- Git vs Story Discrepancies: 0 open after File List sync
- Acceptance Criteria: PASS after follow-up fixes and targeted verification
- Note: 上述结论已被后续 cross-story review 与 `IDENTITY.md` 契约收敛更新覆盖；Story 12-1 当前已正式收尾为 `done`。

### Findings

1. Archived and resolved: KB MVP index contract now uses `index.json`; SQLite is no longer part of Story 12.1 / 12.2 accepted scope. (`crewagent-runtime/electron/stores/runtimeStore.ts`)
2. `applyPersonalKbMutation()` appends/supersedes/deletes Markdown entries before rebuilding the index, but returns failure without rollback if rebuild fails. Because `handleSubmit()` leaves the candidate pending on mutation failure, the same candidate can be retried and write duplicate entries. (`crewagent-runtime/electron/stores/runtimeStore.ts`, `crewagent-runtime/electron/services/personalKbService.ts`)
3. The story’s own verification record is false at the moment. Running `npm --prefix crewagent-runtime test -- electron/services/personalKbService.test.ts electron/stores/runtimeStore.test.ts` still fails in `lists recent daily memory targets in descending date order`, so the DoD line claiming AC tests passed is not supportable. (`_bmad-output/implementation-artifacts/12-1-personal-kb-storage-and-candidate-commit.md`, `crewagent-runtime/electron/stores/runtimeStore.test.ts`)
4. When LLM routing returns a disallowed `targetFile`, `maybeCreateCandidate()` nulls the route and exits without falling back. That violates AC-2’s “invalid routing must fall back safely” requirement and can silently skip an explicit memory-write request. (`crewagent-runtime/electron/services/personalKbService.ts`)
5. `listRecentPersonalKbDailyTargets()` 会把初始化自动创建的空白今日文件一并计入 recent window。结果是当 `days=2` 时，真实的 `2026-03-09.md` 会被空白的 `2026-03-11.md` 挤掉，检索窗口缩小，而且这正是当前 targeted test 失败的原因。 (`crewagent-runtime/electron/stores/runtimeStore.ts`, `crewagent-runtime/electron/stores/runtimeStore.test.ts`)
6. The File List claims review readiness but omits `electron/services/chatContextBuilder.ts` and `electron/services/chatContextBuilder.test.ts`, both of which are part of the current diff and directly support the memory-injection path. The story is therefore not a complete record of the implementation under review. (`_bmad-output/implementation-artifacts/12-1-personal-kb-storage-and-candidate-commit.md`, `crewagent-runtime/electron/services/chatContextBuilder.ts`, `crewagent-runtime/electron/services/chatContextBuilder.test.ts`)

### Resolution Status

1. Resolved: personal KB index contract now uses `index.json`, and related story/design/architecture/PRD references were aligned to the shipped implementation.
2. Resolved: `applyPersonalKbMutation()` now stages Markdown/index/manifest writes, then commits them with rollback protection so failed commits do not leave duplicate side effects.
3. Resolved: verification evidence was rerun and updated to the current targeted command; 84 targeted tests now pass.
4. Resolved: invalid allowlist targets now emit `kb.personal.route.invalid_target_fallback` and downgrade to safe fallback routing.
5. Archived as superseded: the old recent-daily finding was moved into Story 12.2 cross-story tracking, and the final implementation now uses recent non-empty daily retrieval.
6. Resolved: File List now includes `chatContextBuilder.ts` and `chatContextBuilder.test.ts`, matching the actual review surface.
