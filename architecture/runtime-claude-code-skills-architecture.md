# Runtime Claude Code Skills Architecture

> Parent Architecture: `_bmad-output/architecture.md`  
> Parent PRD: `_bmad-output/prd.md`  
> Sub-PRD: `_bmad-output/prd-runtime-claude-code-skills.md`

---

## 1. Context and Constraints

本架构增量定义 Runtime 如何消费 Claude Code 风格 skill，而不是如何安装或分发 skill。

已知约束：

1. skill source 可能来自 Runtime 私有状态下的 `user-global` 目录、package adapter 输出或宿主会话输入，但都必须位于 Runtime 可访问路径下；
2. 当前 Runtime 已有 `fs.*`、`terminal.run`、`python.run`、bundled Node/npm 内部服务，以及 context builder / tool loop；
3. 当前 tool loop 在工具调用后会继续迭代，但不会自动重建下一轮的 system messages 或 tools；
4. package v1.2 尚未完成，因此首版必须采用 runtime-first 入口；
5. skill 的触发激活由主模型决定，不使用前置独立 router。

---

## 2. Core Decisions

### AD-SKL-01: Runtime-first, package-second

- **Decision**: 先实现 runtime-first skill attachment，再由 package v1.2 通过 adapter 对接。
- **Rationale**: 不把 runtime skill 运行机制绑定到 package schema 升级节奏。

### AD-SKL-01B: Layered source composition before discovery

- **Decision**: Runtime 先把 `user-global`、`package`、`session` 三层 skill source 组成一个有效列表，再复用同一条 discovery / registry / activation 管线。
- **Rationale**: 既能提供默认可复用的用户级 skill，又避免把全局 skill 机制散落到 chat、agent、run 各入口。
- **Implementation note**: Story 13.7 先落 `user-global + session` 组成层，并预留 package 扩展点；`package` 层在 Story 13.5 接入。

### AD-SKL-02: LLM-triggered activation, not pre-routing

- **Decision**: 模型通过内部 skill 控制工具自行决定是否激活 skill。
- **Rationale**: 符合 Claude Code skill 的主模型自决模式，也避免再引入一层路由器。

### AD-SKL-03: Lazy instruction injection

- **Decision**: 首轮只注入 skill registry；skill body 只在 activation 后注入。
- **Rationale**: 控制上下文成本，避免所有 skill 说明预注入。

### AD-SKL-04: Control-tool-driven context mutation

- **Decision**: skill activation 不返回普通业务结果，而返回上下文变更结果，驱动下一轮请求重建 system messages 和 tools。
- **Rationale**: 现有 chat/run loop 都支持工具后继续迭代，但不支持单靠普通 tool result 修改后续 prompt 结构。

### AD-SKL-05: Effective-agent-scoped visibility

- **Decision**: skill 可见性和激活态始终绑定当前 `effectiveAgentId`。
- **Rationale**: 与现有 persona/tools 选择方式保持一致，避免 skill 成为“全局会话能力”。

---

## 3. Claude Code Skill Model

本方案兼容的 Claude Code skill 最小模型为：

- 一个 skill 根目录；
- 入口文件 `SKILL.md`；
- `SKILL.md` 中的核心信息：
  - `description`
  - `allowed-tools`（可选）
  - `disable-model-invocation`（可选）
- supporting files 通过 `SKILL.md` 中的相对链接组织。

兼容边界：

- **兼容项**
  - `SKILL.md`
  - `description`
  - supporting files
  - `allowed-tools`
  - `disable-model-invocation`
- **Runtime 扩展项**
  - `attachedSkills`
  - `@skill/<skillId>/...`
  - `ContextMutationResult`
  - `python.run module`
  - `node.run`
  - `npm.install`

本方案明确是 Claude Code skill consumption compatibility，而不是完整 Claude Runtime 复刻。

---

## 4. Runtime Skill Model

Runtime 内部统一 skill 模型至少包含：

```ts
interface AttachedSkillSource {
  source: string
  agentIds?: string[]
}

type SkillSourceLayer = 'user-global' | 'package' | 'session'

interface EffectiveSkillSource extends AttachedSkillSource {
  layer: SkillSourceLayer
  sourceKey: string
}

interface ResolvedSkill {
  skillId: string
  displayName: string
  description: string
  rootPath: string
  skillMdPath: string
  sourceLayer: SkillSourceLayer
  sourceKey: string
  instructions: string
  supportingFiles: Array<{
    relPath: string
    kind: 'text' | 'script' | 'other'
  }>
  allowedTools?: string[]
  disableModelInvocation: boolean
  agentIds?: string[]
}

interface ActiveSkillState {
  skillId: string
  injectedSystemBlocks: string[]
  narrowedToolNames: string[]
}
```

每个 skill 还对应一个 Runtime 私有执行空间：

```ts
interface SkillWorkspace {
  skillId: string
  sourceRoot: string
  workspaceRoot: string
  nodeModulesRoot: string
  packageJsonPath?: string
}
```

会话侧显式输入保持为：

```ts
attachedSkills: Array<{ source: string; agentIds?: string[] }>
```

在此之上，Runtime 内部新增统一来源组成结果：

```ts
effectiveSkillSources: Array<{
  source: string
  agentIds?: string[]
  layer: 'user-global' | 'package' | 'session'
  sourceKey: string
}>
```

`user-global` 根目录固定在 Runtime 私有状态下：

```text
runtime-store/skills/global/
```

并通过 `@state/skills/global/...` 暴露给后续路径解析与诊断展示。

---

## 5. Discovery and Parsing Flow

### 5.1 SkillSourceProvider

新增主进程服务：`SkillSourceProvider`

职责：

- 扫描 Runtime 私有状态下的 `@state/skills/global`
- 在 Story 13.7 中接收会话 `attachedSkills`
- 为 Story 13.5 预留 package adapter 输出接入点
- 给每个 source 打上层级标签
- 在 Story 13.7 中按 `session > user-global` 组成有效来源列表
- 对重复 source 与冲突场景产出结构化诊断

### 5.2 SkillRegistryService

新增主进程服务：`SkillRegistryService`

职责：

- 接收 `effectiveSkillSources`
- 校验路径是否在 Runtime 允许访问的边界内
- 解析 skill 目录或 `SKILL.md`
- 提取 registry 信息
- 生成 supporting file 索引
- 按 agent 过滤可见 skill

### 5.3 Parsing Rules

- `source` 指向目录时，默认查找 `<dir>/SKILL.md`
- `source` 指向文件时，只接受 `SKILL.md`
- `name` 缺失时，以目录名或文件所在目录名作为 `skillId`
- `description` 缺失视为不合格 skill
- supporting files 只索引 `SKILL.md` 中显式相对链接的本地文件
- 索引阶段不全量读取 supporting file 内容

### 5.4 Runtime Alias

新增路径别名：

- `@skill/<skillId>/...`

用途：

- 支持 `fs.*`
- 支持 `python.run file=...`
- 支持日志与调试视图里的稳定路径展示

---

## 6. Prompt Injection Strategy

### 6.1 Initial Prompt

初始 system prompt 只增加一个 `Skill Registry` 块，内容包含：

- 当前 effective agent 可见的 skill 列表
- 每个 skill 的一行 description
- 主要 supporting file 名称摘要
- 是否允许模型自行激活

不在初始 prompt 中注入：

- 完整 `SKILL.md` 正文
- 全量 supporting file 内容

### 6.2 Activated Prompt

当 skill 被激活后，下一轮请求插入两个 system blocks：

1. `Activated Skill`
   - 完整 skill instructions
   - skill root alias
   - 工具收窄说明
2. `Skill Resources`
   - supporting file 索引
   - 明确提示可用 `skill.load_resource` 按需读取

### 6.3 Builder Integration

- `buildChatContextMessages` 已支持 `extraSystemMessages`，继续复用；
- `buildAgentContextMessages` 需要补齐 `extraSystemMessages`；
- run mode 的 execution engine 需要在每轮请求前能接受动态 skill system blocks。

---

## 7. Skill Activation Protocol

### 7.1 Internal Tools

新增两个内部工具：

- `skill.activate`
- `skill.load_resource`

#### `skill.activate`

```ts
interface SkillActivateArgs {
  skillId: string
}
```

约束：

- 仅允许选择当前 registry 中、且 `disable-model-invocation !== true` 的 skill
- 工具 schema 中的可选值按当前可见 skill 动态收窄

#### `skill.load_resource`

```ts
interface SkillLoadResourceArgs {
  skillId: string
  relPath: string
}
```

约束：

- 只能读取当前 active skill
- 只能读取 `SKILL.md` 显式引用过的文本资源
- 不能跳出 skill 根目录

### 7.2 ContextMutationResult

`skill.activate` 成功后返回：

```ts
interface ContextMutationResult {
  ok: true
  mutationType: 'skill_activation'
  skillId: string
  injectedSystemBlocks: string[]
  narrowedToolNames: string[]
}
```

这不是普通 tool result，而是 tool loop 的控制信号。

### 7.3 Loop Changes

chat loop 和 run loop 都必须支持：

1. 执行 `skill.activate`
2. 收到 `ContextMutationResult`
3. 重算当前 active skill state
4. 重建下一轮 `system messages`
5. 依据收窄结果重算 visible tools
6. 再继续同一轮 tool loop

activation 只在当前 loop 内有效，不写入 conversation store，也不写入 workflow state。

---

## 8. Supporting File Loading Model

- supporting file 不在 discovery 或 activation 阶段自动全量读取；
- supporting file 读取通过 `skill.load_resource` 进行；
- 读取结果进入上下文时与普通 tool result 一致；
- supporting file 默认分三类：
  - `text`
  - `script`
  - `other`
- 只有 `text` 默认允许通过 `skill.load_resource` 读取；
- `script` 主要用于 `python.run` / `node.run` / `terminal.run` / `fs.read` 配合执行。

---

## 9. Tool Visibility and Narrowing

### 9.1 Base Rule

最终 visible tools 的计算公式固定为：

`base tools allowed by runtime/agent policy`
`∩ tools mapped from Claude allowed-tools`
`∩ internal tools available under current skill state`

### 9.2 Allowed-Tools Mapping

Claude Code `allowed-tools` 在 Runtime 中按稳定映射收窄：

- `Read` / `Grep` / `Glob` / `LS`
  - 映射到 `fs.read` / `fs.search` / `fs.list`
- `Bash`
  - 映射到 `terminal.run`，以及同属本地执行桥的 `node.run` / `npm.install`
- `Edit` / `Write`
  - 只有在当前 agent/system 已经允许写类文件工具时才保留

无法映射的项：

- 记录 warning
- 不报致命错误
- 不扩权补齐

---

## 10. Python / Node / Terminal Execution Bridge

### 10.1 Python

`python.run` 升级为三选一：

- `code`
- `file`
- `module`

其中：

- `module` 映射到 `python -m <module>`
- `file` 路径解析支持 `@skill/<skillId>/...`

### 10.2 Node.js

新增一等执行工具：

```ts
interface NodeRunArgs {
  code?: string
  file?: string
  module?: string
  args?: string[]
  cwd?: string
}
```

约束：

- `node.run` 为三选一：`code | file | module`
- `file` 路径解析支持 `@skill/<skillId>/...`
- `module` 通过 Runtime bundled Node.js 执行，不依赖宿主 PATH 中的 `node`
- JS/Node skill 的主路径应优先使用 `node.run`，而不是长期依赖 `terminal.run node ...`

### 10.3 npm Dependency Install

新增一等依赖工具：

```ts
interface NpmInstallArgs {
  skillId: string
  packages?: string[]
  packageJson?: string
  dev?: boolean
}
```

安装策略固定为：

1. 每个 skill 在 RuntimeStore 下拥有独立 `SkillWorkspace`
2. `npm.install` 只写入该 workspace，不修改原 skill source 目录
3. workspace 会维护一个刷新式 `source-shadow/`，并在安装前复用或复制 skill 根目录中的 `package.json`
4. `node.run` 执行时默认以该 workspace 作为 npm 依赖解析上下文
5. Node/npm 子进程沿用 Story 5.19 的最小执行环境基线，而不是直接继承完整宿主 `process.env`

自动安装策略：

- 当前实现中，当 active skill 的 `node.run` 因缺失 npm 包失败时：
  1. Runtime 解析缺失包名
  2. 调用 `npm.install`
  3. 安装成功后自动重试一次 `node.run`
- 安装失败时返回结构化错误，不进行无限重试

### 10.4 Dependencies

- Python 第三方依赖继续复用现有 auto-install 机制；
- npm 依赖通过 `npm.install` 和 auto-install-on-missing 机制处理；
- 系统级依赖不自动安装，只做存在性检测和结构化报错；
- 错误需保留稳定 code，便于 skill 级审计。

### 10.5 Terminal

- 若 skill `allowed-tools` 映射允许 `Bash`，且基础 policy 已允许本地执行能力，则模型可调用 `terminal.run`、`shell.exec`，以及同属执行桥的 `node.run` / `npm.install`；
- skill 不新增新的 shell 能力，只决定当前 skill 是否允许继续使用现有终端能力。
- 对 JS/Node skill，优先使用 `node.run` / `npm.install`；仅在确实需要 shell 语义时再退回 `terminal.run` 或 `shell.exec`。

---

## 11. Package v1.2 Adapter

当 Epic 11.1 落地后，新增一个轻量 adapter：

```ts
interface PackageSkillImportAdapter {
  toAttachedSkills(pkg: PackageDefinition, effectiveAgentId: string): AttachedSkillSource[]
}
```

职责：

- 读取 `agents.skills.imports`
- 转换为供 `SkillSourceProvider` 使用的 package-layer `AttachedSkillSource`
- 保留 `agentIds` 绑定信息

不负责：

- 解析 `SKILL.md`
- 注入 prompt
- 激活 skill
- 资源读取
- 工具收窄

---

## 12. Observability and Error Handling

推荐事件：

- `skill.discovered`
- `skill.discovery_failed`
- `skill.activated`
- `skill.activation_failed`
- `skill.resource_loaded`
- `skill.resource_load_failed`
- `skill.tools_narrowed`

所有 skill 审计事件至少应携带：

- `packageId`
- `workflowId`
- `runId`
- `agentId`
- `ts`

当 run mode 的 `effectiveAgentId` 发生变化时，已有 activated skill state 必须先失效，再进入下一轮请求构建，避免 skill 在 agent 边界间泄漏。

常见错误码：

- `SKILL_NOT_FOUND`
- `SKILL_INVALID`
- `SKILL_DESCRIPTION_MISSING`
- `SKILL_ACTIVATION_NOT_ALLOWED`
- `SKILL_RESOURCE_OUT_OF_SCOPE`
- `SKILL_RESOURCE_NOT_DECLARED`
- `SKILL_ALLOWED_TOOLS_INVALID`
- `SKILL_SYSTEM_DEPENDENCY_MISSING`

---

## 13. Open Questions Removed / Final Decisions

以下事项在本架构中已锁定，不再留给实现阶段决策：

- 首版采用 runtime-first，不等待 package spec 1.2；
- skill 激活由主模型触发，不增加独立前置 router；
- skill body 懒加载，不做初始全量注入；
- 通过控制工具返回 `ContextMutationResult`，而不是依赖普通 tool result 改写上下文；
- activation 只在当前 loop 内有效，不持久化；
- JS skill 作为一等执行场景，必须通过 `node.run` 与 `npm.install` 落地，而不是长期依赖 `terminal.run node ...` 作为主路径；
- package v1.2 只做 adapter，对接同一 runtime skill 机制；
- 本能力是 Claude Code skill consumption compatibility，不是完整 Claude runtime 复刻。
