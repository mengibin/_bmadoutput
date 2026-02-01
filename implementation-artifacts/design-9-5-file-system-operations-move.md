# Design: File System Operations (Move)

**Story:** `9-5-file-system-operations-move.md`
**设计原则:** 直观交互、防止意外

---

## 设计目标

1.  **拖拽交互**：支持标准 Drag and Drop (DnD) 操作
2.  **冲突处理**：防止同名文件覆盖
3.  **视觉反馈**：拖拽过程中高亮目标文件夹

---

## 改动范围

| 文件 | 改动类型 | 说明 |
|------|----------|------|
| `runtime/electron/main.ts` | MODIFY | 添加 `fs:move-path` IPC handler |
| `src/components/ProjectExplorer/ProjectExplorer.tsx` | MODIFY | 实现 DnD 事件处理 |
| `src/components/ProjectExplorer/FileTreeItem.tsx` | MODIFY | 添加 draggable 属性 |

---

## 数据流图

```
User Drags Item A
      │
      ▼
onDragStart (Set dataTransfer: path)
      │
      ▼
User Drop on Folder B
      │
      ▼
onDrop (Get path, targetFolder)
      │
      ▼
IPC: fs:move-path(src, dest)
      │
      ▼
Main: Check Dest Exists?
      │
No    Yes -> Error (Conflict)
▼
Main: fs.rename(src, dest)
      │
      ▼
IPC Result
      │
      ▼
Refresh Tree
```

---

## 详细实现

### 1. IPC Handler

- **`fs:move-path`**:
  - `fs.existsSync(dest)` 检查。如果存在，返回错误 `{ success: false, error: 'Target exists' }`。
  - `fs.renameSync(src, dest)`。
  - 错误处理：权限不足、文件被占用等。

### 2. UI Drag & Drop

- **Draggable**: `<div draggable="true" onDragStart={...}>`
- **Droppable**: 文件夹 Item 需要 `onDragOver` (preventDefault), `onDragEnter`, `onDragLeave`, `onDrop`。
- **视觉反馈**: `onDragEnter` 时添加高亮样式（如边框或背景色），`onDragLeave` / `onDrop` 移除。
- **限制**: 只能在 App 内部拖拽（通过检查 dataTransfer 类型）。

---

## 测试策略

### 手动测试

1. 拖拽 `file.txt` 到 `folder/` -> 成功移动。
2. 拖拽 `file.txt` 到已存在同名文件的 `folder/` -> 提示错误（Toast）。
3. 拖拽文件夹到其子文件夹 -> 提示错误或无效。

### 自动测试

- 单元测试 IPC handler 冲突检测逻辑。
