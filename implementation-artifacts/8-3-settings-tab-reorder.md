# Story 8.3: Settings Tab Reorder

## 概述

调整 Settings 页面的 Tab 顺序，按照使用频率和重要性排列，让用户更快找到常用设置。

---

## 用户故事

As a **Consumer**,
I want the Settings tabs to be ordered by importance and frequency of use,
So that I can quickly access the most commonly used settings.

---

## 验收标准

### AC-1: Tab 按规定顺序排列

**Given** I am on the Settings page
**When** I view the tab list
**Then** the tabs are ordered as:
  1. **Package** - 包管理 (import/remove)
  2. **LLM** - LLM provider/model 配置
  3. **MCP** - MCP server 管理
  4. **Tool** - Tool policy 设置
  5. **Engine** - 执行引擎设置
  6. **Python** - Python 环境配置
  7. **Node** - Node.js 配置
  8. **Appearance** - 外观设置

---

## 技术设计

### SettingsPage.tsx 修改

```tsx
// Current tab order (example)
const tabs = [
    { id: 'llm', label: 'LLM', icon: Settings },
    { id: 'appearance', label: 'Appearance', icon: Palette },
    // ...
]

// New tab order
const tabs = [
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

## 实现步骤

1. 打开 `SettingsPage.tsx`
2. 找到 tabs 定义数组
3. 按照验收标准重新排列顺序
4. 测试确保所有 tab 正常工作

---

## 影响分析

| 组件 | 变更 | 风险 |
|:-----|:-----|:-----|
| `SettingsPage.tsx` | 重排 tabs 数组 | 最小 |
