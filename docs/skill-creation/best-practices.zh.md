---
title: "技能创作者的最佳实践"
sidebarTitle: "最佳实践"
description: "如何编写范围界定良好、且与任务相匹配的技能。"
---

## 从真实专业知识出发

技能创建中一个常见的陷阱，是要求 LLM 在未提供领域特定上下文的情况下生成技能——仅依赖 LLM 的通用训练知识。结果就是含糊、泛泛的流程（"妥善处理错误"、"遵循身份验证的最佳实践"），而不是让技能真正有价值的那些具体 API 模式、边界情况和项目约定。

有效的技能以真实专业知识为基础。关键在于把领域特定的上下文输入到创建过程中。

### 从亲手完成的任务中提炼

与 agent 对话完成一项真实任务，在此过程中提供上下文、纠正和偏好。然后把可复用的模式提炼成一个技能。注意以下几点：

- **奏效的步骤** —— 导致成功的操作顺序
- **你做出的纠正** —— 你引导 agent 调整方法的地方（例如 "用库 X 而不是 Y"、"检查边界情况 Z"）
- **输入/输出格式** —— 数据进入和输出时的样子
- **你提供的上下文** —— agent 原本不知道的项目特定事实、约定或约束

### 从现有项目产物中综合

当你已经拥有大量现成知识时，可以把它输入 LLM，让它综合出一个技能。从你团队真实的故障报告和运行手册中综合出的数据管道技能，会胜过从一篇泛泛的"数据工程最佳实践"文章中综合出的技能，因为它捕捉的是*你的* schema、故障模式和恢复流程。关键在于项目特定的材料，而不是通用参考资料。

良好的素材来源包括：

- 内部文档、运行手册和风格指南
- API 规范、schema 和配置文件
- 代码评审意见和 issue 跟踪器（捕捉反复出现的关注点和评审者的期望）
- 版本控制历史，尤其是补丁和修复（通过实际改动的内容揭示模式）
- 真实世界的失败案例及其解决方式

## 用真实执行来打磨

技能的初稿通常需要打磨。让技能面对真实任务运行，然后把结果——所有结果，而不只是失败——反馈到创建过程中。问一问：什么触发了误报？什么被遗漏了？什么可以删掉？

即使只做一轮"执行后修订"，质量也会明显提升，而复杂领域往往需要好几轮。

<Tip>
阅读 agent 的执行轨迹，而不只是最终输出。如果 agent 把时间浪费在无效步骤上，常见原因包括：指令过于含糊（agent 尝试了多种方法才找到可行的那个）、指令不适用于当前任务（agent 仍然照做），或者给出了太多选项却没有明确的默认值。
</Tip>

关于更结构化的迭代方法，包括测试用例、断言和评分，请参见[评估技能输出质量](/skill-creation/evaluating-skills)。

## 明智地使用上下文

技能一旦激活，其完整的 `SKILL.md` 正文就会与对话历史、系统上下文和其他已激活技能一起加载到 agent 的上下文窗口中。技能里的每一个 token 都在与窗口中其他所有内容争夺 agent 的注意力。

### 补充 agent 所缺，省略它已知的

专注于 agent *没有*你的技能就不会知道的内容：项目特定的约定、领域特定的流程、不明显的边界情况，以及要使用的特定工具或 API。你不需要解释 PDF 是什么、HTTP 如何工作，或者数据库迁移会做什么。

````markdown
<!-- Too verbose — the agent already knows what PDFs are -->
## Extract PDF text

PDF (Portable Document Format) files are a common file format that contains
text, images, and other content. To extract text from a PDF, you'll need to
use a library. pdfplumber is recommended because it handles most cases well.

<!-- Better — jumps straight to what the agent wouldn't know on its own -->
## Extract PDF text

Use pdfplumber for text extraction. For scanned documents, fall back to
pdf2image with pytesseract.

```python
import pdfplumber

with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```
````

对每一处内容问自己："如果没有这条指令，agent 会做错吗？"如果答案是否定的，就删掉它。如果不确定，就测试一下。而如果 agent 在没有这个技能的情况下就已经能很好地完成整个任务，那这个技能可能并没有增加价值。关于如何系统地测试这一点，请参见[评估技能输出质量](/skill-creation/evaluating-skills)。

### 设计连贯的单元

决定一个技能应覆盖什么，就像决定一个函数应该做什么：你希望它封装一个连贯的工作单元，并能与其他技能良好组合。范围界定得过窄的技能会迫使单个任务加载多个技能，带来开销和指令冲突的风险。范围界定得过宽的技能则难以被精确激活。一个用于查询数据库并格式化结果的技能可以算作一个连贯单元，而一个还涵盖数据库管理的技能大概就承担得太多了。

### 追求适度的详细程度

过于面面俱到的技能可能弊大于利——agent 难以提取相关内容，并可能被不适用于当前任务的指令触发而走上无效路径。简洁、分步骤的指导加上一个可运行的示例，往往胜过详尽的文档。当你发现自己要覆盖每一种边界情况时，想一想其中大多数是否交给 agent 自己的判断更好。

### 用渐进式披露组织大型技能

[规范](/specification#progressive-disclosure)建议将 `SKILL.md` 保持在 500 行和 5,000 个 token 以内——只保留 agent 每次运行都需要的核心指令。当技能确实需要更多内容时，把详细的参考材料移到 `references/` 或类似目录中的独立文件里。

关键在于告诉 agent *何时*加载每个文件。"如果 API 返回非 200 状态码，请阅读 `references/api-errors.md`"比泛泛的"详情见 references/"更有用。这让 agent 能够按需加载上下文，而不是预先加载，这正是[渐进式披露](/specification#progressive-disclosure)的设计工作方式。

## 校准控制程度

并非技能的每一部分都需要同样程度的规范性。让指令的具体程度与任务的脆弱程度相匹配。

### 让具体程度匹配脆弱程度

**给 agent 自由**，当多种方法都可行、且任务能容忍差异时。对于灵活的指令，解释*为什么*可能比硬性规定更有效——理解指令背后目的的 agent 能做出更好的依赖上下文的决策。代码评审技能可以描述要查找什么，而不规定确切步骤：

```markdown
## Code review process

1. Check all database queries for SQL injection (use parameterized queries)
2. Verify authentication checks on every endpoint
3. Look for race conditions in concurrent code paths
4. Confirm error messages don't leak internal details
```

**要规定得具体**，当操作脆弱、一致性很重要，或必须遵循特定顺序时：

````markdown
## Database migration

Run exactly this sequence:

```bash
python scripts/migrate.py --verify --backup
```

Do not modify the command or add additional flags.
````

大多数技能是两者的混合。请独立校准每一部分。

### 提供默认值，而不是菜单

当多种工具或方法都可能奏效时，选定一个默认值，并简要提及备选方案，而不是把它们作为同等的选项一一列出。

````markdown
<!-- Too many options -->
You can use pypdf, pdfplumber, PyMuPDF, or pdf2image...

<!-- Clear default with escape hatch -->
Use pdfplumber for text extraction:

```python
import pdfplumber
```

For scanned PDFs requiring OCR, use pdf2image with pytesseract instead.
````

### 倾向于流程而非结论

技能应当教会 agent *如何处理*一类问题，而不是针对某个具体实例*产出什么*。比较：

```markdown
<!-- Specific answer — only useful for this exact task -->
Join the `orders` table to `customers` on `customer_id`, filter where
`region = 'EMEA'`, and sum the `amount` column.

<!-- Reusable method — works for any analytical query -->
1. Read the schema from `references/schema.yaml` to find relevant tables
2. Join tables using the `_id` foreign key convention
3. Apply any filters from the user's request as WHERE clauses
4. Aggregate numeric columns as needed and format as a markdown table
```

这并不意味着技能不能包含具体细节——输出格式模板（见[输出格式模板](#templates-for-output-format)）、诸如"绝不输出 PII"之类的约束，以及针对特定工具的指令，都是有价值的。关键在于：即使个别细节很具体，*处理方式*也应当是可推广的。

## 有效指令的模式

这些是用于组织技能内容的可复用技巧。并非每个技能都需要全部——选择适合你任务的那些。

### 陷阱（Gotchas）小节

许多技能中价值最高的内容是一份陷阱清单——那些违背合理假设的环境特定事实。这些不是泛泛的建议（"妥善处理错误"），而是针对 agent 在无人告知时必然会犯的错误的具象纠正：

````markdown
## Gotchas

- The `users` table uses soft deletes. Queries must include
  `WHERE deleted_at IS NULL` or results will include deactivated accounts.
- The user ID is `user_id` in the database, `uid` in the auth service,
  and `accountId` in the billing API. All three refer to the same value.
- The `/health` endpoint returns 200 as long as the web server is running,
  even if the database connection is down. Use `/ready` to check full
  service health.
````

把陷阱放在 `SKILL.md` 中，让 agent 在遇到该情况之前就读到它们。单独的参考文件也可以，前提是你告诉 agent 何时加载它；但对于不明显的问题，agent 可能意识不到这个触发条件。

<Tip>
当 agent 犯了一个你必须纠正的错误时，把这条纠正加入陷阱小节。这是迭代改进技能最直接的方式之一（见[用真实执行来打磨](#refine-with-real-execution)）。
</Tip>

### 输出格式模板

当你需要 agent 以特定格式产出时，提供一个模板。这比用文字描述格式更可靠，因为 agent 擅长对具体结构进行模式匹配。简短的模板可以内联放在 `SKILL.md` 中；对于较长的模板，或只在某些情况下需要的模板，把它们存放在 `assets/` 中并从 `SKILL.md` 引用，这样它们只在需要时才加载。

````markdown
## Report structure

Use this template, adapting sections as needed for the specific analysis:

```markdown
# [Analysis Title]

## Executive summary
[One-paragraph overview of key findings]

## Key findings
- Finding 1 with supporting data
- Finding 2 with supporting data

## Recommendations
1. Specific actionable recommendation
2. Specific actionable recommendation
```
````

### 多步骤工作流的检查清单

明确的检查清单能帮助 agent 跟踪进度并避免跳过步骤，尤其是当步骤之间存在依赖或验证关口时。

```markdown
## Form processing workflow

Progress:
- [ ] Step 1: Analyze the form (run `scripts/analyze_form.py`)
- [ ] Step 2: Create field mapping (edit `fields.json`)
- [ ] Step 3: Validate mapping (run `scripts/validate_fields.py`)
- [ ] Step 4: Fill the form (run `scripts/fill_form.py`)
- [ ] Step 5: Verify output (run `scripts/verify_output.py`)
```

### 验证循环

指示 agent 在继续之前验证自己的工作。模式是：完成工作，运行验证器（一个脚本、一份参考检查清单，或一次自检），修复所有问题，然后重复直到验证通过。

```markdown
## Editing workflow

1. Make your edits
2. Run validation: `python scripts/validate.py output/`
3. If validation fails:
   - Review the error message
   - Fix the issues
   - Run validation again
4. Only proceed when validation passes
```

参考文档也可以充当"验证器"——指示 agent 在定稿前对照参考检查自己的工作。

### 计划-验证-执行

对于批量或破坏性操作，让 agent 以结构化格式创建一份中间计划，对照事实来源进行验证，然后才执行。

```markdown
## PDF form filling

1. Extract form fields: `python scripts/analyze_form.py input.pdf` → `form_fields.json`
   (lists every field name, type, and whether it's required)
2. Create `field_values.json` mapping each field name to its intended value
3. Validate: `python scripts/validate_fields.py form_fields.json field_values.json`
   (checks that every field name exists in the form, types are compatible, and
   required fields aren't missing)
4. If validation fails, revise `field_values.json` and re-validate
5. Fill the form: `python scripts/fill_form.py input.pdf field_values.json output.pdf`
```

关键要素是第 3 步：一个验证脚本，对照事实来源（`form_fields.json`）检查计划（`field_values.json`）。像 "Field 'signature_date' not found — available fields: customer_name, order_total, signature_date_signed" 这样的错误，能给 agent 足够的信息来自行纠正。

### 打包可复用脚本

在[迭代技能](/skill-creation/evaluating-skills)时，比较 agent 在不同测试用例中的执行轨迹。如果你注意到 agent 每次运行都独立地重新发明同一套逻辑——构建图表、解析特定格式、验证输出——这就是一个信号：应当编写一个经过测试的脚本，并将其打包在 `scripts/` 中。

关于设计和打包脚本的更多内容，请参见[在技能中使用脚本](/skill-creation/using-scripts)。

## 后续步骤

一旦你有了可用的技能，有两份指南可以帮助你进一步打磨它：

- **[评估技能输出质量](/skill-creation/evaluating-skills)** —— 设置测试用例、为结果评分，并系统地迭代。
- **[优化技能描述](/skill-creation/optimizing-descriptions)** —— 测试并改进技能的 `description` 字段，使其在正确的提示词上触发。
