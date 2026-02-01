# Design: File System Operations (Delete)

**Story:** `9-4-file-system-operations-delete.md`
**设计原则:** 安全第一、防误删

---

## 设计目标

1.  **右键菜单**：在文件树中集成上下文菜单
2.  **二次确认**：删除操作必须经过用户确认
3.  **递归删除**：支持删除非空文件夹

---

## 改动范围

| 文件 | 改动类型 | 说明 |
|------|----------|------|
| `crewagent-runtime/electron/main.ts` | MODIFY | 调整 `dialog:confirm` 默认聚焦在 Cancel（更安全） |
| `crewagent-runtime/electron/stores/runtimeStore.ts` | MODIFY | 删除逻辑 + 沙箱校验 + 禁止删除项目根目录 |
| `crewagent-runtime/src/pages/FilesPage/FilesPage.tsx` | MODIFY | 文件树右键 Context Menu（Delete） |
| `crewagent-runtime/src/hooks/useFileExplorer.ts` | MODIFY | `deletePath(path)` 统一删除入口 |

---

## 数据流图

```
用户右键文件 -> Select "Delete"
      │
      ▼
Renderer: Show Confirm Dialog
      │
      ▼
User Clicks "Yes"
      │
      ▼
IPC: fs:delete-path(path)
      │
      ▼
Main: Sandbox Check (path startsWith projectRoot?)
      │
      ▼
Main: fs.rm(path, { recursive: true })
      │
      ▼
IPC Result (Success/Fail)
      │
      ▼
Renderer: Toast/Notify & Refresh Tree
```

---

## 详细实现

### 1. IPC Handler

- **`files:delete`**:
  - `path.resolve(target)` 必须在 `projectRoot` 内。
  - 不能删除 `projectRoot` 本身。
  - 使用 `force: true` 忽略不存在的文件错误。

### 2. Context Menu

- 使用 Radix UI Context Menu 或简单的自定义绝对定位 div。
- 菜单项：`Delete`, `Rename` (Future), `Copy Path` (Future).
- 样式需适配 VS Code 风格。

### 3. Confirm Dialog

- 使用原生 `dialog.showMessageBox`（通过 `dialog:confirm` IPC）。
- Enter 默认聚焦在 "Cancel"（通过 `defaultId: 0`），只有点击 "Delete" 才执行。

---

## 测试策略

### 手动测试

1. 创建测试文件夹 `delete-me` 和文件 `delete-me/file.txt`。
2. 右键 `delete-me` -> Delete。
3. 弹出对话框 -> Cancel -> 验证未删除。
4. 右键 -> Delete -> Confirm -> 验证文件夹消失。

### 自动测试

- 单元测试 Main Process IPC handler，验证沙箱逻辑（尝试删除 `/tmp` 或父级路径应失败）。
