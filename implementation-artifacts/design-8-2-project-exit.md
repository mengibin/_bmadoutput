# Design: Project Exit to Start Page

**Story:** `8-2-project-exit-to-start.md`  
**设计原则:** 简单直接、状态清理完整

---

## 设计目标

1. **退出按钮**：在 Sidebar 提供明显的退出入口
2. **状态清理**：清除 appStore 中的项目相关状态
3. **路由跳转**：返回 Start 页面

---

## 改动范围

| 文件 | 改动类型 | 说明 |
|------|----------|------|
| `src/stores/appStore.ts` | MODIFY | 添加 `closeProject()` action |
| `src/components/Sidebar.tsx` | MODIFY | 添加退出按钮 |

---

## 数据流图

```
用户点击 "Exit Project"
        │
        ▼
┌───────────────────────┐
│ appStore.closeProject()│
└───────────────────────┘
        │
        ▼
┌───────────────────────┐
│ 清除项目相关状态       │
│ - currentProject: null │
│ - currentConversation  │
│ - projectFiles: []     │
└───────────────────────┘
        │
        ▼
┌───────────────────────┐
│ UI 自动跳转到 Start    │
│ (基于 currentProject)  │
└───────────────────────┘
```

---

## 详细实现

### 1. appStore 修改

```typescript
// src/stores/appStore.ts

interface AppState {
    currentProject: ProjectMetadata | null
    currentConversation: ConversationMetadata | null
    projectFiles: FileTreeNode[]
    // ...other fields
    
    // Actions
    setCurrentProject: (project: ProjectMetadata | null) => void
    closeProject: () => void
    // ...other actions
}

export const useAppStore = create<AppState>((set) => ({
    currentProject: null,
    currentConversation: null,
    projectFiles: [],
    
    setCurrentProject: (project) => set({ currentProject: project }),
    
    /**
     * 关闭当前项目，清理所有项目相关状态
     */
    closeProject: () => set({
        currentProject: null,
        currentConversation: null,
        projectFiles: [],
        // 可选：清理其他项目相关状态
    }),
    
    // ...other actions
}))
```

### 2. Sidebar 修改

```tsx
// src/components/Sidebar.tsx

import { LogOut } from 'lucide-react'

export function Sidebar() {
    const { currentProject, closeProject } = useAppStore()
    
    return (
        <aside className="sidebar">
            {/* 项目信息头部 */}
            {currentProject && (
                <div className="sidebar-header">
                    <div className="project-info">
                        <span className="project-name">{currentProject.projectName}</span>
                    </div>
                    <button 
                        className="exit-project-btn"
                        onClick={closeProject}
                        title="Exit Project"
                    >
                        <LogOut size={18} />
                    </button>
                </div>
            )}
            
            {/* ...rest of sidebar... */}
        </aside>
    )
}
```

### 3. CSS 样式

```css
/* src/components/Sidebar.css */

.sidebar-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 12px 16px;
    border-bottom: 1px solid var(--border-color);
}

.project-info {
    flex: 1;
    overflow: hidden;
}

.project-name {
    font-weight: 600;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
}

.exit-project-btn {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 32px;
    height: 32px;
    border: none;
    background: transparent;
    border-radius: 6px;
    cursor: pointer;
    color: var(--text-secondary);
    transition: all 0.2s;
}

.exit-project-btn:hover {
    background: var(--hover-bg);
    color: var(--text-primary);
}
```

### 4. 路由处理（如使用 react-router）

```tsx
// src/App.tsx 或 MainLayout.tsx

function MainLayout() {
    const { currentProject } = useAppStore()
    
    // 无项目时显示 Start 页面
    if (!currentProject) {
        return <StartPage />
    }
    
    return (
        <div className="main-layout">
            <Sidebar />
            <MainContent />
        </div>
    )
}
```

---

## 测试策略

### 手动测试

1. 打开项目，验证退出按钮可见
2. 点击退出，验证返回 Start 页面
3. 再次打开同一项目，验证数据保留

### 自动测试

```typescript
describe('closeProject', () => {
    it('should clear currentProject', () => {
        useAppStore.getState().closeProject()
        expect(useAppStore.getState().currentProject).toBeNull()
    })
    
    it('should clear currentConversation', () => {
        useAppStore.getState().closeProject()
        expect(useAppStore.getState().currentConversation).toBeNull()
    })
})
```

---

## 向后兼容

无破坏性变更，仅添加新功能。
