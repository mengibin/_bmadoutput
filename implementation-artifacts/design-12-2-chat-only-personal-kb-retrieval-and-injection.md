# Design: Chat-Only Personal KB Retrieval & Injection

**Story:** `12-2-chat-only-personal-kb-retrieval-and-injection.md`  
**设计原则:** Chat 注入单点门控、Agent/Run 强隔离、双层注入（核心固定 + 长尾检索）、对用户完全透明（仅日志可见）

---

## 设计目标

1. 仅在 `chat` 模式回答前注入个人知识。
2. 固定层注入：始终注入 `SOUL.md` + `USER.md` + `MEMORY.md#Pinned`（如存在）。
3. 检索层注入：按 query 从 `MEMORY.md#General`（兼容 legacy non-pinned 长尾条目）与最近有有效内容的 daily memory 中 `top-k` 注入。
4. `agent/run` 模式严格跳过个人知识检索与注入，并产生日志可观测 `skipped by mode` 信号。
5. 注入状态对终端用户透明，不新增 chat/agent/run 前端提示；可选内部诊断字段不作为 UI 依赖。
6. 注入层必须消费 Story 12.1 mutation 元数据（`memoryId/status`），并遵循“长期记忆沉淀到 SOUL/USER/Pinned”的写入约束。

---

## 非目标（本 Story 不做）

1. 不改变 Story 12.1 的“跨模式显式入库”链路（候选/确认/提交保持不变）。
2. 不提供个人知识可视化编辑器（后续 Story 12.6）。
3. 不引入云端向量库或 MCP-RAG（企业级另行实现）。
4. 不在 `agent/run` 中偷偷读取个人知识（默认 deny）。
5. 不在 Renderer 展示 personal memory injected/skipped 的状态提示。
6. 不将 `MEMORY.md` 全量作为固定层注入（仅 `Pinned` 进入固定层）。

---

## 设计决策

1. **双层注入，不走全量 dump**：固定层承载长期稳定规则，检索层承载场景化长尾；避免只靠检索导致关键规则漏注入。  
2. **检索不走 LLM**：检索/排序用本地规则（关键词重叠 + 文件权重 + 时间衰减），只保留 12.1 中“写入路由走 LLM”。  
3. **注入走 system message**：在 `buildChatContextMessages` 增加 `extraSystemMessages`，把 personal context 作为额外系统段落插入；不修改对话历史文件。  
4. **模式守卫放在 Main 单点**：`callAgentChat` 统一处理 `chat allow` / `agent deny`，避免分散到多个调用方导致漂移。  
5. **OpenClaw 风格吸收**：沿用“核心文件固定注入 + daily 按需检索”思想，在 Runtime 采用 `SOUL/USER/Pinned + tail retrieval` 落地。
6. **写读契约分离**：12.1 负责决定 `Pinned vs General vs daily`；12.2 只读取，不反推。

---

## 当前 Cross-Story 状态（2026-03-11）

1. 12.1 写入分类契约已落地：`targetSection` 已贯穿 `route -> candidate -> commit -> store mutation`。
2. `MEMORY.md` 当前可稳定区分：
   - `Pinned`：固定层
   - `General`：检索层
   - `memory/YYYY-MM-DD.md`：daily 检索层
3. `recent non-empty daily` 窗口已落地：只有存在 active entry 的 daily 文件才会进入 recent retrieval window。
4. 12.2 当前已无未关闭 cross-story blocker，可进入 review。

---

## Code Review Follow-ups (Must Fix)

1. **固定层不得直接 raw dump `SOUL.md` / `USER.md`**：  
固定层必须优先消费 active entry 解析结果，而不是直接整文件读取。只有当文件中不存在结构化记忆块时，才允许 legacy 纯文本 fallback。
2. **`MEMORY.md#Pinned` 必须有真实写入契约**：  
若 Story 12.1 未把长期记忆路由到 `Pinned` 区块，则 Story 12.2 不能宣称 fixed pinned 已完整实现。读取契约与写入契约必须一致。
3. **中文 section 标题匹配不能依赖 `\\b`**：  
`关键记忆`、`长期偏好`、`固定规则` 等标题必须可命中 fixed layer。应采用标题归一化 + substring / token 规则，而非 JS 英文词边界。
4. **recent daily 必须是 recent non-empty daily**：  
自动初始化但没有 active entry 的空白 daily 文件，不得占用 retrieval window。

---

## 改动范围

| 文件 | 改动类型 | 说明 |
|------|----------|------|
| `crewagent-runtime/electron/main.ts` | MODIFY | `callAgentChat` 增加 personal KB 注入决策、模式短路、usage/log 输出 |
| `crewagent-runtime/electron/services/personalKbService.ts` | MODIFY | 新增 chat 检索/排序/预算裁剪/注入块构建能力 |
| `crewagent-runtime/electron/stores/runtimeStore.ts` | MODIFY | 新增 personal KB 读取接口（核心文件 + 最近 daily） |
| `crewagent-runtime/electron/services/chatContextBuilder.ts` | MODIFY | 支持 `extraSystemMessages` 注入并维持现有压缩/历史逻辑 |
| `crewagent-runtime/electron/services/chatContextBuilder.test.ts` | MODIFY | 新增 personal system 注入顺序与预算回归测试 |
| `crewagent-runtime/electron/services/personalKbService.test.ts` | MODIFY | 新增检索命中、未命中、预算截断、模式跳过测试 |
| `crewagent-runtime/electron/services/agentSessionContract.ts` | MODIFY | 收敛 `usage.personalKb` 结构（chat/agent dispatch） |

---

## 数据结构与契约

### 1) 检索结果（Main 内部）

```ts
type PersonalKbSnippet = {
  layer: 'fixed' | 'retrieval'
  file: 'SOUL.md' | 'USER.md' | 'MEMORY.md' | `memory/${string}.md`
  text: string
  memoryId?: string
  status: 'active'
  score: number
  updatedAt?: string
}

type PersonalKbInjectionResult = {
  mode: 'chat' | 'agent' | 'run'
  decision: 'injected' | 'skipped' | 'miss'
  reason:
    | 'mode_guard'
    | 'no_query_match'
    | 'kb_unavailable'
    | 'empty_kb'
    | 'budget_truncated'
    | 'injected'
  fixedSnippets: PersonalKbSnippet[]
  retrievalSnippets: PersonalKbSnippet[]
  usedChars: number
  fixedUsedChars: number
  retrievalUsedChars: number
  budgetChars: number
  fixedBudgetChars: number
  retrievalBudgetChars: number
}
```

### 2) 内部诊断字段（可选，不要求 Renderer 消费）

```ts
type PersonalKbUsage = {
  mode: 'chat' | 'agent' | 'run'
  decision: 'injected' | 'skipped' | 'miss'
  reason: string
  fixedCount: number
  retrievalCount: number
  sourceFiles: string[]
  usedChars: number
  budgetChars: number
}
```

说明（内部）：
- `chat` 命中注入时：`decision=injected`；
- `chat` 固定层有内容但检索层 miss 时：仍可 `decision=injected`（`retrievalCount=0`）；
- `chat` 固定层和检索层都为空时：`decision=miss`；
- `agent/run`：固定 `decision=skipped` + `reason=mode_guard`。
- 该结构若保留，仅用于日志/测试与开发调试，不要求在 UI 层渲染。

---

## 核心流程

### A. Main 单点门控（AC-1/AC-2）

位置：`main.ts -> callAgentChat()`

1. 确定 `mode` 后，先调用 personal KB 注入决策：
   - `mode==='chat'`：先构建固定层，再构建检索层；
   - `mode==='agent'`：直接返回 `skipped(mode_guard)`。
2. 将决策结果记录到运行日志（`kb.personal.injection.hit|miss|skipped`）。
3. `usage.personalKb` 如保留，仅作内部调试字段，不作为产品可见能力。
4. 仅 `chat` 且 `decision=injected` 时，把注入块传入 `buildChatContextMessages`。

### B. 数据加载（AC-1）

`runtimeStore` 暴露只读接口（不暴露给普通用户）：

```ts
getPersonalKbRetrievalSources({ recentDailyDays: number }): Array<{
  path: string
  content: string
  updatedAt: string
}>
```

默认加载集合（按层划分）：
- 固定层：`SOUL.md`、`USER.md`、`MEMORY.md#Pinned`
- 检索层：`MEMORY.md#General`（及 legacy non-pinned sections）+ 最近 2 个有 active entry 的 `memory/YYYY-MM-DD.md`

### C. 固定层构建（AC-1）

`personalKbService` 新增 `buildFixedInjectionLayer()`：

1. `SOUL.md` / `USER.md` 必须优先通过 active entry 解析构建固定层，不得在存在结构化记忆块时直接整文件注入。
2. 仅当 `SOUL.md` / `USER.md` 仍为 legacy 纯文本文件、且未发现 `### [timestamp]` 结构化记忆块时，才允许全文 fallback 注入。
3. 从 `MEMORY.md` 中提取 `Pinned` 区块；该区块必须与写入契约一致，不能只读不写。
4. 固定层不做 query 检索，但要做字符预算裁剪（避免挤爆上下文）。
5. 固定层条目必须经过有效状态过滤（仅 `status=active`）。

补充约束：
- fixed layer 的目标不是“把文件内容塞给模型”，而是“把当前有效的长期记忆塞给模型”。
- fixed layer 输出中不得包含 `status/superseded/deleted/replaces*` 这类内部元数据行。

### D. 检索层构建与排序（AC-1）

`personalKbService` 新增 `buildRetrievalInjectionLayer(query)`：

1. 数据源：`MEMORY.md#General`（及 legacy non-pinned 长尾条目）+ 最近 non-empty daily memory。
2. 解析 Markdown 条目：
   - 优先匹配标准块 `### [timestamp] text`；
   - 无标准块时，用“非空段落回退”提取可检索文本（防止空库误判）。
3. 解析 mutation 元数据并计算有效状态：
   - 仅保留有效 `status=active` 条目；
   - `status=superseded/deleted` 作为 tombstone，屏蔽旧条目；
   - 同一 `memoryId` 仅保留最新一条 active。
4. 关键词切分（中英混合）：
   - 中文：按 2~4 字片段；
   - 英文/数字：按单词。
5. 评分：
   - `score = overlap * weight + fileBoost + recencyBoost`
   - `fileBoost`: `MEMORY(long-tail) > daily`
6. 取 `topK`（建议 6）并去重。
7. 按检索层预算裁剪，超限标记 `reason=budget_truncated`。

recent daily 规则补充：
- “recent” 以 daily 文件日期名倒序为准；
- 先过滤掉没有 active entry 的 daily 文件；
- 再取最近 `N` 个 non-empty daily 文件；
- 自动初始化生成的空白今日文件，不得进入候选集合。

### E. 注入格式（AC-1）

注入为额外 system message（不落对话历史），采用双段结构：

```md
## Personal Memory - Core (Always Applied)
- [SOUL.md] ...
- [USER.md] ...
- [MEMORY.md#Pinned] ...

## Personal Memory - Retrieved (Query Related)
- [MEMORY.md] ...
- [memory/YYYY-MM-DD.md] ...
```

规则：
1. 固定层在 chat 中优先注入。
2. 检索层作为补充注入；若未命中，可只保留固定层。

### F. Context Builder 集成（AC-1）

`chatContextBuilder.ts` 新增参数：

```ts
extraSystemMessages?: string[]
```

行为：
1. 先保持现有 `history + userInput + compression` 流程不变；
2. 在输出 `messages` 中，把 `extraSystemMessages` 插入到 system 前缀末尾（在第一条非 system 消息之前）。

### G. 模式隔离（AC-2）

`agentContextBuilder` 不改，不接 personal 注入参数。  
`run` 路径也不读取 personal KB；若需要调试提示，仅通过 `usage/log` 标记 skip。

### H. 可观测日志（AC-3）

新增日志事件：
- `kb.personal.injection.hit`
- `kb.personal.injection.miss`
- `kb.personal.injection.skipped`

建议日志字段：
- `mode`
- `decision`
- `reason`
- `fixedCount`
- `retrievalCount`
- `sourceFiles`
- `usedChars/budgetChars`

### I. 关键记忆沉淀规则对齐（AC-4）

双层注入稳定性的前提是写入侧分层明确（来自 Story 12.1）：

1. `SOUL.md`：长期行为原则与边界（必须始终生效）。
2. `USER.md`：用户身份与稳定偏好（语言、称呼、输出风格）。
3. `MEMORY.md#Pinned`：高价值长期事实或工作偏好（固定层）。
4. `MEMORY.md#General` + `memory/YYYY-MM-DD.md`：场景化与阶段性信息（检索层）。

约束：
1. “必须长期生效”内容不得只写入 daily。
2. daily 默认不进固定层，避免临时信息污染长期指令。
3. `MEMORY.md#Pinned` / `MEMORY.md#General` 必须是运行时真实可写、可读、可重建的结构，不接受“设计里有、写入链路没有”的状态。
4. 若 12.1 写入侧尚未支持 `Pinned vs General` 路由，则 12.2 在复审通过前必须把该能力列为 blocking dependency。

---

## Renderer 约束（No-op）

目标：对最终用户透明，不新增任何 personal injection 可见提示。

1. `WorksPage` 不新增 `Personal memory applied/skipped` 标签、toast 或调试文案。
2. `agent/run` 页面不新增 `skipped by mode` 提示区块。
3. personal injection 状态仅进入 runtime log（必要时可在开发调试面板查看日志）。

---

## 验证策略

### Unit

1. `personalKbService.test.ts`
   - 固定层始终注入当前有效的 `SOUL/USER`（`MEMORY#Pinned` 有则注入）
   - `SOUL.md` / `USER.md` 发生 `UPDATE/DELETE` 后，旧条目不会再次进入 fixed layer
   - 检索层 chat query 命中返回 retrieval `top-k`
   - 检索层未命中时，仍可只靠固定层返回 injected
   - 固定层与检索层同时为空时返回 miss
   - `deleted/superseded` 条目不参与注入
   - 同一 `memoryId` 仅保留最新 active 条目
   - 中文 section 标题（如 `关键记忆` / `长期偏好`）可被 fixed layer 命中
   - 空白今日 daily 文件不会占用 recent retrieval window
   - 预算超限时裁剪并标记 `budget_truncated`（分 fixed/retrieval）
   - mode=agent/run 时返回 skipped(mode_guard)
2. `chatContextBuilder.test.ts`
   - `extraSystemMessages` 注入顺序正确
   - 不破坏 `USER_INPUT` 包装和 compression 行为

### Integration

1. `chat:send`（chat）产生日志 `kb.personal.injection.hit|miss`，并可区分 `fixedCount/retrievalCount`
2. `agent:dispatch` / `runs:continue` 产生日志 `kb.personal.injection.skipped`
3. UI 无 personal injection 新增提示（回归检查）

### Manual E2E

1. 在 chat 里问与稳定偏好相关问题，确认 `SOUL/USER/Pinned` 固定生效。
2. 在 chat 里问与阶段性话题相关问题，确认来自 long-tail/daily 的检索补充生效。
3. 对已标记长期记忆，确认不会仅存在于 daily 且丢失固定注入。
4. 切换到 agent 模式发同类问题，确认无个人记忆提示且回答不受个人记忆注入影响。
5. run 模式继续执行，检查日志中存在 `kb.personal.injection.skipped`。

---

## 风险与回退

1. **风险：** 检索召回不稳（中英混合表达差异）。  
   **缓解：** 先 lexical 规则 + 文件权重；后续再在 12.6 引入可选 rerank。
2. **风险：** 固定层过长导致上下文预算被占满。  
   **缓解：** 固定层单独预算 + `Pinned` 限定，不允许 `MEMORY.md` 全量直注入。
3. **风险：** 长期记忆未沉淀到固定层，导致仅检索命中才生效。  
   **缓解：** 在 12.1 写入路由增加沉淀规则校验（SOUL/USER/Pinned）。
4. **风险：** agent/run 误注入。  
   **缓解：** Main 单点 mode guard + 单测覆盖 deny path。
5. **回退策略：** 出现回归时仅关闭 chat personal injection 开关（保留 12.1 写入能力）。

---

## AC 映射

1. **AC-1 Chat 双层注入**：A/B/C/D/E/F + Unit/Integration #1  
2. **AC-2 Agent/Run 强隔离**：A/G + Integration #2 + Manual #4/#5  
3. **AC-3 可观测日志**：H + Integration #3  
4. **AC-4 关键记忆沉淀规则**：I + Manual #1/#3
