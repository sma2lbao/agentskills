---
title: "优化技能描述"
sidebarTitle: "优化描述"
description: "如何改进技能的 description，使其在相关提示下可靠触发。"
---

技能只有在被激活时才有用。`SKILL.md` frontmatter 中的 `description` 字段，是 agent 用来判断是否为给定任务加载某个技能的主要机制。描述写得不够充分，技能就会在该触发的时候不触发；描述写得过于宽泛，技能就会在不该触发的时候触发。

本指南介绍如何系统地测试并改进技能描述的触发准确性。

## 技能触发的工作原理

Agent 使用[渐进式披露](/specification#progressive-disclosure)来管理上下文。启动时，它们只加载每个可用技能的 `name` 和 `description` —— 这些信息刚好够判断某个技能何时可能相关。当用户的任务与某个描述匹配时，agent 会把完整的 `SKILL.md` 读入上下文，并遵循其中的指令。

这意味着描述承担了触发的全部责任。如果描述没有传达出技能在何时有用，agent 就不知道要去使用它。

有一个重要的细节：agent 通常只会为那些需要超出自身处理能力的知识或能力的任务去查阅技能。像“读取这个 PDF”这样简单的一步请求，即使描述完全匹配，也可能不会触发 PDF 技能，因为 agent 用基础工具就能处理。真正能体现描述写得好坏差异的，是那些涉及专门知识的任务——不熟悉的 API、特定领域的工作流，或是不常见的格式。

## 编写有效的描述

在开始测试之前，先了解好的描述是什么样子会有所帮助。有几条原则：

- **使用祈使句式。** 把描述写成给 agent 的指令：“当……时使用此技能”，而不是“此技能用于……”。Agent 正在决定是否行动，所以要告诉它何时行动。
- **关注用户意图，而非实现细节。** 描述用户想要达成什么，而不是技能的内部机制。Agent 是拿用户提出的请求去匹配的。
- **宁可写得主动一些。** 明确列出技能适用的场景，包括用户没有直接点名该领域的情况：“即使他们没有明确提到‘CSV’或‘分析’。”
- **保持简洁。** 几句话到一小段通常就合适——长到足以覆盖技能的范围，短到不会在众多技能中撑大 agent 的上下文。[规范](/specification#description-field)强制要求 1024 个字符的硬性上限。

## 设计触发评估查询

要测试触发情况，你需要一组评估查询 —— 也就是真实的用户提示，并标注它们是否应该触发你的技能。

```json eval_queries.json
[
  { "query": "I've got a spreadsheet in ~/data/q4_results.xlsx with revenue in col C and expenses in col D — can you add a profit margin column and highlight anything under 10%?", "should_trigger": true },
  { "query": "whats the quickest way to convert this json file to yaml", "should_trigger": false }
]
```

目标大约 20 条查询：8-10 条应该触发，8-10 条不应该触发。

### 应该触发的查询

这些查询用来检验描述是否覆盖了技能的范围。可以从几个维度来变化它们：

- **措辞**：有些正式，有些随意，有些带拼写错误或缩写。
- **明确程度**：有些直接点名技能的领域（“分析这个 CSV”），有些只描述需求而不点名（“我老板想根据这个数据文件做一张图表”）。
- **详细程度**：把简短的提示和包含大量上下文的提示混在一起——既有简短的“分析我的销售 CSV 并做一张图表”，也有包含文件路径、列名和背景信息的长消息。
- **复杂度**：变化步骤数量和决策点数量。把单步任务和多步工作流放在一起，以检验当技能所处理的任务被埋藏在更长的链条中时，agent 是否还能识别出该技能是相关的。

最有用的应该触发查询，是那些技能本来会有所帮助、但仅从查询本身看不出这种关联的查询。这些正是描述措辞能起决定作用的场景——如果查询已经明确要求了技能所做的事，那么任何合理的描述都会触发。

### 不应该触发的查询

最有价值的负面测试用例是**近似案例**——它们与你的技能共享关键词或概念，但实际上需要的是别的东西。这些用例检验的是描述是否精确，而不只是是否宽泛。

对于一个 CSV 分析技能，较弱的负面示例如下：

- `"Write a fibonacci function"` —— 明显不相关，什么都测不出来。
- `"What's the weather today?"` —— 没有关键词重叠，太简单了。

较强的负面示例如下：

- `"I need to update the formulas in my Excel budget spreadsheet"` —— 共享“spreadsheet”和“data”的概念，但需要的是 Excel 编辑，而不是 CSV 分析。
- `"can you write a python script that reads a csv and uploads each row to our postgres database"` —— 涉及 CSV，但任务是数据库 ETL，而不是分析。

### 保持真实感的技巧

真实的用户提示包含通用测试查询所缺少的上下文。请加入：

- 文件路径（`~/Downloads/report_final_v2.xlsx`）
- 个人背景（`"my manager asked me to..."`）
- 具体细节（列名、公司名称、数据值）
- 口语化表达、缩写，以及偶尔出现的拼写错误

## 测试描述是否会触发

基本做法是：在安装了该技能的情况下，把每条查询都跑一遍你的 agent，并观察 agent 是否调用了它。要确保技能已注册并且能被你的 agent 发现——具体方式因客户端而异（例如技能目录、配置文件或 CLI 标志）。

大多数 agent 客户端都提供某种形式的可观测性——执行日志、工具调用历史或详细输出——让你能看到一次运行中查阅了哪些技能。具体细节请查阅你的客户端文档。如果 agent 加载了你技能的 `SKILL.md`，就说明触发了；如果 agent 在没有查阅它的情况下继续工作，就说明没有触发。

一条查询“通过”的条件是：
- `should_trigger` 为 `true` 且技能被调用，或者
- `should_trigger` 为 `false` 且技能未被调用。

### 多次运行

模型行为是非确定性的——同一条查询可能这一次触发了技能，下一次却没有。请把每条查询运行多次（3 次是一个合理的起点），并计算**触发率**：技能被调用的运行次数占比。

一条应该触发的查询，如果其触发率高于某个阈值（0.5 是一个合理的默认值），就算通过。一条不应该触发的查询，如果其触发率低于该阈值，就算通过。

20 条查询各运行 3 次，就是 60 次调用。你会想用脚本来自动完成这件事。下面是总体结构——请把 `check_triggered` 中的 `claude` 调用和检测逻辑替换成你的 agent 客户端所提供的相应内容：

```bash
#!/bin/bash
QUERIES_FILE="${1:?Usage: $0 <queries.json>}"
SKILL_NAME="my-skill"
RUNS=3

# This example uses Claude Code's JSON output to check for Skill tool calls.
# Replace this function with detection logic for your agent client.
# Should return 0 (success) if the skill was invoked, 1 otherwise.
check_triggered() {
  local query="$1"
  claude -p "$query" --output-format json 2>/dev/null \
    | jq -e --arg skill "$SKILL_NAME" \
      'any(.messages[].content[]; .type == "tool_use" and .name == "Skill" and .input.skill == $skill)' \
      > /dev/null 2>&1
}

count=$(jq length "$QUERIES_FILE")
for i in $(seq 0 $((count - 1))); do
  query=$(jq -r ".[$i].query" "$QUERIES_FILE")
  should_trigger=$(jq -r ".[$i].should_trigger" "$QUERIES_FILE")
  triggers=0

  for run in $(seq 1 $RUNS); do
    check_triggered "$query" && triggers=$((triggers + 1))
  done

  jq -n \
    --arg query "$query" \
    --argjson should_trigger "$should_trigger" \
    --argjson triggers "$triggers" \
    --argjson runs "$RUNS" \
    '{query: $query, should_trigger: $should_trigger, triggers: $triggers, runs: $runs, trigger_rate: ($triggers / $runs)}'
done | jq -s '.'
```

<Tip>
如果你的 agent 客户端支持，你可以在结果已经明确时提前终止一次运行——agent 要么查阅了技能，要么没查阅就开始工作了。这能显著减少运行整套评估集所需的时间和成本。
</Tip>

## 用训练/验证集划分避免过拟合

如果你针对所有查询来优化描述，就会有过拟合的风险——调出的描述在这些特定措辞上有效，但在新的措辞上失效。

解决办法是划分你的查询集：

- **训练集（约 60%）**：你用来发现失败并指导改进的查询。
- **验证集（约 40%）**：你留出来、只用来检查改进是否具备泛化能力的查询。

要确保两个集合都按比例包含应该触发和不应该触发的查询——不要不小心把所有正例都放进同一个集合。请随机打乱，并在各轮迭代中保持划分固定，这样才能做到同类比较。

如果你使用的是像[上文](#running-multiple-times)那样的脚本，可以把查询拆成两个文件——`train_queries.json` 和 `validation_queries.json`——然后分别对每个文件运行该脚本。

## 优化循环

1. **评估**当前描述在*训练集和验证集*上的表现。训练集结果指导你的修改；验证集结果告诉你这些修改是否在泛化。
2. **找出失败项**，在*训练集*中：哪些应该触发的查询没有触发？哪些不应该触发的查询触发了？
   - 只用训练集的失败项来指导修改——无论你是自己修改描述还是让 LLM 来修改，都要把验证集结果排除在这个过程之外。
3. **修改描述。** 着眼于泛化：
   - 如果应该触发的查询失败了，描述可能太窄。请拓宽范围，或补充说明技能在何时有用。
   - 如果不应该触发的查询被误触发了，描述可能太宽。请补充说明技能*不*做什么，或厘清该技能与相邻能力之间的边界。
   - 避免把失败查询中的具体关键词加进去——那是过拟合。相反，要找出这些查询所代表的一般类别或概念，并针对它来处理。
   - 如果迭代几次之后仍然卡住，试着对描述做结构上不同的改法，而不是做增量微调。不同的表述框架或句子结构，可能突破细化所无法突破的瓶颈。
   - 检查描述是否仍在 1024 字符上限之内——描述在优化过程中往往会变长。
4. **重复**步骤 1-3，直到所有*训练集*查询都通过，或者你不再看到有意义的改进。
5. **按验证集通过率挑选最佳迭代**——即*验证集*中通过的查询占比。注意，最佳描述未必是你最后产出的那一版；较早的某个迭代可能比后来那些对训练集过拟合的迭代有更高的验证集通过率。

通常迭代五次就够了。如果表现没有改善，问题可能出在查询上（太简单、太难，或者标注不当），而不是描述上。

<Tip>
[`skill-creator`](https://github.com/anthropics/skills/tree/main/skills/skill-creator) 技能可以端到端地自动完成这个循环：它划分评估集、并行评估触发率、使用 Claude 提出描述改进建议，并生成一份可实时观看的 HTML 报告。
</Tip>

## 应用结果

一旦你选出了最佳描述：

1. 更新 `SKILL.md` frontmatter 中的 `description` 字段。
2. 确认描述在 [1024 字符上限](/specification#description-field)之内。
3. 确认描述能按预期触发。手动试几条提示，做一次快速的基本检查。若要做更严格的测试，请新写 5-10 条查询（应该触发和不应该触发的混合），并把它们跑一遍评估脚本——由于这些查询从未参与优化过程，它们能诚实地检验描述是否具备泛化能力。

改进前后对比：

```yaml
# Before
description: Process CSV files.

# After
description: >
  Analyze CSV and tabular data files — compute summary statistics,
  add derived columns, generate charts, and clean messy data. Use this
  skill when the user has a CSV, TSV, or Excel file and wants to
  explore, transform, or visualize the data, even if they don't
  explicitly mention "CSV" or "analysis."
```

改进后的描述更具体地说明了技能做什么（汇总统计、派生列、图表、数据清洗），也更宽泛地说明了它何时适用（CSV、TSV、Excel；即使没有明确的关键词）。

## 后续步骤

一旦你的技能能够可靠触发，你就会想评估它是否产出了好的结果。请参阅[评估技能输出质量](/skill-creation/evaluating-skills)，了解如何设置测试用例、对结果评分并迭代。
