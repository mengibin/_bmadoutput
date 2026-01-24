# Story 8.4: Project Data Management & Orphan Recovery

Status: review

<!-- Note: Spec validation complete. See validation-report-story-8-4.md -->

## 概述

管理 RuntimeStore 中的项目数据，检测孤儿数据（原项目路径不存在），支持重新绑定或清理，避免数据丢失和存储浪费。

---

## 用户故事

As a **Consumer**,
I want to manage project data stored in RuntimeStore and recover orphan data when project folders are moved,
So that I don't lose chat history when reorganizing my files and can clean up unused data.

---

## 验收标准

### AC-1: 孤儿数据检测

**Given** the Runtime starts
**When** it detects orphan project data (original projectRoot path no longer exists)
**Then** the user is notified of orphan data in Settings or Start page

### AC-2: 孤儿数据展示

**Given** I view orphan project data in Settings
**When** I see the orphan project list
**Then** each orphan entry displays:
  - **项目名称** (from `.crewagent.json.projectName` or folder name)
  - **原路径** (truncated with tooltip for full path)
  - **最后打开时间**
  - **聊天数量**
  - **数据大小** (e.g., "2.3 MB")

### AC-3: 重新绑定文件夹

**Given** I view orphan project data
**When** I click "重新绑定文件夹"
**Then** I can select a new folder to associate with the historical data
**And** the project mapping is updated
**And** all conversations and run history are preserved

### AC-4: 删除孤儿数据

**Given** I view orphan project data
**When** I click "删除数据"
**Then** a confirmation dialog appears
**And** upon confirmation, the orphan data is removed from RuntimeStore

### AC-5: 忽略外接设备

**Given** I view orphan project data from removable storage
**When** I click "忽略(外接设备)"
**Then** the orphan is temporarily hidden until next restart

### AC-6: 项目移动自动匹配

**Given** I move a project folder to a new location
**When** I open the project from the new location
**Then** if `.crewagent.json` contains a persistent `projectId`, the historical data is automatically matched
**And** I see all previous conversations

---

## UI 设计

```
┌─────────────────────────────────────────────────────────────┐
│ 项目数据管理                                                 │
├─────────────────────────────────────────────────────────────┤
│ ⚠️ 发现 2 个孤儿项目数据                                     │
│                                                             │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 📁 my-crewagent-project                                 │ │
│ │    原路径: /Users/mengbin/old-folder/my-crewagent-...   │ │
│ │    最后打开: 2026-01-20 | 聊天: 5 条 | 大小: 2.3 MB      │ │
│ │    [重新绑定文件夹]  [删除数据]                          │ │
│ └─────────────────────────────────────────────────────────┘ │
│                                                             │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 📁 finance-workflow                                     │ │
│ │    原路径: /Volumes/USB/projects/finance-workflow       │ │
│ │    最后打开: 2026-01-15 | 聊天: 12 条 | 大小: 5.1 MB     │ │
│ │    [重新绑定文件夹]  [删除数据]  [忽略(外接设备)]         │ │
│ └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## 技术设计

### 1. ProjectMetadata 扩展

```typescript
interface ProjectMetadata {
    projectId: string         // UUID（持久化标识）
    projectName: string       // 项目名称
    projectRoot: string       // 原始路径
    lastOpenedAt: string      // 最后打开时间
    conversationCount?: number // 聊天数量
    totalSizeBytes?: number   // 数据大小
}
```

### 2. .crewagent.json 修改

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

### 3. RuntimeStore 新方法

```typescript
class RuntimeStore {
    // 检测孤儿项目
    public detectOrphanProjects(): OrphanProject[] {
        const orphans: OrphanProject[] = []
        for (const [projectId, metadata] of Object.entries(this.projects)) {
            if (!fs.existsSync(metadata.projectRoot)) {
                orphans.push({
                    ...metadata,
                    conversationCount: this.countConversations(projectId),
                    totalSizeBytes: this.calculateProjectSize(projectId),
                })
            }
        }
        return orphans
    }
    
    // 重新绑定项目路径
    public rebindProject(projectId: string, newPath: string): { success: boolean } {
        // 更新 projects.json 中的 projectRoot
        // 更新新路径下的 .crewagent.json
    }
    
    // 删除孤儿数据
    public deleteOrphanData(projectId: string): { success: boolean } {
        // 删除 runtime-store/projects/<projectId>/ 目录
        // 从 projects.json 中移除条目
    }
}
```

---

## 实现步骤

1. 扩展 `ProjectMetadata` 接口，添加 `projectId` 等字段
2. 修改 `.crewagent.json` 格式，添加持久化 `projectId`
3. 在 `RuntimeStore` 中添加孤儿检测和管理方法
4. 在 `SettingsPage` 中添加"项目数据管理" Tab/Section
5. 实现 UI 组件，显示孤儿列表和操作按钮
6. 添加重新绑定和删除的 IPC handlers

---

## 影响分析

| 组件 | 变更 | 风险 |
|:-----|:-----|:-----|
| `ProjectMetadata` | 新增字段 | 低（向后兼容） |
| `.crewagent.json` | 新增 `projectId` | 低（自动迁移） |
| `RuntimeStore` | 新增方法 | 中 |
| `SettingsPage` | 新增 Tab/Section | 中 |

## Tasks / Subtasks

- [x] 1) 扩展 `.crewagent.json` 写入持久化 `projectId`（AC: 6）
  - [x] 新建项目生成 UUID（`randomUUID()`）
  - [x] 旧项目自动补齐 `projectId`（向后兼容）

- [x] 2) RuntimeStore：孤儿检测与统计（AC: 1,2）
  - [x] 检测 `projects.json` 中 projectRoot 不存在的条目
  - [x] 统计聊天数量（conversations/index.json）
  - [x] 计算数据目录大小（runtime-store/projects/<projectId>/）

- [x] 3) RuntimeStore：重绑定 / 删除 / 忽略（AC: 3,4,5）
  - [x] 重绑定：更新 projects.json + 写入新路径 `.crewagent.json.projectId`
  - [x] 删除：仅当为孤儿时允许删除运行时数据目录并移除索引
  - [x] 忽略：仅内存标记，重启恢复显示

- [x] 4) IPC & Preload：暴露孤儿管理接口（AC: 1–5）
  - [x] `projects:getOrphanCount` / `projects:getOrphans`
  - [x] `projects:rebindOrphan` / `projects:deleteOrphan` / `projects:ignoreOrphan`

- [x] 5) UI：Settings & Start 页面提示与操作（AC: 1–5）
  - [x] Settings → Project Data：展示孤儿列表 + 重绑定/删除/忽略
  - [x] Start：启动时显示孤儿数据提醒入口

- [x] 6) Tests（AC: 1–6）
  - [x] 移动文件夹后自动匹配（projectId）
  - [x] 孤儿检测/重绑定/删除/忽略的单测覆盖

### Review Follow-ups (AI)

- [x] [AI-Review][HIGH] 修复“旧项目先移动后打开”导致数据无法恢复的问题：允许从孤儿数据强制重绑定到已存在不同 `projectId` 的目录，并把原目录 `projectId` 数据标记为 Detached orphan 以便管理（`crewagent-runtime/electron/stores/runtimeStore.ts`）
- [x] [AI-Review][MEDIUM] `detectOrphanProjects()` 目录大小计算改为异步 + 会话内缓存，避免主进程/Settings 卡顿（`crewagent-runtime/electron/stores/runtimeStore.ts`）
- [x] [AI-Review][MEDIUM] 补齐测试：覆盖 legacy `.crewagent.json` 不含 `projectId` 且“移动后再打开”的恢复路径，并验证 run history 不丢（AC-3）（`crewagent-runtime/electron/stores/runtimeStore.test.ts`）
- [x] [AI-Review][MEDIUM] 跨平台：Windows 增加 “非系统盘” 可移动盘启发式识别；UI 也允许对任意 orphan 执行 “Ignore (Until Restart)”（`crewagent-runtime/electron/stores/runtimeStore.ts`, `crewagent-runtime/src/components/OrphanProjectList.tsx`）
- [x] [AI-Review][LOW] Start 页面孤儿提醒增加 focus 时刷新 + 失败提示与重试（`crewagent-runtime/src/pages/StartPage/StartPage.tsx`）

## File List

- `_bmad-output/implementation-artifacts/8-4-project-data-management.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/stores/runtimeStore.test.ts`
- `crewagent-runtime/src/components/OrphanProjectList.tsx`
- `crewagent-runtime/src/pages/StartPage/StartPage.tsx`
- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/preload.ts`
- `crewagent-runtime/electron/electron-env.d.ts`
- `crewagent-runtime/electron/stores/runtimeStore.test.ts`
- `crewagent-runtime/src/components/OrphanProjectList.tsx` (NEW)
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx`
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.css`
- `crewagent-runtime/src/pages/StartPage/StartPage.tsx`
- `crewagent-runtime/src/pages/StartPage/StartPage.css`

## Dev Agent Record

### Agent Model Used

GPT-5.2（Codex CLI）

### Debug Log References

- `crewagent-runtime`: `npm test`
- `crewagent-runtime`: `npm run lint`

### Completion Notes List

- 新增 `.crewagent.json.projectId`：新项目生成 UUID；旧项目自动补齐 `projectId` 以保持历史数据路径一致。
- RuntimeStore 新增孤儿项目检测/统计、重绑定、删除、忽略（外接设备路径 `/Volumes/` 等提供忽略按钮）。
- Settings 增加 Project Data 区块显示孤儿列表并提供重绑定/删除/忽略；Start 页面启动提示孤儿数据入口。
- 增加单测覆盖：移动后自动匹配、孤儿检测/重绑定/删除/忽略。

## Change Log

- 2026-01-23: 实现孤儿项目数据管理（检测/展示/重绑定/删除/忽略）与持久化 projectId；状态更新为 review。
- 2026-01-23: 代码评审（Senior Developer Review）：发现 High/Medium 问题，已添加 Review Follow-ups（AI）；状态回退为 in-progress。
- 2026-01-23: 修复 Review Follow-ups（H1/M1/M2/M3/L1）并复审通过；状态更新为 review。

## Senior Developer Review (AI)

**Date:** 2026-01-23  
**Outcome:** Approved  

### Summary

| Metric | Value |
|--------|-------|
| Git vs Story Discrepancies | 0（按用户要求忽略无关改动） |
| HIGH Issues | 0 |
| MEDIUM Issues | 0 |
| LOW Issues | 0 |
| Unit Tests | ✅ `npm -C crewagent-runtime test` passed |
| Lint | ✅ `npm -C crewagent-runtime run lint` passed（存在非阻断 TypeScript 版本提示 warning） |

### Review History

- 2026-01-23: Changes Requested（H1/M1/M2/M3/L1）
- 2026-01-23: Approved（已全部修复并补齐测试覆盖）
