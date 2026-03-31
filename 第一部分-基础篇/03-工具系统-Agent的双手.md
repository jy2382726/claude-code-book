# 第3章：工具系统 -- Agent 的双手

> "If all you have is a hammer, everything looks like a nail."
> -- Abraham Maslow

**学习目标：** 掌握 Claude Code 45+ 工具的设计模式，理解工具定义协议、注册机制、编排引擎的完整架构。

---

## 3.1 工具定义协议

Claude Code 的每个工具都遵循一个统一的类型契约 -- `Tool<Input, Output, Progress>`。这个契约定义在工具类型核心模块中，是整个工具系统的基石。理解它，就理解了 Agent "双手"的解剖结构。

### 核心类型：Tool、Tools、ToolDef、buildTool

`Tool` 类型是一个泛型接口，接受三个类型参数：

- `Input extends AnyObject`：使用 Zod schema 定义的工具输入类型，确保每个工具的输入都是一个结构化对象。
- `Output`：工具的输出类型，自由定义。
- `P extends ToolProgressData`：工具的进度数据类型，用于流式反馈。

每个工具必须实现的五要素如下：

**要素一：名称与别名**

每个工具拥有一个唯一的名称标识符，以及可选的别名用于向后兼容。当工具重命名时，旧名称可以通过别名继续匹配。工具查找函数同时检查主名称和别名。

**要素二：Zod Schema**

每个工具使用 Zod 定义其输入参数的 schema。Zod schema 不仅用于运行时验证，还通过转换层生成 JSON Schema 发送给 API，让模型知道每个参数的含义和约束。

**要素三：权限模型**

权限相关的三个方法构成了分层的权限检查管线：第一层是输入验证，在权限检查之前运行，用于拒绝无效输入；第二层是权限检查，包含工具特定的权限逻辑；第三层是运行时属性判断，影响工具的并发调度策略。

**要素四：执行逻辑**

这是工具的核心执行方法。它接收解析后的输入参数、工具使用上下文、权限检查函数、父消息引用和一个可选的进度回调。返回的结果携带输出数据和可选的上下文修改器。

上下文修改器允许工具在执行后修改上下文（如更新文件缓存），这是工具影响后续行为的关键通道。

**要素五：UI 渲染**

工具拥有丰富的渲染方法集合，覆盖了完整的 UI 生命周期：

- `renderToolUseMessage`：工具调用开始时展示（如 "Reading src/foo.ts"）
- `renderToolUseProgressMessage`：工具执行中的进度展示
- `renderToolResultMessage`：工具结果展示
- `renderToolUseRejectedMessage`：权限被拒绝时的展示
- `renderToolUseErrorMessage`：执行出错时的展示
- `renderGroupedToolUse`：多个并行工具的分组展示

每个渲染方法都返回 `React.ReactNode`，使得工具系统与 React 渲染管线深度集成。

### buildTool 工厂函数

`buildTool` 是创建工具的标准工厂函数。它接收一个部分工具定义，自动填充安全默认值。这些默认值遵循"fail-closed"原则：安全性相关的方法（如并发安全判断、只读判断）默认为 false，工具必须显式声明自己安全才能享受并发等优化。

类型系统通过巧妙的类型计算，让开发者只需提供必要字段，而工厂函数的返回类型保证完整的工具接口。如果开发者在定义中提供了某个方法，类型系统会使用开发者提供的签名；如果省略了，则使用默认签名。

---

## 3.2 工具注册与动态发现

### getAllBaseTools() 完整工具清单

`getAllBaseTools()` 是所有内建工具的注册中心。它返回一个扁平数组，包含了 Claude Code 所有可用的工具，包括执行类、文件操作类、搜索类、网络类、智能体类、任务管理类、规划类等。

通过这个函数，我们可以统计出核心工具清单：

| 类别 | 工具 | 职责 |
|------|------|------|
| 执行 | BashTool | 运行 Shell 命令 |
| 文件 | FileReadTool, FileEditTool, FileWriteTool | 读取、编辑、写入文件 |
| 搜索 | GlobTool, GrepTool | 文件名模式匹配、内容搜索 |
| 笔记本 | NotebookEditTool | Jupyter Notebook 编辑 |
| 网络 | WebFetchTool, WebSearchTool | 获取 URL 内容、网络搜索 |
| 智能 | AgentTool | 子智能体入口 |
| 任务 | TodoWriteTool, TaskCreateTool, TaskGetTool, TaskUpdateTool, TaskListTool, TaskStopTool | 任务管理 |
| 规划 | EnterPlanModeTool, ExitPlanModeV2Tool | 计划模式切换 |
| 交互 | AskUserQuestionTool | 向用户提问 |
| 技能 | SkillTool | 调用 slash command 技能 |
| 配置 | ConfigTool | 修改配置 |
| MCP | ListMcpResourcesTool, ReadMcpResourceTool | MCP 资源访问 |
| 工作树 | EnterWorktreeTool, ExitWorktreeTool | Git worktree 管理 |
| 通知 | BriefTool | 消息发送 |
| 搜索发现 | ToolSearchTool | 延迟工具发现 |

### 死代码消除在工具注册中的应用

Claude Code 的工具注册大量使用条件导入实现编译期死代码消除。当特定条件不满足时，对应工具的整个模块不会被包含在最终构建中。功能开关控制的工具也是如此。

功能开关来自构建工具链，在编译时由 bundler 评估。当功能开关关闭时，对应的工具实现代码都会被 tree-shaking 移除。这种模式确保了外部构建（面向第三方用户）不会包含内部工具的代码。

在工具注册函数中，条件注册使用展开运算符，根据运行环境和功能开关决定是否加入特定工具。

### ToolSearchTool 延迟发现机制

当工具数量超过一定阈值时，Claude Code 启用延迟工具发现（deferred tool discovery）。核心思路是：不在初始系统提示中发送所有工具的完整 schema，而是只发送工具名称列表，让模型通过 ToolSearchTool 按需加载。

ToolSearchTool 的实现遵循标准的工厂函数模式。判断一个工具是否应该被延迟的逻辑是：显式标记为总是加载的工具不延迟、MCP 工具总是延迟、工具搜索工具自身不延迟。

这个机制的核心价值在于节省 prompt 空间：当 MCP 服务器注册了数十个工具时，全部发送给 API 会消耗大量 token。延迟发现让模型只在需要时加载工具的完整 schema，显著减少了初始 prompt 的大小。

### 工具过滤管线

从 `getAllBaseTools()` 到最终发送给 API 的工具列表，经过多层过滤：

1. **模式过滤**：根据模式过滤工具。简单模式只保留 Bash、Read、Edit；普通模式排除特殊工具。
2. **拒绝规则过滤**：移除被 blanket deny 规则匹配的工具。
3. **启用状态检查**：过滤掉未启用的工具。
4. **工具池组装**：合并内建工具与 MCP 工具，按名称排序去重。排序的目的是确保 prompt 缓存稳定性 -- 工具顺序变化会导致缓存失效。

---

## 3.3 核心工具深度解析

### BashTool：命令执行的瑞士军刀

BashTool 是 Claude Code 最强大的工具之一，也是最复杂的。它不仅仅是简单的 Shell 执行器，而是集成了多层安全防护的执行环境。

BashTool 在工具系统中的特殊地位体现在以下方面：

- **错误传播**：当 BashTool 执行失败时，会取消所有并行的 Bash 工具调用。这是因为 Bash 命令之间往往存在隐式依赖链（如 `mkdir` 失败后后续命令无意义）。
- **中断行为**：BashTool 可以自定义用户中断时的行为。某些长时间运行的命令（如测试套件）可能选择阻塞而非取消。
- **语义分析**：BashTool 会对命令进行 AST 解析和语义分析，判断命令是否为搜索/读取操作（`isSearchOrReadCommand`），用于 UI 折叠展示。
- **沙盒集成**：通过 `--dangerouslyDisableSandbox` 参数和沙盒配置，控制命令执行的安全边界。

### 文件三件套：FileReadTool、FileEditTool、FileWriteTool

这三个工具构成了 Claude Code 文件操作的完整能力集：

**FileReadTool** 负责读取文件内容。它维护了文件状态缓存，用于追踪哪些文件已被读取，避免重复注入内存附件。

**FileEditTool** 负责精确编辑文件。它使用 `old_string -> new_string` 的精确替换模式，而非行号范围，确保编辑操作在文件变化时仍然正确。FileEditTool 的 `isDestructive` 方法会根据编辑内容判断是否为破坏性操作（如删除大量代码）。

**FileWriteTool** 负责创建或完全覆写文件。这是最"重"的文件操作，权限检查最为严格。

三个工具都支持 `contextModifier`，在执行后更新文件状态缓存，使得后续的工具调用和内存附件注入能看到最新的文件状态。

### 搜索双雄：GlobTool 与 GrepTool

**GlobTool** 使用文件名模式匹配查找文件，底层使用 `fast-glob` 库。它返回匹配的文件路径列表，支持 ignore 模式和最大结果数限制。

**GrepTool** 使用正则表达式搜索文件内容，底层使用 `ripgrep`。它支持多种输出模式（文件名、内容行、计数），以及丰富的过滤选项（文件类型、glob 模式等）。

值得注意的是，当 Ant 原生构建中嵌入了专用的快速搜索工具时，GlobTool 和 GrepTool 会被禁用，因为 Shell 中的 `find` 和 `grep` 命令已经被别名为这些快速工具，BashTool 可以直接使用它们。

### AgentTool：子智能体入口

AgentTool 是 Claude Code 实现多智能体协作的核心工具。它允许主智能体生成子智能体来处理子任务。子智能体拥有独立的上下文窗口和工具集，执行完成后将结果返回给主智能体。

AgentTool 在工具系统中有几个特殊属性：
- 它可能被标记为 `alwaysLoad`，确保在 ToolSearch 启用时仍然在第一轮可见。
- 子智能体通过 `createSubagentContext` 创建独立的 `ToolUseContext`，继承父上下文的部分状态（如权限规则）但拥有独立的消息列表。
- 子智能体的结果通过 `TaskOutputTool` 暴露给主智能体。

---

## 3.4 工具编排引擎

### runTools() 函数与并发分区

工具编排的核心逻辑在工具编排模块中。`runTools` 函数是一个异步生成器，负责调度一批工具调用的执行。

它的调度策略基于 **并发分区**（concurrency partitioning）：

1. 首先将所有工具调用按顺序划分为批次。
2. 每个批次要么是一组连续的并发安全工具，要么是一个单独的非安全工具。
3. 并发安全批次并行执行。
4. 非安全批次串行执行。

分区算法的核心逻辑：遍历所有工具调用，检查每个工具的并发安全属性。如果当前工具安全且前一个批次也安全，则合并到同一批次；否则开启新批次。

举例来说，如果模型请求了四个工具调用：`[Read(a.ts), Read(b.ts), Bash(ls), Read(c.ts)]`，分区结果为：

```
Batch 1: { isConcurrencySafe: true,  blocks: [Read(a.ts), Read(b.ts)] }  -- 并行
Batch 2: { isConcurrencySafe: false, blocks: [Bash(ls)] }                  -- 串行
Batch 3: { isConcurrencySafe: true,  blocks: [Read(c.ts)] }                -- 并行
```

并发执行的并发度上限由环境变量控制，默认为 10。

### StreamingToolExecutor 流式执行

`StreamingToolExecutor` 是 `runTools` 的增强版本，它不等待模型响应完全结束就开始执行工具，而是在流式接收到工具调用块时就立即启动执行。

每个被追踪的工具拥有四阶段状态机：

```
queued --> executing --> completed --> yielded
```

- **queued**：工具已入队，等待执行条件满足。
- **executing**：正在执行。执行前会检查并发条件：只有当没有工具在执行，或所有执行中的工具都是并发安全时，才允许开始执行。
- **completed**：执行完成，结果已收集。但尚未 yield 给上层（需要维持顺序）。
- **yielded**：结果已产出，工具生命周期结束。

StreamingToolExecutor 的关键设计决策：

1. **顺序保证**：即使在流式执行中工具可以并行完成，结果的 yield 仍然保持与请求相同的顺序。结果收集函数在遍历工具列表时，遇到未完成的非安全工具就会停止，确保顺序约束不被违反。

2. **错误传播**：BashTool 执行失败会取消所有并行兄弟工具。非 Bash 工具的错误不会传播 -- 因为读取/搜索操作通常是独立的。

3. **进度即时产出**：工具执行中的进度消息绕过顺序约束，立即 yield 给上层。这使得 UI 可以实时显示工具执行进度，而不必等待前面的工具完成。

4. **丢弃机制**：当流式回退发生时（模型切换到 fallback 模型），标记所有待执行和执行中的工具为废弃，避免过时的结果泄漏。

5. **信号传播**：每个工具执行使用独立的子取消控制器，形成层级化的取消信号链。兄弟工具的错误或用户的 Ctrl+C 都会通过信号传播到正确的工具。

### 工具状态机在对话主循环中的集成

回到对话主循环，工具执行的状态与对话循环的状态紧密集成：

1. **流式执行路径**：当流式工具执行功能启用时，创建 StreamingToolExecutor。在流式接收工具调用块时，立即将工具加入执行队列，同时检查是否有已完成的结果可以立即 yield。

2. **批量执行路径**：当流式执行不可用时，使用传统的批量执行函数在模型响应完全结束后批量执行所有工具。

3. **上下文传播**：工具执行后可能修改上下文（如文件缓存更新），这些修改传播回对话循环，影响后续工具的执行环境。

---

## 实战练习

**练习 1：实现一个自定义工具**

使用 `buildTool` 工厂函数创建一个简单的工具。要求：
- 定义 Zod schema，包含一个 `path` 字段（字符串类型）
- 实现 `call` 方法，返回指定路径的文件信息
- 正确标记 `isReadOnly` 和 `isConcurrencySafe`
- 实现 `renderToolUseMessage` 和 `renderToolResultMessage`

对比你的实现与 FileReadTool 的差异，理解 `buildTool` 默认值的作用。

**练习 2：分析并发分区策略**

给定以下工具调用序列：
```
[GlobTool(*.ts), GrepTool(pattern), BashTool(npm test), FileReadTool(a.ts), FileEditTool(a.ts), GlobTool(*.json)]
```

手动执行 `partitionToolCalls` 的逻辑，画出批次划分结果。然后思考：为什么 FileEditTool 和 GlobTool 不能放在同一个并发批次？

**练习 3：追踪 StreamingToolExecutor 的生命周期**

在 StreamingToolExecutor 的工具入队、工具执行和结果获取环节设置断点。发送一个触发多个并行工具调用的请求（如"搜索所有 TODO 注释并读取相关文件"），观察工具状态如何从 queued 到 executing 到 completed 再到 yielded，以及进度消息如何即时产出。

---

## 关键要点

1. **五要素协议是工具系统的 DNA**：名称、Schema、权限、执行、渲染。每个工具都围绕这五个维度定义自己，而 `buildTool` 的默认值机制让简单工具只需关注核心逻辑。

2. **死代码消除保障构建安全**：通过环境变量和功能开关的条件导入，确保内部工具不会泄漏到外部构建。这是 Agent 系统在多租户环境下的重要工程实践。

3. **并发分区是性能的关键**：`isConcurrencySafe` 的判断决定了工具能否并行执行。正确标记只读工具为并发安全，可以让 Agent 在单轮中同时执行多个搜索/读取操作，大幅减少响应时间。

4. **StreamingToolExecutor 是零等待的工具调度器**：在模型还在生成 tool_use 块时就开始执行工具，通过四阶段状态机和顺序保证，在并行性和一致性之间取得平衡。

5. **工具系统的可扩展性来自类型契约**：`Tool<Input, Output, Progress>` 的泛型设计让每个工具有独立的类型空间，而 `ToolUseContext` 提供了统一的执行环境。添加新工具不需要修改编排引擎的代码。
