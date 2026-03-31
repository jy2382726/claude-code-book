# 附录 B：工具完整清单

本附录列出 Claude Code 源码中注册的全部工具，按功能分类索引。权限模型字段含义：

- **readOnly**：工具是否仅执行读取操作，不修改文件系统或外部状态
- **destructive**：工具是否执行不可逆操作（删除、覆盖、发送）
- **concurrencySafe**：工具是否可安全并行执行（不依赖共享可变状态）

表中 `true`/`false` 表示始终返回该值；`动态` 表示根据输入参数运行时判定。未在源码中显式覆盖的方法使用 `buildTool` 默认值（均为 `false`）。

---

## 1. 文件操作

| 工具名称 | readOnly | destructive | concurrencySafe | 简要说明 |
|---------|----------|-------------|-----------------|---------|
| FileReadTool | `true` | `false` | `true` | 读取文件内容，支持行号范围、PDF 分页、图片、notebook 格式 |
| FileWriteTool | `false` | `false` | `false` | 写入文件，覆盖已有内容或创建新文件 |
| FileEditTool | `false` | `false` | `false` | 精确字符串替换式编辑，支持单处/全局替换 |
| NotebookEditTool | `false` | `false` | `false` | 编辑 Jupyter Notebook 单元格（替换/插入/删除） |

## 2. 搜索

| 工具名称 | readOnly | destructive | concurrencySafe | 简要说明 |
|---------|----------|-------------|-----------------|---------|
| GlobTool | `true` | `false` | `true` | 文件名 glob 模式匹配搜索，按修改时间排序 |
| GrepTool | `true` | `false` | `true` | 基于 ripgrep 的正则内容搜索，支持多模式输出 |
| ToolSearchTool | `false` | `false` | `false` | 工具发现搜索，按关键词匹配延迟加载的工具 |

## 3. 执行

| 工具名称 | readOnly | destructive | concurrencySafe | 简要说明 |
|---------|----------|-------------|-----------------|---------|
| BashTool | `动态` | `false` | `动态` | Shell 命令执行，只读命令可并行，写命令串行。权限依据命令内容动态判定 |
| PowerShellTool | `动态` | `false` | `动态` | Windows PowerShell 命令执行（条件启用，功能与 BashTool 对等） |

## 4. 网络

| 工具名称 | readOnly | destructive | concurrencySafe | 简要说明 |
|---------|----------|-------------|-----------------|---------|
| WebFetchTool | `true` | `false` | `true` | 抓取 URL 内容并转换为 markdown |
| WebSearchTool | `true` | `false` | `true` | 执行网络搜索，返回结构化搜索结果 |

## 5. 智能体

| 工具名称 | readOnly | destructive | concurrencySafe | 简要说明 |
|---------|----------|-------------|-----------------|---------|
| AgentTool | `false` | `false` | `false` | 启动子智能体（内置/自定义），支持 fork、resume、后台执行 |
| SendMessageTool | `动态` | `false` | `false` | 向其他智能体或频道发送消息，纯文本消息为 readOnly |
| TeamCreateTool | `false` | `false` | `false` | 创建多智能体团队（条件启用：Agent Swarms 模式） |
| TeamDeleteTool | `false` | `false` | `false` | 删除已创建的团队 |

## 6. 任务管理

| 工具名称 | readOnly | destructive | concurrencySafe | 简要说明 |
|---------|----------|-------------|-----------------|---------|
| TaskCreateTool | `false` | `false` | `false` | 创建后台任务（条件启用：Todo V2） |
| TaskGetTool | `true` | `false` | `true` | 获取单个任务详情 |
| TaskUpdateTool | `false` | `false` | `false` | 更新任务状态/内容 |
| TaskListTool | `true` | `false` | `true` | 列出所有任务 |
| TaskOutputTool | `动态` | `false` | `动态` | 获取任务输出流，readOnly 时 concurrencySafe |
| TaskStopTool | `false` | `false` | `true` | 停止正在运行的任务 |

## 7. 计划

| 工具名称 | readOnly | destructive | concurrencySafe | 简要说明 |
|---------|----------|-------------|-----------------|---------|
| EnterPlanModeTool | `true` | `false` | `true` | 进入计划模式，限制为只读工具集 |
| ExitPlanModeV2Tool | `true` | `false` | `true` | 退出计划模式，恢复正常工具权限 |

## 8. 工作树

| 工具名称 | readOnly | destructive | concurrencySafe | 简要说明 |
|---------|----------|-------------|-----------------|---------|
| EnterWorktreeTool | `false` | `false` | `false` | 创建 git worktree 并切换工作目录（条件启用） |
| ExitWorktreeTool | `false` | `false` | `false` | 退出 worktree，保留或移除工作目录 |

## 9. 调度

| 工具名称 | readOnly | destructive | concurrencySafe | 简要说明 |
|---------|----------|-------------|-----------------|---------|
| CronCreateTool | `false` | `false` | `false` | 创建 cron 定时任务（条件启用：AGENT_TRIGGERS） |
| CronDeleteTool | `false` | `false` | `false` | 删除 cron 定时任务 |
| CronListTool | `true` | `false` | `true` | 列出所有 cron 定时任务 |
| RemoteTriggerTool | `动态` | `false` | `false` | 远程触发器管理（条件启用：AGENT_TRIGGERS_REMOTE） |

## 10. 交互

| 工具名称 | readOnly | destructive | concurrencySafe | 简要说明 |
|---------|----------|-------------|-----------------|---------|
| AskUserQuestionTool | `true` | `false` | `true` | 向用户提问并等待回复 |
| SkillTool | `false` | `false` | `false` | 调用已注册的 slash command 技能 |
| ConfigTool | `动态` | `false` | `false` | 运行时配置查看/修改（仅 ant 构建可用） |

## 11. MCP

| 工具名称 | readOnly | destructive | concurrencySafe | 简要说明 |
|---------|----------|-------------|-----------------|---------|
| ListMcpResourcesTool | `true` | `false` | `true` | 列出 MCP 服务器提供的资源 |
| ReadMcpResourceTool | `true` | `false` | `true` | 读取 MCP 服务器上的特定资源 |

## 12. 其他

| 工具名称 | readOnly | destructive | concurrencySafe | 简要说明 |
|---------|----------|-------------|-----------------|---------|
| TodoWriteTool | `false` | `false` | `false` | 待办事项面板写入（UI 联动，结果不渲染到 transcript） |
| BriefTool | `false` | `false` | `true` | 控制输出简洁性模式 |
| LSPTool | `true` | `false` | `true` | LSP 语言服务协议操作（条件启用：ENABLE_LSP_TOOL） |
| SleepTool | `false` | `false` | `false` | 延时等待（条件启用：PROACTIVE / KAIROS） |
| TungstenTool | `false` | `false` | `false` | 内部工具（仅 ant 构建可用） |
| SyntheticOutputTool | `true` | `false` | `true` | 合成输出工具（内部基础设施） |
| SnipTool | `false` | `false` | `false` | 历史消息裁剪（条件启用：HISTORY_SNIP） |
| MonitorTool | `false` | `false` | `false` | 监控工具（条件启用：MONITOR_TOOL） |
| WorkflowTool | `false` | `false` | `false` | 工作流脚本执行（条件启用：WORKFLOW_SCRIPTS） |
| ListPeersTool | `false` | `false` | `false` | 列出对等智能体（条件启用：UDS_INBOX） |
| REPLTool | `false` | `false` | `false` | REPL 包装器，在 VM 中提供 Bash/Read/Edit（仅 ant 构建） |
| SuggestBackgroundPRTool | `false` | `false` | `false` | 建议后台 PR 创建（仅 ant 构建） |
| WebBrowserTool | `false` | `false` | `false` | 浏览器工具（条件启用：WEB_BROWSER_TOOL） |
| SendUserFileTool | `false` | `false` | `false` | 向用户发送文件（条件启用：KAIROS） |
| PushNotificationTool | `false` | `false` | `false` | 推送通知（条件启用：KAIROS） |
| SubscribePRTool | `false` | `false` | `false` | PR Webhook 订阅（条件启用：KAIROS_GITHUB_WEBHOOKS） |
| CtxInspectTool | `false` | `false` | `false` | 上下文检查器（条件启用：CONTEXT_COLLAPSE） |
| TerminalCaptureTool | `false` | `false` | `false` | 终端截图捕获（条件启用：TERMINAL_PANEL） |
| VerifyPlanExecutionTool | `false` | `false` | `false` | 计划执行验证（条件启用：CLAUDE_CODE_VERIFY_PLAN） |
| OverflowTestTool | `false` | `false` | `false` | 溢出测试工具（内部测试用） |
| TestingPermissionTool | `true` | `false` | `true` | 权限测试工具（仅 NODE_ENV=test） |

---

## 工具启用条件速查

部分工具通过 feature flag 或环境变量条件启用：

| 条件标识 | 启用的工具 |
|---------|----------|
| `USER_TYPE === 'ant'` | ConfigTool, TungstenTool, REPLTool, SuggestBackgroundPRTool |
| `PROACTIVE` / `KAIROS` | SleepTool |
| `AGENT_TRIGGERS` | CronCreateTool, CronDeleteTool, CronListTool |
| `AGENT_TRIGGERS_REMOTE` | RemoteTriggerTool |
| `MONITOR_TOOL` | MonitorTool |
| `KAIROS` | SendUserFileTool, PushNotificationTool |
| `KAIROS_GITHUB_WEBHOOKS` | SubscribePRTool |
| `ENABLE_LSP_TOOL` | LSPTool |
| `WORKFLOW_SCRIPTS` | WorkflowTool |
| `HISTORY_SNIP` | SnipTool |
| `UDS_INBOX` | ListPeersTool |
| `WEB_BROWSER_TOOL` | WebBrowserTool |
| `CONTEXT_COLLAPSE` | CtxInspectTool |
| `TERMINAL_PANEL` | TerminalCaptureTool |
| `COORDINATOR_MODE` | coordinator 模式额外启用 AgentTool, TaskStopTool, SendMessageTool |
| `Todo V2` | TaskCreateTool, TaskGetTool, TaskUpdateTool, TaskListTool |
| `Agent Swarms` | TeamCreateTool, TeamDeleteTool |
| `Worktree Mode` | EnterWorktreeTool, ExitWorktreeTool |
| `CLAUDE_CODE_SIMPLE` | 精简模式仅保留 BashTool, FileReadTool, FileEditTool |
| `ToolSearch` | ToolSearchTool |
| `CLAUDE_CODE_VERIFY_PLAN` | VerifyPlanExecutionTool |

---

## 工具注册流程

所有工具通过统一的工具工厂函数创建，该函数为未显式定义的方法填充安全默认值：

- `isEnabled()` 默认返回 `true`
- `isReadOnly()` 默认返回 `false`
- `isConcurrencySafe()` 默认返回 `false`
- `isDestructive()` 默认返回 `false`
- `checkPermissions()` 默认返回允许（allow）
- `toAutoClassifierInput()` 默认返回空字符串
- `userFacingName()` 默认返回工具名称

工具注册入口在全局工具注册函数中，返回一个工具数组。运行时通过工具池组装函数将内置工具与 MCP 动态工具合并，按名称排序后去重（内置工具优先）。
