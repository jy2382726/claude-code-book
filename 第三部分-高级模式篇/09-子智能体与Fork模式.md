# 第9章：子智能体与 Fork 模式

> **学习目标：** 深入理解子智能体的生成机制和 Fork 模式的缓存共享策略。

Claude Code 的真正威力不仅在于单轮对话的能力，更在于它可以将复杂任务委派给专门的子智能体（Subagent）并行处理。本章将深入解析 AgentTool 的完整架构，揭示内置智能体的设计哲学，并重点拆解 Fork 模式如何通过精巧的缓存策略实现真正的并行执行而不浪费 token。

---

## 9.1 AgentTool 架构

### 目录结构与模块职责

AgentTool 的代码位于智能体工具目录下，由以下核心模块组成：

| 职责 | 说明 |
|------|------|
| 工具主入口 | 处理输入 schema、路由策略、同步/异步分支 |
| 智能体运行器 | 管理生命周期、上下文构建、MCP 初始化 |
| Fork 模式实现 | 构建缓存安全的消息前缀 |
| 内置智能体注册表 | 根据 feature gate 动态加载 |
| 自定义智能体加载 | 解析 Markdown/JSON 定义 |
| 工具函数集 | 工具过滤、解析、结果格式化等 |

这套架构的核心设计原则是**关注点分离**：工具主入口负责"决定做什么"，运行器负责"怎么做"，Fork 模块负责一种特定的"怎么做得更高效"的策略。

### BaseAgentDefinition 类型定义

所有智能体都基于智能体加载模块中定义的 `BaseAgentDefinition` 类型，包含以下关键字段：智能体唯一标识符（agentType）、使用场景描述（whenToUse）、允许/禁止的工具列表、预加载的 skill 名称、智能体专属 MCP 服务器、生命周期钩子、UI 显示颜色、模型指定、推理努力程度、权限模式、最大轮次限制、是否后台运行、隔离模式、是否省略 CLAUDE.md 上下文等。

从这个类型衍生出三种具体的智能体定义：

- **BuiltInAgentDefinition**（内置智能体）：通过 `getSystemPrompt()` 动态生成系统提示
- **CustomAgentDefinition**（自定义智能体）：来自用户/项目/策略设置
- **PluginAgentDefinition**（插件智能体）：插件提供的智能体，带有 plugin 元数据

### 三种智能体来源

智能体的加载优先级由加载合并函数决定：

1. **内置智能体**（built-in）：优先级最低，作为默认选项
2. **插件智能体**（plugin）：可以覆盖内置智能体
3. **用户设置**（userSettings）→ **项目设置**（projectSettings）→ **Flag 设置**（flagSettings）→ **策略设置**（policySettings）：优先级依次升高

这意味着策略级别的自定义智能体可以覆盖同名内置智能体，为企业部署提供了灵活性。

---

## 9.2 内置智能体

内置智能体在注册模块中通过 `getBuiltInAgents()` 函数返回当前环境下可用的智能体列表。每个智能体都被精心设计为特定领域的专家。

### Explore Agent：只读代码探索专家

Explore Agent 是一个高速只读搜索智能体。它的定义中声明了严格的禁止操作列表：禁止 Agent、ExitPlanMode、FileEdit、FileWrite、NotebookEdit 等工具，并可选择使用 haiku 模型以降低成本，同时省略 CLAUDE.md 以节省 token。

其核心设计理念体现在两个关键决策上：

**第一，严格的只读约束。** 系统提示中声明了严格的禁止操作列表（不能创建文件、修改文件、运行任何改变系统状态的命令），并且在工具层面通过 disallowedTools 物理禁止了 Edit、Write 等工具。

**第二，omitClaudeMd 优化。** Explore 是只读搜索智能体，不需要 commit/PR/lint 规则。这一优化据估计每周可节省 5-15 Gtoken。

### Plan Agent：结构化规划智能体

Plan Agent 复用了 Explore Agent 的工具集（同样禁止 Edit、Write 等修改类工具），但承担不同角色——它是软件架构师和规划专家。Plan Agent 使用 inherit 模型并省略 CLAUDE.md，其系统提示要求它输出结构化的实现计划，并以"关键文件"列表结尾，为后续的实现阶段提供清晰指引。

### General Purpose Agent：通用智能体

General Purpose Agent 是最灵活的智能体，拥有全部工具权限（使用通配符 `'*'` 允许所有工具）。但实际可用工具仍受全局过滤函数的约束。

### Verification Agent：验证智能体

Verification Agent 是一个独特的"对抗性"智能体，设计目标是**尽可能破坏被验证的代码**，而不是确认它能工作。它使用红色 UI 标识强调对抗性质，始终后台运行，禁止修改项目文件，使用 inherit 模型。其系统提示明确警告了两种失败模式：验证回避（找借口不运行测试）和被前 80% 的表面正确性所迷惑。这是一个精心设计的对抗性提示工程案例。

---

## 9.3 Fork 模式：缓存安全的并行执行

Fork 模式是 Claude Code 中最精巧的架构创新之一。它允许主智能体将同一时刻的完整上下文"分叉"给多个并行子任务，同时利用 Anthropic API 的 prompt cache 机制避免重复传输大量 token。

### forkSubagent 模块的核心设计

Fork 模式的激活需要满足 feature gate 开启、非 Coordinator 模式、非非交互模式三个条件。当 Coordinator 模式激活时，Fork 模式会被自动禁用，因为 Coordinator 已经拥有自己的任务委派模型。

在工具主入口的路由逻辑中，当用户省略子智能体类型且 Fork 模式开启时，系统会触发 Fork 路径。

### 继承父对话的完整上下文和系统提示

Fork 子智能体的核心原则是**字节级继承**——子智能体必须与父智能体共享完全相同的 API 请求前缀，才能命中 prompt cache。

在 Fork 路径中，系统会优先使用父智能体已渲染的系统提示字节，避免因重建导致的缓存失效。只有在无法获取已渲染提示时才降级到重建路径。重建可能因特性开关的冷热状态不同而偏离，导致缓存失效，因此传递渲染后的字节是保证精确匹配的关键。

### CacheSafeParams 五个维度

缓存安全参数类型定义了缓存命中的关键契约，包含五个维度：系统提示（systemPrompt）、用户上下文（userContext）、系统上下文（systemContext）、工具定义和模型（toolUseContext）、对话前缀消息（forkContextMessages）。

这五个维度对应 Anthropic API 缓存键的组成：system prompt、tools、model、messages prefix、thinking config。只有这五个维度完全一致时，子请求才能命中父请求的缓存。

Fork 路径通过 `useExactTools` 标志确保工具定义不变——当 Fork 模式激活时，直接使用父工具池而非重新解析智能体工具，从而保持工具定义的字节一致性。

### buildForkedMessages 的巧妙设计

Fork 消息构建函数是缓存共享的核心。它的处理逻辑如下：首先保留完整的父 assistant 消息（包含所有 tool_use 块、thinking、text，但分配新的 UUID）；然后收集所有 tool_use 块；为每个 tool_use 生成统一的固定字符串占位符 tool_result；最后构建单条 user 消息，包含占位符结果和子任务指令。

关键点在于占位符结果是所有 Fork 子智能体共享的固定字符串 "Fork started -- processing in background"。这意味着：

- **所有 Fork 子智能体共享相同的前缀**：`[...历史, assistant(所有 tool_use), user(占位符结果..., 指令)]`
- **只有最后的指令文本不同**：最大化缓存命中面积
- 结果结构为：`[...history, assistant(all_tool_uses), user(placeholder_results..., directive)]`

### 递归 Fork 防护

Fork 子智能体保留了 Agent 工具以维持缓存一致的工具定义，因此必须防止递归 Fork。系统实现了双重检测策略：

1. **querySource 检查**（主防线）：设置在上下文选项上，不受自动压缩影响
2. **消息扫描**（后备）：检测特定标签，应对 querySource 未传递的边界情况

子任务指令生成函数会生成包含严格行为规范的指令，核心内容包括：声明自己是 Fork 工作进程而非主智能体、禁止生成子智能体、禁止提问或建议下一步操作等。

---

## 9.4 自定义智能体

### .claude/agents/ 目录中的定义文件

自定义智能体通过加载合并函数加载。加载流程如下：

1. 扫描 agents 目录中的 Markdown 文件
2. 对每个 Markdown 文件解析智能体定义
3. 并行加载插件智能体和内存快照
4. 合并所有来源，按优先级去重

### Markdown frontmatter 格式

一个完整的自定义智能体 Markdown 文件示例如下：

```markdown
---
name: security-auditor
description: 分析代码中的安全漏洞并生成审计报告
tools:
  - Bash
  - Read
  - Grep
  - Glob
disallowedTools:
  - Write
model: haiku
effort: high
permissionMode: default
maxTurns: 30
color: orange
background: true
memory: project
skills:
  - /commit
mcpServers:
  - slack
hooks:
  PreToolUse:
    - matcher: Bash
      command: "audit-log.sh"
---

你是一位代码安全审计专家。你的职责是：

1. 分析代码中潜在的安全漏洞
2. 检查常见的攻击向量（XSS、SQL 注入、CSRF 等）
3. 生成结构化的审计报告
4. 提供修复建议

报告格式：
- **漏洞级别**：Critical / High / Medium / Low
- **影响范围**：受影响的文件和函数
- **修复建议**：具体的代码修改方案
```

`parseAgentFromMarkdown()` 函数解析这个 frontmatter，将 `name` 映射为 `agentType`，将 Markdown 正文作为系统提示。如果 `memory` 字段启用，系统还会自动注入文件读写工具以支持持久化记忆。

---

## 9.5 智能体工具隔离

### 全局禁止列表

系统定义了全局禁止列表，所有子智能体都不能使用这些工具，包括：防止递归获取输出的工具、Plan mode 相关工具（仅主线程可用）、Agent 工具（防止递归嵌套）、向用户提问的工具、任务停止工具（需要主线程任务状态）。

### 异步智能体白名单

异步（后台）智能体的工具集进一步受限为白名单集合，包括 Read、Write、Edit、Bash、Grep、Glob、WebSearch 等核心工具，但不包括 TaskOutput、Agent 等可能导致递归的工具。

### filterToolsForAgent

工具过滤函数是最终的仲裁者，实现三层过滤机制：MCP 工具始终可用；全局禁止工具一律排除；异步智能体只能使用白名单工具，但进程内队友可以豁免部分限制。

---

## 实战练习

**练习 1：创建一个自定义代码审查智能体**

在项目的 `.claude/agents/` 目录下创建 `code-reviewer.md`，定义一个专门用于代码审查的智能体。要求：
- 只读权限（禁止 Write、Edit）
- 使用 `haiku` 模型以降低成本
- 输出结构化的审查报告

**练习 2：分析 Fork 模式的缓存效率**

假设父对话有 100 条消息，父智能体一次性发出了 3 个 Fork 调用。根据 Fork 消息构建机制，计算：
- 第一个 Fork 子智能体的缓存前缀长度
- 第二、第三个 Fork 子智能体能复用多少缓存

**练习 3：探索 Verification Agent 的对抗策略**

查阅 Verification Agent 的系统提示设计理念，列出其中定义的所有"自我合理化借口"，并思考为什么这些对 LLM 验证任务是必要的。

---

## 关键要点

1. **AgentTool 的三层智能体体系**（内置、自定义、插件）提供了从开箱即用到深度定制的完整光谱，优先级机制允许企业级覆盖。

2. **内置智能体各司其职**：Explore 专注高速只读搜索，Plan 做结构化规划，General Purpose 是万能选手，Verification 采取对抗性验证策略。

3. **Fork 模式的核心创新**是通过字节级继承实现 prompt cache 共享：相同的系统提示、工具定义、消息前缀加上统一的占位符结果，最大化缓存命中面积。

4. **CacheSafeParams 的五个维度**（system prompt、user context、system context、tool context、messages）是缓存命中的充分必要条件，任何维度的偏离都会导致缓存失效。

5. **工具隔离的三层防线**（全局禁止列表、异步白名单、`filterToolsForAgent` 过滤）确保子智能体不会产生递归、不会越权、不会在后台模式下阻塞用户交互。
