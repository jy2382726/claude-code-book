# 第15章：构建你自己的 Agent Harness

> "The rules of thinking are lengthy and fortuitous. They require plenty of thinking of most long duration and deep meditation for a wizard to wrap one's noggin around."
> -- Claude Code 中的注释

**学习目标：** 综合运用全书知识，设计并实现一个自定义 Agent Harness，掌握从对话循环到生产部署的完整工程链路。

---

## 15.1 设计原则回顾与选型指南

### 五大设计原则在实际项目中的应用

全书贯穿了 Agent Harness 的五大设计原则。在动手编码之前，让我们回顾它们如何映射到具体的工程决策：

**原则一：循环优于递归。** Claude Code 的核心查询函数是一个 `AsyncGenerator`，通过 `while(true)` 循环和 `continue` 语句管理状态流转。每次迭代从 `state` 对象中解构出消息列表、压缩追踪、恢复计数等状态，然后在各个 `continue` 站点写入新的 `State` 对象。这种 "解构-重赋值" 模式比递归调用更易调试，也避免了调用栈溢出。

**原则二：Schema 驱动而非硬编码。** 每个工具都通过 Zod Schema 定义输入参数，`buildTool` 工厂函数从部分定义中自动填充默认行为。这意味着工具的验证逻辑、权限检查、描述生成都从同一个 Schema 派生，消除了不一致的根源。

**原则三：渐进式权限。** 权限系统分为四个阶段：`validateInput`（输入校验）、`checkPermissions`（权限检查）、PreToolUse 钩子（前置拦截）、`canUseTool`（用户确认）。每个阶段都可以短路返回，未通过前一阶段的请求不会进入下一阶段。

**原则四：流式优先。** 从模型响应到工具执行结果，所有数据都通过 `AsyncGenerator` 的 `yield` 传递。消费端可以逐条处理消息，而不必等待整个回合完成。这使得实时 UI 更新、流式传输到 SDK 消费者成为可能。

**原则五：可插拔扩展。** 钩子系统在二十多个生命周期节点提供扩展点，从 `SessionStart` 到 `PostToolUse`，从 `PreCompact` 到 `Stop`。每个钩子都是独立的 Shell 命令或 HTTP 端点，通过标准化的 JSON 输入输出协议与 Harness 交互。

### 何时使用 Agent Harness 模式 vs 简单 LLM API 调用

并非每个场景都需要完整的 Agent Harness。决策的关键在于三个维度：

| 维度 | 简单 API 调用 | Agent Harness |
|------|---------------|---------------|
| 交互轮次 | 单次请求-响应 | 多轮自主循环 |
| 工具需求 | 无或仅 Function Calling | 多种工具、权限控制 |
| 上下文管理 | 手动拼接 Prompt | 自动压缩、记忆提取 |
| 错误恢复 | 重试 | 多层恢复（断路器、降级、压缩） |

如果你的任务满足以下任一条件，就应该考虑 Agent Harness 模式：

1. 智能体需要根据中间结果决定下一步行动（自主循环）
2. 涉及文件系统、代码执行、网络请求等副作用操作
3. 对话可能跨越数十甚至数百轮交互
4. 需要在多个环境（CLI、IDE、SDK）中保持一致行为

### 运行时选择：Bun vs. Node.js vs. Python

Claude Code 选择 Bun 作为运行时，主要出于三个考量：

- **启动速度：** Bun 的冷启动时间约为 Node.js 的三分之一，对于 CLI 工具至关重要
- **原生 TypeScript：** 无需预编译步骤，直接运行 `.ts` 文件
- **Bundle 特性：** `bun:bundle` 提供编译时特性开关（`feature('X')`），可以在构建阶段消除不需要的代码路径

如果你的项目主要运行在服务器端且已有 Node.js 基础设施，Node.js 完全可行。Python 适合数据科学和 ML 场景，但在类型安全和工具生态上弱于 TypeScript。本书的代码示例以 TypeScript 编写，可在 Bun 或 Node.js 上运行。

---

## 15.2 核心组件实现路线图

构建 Agent Harness 是一个渐进过程。本节给出六个步骤，每一步都建立在前一步的基础之上，最终产出一个可运行的最小 Harness。

### Step 1：对话循环

对话循环是 Agent Harness 的心脏。Claude Code 的核心查询函数是一个 `AsyncGenerator`。让我们从设计精髓中提取灵感，实现一个精简但完整的对话循环。

首先定义核心类型：

```typescript
// types.ts
export interface Message {
  role: 'system' | 'user' | 'assistant'
  content: string | ContentBlock[]
}

export interface ContentBlock {
  type: 'text' | 'tool_use' | 'tool_result'
  text?: string
  id?: string
  name?: string
  input?: Record<string, unknown>
  content?: string | ContentBlock[]
  is_error?: boolean
  tool_use_id?: string
}

export interface StreamEvent {
  type: 'assistant' | 'tool_results' | 'error' | 'complete'
  content?: unknown
}

export interface LoopState {
  messages: Message[]
  turnCount: number
  abortController: AbortController
}
```

然后实现主循环。注意这里采用了与 Claude Code 相同的模式：`while(true)` 循环配合 `continue` 语句管理状态流转：

```typescript
// agentLoop.ts
import type { Tool } from './toolSystem'

interface AgentDeps {
  callModel: (
    messages: Message[],
    tools: Tool[],
    signal: AbortSignal,
  ) => AsyncGenerator<ContentBlock[]>
  uuid: () => string
}

export async function* agentLoop(
  messages: Message[],
  tools: Tool[],
  deps: AgentDeps,
): AsyncGenerator<StreamEvent, { reason: string }> {
  let state: LoopState = {
    messages,
    turnCount: 0,
    abortController: new AbortController(),
  }

  while (true) {
    const { messages: currentMessages, abortController } = state

    // 收集本次迭代的 assistant 消息和工具调用
    const assistantBlocks: ContentBlock[] = []
    const toolUseBlocks: ContentBlock[] = []

    try {
      // 流式调用模型
      for await (const block of deps.callModel(
        currentMessages,
        tools,
        abortController.signal,
      )) {
        assistantBlocks.push(block)
        if (block.type === 'tool_use') {
          toolUseBlocks.push(block)
        }
        yield {
          type: 'assistant' as const,
          content: block,
        }
      }
    } catch (error) {
      yield { type: 'error', content: String(error) }
      return { reason: 'model_error' }
    }

    // 没有工具调用 -- 对话结束
    if (toolUseBlocks.length === 0) {
      yield { type: 'complete' }
      return { reason: 'completed' }
    }

    // 执行工具并收集结果
    const toolResults: ContentBlock[] = []
    for (const toolCall of toolUseBlocks) {
      const tool = tools.find(t => t.name === toolCall.name)
      if (!tool) {
        toolResults.push({
          type: 'tool_result',
          tool_use_id: toolCall.id,
          content: `Unknown tool: ${toolCall.name}`,
          is_error: true,
        })
        continue
      }

      try {
        const result = await tool.execute(toolCall.input ?? {})
        toolResults.push({
          type: 'tool_result',
          tool_use_id: toolCall.id,
          content: JSON.stringify(result),
        })
      } catch (error) {
        toolResults.push({
          type: 'tool_result',
          tool_use_id: toolCall.id,
          content: `Tool error: ${error}`,
          is_error: true,
        })
      }

      yield { type: 'tool_results', content: toolResults }
    }

    // 更新状态，进入下一轮循环
    state = {
      ...state,
      messages: [
        ...currentMessages,
        {
          role: 'assistant',
          content: assistantBlocks,
        },
        {
          role: 'user',
          content: toolResults,
        },
      ],
      turnCount: state.turnCount + 1,
    }
  }
}
```

这个实现体现了从 Claude Code 源码中提炼的关键设计决策：

1. **状态对象模式：** 循环状态被封装在单一 `state` 变量中，每个 `continue` 站点写入新的 `State` 对象（对应源码中的 `const next: State = { ... }`）
2. **依赖注入：** `deps` 对象封装了所有 I/O 操作（模型调用、UUID 生成），使得测试可以注入模拟实现
3. **生成器输出：** 通过 `yield` 逐条传递事件，消费端可以实时响应

### Step 2：工具系统

Claude Code 的工具系统通过 `buildTool` 工厂函数实现了优雅的默认值填充。每个工具只需定义自己特有的部分，通用行为由工厂提供。让我们实现一个精简版本：

```typescript
// toolSystem.ts
import { z } from 'zod'

export interface Tool {
  name: string
  description: string
  inputSchema: z.ZodType<{ [key: string]: unknown }>
  execute: (input: Record<string, unknown>) => Promise<unknown>
  // 默认行为
  isEnabled: () => boolean
  isReadOnly: (input: unknown) => boolean
  isConcurrencySafe: (input: unknown) => boolean
  isDestructive: (input: unknown) => boolean
  checkPermissions: (
    input: Record<string, unknown>,
  ) => Promise<{ allowed: boolean; reason?: string }>
}

// 默认值集合 -- 对应 Claude Code 的 TOOL_DEFAULTS
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isReadOnly: () => false,
  isConcurrencySafe: () => false,
  isDestructive: () => false,
  checkPermissions: async () => ({ allowed: true }),
}

type ToolDef = Partial<typeof TOOL_DEFAULTS> &
  Omit<Tool, keyof typeof TOOL_DEFAULTS>

export function buildTool(def: ToolDef): Tool {
  return {
    ...TOOL_DEFAULTS,
    ...def,
  } as Tool
}
```

使用示例 -- 定义一个文件读取工具：

```typescript
const readFileTool = buildTool({
  name: 'read_file',
  description: 'Read the contents of a file',
  inputSchema: z.object({
    path: z.string().describe('Absolute path to the file'),
    offset: z.number().optional().describe('Line number to start reading'),
    limit: z.number().optional().describe('Maximum lines to read'),
  }),
  isReadOnly: () => true,
  isConcurrencySafe: () => true,
  async execute(input) {
    const { path, offset, limit } = input as {
      path: string
      offset?: number
      limit?: number
    }
    const fs = await import('fs/promises')
    const content = await fs.readFile(path, 'utf-8')
    const lines = content.split('\n')
    const start = offset ?? 0
    const end = limit ? start + limit : lines.length
    return lines.slice(start, end).join('\n')
  },
})
```

`buildTool` 的精髓在于 **fail-closed 默认值**：`isConcurrencySafe` 默认为 `false`，`isReadOnly` 默认为 `false`。这意味着新工具在显式声明安全性之前，系统会假设最危险的情况。这是 Claude Code 源码中注释所强调的 -- "assume not safe" 和 "assume writes"。

### Step 3：权限管线

Claude Code 的权限检查是一个四阶段管线，每个阶段都可以短路返回：

```typescript
// permissions.ts
export type PermissionDecision =
  | { allowed: true }
  | { allowed: false; reason: string }

export async function checkToolPermission(
  tool: Tool,
  input: Record<string, unknown>,
  context: PermissionContext,
): Promise<PermissionDecision> {
  // 阶段 1：工具自身验证（对应 validateInput）
  if (tool.validateInput) {
    const validation = await tool.validateInput(input, context)
    if (!validation.result) {
      return { allowed: false, reason: validation.message }
    }
  }

  // 阶段 2：工具自身权限检查（对应 checkPermissions）
  const toolPermission = await tool.checkPermissions(input)
  if (!toolPermission.allowed) {
    return { allowed: false, reason: toolPermission.reason ?? 'Denied by tool' }
  }

  // 阶段 3：用户配置规则匹配
  const ruleDecision = matchPermissionRules(tool.name, input, context.rules)
  if (ruleDecision !== 'unknown') {
    return ruleDecision === 'allow'
      ? { allowed: true }
      : { allowed: false, reason: `Blocked by ${ruleDecision} rule` }
  }

  // 阶段 4：用户交互确认（仅交互模式）
  if (context.mode === 'interactive') {
    const userChoice = await context.promptUser(
      `Allow ${tool.name} to execute?`,
    )
    return userChoice
      ? { allowed: true }
      : { allowed: false, reason: 'User denied' }
  }

  return { allowed: false, reason: 'Non-interactive: no matching rule' }
}
```

### Step 4：上下文管理

长对话必然触及 Token 上限。Claude Code 采用了 **渐进式压缩策略**，由多个层级组成：

```typescript
// contextManager.ts
export interface CompressionStrategy {
  name: string
  shouldTrigger: (tokenCount: number, limit: number) => boolean
  compress: (messages: Message[]) => Promise<Message[]>
}

// 策略 1：历史裁剪（最廉价）
const snipStrategy: CompressionStrategy = {
  name: 'snip',
  shouldTrigger: (count, limit) => count > limit * 0.7,
  async compress(messages) {
    // 保留系统消息和最近 N 条消息
    const systemMsgs = messages.filter(m => m.role === 'system')
    const recentMsgs = messages.slice(-20)
    return [...systemMsgs, ...recentMsgs]
  },
}

// 策略 2：摘要压缩（中等成本）
const summaryStrategy: CompressionStrategy = {
  name: 'summary',
  shouldTrigger: (count, limit) => count > limit * 0.9,
  async compress(messages) {
    // 使用 LLM 生成对话摘要
    const summary = await generateSummary(messages)
    return [
      messages[0], // 系统消息
      {
        role: 'user',
        content: `[Conversation summary]\n${summary}`,
      },
    ]
  },
}

export class ContextManager {
  private strategies: CompressionStrategy[] = [
    snipStrategy,
    summaryStrategy,
  ]

  async manageContext(
    messages: Message[],
    tokenCount: number,
    tokenLimit: number,
  ): Promise<{ messages: Message[]; wasCompressed: boolean }> {
    for (const strategy of this.strategies) {
      if (strategy.shouldTrigger(tokenCount, tokenLimit)) {
        const compressed = await strategy.compress(messages)
        return { messages: compressed, wasCompressed: true }
      }
    }
    return { messages, wasCompressed: false }
  }
}
```

### Step 5：记忆系统

```typescript
// memory.ts
export interface MemoryEntry {
  content: string
  source: 'extracted' | 'explicit' | 'project_file'
  timestamp: number
  relevance: number
}

export class MemoryStore {
  private store: Map<string, MemoryEntry> = new Map()

  async extractAndStore(messages: Message[]): Promise<void> {
    // 从对话中提取关键信息并持久化
    const lastExchange = messages.slice(-4)
    const extraction = await extractKeyFacts(lastExchange)
    for (const fact of extraction.facts) {
      this.store.set(fact.key, {
        content: fact.value,
        source: 'extracted',
        timestamp: Date.now(),
        relevance: fact.relevance,
      })
    }
  }

  getRelevantMemories(query: string, limit = 5): MemoryEntry[] {
    return Array.from(this.store.values())
      .filter(m => m.relevance > 0.3)
      .sort((a, b) => b.relevance - a.relevance)
      .slice(0, limit)
  }
}
```

### Step 6：钩子系统

钩子系统是 Agent Harness 的扩展层。Claude Code 支持二十多种钩子事件，每个钩子都是独立的 Shell 命令。以下是一个精简实现：

```typescript
// hooks.ts
export type HookEvent =
  | 'pre_tool_use'
  | 'post_tool_use'
  | 'session_start'
  | 'session_end'
  | 'stop'

export interface HookConfig {
  event: HookEvent
  command: string
  matcher?: string // 工具名称匹配模式
}

export interface HookResult {
  outcome: 'success' | 'blocking' | 'error'
  decision?: 'approve' | 'block'
  reason?: string
  updatedInput?: Record<string, unknown>
}

export class HookRunner {
  constructor(private config: HookConfig[]) {}

  async runHooks(
    event: HookEvent,
    input: Record<string, unknown>,
    toolName?: string,
  ): Promise<HookResult[]> {
    const matching = this.config.filter(h => {
      if (h.event !== event) return false
      if (h.matcher && toolName && !h.matcher.includes(toolName)) return false
      return true
    })

    const results: HookResult[] = []
    for (const hook of matching) {
      const result = await this.executeHook(hook, input)
      results.push(result)
      // 阻塞型结果短路
      if (result.outcome === 'blocking') break
    }
    return results
  }

  private async executeHook(
    hook: HookConfig,
    input: Record<string, unknown>,
  ): Promise<HookResult> {
    const { spawn } = await import('child_process')
    return new Promise(resolve => {
      const child = spawn(hook.command, [], {
        shell: true,
        env: { ...process.env, HOOK_INPUT: JSON.stringify(input) },
        timeout: 10_000,
      })
      let stdout = ''
      child.stdout?.on('data', d => (stdout += d))
      child.on('close', code => {
        if (code !== 0) {
          resolve({
            outcome: code === 2 ? 'blocking' : 'error',
            reason: stdout,
          })
        } else {
          resolve({ outcome: 'success' })
        }
      })
    })
  }
}
```

---

## 15.3 从 Claude Code 学到的架构教训

### 循环依赖的打破策略

在拥有六十多个工具和数百个模块的代码库中，循环依赖是头号架构杀手。Claude Code 采用了多种策略来打破循环：

**Lazy Require 模式。** 条件模块通过 `require()` 在运行时加载，结合特性开关，未使用的模块从不被加载。其模式为：使用特性开关判断是否加载，开启时通过 `require()` 获取模块，关闭时返回 null。

这种模式将导入从编译时延迟到运行时，同时结合特性开关，确保未使用的模块从不被加载。这是 TypeScript 生态中处理可选重依赖的标准做法。

**类型集中导出。** 另一个关键策略是类型从集中位置导入，而非从实现模块导入。这使得类型定义和实现解耦 -- 工具文件不需要导入整个权限系统来获取类型签名，只需从集中导出位置导入类型。

### 大型代码库的模块化设计

Claude Code 的模块化策略遵循一个核心原则：**每个工具都是独立的公民**。工具通过 `buildTool` 工厂函数定义，享有统一的接口但保持内部自治。`Tool` 类型定义了二十多个方法，但只有少数是必须实现的（`call`、`description`、`inputSchema`、`name`），其余都有合理默认值。

`Tools` 类型被定义为 `readonly Tool[]` 而非普通数组，这是刻意的设计 -- 工具集合一旦创建就不应被修改，任何变更都应通过创建新集合来实现。

### 功能开关驱动的渐进式发布

Claude Code 使用 Bun 的 `bun:bundle` 特性实现编译时特性开关。通过 `feature('X')` 函数包裹代码块，未启用的特性在构建产物中完全不存在，没有分支预测惩罚。对于需要渐进式发布的生产系统，这是一个值得借鉴的模式。

### 错误处理与断路器模式

Claude Code 的错误处理展示了一个多层防御体系：

1. **流式错误拦截：** 模型返回的错误被包装为 `AssistantMessage`（`isApiErrorMessage` 标记），而非抛出异常
2. **延迟显示：** 可恢复错误（如 prompt-too-long）被暂时扣留（withheld），尝试恢复后再决定是否展示
3. **断路器：** 压缩失败次数被追踪（`consecutiveFailures`），达到阈值后停止重试
4. **降级策略：** 模型不可用时自动切换到 fallback 模型，清理状态后重试

断路器的实现尤为精妙。源码中 `autoCompactTracking` 的 `consecutiveFailures` 字段在每次压缩失败后递增，并在成功后重置。当连续失败次数超过阈值，循环直接退出而非无限重试。

---

## 15.4 生产化考量

### 遥测与可观测性

生产环境的 Agent Harness 必须具备完善的可观测性。Claude Code 的做法值得参考：在关键路径上埋点记录消息数量、工具结果数量、查询链 ID 和查询深度等指标。

一个生产级 Harness 至少应追踪以下指标：

- **延迟分布：** 每轮循环的模型调用耗时、工具执行耗时
- **Token 预算：** 输入/输出 Token 消耗趋势，压缩频率
- **错误率：** 按错误类型分类的失败率（模型错误、工具错误、权限拒绝）
- **行为指标：** 平均每轮工具调用数、平均完成轮次、用户中断率

### 多环境适配：CLI / IDE / SDK / Server

Claude Code 的依赖注入模式提供了一种优雅的多环境适配方案。通过将 I/O 操作抽象为依赖注入接口（如模型调用、消息压缩、UUID 生成等），核心循环无需关心自己运行在何种环境中。

在 CLI 环境中，生产环境依赖提供真实的文件系统和模型调用。在测试中，可以注入模拟实现。在 SDK 模式中，可以替换为不同的 UI 反馈机制。这种 "核心 + 适配器" 的架构使得同一套 Harness 逻辑能够无缝运行在不同平台上。

关键的环境差异点包括：

| 环境 | 权限交互 | UI 渲染 | 钩子执行 | 会话存储 |
|------|----------|---------|----------|----------|
| CLI | 终端提示符 | Ink/React | Shell 命令 | 本地文件 |
| IDE (VS Code) | 弹窗 | WebView | 同上 | 同上 |
| SDK | 回调函数 | 无 | 同上 | 内存 |
| Server | 自动拒绝/允许 | 无 | HTTP 端点 | 数据库 |

### 安全审计

Agent Harness 的安全边界比传统应用更复杂，因为 LLM 的输出是不可预测的。Claude Code 在安全方面采取了多层防御：

1. **工具权限分级：** 每个工具声明 `isReadOnly`、`isDestructive`、`isConcurrencySafe`，系统根据这些标记决定审批流程
2. **工作区信任检查：** `shouldSkipHookDueToTrust()` 确保钩子不会在不受信任的工作区执行
3. **输入净化：** `backfillObservableInput` 方法在工具输入传递给观察者之前进行清理，但不修改 API 绑定的原始输入（避免破坏 Prompt 缓存）
4. **预算上限：** `maxBudgetUsd` 参数限制单次查询的 Token 开销

---

## 15.5 Agent Harness 的未来

### 多模态交互

当前的工具系统以文本为核心，但多模态交互正在改变这一格局。Claude Code 已经在处理图片输入（`ImageSizeError`、`ImageResizeError`）和 Computer Use 工具。未来的 Agent Harness 将需要：

- **原生多模态工具：** 工具的输入和输出不再局限于文本，而是包含图片、音频、视频
- **视觉理解工具：** 截图分析、UI 元素定位、图表解读
- **语音交互：** 语音输入触发工具调用，语音输出播报工具结果

### 长期运行智能体（Daemon 模式）

Claude Code 的 `taskSummaryModule` 和后台会话机制预示着一种新模式：智能体不再是一次性的命令行工具，而是长期运行的守护进程。这种 Daemon 模式的智能体具备：

- **持久化状态：** 跨会话保持上下文和记忆
- **事件驱动唤醒：** 文件变更、定时任务、外部通知触发行动
- **多智能体协作：** 主智能体协调多个子智能体（Claude Code 的 `AgentTool` 已实现此模式）
- **资源感知调度：** 根据 Token 预算、API 配额、系统负载智能调度任务

### 标准化协议（MCP 演进方向）

Model Context Protocol (MCP) 正在成为工具调用的事实标准。Claude Code 已经深度集成了 MCP：工具可以来自 MCP 服务器，`mcp_info` 字段追踪工具来源，`handleElicitation` 支持 MCP 服务器的交互式请求。

未来的 MCP 演进方向包括：

- **标准化工具发现：** 智能体在运行时动态发现可用工具
- **跨智能体通信协议：** 不同厂商的智能体通过 MCP 交换消息和工具调用
- **沙箱化执行：** MCP 服务器在隔离环境中执行工具，提供更强的安全保证
- **流式结果传输：** 工具执行结果通过 MCP 协议流式返回，支持长时间运行的工具

---

## 实战练习

选择以下场景之一，设计一个完整的 Agent Harness 架构：

**场景 A：代码审查智能体**
- 工具集：Git 操作、文件读取、静态分析、评论发布
- 权限模型：只读模式 + 评论写入需确认
- 上下文策略：按 PR 维度压缩，保留变更摘要
- 钩子：自动运行 lint、类型检查，结果注入上下文

**场景 B：运维监控智能体**
- 工具集：日志查询、指标获取、部署操作、告警管理
- 权限模型：查询自动通过，操作需双人确认
- 上下文策略：滑动窗口 + 异常事件优先保留
- 钩子：Webhook 通知、审计日志记录

**场景 C：文档生成智能体**
- 工具集：代码分析、文档模板、版本对比、文件写入
- 权限模型：自动模式（信任写入目标目录）
- 上下文策略：项目级记忆，跨会话保持风格偏好
- 钩子：格式检查、链接验证

对于你选择的场景，请完成以下设计：

1. 绘制组件关系图（对话循环、工具系统、权限管线、上下文管理、记忆系统、钩子系统之间的调用关系）
2. 定义至少三个工具的完整 `buildTool` 定义
3. 设计权限管线的四个阶段检查逻辑
4. 选择压缩策略并说明触发条件
5. 定义至少两个钩子及其预期行为

---

## 关键要点

1. **循环状态模式：** 使用 `while(true)` + `State` 对象 + `continue` 管理循环状态，比递归调用更易调试和维护。Claude Code 的核心查询函数用约 1700 行代码实现了这一模式，处理了十余种状态转换路径。

2. **工厂函数 + Fail-closed 默认值：** `buildTool` 模式让工具定义保持精简，默认值选择最保守的策略（不安全、有副作用、需确认），工具显式覆盖以声明安全性。

3. **依赖注入隔离 I/O：** `QueryDeps` 模式将所有外部依赖抽象为可注入接口，使得核心逻辑可以在不同环境（CLI、SDK、测试）中复用。

4. **渐进式压缩：** 多层压缩策略（裁剪、微压缩、摘要）根据 Token 使用率逐级触发，避免一刀切的信息丢失。

5. **断路器保护：** 连续失败计数、最大恢复次数、降级策略 -- 这些机制确保智能体在面对持续错误时优雅降级，而非无限循环。

6. **特性开关驱动的渐进式发布：** 编译时特性开关让新功能可以安全地渐进发布，未启用的代码在产物中完全不存在。

7. **安全纵深防御：** 从工具级别（`isDestructive`）到系统级别（工作区信任、预算上限），每一层都提供独立的安全保证。

---

**全书结语**

从第一章理解 Agent Harness 的概念，到本章亲手构建一个自定义实现，我们走过了一段完整的旅程。Claude Code 作为工业级 Agent Harness 的标杆，展示了这项技术在实际生产环境中的样貌：不是简单的 API 调用封装，而是一个融合了对话管理、工具编排、权限控制、上下文工程、记忆系统和可观测性的完整软件架构。

Agent Harness 代表了一种新的软件范式 -- 不是程序员告诉机器每一步该做什么，而是程序员构建一个框架，让机器在这个框架内自主决策。这个框架的质量，决定了智能体的能力上限和安全下限。

未来的软件开发者，需要同时掌握两种能力：传统的确定性编程，以及这种新的 "元编程" -- 为 AI 智能体构建 Harness。希望本书能为你在这条路上的探索提供一个坚实的起点。
