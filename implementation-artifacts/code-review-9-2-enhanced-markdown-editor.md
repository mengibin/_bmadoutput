# Code Review: Story 9-2 – Enhanced Markdown Editor (Split View + Toolbar)

**Date:** 2026-01-27  
**Reviewer:** AI Code Reviewer (BMAD Method)  
**Story File:** `9-2-enhanced-markdown-editor.md`

---

## 范围说明

- 按 Story 9-2 的 AC 对照审计 Runtime 侧 Markdown 编辑体验（toolbar + split view + preview）。  
- 不评估工作区其他无关改动（如 Spreadsheet/Files 其他特性、全局 lint/tsc 历史遗留报错）。

---

## Summary

| Metric | Value |
|--------|-------|
| HIGH Issues | 0 |
| MEDIUM Issues | 0 |
| LOW Issues | 2 |
| Unit Tests | ✅ `npm -C crewagent-runtime run test` passed |
| Lint | ❌ `npm -C crewagent-runtime run lint` fails（仓库既有 `no-explicit-any` 等错误，非本 Story 引入） |
| Build | ❌ `npm -C crewagent-runtime run build:ci` fails（仓库既有 TS 报错，非本 Story 引入） |

---

## Acceptance Criteria 验证

| AC | Status | Evidence |
|----|--------|----------|
| AC-1 Toolbar | ✅ | Toolbar 已实现（按钮/行为齐全）：`crewagent-runtime/src/components/MarkdownEditor/MarkdownEditor.tsx:70`、`crewagent-runtime/src/components/MarkdownEditor/MarkdownEditor.tsx:102`；在 `.md` 的 `Edit` 模式下渲染：`crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.tsx:790`（Story 9-2 AC 已明确：进入 Edit 后显示 toolbar + split view） |
| AC-2 Split View | ✅ | Split view 布局为默认（Editor 左 / Preview 右）：`crewagent-runtime/src/components/MarkdownEditor/MarkdownEditor.css:9`、`crewagent-runtime/src/components/MarkdownEditor/MarkdownEditor.tsx:116` |
| AC-3 Real-time Preview + GFM | ✅ | Preview 由 `value` 实时驱动：`crewagent-runtime/src/components/MarkdownEditor/MarkdownEditor.tsx:136`；使用 `react-markdown` + `remark-gfm`：`crewagent-runtime/src/components/MarkdownEditor/MarkdownPreview.tsx:8` |
| AC-4 Visual Consistency | ✅ | Runtime 使用设计变量（`--surface-*`, `--text-*`, `--border-*`）统一风格；Markdown 编辑/预览样式在组件内聚：`crewagent-runtime/src/components/MarkdownEditor/MarkdownEditor.css:1` |

---

## Resolved During Review

- ✅ 采用 **Option C（维持现状但澄清）**：已回写 Story 9-2 的 AC 文案，明确 **进入 Edit 模式后** 才显示 Toolbar / Split View（同时同步更新 `epics.md` 的对应条目）。

---

## Issues & Recommendations

### L1 — 外链安全属性建议补齐 `noopener`

**Problem**  
`target="_blank"` 时仅设置了 `rel="noreferrer"`。建议使用 `rel="noopener noreferrer"`，避免 `window.opener` 风险（尤其 Electron 场景）。  

**Evidence**  
- `crewagent-runtime/src/components/MarkdownEditor/MarkdownPreview.tsx:14`

---

### L2 — `((void _node), (...))` 可读性一般

**Problem**  
该写法主要为绕过未使用变量告警，但会降低可维护性。  

**Evidence**  
- `crewagent-runtime/src/components/MarkdownEditor/MarkdownPreview.tsx:11`

**Recommendation**  
- 用更直观的方式忽略参数（如仅解构需要的字段/或 eslint 局部规则）。  
