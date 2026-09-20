---
title: "Agent Skills 概览"
sidebarTitle: "概览"
description: "一种为 AI agent 赋予新能力与专业知识的标准化方式。"
---

import { clients } from '/snippets/clients.jsx';
import { LogoCarousel } from '/snippets/LogoCarousel.jsx';

## 什么是 Agent Skills？

Agent Skills 是一种轻量、开放的格式，用于以专业知识和工作流扩展 AI agent 的能力。

从本质上讲，一个技能就是一个包含 `SKILL.md` 文件的文件夹。该文件包含元数据（至少包含 `name` 和 `description`）以及告诉 agent 如何执行特定任务的指令。技能还可以打包脚本、参考资料、模板以及其他资源。

```
my-skill/
├── SKILL.md          # Required: metadata + instructions
├── scripts/          # Optional: executable code
├── references/       # Optional: documentation
├── assets/           # Optional: templates, resources
└── ...               # Any additional files or directories
```

## 为什么选择 Agent Skills？

Agent 的能力日益增强，但往往缺乏可靠完成实际工作所需的上下文。技能通过将流程性知识以及公司、团队和用户特定的上下文打包成可移植、受版本控制的文件夹（agent 按需加载）来解决这一问题。这为 agent 带来了：

- **领域专业知识**：将专业知识——从法律审查流程到数据分析流水线，再到演示文稿排版——捕获为可复用的指令与资源。
- **可重复的工作流**：将多步骤任务转化为一致、可审计的流程。
- **跨产品复用**：构建一次技能，即可在任何兼容技能的 agent 中使用。

## Agent Skills 如何工作？

Agent 通过**渐进式披露**分三个阶段加载技能：

1. **发现**：启动时，agent 只加载每个可用技能的名称和描述，仅足以判断它何时可能相关。

2. **激活**：当任务与某个技能的描述匹配时，agent 会将完整的 `SKILL.md` 指令读入上下文。

3. **执行**：agent 按照指令执行，可选地运行打包的代码或按需加载被引用的文件。

完整指令仅在任务需要时才会加载，因此 agent 可以随身保留大量技能，而只占用很小的上下文空间。

## 在哪里可以使用 Agent Skills？

Agent Skills 受到大量 AI 工具和 agentic 客户端的支持——请查看[客户端展示](/clients)来探索其中的一部分！

<LogoCarousel clients={clients} />

## 开放开发

Agent Skills 格式最初由 [Anthropic](https://www.anthropic.com/) 开发，作为开放标准发布，并已被越来越多的 agent 产品采用。该标准开放给更广泛生态系统的贡献。

欢迎加入 [GitHub](https://github.com/agentskills/agentskills) 或 [Discord](https://discord.gg/MKPE9g8aUy) 上的讨论！

## Agent Skills 快速上手

<CardGroup cols={2}>
  <Card
    title="快速开始"
    icon="rocket"
    href="/skill-creation/quickstart"
  >
    创建你的第一个 Agent Skill，并亲眼看看它的效果。
  </Card>
  <Card
    title="规范"
    icon="file-code"
    href="/specification"
  >
    Agent Skills 的完整格式规范。
  </Card>
</CardGroup>
