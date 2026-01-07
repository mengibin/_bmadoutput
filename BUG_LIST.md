# Bug List

This document is used to track bugs in the project.

## How to add a bug
1. Add a new entry to the "Active Bugs" section.
2. Provide a description.
3. Paste a screenshot if applicable (images will save to `assets/bugs` if your IDE supports it, or put them there manually).
   - *Tip: In VS Code, you can often just paste an image directly into the markdown file and it will ask where to save it.*
4. Set Priority and Status.

---

## Active Bugs

*No active bugs.*

---

## Resolved Bugs

### [BUG-20260104-191500] Package Import fails to switch active package in open project
- **Status**: ✅ Verified
- **Priority**: Medium
- **Date**: 2026-01-04
- **Resolved**: 2026-01-04
- **Description**:
  If the project is opened, when user import a new package in setting panel, the open project should not be switched to the new package. The opened project should bind the old package.

- **Resolution**:
  Updated `importPackage` in `appStore.ts`. It now checks if a project is active. If so, it only adds the imported package to the cache without changing the `activePackage` or `activeProjectConfig`. If no project is open, it sets the new package as active.

### [BUG-20251230-151739] 文件列表显示错误
- **Status**: ✅ Verified
- **Priority**: Medium
- **Date**: 2025-12-30
- **Resolved**: 2025-12-30
- **Description**:
  把mac 的隐藏文件都显示出来了

- **Resolution**:
  在 `runtimeStore.ts` 的 `buildTreeNode` 方法中添加了隐藏文件过滤逻辑，跳过以 `.` 开头的文件（如 `.DS_Store`, `.git`）。

- **Screenshot**:
  ![Screenshot](assets/bugs/bug-20251230-151739.png)
