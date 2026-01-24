# Story 8.2: Project Exit to Start Page

## 概述

允许用户从 Project 内部退出，返回到 Start 页面，以便切换项目或关闭当前工作会话。

---

## 用户故事

As a **Consumer**,
I want to exit the current Project and return to the Start page,
So that I can switch to a different project or close my current work session.

---

## 验收标准

### AC-1: 退出按钮可见

**Given** I am inside a Project (viewing conversations, files, or settings)
**When** I look at the sidebar or header
**Then** I see an "Exit Project" or "Close Project" button/icon

### AC-2: 点击退出返回 Start 页面

**Given** I am inside a Project
**When** I click the "Exit Project" button
**Then** I am returned to the Start page (Home/Welcome screen)
**And** the current project is closed

### AC-3: 项目状态保留

**Given** I exit a Project
**When** I later reopen the same project
**Then** my previous conversations and state are preserved
**And** the project appears in the recent projects list

### AC-4: 最近项目列表

**Given** I am on the Start page
**When** I view recent projects
**Then** I can see the project I just exited
**And** I can click to re-open it

---

## 技术设计

### 1. appStore 修改

```typescript
// In appStore.ts
interface AppState {
    currentProject: ProjectMetadata | null
    // ... other fields
}

const useAppStore = create<AppState>((set) => ({
    currentProject: null,
    
    // NEW: Close current project and return to Start
    closeProject: () => set({ 
        currentProject: null,
        currentConversation: null,
        // Reset other project-specific state
    }),
    
    // ... other actions
}))
```

### 2. UI 修改

在 Sidebar 或 Header 中添加退出按钮：

```tsx
// In Sidebar.tsx or Header.tsx
<button 
    onClick={() => appStore.closeProject()}
    title="Exit Project"
>
    <ExitIcon /> Exit
</button>
```

### 3. 路由处理

```typescript
// When project is closed, navigate to Start page
useEffect(() => {
    if (!currentProject) {
        navigate('/') // or navigate to Start page
    }
}, [currentProject])
```

---

## 实现步骤

1. 在 `appStore` 中添加 `closeProject()` action
2. 在 Sidebar 或 Header 中添加退出按钮
3. 处理路由跳转逻辑
4. 确保项目状态正确保存

---

## 影响分析

| 组件 | 变更 | 风险 |
|:-----|:-----|:-----|
| `appStore` | 添加 `closeProject()` action | 低 |
| Sidebar/Header | 添加退出按钮 | 低 |
| 路由逻辑 | 添加跳转处理 | 低 |
