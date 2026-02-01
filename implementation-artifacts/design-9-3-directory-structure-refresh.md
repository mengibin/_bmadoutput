# Design: Directory Structure Refresh

**Story:** `9-3-directory-structure-refresh.md`
**设计原则:** 响应迅速、资源节约

---

## 设计目标

1.  **自动感知**：使用 Chokidar 监听项目文件变更
2.  **低延迟**：变更发生后 UI 迅速更新
3.  **防抖动**：避免短时间内大量 I/O 导致的频繁重绘

---

## 改动范围

| 文件 | 改动类型 | 说明 |
|------|----------|------|
| `runtime/electron/main.ts` | MODIFY | 初始化 FileWatcherService |
| `runtime/electron/services/FileWatcherService.ts` | NEW | 封装 chokidar 逻辑 |
| `runtime/src/pages/FilesPage/FilesPage.tsx` | MODIFY | 监听 `files:changed` 事件 |
| `runtime/src/components/ProjectExplorer/ProjectExplorer.tsx` | MODIFY | 支持外部触发刷新 |

---

## 数据结构

### FileWatcherService

```typescript
class FileWatcherService {
    private watcher: chokidar.FSWatcher | null = null;
    
    start(projectRoot: string): void
    stop(): void
}
```

### IPC Events

- `files:watch-start` (Renderer -> Main)
- `files:watch-stop` (Renderer -> Main)
- `files:changed` (Main -> Renderer, payload: `{ type: 'add'|'change'|'unlink', path: string }`)

---

## 数据流图

```
App 启动 / 打开项目
      │
      ▼
Main: FileWatcherService.start(root)
      │
      ▼
chokidar.watch(root, { ignored: /node_modules|.git/ })
      │
      ▼
FS Event (e.g., touch new.txt)
      │
      ▼
chokidar 'add' event
      │
      ▼
Debounce (100ms)
      │
      ▼
IPC Send 'files:changed'
      │
      ▼
Renderer: ProjectExplorer
      │
      ▼
Refresh File Tree (Re-fetch or Patch)
```

---

## 详细实现

### 1. Main Process (FileWatcherService)

- **单例模式**：同一时间只 Watch 一个 active project。
- **配置**：`ignoreInitial: true`（启动时不触发 add 事件，由 FilesPage 初始加载负责）。
- **防抖**：使用 `lodash.debounce` 或简单 timer，聚合 100-300ms 内的事件，只发送一次 "refresh needed" 信号（最简单的实现），或者发送具体变更列表（更复杂）。
- **建议 MVP**：发送 "refresh needed" 信号，前端重新 fetch 整个目录树（如果目录树不是特别巨大）。或者只 fetch 变更目录的父级。

### 2. Renderer

- `useEffect` 注册 IPC listener。
- 收到 `files:changed` -> 调用 `refreshTree()`。

---

## 测试策略

### 手动测试

1. 打开 Runtime 文件浏览器。
2. 在 Terminal 使用 `touch test.txt` 创建文件。
3. 观察 UI 是否自动出现 test.txt。
4. 使用 `rm test.txt` 删除文件，观察 UI。
5. 测试 `node_modules` 变更不应触发刷新（Ignored）。

### 自动测试

- 单元测试 FileWatcherService 事件发射逻辑（通过 Mock chokidar）。
