# PRD: Runtime Claude Code Skills Requirements

> **Parent Document**: [Product Requirements Document (CrewAgent)](prd.md)
> **Scope Boundary**: 本文覆盖 Runtime 如何兼容消费 Claude Code 风格 `SKILL.md` skill，包括 Runtime 私有状态下的用户级全局 skill、会话临时挂载 skill，以及后续 package `skills.imports` 接入后的统一运行时承载；本文不包含 Claude marketplace/plugin 安装，也不展开 package spec v1.2 的 schema 细节。

## 1. Problem Statement

当前 Runtime 已具备 `fs / terminal / python` 等内建工具，但缺少一套标准机制来消费 Claude Code 风格的本地 skill 目录，导致以下问题：

1. 现有能力只能让模型“把 skill 当普通文件手动读”，缺少稳定的 skill 发现、注入、激活和资源访问协议；
2. skill 的说明文本与 supporting files 无法按需进入上下文，容易出现上下文浪费或行为不一致；
3. 缺少 `allowed-tools`、`disable-model-invocation` 等 Claude Code 风格控制语义，skill 无法对工具可见性形成细粒度约束；
4. 后续即使 package v1.2 支持 `skills.imports`，如果 runtime 本身没有独立 skill 消费链路，仍无法把 package 中指定的 skill 稳定接入。

本需求的目标是先把 Runtime 自身的 Claude Code skill 消费能力定义清楚，再让 package v1.2 通过适配层对接进同一运行机制。

---

## 2. Goals / Non-Goals

### Goals

- 支持 Runtime 在可访问路径下发现 Claude Code 规范的 `SKILL.md` skill；
- 支持 Runtime 以 `user-global / package / session` 三层来源组成有效 skill source 列表，并按确定性优先级解析；
- 支持将 skill registry 注入 system prompt，供主模型自主决策是否激活；
- 支持 LLM 在对话中触发 skill 激活，并让下一轮请求拿到完整 skill instructions；
- 支持按需读取 skill supporting files，而不是全量预注入；
- 支持 `allowed-tools` 对现有工具可见性进行收窄；
- 支持 skill 与现有 `fs / terminal / python` 能力协作；
- 为后续 package v1.2 `skills.imports` 提供统一运行时承载点。

### Non-Goals

- 不实现 Claude marketplace / plugin 下载与安装；
- 不支持 LLM 自行声明新的宿主路径或远程下载 skill；
- 不在首版中把 skill 自动转译为一组独立强类型工具 schema；
- 不做 conversation 级持久 skill activation；
- 不在本 PRD 中定义 package spec v1.2 的详细 schema。

---

## 3. User Roles

- **Runtime User**
  - 在 chat / agent / run 模式中与主模型交互，希望模型能主动识别和使用已挂载的 skill。
- **Package Author**
  - 后续希望通过 package v1.2 把 skill source 接入 runtime，但不要求 package 自己重实现 skill 运行机制。
- **Runtime Developer**
  - 负责实现 skill discovery、prompt injection、activation、resource loading、tool narrowing 和审计链路。

---

## 4. Functional Requirements

### FR-SKL-01: Skill Discovery and Parsing

- Runtime 必须支持三层 skill source：
  - `user-global`：Runtime 私有状态下的用户级 skill 根目录，默认对所有项目可见；
  - `package`：由 package `skills.imports` 适配得到的 skill source；
  - `session`：宿主在当前 chat / agent / run 请求中显式传入的 skill source 列表。
- Runtime 必须按 `session > package > user-global` 的确定性优先级组成有效 skill source 列表，再在受控可访问范围内解析 skill。
- Runtime 必须对重复 source 和冲突 `skillId` 产出结构化诊断，避免静默覆盖。
- 支持两种入口：
  - 指向 skill 目录；
  - 直接指向 `SKILL.md` 文件。
- Runtime 必须将 skill 解析为统一内部模型，至少包含：
  - `skillId`
  - `displayName`
  - `description`
  - `rootPath`
  - `supportingFiles`
  - `allowedTools`
  - `disableModelInvocation`
- `name` 缺失时可回退为目录名，但 `description` 必须在 registry 阶段可用。

### FR-SKL-02: Skill Registry Injection

- Runtime 必须在初始 system prompt 中注入 skill registry。
- Skill registry 至少包含：
  - skill 名称或 id
  - 一句话 description
  - 主要 supporting files 摘要
  - 是否允许模型自行激活
- 首轮注入时不得把完整 `SKILL.md` 正文全量塞入 system prompt。

### FR-SKL-03: LLM-Driven Activation

- Skill 激活必须由主模型在对话过程中自主触发，不依赖前置 router。
- Runtime 必须为模型提供一个可调用的 skill 激活协议。
- 激活成功后，下一轮 LLM 请求必须拿到完整 skill instructions。
- skill 激活仅在当前 tool loop 内有效，不要求持久化到会话存储。

### FR-SKL-04: Supporting File Access

- Runtime 必须支持按需读取 supporting files。
- supporting files 只能从当前 active skill 范围内读取，且必须受 skill 根目录边界限制。
- supporting files 不得在 skill 激活时自动全量注入。
- 文本 supporting files 的读取结果必须作为普通工具结果进入上下文，便于模型后续推理。

### FR-SKL-05: Allowed-Tools Narrowing

- Runtime 必须支持解析 Claude Code skill 的 `allowed-tools` 控制语义。
- `allowed-tools` 只能对当前 agent/system 已有工具进行收窄，不能扩权。
- skill 激活后，最终可见工具集合必须是：
  - 当前基础 tool policy 允许的工具
  - 与 skill 映射兼容的工具
  - 当前 active skill 明确允许的工具

### FR-SKL-06: Runtime Execution Bridge

- Skill 被激活后，模型必须能通过现有 runtime 工具使用 skill 及其 supporting scripts。
- Runtime 必须支持 supporting scripts 与 `python.run` / `node.run` / `terminal.run` 的协作。
- 为兼容模块入口：
  - `python.run` 必须支持 `module` 形式；
  - `node.run` 必须支持 `code`、`file`、`module` 形式。
- Runtime 必须支持通过 `npm.install` 为 skill 自动安装 npm 依赖，并在允许自动安装时执行“安装后重试”。
- npm 依赖安装不得直接污染原始 skill source 目录，必须落在 Runtime 控制的隔离执行空间内。
- 对 Python 第三方依赖仍使用现有 auto-install 机制；对 npm 依赖使用 `npm.install` / 自动安装；对系统依赖只做显式检查与结构化错误返回。

### FR-SKL-07: Package Adapter Compatibility

- 后续 package v1.2 必须能够把 `skills.imports` 适配到与 runtime-first skill 相同的内部 skill source 结构。
- package 适配层只负责把 package 中声明的 skill source 交给 runtime skill 管线，不得复制 discovery / activation / resource loading 逻辑。

### FR-SKL-08: Audit and Diagnostics

- Runtime 必须记录 skill 生命周期关键事件，至少包括：
  - discovery
  - activation
  - resource load
  - error
- 日志必须可区分：
  - skill 发现成功/失败
  - skill 激活成功/失败
  - resource 读取成功/失败
  - 工具收窄后的可见集合

---

## 5. Non-Functional Requirements

### NFR-SKL-01: Context Efficiency

- 初始 prompt 注入必须以 registry 形式为主，避免 skill body 全量灌入导致上下文膨胀。
- supporting files 只在需要时按需加载。

### NFR-SKL-02: Deterministic Boundaries

- skill 可访问边界必须由宿主和 runtime 明确决定，不能由模型临场扩展。
- tool narrowing 必须基于稳定映射规则，不允许模糊推断扩权。

### NFR-SKL-03: No Privilege Escalation

- skill 不得绕过 agent/system tool policy；
- skill 不得突破 Runtime 文件访问边界；
- skill supporting files 不得跳出当前 skill 根目录。

### NFR-SKL-04: Graceful Failure

- `SKILL.md` 缺失、frontmatter 非法、resource 越界、allowed-tools 不可映射、系统依赖缺失等场景必须给出结构化错误；
- 错误发生后，对话仍应能回到基础模型能力，而不是直接使整个会话失效。

---

## 6. Security / Boundaries

- skill source 必须位于 runtime 可访问范围内，可由 Runtime 自身管理、package adapter 提供或宿主传入，但都不由 LLM 自行声明新路径；
- `user-global` skill 根目录必须由 Runtime 自身管理在私有状态目录下，不能由模型动态改写来源层级；
- supporting files 必须按 skill 根目录做边界校验；
- `allowed-tools` 仅做收窄，不做扩权；
- `disable-model-invocation: true` 的 skill 不得允许模型自主激活；
- skill 激活态不跨 agent 传播，不跨 future subworkflow `callStack` 传播；
- skill resource 读取与脚本执行都必须纳入现有审计与沙箱体系。

---

## 7. Observability / Audit

建议至少记录以下事件：

- `skill.discovered`
- `skill.discovery_failed`
- `skill.activated`
- `skill.activation_failed`
- `skill.resource_loaded`
- `skill.resource_load_failed`
- `skill.tools_narrowed`

每条事件至少带：

- `packageId`（如有）
- `workflowId`
- `runId`
- `agentId`
- `skillId`
- `source`
- `timestamp`

---

## 8. Compatibility / Migration

- 本能力首先服务于 Runtime 自身管理的 `user-global` 与 `session` skill source，不以 package spec v1.2 为前置；
- package v1.2 落地后，只新增 adapter，把 `skills.imports` 映射到同一 skill source 组成层；
- Epic 11 继续负责 package-bound skills 与 spec 升级；
- Epic 13 负责 runtime-first Claude Code skill consumption compatibility；
- 旧的仅靠 `fs.read` 手动读取 skill 目录的模式仍可作为兼容 fallback，但不再是主路径。

---

## 9. Acceptance Summary

本能力完成时，必须满足以下最小闭环：

1. Runtime 可将 `user-global`、`package`、`session` skill source 组成一个有效来源集合，并以确定性优先级解析；
2. 主模型在首轮能从 system prompt 中看到 skill registry；
3. 主模型可在对话中自行触发 skill activation；
4. skill 激活后，下一轮请求能拿到完整 skill instructions；
5. 模型可按需读取 supporting files，而不是一次性预注入；
6. skill 激活后工具集合会被收窄，而不是扩权；
7. 模型可通过现有 `fs / terminal / python` 以及新增的 `node.run` / `npm.install` 配合 skill 完成任务；
8. 后续 package v1.2 仅需接入 adapter 即可复用这条运行机制。

---

## 10. Notes

- 本 PRD 面向“Claude Code skill consumption compatibility”，不是对 Claude Runtime 的完整复刻。
- 本 PRD 不定义具体实现文件与类名，但要求后续架构文档锁定接口与运行时控制协议。
