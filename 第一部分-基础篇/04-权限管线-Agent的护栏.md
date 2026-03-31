# 第4章：权限管线 -- Agent 的护栏

> 学习目标：理解 Claude Code 四阶段权限检查流程、权限上下文的设计哲学、五种权限模式的行为差异，以及权限规则的更新与持久化机制。

当 Agent 自主执行任务时，它可能在一次会话中调用数十个工具——写文件、运行 Bash 命令、搜索代码。每一次调用都潜藏着风险：一个不恰当的 `rm -rf`，一次意外的 `npm publish`，都可能造成不可逆的后果。权限管线（Permission Pipeline）正是 Claude Code 为 Agent 构建的安全护栏。它不是简单的"允许/拒绝"开关，而是一套精心设计的多阶段检查机制，在自动化效率与安全控制之间寻找精确的平衡。

本章将从架构设计出发，深入拆解权限管线的每一个阶段，理解其设计决策，并通过实战练习掌握配置方法。

## 4.1 权限管线的四个阶段

Claude Code 的权限检查并非单一函数的布尔判断，而是一条由四个阶段组成的管线（Pipeline）。每个阶段都有自己的职责和短路逻辑：只要前一阶段做出终局决定，后续阶段便不再执行。

这四个阶段分别是：validateInput（输入验证）、hasPermissionsToUseTool（规则匹配）、checkPermissions（上下文评估）和交互式提示（用户确认）。它们共同构成了从"数据合法性"到"人类授权"的完整防线。

### 4.1.1 阶段一：validateInput -- Zod Schema 验证

权限管线的第一道关卡并非权限本身，而是输入数据的合法性。在权限检查函数中，工具的输入会先经过 Zod Schema 解析，使用严格的结构验证。如果输入不符合 Schema 定义（例如 Bash 工具缺少必需的 `command` 字段），解析将抛出异常，工具调用被直接拒绝。这种设计将"数据不合法"与"权限不足"区分开来——前者是编程错误，后者是策略决策。

值得注意的是，解析失败时 `toolPermissionResult` 保持默认的 `passthrough` 状态，随后会被转换为 `ask` 行为，这意味着即使数据验证失败，系统也会优雅地降级为请求用户确认，而非直接崩溃。

### 4.1.2 阶段二：hasPermissionsToUseTool -- 规则匹配

规则匹配是权限管线的核心。`hasPermissionsToUseToolInner` 函数按照严格的优先级顺序依次检查三类规则：deny 规则、ask 规则和 allow 规则。

**步骤 1a：工具级 deny 检查。** 这是最高的优先级。如果某个工具被整体 deny，调用立即被拒绝。

`getDenyRuleForTool` 函数遍历所有来源（userSettings、projectSettings、localSettings、flagSettings、policySettings、cliArg、command、session）的 deny 规则进行匹配。匹配逻辑支持精确工具名匹配和 MCP 服务器级通配符——例如规则 `mcp__server1__*` 可以匹配该服务器下的所有工具。

**步骤 1b：工具级 ask 检查。** 当工具被配置为"总是询问"时，系统会强制弹出确认提示。但有一个例外：当 Bash 工具在沙箱模式下且配置了沙箱自动放行选项时，沙箱化命令可以跳过 ask 规则。

**步骤 2b：工具级 allow 检查。** 如果没有 deny 和 ask 规则命中，系统检查是否存在 allow 规则直接放行。

规则匹配的核心抽象是 PermissionRule 类型，它将来源、行为和目标统一为一个结构化对象。

### 4.1.3 阶段三：checkPermissions -- 上下文评估

每个工具可以实现自己的 `checkPermissions` 方法，进行更精细的上下文评估。例如 Bash 工具会解析命令中的子命令、检查路径安全性、匹配前缀规则等。这一阶段的调用发生在阶段二的规则匹配之前（代码中的步骤 1c），但其结果可能被后续的 allow 规则覆盖。

`checkPermissions` 返回的 `PermissionResult` 有四种行为：

- `allow`：直接放行，可能携带 `updatedInput`
- `deny`：拒绝执行
- `ask`：需要用户确认
- `passthrough`：交由后续阶段决定（最终会变为 `ask`）

`passthrough` 是一个独特的设计。当一个工具没有特殊的权限逻辑时，它可以返回 `passthrough`，让管线继续流动。在后续步骤中，`passthrough` 被自动转换为 `ask`，附带权限请求消息。

### 4.1.4 阶段四：交互式提示 -- 用户确认

当管线流转到 `ask` 状态时，权限确认 Hook 接管控制。这个 React Hook 将权限请求转化为用户交互界面。`ask` 状态触发的流程包含多条分支路径：协调器权限处理、swarm worker 权限处理、投机性分类器检查，以及交互式权限提示。

最关键的是交互式处理分支，它将权限请求推入确认队列，在终端中渲染确认界面，等待用户做出"允许/拒绝/本次允许"等选择。

在用户响应之前，系统可能还会运行异步的分类器检查（`pendingClassifierCheck`）。这意味着用户在看到提示后，分类器可能在用户还在思考时就已经自动批准了该操作，实现了一种"竞赛"机制——谁先做出决定，谁就生效。

## 4.2 PermissionContext 的设计

### 4.2.1 ToolPermissionContext 类型结构

`ToolPermissionContext` 是整个权限系统的核心数据结构，它携带了权限检查所需的所有上下文信息。它包含权限模式、额外工作目录、各级规则（allow/deny/ask）、bypass 模式可用性标志、是否应避免权限提示等字段。

这个类型的精妙之处在于它的不可变性（所有字段均为 readonly）。每次权限更新都会产生一个新的 `ToolPermissionContext` 对象，而非修改现有对象。这种不可变数据模式确保了在并发环境下的一致性——多个工具同时检查权限时，各自读取的上下文不会因其他工具的权限更新而意外变化。

其中规则按来源索引，这种设计允许系统精确追踪每条规则的来源，在规则管理界面中展示"此规则来自项目级设置"等信息。

### 4.2.2 PermissionDecision 的来源：hook、user、classifier

权限决策（PermissionDecision）有三个来源：

**hook 来源**：外部 Hook 脚本可以通过权限请求钩子在用户看到提示之前做出决策。这在 CI/CD 环境中特别有用——Hook 脚本可以根据自定义逻辑自动批准或拒绝特定操作。

**user 来源**：用户在终端界面中的手动选择。`permanent` 标志表示用户是否选择将此决策持久化到配置文件。

**classifier 来源**：AI 分类器在 auto 模式下做出的自动决策，后面会详细讨论。

`PermissionDecision` 本身是一个联合类型，包含 allow、ask 和 deny 三种行为。

### 4.2.3 ResolveOnce 模式：原子化的竞争解决

在权限交互中，多个异步参与者可能同时尝试解决（resolve）同一个权限请求——用户点击"允许"的同时分类器也返回了"通过"。`ResolveOnce` 模式正是为解决这种竞争条件而设计的。它提供 resolve、isResolved 和 claim 三个方法。

其实现使用了一个 `claimed` 标志确保原子性。`claim()` 方法是关键——它提供了一种"先到先得"的原子操作。在 JavaScript 的单线程模型中，`claim()` 保证了即使两个异步回调几乎同时被调度，也只有一个能成功 claim，另一个会被拒绝。这是一种轻量级的互斥锁实现，避免了分布式系统中常见的锁开销。

## 4.3 权限模式谱系

Claude Code 定义了五种内部权限模式，它们构成了一个从严格到宽松的谱系。

### 4.3.1 default 模式：逐次确认

`default` 模式是最保守的模式。除了被明确 allow 规则放行的工具外，每次工具调用都需要用户确认。这是普通用户启动 Claude Code 时的默认体验。在权限模式配置中，default 模式体现了其保守基调，UI 标识上无特殊标记，暗示这是"标准状态"。

### 4.3.2 plan 模式：只读为主

`plan` 模式将 Agent 限制在只读操作范围内。它的 UI 标识是一个暂停图标。

在 plan 模式下，写入类工具（Edit、Write）的权限检查会返回 `deny`，但读取类工具（Read、Grep、Glob）正常放行。值得注意的细节是当 plan 模式的 bypass 可用性标志为 true 时，意味着用户原本使用的是 bypass 模式然后切换到了 plan 模式，此时 bypass 的放行逻辑仍然生效。

### 4.3.3 auto 模式：自动审批（带分类器）

`auto` 模式是 Claude Code 最精巧的模式之一。它使用 AI 分类器（称为 "YOLO classifier"）来代替人工审批。当权限管线到达 `ask` 状态且模式为 `auto` 时，系统会调用分类器进行 AI 判断，将工具名称和输入格式化为分类器可理解的格式，并发送对话上下文和工具列表供分类器参考。

auto 模式实现了多层优化以减少分类器的 API 调用开销：

1. **acceptEdits 快速路径**：在调用分类器之前，先用 acceptEdits 模式检查工具是否会被放行。如果在 acceptEdits 模式下通过，直接放行，无需调用分类器。
2. **安全工具白名单**：不需要分类器检查的安全工具包括 Read、Grep、Glob、TodoWrite 等只读或低风险工具。
3. **拒绝追踪**：当分类器连续拒绝多次后，系统会自动回退到交互式提示，防止 Agent 在无意义的循环中浪费 token。

auto 模式的一个重要安全设计是：某些安全检查类型的决策（如涉及 `.git/`、`.claude/` 目录的操作）是"分类器免疫"的——即使分类器试图批准它们，这些安全检查也不会被绕过。

### 4.3.4 bypassPermissions 模式：完全跳过

`bypassPermissions` 模式是权限系统的"关闭开关"。当激活时，除了被 deny 规则阻止和不可绕过的安全检查之外，所有工具调用都自动放行。判断是否应该绕过权限时会检查当前模式是否为 bypassPermissions，或者模式为 plan 但 bypass 标志可用。

但即使 bypass 模式也无法突破以下防线：
- 步骤 1a 的 deny 规则（在 bypass 检查之前执行）
- 步骤 1e 的 `requiresUserInteraction` 检查
- 步骤 1f 的内容级 ask 规则
- 步骤 1g 的 safetyCheck

这种分层防御设计确保了"完全信任"模式下仍有最低限度的安全保障。

### 4.3.5 bubble 模式：子智能体权限冒泡

`bubble` 模式是内部模式（不对外暴露），用于子智能体（subagent）场景。当主 Agent 生成子智能体时，子智能体的权限检查会"冒泡"回主 Agent 的权限上下文，确保子智能体不会获得超出主 Agent 的权限。

内部权限模式类型包含了所有模式（包括 auto 和 bubble），而外部可见性检查函数确保内部模式不会泄露到外部接口。

## 4.4 BashTool 的权限细节

Bash 工具是权限系统中最复杂的工具，因为 shell 命令的组合性和表达力远超其他工具。Claude Code 为其设计了专门的规则匹配机制。

### 4.4.1 命令分类与通配符匹配

权限规则字符串的解析由专用函数处理，支持三种格式：不含括号的工具名匹配，以及含括号的 ToolName(content) 格式（用于指定工具级别的精确命令匹配）。

对于 Bash 工具，`ruleContent` 可以是：
- 精确命令：`Bash(npm test)` -- 仅匹配 `npm test`
- 前缀规则：`Bash(npm:*)` -- 匹配所有以 `npm` 开头的命令
- 通配符规则：`Bash(git commit *)` -- 匹配 `git commit` 后跟任意参数

通配符匹配引擎实现了完整的模式匹配，处理转义序列，将通配符 `*` 转换为正则表达式，支持转义 `\*` 和 `\\`。

一个优雅的细节：当模式以 ` *`（空格加通配符）结尾且这是唯一的通配符时，尾部的空格和参数变为可选的。这意味着规则 `git *` 既匹配 `git add .` 也匹配裸 `git` 命令——与 `git:*` 前缀语义保持一致。

### 4.4.2 前缀提取规则

前缀提取处理了向后兼容的 `:*` 语法，通过正则匹配提取冒号前的部分作为前缀。

当解析规则时，系统首先检查 `:*` 语法（前缀匹配），然后检查通配符，最后回退到精确匹配。

### 4.4.3 分类器自动审批机制

Bash 工具有自己的分类器自动审批机制，与 auto 模式的 YOLO 分类器并行运行。当权限请求处于 `ask` 状态时，系统会尝试"投机性"运行分类器检查。

这里有一个精心设计的超时机制——使用 Promise 竞争让分类器检查与 2 秒定时器竞争。如果分类器在 2 秒内返回高置信度的匹配结果，则自动批准该操作；否则用户就会看到交互式提示。这种"尽力而为"的策略确保了分类器不会成为用户体验的瓶颈。

## 4.5 权限更新与持久化

### 4.5.1 PermissionUpdate 模式

权限更新通过 `PermissionUpdate` 联合类型表达，支持六种操作：添加规则、替换规则、移除规则、设置模式、添加目录和移除目录。

每种操作都指定了更新要应用到的配置源：全局用户设置、项目级设置、本地（不提交到版本控制）设置、仅当前会话、命令行参数。

### 4.5.2 applyPermissionUpdates 与 persistPermissionUpdates

`applyPermissionUpdates` 将更新应用到内存中的权限上下文，是即时生效的。它遍历所有更新，逐个应用到上下文对象上，返回更新后的新上下文。

`applyPermissionUpdate` 处理每种更新类型。以 `addRules` 为例，它会将规则字符串添加到对应行为类型（allow/deny/ask）和对应来源的规则列表中，通过不可变更新产生新的上下文对象。

而 `persistPermissionUpdates` 将更新写入文件系统，确保在重启后仍然生效。只有本地设置、用户设置和项目设置三个目标支持持久化。会话级和命令行参数级的更新仅在运行时生效。

这两个函数的分离设计非常重要：内存应用是同步且即时的，而文件持久化可能涉及 I/O 操作。在权限上下文的用户允许处理方法中，两者被串联调用。

持久化函数返回一个布尔值指示是否有持久化更新发生，这被用于日志记录中标记"永久"还是"临时"决策。

---

## 实战练习：配置项目级 .claude/settings.json

让我们通过一个实际场景来理解权限配置。假设你正在开发一个 Node.js 项目，希望 Claude Code 能够自动执行以下操作：

1. 运行 `npm test` 和 `npm run lint` 而不需要每次确认
2. 允许执行所有 `git` 命令
3. 禁止执行 `npm publish`
4. 禁止删除 `node_modules` 之外的目录

创建项目根目录下的 `.claude/settings.json`：

```json
{
  "permissions": {
    "allow": [
      "Bash(npm test)",
      "Bash(npm run lint)",
      "Bash(git:*)",
      "Read",
      "Glob",
      "Grep"
    ],
    "deny": [
      "Bash(npm publish)",
      "Bash(rm -rf *)"
    ]
  }
}
```

**解析规则：**

- `Bash(npm test)` -- 精确匹配，仅允许 `npm test` 这一条命令
- `Bash(git:*)` -- 前缀匹配，允许所有 `git` 开头的命令（`git add`、`git commit` 等）
- `Read` -- 无 `ruleContent`，表示允许所有 Read 工具调用
- `Bash(npm publish)` -- deny 规则，精确拒绝发布操作

如果你希望规则仅在本地生效（不提交到 Git），可以改用 `.claude/settings.local.json`，其配置格式相同，但 `.gitignore` 默认排除它。

更精细的控制可以使用通配符：

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run *)",
      "Bash(npx eslint *)"
    ],
    "deny": [
      "Bash(npm publish *)",
      "Bash(* > /etc/*)"
    ]
  }
}
```

`Bash(npm run *)` 会匹配 `npm run build`、`npm run test:coverage` 等所有子命令。而 `Bash(* > /etc/*)` 使用多个通配符来拦截任何尝试写入 `/etc` 目录的命令。

---

## 关键要点总结

1. **四阶段管线**：权限检查按照 validateInput -> 规则匹配 -> checkPermissions -> 交互式提示的顺序逐级推进，任何阶段都可以做出终局决定。

2. **优先级铁律**：deny 规则的优先级最高，始终在 allow 规则之前检查。即使用户配置了 `bypassPermissions` 模式，deny 规则和 safetyCheck 仍然生效。

3. **PermissionContext 的不可变性**：每次权限更新产生新的上下文对象，确保并发安全。

4. **ResolveOnce 原子竞争**：用户交互和分类器自动审批之间通过 `claim()` 原子操作解决竞争，保证只有一个决策者。

5. **五模式谱系**：从最严格的 `default` 到最宽松的 `bypassPermissions`，中间有 `plan`（只读）、`auto`（AI 分类器）、`bubble`（子智能体冒泡）三种特殊模式，覆盖从交互开发到自动化的完整场景。

6. **Bash 工具的精细控制**：支持精确匹配、前缀匹配（`:*` 语法）和通配符匹配三种规则格式，以及投机性分类器自动审批。

7. **双层更新机制**：`applyPermissionUpdates` 即时修改内存上下文，`persistPermissionUpdates` 异步写入文件系统，两者分离确保了响应速度和数据持久化。
