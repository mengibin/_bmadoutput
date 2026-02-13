# Epic Retrospective: BAI-2 Prompt Composer + Context + Tool Loop

**Date:** 2026-02-11  
**Epic:** `BAI-2`  
**Scope:** `BAI-2.1 ~ BAI-2.6`

## 1. Objective

回顾 Epic BAI-2 的交付质量与风险，把可复用决策沉淀到后续 Epic（特别是 BAI-3 changeSet/validate/apply）。

## 2. Status Snapshot

- `BAI-2.1` Build Prompt Composer Layering Engine: `done`
- `BAI-2.2` Build Domain Persona Resolver: `done`
- `BAI-2.3` Build Initial Context Envelope: `done`
- `BAI-2.4` Implement Semantic Read Tools: `done`
- `BAI-2.5` Implement ToolCall Loop Controller: `done`
- `BAI-2.6` Context Budget + Trimming Policy: `done`

## 3. What Was Delivered

1. 建立分层系统指令组装能力（Identity/Domain/Mode/Policy/Context）。
2. 支持领域 persona 自动解析与注入，替代固定 Builder 助手身份。
3. 建立初始上下文信封（target snapshot + dependency digest + revision info）。
4. 落地 `builder.*.read` 语义化读取工具，支撑 ToolCall 循环取证。
5. 实现工具循环控制（终止条件、repair loop、错误分类）。
6. 增加上下文预算与裁剪策略，保障 token 成本可控且目标对象完整。

## 4. What Worked Well

1. 读写边界思路清晰：读取能力开放，写入继续收敛到后续 `changeSet` 通道。
2. Prompt Composer 与 Tool Loop 解耦，便于单测和未来 provider 扩展。
3. 裁剪策略具备报告输出（`trimReport`），可观测性比初版更好。

## 5. Gaps / Risks

1. Story 已全部 `done`，但 provider 差异下的超时/限流韧性仍需在 BAI-5 继续加强。
2. 可观测性需要进入产品化阶段（统一审计事件、指标看板、报警阈值）。
3. 并发原子锁仍延期，当前保持 single-writer MVP 假设。

## 6. Decisions Carried Forward to BAI-3

1. 写入必须走统一 contract：`changeSet -> validate -> confirm -> apply`。
2. `target_snapshot` 是核心上下文，后续所有校验与 apply 以其为主轴。
3. 对象 revision 检查在 apply 前强制执行，不做静默覆盖。
4. `AI_BAD_RESPONSE`、`AI_CONTEXT_BUDGET_EXCEEDED` 等错误码沿用统一规范。

## 7. Action Items

1. 将 BAI-2 固化语义写入架构文档与 sprint 状态（已完成）。
2. 下一阶段聚焦 BAI-5：安全、审计、可观测与灰度上线。
3. 保持 BAI-2 不再扩 scope，仅做缺陷修复类维护。

## 8. Evidence Links

- Epic: `_bmad-output/epics-builder-ai.md`  
- Architecture: `_bmad-output/architecture/builder-ai-llm-interaction-architecture.md`  
- Tech Spec (BAI-2): `_bmad-output/implementation-artifacts/tech-spec-bai-2-prompt-composer-context-tool-loop.md`  
- Sprint Status: `_bmad-output/implementation-artifacts/sprint-status-builder-ai.yaml`
