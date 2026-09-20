---
title: "如何为你的 agent 添加技能支持"
sidebarTitle: "添加技能支持"
description: "为 AI agent 或开发工具添加 Agent Skills 支持的指南。"
---

本指南将逐步介绍如何为 AI agent 或开发工具添加 Agent Skills 支持。它涵盖完整生命周期：发现技能、将技能告知模型、将其内容加载到上下文中，以及让这些内容随时间保持有效。

无论你的 agent 架构如何，核心集成方式都是一样的。实现细节因两个因素而异：

- **技能存放在哪里？** 本地运行的 agent 可以扫描用户文件系统以查找技能目录。云端托管或沙箱化的 agent 则需要替代的发现机制——API、远程注册表或打包的资产。
- **模型如何访问技能内容？** 如果模型具备文件读取能力，它可以直接读取 `SKILL.md` 文件。否则，你需要提供专用工具，或以编程方式将技能内容注入提示词。

本指南会指出这些差异在何处产生影响。你不需要支持每一种场景——遵循适合你的 agent 的路径即可。

**前提条件**：熟悉 [Agent Skills 规范](/specification)，其中定义了 `SKILL.md` 文件格式、frontmatter 字段以及目录约定。

## 核心原则：渐进式披露

每个兼容技能的 agent 都遵循相同的三层加载策略：

| 层级 | 加载内容 | 时机 | Token 成本 |
|------|--------------|------|------------|
| 1. 目录 | 名称 + 描述 | 会话开始 | 每个技能约 50-100 个 token |
| 2. 指令 | 完整的 `SKILL.md` 正文 | 技能被激活时 | &lt;5000 个 token（推荐） |
| 3. 资源 | 脚本、引用、资产 | 指令引用它们时 | 因情况而异 |

模型从一开始就能看到目录，因此它知道有哪些可用技能。当它判定某个技能相关时，会加载完整指令。如果这些指令引用了配套文件，模型会按需逐个加载。

这样既保持了基础上下文精简，又让模型能够按需获取专业知识。安装了 20 个技能的 agent 不会预先付出 20 套完整指令的 token 成本——只有在某次对话中实际用到的才会付出。

## 第 1 步：发现技能

在会话启动时，找出所有可用技能并加载其元数据。

### 扫描位置

扫描哪些目录取决于你的 agent 环境。大多数本地运行的 agent 至少扫描两个范围：

- **项目级**（相对于工作目录）：特定于某个项目或仓库的技能。
- **用户级**（相对于主目录）：对某个用户在所有项目中都可用的技能。

其他范围也是可能的——例如，由管理员部署的组织级技能，或与 agent 本身打包在一起的技能。合适的范围集合取决于你的 agent 部署模型。

在每个范围内，考虑同时扫描**客户端专属目录**和 **`.agents/skills/` 约定**：

| 范围 | 路径 | 用途 |
|-------|------|---------|
| 项目 | `<project>/.<your-client>/skills/` | 你的客户端原生位置 |
| 项目 | `<project>/.agents/skills/` | 跨客户端互操作 |
| 用户 | `~/.<your-client>/skills/` | 你的客户端原生位置 |
| 用户 | `~/.agents/skills/` | 跨客户端互操作 |

`.agents/skills/` 路径已成为被广泛采用的跨客户端技能共享约定。虽然 Agent Skills 规范并未强制规定技能目录存放的位置（它只定义了目录内部应包含什么），但扫描 `.agents/skills/` 意味着其他合规客户端安装的技能会自动对你的客户端可见，反之亦然。

<Note>
一些实现出于务实的兼容性考虑，也会扫描 `.claude/skills/`（项目级和用户级皆是），因为许多现有技能都安装在那里。其他额外位置包括向上追溯至 git 根目录的祖先目录（对 monorepo 很有用）、[XDG](https://specifications.freedesktop.org/basedir-spec/latest/) 配置目录，以及用户配置的路径。
</Note>

### 扫描什么

在每个技能目录中，查找**包含一个名称完全为 `SKILL.md` 的文件的子目录**：

```
~/.agents/skills/
├── pdf-processing/
│   ├── SKILL.md          ← discovered
│   └── scripts/
│       └── extract.py
├── data-analysis/
│   └── SKILL.md          ← discovered
└── README.md             ← ignored (not a skill directory)
```

实用的扫描规则：

- 跳过不会包含技能的目录，例如 `.git/` 和 `node_modules/`
- 可选地遵循 `.gitignore`，以避免扫描构建产物
- 设置合理的边界（例如，最大深度 4-6 层，最多 2000 个目录），以防止在大型目录树中无节制地扫描

### 处理名称冲突

当两个技能共享同一个 `name` 时，应用确定性的优先级规则。

现有实现中普遍遵循的约定是：**项目级技能覆盖用户级技能。**

在同一范围内（例如，在 `<project>/.agents/skills/` 和 `<project>/.<your-client>/skills/` 下都找到两个名为 `code-review` 的技能），采用先找到优先或后找到优先都可以——选择一种并保持一致。发生冲突时记录一条警告，让用户知道某个技能被遮蔽了。

### 信任考量

项目级技能来自正在处理的仓库，该仓库可能是不可信的（例如，刚克隆的开源项目）。考虑对项目级技能的加载加以信任检查——只有当用户将项目文件夹标记为可信时才加载它们。这可以防止不可信的仓库悄悄将指令注入 agent 的上下文。

### 云端托管与沙箱化的 agent

如果你的 agent 运行在容器或远程服务器上，它将无法访问用户的本地文件系统。发现机制需要根据技能范围以不同方式工作：

- **项目级技能**通常是最简单的情况。如果 agent 操作的是克隆下来的仓库（即使在沙箱内），项目级技能会随代码一起存在，可以从仓库的目录树中扫描到。
- **用户级和组织级技能**在沙箱中并不存在。你需要从外部来源提供它们——例如克隆一个配置仓库、通过 agent 的设置接受技能 URL 或包，或让用户通过 Web UI 上传技能目录。
- **内置技能**可以作为静态资产打包进 agent 的部署产物中，使其在每次会话中都可用，无需外部获取。

一旦技能对 agent 可用，生命周期的其余部分——解析、披露、激活——的工作方式都是相同的。

## 第 2 步：解析 `SKILL.md` 文件

对每个发现的 `SKILL.md`，提取元数据和正文内容。

### Frontmatter 提取

`SKILL.md` 文件包含两部分：位于 `---` 分隔符之间的 YAML frontmatter，以及闭合分隔符之后的 markdown 正文。解析方式：

1. 找到文件开头处的起始 `---` 及其之后的闭合 `---`。
2. 解析两者之间的 YAML 块。提取 `name` 和 `description`（必填），以及任何可选字段。
3. 闭合 `---` 之后的所有内容，去除首尾空白后，即为技能的正文内容。

完整的 frontmatter 字段集合及其约束请参见[规范](/specification)。

### 处理格式错误的 YAML

为其他客户端编写的技能文件可能包含技术上有误、但其解析器恰好能接受的 YAML。最常见的问题是未加引号的值中包含冒号：

```yaml
# Technically invalid YAML — the colon breaks parsing
description: Use this skill when: the user asks about PDFs
```

考虑一种回退方案：将此类值用引号包裹，或将其转换为 YAML 块标量后再重试。这能以极小的成本提升跨客户端兼容性。

### 宽松校验

对问题发出警告，但在可能的情况下仍然加载技能：

- 名称与父目录名不匹配 → 警告，仍然加载
- 名称超过 64 个字符 → 警告，仍然加载
- 描述缺失或为空 → 跳过该技能（描述对披露至关重要），记录错误
- YAML 完全无法解析 → 跳过该技能，记录错误

记录诊断信息，以便将其呈现给用户（在调试命令、日志文件或 UI 中），但不要因为表面问题而阻止技能加载。

<Note>
[规范](/specification)对 `name` 字段定义了严格约束（与父目录匹配、字符集、最大长度）。上面的宽松做法有意放宽了这些约束，以提升与为其他客户端编写的技能的兼容性。
</Note>

### 存储什么

至少，每条技能记录需要三个字段：

| 字段 | 描述 |
|-------|-------------|
| `name` | 来自 frontmatter |
| `description` | 来自 frontmatter |
| `location` | `SKILL.md` 文件的绝对路径 |

将它们存储在以 `name` 为键的内存映射中，以便在激活时快速查找。

你也可以在发现时存储**正文**（frontmatter 之后的 markdown 内容），或在激活时从 `location` 读取它。存储它可以让激活更快；在激活时读取它在总体上占用更少内存，并能获取两次激活之间对技能文件的修改。

技能的**基目录**（`location` 的父目录）在之后解析相对路径和枚举打包资源时需要用到——在需要时从 `location` 推导出来。

## 第 3 步：向模型披露可用技能

告诉模型存在哪些技能，而不加载它们的完整内容。这是[渐进式披露的第 1 层](#the-core-principle-progressive-disclosure)。

### 构建技能目录

对每个发现的技能，将 `name`、`description` 以及可选的 `location`（`SKILL.md` 文件的路径）以适合你技术栈的结构化格式包含进来——XML、JSON 或项目符号列表都可以：

```xml
<available_skills>
  <skill>
    <name>pdf-processing</name>
    <description>Extract PDF text, fill forms, merge files. Use when handling PDFs.</description>
    <location>/home/user/.agents/skills/pdf-processing/SKILL.md</location>
  </skill>
  <skill>
    <name>data-analysis</name>
    <description>Analyze datasets, generate charts, and create summary reports.</description>
    <location>/home/user/project/.agents/skills/data-analysis/SKILL.md</location>
  </skill>
</available_skills>
```

`location` 字段有两个用途：它支持文件读取式激活（见[第 4 步](#step-4-activate-skills)），并为模型提供用于解析技能正文中相对引用（如 `scripts/evaluate.py`）的基础路径。如果你的专用激活工具在其结果中提供了技能目录路径（见第 4 步中的[结构化包装](#structured-wrapping)），则可以省略目录中的 `location`。否则，请包含它。

每个技能会给目录增加大约 50-100 个 token。即使安装了数十个技能，目录依然保持紧凑。

### 目录放在哪里

两种方式很常见：

**系统提示词章节**：将目录作为带标签的章节添加到系统提示词中，并在其前面附上关于如何使用技能的简要说明。这是最简单的方式，适用于任何能够访问文件读取工具的模型。

**工具描述**：将目录嵌入到专用技能激活工具的描述中（见[第 4 步](#step-4-activate-skills)）。这可以让系统提示词保持整洁，并自然地将发现与激活耦合在一起。

两者都可行。放在系统提示词中更简单、兼容性更广；当你有专用激活工具时，嵌入工具描述会更整洁。

### 行为指令

在目录旁附上一段简短的指令块，告诉模型如何以及何时使用技能。措辞取决于你支持哪种激活机制（见[第 4 步](#step-4-activate-skills)）：

**如果模型通过读取文件来激活技能：**

```
The following skills provide specialized instructions for specific tasks.
When a task matches a skill's description, use your file-read tool to load
the SKILL.md at the listed location before proceeding.
When a skill references relative paths, resolve them against the skill's
directory (the parent of SKILL.md) and use absolute paths in tool calls.
```

**如果模型通过专用工具激活技能：**

```
The following skills provide specialized instructions for specific tasks.
When a task matches a skill's description, call the activate_skill tool
with the skill's name to load its full instructions.
```

保持这些指令简洁。目标是告诉模型技能存在以及如何加载它们——技能内容本身会在加载后提供详细指令。

### 过滤

有些技能应被排除在目录之外。常见原因：

- 用户在设置中禁用了该技能
- 权限系统拒绝访问该技能
- 该技能选择不参与模型驱动的激活（例如，通过 `disable-model-invocation` 标志）

**将过滤掉的技能完全隐藏**，而不是列出它们并在激活时阻止。这可以防止模型浪费轮次尝试加载它无法使用的技能。

### 当没有可用技能时

如果未发现任何技能，则完全省略目录和行为指令。不要显示空的 `<available_skills/>` 块，也不要注册没有任何有效选项的技能工具——这会迷惑模型。

## 第 4 步：激活技能

当模型或用户选择某个技能时，将完整指令送入对话上下文。这是[渐进式披露的第 2 层](#the-core-principle-progressive-disclosure)。

### 模型驱动的激活

大多数实现依赖模型自身的判断作为激活机制，而不是在 harness 侧实现触发器匹配或关键词检测。模型读取目录（来自[第 3 步](#step-3-disclose-available-skills-to-the-model)），判定某个技能与当前任务相关，然后加载它。

两种实现模式：

**文件读取式激活**：模型使用目录中的 `SKILL.md` 路径调用其标准文件读取工具。无需特殊基础设施——agent 现有的文件读取能力就足够了。模型以工具结果的形式收到文件内容。当模型有文件访问权限时，这是最简单的方式。

**专用工具激活**：注册一个工具（例如 `activate_skill`），它接收技能名称并返回内容。当模型无法直接读取文件时必须使用这种方式，而在模型能够读取文件时它也是可选（但有用）的。相比直接读取文件的优势：

- 控制返回的内容——例如剥离或保留 YAML frontmatter（见下文[模型收到什么](#what-the-model-receives)）
- 将内容包装在结构化标签中，以便在上下文管理期间识别
- 在指令旁列出打包资源（例如 `references/*`）
- 强制执行权限或提示用户同意
- 跟踪激活以用于分析

<Tip>
如果你使用专用激活工具，请将 `name` 参数限制为有效技能名称的集合（例如，在工具 schema 中作为 enum）。这可以防止模型虚构不存在的技能名称。如果没有可用技能，则完全不要注册该工具。
</Tip>

### 用户显式激活

用户也应能够直接激活技能，而无需等待模型做决定。最常见的模式是 harness 拦截的**斜杠命令或提及语法**（`/skill-name` 或 `$skill-name`）。具体语法由你决定——关键思想是 harness 负责查找和注入，因此模型无需自行执行激活操作即可收到技能内容。

自动补全组件（在用户输入时列出可用技能）也能让这一功能更易被发现。

### 模型收到什么

当技能被激活时，模型会收到该技能的指令。对于内容具体呈现为什么样子，有两种选择：

**完整文件**：模型看到整个 `SKILL.md`，包括 YAML frontmatter。这是文件读取式激活的自然结果，此时模型读取的是原始文件。对于专用工具来说，这也是一个有效的选择。Frontmatter 可能包含在激活时有用的字段——例如，[`compatibility`](/specification#compatibility-field) 会说明环境要求，可用于指导模型如何执行技能的指令。

**仅正文（剥离 frontmatter）**：harness 解析并移除 YAML frontmatter，只返回 markdown 指令。在拥有专用激活工具的现有实现中，大多数采用这种方式——在发现阶段提取 `name` 和 `description` 之后剥离 frontmatter。

两种方式在实践中都可行。

### 结构化包装

如果你使用专用激活工具，考虑将技能内容包装在用于识别的标签中。例如：

```xml
<skill_content name="pdf-processing">
# PDF Processing

## When to use this skill
Use this skill when the user needs to work with PDF files...

[rest of SKILL.md body]

Skill directory: /home/user/.agents/skills/pdf-processing
Relative paths in this skill are relative to the skill directory.

<skill_resources>
  <file>scripts/extract.py</file>
  <file>scripts/merge.py</file>
  <file>references/pdf-spec-summary.md</file>
</skill_resources>
</skill_content>
```

这有实际好处：

- 模型可以清楚地区分技能指令与其他对话内容
- harness 可以在上下文压缩期间识别技能内容（[第 5 步](#step-5-manage-skill-context-over-time)）
- 打包资源会呈现给模型，而不会被急切加载

### 列出打包资源

当专用激活工具返回技能内容时，它还可以枚举技能目录中的配套文件（脚本、引用、资产）——但**不应急切读取它们**。当技能的指令引用这些文件时，模型会使用其文件读取工具按需加载特定文件。

对于大型技能目录，考虑对列表设置上限，并说明它可能不完整。

### 权限允许列表

如果你的 agent 有控制文件访问的权限系统，请**将技能目录加入允许列表**，这样模型就能读取打包资源，而不会触发用户确认提示。否则，每一次对打包脚本或引用文件的引用都会弹出权限对话框，破坏那些包含 `SKILL.md` 之外资源的技能的流程。

## 第 5 步：随时间管理技能上下文

一旦技能指令进入对话上下文，就要让它们在会话期间保持有效。

### 保护技能内容不被上下文压缩

如果你的 agent 在上下文窗口填满时截断或摘要较早的消息，请**将技能内容排除在裁剪之外**。技能指令是持久的行为指导——在对话中途丢失它们会悄悄降低 agent 的性能，且没有任何可见错误。模型会继续运行，但失去了技能所提供的专门指令。

常见做法：

- 将技能工具输出标记为受保护，使裁剪算法跳过它们
- 使用第 4 步中的[结构化标签](#structured-wrapping)来识别技能内容，并在压缩期间保留它

### 去重激活

考虑跟踪当前会话中已激活了哪些技能。如果模型（或用户）尝试加载一个已在上下文中的技能，你可以跳过重复注入，以避免相同指令在对话中多次出现。

### 子 agent 委派（可选）

这是一种仅被部分客户端支持的高级模式。技能不在主对话中注入指令，而是在**单独的子 agent 会话**中运行。子 agent 接收技能指令，执行任务，并将工作摘要返回给主对话。

当技能的工作流复杂到足以受益于一个专门的、专注的会话时，这种模式很有用。
