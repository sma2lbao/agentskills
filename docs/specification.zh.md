---
title: "规范"
description: "Agent Skills 的完整格式规范。"
---

## 目录结构

技能是一个目录，其中至少包含一个 `SKILL.md` 文件：

```
skill-name/
├── SKILL.md          # Required: metadata + instructions
├── scripts/          # Optional: executable code
├── references/       # Optional: documentation
├── assets/           # Optional: templates, resources
└── ...               # Any additional files or directories
```

## `SKILL.md` 格式

`SKILL.md` 文件必须包含 YAML frontmatter，其后是 Markdown 内容。

### Frontmatter

| 字段 | 是否必需 | 约束 |
|-------|----------|-------------|
| `name` | 是 | 最多 64 个字符。仅限小写字母、数字和连字符。不得以连字符开头或结尾。 |
| `description` | 是 | 最多 1024 个字符。不能为空。描述该技能做什么以及何时使用。 |
| `license` | 否 | 许可证名称，或对随技能一同提供的许可证文件的引用。 |
| `compatibility` | 否 | 最多 500 个字符。说明环境要求（目标产品、系统软件包、网络访问等）。 |
| `metadata` | 否 | 用于附加元数据的任意键值映射（从字符串键到字符串值的映射）。 |
| `allowed-tools` | 否 | 以空格分隔的字符串，列出该技能可以使用的、已预先批准的工具。（实验性） |

<Card>
**最小示例：**

```markdown SKILL.md
---
name: skill-name
description: A description of what this skill does and when to use it.
---
```

**包含可选字段的示例：**

```markdown SKILL.md
---
name: pdf-processing
description: Extract PDF text, fill forms, merge files. Use when handling PDFs.
license: Apache-2.0
metadata:
  author: example-org
  version: "1.0"
---
```
</Card>

#### `name` 字段

必需的 `name` 字段：
- 必须为 1-64 个字符
- 只能包含 Unicode 小写字母数字字符（`a-z`、`0-9`）和连字符（`-`）
- 不得以连字符（`-`）开头或结尾
- 不得包含连续的连字符（`--`）
- 必须与父目录名称一致

<Card>
**有效示例：**
```yaml
name: pdf-processing
```
```yaml
name: data-analysis
```
```yaml
name: code-review
```

**无效示例：**
```yaml
name: PDF-Processing  # uppercase not allowed
```
```yaml
name: -pdf  # cannot start with hyphen
```
```yaml
name: pdf--processing  # consecutive hyphens not allowed
```
</Card>

#### `description` 字段

必需的 `description` 字段：
- 必须为 1-1024 个字符
- 应同时描述该技能做什么以及何时使用
- 应包含有助于智能体识别相关任务的具体关键词

<Card>
**好的示例：**
```yaml
description: Extracts text and tables from PDF files, fills PDF forms, and merges multiple PDFs. Use when working with PDF documents or when the user mentions PDFs, forms, or document extraction.
```

**差的示例：**
```yaml
description: Helps with PDFs.
```
</Card>

#### `license` 字段

可选的 `license` 字段：
- 指定适用于该技能的许可证
- 建议保持简短（可以是许可证名称，或随技能一同提供的许可证文件的名称）

<Card>
**示例：**
```yaml
license: Proprietary. LICENSE.txt has complete terms
```
</Card>

#### `compatibility` 字段

可选的 `compatibility` 字段：
- 如果提供，必须为 1-500 个字符
- 仅当你的技能有特定环境要求时才应包含
- 可以说明目标产品、所需的系统软件包、网络访问需求等。

<Card>
**示例：**
```yaml
compatibility: Designed for Claude Code (or similar products)
```
```yaml
compatibility: Requires git, docker, jq, and access to the internet
```
```yaml
compatibility: Requires Python 3.14+ and uv
```
</Card>

<Note>
大多数技能不需要 `compatibility` 字段。
</Note>

#### `metadata` 字段

可选的 `metadata` 字段：
- 从字符串键到字符串值的映射
- 客户端可用它来存储 Agent Skills 规范未定义的附加属性
- 建议让键名具有足够唯一性，以避免意外冲突

<Card>
**示例：**
```yaml
metadata:
  author: example-org
  version: "1.0"
```
</Card>

#### `allowed-tools` 字段

可选的 `allowed-tools` 字段：
- 以空格分隔的字符串，列出已预先批准运行的工具
- 实验性。不同智能体实现对该字段的支持可能不同

<Card>
**示例：**
```yaml
allowed-tools: Bash(git:*) Bash(jq:*) Read
```
</Card>

### 正文内容

frontmatter 之后的 Markdown 正文包含技能指令。格式没有任何限制。只要有助于智能体有效完成任务，写什么都可以。

建议包含的章节：
- 分步说明
- 输入与输出示例
- 常见边界情况

注意，一旦智能体决定激活某个技能，就会加载整个文件。如果 `SKILL.md` 内容较长，考虑将其拆分到被引用的文件中。

## 可选目录

除必需的 `SKILL.md` 外，技能目录可以包含任何文件和目录。以下约定是组织常见内容类型的建议。

### `scripts/`

包含智能体可以运行的可执行代码。脚本应当：
- 自成一体，或清晰地记录依赖
- 包含有用的错误信息
- 优雅地处理边界情况

支持的语言取决于智能体实现。常见选择包括 Python、Bash 和 JavaScript。

### `references/`

包含智能体在需要时可以阅读的附加文档：
- `REFERENCE.md` - 详细的技术参考
- `FORMS.md` - 表单模板或结构化数据格式
- 领域特定文件（`finance.md`、`legal.md` 等）

让每个[引用文件](#file-references)保持聚焦。智能体按需加载这些文件，因此文件越小，占用的上下文越少。

### `assets/`

包含静态资源：
- 模板（文档模板、配置模板）
- 图片（示意图、示例）
- 数据文件（查找表、schema）

## 渐进式披露

智能体*渐进式*加载技能，只在任务需要时才引入更多细节。技能的结构应便于利用这一点：

1. **元数据**（约 100 tokens）：启动时会为所有技能加载 `name` 和 `description` 字段
2. **指令**（建议少于 5000 tokens）：技能被激活时加载完整的 `SKILL.md` 正文
3. **资源**（按需）：文件（例如 `scripts/`、`references/` 或 `assets/` 中的文件）仅在需要时加载

让主 `SKILL.md` 保持在 500 行以内。把详细的参考资料移到单独的文件中。

## 文件引用

引用技能中的其他文件时，使用相对于技能根目录的相对路径：

```markdown SKILL.md
See [the reference guide](references/REFERENCE.md) for details.

Run the extraction script:
scripts/extract.py
```

让文件引用保持在距离 `SKILL.md` 一级的深度。避免深层嵌套的引用链。

## 校验

使用 [skills-ref](https://github.com/agentskills/agentskills/tree/main/skills-ref) 参考库来校验你的技能：

```bash
skills-ref validate ./my-skill
```

这会检查你的 `SKILL.md` frontmatter 是否有效，以及是否遵循所有命名约定。
