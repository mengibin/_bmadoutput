# Design: Runtime UI Localization & Language Switching

**Story:** `8-7-runtime-ui-localization-and-language-switching.md`
**Status:** implemented
**设计原则:** 仅中英文、单一语言决策、端到端错误契约、即时切换、诊断原文不丢失

---

## 1. 设计结论

Story 8.7 采用以下实现方案：

1. 使用 `i18next` + `react-i18next` 管理 Renderer 第一方界面文案。
2. 只提供 `zh-CN` 与 `en-US` 两套语言资源；`system` 是偏好模式，不是第三种 locale。
3. 在 `shared/i18n` 中定义 Main 与 Renderer 共用的 locale 类型、解析器、错误 envelope 和双语后台错误 catalog。
4. `RuntimeSettings.uiLanguage` 持久化用户偏好，`effectiveLocale` 由统一 resolver 计算。
5. App 初始化读取设置后、业务界面渲染前切换 i18next 语言；切换语言不 reload 页面。
6. 所有 IPC handler 通过统一注册包装器处理失败响应，使后台用户可见错误至少具备本地化摘要、建议和原始诊断。
7. 用户内容、LLM 输出、工作流/Package 内容、文件内容、工具协议和原始日志不翻译。

---

## 2. 当前实现审计

| 区域 | 当前状态 | 设计处理 |
|:-----|:---------|:---------|
| Renderer | 64 个 TSX 组件，存在大量中英文硬编码 | 按共享布局、核心页面、长尾组件分批迁移到 `t()` |
| Runtime 设置 | 已持久化 theme、skipSplash 等，无语言字段 | 增加向后兼容的 `uiLanguage` |
| 启动过程 | Store 初始化完成后才渲染业务 UI | 在 `initialize()` 中先应用 effective locale，避免完整页面闪烁 |
| `workflowGreeting` | 单独读取 `navigator.languages` | 改为接收全局 effective locale |
| Electron Main | 99 个 IPC handler | 统一使用本地化 handler wrapper |
| 后台错误 | 多处直接返回英文 `error` 字符串或 `{code,message}` | 规范化为共享 error envelope，保留兼容字符串 |
| 诊断信息 | 原始错误与用户提示混在一起 | 本地化用户摘要；原始 message/stack/stderr 放入 diagnostic |

---

## 3. 依赖与官方约束

- `react-i18next` 官方推荐 React Hooks 使用 `useTranslation()`，并通过 `i18n.changeLanguage()` 即时切换语言。
- `i18next` 明确支持 `fallbackLng`；生产环境必须指定已有语言而不是默认 `dev`。
- React 已负责文本转义，因此 i18next React 配置使用 `interpolation.escapeValue = false`。
- 本项目 TypeScript 为 strict 模式；翻译资源从英文基准资源推导 key 类型，并通过测试校验中英文 key 对齐。

参考：

- https://react.i18next.com/latest/usetranslation-hook
- https://react.i18next.com/latest
- https://www.i18next.com/principles/fallback
- https://www.i18next.com/overview/typescript

---

## 4. Locale 模型

```typescript
export type UiLanguagePreference = 'system' | 'zh-CN' | 'en-US'
export type SupportedLocale = 'zh-CN' | 'en-US'

export function resolveSupportedLocale(
  preference: UiLanguagePreference,
  systemLanguages: readonly string[],
): SupportedLocale
```

解析规则：

1. 显式 `zh-CN` / `en-US` 直接返回。
2. `system` 下按系统语言列表顺序查找第一个非空 locale。
3. `zh`、`zh-CN`、`zh-SG`、`zh-Hans*` 解析为 `zh-CN`。
4. 其他所有 locale（包括 `zh-TW`、`zh-HK`、`ja`）回退为 `en-US`，因为当前只交付简体中文和英文。
5. 缺失、非法或旧设置值归一化为 `system`。

`document.documentElement.lang` 始终设置为 effective locale。

---

## 5. 设置与启动数据流

```text
RuntimeStore settings.json
        │ uiLanguage preference
        ▼
appStore.initialize()
        │ resolve(system languages)
        ├── i18n.changeLanguage(effectiveLocale)
        ├── documentElement.lang = effectiveLocale
        └── set appStore uiLanguage/effectiveLocale
                    │
                    ▼
             render application
```

语言切换：

```text
Settings selector
  -> appStore.saveUiLanguage(preference)
  -> settings:update
  -> RuntimeStore validates + persists preference
  -> Main locale source immediately observes updated settings
  -> Renderer resolves locale + i18n.changeLanguage()
  -> React rerenders current route without reload
```

若保存失败，Renderer 恢复上一偏好并展示当前语言的错误提示。

---

## 6. 资源组织

```text
shared/i18n/
  locale.ts                 # shared types + normalization/resolution
  errors.ts                 # LocalizedAppError + catalogs + formatting

src/i18n/
  index.ts                  # i18next initialization
  resources/
    en-US.ts                # canonical key set
    zh-CN.ts                # identical key structure
  formatters.ts             # date/time/number/list formatting
  resourceParity.test.ts    # recursive key parity validation
```

首期使用单一 `translation` namespace 和语义化嵌套 key，避免在迁移期引入 namespace 懒加载和 Suspense。资源按顶层领域分组：

- `common`
- `navigation`
- `start`
- `workspace`
- `works`
- `runs`
- `files`
- `settings`
- `knowledge`
- `widgets`
- `errors`
- `accessibility`

英文资源是 canonical key set。中文资源必须通过递归 key parity 测试。

---

## 7. 后台错误契约

```typescript
export interface LocalizedAppError {
  code: string
  messageKey: AppErrorMessageKey
  params?: Record<string, string | number>
  actionKey?: AppErrorActionKey
  message: string
  action?: string
  diagnostic?: {
    message?: string
    stack?: string
    stderr?: string
    details?: unknown
  }
}
```

### 7.1 规则

- 已知应用错误使用稳定 code/key；catalog 同时提供中英文 summary/action。
- Main 根据 `runtimeStore.getSettings().uiLanguage` 和系统 locale 解析 effective locale。
- IPC wrapper 处理 `success:false` / `ok:false` 的响应：
  - 若已有结构化 code，按 code 解析。
  - 若只有字符串，使用 `IPC_OPERATION_FAILED` 包装，原字符串进入 diagnostic。
  - 保留兼容 `error: string` 时，该字符串必须为当前 locale 的用户消息。
  - 额外返回 `errorInfo: LocalizedAppError` 供新 UI 使用。
- `{code,message}` 形式保持对象结构，但 `message` 替换为本地化用户消息，原 message 放入 diagnostic。
- 不翻译第三方原文、命令输出、stack、stderr 或日志字段。
- Renderer 可使用 `messageKey + params` 按当前语言重新解析，支持已展示错误随语言变化。

### 7.2 首批稳定错误

- `IPC_OPERATION_FAILED`
- `INVALID_INPUT`
- `NOT_FOUND`
- `PERMISSION_DENIED`
- `PROJECT_REQUIRED`
- `PACKAGE_NOT_FOUND`
- `CONVERSATION_NOT_FOUND`
- `OFFLINE_MODE`
- `LLM_NOT_CONFIGURED`
- `INSTALL_FAILED`
- `UNKNOWN`

未知业务 code 使用 `UNKNOWN` 文案，但保留原 code 和 diagnostic。

---

## 8. Renderer 迁移策略

### 8.1 第一批：全局可见

- `AppMessageDialog`
- `AppShell` / `Sidebar` / `StatusBar`
- `SplashScreen`
- `StartPage`
- `WorkspacePage`
- `SettingsPage` 的 Appearance 与通用页面标题

### 8.2 第二批：核心工作流

- `WorksPage`、Plan 控件及对话删除确认
- `RunsPage`、RunWorkspace、ChatPanel、ChatInput、消息状态
- `FilesPage`、编辑器/预览器操作文本
- `KnowledgePage`

### 8.3 第三批：共享与长尾

- Package/Project/Placeholder 页面
- Orphan/Activation/BlockingNotice
- 日志组件
- Embedded widgets
- Tooltip、placeholder、title、aria-label 与空状态

动态用户内容必须作为插值参数传入，不作为翻译 key。

---

## 9. 测试设计

| 测试 | 证明内容 |
|:-----|:---------|
| `shared/i18n/locale.test.ts` | 系统 locale 解析、非法偏好、仅中英文 |
| `shared/i18n/errors.test.ts` | 双语错误、参数插值、未知错误、诊断保留 |
| `src/i18n/resourceParity.test.ts` | 两套资源 key 完全一致 |
| appStore tests | 旧设置迁移、初始化语言、保存与切换、失败回滚 |
| Settings tests | 三个偏好选项、即时切换、选中状态 |
| layout/core page tests | Sidebar/Start/Works/Chat 中英文代表性渲染 |
| Main IPC tests | 失败响应的双语 envelope 与兼容字段 |
| full test/lint/build | 回归、类型和打包验证 |

另外执行用户可见硬编码扫描，排除测试 fixture、用户内容渲染和协议常量后检查残留。

---

## 10. 影响文件（预期）

### 新增

- `shared/i18n/locale.ts`
- `shared/i18n/errors.ts`
- `src/i18n/index.ts`
- `src/i18n/formatters.ts`
- `src/i18n/resources/en-US.ts`
- `src/i18n/resources/zh-CN.ts`
- 对应测试文件

### 修改

- `package.json` / lockfile
- `electron/stores/runtimeStore.ts`
- `electron/main.ts`
- `electron/preload.ts`
- `electron/electron-env.d.ts`
- `src/stores/appStore.ts`
- `src/App.tsx`
- `src/main.tsx`
- 第一方用户界面组件与测试

---

## 11. 设计完成门槛

- Story AC 均有明确实现位置和测试证据规划。
- 仅支持 `zh-CN` / `en-US`，不存在第三种有效 locale。
- Main/后台错误与 Renderer UI 使用同一 locale 决策。
- 兼容现有 IPC 消费者，不因结构化错误升级破坏现有页面。
- 原始诊断数据完整保留，用户消息与诊断明确分层。
- 语言切换不 reload、不清空 Store、不改变 LLM 回复语言。
