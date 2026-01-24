# Design: Settings Tab Reorder

**Story:** `8-3-settings-tab-reorder.md`  
**设计原则:** 最小改动

---

## 设计目标

1. 按使用频率和重要性重新排列 Settings 页面的 Tab

---

## 改动范围

| 文件 | 改动类型 | 说明 |
|------|----------|------|
| `src/pages/SettingsPage/SettingsPage.tsx` | MODIFY | 重排 tabs 数组 |

---

## 当前 Tab 顺序（示例）

```typescript
const tabs = [
    { id: 'llm', label: 'LLM', icon: Settings },
    { id: 'mcp', label: 'MCP', icon: Server },
    { id: 'tool', label: 'Tool', icon: Wrench },
    { id: 'appearance', label: 'Appearance', icon: Palette },
    { id: 'package', label: 'Package', icon: Package },
    // ...
]
```

---

## 新 Tab 顺序

```typescript
const tabs = [
    { id: 'package', label: 'Package', icon: Package },     // 1. 包管理
    { id: 'llm', label: 'LLM', icon: Settings },            // 2. LLM 配置
    { id: 'mcp', label: 'MCP', icon: Server },              // 3. MCP 服务器
    { id: 'tool', label: 'Tool', icon: Wrench },            // 4. 工具策略
    { id: 'engine', label: 'Engine', icon: Play },          // 5. 执行引擎
    { id: 'python', label: 'Python', icon: Terminal },      // 6. Python
    { id: 'node', label: 'Node', icon: Terminal },          // 7. Node.js
    { id: 'appearance', label: 'Appearance', icon: Palette }, // 8. 外观
]
```

---

## 实现

```tsx
// src/pages/SettingsPage/SettingsPage.tsx

// 找到 tabs 定义，按新顺序排列
const tabs: TabItem[] = [
    { id: 'package', label: 'Package', icon: Package },
    { id: 'llm', label: 'LLM', icon: Settings },
    { id: 'mcp', label: 'MCP', icon: Server },
    { id: 'tool', label: 'Tool', icon: Wrench },
    { id: 'engine', label: 'Engine', icon: Play },
    { id: 'python', label: 'Python', icon: Terminal },
    { id: 'node', label: 'Node', icon: Terminal },
    { id: 'appearance', label: 'Appearance', icon: Palette },
]
```

---

## 测试

1. 打开 Settings 页面
2. 验证 Tab 按新顺序显示
3. 验证每个 Tab 点击后内容正确
