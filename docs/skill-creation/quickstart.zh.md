---
title: "快速上手"
description: "创建你的第一个 Agent Skill，并看到它在 VS Code 中生效。"
---

在本教程中，你将创建一个技能，赋予 agent 使用随机数生成器掷骰子的能力。

## 前置条件

- [VS Code](https://code.visualstudio.com/) 以及 [GitHub Copilot](https://marketplace.visualstudio.com/items?itemName=GitHub.copilot)

<Note>
本教程使用 VS Code，但 Agent Skills 是一种开放格式。同一个技能可以在任何兼容的 agent 中工作，包括 Claude Code 和 OpenAI Codex。
</Note>

## 创建技能

技能是一个包含 `SKILL.md` 文件的文件夹。VS Code 默认在 `.agents/skills/` 中查找技能。在你的项目中创建 `.agents/skills/roll-dice/SKILL.md`：

````markdown .agents/skills/roll-dice/SKILL.md
---
name: roll-dice
description: Roll dice using a random number generator. Use when asked to roll a die (d6, d20, etc.), roll dice, or generate a random dice roll.
---

To roll a die, use the following command that generates a random number from 1
to the given number of sides:

```bash
echo $((RANDOM % <sides> + 1))
```

```powershell
Get-Random -Minimum 1 -Maximum (<sides> + 1)
```

Replace `<sides>` with the number of sides on the die (e.g., 6 for a standard
die, 20 for a d20).
````

就是这样——一个文件，不到 20 行。以下是各个部分的作用：

- **`name`** —— 技能的简短标识符。必须与文件夹名称一致。
- **`description`** —— 告诉 agent 何时使用这个技能。agent 正是据此决定是否激活它。
- **正文（The body）** —— 技能激活时 agent 遵循的指令。在这里，agent 被指示使用终端命令生成一个随机数，并代入用户请求中的面数。

## 试一试

1. 在 VS Code 中打开你的项目。
2. 打开 Copilot Chat 面板。
3. 在聊天面板底部的模式下拉菜单中选择 **Agent** 模式。
4. 输入 `/skills`，确认 `roll-dice` 出现在列表中。如果没有出现，请检查文件是否位于相对于项目根目录的 `.agents/skills/roll-dice/SKILL.md`。
5. 提问：**"Roll a d20"**

agent 应当激活 `roll-dice` 技能。它可能会请求运行终端命令的权限——允许即可。它会运行该命令并返回一个 1 到 20 之间的随机数。

<Note>
工具使用可靠性因模型而异——有些模型会始终如一地遵循技能指令并运行命令，而另一些模型可能会尝试自行作答。如果 agent 没有运行终端命令就作出了回应，请尝试从模型下拉菜单中选择其他模型。
</Note>

## 工作原理

以下是幕后发生的事情：

1. **发现（Discovery）** —— 聊天会话启动时，agent 扫描默认技能目录并找到了你的技能。它只读取了 `name` 和 `description`，仅够判断该技能何时可能相关。

2. **激活（Activation）** —— 当你询问掷骰子时，agent 将你的问题与技能的 description 匹配，并将完整的 `SKILL.md` 正文加载到上下文中。

3. **执行（Execution）** —— agent 遵循正文中的指令，根据你请求中的面数调整终端命令。

这个过程使用**渐进式披露**，让 agent 无需预先加载所有指令即可访问大量技能。

## 后续步骤

你已经创建了一个可用的 Agent Skill。接下来可以：

- **[最佳实践](/skill-creation/best-practices)** —— 如何编写范围界定良好且有效的技能。
- **[优化技能描述](/skill-creation/optimizing-descriptions)** —— 测试并改进技能的 description，使其在正确的提示词上激活。
- **[规范](/specification)** —— `SKILL.md` 文件的完整格式参考。
- **[示例技能](https://github.com/anthropics/skills)** —— 在 GitHub 上浏览真实世界的技能。
