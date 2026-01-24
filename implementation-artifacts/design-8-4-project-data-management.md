# Design: Project Data Management & Orphan Recovery

**Story:** `8-4-project-data-management.md`  
**设计原则:** 数据安全、用户可控、向后兼容

---

## 设计目标

1. **孤儿检测**：检测 projectRoot 不存在的项目数据
2. **信息展示**：展示项目名、路径、大小等识别信息
3. **重新绑定**：允许绑定到新路径
4. **删除清理**：安全删除孤儿数据
5. **自动匹配**：基于持久化 projectId 自动匹配移动后的项目

---

## 改动范围

| 文件 | 改动类型 | 说明 |
|------|----------|------|
| `electron/stores/runtimeStore.ts` | MODIFY | 扩展 ProjectMetadata，添加方法 |
| `src/pages/SettingsPage/SettingsPage.tsx` | MODIFY | 添加项目数据管理 Tab |
| `src/components/OrphanProjectList.tsx` | NEW | 孤儿项目列表组件 |
| `electron/main.ts` | MODIFY | 添加 IPC handlers |

---

## 数据结构

### ProjectMetadata 扩展

```typescript
interface ProjectMetadata {
    projectId: string           // 持久化 UUID（用于跨路径匹配）
    projectName: string         // 项目名称
    projectRoot: string         // 原始路径
    lastOpenedAt: string        // 最后打开时间
    conversationCount?: number  // 聊天数量（运行时计算）
    totalSizeBytes?: number     // 数据大小（运行时计算）
}
```

### .crewagent.json 扩展

```json
{
    "schemaVersion": "1",
    "projectId": "550e8400-e29b-41d4-a716-446655440000",
    "projectName": "my-app",
    "artifactsDir": "artifacts",
    "activePackageId": null,
    "createdAt": "2026-01-21T00:00:00Z",
    "updatedAt": "2026-01-23T00:00:00Z"
}
```

### OrphanProject 类型

```typescript
interface OrphanProject extends ProjectMetadata {
    isOrphan: true
    conversationCount: number
    totalSizeBytes: number
}
```

---

## 数据流图

### 孤儿检测流程

```
┌─────────────────────────────────────┐
│ RuntimeStore.detectOrphanProjects() │
└─────────────────────────────────────┘
              │
              ▼
    ┌─────────────────────┐
    │ 遍历 projects.json   │
    └─────────────────────┘
              │
              ▼
  ┌───────────────────────────┐
  │ fs.existsSync(projectRoot)?│
  └───────────────────────────┘
         │          │
        Yes         No
         │          │
         ▼          ▼
      [跳过]   ┌────────────────┐
               │ 标记为孤儿      │
               │ 计算详细信息    │
               │ - conversationCount
               │ - totalSizeBytes
               └────────────────┘
                      │
                      ▼
              ┌──────────────────┐
              │ 返回 OrphanProject[]│
              └──────────────────┘
```

### 重绑定流程

```
用户选择新路径
      │
      ▼
┌───────────────────────────┐
│ RuntimeStore.rebindProject │
│ (projectId, newPath)       │
└───────────────────────────┘
      │
      ├─► 更新 projects.json
      │   └─ projectRoot = newPath
      │
      └─► 更新 newPath/.crewagent.json
          └─ projectId = 保持原值
```

---

## 详细实现

### 1. RuntimeStore 方法

```typescript
// electron/stores/runtimeStore.ts

class RuntimeStore {
    /**
     * 检测孤儿项目（projectRoot 不存在）
     */
    public detectOrphanProjects(): OrphanProject[] {
        const orphans: OrphanProject[] = []
        
        for (const [projectId, metadata] of Object.entries(this.projects)) {
            if (!fs.existsSync(metadata.projectRoot)) {
                const runtimePath = this.getProjectRuntimeRoot(metadata.projectRoot)
                
                orphans.push({
                    ...metadata,
                    projectId,
                    isOrphan: true,
                    conversationCount: this.countConversations(runtimePath),
                    totalSizeBytes: this.calculateDirSize(runtimePath),
                })
            }
        }
        
        return orphans
    }
    
    /**
     * 统计对话数量
     */
    private countConversations(runtimePath: string): number {
        const indexPath = path.join(runtimePath, 'conversations', 'index.json')
        if (!fs.existsSync(indexPath)) return 0
        
        try {
            const data = JSON.parse(fs.readFileSync(indexPath, 'utf-8'))
            return Array.isArray(data) ? data.length : 0
        } catch {
            return 0
        }
    }
    
    /**
     * 计算目录大小
     */
    private calculateDirSize(dirPath: string): number {
        if (!fs.existsSync(dirPath)) return 0
        
        let totalSize = 0
        const files = this.walkDir(dirPath)
        for (const file of files) {
            try {
                totalSize += fs.statSync(file).size
            } catch {}
        }
        return totalSize
    }
    
    /**
     * 重新绑定项目路径
     */
    public rebindProject(projectId: string, newPath: string): { success: boolean; error?: string } {
        const metadata = this.projects[projectId]
        if (!metadata) {
            return { success: false, error: 'Project not found' }
        }
        
        if (!fs.existsSync(newPath)) {
            return { success: false, error: 'New path does not exist' }
        }
        
        // 更新 projects.json
        const updated: ProjectMetadata = {
            ...metadata,
            projectRoot: newPath,
            lastOpenedAt: new Date().toISOString(),
        }
        this.projects[projectId] = updated
        this.saveProjectsIndex()
        
        // 更新 .crewagent.json
        const configPath = path.join(newPath, '.crewagent.json')
        try {
            let config = {}
            if (fs.existsSync(configPath)) {
                config = JSON.parse(fs.readFileSync(configPath, 'utf-8'))
            }
            config = { ...config, projectId }
            fs.writeFileSync(configPath, JSON.stringify(config, null, 2))
        } catch (error) {
            console.warn('Failed to update .crewagent.json:', error)
        }
        
        return { success: true }
    }
    
    /**
     * 删除孤儿项目数据
     */
    public deleteOrphanData(projectId: string): { success: boolean; error?: string } {
        const metadata = this.projects[projectId]
        if (!metadata) {
            return { success: false, error: 'Project not found' }
        }
        
        // 检查是否真的是孤儿
        if (fs.existsSync(metadata.projectRoot)) {
            return { success: false, error: 'Project still exists, cannot delete' }
        }
        
        // 删除 runtime 数据目录
        const runtimePath = this.getProjectRuntimeRoot(metadata.projectRoot)
        try {
            if (fs.existsSync(runtimePath)) {
                fs.rmSync(runtimePath, { recursive: true, force: true })
            }
        } catch (error) {
            return { success: false, error: `Failed to delete data: ${error}` }
        }
        
        // 从索引中移除
        delete this.projects[projectId]
        this.saveProjectsIndex()
        
        return { success: true }
    }
}
```

### 2. IPC Handlers

```typescript
// electron/main.ts

ipcMain.handle('project:detectOrphans', async () => {
    return runtimeStore.detectOrphanProjects()
})

ipcMain.handle('project:rebind', async (_, { projectId, newPath }) => {
    return runtimeStore.rebindProject(projectId, newPath)
})

ipcMain.handle('project:deleteOrphan', async (_, { projectId }) => {
    return runtimeStore.deleteOrphanData(projectId)
})
```

### 3. OrphanProjectList 组件

```tsx
// src/components/OrphanProjectList.tsx

import { useState, useEffect } from 'react'
import { Folder, Trash2, Link, AlertTriangle } from 'lucide-react'

interface OrphanProject {
    projectId: string
    projectName: string
    projectRoot: string
    lastOpenedAt: string
    conversationCount: number
    totalSizeBytes: number
}

function formatBytes(bytes: number): string {
    if (bytes < 1024) return `${bytes} B`
    if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`
    return `${(bytes / (1024 * 1024)).toFixed(1)} MB`
}

export function OrphanProjectList() {
    const [orphans, setOrphans] = useState<OrphanProject[]>([])
    const [loading, setLoading] = useState(true)
    
    useEffect(() => {
        loadOrphans()
    }, [])
    
    async function loadOrphans() {
        setLoading(true)
        const result = await window.electron.detectOrphanProjects()
        setOrphans(result)
        setLoading(false)
    }
    
    async function handleRebind(projectId: string) {
        const result = await window.electron.selectDirectory()
        if (!result) return
        
        const res = await window.electron.rebindProject({ projectId, newPath: result })
        if (res.success) {
            loadOrphans()
        } else {
            alert(res.error)
        }
    }
    
    async function handleDelete(projectId: string, projectName: string) {
        const confirmed = confirm(`确定删除 "${projectName}" 的所有数据？此操作不可恢复。`)
        if (!confirmed) return
        
        const res = await window.electron.deleteOrphanProject({ projectId })
        if (res.success) {
            loadOrphans()
        } else {
            alert(res.error)
        }
    }
    
    if (loading) return <div>Loading...</div>
    
    if (orphans.length === 0) {
        return <div className="no-orphans">✅ 没有孤儿项目数据</div>
    }
    
    return (
        <div className="orphan-list">
            <div className="orphan-header">
                <AlertTriangle size={16} />
                发现 {orphans.length} 个孤儿项目数据
            </div>
            
            {orphans.map((orphan) => (
                <div key={orphan.projectId} className="orphan-item">
                    <div className="orphan-icon">
                        <Folder size={24} />
                    </div>
                    <div className="orphan-info">
                        <div className="orphan-name">{orphan.projectName}</div>
                        <div className="orphan-path" title={orphan.projectRoot}>
                            原路径: {orphan.projectRoot}
                        </div>
                        <div className="orphan-meta">
                            最后打开: {new Date(orphan.lastOpenedAt).toLocaleDateString()} | 
                            聊天: {orphan.conversationCount} 条 | 
                            大小: {formatBytes(orphan.totalSizeBytes)}
                        </div>
                    </div>
                    <div className="orphan-actions">
                        <button onClick={() => handleRebind(orphan.projectId)}>
                            <Link size={16} /> 重新绑定
                        </button>
                        <button 
                            className="danger"
                            onClick={() => handleDelete(orphan.projectId, orphan.projectName)}
                        >
                            <Trash2 size={16} /> 删除
                        </button>
                    </div>
                </div>
            ))}
        </div>
    )
}
```

---

## 向后兼容

### 现有项目迁移

```typescript
// 打开项目时自动生成 projectId
private ensureProjectConfig(projectRoot: string): ProjectConfig {
    const existing = this.readProjectConfig(projectRoot)
    
    if (existing && existing.projectId) {
        return existing
    }
    
    // 生成新的 projectId
    const projectId = randomUUID()
    const config: ProjectConfig = {
        ...existing,
        projectId,
        schemaVersion: '1',
        projectName: existing?.projectName || path.basename(projectRoot),
        // ...
    }
    
    this.writeProjectConfig(projectRoot, config)
    return config
}
```

---

## 测试策略

### 手动测试

1. 创建项目，关闭应用
2. 移动项目文件夹
3. 重新打开应用，验证检测到孤儿
4. 测试重新绑定功能
5. 测试删除功能

### 自动测试

```typescript
describe('Orphan Project Detection', () => {
    it('should detect projects with missing projectRoot', () => {})
    it('should calculate conversation count correctly', () => {})
    it('should calculate dir size correctly', () => {})
})

describe('Rebind Project', () => {
    it('should update projectRoot in projects.json', () => {})
    it('should update .crewagent.json in new path', () => {})
})

describe('Delete Orphan', () => {
    it('should remove runtime data directory', () => {})
    it('should remove from projects.json', () => {})
    it('should not delete if project still exists', () => {})
})
```
