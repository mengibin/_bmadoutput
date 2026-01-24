# Validation Report: Story 8-4

**Story**: 8-4 – Project Data Management & Orphan Recovery  
**Validated**: 2026-01-23  
**Status**: ✅ **SPEC APPROVED (Ready for Implementation)**

---

## 1. Story 完整性检查

| 检查项 | 状态 | 说明 |
|:-------|:-----|:-----|
| 用户故事清晰 | ✅ | 管理孤儿数据，支持重绑定/删除 |
| 验收标准完整 | ✅ | 6 个 AC 覆盖检测、展示、操作 |
| 技术设计可行 | ✅ | 完整代码示例和 UI Mockup |
| 实现步骤明确 | ✅ | 6 步详细指引 |
| 风险评估 | ✅ | 中等风险（涉及数据操作） |

---

## 2. 验收标准验证

### AC-1: 孤儿数据检测
- ✅ 可行：遍历 `projects.json`，检查 `projectRoot` 是否存在
- ✅ `fs.existsSync(projectRoot)` 判断

### AC-2: 孤儿数据展示
- ✅ 可行：展示项目名、原路径、最后打开时间、聊天数量、大小
- ✅ UI Mockup 提供清晰设计

### AC-3: 重新绑定文件夹
- ✅ 可行：更新 `projects.json` 中的 `projectRoot`
- ⚠️ 需同步更新新路径下的 `.crewagent.json`

### AC-4: 删除孤儿数据
- ✅ 可行：删除 `runtime-store/projects/<projectId>/` 目录
- ✅ 需确认对话框防止误删

### AC-5: 忽略外接设备
- ✅ 可行：使用内存标记，不持久化

### AC-6: 项目移动自动匹配
- ✅ 可行：基于 `.crewagent.json` 中的持久化 `projectId`
- ⚠️ 需向后兼容现有项目（自动生成 UUID）

---

## 3. 技术可行性

| 依赖项 | 状态 | 说明 |
|:-------|:-----|:-----|
| `ProjectMetadata` | ✅ 需扩展 | 添加新字段 |
| `.crewagent.json` | ✅ 需扩展 | 添加 `projectId` |
| `RuntimeStore` | ✅ 需添加方法 | 3 个新方法 |
| Settings UI | ✅ 需添加 | 新 Tab/Section |

---

## 4. Verdict

✅ **Story 8-4 Spec 已通过验证，可进入实现阶段**

**实现难度**: 中等

### 关键注意事项
1. **向后兼容**: 现有项目无 `projectId`，需自动迁移生成
2. **数据安全**: 删除操作需二次确认
3. **外接设备**: 检测 `/Volumes/` 路径提供"忽略"选项

---

## 5. 预估工作量

| 任务 | 预估 |
|:-----|:-----|
| `ProjectMetadata` 扩展 | 0.5h |
| `.crewagent.json` 格式升级 | 0.5h |
| `RuntimeStore` 新方法 | 2h |
| Settings UI 组件 | 3h |
| IPC handlers | 1h |
| 测试验证 | 1h |
| **总计** | **8h** |
