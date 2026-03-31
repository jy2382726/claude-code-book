# 第6章：记忆系统 -- Agent 的长期记忆

> **学习目标：** 掌握四种记忆类型的设计意图和自动提取机制，理解基于 Fork 模式的缓存感知架构，学会设计持久化的 Agent 记忆系统。

---

人类之所以能够在多次对话中保持连贯，是因为我们有记忆。同样，一个真正有用的 Agent 不能每次对话都从零开始——它需要记住用户是谁、项目在做什么、以及哪些做法被验证过。Claude Code 的记忆系统（memdir）正是为此而生：一个基于文件的、类型化的、跨会话持久的记忆架构。

## 6.1 四种记忆类型的分类学

### 6.1.1 闭合类型系统

Claude Code 的记忆被约束为一个闭合的四类型分类系统，定义在记忆类型常量中：user、feedback、project、reference。

这四种类型的设计哲学是：**只保存不可从当前项目状态推导的信息**。代码模式、架构、文件结构和 Git 历史都可以通过工具（grep、git log）实时获取，因此不属于记忆的范畴。

### 6.1.2 四种类型的详细解析

**user -- 用户画像**

存储用户的角色、目标、知识背景。帮助 Agent 针对不同专业水平的用户调整协作方式——与资深工程师和初学者应该用不同的方式交流。

```
when_to_save: 当了解到用户的角色、偏好、知识背景时
how_to_use:  当需要根据用户画像调整解释深度和协作方式时
```

示例：用户说"我写了十年 Go，但这是第一次接触 React"，Agent 会保存一条 user 类型的记忆，在未来解释前端概念时使用后端类比。

**feedback -- 反馈指导**

记录用户对 Agent 行为的纠正和确认。这是最重要的记忆类型之一——它让 Agent 在未来的对话中保持行为一致性。

```
when_to_save: 用户纠正你的做法（"不要那样"）或确认非显而易见的做法成功时
body_structure: 规则本身 + Why: 原因 + How to apply: 适用场景
```

关键设计：不仅记录失败（纠正），也记录成功（确认）。如果只保存纠正，Agent 会变得过于谨慎，偏离已被验证的方法。

**project -- 项目状态**

记录项目的非代码状态——决策、截止日期、正在进行的工作。代码和 Git 历史是可推导的，但"为什么要这样做"和"什么时候需要完成"这类信息不是。

```
when_to_save: 了解到谁在做什么、为什么做、何时完成时
body_structure: 事实或决策 + Why: 动机 + How to apply: 对建议的影响
```

特别注意：相对日期必须转换为绝对日期（"周四" -> "2026-03-05"），因为记忆是跨会话持久的，相对日期在未来的对话中会失去意义。

**reference -- 外部引用**

指向外部系统的指针——Linear 项目、Grafana 仪表盘、Slack 频道。这些信息不在代码仓库中，但对理解项目上下文至关重要。

```
when_to_save: 了解到外部系统的资源和用途时
how_to_use:  当用户引用外部系统或需要查找外部信息时
```

### 6.1.3 明确排除的信息

记忆类型校验模块中明确列出了不应该保存为记忆的内容：

- 代码模式、惯例、架构、文件路径 -- 可以通过阅读代码推导
- Git 历史 -- `git log` / `git blame` 是权威来源
- 调试解决方案 -- 修复已经在代码中，上下文在 commit message 中
- CLAUDE.md 中已有的文档
- 临时任务细节 -- 当前对话的临时状态

甚至当用户**显式要求**保存这些信息时，系统也会引导到更有价值的方向："如果你想保存 PR 列表，请告诉我其中有什么**令人意外**或**非显而易见**的部分——那才是值得保存的。"

## 6.2 记忆文件格式

### 6.2.1 Frontmatter 格式

每条记忆是一个独立的 Markdown 文件，使用 YAML Frontmatter 声明元数据。格式要求包含 name（记忆名称）、description（一行描述，用于判断未来对话的相关性）和 type（四种类型之一）三个字段。`type` 字段必须是四种类型之一（严格校验），没有 type 字段的遗留文件可以继续工作，但无法被类型过滤。

### 6.2.2 MEMORY.md 索引文件

`MEMORY.md` 是记忆系统的入口点——它不是记忆本身，而是一个索引文件。每次对话开始时，它被自动加载到上下文中，让 Agent 快速了解已有的记忆概况。

记忆目录模块的常量定义了索引的容量限制：索引文件名为 `MEMORY.md`，最多 200 行，最大 25KB。

索引条目的格式要求每条一行，不超过 150 字符：

```markdown
- [Title](file.md) -- 一行钩子描述
```

`truncateEntrypointContent` 函数实现了双重容量保护：先按行截断（200 行上限），再按字节截断（25KB 上限）。超限时，文件末尾会追加一条警告说明哪个上限被触发。

### 6.2.3 记忆文件的目录结构

记忆文件的存储路径由路径解析模块中的函数决定。默认路径为：

```
~/.claude/projects/<sanitized-git-root>/memory/
```

路径解析的优先级：

1. `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` 环境变量（Cowork 模式的全路径覆盖）
2. `autoMemoryDirectory` 设置（仅限 policySettings/localSettings/userSettings，**projectSettings 被排除**）
3. 默认的 `<memoryBase>/projects/<sanitized-git-root>/memory/`

projectSettings 再次被排除，原因和上一章相同——防止恶意仓库通过 `autoMemoryDirectory: "~/.ssh"` 将写入重定向到敏感目录。路径校验函数对此做了严格的安全检查：拒绝相对路径、根路径、Windows 盘符根、UNC 路径和 null 字节注入。

## 6.3 自动记忆提取

### 6.3.1 基于 Fork 模式的后台提取

记忆提取系统的核心在记忆提取模块中。它不是由主对话直接执行的，而是通过 `runForkedAgent` 在后台运行——一种"完美分叉"模式。

所谓"完美分叉"，是指后台 Agent 与主对话共享完全相同的系统提示（system prompt）和工具集。这意味着：

- 后台 Agent 拥有与主 Agent 相同的上下文理解能力
- 后台 Agent 使用相同的工具集，但受到更严格的权限限制
- **提示缓存（prompt cache）在主对话和后台 Agent 之间共享**

提取的执行时机是在每次完整查询循环结束时（在模型产生最终回复且无工具调用时触发）。

### 6.3.2 互斥机制

记忆提取模块中的互斥检查函数实现了一个精妙的机制：如果主 Agent 已经在本次对话中写入了记忆文件，则后台提取直接跳过。主对话的 system prompt 中已包含完整的记忆保存指令。当主 Agent 主动保存了记忆时，后台的 forked Agent 会检测到这一事实并跳过本次提取——两者是**互斥**的，避免重复写入。

### 6.3.3 工具权限白名单

后台 Agent 的工具权限通过 `createAutoMemCanUseTool` 函数严格控制：

| 工具 | 权限 |
|------|------|
| Read / Grep / Glob | 不受限制（只读） |
| Bash | 仅限只读命令（ls、find、grep 等） |
| Edit / Write | 仅限记忆目录内的路径 |
| REPL | 允许（但内部调用受上述限制） |
| 其他所有工具 | 拒绝 |

这个设计使得后台 Agent 拥有足够的读取能力来理解对话内容，但写入能力被限制在记忆目录内——它不能修改项目代码或执行危险命令。

### 6.3.4 节流与协调

提取不是每次对话都运行的。系统实现了基于计数器的节流机制：每经过若干轮次后才会触发一次提取，通过功能开关配置阈值。

此外，还有一个 **trailing extraction**（尾随提取）机制来处理并发问题。当提取正在进行时又有新的对话完成，新的上下文会被暂存。当前提取完成后，会用最新暂存的上下文运行一次尾随提取。尾随提取跳过节流计数器——它处理的是已经完成的工作，不应该被节流延迟。

## 6.4 缓存感知的记忆架构

### 6.4.1 提示缓存共享

在 LLM API 中，提示缓存（prompt cache）是一种重要的成本优化机制——如果两次请求的前缀相同，API 可以复用已计算的 KV cache，显著降低延迟和成本。

Claude Code 的 forked Agent 模式通过 `CacheSafeParams` 类型实现缓存共享。参数提取函数从上下文中提取共享参数，包括系统提示、用户上下文、系统上下文、工具使用上下文和消息历史。

这意味着后台提取 Agent 的 API 请求前缀与主对话完全相同——API 提供商可以命中缓存前缀，避免重新计算。在一个典型的会话中，这可以节省大量 token 消耗。

### 6.4.2 工具列表的一致性要求

缓存共享有一个隐含约束：**工具列表是 API 缓存 key 的一部分**。如果 forked Agent 使用与主 Agent 不同的工具集，缓存就无法命中。这就是为什么工具权限过滤使用 `canUseTool` 回调而非不同的工具列表——工具列表保持一致，只是执行时被过滤。

### 6.4.3 记忆的生命周期

记忆启用的决策链在路径解析模块中定义：

1. `CLAUDE_CODE_DISABLE_AUTO_MEMORY` 环境变量（1/true -> 关闭）
2. `--bare`（SIMPLE 模式） -> 关闭
3. CCR 无持久存储（无 `CLAUDE_CODE_REMOTE_MEMORY_DIR`） -> 关闭
4. `settings.json` 中的 `autoMemoryEnabled` 字段
5. 默认：启用

后台提取还需要通过 GrowthBook 功能门控。这是双层控制：编译时特性标志加运行时 GrowthBook 实验。

### 6.4.4 记忆的读取与验证

记忆类型校验模块中定义了记忆读取的核心原则：**记忆是一个时间点的快照，而非当前的事实**。

```
"记忆说 X 存在" 不等于 "X 现在存在"。
```

具体验证规则：

- 如果记忆命名了文件路径：检查文件是否存在
- 如果记忆命名了函数或标志：grep 查找
- 如果用户即将根据你的建议行动（而非询问历史）：先验证

这体现了 Claude Code 的一个深层设计哲学：**记忆是被信任的线索，而非被信任的结论**。它指导 Agent 去哪里找信息，但不替代 Agent 对当前状态的独立验证。

---

## 实战练习

### 练习 1：记忆类型分类

以下信息应该保存为哪种记忆类型？

1. "我们团队使用 Linear 项目 'BACKEND' 来追踪后端 Bug"
2. "用户是初级开发者，第一次使用 TypeScript"
3. "集成测试必须使用真实数据库，不要 mock -- 上次 mock 导致生产事故"
4. "认证中间件重写是因为法律合规要求，不是技术债"

**参考答案**：
1. `reference` -- 外部系统指针
2. `user` -- 用户画像
3. `feedback` -- 行为指导（包含 Why: 生产事故）
4. `project` -- 项目决策（包含 Why: 法律合规）

### 练习 2：Frontmatter 编写

为以下场景编写记忆文件的 Frontmatter 和内容：

场景：用户说"以后每次提交代码都要先运行 `npm run lint`，上次有人提交了未 lint 的代码导致 CI 失败了一整天"。

**参考答案**：

```markdown
---
name: pre-commit-lint-requirement
description: Must run npm run lint before every commit; CI failed for a full day due to unlinted code
type: feedback
---

**Rule**: Run `npm run lint` before every code commit.

**Why**: A previous commit with unlinted code caused CI to fail for an entire day, blocking the team.

**How to apply**: Before using git commit, always run `npm run lint` first and fix any errors. This applies to all files changed in the commit, not just new files.
```

### 练习 3：缓存感知架构分析

假设你要为 forked Agent 添加一个新工具 `MemorySearch`（用于语义搜索记忆文件）。以下两种方案哪种更好？

- 方案 A：在 forked Agent 的工具列表中新增 `MemorySearch`，替换 `Grep`
- 方案 B：保持工具列表不变，通过 `canUseTool` 权限回调限制 Grep 只能搜索记忆目录

**参考答案**：方案 B 更好。方案 A 改变了工具列表，导致 API 缓存 key 不同，无法共享主对话的提示缓存。方案 B 保持了工具列表的一致性，权限在执行时而非定义时过滤，维护了缓存共享能力。

---

## 关键要点

1. **闭合四类型**：user、feedback、project、reference -- 只保存不可从代码推导的信息，排除代码模式、Git 历史等可推导内容。
2. **双重容量保护**：MEMORY.md 索引限制 200 行 / 25KB，先按行截断再按字节截断。
3. **Fork 模式**：后台 Agent 完美分叉主对话，共享提示缓存，通过 `canUseTool` 白名单限制写入权限。
4. **互斥提取**：主 Agent 和后台 Agent 的记忆写入是互斥的——主 Agent 写入时后台跳过。
5. **缓存感知**：工具列表的一致性是缓存共享的前提，权限过滤使用运行时回调而非编译时不同的工具列表。
6. **验证优先**：记忆是快照而非事实，推荐前必须验证当前状态。
