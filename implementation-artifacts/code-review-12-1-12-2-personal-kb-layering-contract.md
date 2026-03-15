# Code Review: Story 12.1 / 12.2 – Personal KB Layering Contract

**Date:** 2026-03-11  
**Reviewer:** AI Code Reviewer (BMAD Method)  
**Scope:** `12-1-personal-kb-storage-and-candidate-commit` + `12-2-chat-only-personal-kb-retrieval-and-injection`

---

## Summary

| Metric | Value |
|--------|-------|
| HIGH Issues | 0 |
| MEDIUM Issues | 0 |
| LOW Issues | 0 |
| Review Outcome | Approved |

---

## Findings

### H1 — 12.1/12.2 的“写入分层契约”未统一（Resolved 2026-03-11）

**Problem**  
Story 12.2 的双层注入要求固定层读取 `SOUL.md` / `USER.md` / `MEMORY.md#Pinned`，检索层读取 `MEMORY.md` 长尾与 daily。  
该问题已在 2026-03-11 修复：12.1 现已把 `targetSection` 贯穿到 `route -> candidate -> commit -> store mutation`，并阻止 `MEMORY.md` 默认落到 `Pinned`。

**Impact**  
- 当前该问题已关闭，不再作为 open blocker。

**Required Fix**  
1. 12.1 已拥有明确的写入契约：
   - `SOUL.md` / `USER.md`：固定层核心记忆
   - `MEMORY.md#Pinned`：长期固定层记忆
   - `MEMORY.md#General`：长尾检索层记忆
   - `memory/YYYY-MM-DD.md`：daily 检索层记忆
2. 路由/候选/提交路径现已表达 `targetSection`，不再只返回 `targetFile`。
3. 12.2 继续只消费该契约，不负责反推。

---

### M1 — recent daily 检索窗口定义不清（Resolved 2026-03-11）

**Problem**  
当前实现与部分历史文档把自动初始化的“空白今日 daily 文件”也算进 recent daily window。  
这会占掉检索窗口名额，降低真实 daily 记忆的召回。

**Impact**  
- 当前该问题已关闭，不再作为 open blocker。

**Required Fix**  
1. 12.2 的检索层现已改为“最近 N 个有 active entry 的 daily 文件”，而不是“最近 N 个文件名”。
2. 自动初始化但无 active entry 的空白今日文件，不再进入 recent retrieval window。
3. 已补回归测试覆盖 recent non-empty daily 规则。

---

## Architecture Decision

为避免后续误解，Epic 12 MVP 统一采用以下约束：

1. KB 本地索引统一为 `index.json` 元数据 / chunk catalog。
2. KB MVP 不以 SQLite 作为当前 personal / project knowledge base 的设计前提。
3. 若未来确实需要 SQLite / FTS，应作为后续独立 story 引入，而不是保留在当前文档中造成语义漂移。

---

## Story Disposition

- Story 12.1: `ready-for-review`
  原因：`Pinned vs General vs daily` 写入契约已落地，并已补回归测试。
- Story 12.2: `ready-for-review`
  原因：recent non-empty daily 检索规则已落地，并已补回归测试。
