# Epic Retrospective: Epic 14 Runtime SDK Wiki and Trusted SDK Tool Governance

**Date:** 2026-05-15
**Epic:** `Epic 14`
**Scope:** `14.1 ~ 14.6`

## 1. Objective

回顾 Epic 14 的交付质量、架构边界和后续 SDK 集成风险，把 SDK Wiki Pack、SDK 知识规划、trusted MCP governance 与 SAM golden path 的经验固化为后续 SDK 接入规则。

## 2. Status Snapshot

- `14.1` SDK Wiki Pack Import and Registry: `done`
- `14.2` SDK Wiki Search, Read, Symbol, and Relation Module: `done`
- `14.3` SDK Wiki Pack Management UI and Remove: `done`
- `14.4` SDK API Usage Planning with Source References: `done`
- `14.5` Trusted SDK Tool Governance and Audit: `done`
- `14.6` SAM Golden Path and Generic SDK Adapter Contract: `done`

## 3. What Was Delivered

1. 建立 RuntimeStore 下独立的 SDK Wiki Pack 存储、registry、事务导入和删除生命周期。
2. 对齐 `sdk-wiki-builder` 的 package/page/hash 规则，明确 README、`wiki/index.md`、`wiki/log.md` 不属于 page。
3. 增加内部 `sdk_wiki.*` 能力：list/search/read/resolve/expand/plan，支持主 LLM 按需读取 SDK 知识。
4. 在 Settings 中提供 SDK Wiki Pack 的导入、列表、错误展示和删除入口。
5. 增加 `sdk_wiki.plan_api_usage`，返回 source-referenced API 使用计划和 `missingInformation`。
6. 将 Story 14.5 从 Runtime 安全确认门禁重构为 trusted MCP governance：Runtime 只做 metadata observation 和 audit，不再做 SDK risk 用户确认。
7. 增加 SAM pressure-load golden path，验证 Pressure、Surface、StaticStep 的来源引用计划。
8. 沉淀 Generic SDK Adapter Contract，约束未来 SDK 接入不硬编码 SAM。

## 4. What Worked Well

1. 严格串行 BMAD 拆分有效降低了返工：先导入/注册，再查询/规划，再治理审计，最后 golden path。
2. 14.3 提前补上 Settings 导入/删除入口，避免 SDK Wiki 能力停留在开发者 IPC 层。
3. Builder package/page 规则被及时写入 story 和 importer 校验，修复了 README/index/log 被误当 page 的风险。
4. `SdkWikiService` 保持 deterministic retrieval scaffold，没有在 SDK service 内隐藏 LLM 调用，边界清晰。
5. 14.5 的安全边界调整是正确的：执行安全应由 MCP/集成软件保证，Runtime 重复确认会破坏 Agent 自主执行路径。
6. 14.6 用测试和 contract 验证了通用路径，而不是增加 SAM 专用生产分支。

## 5. Gaps / Risks

1. 当前 SAM golden path 使用 fixture/mock adapter，不等同于真实 SAM MCP 端到端运行。
2. `plan_api_usage` 仍是 retrieval-backed scaffold，复杂工程任务的排序、分步粒度和 API dependency 质量需要真实 SDK Pack 继续校准。
3. Runtime 现在信任 MCP 暴露的工具安全性；因此 MCP 工具治理、权限、路径约束、求解资源限制必须在集成软件侧有明确实现和测试。
4. SDK Wiki Pack schema 和 builder 输出还会演化，需要保持 importer 与 builder contract 同步。
5. 多 SDK、多版本同时安装后的默认选择策略仍偏 MVP，后续项目级默认 SDK 选择可能需要单独 story。
6. UI 目前在 Settings 管理 SDK Wiki Pack；如果安装包数量增多，可能需要独立 SDK Knowledge 页面。

## 6. Decisions Carried Forward

1. SDK 语义理解和 API planning 归 Runtime 主 LLM 与 `sdk_wiki.*`，MCP/Bridge 不做隐藏 LLM reasoning。
2. SDK execution safety 归 MCP/集成软件；Runtime 不做 SDK risk confirmation token，不因 `solve/destructive/file_write` metadata 拦截执行。
3. Runtime effective tool policy 仍是 Runtime 层唯一工具可见性/可执行性边界；SDK governance metadata 不能启用被禁用的工具。
4. SDK Wiki Pack 必须遵守 builder page discovery：排除任意 `README.md`、`wiki/index.md`、`wiki/log.md`。
5. API 建议和计划步骤必须有 source refs；缺失知识应返回 `missingInformation`，不能编造 API。
6. Future SDK integrations must use the generic adapter contract and avoid Runtime branches on specific SDK ids such as `sam`.

## 7. Action Items

1. 真实 SAM MCP 集成时，按 `sdk-adapter-contract-14-6.md` 增加 MCP-side safety tests。
2. 为真实 SAM SDK Wiki Pack 跑一次完整 import/list/search/read/plan 验收，替代 fixture-only confidence。
3. 根据真实 SDK Pack 质量，补充 `plan_api_usage` 排序和 dependency expansion 的回归样例。
4. 后续需要项目级 SDK 默认选择时，新建独立 story，不混入 Epic 14 已完成范围。
5. 保持 14.5 的 trusted MCP 边界不回退；如需要用户确认，应由 MCP 工具或上层 workflow 明确设计，不作为 Runtime SDK risk 默认行为。

## 8. Evidence Links

- PRD: `_bmad-output/prd-runtime-sdk-wiki-and-tool-risk.md`
- Epic: `_bmad-output/epics-runtime-sdk-wiki-and-tool-risk.md`
- Architecture: `_bmad-output/architecture/runtime-sdk-wiki-and-tool-risk-architecture.md`
- Development Plan: `_bmad-output/implementation-artifacts/epic-14-runtime-sdk-wiki-development-plan.md`
- Story 14.1: `_bmad-output/implementation-artifacts/14-1-sdk-wiki-pack-import-and-registry.md`
- Story 14.2: `_bmad-output/implementation-artifacts/14-2-sdk-wiki-search-read-symbol-and-relation-module.md`
- Story 14.3: `_bmad-output/implementation-artifacts/14-3-sdk-wiki-pack-management-ui-and-remove.md`
- Story 14.4: `_bmad-output/implementation-artifacts/14-4-sdk-api-usage-planning-with-source-references.md`
- Story 14.5: `_bmad-output/implementation-artifacts/14-5-sdk-tool-risk-confirmation-and-audit-gate.md`
- Story 14.6: `_bmad-output/implementation-artifacts/14-6-sam-golden-path-and-generic-sdk-adapter-contract.md`
- Adapter Contract: `_bmad-output/implementation-artifacts/sdk-adapter-contract-14-6.md`
- Sprint Status: `_bmad-output/implementation-artifacts/sprint-status.yaml`
