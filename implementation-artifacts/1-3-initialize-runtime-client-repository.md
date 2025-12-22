# Story 1.3: Initialize Runtime Client Repository

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a Developer,  
I want to initialize `crewagent-runtime` using the electron-vite starter (React + TypeScript),  
so that I can begin building the Desktop Runtime Client.

## Acceptance Criteria

1. **Given** Node.js is installed  
   **When** I run `npm create electron-vite@latest crewagent-runtime` and select the **React** template  
   **Then** the Electron + Vite + React project is created with TypeScript.
2. **Given** the project is created  
   **When** I run `npm run dev` in `crewagent-runtime/`  
   **Then** the Electron app starts successfully.

## Tasks / Subtasks

- [x] 1) 生成 electron-vite 项目（AC: 1）
  - [x] 确认当前目录下不存在同名目录 `crewagent-runtime/`（避免覆盖）
  - [x] 完成 Electron + Vite + React + TS 工程初始化（基于 electron-vite/react-ts 模板）
- [x] 2) 最小运行验证（AC: 2）
  - [x] `cd crewagent-runtime && npm install`
  - [x] `npm run dev` 启动成功（Vite dev server 可访问；electron 主/预加载构建产物生成）
- [ ] 3) 对齐架构目录骨架（建议，便于后续 Story 直接落点）
  - [ ] 创建/调整目录：`src/main/`, `src/renderer/`, `src/preload/`, `src/shared/`（若模板已有则保持）

### Review Follow-ups (AI)

- [x] [AI-Review][LOW] 增加 Node 版本约束/说明（例如 `.nvmrc` 或 `package.json#engines`）
- [x] [AI-Review][LOW] 补充 `README.md`：明确 dev/build/test 命令与端口说明

## Dev Notes

### 架构/规范（必须遵循）

- Runtime：Electron + Vite + React + TypeScript
- 目录结构目标参考 `_bmad-output/architecture.md` 的 “Runtime Client (`crewagent-runtime/`)”

### References

- `_bmad-output/epics.md`（Epic 1 / Story 1.3）
- `_bmad-output/architecture.md`（Runtime Client 目录结构；MCP Drivers / Sandboxed FS）

## Dev Agent Record

### Agent Model Used

GPT-5.2 (Codex CLI)

### Debug Log References

### Completion Notes List

- `crewagent-runtime/` 已初始化（Electron + Vite + React + TypeScript）
- 已验证 `npm run dev` 可启动（Vite dev server 可访问；electron 主/预加载产物生成）
- 已补充 Node 版本约束（`.nvmrc` + `package.json#engines`）与启动说明（README）

### File List

- `crewagent-runtime/`（runtime 工程）
- `crewagent-runtime/.nvmrc`
- `crewagent-runtime/package.json`
- `crewagent-runtime/vite.config.ts`
- `crewagent-runtime/electron/`
- `crewagent-runtime/src/`
- `crewagent-runtime/public/`
- `crewagent-runtime/README.md`
