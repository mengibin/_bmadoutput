# Story 9.1: CSV/Excel 预览与编辑（按文件原样展示）

Status: done

## 背景 / 目标

在 Runtime 的文件预览区中，为 `.csv` / `.xls` / `.xlsx` 提供**表格化预览**与**编辑**能力，且交互模式与其它文件类型保持一致：

- 默认打开为 **Preview**（只读预览）
- 用户点击 **Edit** 后进入编辑态（可修改单元格内容）
- Preview 与 Edit 的 UI 样式一致，仅“是否可编辑”不同

其中：

- **Excel**：预览必须尽可能“按 Excel 文件本来的内容和布局样式”展示（不做“first row as header”等二次推断/改造）。
- **CSV**：使用二维表格预览，**第一行作为 header（列名）**。
- **滚动条**：Excel 预览/编辑必须具有**横向 + 纵向**滚动条，且滚动过程中不应自动跳回顶部/原位置。

---

## 用户故事

作为一名 **Consumer**，
我希望在 Runtime 内按文件原样预览 Excel，并在需要时切换到 Edit 模式修改 Excel/CSV 的内容，
从而无需打开外部 Excel 应用即可完成查看与轻量编辑。

---

## 验收标准（Acceptance Criteria）

### AC-1：交互模式与其它文件一致（Preview/Edit）
**Given** 我在 Files/Workspace 里打开一个 `.csv` / `.xlsx` 文件  
**Then** 默认进入 **Preview（只读）**  
**And** UI 上提供与其它文件一致的 **Preview / Edit** 切换  
**When** 我点击 **Edit**  
**Then** 进入可编辑状态（可修改单元格内容）  
**And** Preview 与 Edit 的视觉样式保持一致（仅交互能力不同）

### AC-2：Excel 必须按源文件内容展示（不做“表头推断”）
**Given** 我打开一个 Excel 文件（`.xls` / `.xlsx`）  
**Then** 预览展示遵循 Excel 文件内容与结构  
**And** **不把第一行自动当成 header**（不进行“first row as header”推断）  
**And** 支持**多 Sheet**（以 tab 的形式切换）

### AC-3：Excel 预览/编辑的滚动条与滚动稳定性
**Given** 我正在预览或编辑 Excel  
**Then** 表格区域必须同时支持**横向滚动条**与**纵向滚动条**  
**And** 我向下/向右滚动时，不应出现“滚动后自动回到原位置”的回弹/跳转问题

### AC-4：CSV 使用二维表格预览，第一行作为 header
**Given** 我打开一个 CSV 文件  
**Then** 以二维表格方式渲染  
**And** CSV 的**第一行作为 header（列名）**，数据从第二行开始显示  
**And** Edit 模式下可编辑单元格内容并保持表格样式一致

### AC-5：保存行为
**Given** 我在 Edit 模式下修改了 CSV/Excel 的单元格  
**Then** 文件会被标记为 dirty（可保存）  
**When** 我点击 Save 或使用 `Ctrl/Cmd + S`  
**Then** 变更写回原文件（CSV 为 UTF-8；Excel 为二进制内容）  
**And** 保存成功后 dirty 状态清除

---

## 非目标（Non-goals）

- 不承诺完整复刻 Excel 的所有高级能力（例如：图表、宏、复杂条件格式、透视表等）。
- 公式计算可作为增强项（可先做到“显示公式文本/结果”，不强制实现完整计算引擎）。

---

## 风险与注意事项

- **样式保真**：Excel 的行高/列宽/合并单元格/基础样式需尽可能保留，否则会被认为“改变了原样式”。
- **滚动稳定**：组件不应在滚动过程中频繁重建（destroy/recreate）导致 scroll position 重置。
- **性能**：大文件需要分页/惰性加载策略，但仍需保证用户可访问全部数据。

---

## 设计文档

见：`_bmad-output/implementation-artifacts/design-9-1-csv-excel-editor.md`
