# Design: Enhanced Markdown Editor

**Story:** `9-2-enhanced-markdown-editor.md`
**设计原则:** 一致性、所见即所得、高效

---

## 设计目标

1.  **复刻 Builder 体验**：将 Builder 端成熟的 Markdown 编辑器（Toolbar + Split View）移植到 Runtime。
2.  **实时预览**：编辑左侧，右侧实时渲染 GFM。
3.  **快捷工具栏**：提供常用格式化按钮。

---

## 改动范围

| 文件 | 改动类型 | 说明 |
|------|----------|------|
| `crewagent-runtime/package.json` | MODIFY | 添加 `react-markdown`, `remark-gfm` 等依赖 |
| `crewagent-runtime/src/components/MarkdownEditor/*` | NEW | 实现 `MarkdownEditor`（含 Toolbar）与 `MarkdownPreview`（GFM 渲染） |
| `crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.tsx` | MODIFY | 在 `.md` 文件的 **Edit** 模式下替换原有 `textarea` 为 `MarkdownEditor` |

---

## 数据结构

### MarkdownEditor Props

```typescript
interface MarkdownEditorProps {
    value: string
    onChange: (value: string) => void
    readOnly?: boolean
    filePath?: string
}
```

---

## 数据流图

```
WorkspacePage File Tab（Edit）加载内容
      │
      ▼
MarkdownEditor 组件
  ├── Editor Pane (textarea)
  │     │
  │     ▼
  │   onChange -> update draft
  │     │
  │     ▼
  │   Sync Scroll (optional)
  │
  └── Preview Pane (react-markdown)
        ▲
        │
    渲染 Markdown AST
```

---

## 详细实现

### 1. 组件移植

- 从 `crewagent-builder-frontend` 复制代码。
- **注意样式兼容**：Builder 使用 Tailwind；Runtime 侧采用 CSS 变量（`--surface-*`, `--text-*` 等）与组件内 CSS，需确保暗色主题下可读性与间距一致。
- **图标**：确认 `lucide-react` 版本一致性。

### 2. Toolbar 实现

- 按钮列表：Bold, Italic, Link, Code, List, Table等。
- 操作逻辑：
  - 获取 `selectionStart`, `selectionEnd`。
  - 插入 Markdown 符号（如 `**text**`）。
  - 恢复光标位置。

### 3. Split View 布局

- 使用 Flex 布局：
  - `flex-1` for Editor
  - `flex-1` for Preview
- 移动端/小屏幕下可能需要 Tab 切换（Editor | Preview），但在 Desktop App 中 Split View 是默认更好的选择。

---

## 测试策略

### 手动测试

1. 打开 .md 文件。
2. 点击 Toolbar 按钮，验证文本插入正确。
3. 在左侧输入，验证右侧渲染正确（包括表格、代码块）。
4. 验证滚动同步（如果实现了）。

### 自动测试

- 单元测试 Toolbar 工具函数（text insertion logic）。
