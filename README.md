<h1 align="center">驾驭智能体</h1>
<p align="center"><strong>Claude Code 设计哲学与 Agent Harness 实战</strong></p>
<p align="center">从零到精通，深入理解 AI 编程助手的核心架构与最佳实践</p>

---

> **⚠️ Disclaimer / 免责声明**
>
> 本书基于对 Claude Code 公开文档和产品行为的架构分析编写，**未引用、未使用任何未公开或未授权的源码**。Claude Code 为 Anthropic PBC 产品，本书不隶属于、未获授权于、也不代表 Anthropic。
>
> This book is based on analysis of Claude Code's public documentation and product behavior. **No unpublished or unauthorized source code is referenced or used.** Claude Code is a product of Anthropic PBC. This book is not affiliated with or endorsed by Anthropic.

---

## 背景

2026 年 3 月 31 日，安全研究员 [Chaofan Shou (@Fried_rice)](https://x.com/Fried_rice) 发现 Anthropic 发布在 npm registry 中的 `@anthropic-ai/claude-code` 包存在构建配置失误：包中包含的 source map（`.map`）文件引用了 Anthropic Cloudflare R2 存储桶中的完整源码压缩包，且该存储桶未设置访问控制。消息经公开披露后迅速传播（原始推文获得超 1700 万次浏览），引发了技术社区对 Agent 架构的广泛讨论。Anthropic 随后修补了该配置问题。

本书写作的初衷正是受到这场讨论的启发——当 Agent 架构成为开发者社区的热门话题，我们意识到需要一本系统性的书来讲解 Agent Harness 的设计原理。

---

## 这本书讲什么

Claude Code 是目前最完整的工业级 Agent Harness 实现之一。本书不做使用教程，不列 Prompt 技巧，而是带你拆解它的架构——对话循环如何驱动、工具权限为何采用四阶段管线、上下文压缩怎样在 token 预算内运转。

读懂了 Claude Code 的设计，你就拥有了一套可迁移到任何 Agent 框架的心智模型。

## 适合谁读

- 想构建自己的 Agent Harness 的**架构师**
- 不满足于调用 API、想理解底层机制的**高级工程师**
- 对 LLM Agent 工程实现感兴趣的**研究者**
- 希望最大化利用 Claude Code 的**实践者**

## 目录

### [前言](00-前言.md) — 为什么写这本书

### 第一部分：基础篇
| 章节 | 标题 | 核心内容 |
|:---:|------|---------|
| [第 1 章](第一部分-基础篇/01-智能体编程的新范式.md) | 智能体编程的新范式 | Agent 范式转移、Harness 定义、Claude Code 全景架构 |
| [第 2 章](第一部分-基础篇/02-对话循环-Agent的心跳.md) | 对话循环 — Agent 的心跳 | AsyncGenerator 主循环、流式响应、错误恢复、turn 状态机 |
| [第 3 章](第一部分-基础篇/03-工具系统-Agent的双手.md) | 工具系统 — Agent 的双手 | 23 种工具、输入 Schema、并发安全、中断行为、Tool 注册表 |
| [第 4 章](第一部分-基础篇/04-权限管线-Agent的护栏.md) | 权限管线 — Agent 的护栏 | 四阶段权限检查、权限模式、用户审批流、bypass 机制 |

### 第二部分：核心系统篇
| 章节 | 标题 | 核心内容 |
|:---:|------|---------|
| [第 5 章](第二部分-核心系统篇/05-设置与配置-Agent的基因.md) | 设置与配置 — Agent 的基因 | 分层配置、settings.json 体系、Zod v4 校验、远程托管设置 |
| [第 6 章](第二部分-核心系统篇/06-记忆系统-Agent的长期记忆.md) | 记忆系统 — Agent 的长期记忆 | 持久化记忆、MEMORY.md 索引、自动提取、跨会话保持 |
| [第 7 章](第二部分-核心系统篇/07-上下文管理-Agent的工作记忆.md) | 上下文管理 — Agent 的工作记忆 | Token 预算、上下文压缩、优先级排序、系统提示组装 |
| [第 8 章](第二部分-核心系统篇/08-钩子系统-Agent的生命周期扩展点.md) | 钩子系统 — Agent 的生命周期扩展点 | Hook 规范、tool permission hooks、pre/post 执行、阻断控制 |

### 第三部分：高级模式篇
| 章节 | 标题 | 核心内容 |
|:---:|------|---------|
| [第 9 章](第三部分-高级模式篇/09-子智能体与Fork模式.md) | 子智能体与 Fork 模式 | AgentTool、worktree 隔离、上下文继承、并行调度 |
| [第 10 章](第三部分-高级模式篇/10-协调器模式-多智能体编排.md) | 协调器模式 — 多智能体编排 | Coordinator 架构、Team 机制、消息传递、任务分配 |
| [第 11 章](第三部分-高级模式篇/11-技能系统与插件架构.md) | 技能系统与插件架构 | Skill 协议、Plugin 加载、MCP 工具桥接、扩展点设计 |
| [第 12 章](第三部分-高级模式篇/12-MCP集成与外部协议.md) | MCP 集成与外部协议 | Model Context Protocol、Server 管理、LSP 集成、协议桥接 |

### 第四部分：工程实践篇
| 章节 | 标题 | 核心内容 |
|:---:|------|---------|
| [第 13 章](第四部分-工程实践篇/13-流式架构与性能优化.md) | 流式架构与性能优化 | 并行预取、惰性加载、Token 估算、启动优化 |
| [第 14 章](第四部分-工程实践篇/14-Plan模式与结构化工作流.md) | Plan 模式与结构化工作流 | Plan/Act 分离、EnterPlanMode/ExitPlanMode、结构化审批 |
| [第 15 章](第四部分-工程实践篇/15-构建你自己的Agent-Harness.md) | 构建你自己的 Agent Harness | 从零实现 Mini Harness，融会贯通全书内容 |

### 附录
| 附录 | 标题 |
|:---:|------|
| [附录 A](附录/A-源码导航地图.md) | 源码导航地图 |
| [附录 B](附录/B-工具完整清单.md) | 工具完整清单 |
| [附录 C](附录/C-功能标志速查表.md) | 功能标志速查表 |
| [附录 D](附录/D-术语表.md) | 术语表 |

## 阅读路径

```
时间有限？ → 第 1-2 章（建立心智模型）→ 第 3-4 章（核心机制）

有经验的工程师？ → 直接从第二部分开始，遇到概念缺口再回溯

完整学习？ → 按顺序阅读，完成每章实战练习
```

## 约定

- 中文写作，技术术语保留英文原文
- 每章结构：学习目标 → 核心概念 → 架构图 → 实战练习 → 关键要点
- 代码示例为说明设计模式的示意代码，非产品源码

## License

本书内容采用 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 协议发布——可自由分享和改编，但须署名、非商业使用、并以相同协议共享。

Claude Code 为 Anthropic PBC 产品，本书的分析基于公开资料，未引用任何未授权源码。
