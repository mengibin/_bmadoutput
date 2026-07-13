# Story 8.7: Runtime UI Localization & Language Switching

Status: done

## 概述

为 CrewAgent Runtime 建立统一的端到端国际化能力。当前只支持简体中文（`zh-CN`）和英文（`en-US`）两种语言；设置提供“跟随系统、简体中文、English”三种选择模式，用户可即时切换，选择结果跨启动持久化。

除第一方界面外，Electron Main 和 Runtime 后台服务产生的应用自有、用户可见错误摘要、原因及处理建议也必须支持中英文。本 Story 不自动翻译用户数据、工作流内容、Agent/LLM 输出、文件内容、第三方原始输出或协议数据。

---

## 用户故事

As a **Consumer**,
I want to switch the CrewAgent Runtime interface language,
So that I can use the application in my preferred language without changing my operating system language.

---

## 背景与现状

- Runtime 当前存在中英文硬编码界面文本，尚无统一 i18n 资源和翻译入口。
- `workflowGreeting` 仅根据浏览器语言选择中英文问候，不能代表全局界面语言。
- `RuntimeSettings` 已持久化主题等外观偏好，适合增加界面语言设置。
- Electron Main、IPC handler 和后台服务目前存在直接返回英文错误字符串的路径，无法随界面语言一致切换。
- 多语言切换必须与 LLM 回复语言、用户知识库中的语言偏好解耦。

> 设计文档：`_bmad-output/implementation-artifacts/design-8-7-runtime-ui-localization-and-language-switching.md`

---

## 验收标准

### AC-1：首次启动与系统语言解析

**Given** Runtime 没有已保存的界面语言偏好
**When** 应用首次启动
**Then** 默认使用 `system` 模式
**And** `zh`、`zh-CN`、`zh-SG` 等受支持的中文系统区域解析为 `zh-CN`
**And** `en` 系列及其他暂不支持的系统区域回退为 `en-US`

### AC-2：设置入口与可选语言

**Given** 用户进入 Settings → Appearance
**When** 查看 Interface Language 设置
**Then** 可选择：
  - `Follow System`
  - `简体中文`
  - `English`
**And** 当前保存的选择具有明确选中状态

### AC-3：即时切换与持久化

**Given** 用户选择与当前不同的界面语言
**When** 新选择生效
**Then** 当前已挂载的第一方 Runtime UI 无需重启即可更新
**And** 当前路由、项目、会话、表单输入和未保存编辑状态不丢失
**And** `uiLanguage` 写入 Runtime 设置
**And** 下次启动在首个有意义界面渲染前恢复语言，避免先显示错误语言再闪烁切换

### AC-4：第一方界面覆盖范围

**Given** 某个受支持语言已生效
**When** 用户访问 Runtime 核心界面
**Then** 以下第一方内容使用所选语言：
  - Start、Works/Chat、Runs、Files、Settings 等页面
  - Sidebar、顶部标签、状态栏和导航项
  - 标题、按钮、表单标签、占位文本、帮助文本和空状态
  - 应用自有的确认对话框、Toast、加载/成功/失败状态和校验消息
  - Tooltip、`aria-label`、可访问名称和应用自有菜单文本
**And** 不允许以新硬编码字符串绕过统一翻译入口

### AC-5：动态文本与本地化格式

**Given** 界面包含变量、计数或时间信息
**When** 文本被渲染
**Then** 使用安全插值，不通过字符串拼接构造句子
**And** 日期、时间和数字通过 `Intl` 或统一 formatter 按有效 locale 格式化
**And** 单复数/计数文本使用可扩展的 plural 规则
**And** 文档根节点的 `lang` 与有效 locale 一致

### AC-6：内容与协议不被翻译

**Given** 用户切换界面语言
**When** Runtime 展示业务或用户内容
**Then** 下列内容保持原样：
  - 用户消息与 LLM/Agent 回复
  - Workflow、Agent、Package 的作者自定义名称与正文
  - Artifact、文件、代码、Markdown 和执行日志原文
  - Tool 参数/结果、MCP 内容、第三方 `stderr`、堆栈和原始诊断详情
**And** `uiLanguage` 不注入 LLM 提示词，不改变模型回复语言
**And** IPC channel、持久化 enum、工具名、审计字段等内部标识保持稳定英文标识

### AC-7：Main/后台错误信息端到端双语化

**Given** Electron Main、IPC handler 或 Runtime 后台服务发生已知应用错误
**When** 错误被创建、跨 IPC 返回或向用户展示
**Then** 面向用户的错误摘要、原因和可执行处理建议均提供 `zh-CN` 与 `en-US` 文案
**And** 按当前有效界面语言返回或解析，不允许后台只提供固定英文用户消息
**And** 使用共享的稳定 `errorCode` / `messageKey`、插值参数和可选 `actionKey` 表达错误
**And** Renderer 切换语言后，可根据稳定 key 重新解析已有错误的用户可见文本
**And** 技术诊断详情、堆栈和第三方原始输出保留原文，用于展开查看或日志排查
**And** 不依赖匹配英文错误句子决定翻译结果

**Given** 后台收到暂不识别的第三方或系统错误
**When** 该错误需要展示给用户
**Then** 使用当前语言的通用错误摘要与排障提示包装原始错误
**And** 原始错误只作为诊断详情保留，不作为唯一的用户提示

### AC-8：缺失翻译与资源异常回退

**Given** 某个翻译 key 缺失或 locale 资源加载失败
**When** 相关界面渲染
**Then** 使用 `en-US` 文本回退
**And** 应用不崩溃
**And** 生产界面不显示原始翻译 key
**And** 开发/测试环境能报告缺失 key，便于修复

### AC-9：自动化验证

**Given** 多语言功能已实现
**When** 单元测试和 CI 检查运行
**Then** 至少覆盖：
  - 系统 locale 到有效 locale 的解析与回退
  - 旧设置无 `uiLanguage` 时的兼容迁移
  - 用户选择的保存、恢复和即时切换
  - 中英文资源 key 集合一致性
  - 插值、计数及日期/数字 formatter
  - Settings、Sidebar、Start 和 Works/Chat 的代表性中英文渲染
  - Main/后台已知错误在 `zh-CN` 与 `en-US` 下的错误 envelope 和文案
  - 未知后台错误的本地化包装及原始诊断保留
  - 缺失 key 与资源异常回退

---

## 功能范围

### 首期包含

- 仅 `zh-CN` 与 `en-US` 两套第一方界面及应用后台错误资源。
- `system`、`zh-CN`、`en-US` 三种用户设置值。
- Runtime Renderer 全局语言上下文和即时切换。
- Electron Main/后台服务能够获得当前有效 locale，并在语言切换后同步更新。
- 核心页面、共享组件、应用自有提示及可访问文本迁移。
- 统一的日期、时间、数字、插值与计数格式化方式。
- 已知应用错误的共享双语 catalog、结构化 error envelope 和本地化展示路径。

### 首期不包含

- 自动翻译用户、工作流、Package、Agent 或文件内容。
- 根据 UI 语言自动改变 LLM 回复语言。
- 翻译第三方 SDK/插件/MCP、操作系统或命令行工具原样返回的诊断内容；但必须用中英文应用提示进行包装。
- 强制改变操作系统原生文件选择器等系统 UI 的语言。
- `zh-TW`、日语等额外语言。

---

## 技术约束与设计关注点

### 1. 设置模型

建议增加稳定的偏好值与运行时解析值：

```typescript
type UiLanguagePreference = 'system' | 'zh-CN' | 'en-US'
type SupportedLocale = 'zh-CN' | 'en-US'

interface RuntimeSettings {
    // existing settings...
    uiLanguage: UiLanguagePreference
}
```

- `uiLanguage` 缺失时按 `system` 处理，保证旧设置向后兼容。
- UI 只消费解析后的 `SupportedLocale`，不要在各组件重复判断系统语言。
- 保存的是偏好值；系统语言变化后，`system` 模式应在下次启动或可检测到变化时重新解析。

### 2. i18n 资源组织

可采用成熟 React i18n 方案。Renderer 文案按领域拆分；Main/后台错误使用可被两侧复用的共享契约与双语 catalog：

```text
shared/i18n/
  locale.ts
  errors/
    types.ts
    catalog.en-US.ts
    catalog.zh-CN.ts
src/i18n/
  index.ts
  formatters.ts
  locales/
    en-US/
      common.ts
      navigation.ts
      settings.ts
      works.ts
      files.ts
      errors.ts
    zh-CN/
      common.ts
      navigation.ts
      settings.ts
      works.ts
      files.ts
      errors.ts
```

- `en-US` 是资源回退基准。
- key 使用稳定语义名称，不直接使用英文原句作为 key。
- 不允许 HTML 注入；富文本翻译需使用框架安全组件或受控 token。
- locale 资源不得包含密钥或运行时业务数据。

### 3. 启动顺序

- 在 App 首个有意义渲染前读取已持久化的 `uiLanguage` 并初始化 locale。
- 初始化期间可复用现有启动/加载界面，但不得先渲染完整错误语言页面。
- 切换语言不触发 Router 重建、Store 清空或 Project/Conversation reload。

### 4. 主进程与 Renderer 边界

- 有效 locale 必须通过共享设置读取或明确的 IPC 同步到 Electron Main/后台服务；切换完成后新错误立即使用新语言。
- Main/后台应用错误使用共享结构，例如 `{ errorCode, messageKey, params, actionKey?, diagnostic? }`，不得只返回硬编码英文 `error` 字符串。
- 应用自有后台错误的摘要、原因和建议在共享 catalog 中同时提供 `zh-CN` / `en-US`，Main 与 Renderer 可复用同一 key 和参数契约。
- 若 IPC 为兼容现有调用方保留 `error: string`，该字段也必须按有效 locale 生成，同时保留结构化 error envelope 作为规范接口。
- Renderer 优先根据 key 和当前 locale 解析，从而支持已展示错误在即时切换语言后重新渲染。
- 未知错误使用当前语言的通用摘要和处理建议包装；原始 message、堆栈和第三方 `stderr` 仅放入诊断详情与日志。
- Electron/OS 原生对话框若由应用提供 `title`、`message`、`buttonLabel`，应用自有字段应从当前 locale 获取；系统自有固定文本不在控制范围内。

### 5. 迁移原则

- 优先迁移共享导航、Settings、Start、Works/Chat，再覆盖 Files、Runs 和长尾组件。
- 以用户可见文本为边界，不改协议、数据库字段、工具名和日志原始数据。
- 盘点 Electron Main、IPC handler、RuntimeStore 和后台 service 的用户可见错误；已知错误迁移到共享 error catalog。
- `workflowGreeting` 应改用全局有效 locale，停止单独读取 `navigator.language` 形成第二套语言决策。

---

## Tasks / Subtasks

- [x] 1) 建立 i18n 基础设施与 locale 解析（AC: 1,5,8）
  - [x] 定义 `UiLanguagePreference`、`SupportedLocale` 和唯一 locale resolver
  - [x] 建立 `en-US` / `zh-CN` 分域资源、fallback 和安全插值
  - [x] 在 `shared/i18n` 建立可供 Main 与 Renderer 复用的错误类型及双语 catalog
  - [x] 提供统一日期、时间、数字及计数 formatter
  - [x] 同步更新根节点 `lang`

- [x] 2) 扩展 Runtime 设置和启动恢复（AC: 1,3）
  - [x] `RuntimeSettings`、`defaultSettings` 和 Renderer store 增加 `uiLanguage`
  - [x] 兼容无该字段的旧 `settings.json`
  - [x] 在首个有意义渲染前应用保存或系统解析的 locale
  - [x] 初始化及切换时将有效 locale 同步到 Electron Main/后台服务
  - [x] 验证切换不清空当前页面与编辑状态

- [x] 3) 增加 Settings → Appearance 语言选择器（AC: 2,3）
  - [x] 展示 Follow System / 简体中文 / English
  - [x] 即时应用并持久化选择
  - [x] 提供键盘操作、可访问标签和明确选中状态

- [x] 4) 迁移第一方核心 UI 文本（AC: 4,5,6）
  - [x] AppShell、Sidebar、StatusBar 与共享 Dialog/Toast
  - [x] Start、Works/Chat、Runs、Files、Settings 页面
  - [x] 标题、按钮、表单、空状态、校验、Tooltip 与 aria 文本
  - [x] 将 `workflowGreeting` 统一到全局有效 locale

- [x] 5) 建立 Main/后台错误端到端双语路径（AC: 7,8）
  - [x] 盘点 Main、IPC handler、RuntimeStore 和后台 service 的用户可见错误
  - [x] 为可预期应用错误定义稳定 error code/key、参数及可选 action key
  - [x] 为每个已知错误同时提供 `zh-CN` / `en-US` 摘要、原因和处理建议
  - [x] IPC 返回结构化 error envelope；兼容字符串也按有效 locale 生成
  - [x] Renderer 按当前 locale 解析，并保留可展开的原始诊断详情
  - [x] 未知错误使用双语通用包装，不通过英文句子匹配

- [x] 6) 自动化测试与翻译完整性检查（AC: 9）
  - [x] locale resolver、偏好持久化、即时切换和 formatter 单测
  - [x] 中英文 key 对齐和缺失 key 回退检查
  - [x] Settings、Sidebar、Start、Works/Chat 代表性组件测试
  - [x] Main/后台已知错误的中英文 envelope、插值和 IPC 集成测试
  - [x] 未知错误本地化包装且原始诊断不丢失的测试
  - [x] 回归验证用户内容、协议标识和 LLM 回复语言未被修改

---

## 依赖与风险

| 项目 | 说明 | 缓解措施 |
|:-----|:-----|:---------|
| 硬编码文本数量 | Runtime 页面和组件较多，容易遗漏 | 按共享组件和页面分批迁移，并增加 key 完整性检查 |
| 启动闪烁 | 异步读取设置可能先显示默认语言 | 在首个业务 UI 渲染前完成 locale 初始化 |
| 状态丢失 | 通过重载实现切换会丢编辑状态 | 使用上下文更新和 React 重渲染，禁止整页 reload |
| 错误消息耦合 | Main/后台现有接口可能只有英文 `error` 字符串 | 引入共享 error envelope；兼容字段同步本地化，逐步迁移调用方 |
| 语言状态不同步 | Renderer 已切换但后台仍使用旧 locale | 设置更新与 Main locale 更新使用同一保存流程并增加 IPC 集成测试 |
| 内容误翻译 | UI 文本与用户/协议内容边界不清 | 只翻译第一方静态展示层，明确负面测试 |

---

## Definition of Done

- 所有验收标准通过。
- `zh-CN`、`en-US` 资源 key 校验通过，核心界面无明显混合语言或 raw key。
- Main/后台应用自有错误的摘要、原因和处理建议均有中英文资源，且 IPC 集成测试通过。
- 未知后台错误使用本地化通用包装，原始诊断、堆栈和第三方输出未丢失。
- 切换语言不需要重启，不丢当前工作状态，重启后偏好正确恢复。
- 用户/工作流/LLM/文件内容和内部协议标识未因 UI 语言变化而改变。
- 相关单元测试与现有 Runtime 回归测试通过。
- Story 实现完成后先进入 `review`；独立代码审查通过且 HIGH / MEDIUM 问题清零后更新为 `done`。

---

## Dev Agent Record

### Agent Model Used

GPT-5（Codex）

### Debug Log References

- `npm test -- --maxWorkers=1`：62 个测试文件、755 个测试全部通过。
- `npm run lint`：通过，0 warning / 0 error。
- `npm run build:ci`：TypeScript、Renderer、Electron Main 与 Preload 生产构建通过。
- `git diff --check`：通过；仅 Git 提示两个既有 CRLF 文件后续写入会转换为 LF。

### Completion Notes List

1. 引入 `i18next` / `react-i18next`，交付 `zh-CN`、`en-US` 两套资源和 `system` / `zh-CN` / `en-US` 三种偏好值；旧设置自动按 `system` 迁移。
2. App 在首个业务界面渲染前应用保存语言；切换即时更新已挂载 UI、`document.lang` 与 macOS 原生菜单，不重载页面或清空工作状态。
3. Start、Works/Chat、Runs、Files、Knowledge、Settings、Workspace、共享布局、对话框、文件预览、交互组件及可访问文本已迁移到统一翻译入口。
4. Electron Main 的 IPC handler 统一接入本地化失败包装；共享错误契约提供稳定 code/key、摘要、原因、建议和原始 diagnostic。
5. Renderer 的文件/进度错误保留结构化 envelope，可在语言切换后重新解析已有错误；第三方 message、stack、stderr 与 details 保持原文。
6. BMAD 对抗式 review 发现并修复 6 项问题；最终 HIGH / MEDIUM 未解决项均为 0。详见 `code-review-8-7-runtime-ui-localization-and-language-switching.md`。

### File List

- `crewagent-runtime/package.json`
- `crewagent-runtime/package-lock.json`
- `crewagent-runtime/vitest.config.ts`
- `crewagent-runtime/vitest.setup.ts`（新增）
- `crewagent-runtime/shared/i18n/*`（新增：locale、Main 菜单、错误 catalog/契约与测试）
- `crewagent-runtime/src/i18n/*`（新增：初始化、formatter、双语资源与完整性/渲染测试）
- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/preload.ts`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/stores/runtimeStore.test.ts`
- `crewagent-runtime/src/main.tsx`
- `crewagent-runtime/src/App.tsx`
- `crewagent-runtime/src/vite-env.d.ts`
- `crewagent-runtime/src/stores/appStore.ts`
- `crewagent-runtime/src/stores/appStore.test.ts`
- `crewagent-runtime/src/hooks/useConversationWorkspace.ts`
- `crewagent-runtime/src/hooks/useFileExplorer.ts`
- `crewagent-runtime/src/hooks/useWorkflowProgress.ts`
- `crewagent-runtime/src/components/**`（第一方共享组件与文件/日志/工作流界面迁移）
- `crewagent-runtime/src/pages/**`（Start、Works、Runs、Files、Knowledge、Settings、Workspace 等页面与代表性测试迁移）
- `_bmad-output/epics.md`
- `_bmad-output/implementation-artifacts/8-7-runtime-ui-localization-and-language-switching.md`
- `_bmad-output/implementation-artifacts/design-8-7-runtime-ui-localization-and-language-switching.md`
- `_bmad-output/implementation-artifacts/code-review-8-7-runtime-ui-localization-and-language-switching.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`

## Change Log

- 2026-07-13：完成 Runtime 简体中文/英文界面与后台错误国际化、即时切换、设置持久化、完整自动化验证和 BMAD 对抗式代码审查；Story 状态更新为 `done`。
