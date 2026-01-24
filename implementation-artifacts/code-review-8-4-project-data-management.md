# Code Review: Story 8-4 – Project Data Management & Orphan Recovery

**Date:** 2026-01-23  
**Reviewer:** AI Code Reviewer (BMAD Method)  
**Story File:** `8-4-project-data-management.md`

---

## 范围说明

- 按用户要求，本次 review 不评估工作区其他无关改动；仅对 Story 8-4 的实现与 AC 做对照审计。

---

## Summary

| Metric | Value |
|--------|-------|
| HIGH Issues | 0 |
| MEDIUM Issues | 0 |
| LOW Issues | 0 |
| Unit Tests | ✅ `npm -C crewagent-runtime test` passed |
| Lint | ✅ `npm -C crewagent-runtime run lint` passed（存在非阻断的 TypeScript 版本提示 warning） |

---

## Resolved Issues (from prior review)

- ✅ **H1 (legacy 先移动后打开 → 不可恢复)**：加入强制重绑定（确认对话框）+ 冲突数据自动标记为 Detached orphan，可在 Settings 中继续管理：`crewagent-runtime/electron/main.ts:798`、`crewagent-runtime/electron/stores/runtimeStore.ts:2027`
- ✅ **M1 (同步目录大小遍历卡主进程)**：目录大小统计改为异步并缓存（会话内避免重复扫描）：`crewagent-runtime/electron/stores/runtimeStore.ts:1989`
- ✅ **M2 (缺少 legacy 迁移测试)**：补齐 legacy 无 `projectId` + 移动后再打开 + 强制重绑定 + run history 保留测试：`crewagent-runtime/electron/stores/runtimeStore.test.ts:678`
- ✅ **M3 (Windows 外接盘识别/Ignore 可用性)**：Windows 增加非系统盘启发式 removable 识别；UI 对所有 orphan 都提供 “Ignore (Until Restart)”：`crewagent-runtime/electron/stores/runtimeStore.ts:1905`、`crewagent-runtime/src/components/OrphanProjectList.tsx:146`
- ✅ **L1 (Start 页不刷新/失败无提示)**：Start 页增加 focus 刷新 + 错误提示与重试：`crewagent-runtime/src/pages/StartPage/StartPage.tsx:25`

---

## Acceptance Criteria 验证

| AC | Status | Evidence |
|----|--------|----------|
| AC-1 孤儿检测 | ✅ | `countOrphanProjects()` / `detectOrphanProjects()`：`crewagent-runtime/electron/stores/runtimeStore.ts:1966` |
| AC-2 展示信息 | ✅ | Settings “Project Data” + `OrphanProjectList`：`crewagent-runtime/src/components/OrphanProjectList.tsx:1` |
| AC-3 重绑定 | ✅ | 支持普通重绑定 + 冲突时强制重绑定（确认）并保留历史数据：`crewagent-runtime/electron/main.ts:798`、`crewagent-runtime/electron/stores/runtimeStore.ts:2027` |
| AC-4 删除 | ✅ | `deleteOrphanData()`：`crewagent-runtime/electron/stores/runtimeStore.ts:2137` |
| AC-5 忽略外接设备 | ✅ | removable 情况显示 “Ignore (External)”；非 removable 仍可 “Ignore (Until Restart)”：`crewagent-runtime/src/components/OrphanProjectList.tsx:146` |
| AC-6 移动自动匹配 | ✅ | 新项目 UUID 持久化自动匹配；legacy 场景可通过强制重绑定恢复：`crewagent-runtime/electron/stores/runtimeStore.test.ts:499`、`crewagent-runtime/electron/stores/runtimeStore.test.ts:678` |
