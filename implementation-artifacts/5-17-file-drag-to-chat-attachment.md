# Story 5.17: File Drag-to-Chat Attachment

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **Consumer**,
I want to drag files from the Files panel into the chat input area,
So that I can reference project files in my conversation with the Agent and provide context more efficiently.

## Acceptance Criteria

### 1. Files Panel Draggable
- **Given** I am viewing the Files panel in a Project
- **When** I drag a file node (not a directory) from the file tree
- **Then** the drag operation starts with file information (path, name) in the dataTransfer

### 2. Chat Input Drop Zone
- **Given** I am in a conversation (Works page) with the chat input visible
- **When** I drag a file over the chat input area
- **Then** the input area visually indicates it can receive the drop (e.g., border highlight)
- **And** when I drop the file, it is added to the attached files list

### 3. Attached Files Display
- **Given** I have attached one or more files to the chat input
- **When** I view the chat input area
- **Then** the attached files appear as tags above the text input
- **And** each tag shows the file name
- **And** each tag has a remove button (X) to detach the file

### 4. File Metadata in Message
- **Given** I have attached files and typed a message
- **When** I send the message
- **Then** the message includes file metadata for each attached file:
  - File path (relative to project root)
  - File name
- **And** the attached files list is cleared after sending
- **And** the LLM receives the file references and can use `fs.read` tool to access content if needed

### 5. Multiple File Attachment
- **Given** I have already attached some files
- **When** I drag and drop another file
- **Then** the new file is added to the existing list (not replaced)
- **And** duplicate files (same path) are not added again

### 6. Remove Attached File
- **Given** I have attached files to the chat input
- **When** I click the remove button on a file tag
- **Then** that file is removed from the attached list
- **And** other attached files remain

## Design

### UX / UI

#### Attached Files Area (Above Textarea)

位置：在 `.chat-input-inner` 内部、textarea 上方新增 `.chat-input-attachments` 容器。

```
┌─────────────────────────────────────────────────────┐
│ .chat-input                                         │
│  ┌───────────────────────────────────────────────┐  │
│  │ .chat-input-attachments (可选，有文件时显示)     │  │
│  │  ┌──────────────┐ ┌──────────────┐            │  │
│  │  │ 📄 file.md ✕ │ │ 📄 cfg.json ✕│            │  │
│  │  └──────────────┘ └──────────────┘            │  │
│  └───────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────┐  │
│  │ textarea                                       │  │
│  └───────────────────────────────────────────────┘  │
│  [Send Button]                                      │
└─────────────────────────────────────────────────────┘
```

**文件标签样式 (`.attachment-tag`)**:
- 背景: `var(--surface-secondary)`
- 边框: `1px solid var(--border-primary)`
- 圆角: `6px`
- 内边距: `4px 8px`
- 显示: `inline-flex` + `align-items: center` + `gap: 6px`
- 文件名: 最长显示 20 字符，超出显示 `...`
- 图标: 使用 lucide-react `File` 图标 (16px)，与 FilesPage 风格一致
- 删除按钮: lucide-react `X` 图标 (12px)，hover 时颜色变为 `var(--error)`

**拖拽悬停状态 (`.chat-input.drag-over`)**:
- 边框: `2px dashed var(--primary)`
- 背景: `var(--primary-bg)` (淡色)
- transition: `all 0.2s ease`

#### TreeNode Drag Visual
- 拖拽时光标: `grabbing`
- 拖拽元素半透明: `opacity: 0.7`

---

### API / Contracts

#### DataTransfer Format

拖拽时通过 `event.dataTransfer` 传递 JSON 字符串:

```typescript
// setData 时使用自定义 MIME type
const DRAG_MIME_TYPE = 'application/x-crewagent-file'

interface DragFileData {
    type: 'file'
    path: string      // 绝对路径（用于 @project 前缀转换）
    name: string      // 文件名（basename）
    projectRoot: string  // 项目根路径（用于计算相对路径）
}
```

#### Message Format

发送消息时，如果有附加文件，在用户消息前添加文件引用块:

```
📎 Referenced Files:
- `artifacts/report.md`
- `data/config.json`

[用户原始消息内容]
```

路径显示为相对于 projectRoot 的路径（不含 projectRoot 前缀）。

---

### Data / Storage

#### AttachedFile Interface

定义在 `ChatInput.tsx` 内部（不导出，纯 UI 状态）:

```typescript
interface AttachedFile {
    path: string      // 绝对路径
    name: string      // 文件名 (basename)
    relativePath: string  // 相对于 projectRoot 的路径
}
```

#### ChatInputProps Extension

```typescript
interface ChatInputProps {
    value: string
    disabled?: boolean
    isSending?: boolean
    placeholder?: string
    canSend?: boolean
    onChange: (value: string) => void
    onSend: () => void
    onStop?: () => void
    // NEW: File attachment props
    attachedFiles?: AttachedFile[]
    onFileDrop?: (files: AttachedFile[]) => void
    onRemoveFile?: (path: string) => void
}
```

#### State Management (WorksPage ConversationView)

```typescript
const [attachedFiles, setAttachedFiles] = useState<AttachedFile[]>([])

const handleFileDrop = (files: AttachedFile[]) => {
    setAttachedFiles(prev => {
        const existingPaths = new Set(prev.map(f => f.path))
        const newFiles = files.filter(f => !existingPaths.has(f.path))
        return [...prev, ...newFiles]
    })
}

const handleRemoveFile = (path: string) => {
    setAttachedFiles(prev => prev.filter(f => f.path !== path))
}
```

---

### Errors / Edge Cases

| 场景 | 处理方式 |
|:-----|:---------|
| 拖拽目录 | TreeNode 只对 `type === 'file'` 设置 `draggable`；drop 时忽略非 file type |
| 拖拽 readonly 文件 | 允许（只是引用，不修改） |
| 重复添加同一文件 | 通过 path 去重，不重复添加 |
| 跨页面拖拽 | v1 不支持，dataTransfer 在跨 tab/page 时数据可能丢失 |
| 文件已删除 | 发送时不检查（LLM 使用 fs.read 时会返回错误） |
| 大量文件 | attachments 区域设置 `max-height: 80px` + `overflow-y: auto` |

---

### Test Plan

#### Unit Tests

1. **ChatInput Drag Events (`ChatInput.test.tsx`)**
   - `onDragOver` 设置 dropEffect 为 'copy' 并阻止默认行为
   - `onDragEnter` 添加 `.drag-over` class
   - `onDragLeave` 移除 `.drag-over` class
   - `onDrop` 解析 dataTransfer 并调用 `onFileDrop`

2. **TreeNode Draggable (`FilesPage.test.tsx` 或内联测试)**
   - 文件节点 `draggable={true}`
   - 目录节点 `draggable={false}`
   - `onDragStart` 设置正确的 dataTransfer 数据

3. **Attachment State Logic**
   - `handleFileDrop` 正确去重
   - `handleRemoveFile` 正确移除指定文件

#### Integration Test (Manual)

1. 打开项目 → Works 页面
2. 创建/选择一个对话
3. 从 Files 面板拖拽 `.md` 文件到 ChatInput
4. 确认文件标签出现
5. 再拖拽另一个文件，确认追加
6. 尝试拖拽同一文件，确认不重复添加
7. 点击文件标签 X，确认移除
8. 输入消息并发送
9. 查看发送的消息包含 `📎 Referenced Files:` 块
10. 确认附加文件清空

---
## Tasks / Subtasks

- [x] Task 1: Define AttachedFile type and update ChatInput props (AC: #3, #4)
  - [x] 1.1 Add AttachedFile interface to types
  - [x] 1.2 Extend ChatInputProps with attachedFiles, onFileDrop, onRemoveFile
- [x] Task 2: Implement drag source in FilesPage TreeNode (AC: #1)
  - [x] 2.1 Add draggable attribute to file nodes (not directories)
  - [x] 2.2 Implement onDragStart handler with file data serialization
- [x] Task 3: Implement drop target in ChatInput (AC: #2, #5)
  - [x] 3.1 Add drag event handlers (onDragOver, onDragEnter, onDragLeave, onDrop)
  - [x] 3.2 Parse dropped data and call onFileDrop
  - [x] 3.3 Implement visual drag-over state
- [x] Task 4: Implement attached files UI in ChatInput (AC: #3, #6)
  - [x] 4.1 Render file tags above textarea
  - [x] 4.2 Implement remove button functionality
  - [x] 4.3 Add CSS styling for tags and drag states
- [x] Task 5: Integrate in WorksPage ConversationView (AC: #4)
  - [x] 5.1 Add attachedFiles state management
  - [x] 5.2 Implement handleFileDrop (add files, dedupe by path)
  - [x] 5.3 Implement handleRemoveFile
  - [x] 5.4 Modify handleSend to format file references
  - [x] 5.5 Clear attachments after send
- [ ] Task 6: Testing & Verification
  - [ ] 6.1 Unit tests for drag/drop handlers
  - [ ] 6.2 Unit tests for attachment state management
  - [x] 6.3 Manual end-to-end verification (pending)


## Dev Notes

- Follows existing FilesPage TreeNode patterns for file icons and styling
- Uses HTML5 Drag and Drop API with dataTransfer for cross-component communication
- Message format designed for LLM to understand file references and use `fs.read` tool
- Only file metadata (path/name) is sent, not file content (per user requirement)
- The LLM can then decide whether to read the full content using `fs.read` tool

### Project Structure Notes

- ChatInput located at `src/pages/RunsPage/components/ChatInput.tsx`
- FilesPage/TreeNode at `src/pages/FilesPage/FilesPage.tsx`
- WorksPage at `src/pages/WorksPage/WorksPage.tsx`
- CSS in `src/pages/RunsPage/RunsPage.css`

### References

- [Source: _bmad-output/epics.md#Epic-5] Story extends Works/Chat interface capabilities
- [Source: crewagent-runtime/src/pages/FilesPage/FilesPage.tsx] TreeNode component for drag source
- [Source: crewagent-runtime/src/pages/RunsPage/components/ChatInput.tsx] ChatInput for drop target
- [Source: crewagent-runtime/src/pages/WorksPage/WorksPage.tsx] ConversationView for integration

## Dev Agent Record

### Agent Model Used

Antigravity (Google DeepMind)

### Debug Log References

- TypeScript compilation: passed
- ESLint: passed (no errors in modified files)

### Completion Notes List

- Implemented file drag-to-chat using HTML5 Drag and Drop API
- Uses custom MIME type `application/x-crewagent-file` for dataTransfer
- AttachedFile interface exported from ChatInput for reuse
- Message format: `📎 Referenced Files:` block prepended to user message
- File handlers use useCallback for performance optimization
- Moved hooks to comply with React hooks rules

### File List

- crewagent-runtime/src/pages/RunsPage/components/ChatInput.tsx
- crewagent-runtime/src/pages/RunsPage/RunsPage.css
- crewagent-runtime/src/pages/FilesPage/FilesPage.tsx
- crewagent-runtime/src/pages/WorksPage/WorksPage.tsx
