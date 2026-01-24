# Design: First Launch Experience & Default Package Initialization

**Story:** `8-1-default-package-initialization.md`  
**设计原则:** 最小改动、无侵入性、容错优先

---

## 设计目标

1. **首次启动画面**：展示欢迎/启动画面，自动标记已完成
2. **默认包导入**：如存在默认包，自动导入
3. **容错**：任何失败不阻塞正常启动

---

## 改动范围

| 文件 | 改动类型 | 说明 |
|------|----------|------|
| `electron/stores/runtimeStore.ts` | MODIFY | 添加 2 个新方法 |
| `src/types/settings.ts` | MODIFY | 添加 `skipSplashScreen` 字段 |
| `src/App.tsx` 或 `SplashScreen.tsx` | MODIFY/NEW | 启动画面逻辑 |
| `electron-builder.json5` | MODIFY | 添加 extraResources 配置 |

---

## 数据流图

```
App 启动
    │
    ▼
┌────────────────────────────────┐
│ Check settings.skipSplashScreen │
└────────────────────────────────┘
    │
    ├── false (首次) ──────────────┐
    │                              ▼
    │                   ┌──────────────────┐
    │                   │ 显示 SplashScreen │
    │                   └──────────────────┘
    │                              │
    │                              ▼
    │                   ┌──────────────────────────┐
    │                   │ 完成后设置 skipSplash=true │
    │                   └──────────────────────────┘
    │                              │
    └── true (后续) ◄──────────────┘
                   │
                   ▼
        ┌────────────────────────────┐
        │ RuntimeStore Constructor   │
        │ ├─ ensureDirectories()     │
        │ ├─ loadPackagesIndex()     │
        │ ├─ loadSettings()          │
        │ └─ tryInitializeDefaultPkg()│
        └────────────────────────────┘
                   │
                   ▼
        ┌────────────────────────────┐
        │ packages.length === 0?     │
        └────────────────────────────┘
           │              │
           No             Yes
           │              │
           ▼              ▼
        [跳过]    ┌─────────────────────┐
                 │ findDefaultPackage() │
                 └─────────────────────┘
                          │
                    ┌─────┴─────┐
                    │           │
                   null       found
                    │           │
                    ▼           ▼
                 [跳过]   importPackage()
```

---

## 详细实现

### 1. RuntimeSettings 扩展

```typescript
// src/types/settings.ts

interface RuntimeSettings {
    llm: LLMConfig
    tools: AgentToolPolicy
    engine: EngineSettings
    python: PythonSettings
    node: NodeSettings
    mcpServers: McpServerConfig[]
    
    // NEW: 首次启动控制
    skipSplashScreen?: boolean
}

// defaultSettings
const defaultSettings: RuntimeSettings = {
    // ...existing...
    skipSplashScreen: false,
}
```

### 2. RuntimeStore 方法

```typescript
// electron/stores/runtimeStore.ts

class RuntimeStore {
    constructor() {
        this.storePath = path.join(app.getPath('userData'), 'runtime-store')
        // ...existing...
        this.ensureDirectories()
        this.loadPackagesIndex()
        this.loadProjectsIndex()
        this.loadSettings()
        
        // NEW: 首次启动时尝试导入默认包
        this.tryInitializeDefaultPackage()
    }
    
    /**
     * 首次启动时尝试导入默认包
     * - 仅当 packages 为空时执行
     * - 任何错误不阻塞启动
     */
    private tryInitializeDefaultPackage(): void {
        // 已有包，跳过
        if (Object.keys(this.packages).length > 0) {
            return
        }
        
        // 查找默认包
        const defaultPkgPath = this.findDefaultPackage()
        if (!defaultPkgPath) {
            console.log('[Init] No default package found, skipping')
            return
        }
        
        // 异步导入，不阻塞启动
        console.log('[Init] First launch, importing default package:', defaultPkgPath)
        this.importPackage(defaultPkgPath)
            .then((result) => {
                if (result.success) {
                    console.log('[Init] Default package imported:', result.package?.name)
                } else {
                    console.warn('[Init] Failed to import default package:', result.error)
                }
            })
            .catch((error) => {
                console.warn('[Init] Error importing default package:', error)
            })
    }
    
    /**
     * 查找默认包路径
     * - 优先查找 production 路径 (process.resourcesPath)
     * - 回退 development 路径
     */
    private findDefaultPackage(): string | null {
        const searchPaths = [
            path.join(process.resourcesPath, 'default-package'),
            path.join(app.getAppPath(), 'resources', 'default-package'),
        ]
        
        for (const dir of searchPaths) {
            if (!fs.existsSync(dir)) continue
            
            try {
                const files = fs.readdirSync(dir).filter(f => f.endsWith('.bmad'))
                if (files.length > 0) {
                    return path.join(dir, files[0])
                }
            } catch (err) {
                console.warn('[Init] Error reading default-package dir:', err)
            }
        }
        
        return null
    }
}
```

### 3. SplashScreen 组件

```tsx
// src/components/SplashScreen.tsx

import { useEffect, useState } from 'react'
import { useAppStore } from '../stores/appStore'

export function SplashScreen({ onComplete }: { onComplete: () => void }) {
    const [fadeOut, setFadeOut] = useState(false)
    
    useEffect(() => {
        // 显示 3 秒后淡出
        const timer = setTimeout(() => {
            setFadeOut(true)
            setTimeout(onComplete, 500) // 淡出动画后完成
        }, 3000)
        
        return () => clearTimeout(timer)
    }, [onComplete])
    
    return (
        <div className={`splash-screen ${fadeOut ? 'fade-out' : ''}`}>
            <img src="/app-icon.png" alt="CrewAgent" />
            <h1>Welcome to CrewAgent</h1>
            <p>Preparing your workspace...</p>
        </div>
    )
}
```

### 4. App.tsx 集成

```tsx
// src/App.tsx

function App() {
    const [showSplash, setShowSplash] = useState(false)
    const [initialized, setInitialized] = useState(false)
    
    useEffect(() => {
        async function init() {
            const settings = await window.electron.getSettings()
            
            if (!settings.skipSplashScreen) {
                setShowSplash(true)
            } else {
                setInitialized(true)
            }
        }
        init()
    }, [])
    
    const handleSplashComplete = async () => {
        // 标记已完成首次启动
        await window.electron.updateSettings({ skipSplashScreen: true })
        setShowSplash(false)
        setInitialized(true)
    }
    
    if (showSplash) {
        return <SplashScreen onComplete={handleSplashComplete} />
    }
    
    if (!initialized) {
        return <LoadingSpinner />
    }
    
    return <MainApp />
}
```

### 5. electron-builder.json5 配置

```json5
// electron-builder.json5

{
    // ...existing config...
    "extraResources": [
        // ...existing resources (python, node)...
        {
            "from": "resources/default-package",
            "to": "default-package",
            "filter": ["**/*.bmad"]
        }
    ]
}
```

---

## 目录结构

```
crewagent-runtime/
├── resources/
│   └── default-package/           # 放入 .bmad 文件后自动打包
│       └── getting-started.bmad   # 可选
├── electron/
│   └── stores/
│       └── runtimeStore.ts        # 修改
└── src/
    ├── App.tsx                    # 修改
    └── components/
        └── SplashScreen.tsx       # 新建
```

---

## 测试策略

### 手动测试

1. **首次启动**: 删除 userData，启动应用，验证显示启动画面
2. **后续启动**: 再次启动，验证跳过启动画面
3. **默认包导入**: 在 resources/default-package/ 放入 .bmad，验证自动导入
4. **无默认包**: 删除默认包目录，验证启动正常
5. **打包测试**: 构建 DMG，验证 extraResources 正确

### 自动测试

```typescript
describe('Default Package Initialization', () => {
    it('should skip when packages already exist', () => {})
    it('should import default package on first launch', () => {})
    it('should handle missing default package gracefully', () => {})
    it('should handle import failure gracefully', () => {})
})
```

---

## 向后兼容

| 场景 | 行为 |
|------|------|
| 现有用户（已有设置） | `skipSplashScreen` 默认 false，首次显示后不再显示 |
| 无默认包目录 | 静默跳过，不影响启动 |
| 默认包损坏 | 错误日志，不阻塞启动 |
