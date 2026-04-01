<div align="center">

# 解码 Agent Harness

**Claude Code 架构深度剖析**

<br/>

当所有人都在教你怎么 **用** AI Agent——

**这本书带你拆开它。**

<br/>

[![在线阅读](https://img.shields.io/badge/📖-在线阅读-9f7aea?style=for-the-badge)](https://lintsinghua.github.io/)
[![GitHub Stars](https://img.shields.io/github/stars/lintsinghua/claude-code-book?style=social)](https://github.com/lintsinghua/claude-code-book/stargazers)
[![License](https://img.shields.io/badge/license-CC%20BY--NC--SA%204.0-lightgrey)](LICENSE)

<img width="2880" height="1558" alt="image" src="https://github.com/user-attachments/assets/39efa7d4-4521-444e-a222-fd0acb756e51" />

</div>

---

对话循环如何驱动？工具权限为何是四阶段管线？上下文压缩怎样在 token 预算内运转？子智能体如何通过 Fork 继承父级上下文？

读懂 Claude Code 的设计决策，你就拥有了一套**可迁移到任何 Agent 框架**的心智模型。

> **声明：** 本书基于对 Claude Code 公开文档和产品行为的架构分析编写，未引用、未使用任何未公开或未授权的源码。Claude Code 为 Anthropic PBC 产品，本书不隶属于、未获授权于、也不代表 Anthropic。

---

## 这本书有什么不同

**不做使用教程，不列 Prompt 技巧。**

市面上充斥着"如何写好 Prompt"和"如何调用 Agent API"的指南。但如果你想知道一个生产级 Agent 系统的**骨架**是怎么搭的——为什么用异步生成器而不是回调？为什么权限检查要分四个阶段？上下文窗口用完了怎么办？——几乎没有资料可查。

这本书填补了这个空白。它以 Claude Code 为案例，拆解一个 Agent Harness 的**每一个**核心子系统，讲清楚每个工程决策背后的"为什么"。

---

## 目录

### Part 1. 基础篇 — 建立心智模型

| | 章节 | 核心内容 |
|:-:|------|---------|
| 01 | [智能体编程的新范式](第一部分-基础篇/01-智能体编程的新范式.md) | 从聊天到工具调用，范式如何转移 |
| 02 | [对话循环 — Agent 的心跳](第一部分-基础篇/02-对话循环-Agent的心跳.md) | 异步生成器驱动的永动主循环 |
| 03 | [工具系统 — Agent 的双手](第一部分-基础篇/03-工具系统-Agent的双手.md) | 45+ 工具的注册、过滤与并发调度 |
| 04 | [权限管线 — Agent 的护栏](第一部分-基础篇/04-权限管线-Agent的护栏.md) | 四阶段安全管线与权限模式谱系 |

### Part 2. 核心系统篇 — 深入子系统

| | 章节 | 核心内容 |
|:-:|------|---------|
| 05 | [设置与配置 — Agent 的基因](第二部分-核心系统篇/05-设置与配置-Agent的基因.md) | 六层配置优先级与供应链攻击防御 |
| 06 | [记忆系统 — Agent 的长期记忆](第二部分-核心系统篇/06-记忆系统-Agent的长期记忆.md) | 持久化、索引、自动提取与跨会话保持 |
| 07 | [上下文管理 — Agent 的工作记忆](第二部分-核心系统篇/07-上下文管理-Agent的工作记忆.md) | 四级渐进压缩与 token 预算管理 |
| 08 | [钩子系统 — Agent 的生命周期扩展点](第二部分-核心系统篇/08-钩子系统-Agent的生命周期扩展点.md) | 26 个生命周期事件与安全边界 |

### Part 3. 高级模式篇 — Agent 的组合与扩展

| | 章节 | 核心内容 |
|:-:|------|---------|
| 09 | [子智能体与 Fork 模式](第三部分-高级模式篇/09-子智能体与Fork模式.md) | 字节级上下文继承与并行子任务 |
| 10 | [协调器模式 — 多智能体编排](第三部分-高级模式篇/10-协调器模式-多智能体编排.md) | Coordinator-Worker 架构与 Team 机制 |
| 11 | [技能系统与插件架构](第三部分-高级模式篇/11-技能系统与插件架构.md) | 零配置可用、可配置强大的技能协议 |
| 12 | [MCP 集成与外部协议](第三部分-高级模式篇/12-MCP集成与外部协议.md) | Model Context Protocol 与协议桥接 |

### Part 4. 工程实践篇 — 从原理到构建

| | 章节 | 核心内容 |
|:-:|------|---------|
| 13 | [流式架构与性能优化](第四部分-工程实践篇/13-流式架构与性能优化.md) | 并行预取、惰性加载、缓存共享 |
| 14 | [Plan 模式与结构化工作流](第四部分-工程实践篇/14-Plan模式与结构化工作流.md) | 计划与执行分离、定时触发 |
| 15 | [构建你自己的 Agent Harness](第四部分-工程实践篇/15-构建你自己的Agent-Harness.md) | 六步从零实现，融会贯通全书 |

### Appendix

| | 内容 |
|:-:|------|
| [A](附录/A-源码导航地图.md) | 架构导航地图 — 模块依赖与数据流 |
| [B](附录/B-工具完整清单.md) | 工具完整清单 — 50+ 工具速查 |
| [C](附录/C-功能标志速查表.md) | 功能标志速查表 — 89 个 Feature Flag |
| [D](附录/D-术语表.md) | 术语表 — 100 条中英对照 |

---

## 怎么读

**时间紧张？** Ch1 → Ch2 → Ch4 → Ch15，拿到心智模型和动手能力就够用。

**有经验的架构师？** 直接读 Part 2（核心系统）和 Part 3（高级模式），遇到概念缺口回溯 Part 1。

**系统学习？** 从头到尾读一遍，完成每章实战练习，最后在 Ch15 动手构建自己的 Harness。预计 2–3 周。

---

## 适合谁

- **架构师** — 你想构建自己的 Agent 框架，需要一张完整的设计空间地图
- **高级工程师** — 你不满足于调 API，想理解工具调用、流式处理、权限管控的底层机制
- **研究者** — 你想从实现角度理解 Agent 系统的运作方式
- **Claude Code 用户** — 你想理解设计意图，用得更准、调得更深

---

## 背景

2026 年 3 月 31 日，安全研究员 [Chaofan Shou (@Fried_rice)](https://x.com/Fried_rice) 发现 npm registry 中的 `@anthropic-ai/claude-code` 包存在构建配置失误，source map 文件引用了未设访问控制的 Cloudflare R2 存储桶。披露推文获得超 1700 万次浏览，引发了技术社区对 Agent 架构的空前讨论。

这本书的诞生正是受到这场讨论的启发。

---

## 贡献

欢迎 Issue 和 PR — 修正错误、补充案例、改进结构。

## 致谢

[Linux.Do](https://linux.do/) 社区

---

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=lintsinghua/claude-code-book&type=Date)](https://star-history.com/#lintsinghua/claude-code-book&Date)

---

<p align="center">
  <a href="https://creativecommons.org/licenses/by-nc-sa/4.0/">
    <img src="https://img.shields.io/badge/license-CC%20BY--NC--SA%204.0-lightgrey" alt="CC BY-NC-SA 4.0" />
  </a>
  <br/><br/>
  可自由分享和改编，但须署名、非商业使用、并以相同协议共享。
</p>
