---
title: "评估技能输出质量"
sidebarTitle: "评估技能"
description: "如何通过评估驱动的迭代来测试你的技能是否产出好的结果。"
---

你写了一个技能，在一条提示上试了试，看起来能用。但它是否可靠——在各种不同的提示下、在边缘情况下、比完全不用技能更好？运行结构化的评估（eval）能回答这些问题，并给你一个系统地改进技能的反馈循环。

## 设计测试用例

一个测试用例由三部分组成：

- **提示（Prompt）**：一条真实的用户消息——某个人真的会打出来的那种话。
- **预期输出**：对成功是什么样子的、人类可读的描述。
- **输入文件**（可选）：技能需要处理的文件。

把测试用例存放在技能目录内的 `evals/evals.json` 中：

```json evals/evals.json
{
  "skill_name": "csv-analyzer",
  "evals": [
    {
      "id": 1,
      "prompt": "I have a CSV of monthly sales data in data/sales_2025.csv. Can you find the top 3 months by revenue and make a bar chart?",
      "expected_output": "A bar chart image showing the top 3 months by revenue, with labeled axes and values.",
      "files": ["evals/files/sales_2025.csv"]
    },
    {
      "id": 2,
      "prompt": "there's a csv in my downloads called customers.csv, some rows have missing emails — can you clean it up and tell me how many were missing?",
      "expected_output": "A cleaned CSV with missing emails handled, plus a count of how many were missing.",
      "files": ["evals/files/customers.csv"]
    }
  ]
}
```

**编写好的测试提示的技巧：**

- **从 2-3 个测试用例开始。** 在见到第一轮结果之前，不要投入过多。之后你可以再扩充测试集。
- **让提示多样化。** 使用不同的措辞、详细程度和正式程度。有些提示应该随意（“hey can you clean up this csv”），有些则要精确（“Parse the CSV at data/input.csv, drop rows where column B is null, and write the result to data/output.csv”）。
- **覆盖边缘情况。** 至少包含一条测试边界条件的提示——格式错误的输入、不寻常的请求，或者技能指令可能含义模糊的情况。
- **使用真实的上下文。** 真实用户会提到文件路径、列名和个人背景。像“process this data”这样的提示太含糊，测不出任何有用的东西。

暂时不必操心定义具体的通过/失败检查——先写好提示和预期输出就行。等你看过第一轮运行产出什么之后，再添加详细的检查（称为断言）。

## 运行评估

核心模式是把每个测试用例运行两次：一次**带技能**，一次**不带技能**（或者带一个旧版本）。这样你就有了一个可作比较的基线。

### 工作区结构

把评估结果整理在与技能目录并列的一个工作区目录中。每完整跑完一轮评估循环，都会有自己独立的 `iteration-N/` 目录。在其中，每个测试用例都有一个评估目录，里面包含 `with_skill/` 和 `without_skill/` 子目录：

```
csv-analyzer/
├── SKILL.md
└── evals/
    └── evals.json
csv-analyzer-workspace/
└── iteration-1/
    ├── eval-top-months-chart/
    │   ├── with_skill/
    │   │   ├── outputs/       # Files produced by the run
    │   │   ├── timing.json    # Tokens and duration
    │   │   └── grading.json   # Assertion results
    │   └── without_skill/
    │       ├── outputs/
    │       ├── timing.json
    │       └── grading.json
    ├── eval-clean-missing-emails/
    │   ├── with_skill/
    │   │   ├── outputs/
    │   │   ├── timing.json
    │   │   └── grading.json
    │   └── without_skill/
    │       ├── outputs/
    │       ├── timing.json
    │       └── grading.json
    └── benchmark.json         # Aggregated statistics
```

你手写的主要文件是 `evals/evals.json`。其他 JSON 文件（`grading.json`、`timing.json`、`benchmark.json`）都是在评估过程中产生的——由 agent、脚本或你自己生成。

### 启动运行

每次评估运行都应从干净的上下文开始——不残留此前运行或技能开发过程的状态。这能确保 agent 只遵循 `SKILL.md` 告诉它的内容。在支持子 agent 的环境（例如 Claude Code）中，这种隔离是自然而然就有的：每个子任务都从全新状态开始。如果没有子 agent，就为每次运行使用单独的会话。

每次运行都需要提供：

- 技能路径（基线运行则不提供技能）
- 测试提示
- 所有输入文件
- 输出目录

下面是一个示例，展示你会给 agent 的、针对单次带技能运行的指令：

```
Execute this task:
- Skill path: /path/to/csv-analyzer
- Task: I have a CSV of monthly sales data in data/sales_2025.csv.
  Can you find the top 3 months by revenue and make a bar chart?
- Input files: evals/files/sales_2025.csv
- Save outputs to: csv-analyzer-workspace/iteration-1/eval-top-months-chart/with_skill/outputs/
```

对于基线，使用同样的提示，但不给技能路径，并把结果保存到 `without_skill/outputs/`。

在改进一个已有技能时，用之前的版本作为基线。编辑之前先给它做个快照（`cp -r <skill-path> <workspace>/skill-snapshot/`），把基线运行指向该快照，并把结果保存到 `old_skill/outputs/`，而不是 `without_skill/`。

### 采集计时数据

计时数据让你能比较技能相对于基线花费了多少时间和 token——一个大幅提升输出质量但让 token 用量增加两倍的技能，与一个既更好又更便宜的技能是两种不同的取舍。每次运行完成后，记录 token 数量和耗时：

```json timing.json
{
  "total_tokens": 84852,
  "duration_ms": 23332
}
```

<Tip>
在 Claude Code 中，子 agent 任务完成时，[任务完成通知](https://platform.claude.com/docs/en/agent-sdk/typescript#sdk-task-notification-message)会包含 `total_tokens` 和 `duration_ms`。请立即保存这些值——它们不会被持久化到其他任何地方。
</Tip>

## 编写断言

断言是关于输出应该包含什么或达成什么的可验证陈述。等你看过第一轮输出之后再添加它们——在技能跑起来之前，你往往并不知道“好”是什么样子。

好的断言：

- `"The output file is valid JSON"` —— 可以通过程序验证。
- `"The bar chart has labeled axes"` —— 具体且可观察。
- `"The report includes at least 3 recommendations"` —— 可计数。

弱的断言：

- `"The output is good"` —— 太含糊，无法评分。
- `"The output uses exactly the phrase 'Total Revenue: $X'"` —— 太脆弱；措辞不同但正确的输出也会失败。

并非所有东西都需要断言。有些品质——写作风格、视觉设计、输出是否“感觉对”——很难拆解成通过/失败的检查。这些更适合在[人工评审](#reviewing-results-with-a-human)中发现。把断言留给那些可以客观检查的东西。

在 `evals/evals.json` 中为每个测试用例添加断言：

```json evals/evals.json highlight={9-14}
{
  "skill_name": "csv-analyzer",
  "evals": [
    {
      "id": 1,
      "prompt": "I have a CSV of monthly sales data in data/sales_2025.csv. Can you find the top 3 months by revenue and make a bar chart?",
      "expected_output": "A bar chart image showing the top 3 months by revenue, with labeled axes and values.",
      "files": ["evals/files/sales_2025.csv"],
      "assertions": [
        "The output includes a bar chart image file",
        "The chart shows exactly 3 months",
        "Both axes are labeled",
        "The chart title or caption mentions revenue"
      ]
    }
  ]
}
```

## 给输出评分

评分是指针对实际输出逐条评估每个断言，并记录 **PASS** 或 **FAIL** 以及具体证据。证据应当引用或指向输出，而不只是陈述观点。

最简单的做法是把输出和断言交给 LLM，让它逐条评估。对于可以用代码检查的断言（JSON 是否有效、行数是否正确、文件是否存在且尺寸符合预期），请使用验证脚本——对于机械性的检查，脚本比 LLM 的判断更可靠，而且可以跨迭代复用。

```json grading.json
{
  "assertion_results": [
    {
      "text": "The output includes a bar chart image file",
      "passed": true,
      "evidence": "Found chart.png (45KB) in outputs directory"
    },
    {
      "text": "The chart shows exactly 3 months",
      "passed": true,
      "evidence": "Chart displays bars for March, July, and November"
    },
    {
      "text": "Both axes are labeled",
      "passed": false,
      "evidence": "Y-axis is labeled 'Revenue ($)' but X-axis has no label"
    },
    {
      "text": "The chart title or caption mentions revenue",
      "passed": true,
      "evidence": "Chart title reads 'Top 3 Months by Revenue'"
    }
  ],
  "summary": {
    "passed": 3,
    "failed": 1,
    "total": 4,
    "pass_rate": 0.75
  }
}
```

### 评分原则

- **PASS 必须有具体证据。** 不要做有利于输出的假设。如果断言说“包含一段总结”，而输出中有一个标题为“Summary”的小节却只有一句含糊的话，那就是 FAIL——标签在，实质不在。
- **要审视断言本身，而不只是结果。** 在评分过程中，注意哪些断言太容易（无论技能质量如何总是通过）、太难（即使输出很好也总是失败），或不可验证（仅凭输出无法检查）。在下一次迭代中修正这些问题。

<Tip>
要比较两个技能版本，可以试试**盲评**：把两份输出都交给 LLM 评审，但不透露哪份来自哪个版本。评审者按照自己的评分标准对整体品质打分——组织结构、格式、可用性、完成度——从而不受“哪个版本应该更好”这一偏见的影响。这与断言评分互为补充：两份输出可能都通过了所有断言，但在整体质量上差异显著。
</Tip>

## 汇总结果

当这一轮迭代中的每次运行都评分完毕后，按配置计算汇总统计，并把它们与各评估目录一起保存到 `benchmark.json`（例如 `csv-analyzer-workspace/iteration-1/benchmark.json`）：

```json benchmark.json
{
  "run_summary": {
    "with_skill": {
      "pass_rate": { "mean": 0.83, "stddev": 0.06 },
      "time_seconds": { "mean": 45.0, "stddev": 12.0 },
      "tokens": { "mean": 3800, "stddev": 400 }
    },
    "without_skill": {
      "pass_rate": { "mean": 0.33, "stddev": 0.10 },
      "time_seconds": { "mean": 32.0, "stddev": 8.0 },
      "tokens": { "mean": 2100, "stddev": 300 }
    },
    "delta": {
      "pass_rate": 0.50,
      "time_seconds": 13.0,
      "tokens": 1700
    }
  }
}
```

`delta` 告诉你技能的成本（更多时间、更多 token）和它带来的收益（更高的通过率）。一个增加 13 秒但把通过率提升 50 个百分点的技能，多半是值得的。一个让 token 用量翻倍却只提升 2 个百分点的技能，可能就不值得。

<Note>
标准差（`stddev`）只有在每个评估运行多次时才有意义。在只有 2-3 个测试用例且每个只跑一次的早期迭代中，请关注原始通过计数和 delta——随着你扩充测试集并对每个评估运行多次，这些统计指标才会变得有用。
</Note>

## 分析规律

汇总统计可能掩盖重要的规律。在计算完基准数据之后：

- **移除或替换在两种配置下都总是通过的断言。** 这些断言提供不了任何有用信息——模型不用技能也能处理得很好。它们抬高了带技能的通过率，却没有反映技能的实际价值。
- **调查在两种配置下都总是失败的断言。** 要么断言本身有问题（要求了模型做不到的事），要么测试用例太难，要么断言检查的东西不对。在下一次迭代之前修正这些问题。
- **研究带技能通过、不带技能失败的断言。** 这正是技能明显在创造价值的地方。要理解*为什么*——是哪些指令或脚本带来了差异？
- **当结果在各次运行之间不一致时，收紧指令。** 如果同一个评估有时通过有时失败（在基准数据中表现为较高的 `stddev`），可能是这个评估本身不稳定（对模型的随机性敏感），也可能是技能的指令模糊到模型每次都会做出不同解读。请添加示例或更具体的指引来减少歧义。
- **检查时间和 token 的离群值。** 如果某个评估的耗时是其他评估的 3 倍，请阅读它的执行记录（模型在运行期间所做全部操作的完整日志）来找出瓶颈。

## 结合人工评审结果

断言评分和规律分析能发现很多问题，但它们只能检查你想到了要写断言的那些方面。人工评审者能带来新鲜的视角——发现你未曾预料的问题，注意到输出在技术上正确却偏离了重点，或识别出难以用通过/失败检查来表达的问题。对每个测试用例，请把实际输出与评分结果放在一起评审。

为每个测试用例记录具体的反馈，并保存在工作区中（例如作为 `feedback.json` 与各评估目录并列）：

```json feedback.json
{
  "eval-top-months-chart": "The chart is missing axis labels and the months are in alphabetical order instead of chronological.",
  "eval-clean-missing-emails": ""
}
```

“The chart is missing axis labels”是可操作的；“looks bad”则不是。反馈为空意味着输出看起来没问题——该测试用例通过了你的评审。在[迭代步骤](#iterating-on-the-skill)中，把改进重点放在你提出具体意见的那些测试用例上。

## 迭代改进技能

评分和评审之后，你会有三个信号来源：

- **失败的断言**指向具体的缺口——缺失的步骤、不清楚的指令，或技能未处理的情况。
- **人工反馈**指向更广泛的品质问题——方法错了、输出结构糟糕，或者技能产出了技术上正确但没什么帮助的结果。
- **执行记录**揭示了事情出错的*原因*。如果 agent 忽略了某条指令，那条指令可能含义模糊。如果 agent 把时间花在无成效的步骤上，那些指令可能需要简化或删除。

把这些信号转化为技能改进的最有效方式，是把这三者连同当前的 `SKILL.md` 一起交给 LLM，让它提出修改建议。LLM 能够综合出跨越失败断言、评审者意见和记录中行为的规律，而人工去关联这些会非常繁琐。向 LLM 提问时，请包含以下准则：

- **从反馈中泛化。** 技能会被用于许多不同的提示，而不只是测试用例。修正应当广泛地解决根本问题，而不是为具体示例打上狭窄的补丁。
- **保持技能精简。** 更少但更好的指令，往往胜过详尽无遗的规则。如果记录显示存在无效工作（不必要的验证、不需要的中间输出），请删除那些指令。如果不断增加规则却让通过率停滞不前，技能可能被约束过度了——试着删掉一些指令，看看结果是否能保持或改善。
- **解释为什么。** 基于推理的指令（“做 X，因为 Y 往往会导致 Z”）比僵硬的命令（“永远要做 X，绝不要做 Y”）效果更好。当模型理解了目的，它们会更可靠地遵循指令。
- **把重复的工作打包。** 如果每次测试运行都各自写了一个类似的辅助脚本（图表生成器、数据解析器），这就是一个信号：应该把该脚本打包进技能的 `scripts/` 目录。具体做法请参阅[使用脚本](/skill-creation/using-scripts)。

### 循环

1. 把评估信号和当前的 `SKILL.md` 交给 LLM，让它提出改进建议。
2. 评审并应用这些修改。
3. 在新的 `iteration-<N+1>/` 目录中重新运行所有测试用例。
4. 对新的结果进行评分和汇总。
5. 结合人工评审。重复。

当你对结果满意、反馈持续为空，或者各轮迭代之间不再看到有意义的改进时，就可以停止。

<Tip>
[`skill-creator`](https://github.com/anthropics/skills/tree/main/skills/skill-creator) 技能可以自动化这一工作流的许多环节——运行评估、给断言评分、汇总基准数据，以及呈现结果供人工评审。
</Tip>
