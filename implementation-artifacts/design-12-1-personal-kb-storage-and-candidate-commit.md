# Design: Personal KB Storage & Candidate Commit (Cross-Mode Write)

**Story:** `12-1-personal-kb-storage-and-candidate-commit.md`  
**设计原则:** Markdown 真相源、LLM 路由 + 系统硬约束、显式确认写入、三模式入库统一、写入分层明确、注入边界清晰

---

## 设计目标

1. 在 Runtime 私有目录初始化个人知识库固定结构（不落在项目根目录）。
2. 仅当用户明确“记忆意图”时生成候选，并通过确认交互后才允许落盘。
3. 每次确认写入都带 `source` / `updatedAt` 元数据，并增量更新索引。
4. 提供面向非技术用户的最小治理能力：`清空个人记忆` 与 `重新整理记忆`。
5. `UPDATE/DELETE` 通过系统目标解析 + 软状态变更落盘，保证后续注入可过滤旧记忆。
6. 明确 `MEMORY.md` 的写入分层：`Pinned`（固定层）/ `General`（检索层）/ `daily`（仅日记忆）。

---

## 非目标（本 Story 不做）

1. 不做个人知识检索注入回答（Story 12.2）。
2. 不在 Agent/Run 模式接入个人知识检索注入（仅支持显式入库）。
3. 不做个人知识可视化编辑器（后续 Story 12.6）。
4. 不暴露文件级技术细节（路径/文件存在性）给普通用户。

---

## 当前 Cross-Story Review 约束（2026-03-11）

1. Story 12.1 是个人知识“写入分层契约”的主责 story，必须决定某条记忆进入 `Pinned`、`General` 还是 `daily`。
2. Story 12.2 只消费该契约，不负责从 `MEMORY.md` 内容反推 fixed/retrieval 归属。
3. KB MVP 本地索引统一为 `index.json` 元数据层；本 design 不再以 SQLite 作为前提。
4. 当前已落地：`targetSection` 贯穿路由/候选/提交；Store 提交层不再允许 `MEMORY.md` 默认落到 `Pinned`。

## 当前 Single-Story Review 约束（2026-03-11）

1. fallback 规则不能因为命中 `偏好/preference` 就直接把内容写入 `USER.md`；`USER.md` 只保留稳定身份、称呼、语言、长期默认风格等固定用户设定。
2. scene/workflow-specific 的可复用经验，即使包含“偏好”表达，只要缺少长期固定信号，也应优先进入 `MEMORY.md#General`。
3. `UPDATE/DELETE` 的本地目标解析必须优先使用 LLM 路由抽取的 `candidateText`；完整用户输入只能作为辅助判别或 fallback，不得直接作为主匹配文本。
4. 设计验收必须补回归测试，覆盖“偏好关键词误入 USER”与“旧值 + 新值同句”的目标解析场景。

---

## 改动范围

| 文件 | 改动类型 | 说明 |
|------|----------|------|
| `crewagent-runtime/electron/stores/runtimeStore.ts` | MODIFY | 增加 personal KB 路径/初始化/写入/重建能力 |
| `crewagent-runtime/electron/services/personalKbService.ts` | NEW | 候选生成、确认提交、重建编排（跨模式写入） |
| `crewagent-runtime/electron/services/personalKbService.test.ts` | NEW | 候选/提交/拒绝/重建单测 |
| `crewagent-runtime/electron/main.ts` | MODIFY | `chat:send`/`agent:dispatch`/`runs:*` 接入候选确认链路；新增 personal KB IPC |
| `crewagent-runtime/electron/preload.ts` | MODIFY | 暴露 personal KB IPC 给 Renderer |
| `crewagent-runtime/src/vite-env.d.ts` | MODIFY | 补充 IPC 类型声明 |
| `crewagent-runtime/src/stores/appStore.ts` | MODIFY | personal KB 简化治理 action（clear/rebuild） |
| `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx` | MODIFY | 增加 personal KB 极简治理卡片（清空/重整） |
| `crewagent-runtime/src/pages/SettingsPage/SettingsPage.css` | MODIFY | personal KB 卡片样式 |

---

## 存储与数据模型

### 1) 目录布局（Runtime 私有）

```text
<userData>/runtime-store/kb/personal/
  USER.md
  SOUL.md
  MEMORY.md
  memory/
    YYYY-MM-DD.md
  index.json
  manifest.json
```

说明：
- 目录由 Runtime 自动创建，不依赖项目目录存在。
- Markdown 是事实源；`index.json` 是检索/统计加速元数据层，不是事实源。

### 2) `manifest.json`（建议字段）

```json
{
  "version": "1.0",
  "initializedAt": "2026-03-09T10:00:00.000Z",
  "updatedAt": "2026-03-09T10:05:00.000Z",
  "index": {
    "status": "ready",
    "lastIndexedAt": "2026-03-09T10:05:00.000Z",
    "entryCount": 12
  },
  "files": [
    { "path": "SOUL.md", "updatedAt": "2026-03-09T10:05:00.000Z", "sha256": "..." }
  ]
}
```

### 3) 候选模型（仅内存，不持久化）

```ts
type PersonalKbCandidate = {
  candidateId: string
  conversationId: string
  sourceMode: 'chat' | 'agent' | 'run'
  sourceText: string
  normalizedText: string
  routeIntent: 'ADD_MEMORY' | 'UPDATE_MEMORY' | 'DELETE_MEMORY'
  targetFile: 'USER.md' | 'SOUL.md' | 'MEMORY.md' | `memory/${string}.md`
  targetSection?: 'Pinned' | 'General'
  mutationTarget?: {
    targetFile: 'USER.md' | 'SOUL.md' | 'MEMORY.md' | `memory/${string}.md`
    text: string
    memoryId?: string
    legacyTextKey: string
    sectionTitle?: string
    score: number
  }
  routeConfidence: number
  routeReason?: string
  createdAt: string
  expiresAt: string
  status: 'pending' | 'committed' | 'rejected' | 'expired'
  finalizedAt?: string
}
```

说明：
- 未确认候选只驻留内存（Map），应用重启即丢弃，满足“未确认不落盘”。
- 候选默认 TTL（建议 30 分钟）；超过 `expiresAt` 后不可提交。
- 提交状态机：`pending -> committed|rejected|expired`，同一 `candidateId` 只能 finalize 一次。
- 对 `MEMORY.md` 候选，`targetSection` 是必需语义，至少区分 `Pinned` 与 `General`。

### 4) Markdown 记录格式（提交后）

统一追加为可读 + 可解析块：

```md
### [2026-03-09T10:05:00.000Z] 以后默认使用中文回复
- source: mode:<mode>/conversation:<conversationId>
- updatedAt: 2026-03-09T10:05:00.000Z
- memoryId: 2f2f5c4a-bd8b-45f4-8c12-0dd4a3f2e57d
- status: active
```

`UPDATE/DELETE` 追加状态块示例：

```md
### [2026-03-09T11:00:00.000Z] 以后默认使用中文回复
- source: mode:chat/conversation:conv-1
- updatedAt: 2026-03-09T11:00:00.000Z
- memoryId: 2f2f5c4a-bd8b-45f4-8c12-0dd4a3f2e57d
- status: superseded
- supersededBy: 5c63f527-46ef-43bb-9f4f-f9b2d72f64ce
```

该格式用于重建索引时回扫解析，并支持注入层按 `status` 过滤。

`MEMORY.md` 的推荐结构：

```md
# MEMORY

## Pinned
### [2026-03-09T10:05:00.000Z] 长期偏好：先给结论
...

## General
### [2026-03-09T10:06:00.000Z] 长尾偏好：输出里给一个示例
...
```

---

## 核心流程设计

### A. 初始化流程（AC-1）

1. Runtime 启动后，`RuntimeStore.ensurePersonalKbInitialized()` 保障目录与基础文件存在。
2. 三模式用户输入入口调用前再次幂等确保（防止历史版本目录缺失）。
3. 创建缺失的当天文件 `memory/YYYY-MM-DD.md`。
4. `manifest.json` 写入初始化元数据；`index.json` 缺失时创建空索引元数据。

### B. 候选生成与确认（AC-2，三模式）

1. `chat:send` / `agent:dispatch` / `runs:*` 的用户输入入口先检查是否为 `WIDGET_SUBMIT`。
2. 若非 `WIDGET_SUBMIT`，调用 LLM 路由器执行结构化分类与抽取，输出：
   - `intent`: `ADD_MEMORY | UPDATE_MEMORY | DELETE_MEMORY | NO_MEMORY | UNKNOWN`
   - `targetFile`: `USER.md | SOUL.md | MEMORY.md | memory/YYYY-MM-DD.md`
   - `targetSection`: `Pinned | General`（仅当 `targetFile=MEMORY.md` 时必填）
   - `candidateText`: 候选写入文本
   - `confidence`: `0..1`
   - `reason`: 可解释路由依据（供日志/调试）
3. 服务层执行硬约束校验：`targetFile` 白名单、结构完整性、长度限制。
4. 若 `targetFile=MEMORY.md`，服务层必须再校验 `targetSection`：
   - 长期固定生效 -> `Pinned`
   - 可复用但非固定生效 -> `General`
   - 若 LLM route 缺失该字段，可在服务层通过规则兜底/旧 section 归一化补齐；
   - 但提交层不得在 `MEMORY.md` 新增写入时默认落到 `Pinned`
   - fallback 对 `USER.md` 的判定必须收窄：只有稳定身份与长期默认设定才可直达 `USER.md`；不能仅凭 `偏好/preference` 关键词判定为固定层
5. 置信度门控：
   - `confidence >= T_high`：生成候选（仅内存）并下发 `confirmation` 组件。
   - `T_low <= confidence < T_high`：返回追问确认，不生成可提交候选。
   - `< T_low` 或结构非法：走规则兜底（关键词/模板）或直接判定 `NO_MEMORY`。
6. 若 `intent` 为 `UPDATE_MEMORY/DELETE_MEMORY`，提交候选前必须先做本地目标解析：
   - 优先使用 route 输出的 `candidateText` 作为主查询；
   - 完整用户输入仅用于辅助歧义消解，不作为第一匹配源；
   - 唯一命中 active 条目：允许进入确认；
   - 未命中或歧义：不生成可提交候选，记录 `target_not_resolved` 日志并走追问/普通对话路径。
7. 未确认前，不写任何知识文件。

### C. 提交/拒绝（AC-2/AC-3）

1. 用户提交 widget 后，Renderer 从当前模式入口发送 `WIDGET_SUBMIT`。
2. Main 识别 `widgetId` 前缀（如 `personal_kb_commit:<candidateId>`），并按顺序执行校验：
   - `origin` 必须为 `system.personal_kb`
   - `candidateId` 存在且处于 `pending`
   - `conversationId` 必须匹配候选来源会话
   - 当前时间不得超过 `expiresAt`
   - 若候选已 `committed/rejected/expired`，视为重复提交或失效
3. 校验通过后执行：
   - `confirmed=true`：
     - `ADD_MEMORY`: 追加 `status=active` 记忆块；
     - `UPDATE_MEMORY`: 追加旧记忆 `status=superseded` + 新记忆 `status=active`；
     - `DELETE_MEMORY`: 追加 `status=deleted` 记忆块；
     - 所有写入都带 `source/updatedAt/memoryId` 元数据，并增量更新索引。
     - `MEMORY.md` 新增写入不得统一默认进入 `Pinned`，必须遵循 `targetSection` 契约。
   - `confirmed=false`：仅丢弃候选。
4. 返回 assistant ack 文案；并通过 `conversations:updated` 刷新会话显示。
5. 幂等约束：同一 `candidateId` 重复 `WIDGET_SUBMIT` 必须返回确定性错误，不得产生二次写入。

说明（避免歧义）：
- personal KB 使用的是 **System confirmation**（系统门控确认），不是 LLM 在对话中临时生成的业务确认。
- 两者可复用同一个 `confirmation` UI 组件，但必须通过元数据区分来源：
  - `widgetId`: `personal_kb_commit:<candidateId>`
  - `origin`: `system.personal_kb`
- 用户可见文案必须隐藏内部文件路径/文件名，只展示“操作类型 + 候选内容”。
- 对 `origin=system.personal_kb` 的 `WIDGET_SUBMIT`，Main 必须前置处理并短路返回，不得进入 LLM 循环。
- 推荐错误码：
  - `KB_PERSONAL_CANDIDATE_EXPIRED`
  - `KB_PERSONAL_CANDIDATE_INVALID`（不存在、会话不匹配、来源非法）
  - `KB_PERSONAL_CANDIDATE_ALREADY_FINALIZED`

### D. 索引重建（AC-4）

1. Settings 触发 `kb:personal:rebuildIndex`。
2. 服务层遍历 `USER/SOUL/MEMORY/memory/*.md`，解析条目并重建 `index.json`。
3. 成功后刷新 `manifest.index`（`status/lastIndexedAt/entryCount`）。
4. 若重建失败，返回结构化错误并保留 Markdown 真相源不变。

### E. 清空个人记忆（新增，AC-5）

1. Settings 触发 `kb:personal:clearAll`（需二次确认）。
2. 服务层清空 `USER.md`、`SOUL.md`、`MEMORY.md` 与 `memory/*.md` 内容（保留目录与基础文件）。
3. 重置 `index.json`（清空或重建为空索引元数据），并刷新 `manifest.index` 计数为 0。
4. 返回简化结果文案（成功/失败）；失败时不破坏基础结构文件。

### F. LLM 路由契约（新增）

1. 路由调用采用 `response_format=json_schema`（或同等级结构化约束），禁止自由文本解析。
2. Route schema 版本化（如 `personal_kb_route.v1`），后续迭代兼容。
3. 路由失败策略：
   - 超时/429/网络异常 -> 规则兜底；
   - schema 校验失败 -> 丢弃该次路由结果并记录 `KB_PERSONAL_ROUTE_SCHEMA_INVALID`；
   - 白名单校验失败 -> 强制降级为追问确认，不进入 commit。
4. 当 `targetFile=MEMORY.md` 时，route schema 必须包含 `targetSection: Pinned | General`（或等价字段），不能只返回文件名。
5. 路由结果仅用于“候选生成”，最终写入仍以用户确认为唯一提交门槛。
6. 若 `targetSection` 缺失或非法，系统必须在服务层降级为规则兜底/追问；提交层不得把缺失 section 的 `MEMORY.md` 写入默认落到 `Pinned`。

---

## IPC 合约（新增）

### `kb:personal:getStatus`

返回：

```ts
{
  success: true,
  status: {
    initialized: boolean
    rootPath: string
    files: Array<{ path: string; exists: boolean; sizeBytes: number }>
    index: { exists: boolean; status: 'ready' | 'stale' | 'missing'; lastIndexedAt?: string; entryCount?: number }
  }
}
```

### `kb:personal:rebuildIndex`

请求：

```ts
{ force?: boolean }
```

返回：

```ts
{ success: true, result: { entryCount: number, rebuiltAt: string } }
```

错误码（示例）：`KB_PERSONAL_REBUILD_FAILED` / `KB_PERSONAL_NOT_INITIALIZED`

### `kb:personal:clearAll`

请求：

```ts
{ confirmed: boolean }
```

返回：

```ts
{ success: true, result: { clearedFiles: number, clearedAt: string } }
```

错误码（示例）：`KB_PERSONAL_CLEAR_REQUIRES_CONFIRM` / `KB_PERSONAL_CLEAR_FAILED`

### `WIDGET_SUBMIT`（personal KB system confirmation）

当 `origin=system.personal_kb` 时，提交路径建议统一返回：

```ts
{
  success: boolean,
  result?: { committed: boolean; candidateId: string },
  errorCode?:
    | 'KB_PERSONAL_CANDIDATE_EXPIRED'
    | 'KB_PERSONAL_CANDIDATE_INVALID'
    | 'KB_PERSONAL_CANDIDATE_ALREADY_FINALIZED'
}
```

---

## 模式集成点（写入三模式，注入仅 Chat）

在三模式用户输入入口加入前置分支：

1. 先处理 personal KB widget 提交（commit/reject）。
2. 再处理 LLM 记忆路由并生成候选。
3. 两者都未命中时，回到各模式原有流程（chat/agent/run）。

约束：
- 三模式均可触发“显式入库”链路；
- 个人知识“检索/注入”仅在 Chat（Story 12.2）启用，Agent/Run 必须跳过。
- `chat:send` / `agent:dispatch` / `runs:*` 的处理优先级固定为：
  1. `WIDGET_SUBMIT` 且 `origin=system.personal_kb` -> 直接 commit/reject（不进 LLM）
  2. personal KB 路由候选生成（需要时下发 system confirmation）
  3. 其余请求才进入原有 LLM 执行链路

---

## UI 设计（本 Story 最小可用）

### Works / Conversation

- 复用现有 widget 渲染与 `WIDGET_SUBMIT` 提交流程，不新增组件类型。
- personal KB 候选确认使用 `confirmation` widget，但其语义为 **System confirmation（前置门控）**。
- 必须与 LLM 生成的普通 confirmation 区分来源：`origin=system.personal_kb`。

### Settings

- 增加 “Personal Knowledge Base” 极简卡片，仅展示：
  - 简单状态文案（`正常` / `需要整理`）
- 提供两个按钮：
  - `清空个人记忆`（危险操作，二次确认）
  - `重新整理个人记忆`（触发重建索引）
- 默认不展示文件级细节（路径、文件存在性、技术状态码）。

---

## 观测与日志

新增日志事件（`runtimeStore.addLog`）：

- `kb.personal.candidate.created`
- `kb.personal.candidate.committed`
- `kb.personal.candidate.rejected`
- `kb.personal.candidate.submit_blocked`
- `kb.personal.cleared`
- `kb.personal.index.rebuild.started`
- `kb.personal.index.rebuild.completed`
- `kb.personal.index.rebuild.failed`

日志字段至少包含：`conversationId/candidateId/targetFile/result/errorCode`。

---

## 测试方案

### 单元测试

1. 初始化：首次调用创建完整目录结构与默认文件。
2. 候选：LLM 路由高置信触发候选；低置信触发追问；普通输入不触发。
3. 提交：`confirmed=true` 写入 Markdown 且包含 `source/updatedAt`。
4. 拒绝：`confirmed=false` 不写入文件。
5. 路由异常：LLM 超时/返回非法 JSON 时触发兜底路径，且不误写入。
6. 清空：执行 `clearAll` 后 `USER.md`、`SOUL.md`、`MEMORY.md` 与 `memory/*.md` 内容清空，索引归零。
7. 重建：删除/损坏 `index.json` 后可从 Markdown 重新生成。
8. 防重放：`candidate` 过期、会话不匹配、重复提交时返回确定性错误且无落盘。
9. fallback 精度：包含“偏好/preference”但属于场景性经验的输入，应进入 `MEMORY.md#General`，不得误写到 `USER.md`。
10. `UPDATE/DELETE` 解析：当用户输入同时包含旧值与新值时，系统应优先使用 `candidateText` 命中目标，而不是直接用整句做主匹配。

### 集成测试（Main + IPC）

1. `chat:send(记忆意图)` -> LLM 路由返回高置信 -> 收到 widget request。
2. `agent:dispatch/runs:*` 输入显式记忆意图 -> LLM 路由 -> 收到 widget request。
3. LLM 路由低置信或非法结构 -> 追问/兜底，不落盘。
4. `UPDATE/DELETE` 但未找到唯一目标 -> 不生成可提交候选，不落盘。
5. 任一模式 `WIDGET_SUBMIT confirmed=true` -> 文件落盘 + 索引更新。
6. 任一模式 `WIDGET_SUBMIT confirmed=false` -> 无落盘。
7. `kb:personal:clearAll(confirmed=true)` -> 记忆清空 + 索引清零。
8. `WIDGET_SUBMIT(origin=system.personal_kb)` 命中时，不触发 `callAgentChat`/LLM 请求。
9. `WIDGET_SUBMIT` 重复提交同一 `candidateId` -> 返回 `KB_PERSONAL_CANDIDATE_ALREADY_FINALIZED` 且无重复写入。
10. 候选确认 UI 不显示 `MEMORY.md`/`memory/*.md` 等内部文件名。
11. `WIDGET_SUBMIT` 使用过期候选或会话不匹配 -> 返回 `KB_PERSONAL_CANDIDATE_EXPIRED`/`KB_PERSONAL_CANDIDATE_INVALID` 且无落盘。
12. 包含“偏好/preference”但明显属于场景性经验的输入 -> 生成 `MEMORY.md#General` 候选，而不是 `USER.md`。
13. 更新指令同时包含旧值与新值时 -> target resolve 优先基于 `candidateText` 命中唯一 active 条目。

### 手工验证

1. Chat 中输入“记住这个：以后默认中文”，确认后检查 `MEMORY.md`。
2. Settings 点击“清空个人记忆”，二次确认后检查记忆文件与索引归零。
3. Settings 点击“重新整理个人记忆”，检查状态变化与日志事件。
4. Agent/Run 模式执行一轮，确认可触发显式入库；但不触发个人知识检索/注入。
5. 候选确认后再次点击提交，确认系统给出“已处理”反馈且文件没有重复写入。

---

## Code-Review 开放问题答复（12-1）

1. **`UPDATE/DELETE` 如何判断目标？**  
   由系统本地解析 active 记忆条目并做歧义检测；LLM 仅提供路由意图与候选文本。解析时应优先使用 `candidateText`，未命中/歧义时不进入可提交候选。
2. **候选确认是否显示内部目标文件？**  
   不显示。用户只看到“操作类型 + 内容”，内部文件路径完全透明。
3. **软状态会不会导致注入混乱？**  
   Story 12.2 明确注入只消费有效 `status=active` 条目，并处理 `superseded/deleted` tombstone。

---

## 风险与取舍

1. **候选内存态不跨重启保留**：优点是满足“未确认不持久化”；代价是重启后需重新触发候选。
2. **LLM 路由带来不确定性**：通过 schema 约束 + 置信度门控 + 白名单校验 + 规则兜底控制风险。
3. **索引失败不影响真相源**：Markdown 写入成功优先，索引失败通过 `status=stale` 可恢复。
4. **清空操作不可逆风险**：通过二次确认 + 明确文案降低误操作概率。
5. **重复提交/重放风险**：通过 `candidateId + conversationId + TTL` 校验与一次性 finalize 状态机避免重复落盘。

---

## 验收映射

- AC-1 → 初始化流程 + 目录结构检查
- AC-2 → 候选仅内存 + confirmation widget gate
- AC-3 → 提交写入元数据 + 增量索引
- AC-4 → `kb:personal:rebuildIndex` 从 Markdown 回扫重建
- AC-5 → `kb:personal:clearAll` 清空个人记忆并重置索引
- AC-6 → `origin=system.personal_kb` 的 `WIDGET_SUBMIT` 前置短路，不进入 LLM
- AC-7 → 候选提交校验（`candidateId + conversationId + TTL`）+ 幂等防重放

---

## Definition of Done 对齐（Story 12.1）

1. AC-1~AC-7 对应实现与测试全部通过。
2. `WIDGET_SUBMIT(origin=system.personal_kb)` 前置短路与“无 LLM 调用”有自动化验证。
3. 幂等/防重放错误码链路（expired/invalid/already_finalized）有集成测试覆盖。
4. 关键日志事件可观测，且包含 `candidate.submit_blocked`。
