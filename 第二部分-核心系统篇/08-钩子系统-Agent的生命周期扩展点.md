# 第8章：钩子系统 -- Agent 的生命周期扩展点

> **学习目标：** 掌握 26 个生命周期事件和 5 种钩子类型，理解钩子的配置、执行和安全模型。

---

## 8.1 钩子类型与执行模型

钩子（Hook）是 Claude Code 生命周期中用户可自定义的扩展点。通过钩子，用户可以在不修改 Claude Code 源码的前提下，在关键节点注入自定义逻辑 -- 从审批工具调用、修改工具输入，到拦截用户请求、注入额外上下文。

### 五种钩子类型

钩子 Schema 定义模块中定义了四种可持久化的钩子类型，加上仅在运行时存在的 FunctionHook，共五种：

**1. Command 钩子**（`BashCommandHookSchema`）

最常见的钩子类型，执行 Shell 命令。支持选择 Shell 解释器（bash/powershell）、自定义超时、自定义状态消息，以及 `once` 标志（执行一次后自动移除）。

**2. Prompt 钩子**（`PromptHookSchema`）

调用 LLM 评估钩子输入，输出 JSON 响应。支持指定模型（默认使用快速小模型）和 `$ARGUMENTS` 占位符来引用钩子输入。

**3. Agent 钩子**（`AgentHookSchema`）

Agentic 验证器钩子。与 Prompt 钩子类似，但设计用于需要多步推理的验证场景，例如验证单元测试是否通过。

**4. HTTP 钩子**（`HttpHookSchema`）

向指定 URL POST 钩子输入 JSON。支持自定义请求头、环境变量插值（通过 `allowedEnvVars` 白名单控制），适合与外部系统集成。

**5. Function 钩子**

仅在运行时存在的内存钩子，执行 TypeScript 回调函数。无法持久化到配置文件，生命周期与会话绑定。回调函数接收消息数组和可选的中止信号，返回布尔值表示是否成功。

### 同步 vs 异步钩子

Command 钩子支持三种执行模式：

- **同步模式**（默认）：阻塞当前操作，等待钩子完成后根据结果决定是否继续。
- **异步模式**（`async: true`）：在后台运行，不阻塞当前操作。钩子结果对模型不可见。
- **异步唤醒模式**（`asyncRewake: true`）：在后台运行，但当钩子以退出码 2 结束时，注入错误消息唤醒模型继续对话。这暗示 `async` 属性。

异步钩子的实现在后台执行函数中完成。异步唤醒模式的钩子绕过常规的注册表，在完成时通过通知队列注入消息。

---

## 8.2 核心生命周期事件

SDK 核心类型模块定义了完整的 `HOOK_EVENTS` 数组，共 26 个生命周期事件，涵盖工具调用、用户交互、会话管理、子代理、压缩、权限、配置变更等所有关键节点。

以下按功能分组介绍核心事件。

### 工具调用生命周期

**PreToolUse**（工具执行前）

最重要的拦截点。输入是工具调用的参数 JSON。钩子可以通过返回 `decision: "block"` 来阻止工具执行，或通过 `updatedInput` 修改工具的实际输入参数。退出码语义：

- 退出码 0：stdout/stderr 不展示给模型
- 退出码 2：展示 stderr 给模型并阻止工具调用
- 其他退出码：展示 stderr 给用户但不阻止

**PostToolUse**（工具执行后）

输入包含工具调用参数和响应。可以用于审计日志、结果后处理。支持 `updatedMCPToolOutput` 字段来覆盖 MCP 工具的实际输出。

**PostToolUseFailure**（工具执行失败后）

在工具执行因错误、中断或超时而失败时触发。输入包含 `error`、`error_type`、`is_interrupt` 和 `is_timeout` 字段，提供详细的失败诊断。

### 用户交互生命周期

**UserPromptSubmit**（用户提交提示时）

在用户提交消息后、模型处理前触发。这是修改用户输入或注入额外上下文的关键时机。退出码 2 可以完全阻止消息处理并擦除原始提示。

**Notification**（通知发送时）

当系统发送通知时触发。通知类型包括 `permission_prompt`、`idle_prompt`、`auth_success` 等。

### 会话生命周期

**SessionStart**（会话启动）

会话启动时触发，来源包括 `startup`（新启动）、`resume`（恢复会话）、`clear`（清除重置）、`compact`（压缩后重启）。钩子的 stdout 会展示给 Claude。阻塞错误被忽略 -- 会话启动钩子不应阻止会话启动。

**SessionEnd**（会话结束）

会话结束时触发，原因包括 `clear`、`logout`、`prompt_input_exit`、`other`。注意 SessionEnd 钩子有独立的超时限制（默认 1,500ms），因为它们在关闭流程中运行。

**Stop**（助手响应结束前）

在 Claude 即将结束响应前触发。退出码 2 可以将 stderr 注入模型并强制继续对话。这是实现"确保任务完成"逻辑的关键事件。

**StopFailure**（因 API 错误而结束时）

当轮次因 API 错误（限流、认证失败等）而结束时触发，替代 Stop 事件。这是一个即发即忘事件 -- 钩子输出和退出码都被忽略。

### 子代理生命周期

**SubagentStart / SubagentStop**

子代理（Agent 工具调用）启动和结束时触发。输入包含 `agent_id` 和 `agent_type`。SubagentStart 的 stdout 展示给子代理；SubagentStop 的退出码 2 可以让子代理继续运行。

### 压缩生命周期

**PreCompact / PostCompact**

压缩前后触发。PreCompact 的 stdout 会作为自定义压缩指令附加到压缩提示中，允许用户定制摘要行为。退出码 2 可以阻止压缩。PostCompact 接收压缩摘要作为输入。

PreCompact 钩子的处理流程包括：构建钩子输入、执行钩子、提取自定义指令、合并用户指令与钩子指令。

### 其他事件

- **PermissionRequest**：权限对话框显示时触发，钩子可以返回 `decision` 来允许或拒绝。
- **PermissionDenied**：自动模式分类器拒绝工具调用时触发，钩子可以建议重试。
- **Setup**：仓库初始化和维护时触发。
- **ConfigChange**：配置文件变更时触发，钩子可以阻止变更生效。
- **Elicitation / ElicitationResult**：MCP 服务器请求用户输入时触发。
- **CwdChanged / FileChanged**：工作目录变更和文件变更时触发。
- **InstructionsLoaded**：指令文件加载时触发（仅可观测，不支持阻止）。

---

## 8.3 钩子响应协议

钩子的输出不仅是一段 stdout 文本，而是一个结构化的 JSON 响应协议。钩子执行模块中的输出解析函数负责解析和处理这个协议。

### 顶层决策字段

**decision 字段**：`approve` 或 `block`

当钩子返回 `{"decision": "approve"}` 时，工具调用被允许继续；当返回 `{"decision": "block"}` 时，工具调用被阻止，`reason` 字段的值作为阻止原因展示。

在钩子输出处理函数中，当解析到 `decision` 字段时，`approve` 对应允许继续执行，`block` 对应拒绝并附带阻止原因。

### hookSpecificOutput 事件特定输出

`hookSpecificOutput` 字段包含事件特定的结构化响应，通过 `hookEventName` 标识目标事件：

**PreToolUse 特定字段**：

- `permissionDecision`：覆盖权限决策，值为 `allow`、`deny` 或 `ask`
- `permissionDecisionReason`：决策原因
- `updatedInput`：运行时修改工具输入

`updatedInput` 是一个强大的能力 -- 钩子可以在不改变用户意图的前提下，修改实际发送给工具的参数。例如，自动为所有 Bash 命令添加特定前缀，或过滤敏感参数。

**UserPromptSubmit 特定字段**：

- `additionalContext`：注入额外上下文到用户提示中

这个字段允许钩子在用户消息到达模型之前，附加额外的上下文信息，而不修改用户的原始输入。

### additionalContext：注入额外上下文

`additionalContext` 是多个事件支持的通用字段。它将钩子生成的额外信息注入到模型的上下文中，作为系统提醒消息（system reminder）附加。这在以下场景中特别有用：

- 在 `SessionStart` 事件中注入项目状态信息
- 在 `PostToolUse` 事件中添加工具使用的额外说明
- 在 `Setup` 事件中注入环境配置信息

### continue 字段

`continue` 字段控制助手是否应该继续响应。当设为 `false` 时，助手停止生成，`stopReason` 字段可以提供停止原因。这允许钩子在检测到特定条件时强制助手停止。

---

## 8.4 钩子配置与安全

### 配置验证

钩子配置通过 Zod Schema 进行严格验证。钩子 Schema 模块定义了完整的类型系统：

- `HookCommandSchema`：四种可持久化钩子的 discriminated union
- `HookMatcherSchema`：匹配器配置，包含 `matcher` 字符串模式和 `hooks` 数组
- `HooksSchema`：顶层配置，使用 `partialRecord` 映射事件到匹配器数组

匹配器使用权限规则语法进行过滤（`if` 条件字段）。例如 `"Bash(git *)"` 只在 Bash 工具调用以 `git` 开头时触发钩子。该条件字段是一个可选的字符串，支持权限规则语法模式匹配。

### 钩子来源与优先级

钩子来自多个配置源。`getAllHooks` 函数展示了收集流程：首先检查是否限制为仅托管钩子，如果没有限制，则从 userSettings、projectSettings、localSettings 三个来源收集钩子，最后还会获取会话钩子。

`sortMatchersByPriority` 函数定义了优先级排序。优先级基于配置源常量数组，其顺序为 localSettings、projectSettings、userSettings。

完整的优先级顺序为：

1. **userSettings**（用户全局设置，`~/.claude/settings.json`）-- 最高优先级
2. **projectSettings**（项目设置，`.claude/settings.json`）
3. **localSettings**（本地设置，`.claude/settings.local.json`）
4. **pluginHook**（插件钩子）-- 优先级较低（代码中赋值 999）
5. **builtinHook**（内置钩子）-- 同样为低优先级
6. **sessionHook**（会话钩子）-- 运行时临时钩子

Plugin 钩子和内置钩子在排序中被赋予最低的优先级数值，确保它们不会覆盖用户配置的钩子。

### 紧急禁用开关

Claude Code 提供了多层安全开关来应对钩子相关风险。

**全局禁用**：当 `policySettings`（企业策略配置）中设置 `disableAllHooks: true` 时，所有钩子（包括托管钩子）都被禁用。这是终极紧急开关。

**限制为仅托管钩子**：检查策略设置中的 `allowManagedHooksOnly` 字段，当启用此模式时，用户/项目/本地配置中的钩子被全部屏蔽，只允许企业管理员通过策略配置部署的钩子运行。

**工作区信任检查**：所有钩子在交互模式下都要求工作区已被信任。该检查函数判断当前是否为非交互式会话（SDK 模式信任隐式授予）以及用户是否已接受信任对话框。

所有钩子在交互模式下都要求工作区已被信任。这是一个纵深防御（defense-in-depth）措施 -- 钩子配置存储在 `.claude/settings.json` 中，在不受信任的工作区中执行这些命令是不安全的。

`executeHooks` 函数将这些检查串联起来，形成完整的安全门禁：首先检查全局禁用开关，然后检查简单模式标志，最后检查工作区信任状态，全部通过后才正常执行钩子。

### 会话钩子的特殊设计

会话钩子使用 `Map` 而非 `Record` 来存储状态，这是一个精心的性能设计。

在并行工作流中，N 个 Agent 可能在同一个同步时钟周期内注册钩子。使用 `Record` 的展开操作会导致 O(N^2) 的总开销，而 `Map.set()` 是 O(1)，并且通过返回未修改的 `prev` 引用避免触发约 30 个 store 监听器。

---

## 实战练习

**练习 1：配置安全审查钩子**

编写一个 PreToolUse 钩子配置，要求：
- 仅拦截 `Write` 工具调用
- 检查目标路径是否在允许的目录范围内
- 如果路径不合法，以退出码 2 阻止操作并显示原因

提示：钩子输入通过 stdin 以 JSON 格式传入，包含 `tool_name` 和 `tool_input` 字段。

**练习 2：会话上下文注入**

设计一个 SessionStart 钩子，在每次会话启动时：
- 读取项目的 `package.json` 获取依赖列表
- 将关键依赖信息作为 `additionalContext` 注入

思考：如何处理 `package.json` 不存在的情况？

**练习 3：钩子优先级分析**

给定以下钩子配置（假设同一事件）：
- `~/.claude/settings.json` 中定义了钩子 A
- `.claude/settings.json` 中定义了钩子 B
- 插件注册了钩子 C
- 运行时通过 `addFunctionHook` 注册了钩子 D

请分析：当事件触发时，这些钩子的执行顺序是什么？如果钩子 A 阻止了操作，钩子 B/C/D 是否仍会执行？

---

## 关键要点

1. **五种钩子类型**：command（Shell 命令）、prompt（LLM 评估）、agent（Agentic 验证）、http（HTTP 调用）、function（运行时回调），前四种可持久化，最后一种仅在会话内存中存在。
2. **26 个生命周期事件**：覆盖工具调用、用户交互、会话管理、子代理、压缩、权限、配置变更等完整的 Agent 生命周期。
3. **结构化响应协议**：钩子输出是 JSON，包含 `decision`（审批/阻止）、`updatedInput`（修改工具输入）、`additionalContext`（注入上下文）等字段。
4. **优先级排序**：userSettings > projectSettings > localSettings > pluginHook > builtinHook > sessionHook，用户配置始终具有最高优先级。
5. **多层安全机制**：全局禁用开关（`disableAllHooks`）、仅托管钩子模式（`allowManagedHooksOnly`）、工作区信任检查，三层安全门禁逐级过滤。
6. **异步钩子**：`async` 和 `asyncRewake` 模式允许钩子在后台运行，后者在出错时可以唤醒模型，适合长时间运行的监控任务。
7. **会话钩子的 Map 设计**：使用 `Map` 而非 `Record` 存储，在高并发场景下将 O(N^2) 的开销降为 O(1)。
