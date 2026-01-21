# Story 5.18: Default Package Initialization

## 概述

当 Runtime 首次启动（或 RuntimeStore 为空）时，检查是否存在内置的默认 `.bmad` 包。如果存在则自动导入，如果不存在则跳过，走现有流程。

---

## 用户故事

As a **Consumer**,
I want the Runtime to automatically import a default `.bmad` package on first launch if one is bundled,
So that I can immediately start using the system without needing to manually import a package.

---

## 验收标准

### AC-1: 首次初始化时自动导入默认包（如存在）

**Given** the Runtime starts for the first time (empty RuntimeStore, no packages imported)
**And** a default package exists in `resources/default-package/`
**When** the `RuntimeStore` initializes
**Then** the default `.bmad` package is automatically imported
**And** the package appears in the packages list

### AC-2: 无默认包时正常跳过

**Given** the Runtime starts for the first time
**And** no default package exists in `resources/default-package/`
**When** the `RuntimeStore` initializes
**Then** the initialization continues normally with no errors
**And** the packages list remains empty (current behavior)

### AC-3: 已有包时不重复导入

**Given** the RuntimeStore already contains packages (not first launch)
**When** the `RuntimeStore` initializes
**Then** the default package check is skipped
**And** existing packages remain unchanged

### AC-4: 导入失败容错

**Given** the default package exists but import fails (corrupted file, validation error, etc.)
**When** the Runtime initializes
**Then** the Runtime continues to function normally
**And** the error is logged to console
**And** the user can still manually import packages

---

## 技术设计

### 1. 文件结构

```text
crewagent-runtime/
├── resources/
│   └── default-package/        # 可选目录，放入 .bmad 文件即可自动导入
│       └── *.bmad              # 任意 .bmad 文件
└── electron/
    └── stores/
        └── runtimeStore.ts     # 主要修改点
```

### 2. RuntimeStore 修改

```typescript
// In RuntimeStore constructor, after existing initialization:
constructor() {
    // ... existing code ...
    this.ensureDirectories()
    this.loadPackagesIndex()
    this.loadProjectsIndex()
    this.loadSettings()
    
    // NEW: Try to initialize default package on first launch
    this.tryInitializeDefaultPackage()
}

private tryInitializeDefaultPackage() {
    // Skip if packages already exist
    if (Object.keys(this.packages).length > 0) {
        return
    }
    
    // Find default package if exists
    const defaultPackagePath = this.findDefaultPackage()
    if (!defaultPackagePath) {
        // No default package, continue with current flow
        return
    }
    
    // Try to import
    console.log('First launch: importing default package from', defaultPackagePath)
    this.importPackage(defaultPackagePath)
        .then((result) => {
            if (result.success) {
                console.log('Default package imported:', result.package?.name)
            } else {
                console.warn('Failed to import default package:', result.error)
            }
        })
        .catch((error) => {
            console.warn('Error importing default package:', error)
        })
}

private findDefaultPackage(): string | null {
    const searchPaths = [
        // Production: bundled in extraResources
        path.join(process.resourcesPath, 'default-package'),
        // Development: in project resources
        path.join(app.getAppPath(), 'resources', 'default-package')
    ]
    
    for (const dir of searchPaths) {
        if (!fs.existsSync(dir)) continue
        
        const files = fs.readdirSync(dir).filter(f => f.endsWith('.bmad'))
        if (files.length > 0) {
            return path.join(dir, files[0])  // Return first .bmad file found
        }
    }
    
    return null
}
```

### 3. electron-builder.json5 修改（可选）

只有当 `resources/default-package/` 目录存在时才需要配置：

```json5
{
  "extraResources": [
    // ... existing resources ...
    {
      "from": "resources/default-package",
      "to": "default-package",
      "filter": ["**/*.bmad"]
    }
  ]
}
```

---

## 实现步骤

1. 修改 `RuntimeStore` 添加 `tryInitializeDefaultPackage()` 和 `findDefaultPackage()` 方法
2. 在构造函数末尾调用 `tryInitializeDefaultPackage()`
3. （可选）在 `electron-builder.json5` 添加 extraResources 配置

---

## 影响分析

| 组件 | 变更 | 风险 |
|:-----|:-----|:-----|
| `RuntimeStore` | 添加两个私有方法 | 低 |
| 现有流程 | 无默认包时完全兼容 | 无 |
