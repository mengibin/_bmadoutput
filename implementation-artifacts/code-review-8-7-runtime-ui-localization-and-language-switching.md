# Code Review: Story 8.7 – Runtime UI Localization & Language Switching

**Date:** 2026-07-13
**Reviewer:** AI Code Reviewer（BMAD Method）
**Story File:** `8-7-runtime-ui-localization-and-language-switching.md`

---

## 范围说明

- 本次 review 仅审计 Story 8.7 的 Runtime 多语言实现、Main/后台错误本地化、设置持久化和自动化测试。
- 用户内容、LLM/Agent 输出、工作流/Package 正文、文件内容、协议字段和第三方原始诊断明确不属于翻译对象。

## Summary

| Metric | Value |
|:-------|:------|
| HIGH Issues | 0（2 项已修复） |
| MEDIUM Issues | 0（4 项已修复） |
| LOW Issues | 0 |
| Unit / Integration Tests | ✅ 62 files / 755 tests passed |
| Lint | ✅ passed |
| Production Build | ✅ `npm run build:ci` passed |
| Diff Hygiene | ✅ `git diff --check` passed |

## Resolved Findings

### H1：统一 IPC wrapper 将 listener 抛错转换成普通成功返回值

- **风险：** `dialog:confirm` 等返回 boolean 的调用在异常时会收到 truthy object，可能把失败误判为确认成功；`app:getVersion` 等 primitive contract 也会被破坏。
- **修复：** listener 抛错时改为抛出附带 `errorInfo`、`code` 和 `cause` 的 `LocalizedAppError`，保留 Promise rejection 语义与本地化消息。
- **证据：** `shared/i18n/errors.ts` 的 `localizeThrownError()`；`electron/main.ts` 的 `handleLocalizedIpc()`；对应单元测试。

### H2：结构化第三方错误没有完整保留 stack / stderr

- **风险：** 安装器、命令或 SDK 的原始排障信息在跨 IPC 后丢失，违反 AC-6 / AC-7。
- **修复：** diagnostic 同时保留 Error stack、普通对象 stack、stderr、message 和 details；用户提示仍只展示本地化摘要/原因/建议。
- **证据：** `shared/i18n/errors.ts` 与 `shared/i18n/errors.test.ts`。

### M1：`{ success: false, error: '' }` 会绕过未知错误本地化

- **风险：** 后台未提供 message 时 UI 可能显示空白，无法提供通用排障建议。
- **修复：** 显式 `success:false` / `ok:false` 一律进入错误包装；空 message 仅在 diagnostic 中保持为空，用户提示使用 `IPC_OPERATION_FAILED`。
- **证据：** `localizeIpcFailure()` 的 explicit-failure 分支和空错误测试。

### M2：已有错误仅保存本地化字符串，切换语言后无法更新

- **风险：** 语言切换时页面已有错误仍停留在旧语言，不满足 AC-7。
- **修复：** 新增 `resolveLocalizedIpcErrorMessage()`；文件浏览器和运行进度状态保存完整失败响应，在每次渲染时使用 effective locale 重新解析稳定 code/key。
- **证据：** `shared/i18n/errors.ts`、`src/hooks/useFileExplorer.ts`、`src/hooks/useWorkflowProgress.ts` 及双语重解析测试。

### M3：system 模式检测到语言变化后，macOS 原生菜单未刷新

- **风险：** Renderer 已更新但应用菜单仍使用旧语言，形成 Main/Renderer 不一致。
- **修复：** 增加 `app:refreshLocale` IPC；`languagechange` 触发 store refresh 时同步要求 Main 重建菜单。
- **证据：** `electron/main.ts`、`electron/preload.ts`、`src/App.tsx`、`src/stores/appStore.ts` 和 store 测试。

### M4：文件操作 prompt / confirm / alert 仍有硬编码英文

- **风险：** 中文模式下新建、重命名、删除、移动等核心 Files 流程出现混合语言。
- **修复：** 全部迁移到翻译资源，并通过统一错误解析器展示后台错误；会话创建/删除的应用自有文案同时迁移。
- **证据：** `src/hooks/useFileExplorer.ts`、`src/hooks/useConversationWorkspace.ts`、`src/pages/WorksPage/conversationDeleteDialog.ts` 和资源 parity 测试。

## Acceptance Criteria Verification

| AC | Status | Evidence |
|:---|:-------|:---------|
| AC-1 系统语言解析 | ✅ | `shared/i18n/locale.ts`，15 个 resolver / fallback 测试 |
| AC-2 设置入口 | ✅ | Settings Appearance 三选项与组件测试 |
| AC-3 即时切换与持久化 | ✅ | `appStore.saveUiLanguage()` optimistic apply / rollback，RuntimeStore 持久化测试 |
| AC-4 第一方 UI 覆盖 | ✅ | 核心页面、共享组件、对话框、tooltip、aria 文案迁移；资源扫描禁止源码中文硬编码 |
| AC-5 动态格式 | ✅ | plural 资源、Intl formatter、根节点 `lang` |
| AC-6 内容/协议不翻译 | ✅ | 翻译仅位于第一方展示层；用户/LLM/文件/工具内容和协议标识未改写 |
| AC-7 后台错误双语 | ✅ | Main 全 handler wrapper、共享 error envelope、known/unknown/diagnostic/relocalize 测试 |
| AC-8 缺失资源回退 | ✅ | `fallbackLng: en-US`、安全 missing-key handler 和测试 |
| AC-9 自动化验证 | ✅ | 62 个测试文件、755 个测试；lint/build 全通过 |

## Residual Notes

- 生产构建仍报告项目既有的 `jspreadsheet-ce` eval、bundle 体积以及 `mcpInstallService` 静态/动态混合导入警告；这些不由 Story 8.7 引入，也不影响本次验收。
- SSR 代表性渲染测试会输出 React Router / `useLayoutEffect` 的既有测试环境 warning；测试结果通过，桌面 Renderer 不使用 SSR。

## Final Decision

**APPROVED / DONE**：全部 HIGH 与 MEDIUM finding 已修复，Story 8.7 的验收标准和质量门禁均通过。
