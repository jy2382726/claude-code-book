# 第14章：Plan 模式与结构化工作流

> "Plans are nothing; planning is everything." -- Dwight D. Eisenhower

**学习目标：** 理解 Claude Code 的规划模式（Plan Mode）和工作流编排系统，掌握 EnterPlanMode/ExitPlanMode 的模式切换机制，了解计划文件的存储与恢复策略，以及调度系统（Cron、RemoteTrigger）和后台任务管理的实现细节。

---

## 14.1 Plan 模式的架构

### 14.1.1 设计哲学：先规划后执行

Plan 模式是 Claude Code 中最独特的设计之一。它将 Agent 的行为分为两个阶段：只读探索阶段（规划）和可写执行阶段（实施）。这种分离的核心理念是：在动手之前先对齐意图，避免方向性错误导致的返工。

EnterPlanModeTool 是进入规划模式的入口。它的 prompt 系统针对不同用户类型（外部用户 vs 内部 ant 用户）提供了不同的指导策略。

对外部用户，系统倾向于鼓励使用 Plan 模式，提示模型优先使用 Plan 模式来处理实现任务。对内部 ant 用户，系统更加节制，建议在有疑问时直接开始工作并通过提问澄清，而非进入完整的规划阶段。

这种差异化策略反映了不同的使用场景：外部用户更看重安全性和对齐，而内部用户更看重效率和流畅度。

### 14.1.2 模式切换机制

当用户批准进入 Plan 模式后，EnterPlanModeTool 的 `call` 方法执行模式切换，核心逻辑包括：检查是否在 Agent 上下文中（如果是则抛出错误），调用权限过渡处理函数切换到 plan 模式，以及通过状态更新函数将权限上下文切换为 plan 模式配置。

关键点：
- **Agent 上下文中禁止进入 Plan 模式**：子 Agent 不应该进入规划模式，这是架构层面的约束。
- **prepareContextForPlanMode**：运行 classifier activation 的副作用，确保在 plan 模式下权限配置正确。
- **prePlanMode 保存**：原始模式被保存到 `prePlanMode` 字段，供 ExitPlanMode 恢复使用。

进入 Plan 模式后，Agent 收到的 tool_result 包含明确的行为指令：

```
In plan mode, you should:
1. Thoroughly explore the codebase to understand existing patterns
2. Identify similar features and architectural approaches
3. Consider multiple approaches and their trade-offs
4. Use AskUserQuestion if you need to clarify the approach
5. Design a concrete implementation strategy
6. When ready, use ExitPlanMode to present your plan for approval

Remember: DO NOT write or edit any files yet.
This is a read-only exploration and planning phase.
```

### 14.1.3 退出 Plan 模式

ExitPlanModeV2Tool 负责从 Plan 模式退出并恢复原始权限模式。它的设计比 EnterPlanMode 复杂得多，因为需要处理多种场景。

**模式恢复**是核心逻辑。系统从 `prePlanMode` 中读取保存的原始模式并恢复。但存在一个关键的"断路器"防御：如果 `prePlanMode` 是 `auto`，但 auto mode 的 gate 当前关闭（如因 circuit breaker 触发），则回退到 `default` 模式。

**Teammate 路径**：当 Agent 作为 teammate 运行且 Plan 模式是强制要求时，ExitPlanMode 不会弹出本地审批 UI，而是通过 mailbox 机制将计划发送给 team lead 审批。审批请求包含发送者、时间戳、计划文件路径、计划内容和请求 ID 等信息。

---

## 14.2 计划验证机制

### 14.2.1 验证智能体

`registerPlanVerificationHook` 在 `ExitPlanModeV2Tool` 中被引用，其注释揭示了一个重要设计决策：验证钩子必须在 context clear 之后注册，因为 context clear 会清除所有 hooks。

验证钩子必须在 context clear 之后注册，因为 context clear 会清除所有 hooks。这意味着验证智能体运行在"清除上下文并开始实施"之后的阶段，作为一个独立的后台 Agent 来检验实施结果是否符合计划。

### 14.2.2 计划文件的持久化与恢复

计划文件管理是 Plan 模式的基础设施，由专门的工具函数模块提供。

**文件路径生成**采用 slug-based 命名策略。主会话使用简单的 `{slug}.md` 格式，子 Agent 使用 `{slug}-agent-{agentId}.md` 格式，避免文件冲突。

**Slug 生成**是惰性的：首次访问时生成，后续从缓存读取。如果生成的 slug 对应的文件已存在，最多重试 10 次。

**恢复机制**是多层次的。`copyPlanForResume` 在会话恢复时从三个来源尝试恢复计划：

1. **直接读取计划文件**：最简单的路径，如果文件存在直接返回。
2. **文件快照恢复**（`findFileSnapshotEntry`）：从 transcript 中的 `file_snapshot` 系统消息恢复，这在远程会话（CCR）中尤其重要，因为本地文件不会在会话之间持久化。
3. **消息历史恢复**（`recoverPlanFromMessages`）：从三种消息格式中提取计划内容：
   - ExitPlanMode 的 `tool_use` input（`normalizeToolInput` 注入的计划内容）
   - User message 的 `planContent` 字段（clear context 流程中设置）
   - Attachment message 的 `plan_file_reference`（auto-compact 时保留计划）

**Fork 恢复**使用完全不同的策略。`copyPlanForFork` 为 fork 的会话生成一个新的 slug，并将原始计划文件复制到新路径。这种设计防止了原始会话和 fork 会话互相覆盖对方的计划文件。

---

## 14.3 工作流系统

### 14.3.1 WorkflowTool 与 Skill 系统

Claude Code 的工作流能力主要由 Skill 系统提供。Skill 本质上是预定义的 prompt 模板，可以通过 slash command 触发。Skill 执行的核心准备函数处理三件事：

1. **Prompt 替换**：将 `$ARGUMENTS` 占位符替换为用户提供的参数。
2. **权限扩展**：通过 `createGetAppStateWithAllowedTools` 为 fork 的执行上下文添加允许的工具列表。
3. **Agent 选择**：优先使用 command 指定的 agent type，否则回退到 `general-purpose` agent。

### 14.3.2 Fork Agent 的状态隔离

`createSubagentContext` 创建了一个完全隔离的 ToolUseContext。默认情况下，所有可变状态都被隔离：

- **readFileState**：从父上下文克隆。
- **abortController**：创建子控制器，链接到父控制器（父中止传播到子）。
- **setAppState**：默认为 no-op，除非显式选择共享（`shareSetAppState`）。
- **UI callbacks**（addNotification、setToolJSX 等）：全部为 undefined，子 Agent 不能控制父 UI。

---

## 14.4 调度系统

### 14.4.1 ScheduleCronTool：本地定时任务

Claude Code 的调度系统支持两类定时任务：

- **One-shot**（`recurring: false`）：触发一次后自动删除。
- **Recurring**（`recurring: true`）：按计划重复触发，从当前时间重新调度。

任务存储在 `<project>/.claude/scheduled_tasks.json` 中，文件格式为：

```json
{
  "tasks": [
    {
      "id": "a1b2c3d4",
      "cron": "0 * * * *",
      "prompt": "check the deploy status",
      "createdAt": 1710000000000,
      "recurring": true
    }
  ]
}
```

每个 CronTask 包含一个 `durable` 运行时标志。`durable: false` 的任务仅存在于进程内存中，会话结束即消失。写入磁盘的任务会剥离这个标志。

### 14.4.2 CronScheduler：调度器核心

`createCronScheduler` 实现了完整的调度器，包含以下关键特性：

**文件锁**：通过 `tryAcquireSchedulerLock` 确保同一个项目目录下的多个 Claude 会话不会重复触发同一个 on-disk 任务。非 owner 会话每 5 秒探测一次锁，如果 owner 崩溃则接管。

**Jitter 机制**：为避免大量会话在同一时刻触发推理请求（惊群效应），recurring 任务添加了确定性的正向延迟。延迟与两次触发之间的间隔成正比，默认不超过 15 分钟。对于每小时执行的任务，实际触发时间会在 `:00` 到 `:06` 之间随机分散。

One-shot 任务使用反向 jitter（提前触发），通过 `oneShotMinuteMod` 控制：默认值为 30，意味着只有 `:00` 和 `:30` 的整点触发会添加 jitter。

**错过任务检测**：启动时检查是否有任务的下次触发时间已经过去，如果有则通知用户。

**自动过期**：Recurring 任务默认 7 天后自动过期（`recurringMaxAgeMs`），防止无限递归导致内存泄漏。`permanent` 标志的任务（如 assistant mode 的内置任务）豁免于此限制。

### 14.4.3 RemoteTriggerTool：远程触发

RemoteTriggerTool 提供了远程 Agent 的触发管理能力，支持 list、get、create、update、run 五种操作。

它通过 Anthropic API 的 triggers 端点工作，使用 OAuth bearer token 认证，并携带组织标识和 beta 标志等头部信息。

工具的启用受两个条件门控：feature flag `tengu_surreal_dali` 和 policy limit `allow_remote_sessions`。只有在两个条件都满足时，RemoteTriggerTool 才会出现在工具列表中。

### 14.4.4 定时任务的会话级生命周期

Session-scoped 任务（`durable: false`）的生命周期与会话绑定。它们存储在 bootstrap state 中，不写入磁盘。

`listAllCronTasks` 合并文件任务和 session 任务，返回统一的任务列表。

在 scheduler 的 `check()` 方法中，session 任务和文件任务走不同的处理路径。Session 任务直接在内存中操作（同步删除），文件任务通过 `removeCronTasks` 异步写入磁盘。

---

## 14.5 后台任务与主动模式

### 14.5.1 SleepTool

SleepTool 被列为后台任务管理的一部分。在 Claude Code 的工具注册中，它与其他调度相关工具一起注册。SleepTool 允许 Agent 在执行过程中暂停一段时间，这在长时间运行的后台任务场景中特别有用，例如轮询部署状态或等待外部系统完成操作。

### 14.5.2 后台会话管理

`runForkedAgent` 是后台 Agent 执行的基础。它通过 `createSubagentContext` 创建完全隔离的上下文，运行独立的查询循环，并追踪完整的 usage 指标。

关键的隔离设计包括：

1. **文件状态缓存克隆**（`cloneFileStateCache`）：子 Agent 的文件读取不影响父 Agent 的缓存。
2. **独立的 AbortController**：子 Agent 可以被独立中止，而不影响父 Agent。
3. **权限提示抑制**：后台 Agent 的 `getAppState` 包装了 `shouldAvoidPermissionPrompts: true`，避免后台操作弹出 UI 提示。
4. **Transcript 记录**：将子 Agent 的消息记录到独立的 sidechain 中，与主会话的消息分离。

`lastCacheSafeParams` 是一个全局 slot，用于在 post-turn hooks 中保存当前轮次的 cache-safe params。这使得 post-turn fork（如 promptSuggestion、postTurnSummary）可以直接复用主循环的 prompt cache，无需每个调用者手动传递参数。

---

## 实战练习

### 练习 1：体验 Plan 模式完整流程

在 Claude Code REPL 中输入一个非平凡的实现任务（如"为项目添加一个配置验证模块"），观察：
1. EnterPlanMode 触发的条件（模型是否主动进入了 Plan 模式）
2. Plan 模式下模型的行为（是否只使用了只读工具）
3. 计划文件的内容和存储位置
4. ExitPlanMode 的审批流程和模式恢复

### 练习 2：创建 Cron 定时任务

使用 `/schedule` 技能创建一个 one-shot 和一个 recurring 定时任务。检查 `.claude/scheduled_tasks.json` 的内容变化。终止并重新启动 Claude Code，观察错过任务的检测通知。

### 练习 3：分析计划文件恢复

在一个远程会话（CCR）中，进入 Plan 模式并创建一个计划。终止会话后恢复（`--resume`），检查计划文件是否被正确恢复。结合 `recoverPlanFromMessages` 的三种恢复来源，分析你的场景走了哪条路径。

---

## 关键要点

1. **Plan 模式是只读探索与可写执行的分离**，通过权限模式切换实现。EnterPlanMode 保存原始模式并切换到 plan，ExitPlanMode 恢复原始模式并处理断路器防御。
2. **计划文件管理采用 slug-based 命名和惰性生成**，支持三层恢复策略（直接读取、文件快照、消息历史），确保计划在各种故障场景下不丢失。
3. **Fork Agent 通过 CacheSafeParams 实现提示缓存共享**，通过 createSubagentContext 实现状态隔离，确保子 Agent 不干扰主 Agent 的状态。
4. **调度系统支持文件持久化和会话级两种任务**，通过文件锁防止多会话重复触发，通过 jitter 机制防止惊群效应，通过自动过期防止无限递归。
5. **Teammate 的 Plan 审批走 mailbox 机制**，而非本地 UI 弹窗，这是分布式 Agent 协作的基础设施。
