# Design: CSV/Excel 预览与编辑（Excel 按源文件原样展示）

**Story:** `_bmad-output/implementation-artifacts/9-1-csv-excel-editor.md`

---

## 设计原则

1. **交互一致性**：操作模式与其它文件一致（Preview/Edit 切换）。
2. **Excel 保真**：Excel 预览/编辑尽可能按源文件内容展示，避免二次“推断/改造”（例如 first row as header）。
3. **样式不变**：Preview 与 Edit 视觉样式一致，仅交互能力不同；保存时尽可能只修改“值”，不破坏原文件布局/样式。
4. **可滚动**：表格区域必须有横向与纵向滚动条，并且滚动过程稳定（不回弹、不跳顶）。
5. **不阻塞 UI**：解析与初始化避免阻塞主线程；大文件提供分页/惰性策略。

---

## 方案概述

统一使用 `jspreadsheet-ce` 作为表格渲染与编辑引擎，但遵循两种文件的不同语义：

- **Excel（.xls/.xlsx）**
  - 视图：像 Excel 一样展示 sheet（tab）、单元格、合并单元格、行高/列宽、基础样式（尽可能）。
  - 规则：不做表头推断；列头显示 A/B/C...（类似 Excel），第一行仍是数据。
  - 保存：基于“原 workbook”做增量更新，尽可能只更新单元格值，保留合并/行列宽高/样式等元数据。

- **CSV（.csv）**
  - 视图：二维表格展示，第一行作为 header（列名），数据从第二行开始。
  - 保存：将 header + data 写回 CSV（UTF-8）。

Preview/Edit 模式通过同一组件的 `readOnly/editable` 开关实现，保证两种模式样式一致。

---

## 关键依赖

- `jspreadsheet-ce`（表格渲染/编辑）
- `jsuites`（jspreadsheet 依赖；版本需与 jspreadsheet-ce 匹配，避免交互/滚动异常）
- `xlsx`（读取/写回 Excel）
- `papaparse`（读取/写回 CSV）

---

## UI / 交互设计

### 1) FilesPage / WorkspacePage 的行为

- 默认进入 **Preview**（只读）
- 用户点击 **Edit** 后进入编辑态（可编辑）
- 编辑态显示 Save（以及 `Ctrl/Cmd + S`）
- 只读文件：Edit disabled；保持 Preview

> 注意：不要为了“表格内部滚动”而让外层页面滚动。外层容器对 spreadsheet 应为 `overflow: hidden`，滚动条只出现在表格内容区。

### 2) Excel 的多 Sheet

- UI 使用类似 Excel 的 tab 切换 Sheet
- Sheet 切换不应触发整组件频繁 destroy/recreate（会导致滚动位置丢失）

推荐：优先使用 `jspreadsheet.tabs(container, sheetsOptions[])` 内建 workbook/tab 能力，减少自定义 tab 布局与状态同步复杂度。

---

## 数据模型

### 1) Excel：WorkbookRef（保留原样式/结构）

```ts
type ExcelWorkbookState = {
  bookType: 'xls' | 'xlsx'
  workbook: XLSX.WorkBook
  sheetNames: string[]
  // 用于回写：记录原始 !ref，合并单元格，行高列宽等
}
```

### 2) CSV：Header + Rows

```ts
type CsvState = {
  headers: string[]
  rows: string[][]
}
```

---

## Excel 渲染：从 SheetJS -> jSpreadsheet 配置

目标：在 UI 中尽可能保留“原 Excel 样式/布局”。

### 需要保留的关键信息（优先级从高到低）

1. **单元格值**（包含空值）
2. **合并单元格**：`worksheet['!merges']` -> `mergeCells`
3. **列宽 / 行高**：`worksheet['!cols']` / `worksheet['!rows']`
4. **基础样式**（尽可能）：背景色、字体加粗、对齐等（SheetJS style -> CSS 字符串）

### 推荐策略

- 读取：`XLSX.read(arrayBuffer, { type: 'array', cellStyles: true, cellFormula: true })`
- 渲染：为每个 sheet 生成 `jspreadsheet` options：
  - `data`: 2D array（不要跳过第一行）
  - `mergeCells`: 由 `!merges` 转换
  - `columns`: 宽度来自 `!cols[].wpx`
  - `rows`: 高度来自 `!rows[].hpx`
  - `style`: 将 cell 样式映射为 CSS（best-effort）
- 多 sheet：使用 `jspreadsheet.tabs` 创建 workbook UI

---

## CSV 渲染：二维表格 + first row as header

- 读取：`Papa.parse(csv, { skipEmptyLines: false })`
- `headers = rows[0]`，`dataRows = rows.slice(1)`
- 渲染：`jspreadsheet(container, { columns: headers -> title, data: dataRows })`
- 保存：`Papa.unparse([headers, ...dataRows])`

---

## 滚动条与“滚动回弹”问题的设计处理

### 问题成因（典型）

- 组件在滚动过程中触发了 destroy/recreate（例如依赖数组变化导致重新初始化实例），导致 scrollTop 被重置。
- 外层容器也在滚动（双滚动容器），滚动事件链路导致回弹。
- 容器高度不稳定（未设置 min-height/height），使得表格不断重新计算布局。

### 设计约束 / 解决方案

1. **实例生命周期稳定**：同一文件/同一模式下，不频繁 destroy/recreate。
2. **高度计算**：使用 `ResizeObserver` 获取可用高度，并用 `tableOverflow: true` + `tableHeight` 固定表格滚动区域。
3. **唯一滚动容器**：外层（file preview panel）`overflow: hidden`；滚动发生在 `.jspreadsheet-content/.jexcel_content`。
4. **模式切换保留滚动位置**：Preview -> Edit 切换前记录 scrollLeft/scrollTop，切换后恢复。

---

## 保存策略（Excel / CSV）

### Excel 保存（尽量不破坏样式）

推荐：对“原 workbook”做增量更新，仅更新用户编辑过的单元格值：

1. 从 jspreadsheet 获取每个 sheet 的 2D 值矩阵
2. 对应写回原 `workbook.Sheets[sheetName]` 的 cell 对象（更新 `.v` / `.t`，保留 `.s` / `!merges` / `!cols` / `!rows`）
3. `XLSX.write(workbook, { type: 'base64', bookType })`
4. IPC 写回文件（base64 -> Buffer）

### CSV 保存

直接 `unparse` 并写回 UTF-8 文本即可。

---

## 改动范围（建议）

| 文件 | 改动类型 | 说明 |
|------|----------|------|
| `crewagent-runtime/src/components/Spreadsheet/SpreadsheetEditor.tsx` | MODIFY | 支持 Preview/Edit 模式；Excel 保真渲染；稳定滚动与保存策略。 |
| `crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.tsx` | MODIFY | 恢复/统一 Preview/Edit 的操作模式用于 CSV/Excel。 |
| `crewagent-runtime/src/pages/FilesPage/FilesPage.tsx` | MODIFY | 同上。 |
| `crewagent-runtime/electron/*` | MODIFY | Excel 读写使用 base64（二进制）通道。 |
| `crewagent-runtime/src/pages/*/*.css` | MODIFY | 修复外层 overflow，确保表格内部滚动条生效。 |

---

## 测试策略

### 手动测试（必测）

1. Excel 多 sheet：tab 可切换，内容与布局（合并单元格/行高列宽）基本一致。
2. Excel 滚动：横向+纵向滚动条出现，滚动不回弹。
3. Excel 编辑：Edit 模式可编辑，Save 写回后重新打开仍保持原样式（至少合并/行列宽高不丢）。
4. CSV 表头：第一行作为 header；Edit 后保存写回 header + data。
5. Preview/Edit 切换：切换后滚动位置尽量保持（至少不跳回顶部）。

### 自动测试（可选/增强）

- 解析函数的单元测试：CSV header 规则、Excel merge/cols/rows 映射正确性。
