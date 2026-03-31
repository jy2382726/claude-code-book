# 第5章：设置与配置 -- Agent 的基因

> **学习目标：** 理解六层配置源的合并规则和安全边界，掌握功能开关系统的编译时优化机制，以及 Zustand-like 不可变状态存储的设计哲学。

---

Claude Code 的行为不是由某个单一配置文件决定的，而是由六层配置源逐层合并而成。这些配置源就像 Agent 的"基因"——它们在 Agent 启动前就已写定，决定了 Agent 能做什么、不能做什么、以及以何种方式去做。理解这套体系，是驾驭 Claude Code 行为定制能力的第一步。

## 5.1 六层配置源的优先级体系

### 5.1.1 配置源定义与顺序

Claude Code 的配置源在配置常量模块中定义为一个有序数组，包含五种配置来源：用户全局设置（userSettings）、项目共享设置（projectSettings）、项目本地设置（localSettings，gitignored）、CLI 标志设置（flagSettings）和企业策略设置（policySettings）。

配置加载函数 `loadSettingsFromDisk()` 遵循一个关键原则：**后加载者覆盖前者**。合并并非简单的全量替换，而是使用深度合并策略配合自定义规则。

实际执行时还有一个隐藏的最低优先级层：**pluginSettings**（插件设置）。在配置加载流程中，插件设置首先被加载为合并的基础，后续各层配置在此基础上依次叠加。

因此，完整的优先级链从低到高是：

**pluginSettings -> userSettings -> projectSettings -> localSettings -> flagSettings -> policySettings**

### 5.1.2 合并规则

合并的核心在自定义合并策略函数 `settingsMergeCustomizer` 中：该函数对数组类型执行拼接并去重操作，其他类型则交给默认的深度合并逻辑处理。

这套规则的关键特性是：

- **数组类型**：拼接并去重（而非替换）。例如 `permissions.allow` 字段会累积所有源的规则。
- **对象类型**：深度合并。嵌套属性逐层覆盖。
- **标量类型**：后者直接覆盖前者。

这意味着，如果 `userSettings` 设置了 `model: "claude-sonnet-4"`，而 `policySettings` 设置了 `model: "claude-opus-4"`，最终生效的是 `"claude-opus-4"`。但如果两者都设置了 `permissions.allow: ["Bash(*)"]`，最终结果是合并后的去重数组。

### 5.1.3 配置文件的实际路径

每个配置源对应一个具体的文件路径，由配置路径映射函数确定：

| 配置源 | 文件路径 | 说明 |
|--------|----------|------|
| userSettings | `~/.claude/settings.json` | 全局用户设置 |
| projectSettings | `<project>/.claude/settings.json` | 项目共享设置（提交到 Git） |
| localSettings | `<project>/.claude/settings.local.json` | 项目本地设置（加入 .gitignore） |
| flagSettings | CLI `--settings` 参数指定路径 | 一次性覆盖 |
| policySettings | 平台相关的 managed-settings.json | 企业管理 |

其中 `policySettings` 的解析最为复杂。它遵循 **"first source wins"**（第一个非空源胜出）策略，优先级从高到低为：

1. **远程 API 设置**（通过 `getRemoteManagedSettingsSyncFromCache`）
2. **MDM 设置**（macOS plist / Windows HKLM）
3. **managed-settings.json 文件** + `managed-settings.d/*.json` drop-in 目录
4. **HKCU 注册表**（Windows 用户级，最低优先级）

### 5.1.4 策略层的特殊地位

与其他配置源使用深度合并不同，`policySettings` 采用了完全不同的解析逻辑。它不是从文件读取后合并，而是按优先级查找第一个非空源（远程 API 设置 > MDM 设置 > managed-settings.json 文件 > HKCU 注册表）。

这意味着企业管理员只需在一个位置配置策略，系统就会使用最高优先级的那个来源，而非合并所有来源。

## 5.2 安全边界设计

配置系统的核心安全挑战是：`projectSettings`（`.claude/settings.json`）会被提交到 Git 仓库，这意味着克隆恶意仓库的用户可能在不知情的情况下加载攻击者的配置。Claude Code 的防护策略是：**在安全敏感的检查中，系统性地排除 `projectSettings`**。

### 5.2.1 shouldAllowManagedHooksOnly

在钩子配置快照模块中，`shouldAllowManagedHooksOnly` 函数决定了是否只允许托管（managed）hooks 运行。它会检查策略设置中是否启用了 `allowManagedHooksOnly`，如果启用了则返回 true。

当此函数返回 `true` 时，hooks 的执行只使用来自 `policySettings` 的 hooks，所有来自 user/project/local 的 hooks 被跳过。这是一个企业安全特性：管理员可以确保只有经过审核的 hooks 在组织内运行。

### 5.2.2 pluginOnlyPolicy

插件策略模块实现了 `strictPluginOnlyCustomization` 策略，它定义了四种可锁定的"定制面"（customization surfaces）：skills、agents、hooks、mcp。

核心判断函数 `isRestrictedToPluginOnly` 检查策略配置：如果策略为 `true`，则锁定所有面；如果策略是数组，则只锁定指定的面。

被锁定的面中，只有以下来源被信任：

- **plugin** -- 通过 `strictKnownMarketplaces` 单独管控
- **policySettings** -- 由管理员设置，天然可信
- **built-in / bundled** -- 随 CLI 发布，非用户编写

而用户级别（`~/.claude/*`）和项目级别（`.claude/*`）的定制全部被阻断。

### 5.2.3 projectSettings 的系统性排除

在安全敏感函数中，`projectSettings` 被一致排除。以下函数展示了相同的模式：检查 `skipDangerousModePermissionPrompt` 时，只从 userSettings、localSettings、flagSettings 和 policySettings 中读取，**有意排除 projectSettings**。

同样的排除模式出现在 `hasAutoModeOptIn()`、`getUseAutoModeDuringPlan()` 和 `getAutoModeConfig()` 中。注释一致指向同一个原因：**projectSettings is intentionally excluded -- a malicious project could otherwise auto-bypass the dialog (RCE risk)**。

这种防御的假设是：用户自己的 settings（`userSettings`/`localSettings`）是可信的，因为它们在用户的文件系统上，由用户自己编辑；而 `projectSettings` 可能来自克隆的第三方仓库，存在供应链攻击风险。

## 5.3 功能开关系统

Claude Code 的功能开关系统分为两层：**编译时**的 `feature()` 函数和**运行时**的 GrowthBook 实验框架。

### 5.3.1 编译时死代码消除

`feature()` 函数通过 bundler 引入。当 `feature('FEATURE_NAME')` 返回 `false` 时，bundler 会将对应的代码分支完全移除。这是一种编译时的死代码消除（dead code elimination）。

在整个代码库中可以看到大量这种模式：根据特性标志决定是否编译某段功能代码，未启用的功能在构建产物中完全不存在。

主要的功能标志包括：

| 功能标志 | 作用域 | 说明 |
|----------|--------|------|
| `KAIROS` | 核心架构 | Assistant 模式（长驻会话） |
| `EXTRACT_MEMORIES` | 记忆系统 | 后台记忆提取 |
| `TRANSCRIPT_CLASSIFIER` | 权限系统 | 自动模式分类器 |
| `TEAMMEM` | 协作系统 | 团队记忆 |
| `CHICAGO_MCP` | 工具系统 | Computer Use MCP |
| `TEMPLATES` | 任务系统 | 模板与工作流 |
| `BUDDY` | UI 系统 | 伴侣精灵 |
| `DAEMON` | 架构 | 后台守护进程 |
| `BRIDGE_MODE` | 架构 | 桥接模式 |

### 5.3.2 GrowthBook 实验框架

对于需要在运行时动态控制的功能，Claude Code 使用 GrowthBook A/B 测试框架。主要的入口函数 `getFeatureValue_CACHED_MAY_BE_STALE` 接受特性名称和默认值，从缓存中同步读取实验配置。

函数名中的 "CACHED_MAY_BE_STALE" 直白地说明了其语义：值来自缓存，可能在跨进程的场景下已过时。这是为了在启动关键路径上避免异步等待——同步读取缓存比阻塞等待远程配置更可取。

在记忆系统中可以看到 GrowthBook 功能标志的使用——通过检查随机命名的特性标志来决定是否启用某项记忆功能。

这些使用随机动物名称命名的功能标志（如 `tengu_passport_quail`、`tengu_coral_fern`、`tengu_moth_copse`）是 GrowthBook 实验的标准做法——随机名称避免了语义偏见，并且不会与编译时的 `feature()` 函数冲突。

## 5.4 状态管理系统

### 5.4.1 Store：极简不可变状态容器

状态管理模块实现了一个只有 34 行的泛型 Store，其设计灵感来自 Zustand。Store 提供三个核心方法：`getState` 获取当前状态、`setState` 通过 updater 函数更新状态、`subscribe` 订阅状态变化。

三个关键设计决策：

1. **不可变更新**：`setState` 接受一个 updater 函数 `(prev: T) => T`，要求调用者返回全新的状态对象。通过 `Object.is` 比较新旧引用——只有引用发生变化时才通知监听者。
2. **泛型 `onChange` 回调**：在 Store 创建时传入，每次状态变更都会携带新旧状态被调用。这在 `AppStateProvider` 中被用于响应外部设置变更。
3. **Set-based 监听者管理**：使用 `Set` 而非数组，自动去重；`subscribe` 返回取消函数，遵循 React 的 cleanup 模式。

### 5.4.2 AppState：全局状态的类型定义

全局状态类型 `AppState` 被标记为深度不可变（DeepImmutable），包含了超过 50 个状态字段，涵盖了：

- **设置层**：`settings`（合并后的 SettingsJson）、`verbose`、`mainLoopModel`
- **UI 层**：`expandedView`、`footerSelection`、`statusLineText`
- **工具权限层**：`toolPermissionContext`（当前权限模式、允许的工具列表等）
- **MCP 层**：`mcp.clients`、`mcp.tools`、`mcp.commands`
- **插件层**：`plugins.enabled`、`plugins.disabled`、`plugins.errors`
- **桥接层**：`replBridgeEnabled`、`replBridgeConnected` 等十个桥接相关状态
- **Agent 层**：`agentDefinitions`、`agentNameRegistry`、`teamContext`
- **推测执行层**：`speculation`（空闲/活跃状态机）

默认状态由 `getDefaultAppState()` 构建，它在启动时获取合并后的设置，并初始化所有子系统为安全的默认值。

### 5.4.3 AppStateProvider：React Context 封装

`AppStateProvider` 将 Store 封装为 React Context。Store 只创建一次（通过 `useState` 的惰性初始化），Provider 本身不会因状态变化而重新渲染。消费者通过两个 hooks 访问状态：

- **`useAppState(selector)`**：使用 `useSyncExternalStore` 订阅状态切片。只有 selector 返回值发生 `Object.is` 变化时才触发重渲染。这是一种精细化的订阅机制，避免了"订阅整个状态树导致全组件树重渲染"的问题。
- **`useSetAppState()`**：只获取 `setState` 函数，不订阅任何状态。返回的引用永远不会变化，使用此 hook 的组件不会因状态变更而重渲染。

此外还有一个安全变体 `useAppStateMaybeOutsideOfProvider`，用于可能在 `AppStateProvider` 之外渲染的组件——它在没有 Provider 时返回 `undefined` 而非抛出异常。

---

## 实战练习

### 练习 1：配置合并预测

假设存在以下配置：

- `~/.claude/settings.json`：`{ "permissions": { "allow": ["Bash(ls)"] }, "model": "sonnet" }`
- `.claude/settings.json`：`{ "permissions": { "allow": ["Read(*)"] }, "hooks": { "Stop": [...] } }`
- `.claude/settings.local.json`：`{ "permissions": { "allow": ["Bash(git *)"] } }`

请预测合并后的 `permissions.allow` 和 `model` 值。

**参考答案**：`permissions.allow` 为 `["Bash(ls)", "Read(*)", "Bash(git *)"]`（数组拼接去重），`model` 为 `"sonnet"`（无更高优先级覆盖）。

### 练习 2：安全边界分析

如果你是企业管理员，希望确保：(1) 所有用户只能使用管理员批准的 hooks；(2) 用户不能自行安装 MCP 服务器。你应该在 `managed-settings.json` 中设置哪些字段？

**参考答案**：设置 `allowManagedHooksOnly: true` 和 `strictPluginOnlyCustomization: ["mcp"]`。

### 练习 3：状态订阅优化

一个组件需要显示当前模型名和 verbose 标志。以下两种写法哪种更优？

- **写法 A**：通过 `useAppState` 订阅整个状态对象
- **写法 B**：分别通过 `useAppState` 精确订阅 `mainLoopModel` 和 `verbose` 两个字段

**参考答案**：写法 B 更优。写法 A 订阅了整个状态树，任何状态变化都会导致重渲染。写法 B 精确订阅两个字段，只有这两个字段变化时才触发重渲染。

---

## 关键要点

1. **六层优先级链**：pluginSettings -> userSettings -> projectSettings -> localSettings -> flagSettings -> policySettings，后者覆盖前者。
2. **合并策略**：数组拼接去重，对象深度合并，标量直接覆盖。
3. **安全边界**：`projectSettings` 在所有安全敏感检查中被排除，防止恶意仓库的供应链攻击。
4. **双重功能开关**：`feature()` 提供编译时死代码消除，GrowthBook 提供运行时实验控制。
5. **不可变状态**：Store 使用 `Object.is` 引用比较，强制不可变更新模式，配合 React 的 `useSyncExternalStore` 实现精细化的订阅渲染。
