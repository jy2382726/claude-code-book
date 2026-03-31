# 第12章 MCP 集成与外部协议

> "协议是系统间沟通的语言，好的协议让集成变成组合而非编码。"
> -- 《分布式系统设计》改编

**学习目标：** 理解 Model Context Protocol（MCP）在 Claude Code 中的集成架构；掌握连接管理、工具映射和安全策略；了解 Bridge 系统如何实现 IDE 集成和远程控制。

---

## 12.1 MCP 架构概览

Model Context Protocol（MCP）是 Anthropic 推出的开放标准，旨在让大语言模型以统一的方式连接外部数据源和工具。Claude Code 作为 MCP 的核心客户端实现，集成了完整的连接管理、工具发现和安全策略。

### 12.1.1 支持的传输协议

Claude Code 支持多种 MCP 传输协议，包括 stdio、sse、sse-ide、http、ws 和 sdk 等类型。

每种协议对应不同的服务器配置类型：

| 协议类型 | 配置类型 | 使用场景 |
|---------|---------|---------|
| `stdio` | `McpStdioServerConfig` | 本地子进程通信，最常用。通过 `command` 和 `args` 启动本地 MCP 服务器进程 |
| `sse` | `McpSSEServerConfig` | Server-Sent Events 连接，用于远程 HTTP 服务 |
| `sse-ide` | `McpSSEIDEServerConfig` | IDE 扩展专用的 SSE 连接，包含 `ideName` 字段 |
| `http` | `McpHTTPServerConfig` | HTTP Streamable 连接（MCP 规范新协议） |
| `ws` | `McpWebSocketServerConfig` | WebSocket 连接 |
| `ws-ide` | `McpWebSocketIDEServerConfig` | IDE 扩展专用的 WebSocket 连接 |
| `sdk` | `McpSdkServerConfig` | SDK 内部调用，不启动实际进程或网络连接 |
| `claudeai-proxy` | `McpClaudeAIProxyServerConfig` | Claude.ai 平台代理服务器 |

其中 `stdio` 是最常见的类型，通过 `command` 和 `args` 启动本地 MCP 服务器进程，并可传入环境变量。`sse` 类型用于远程 HTTP 服务，通过 `url` 和 `headers` 配置连接。

### 12.1.2 连接管理器 MCPConnectionManager

MCP 连接管理器是一个 React Context Provider，为整个组件树提供 MCP 连接管理能力。其核心接口提供两个操作：重新连接指定 MCP 服务器（返回更新后的工具列表、命令列表和资源列表）以及启用/禁用指定 MCP 服务器。

子组件通过两个 Hook 访问这些操作：一个获取重连函数，一个获取切换函数。两者都要求在 MCPConnectionManager 组件树内部使用，否则会抛出错误。

### 12.1.3 服务器连接状态

MCP 服务器连接有五种状态：已连接（Connected）、连接失败（Failed）、需要认证（NeedsAuth）、等待连接（Pending，支持重试）和已禁用（Disabled）。

这是一个典型的有限状态机设计：

```
            +---> Connected <---+
            |        |          |
    connect |    error|    reconnect
            |        v          |
            +--- Failed         |
                 |              |
            retry|              |
                 v              |
              Pending -------->+
                 |
           disable|
                 v
              Disabled
```

`ConnectedMCPServer` 包含完整的 MCP Client 实例、服务器能力声明、配置信息和清理函数。`PendingMCPServer` 记录了重连尝试次数和最大重连次数，支持指数退避重试。

---

## 12.2 MCP 工具集成

### 12.2.1 工具发现与映射

MCP 服务器连接成功后，Claude Code 通过 `tools/list` 请求获取服务器提供的工具列表。工具映射的核心逻辑包括：对返回的工具列表进行 Unicode 清理，根据配置决定是否添加服务器名前缀，然后将每个 MCP 工具映射为内部 Tool 对象。

每个 MCP 工具被映射为 Claude Code 内部的 `Tool` 对象，包含以下关键属性：

- `isMcp: true`：标识这是一个 MCP 工具
- `mcpInfo`：包含 `serverName` 和 `toolName` 的元信息，用于权限检查和显示
- `isConcurrencySafe()`：映射自 MCP 工具注解的 `readOnlyHint`
- `isDestructive()`：映射自 `destructiveHint`
- `isOpenWorld()`：映射自 `openWorldHint`
- `alwaysLoad`：当工具声明 `_meta.anthropic/alwaysLoad === true` 时，跳过延迟加载直接注入 system prompt

### 12.2.2 工具名称前缀：mcp__server__tool

MCP 工具采用统一的三段式命名规则 `mcp__{server}__{tool}`。命名工具函数将服务器名和工具名拼接为全限定名，解析函数则执行反向操作，按双下划线拆分提取服务器名和工具名。

关键设计细节：

**双下划线分隔**：工具名格式为 `mcp__{server}__{tool}`。如果服务器名包含双下划线，解析可能不准确——但这在实践中极少发生。

**SDK 前缀跳过**：当环境变量 `CLAUDE_AGENT_SDK_MCP_NO_PREFIX` 设置且服务器类型为 `sdk` 时，工具使用原始名称注册，允许 MCP 工具按名称覆盖内置工具。

**权限检查使用全限定名**：`getToolNameForPermissionCheck` 函数确保权限规则匹配使用 `mcp__server__tool` 格式，防止名为 "Write" 的 MCP 工具意外匹配到内置 Write 工具的权限规则。

### 12.2.3 MCP 工具的权限模型

MCP 工具的权限检查默认返回 `passthrough` 行为，意味着每次调用都需要用户确认。但系统提供了自动允许的路径，用户可以在 `.claude/settings.local.json` 中添加规则来预授权特定的 MCP 工具。

```json
{
  "permissions": {
    "allow": [
      "mcp__my_server__read_file",
      "mcp__github__*"
    ]
  }
}
```

---

## 12.3 MCP 权限与安全

### 12.3.1 配置范围与作用域

MCP 服务器的配置有七个作用域：local（项目本地配置）、user（用户全局配置）、project（项目共享配置）、dynamic（运行时动态添加）、enterprise（企业管理配置）、claudeai（Claude.ai 平台连接器）和 managed（托管策略配置）。

### 12.3.2 服务器审批与白名单

企业管理员可以通过 `allowedMcpServers` 和 `deniedMcpServers` 策略控制哪些 MCP 服务器可以使用。这是三层安全策略的核心。

**黑名单（Denylist）**：绝对优先级。支持三种匹配方式：按服务器名称匹配、按命令数组匹配（stdio 服务器）和按 URL 模式匹配（远程服务器，支持通配符）。

**白名单（Allowlist）**：如果定义了白名单，则只有白名单中的服务器被允许。白名单同样支持名称、命令和 URL 三种匹配方式。

**URL 通配符匹配**：支持 `*` 通配符，例如：

```
"https://example.com/*"       匹配 "https://example.com/api/v1"
"https://*.example.com/*"     匹配 "https://api.example.com/path"
"https://example.com:*\/*"    匹配任意端口
```

**策略过滤函数** `filterMcpServersByPolicy` 是所有配置入口的统一过滤器，遍历所有服务器配置，将 SDK 类型和通过策略检查的服务器放入允许列表，其余放入阻止列表。注意 SDK 类型的服务器被豁免于策略检查——它们是 SDK 管理的传输占位符，CLI 不会为它们启动进程或打开网络连接。

### 12.3.3 插件去重

当插件提供的 MCP 服务器与手动配置的服务器指向相同的底层进程/URL 时，系统会自动去重。去重函数使用签名比较：stdio 服务器以命令数组的 JSON 字符串作为签名，远程服务器以原始 URL 作为签名（如果 URL 是 CCR 代理路径，会先解包获取真实 URL），SDK 服务器签名为 null 不做去重。

签名规则：

- `stdio` 服务器：签名 = `stdio:${JSON.stringify([command, ...args])}`
- 远程服务器：签名 = `url:${原始URL}`（如果 URL 是 CCR 代理路径，会先解包获取真实 URL）
- SDK 服务器：签名为 `null`，不做去重

去重优先级：**手动配置 > 插件配置 > Claude.ai 连接器**。这确保用户的显式配置始终优先。

### 12.3.4 IDE 工具白名单

对于 IDE 集成的 MCP 服务器，系统进一步限制了可用的工具范围，只有 `mcp__ide__executeCode` 和 `mcp__ide__getDiagnostics` 被允许加载。这防止了恶意的 IDE 扩展通过 MCP 协议暴露危险工具。

---

## 12.4 IDE 集成：Bridge 系统

Bridge 系统是 Claude Code 与外部世界双向通信的核心层。它实现了与 VS Code、JetBrains 等 IDE 的集成，以及 Claude.ai 平台的远程控制功能。

### 12.4.1 架构概览

Bridge 系统位于 bridge 通信目录中，包含超过 30 个模块文件，构成了一个完整的通信层。关键组件如下：

| 职责 | 说明 |
|------|------|
| REST API 客户端 | 与 claude.ai 后端通信 |
| 消息路由和去重 | 处理入站/出站消息 |
| 传输层抽象 | 封装 v1 (HybridTransport) 和 v2 (SSE + CCR) |
| REPL 桥接核心 | 管理会话生命周期 |
| 远程控制核心逻辑 | 管理远程控制功能 |
| Bridge 功能门控和权限检查 | 控制功能访问 |
| 类型定义 | 定义通信类型 |
| 会话创建和子进程管理 | 管理会话生命周期 |
| JWT 工具 | 用于认证 |

### 12.4.2 双向通信层

Bridge 的通信架构是双向的，支持两类数据流：

**出站（CLI -> 外部）**：CLI 中的对话消息、工具调用结果、状态更新等通过传输层发送到外部消费者（IDE、claude.ai 等）。

**入站（外部 -> CLI）**：来自 IDE 或 claude.ai 的用户消息、权限响应、控制请求等通过传输层到达 CLI。

消息路由的核心逻辑实现了三重过滤：首先处理权限响应，其次处理控制请求（initialize、set_model、interrupt 等），最后处理用户消息——此时需要检查回声过滤（该消息是否已作为出站发送过）和重投递过滤（是否已处理过该入站消息）。

`BoundedUUIDSet` 是一个 FIFO 有界集合，使用环形缓冲区实现，内存占用恒定为 O(capacity)，用于高效地检测重复消息。

### 12.4.3 控制协议

服务器可以发送控制请求来远程管理 CLI 会话。`handleServerControlRequest` 函数支持以下控制子类型：

| 子类型 | 用途 | 响应 |
|--------|------|------|
| `initialize` | 初始化握手，报告能力 | 返回 commands, output_style, models, account, pid |
| `set_model` | 远程切换模型 | 调用 `onSetModel` 回调 |
| `set_max_thinking_tokens` | 调整思考 token 预算 | 调用 `onSetMaxThinkingTokens` 回调 |
| `set_permission_mode` | 切换权限模式 | 调用 `onSetPermissionMode` 回调，返回策略裁决 |
| `interrupt` | 中断当前执行 | 调用 `onInterrupt` 回调 |

在 **outbound-only** 模式下，除 `initialize` 外的所有可变请求都被拒绝，返回错误信息告知需要本地启用 Remote Control 才能允许入站控制。

### 12.4.4 传输层抽象

传输层定义了统一的 `ReplBridgeTransport` 接口，支持两种实现：

**v1 适配器**（`createV1ReplTransport`）：封装 `HybridTransport`，使用 WebSocket 读取 + HTTP POST 写入到 Session-Ingress 服务。

**v2 适配器**：封装 SSETransport（读取）+ CCRClient（写入到 CCR v2 端点）。v2 的关键改进包括：SSE 序列号延续（传输切换时携带上一个流的序列号高位标记，避免服务器重放完整历史）、Epoch 管理和心跳（CCRClient 定期发送心跳维持租约）、多会话安全的认证（通过闭包提供每实例认证，避免多会话场景下的环境变量竞争）。

### 12.4.5 VS Code 和 JetBrains 扩展集成

IDE 扩展通过 `sse-ide` 和 `ws-ide` 类型的 MCP 服务器与 Claude CLI 通信。这些是内部专用类型，包含 IDE 标识信息（如 IDE 名称、是否在 Windows 上运行）。

集成流程如下：

1. IDE 扩展启动 MCP 服务器，监听本地端口
2. IDE 扩展将服务器 URL 通过环境变量或 CLI 参数传递给 Claude Code
3. Claude Code 建立 SSE/WebSocket 连接
4. IDE 扩展通过 MCP 协议提供文件操作、诊断信息等工具
5. Claude Code 将这些工具映射为 `mcp__ide__*` 命名空间的工具

IDE 工具受到白名单限制，只有 `mcp__ide__executeCode` 和 `mcp__ide__getDiagnostics` 被允许加载。

### 12.4.6 Bridge 权限门控

Bridge（远程控制）功能需要 claude.ai 订阅，并通过多层门控：首先检查 feature gate 是否开启，然后验证用户是否为 claude.ai 订阅者，以及 GrowthBook 功能标志是否开启。

完整的诊断函数 `getBridgeDisabledReason` 依次检查：

1. 是否为 claude.ai 订阅者（排除 Bedrock/Vertex/Foundry 等）
2. 是否具有完整的 profile scope（排除 setup-token 等受限 token）
3. 是否有组织 UUID（排除信息不完整的登录）
4. GrowthBook 功能门控 `tengu_ccr_bridge` 是否开启

API 客户端的认证使用 OAuth Bearer Token，并支持 401 时的自动 Token 刷新和重试。

---

## 实战练习

### 练习 1：配置一个 stdio 类型的 MCP 服务器

在项目根目录创建 `.mcp.json`：

```json
{
  "mcpServers": {
    "filesystem": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"],
      "env": {}
    }
  }
}
```

启动 Claude Code 后，验证工具是否加载：
- 观察启动日志中的 MCP 连接状态
- 尝试使用 `mcp__filesystem__*` 前缀的工具

### 练习 2：理解工具名称解析

根据 `mcp__{server}__{tool}` 的命名规则，分析以下场景：
- 解析 `mcp__github__create_issue` 会得到什么结果？
- 构建 `mcp__my_server__read_file` 需要什么输入？
- 包含双下划线的工具名 `mcp__my__special__tool` 会如何解析？

### 练习 3：配置企业级 MCP 安全策略

在企业管理配置中设置白名单和黑名单：

```json
{
  "allowedMcpServers": [
    { "serverName": "approved-server" },
    { "serverCommand": ["npx", "-y", "@modelcontextprotocol/server-filesystem"] },
    { "serverUrl": "https://mcp.company.com/*" }
  ],
  "deniedMcpServers": [
    { "serverName": "dangerous-server" }
  ]
}
```

测试不同配置的服务器是否能被正确允许或阻止。

---

## 关键要点

1. **多协议支持**：MCP 支持 stdio、SSE、HTTP、WebSocket 四种外部协议，以及 sdk（内部）、sse-ide/ws-ide（IDE 专用）、claudeai-proxy（平台代理）三种内部类型
2. **三段式命名**：`mcp__{server}__{tool}` 命名规则确保不同服务器的工具不会冲突，同时支持 SDK 模式下的前缀跳过以覆盖内置工具
3. **多层安全策略**：denylist（绝对优先）-> allowlist（白名单门控）-> IDE 工具白名单（细粒度控制）-> 每次调用的权限确认
4. **签名去重**：通过 `stdio:JSON.stringify([cmd,...args])` 和 `url:originalUrl` 签名确保手动配置优先于插件和连接器
5. **Bridge 双向通信**：通过统一的 `ReplBridgeTransport` 接口抽象 v1/v2 传输差异，支持回声去重（`BoundedUUIDSet`）、控制协议和仅出站模式
6. **SSE 序列号延续**：v2 传输在切换时携带序列号高位标记，避免服务器重放完整会话历史，是解决"消息风暴"问题的关键设计
7. **权限门控**：Bridge 功能需要 claude.ai 订阅 + 组织 UUID + GrowthBook 功能标志三重验证，确保只有授权用户可以使用远程控制
