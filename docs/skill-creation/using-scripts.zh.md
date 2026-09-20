---
title: "在技能中使用脚本"
sidebarTitle: "使用脚本"
description: "如何在你的技能中运行命令并打包可执行脚本。"
---

技能可以指示 agent 运行 shell 命令，并在 `scripts/` 目录中打包可复用的脚本。本指南涵盖一次性命令、自带依赖的自包含脚本，以及如何为 agent 使用场景设计脚本接口。

## 一次性命令

当已有的包已经能做到你需要的事情时，你可以直接在 `SKILL.md` 的说明中引用它，而无需 `scripts/` 目录。许多生态系统都提供在运行时自动解析依赖的工具。

<Tabs sync={false}>
  <Tab title="uvx">
    [uvx](https://docs.astral.sh/uv/guides/tools/) 在隔离环境中运行 Python 包，并具有激进的缓存机制。它随 [uv](https://docs.astral.sh/uv/) 一起提供。

    ```bash
    uvx ruff@0.8.0 check .
    uvx black@24.10.0 .
    ```

    - 不随 Python 捆绑提供 —— 需要单独安装。
    - 快速。缓存激进，因此重复运行几乎瞬间完成。
  </Tab>
  <Tab title="pipx">
    [pipx](https://pipx.pypa.io/) 在隔离环境中运行 Python 包。可通过操作系统包管理器获取（`apt install pipx`、`brew install pipx`）。

    ```bash
    pipx run 'black==24.10.0' .
    pipx run 'ruff==0.8.0' check .
    ```

    - 不随 Python 捆绑提供 —— 需要单独安装。
    - `uvx` 的成熟替代方案。虽然 `uvx` 已成为标准推荐，但 `pipx` 仍是可靠的选择，并且在操作系统包管理器中的可用性更广。
  </Tab>
  <Tab title="npx">
    [npx](https://docs.npmjs.com/cli/commands/npx) 运行 npm 包，并按需下载它们。它随 npm 一起提供（npm 随 Node.js 一起提供）。

    ```bash
    npx eslint@9 --fix .
    npx create-vite@6 my-app
    ```

    - 随 Node.js 捆绑提供 —— 无需额外安装。
    - 下载包、运行它，并缓存以供将来使用。
    - 使用 `npx package@version` 固定版本以确保可复现性。
  </Tab>
  <Tab title="bunx">
    [bunx](https://bun.sh/docs/cli/bunx) 是 Bun 中与 `npx` 对应的工具。它随 [Bun](https://bun.sh/) 一起提供。

    ```bash
    bunx eslint@9 --fix .
    bunx create-vite@6 my-app
    ```

    - 在基于 Bun 的环境中可直接替代 `npx`。
    - 仅当用户环境使用 Bun 而非 Node.js 时才适用。
  </Tab>
  <Tab title="deno run">
    [deno run](https://docs.deno.com/runtime/reference/cli/run/) 直接从 URL 或说明符运行脚本。它随 [Deno](https://deno.com/) 一起提供。

    ```bash
    deno run npm:create-vite@6 my-app
    deno run --allow-read npm:eslint@9 -- --fix .
    ```

    - 访问文件系统/网络需要权限标志（`--allow-read` 等）。
    - 使用 `--` 分隔 Deno 标志与工具自身的标志。
  </Tab>
  <Tab title="go run">
    [go run](https://pkg.go.dev/cmd/go#hdr-Compile_and_run_Go_program) 直接编译并运行 Go 包。它内置于 `go` 命令中。

    ```bash
    go run golang.org/x/tools/cmd/goimports@v0.28.0 .
    go run github.com/golangci/golangci-lint/cmd/golangci-lint@v1.62.0 run
    ```

    - 内置于 Go —— 无需额外工具。
    - 固定版本或使用 `@latest`，让命令更明确。
  </Tab>
</Tabs>

**技能中一次性命令的提示：**

- **固定版本**（例如 `npx eslint@9.0.0`），使命令的行为随时间保持一致。
- **声明前置条件**，写在你的 `SKILL.md` 中（例如 "Requires Node.js 18+"），而不要假设 agent 的环境已具备它们。对于运行时级别的要求，请使用 [`compatibility` frontmatter 字段](/specification#compatibility-field)。
- **把复杂命令移入脚本。** 当你调用一个只带少量标志的工具时，一次性命令效果很好。当命令变得足够复杂、难以一次写对时，`scripts/` 中经过测试的脚本会更可靠。

## 从 `SKILL.md` 引用脚本

使用**相对于技能目录根目录的相对路径**来引用打包的文件。agent 会自动解析这些路径 —— 无需绝对路径。

在你的 `SKILL.md` 中列出可用的脚本，让 agent 知道它们存在：

```markdown SKILL.md
## Available scripts

- **`scripts/validate.sh`** — Validates configuration files
- **`scripts/process.py`** — Processes input data
```

然后指示 agent 运行它们：

````markdown SKILL.md
## Workflow

1. Run the validation script:
   ```bash
   bash scripts/validate.sh "$INPUT_FILE"
   ```

2. Process the results:
   ```bash
   python3 scripts/process.py --input results.json
   ```
````

<Note>
同样的相对路径约定也适用于 `references/*.md` 之类的支持文件 —— 脚本执行路径（在代码块中）相对于 **技能目录根目录**，因为 agent 从那里运行命令。
</Note>

## 自包含脚本

当你需要可复用的逻辑时，在 `scripts/` 中打包一个内联声明自身依赖的脚本。agent 可以用单条命令运行该脚本 —— 无需单独的清单文件或安装步骤。

有多种语言支持内联依赖声明：

<Tabs sync={false}>
  <Tab title="Python">
    [PEP 723](https://peps.python.org/pep-0723/) 定义了内联脚本元数据的标准格式。在 `# ///` 标记内的 TOML 块中声明依赖：

    ```python scripts/extract.py
    # /// script
    # dependencies = [
    #   "beautifulsoup4",
    # ]
    # ///

    from bs4 import BeautifulSoup

    html = '<html><body><h1>Welcome</h1><p class="info">This is a test.</p></body></html>'
    print(BeautifulSoup(html, "html.parser").select_one("p.info").get_text())
    ```

    使用 [uv](https://docs.astral.sh/uv/) 运行（推荐）：

    ```bash
    uv run scripts/extract.py
    ```

    `uv run` 会创建隔离环境、安装声明的依赖，然后运行脚本。[pipx](https://pipx.pypa.io/)（`pipx run scripts/extract.py`）也支持 PEP 723。

    - 使用 [PEP 508](https://peps.python.org/pep-0508/) 说明符固定版本：`"beautifulsoup4>=4.12,<5"`。
    - 使用 `requires-python` 约束 Python 版本。
    - 使用 `uv lock --script` 创建锁文件，以实现完全可复现性。
  </Tab>
  <Tab title="Deno">
    Deno 的 `npm:` 和 `jsr:` 导入说明符让每个脚本默认就是自包含的：

    ```typescript scripts/extract.ts
    #!/usr/bin/env -S deno run

    import * as cheerio from "npm:cheerio@1.0.0";

    const html = `<html><body><h1>Welcome</h1><p class="info">This is a test.</p></body></html>`;
    const $ = cheerio.load(html);
    console.log($("p.info").text());
    ```

    ```bash
    deno run scripts/extract.ts
    ```

    - npm 包使用 `npm:`，Deno 原生包使用 `jsr:`。
    - 版本说明符遵循 semver：`@1.0.0`（精确）、`@^1.0.0`（兼容）。
    - 依赖会被全局缓存。使用 `--reload` 强制重新获取。
    - 带原生插件（node-gyp）的包可能无法工作 —— 提供预编译二进制的包效果最好。
  </Tab>
  <Tab title="Bun">
    当找不到 `node_modules` 目录时，Bun 会在运行时自动安装缺失的包。直接在导入路径中固定版本：

    ```typescript scripts/extract.ts
    #!/usr/bin/env bun

    import * as cheerio from "cheerio@1.0.0";

    const html = `<html><body><h1>Welcome</h1><p class="info">This is a test.</p></body></html>`;
    const $ = cheerio.load(html);
    console.log($("p.info").text());
    ```

    ```bash
    bun run scripts/extract.ts
    ```

    - 无需 `package.json` 或 `node_modules`。TypeScript 可原生工作。
    - 包会被全局缓存。首次运行会下载；后续运行几乎瞬间完成。
    - 如果目录树的任意上层存在 `node_modules` 目录，自动安装会被禁用，Bun 会回退到标准的 Node.js 解析方式。
  </Tab>
  <Tab title="Ruby">
    自 Ruby 2.6 起，Bundler 随 Ruby 一起提供。使用 `bundler/inline` 直接在脚本中声明 gem：

    ```ruby scripts/extract.rb
    require 'bundler/inline'

    gemfile do
      source 'https://rubygems.org'
      gem 'nokogiri'
    end

    html = '<html><body><h1>Welcome</h1><p class="info">This is a test.</p></body></html>'
    doc = Nokogiri::HTML(html)
    puts doc.at_css('p.info').text
    ```

    ```bash
    ruby scripts/extract.rb
    ```

    - 显式固定版本（`gem 'nokogiri', '~> 1.16'`）—— 没有锁文件。
    - 工作目录中已存在的 `Gemfile` 或 `BUNDLE_GEMFILE` 环境变量可能会造成干扰。
  </Tab>
</Tabs>

## 为 agent 使用场景设计脚本

当 agent 运行你的脚本时，它会读取 stdout 和 stderr 来决定下一步做什么。一些设计选择能让脚本对 agent 来说易用得多。

### 避免交互式提示

这是 agent 执行环境的硬性要求。agent 在非交互式 shell 中运行 —— 它们无法响应 TTY 提示、密码对话框或确认菜单。因交互式输入而阻塞的脚本会无限期挂起。

通过命令行标志、环境变量或 stdin 接受所有输入：

```
# Bad: hangs waiting for input
$ python scripts/deploy.py
Target environment: _

# Good: clear error with guidance
$ python scripts/deploy.py
Error: --env is required. Options: development, staging, production.
Usage: python scripts/deploy.py --env staging --tag v1.2.3
```

### 用 `--help` 记录用法

`--help` 的输出是 agent 了解你脚本接口的主要方式。包含简要描述、可用标志和用法示例：

```
Usage: scripts/process.py [OPTIONS] INPUT_FILE

Process input data and produce a summary report.

Options:
  --format FORMAT    Output format: json, csv, table (default: json)
  --output FILE      Write output to FILE instead of stdout
  --verbose          Print progress to stderr

Examples:
  scripts/process.py data.csv
  scripts/process.py --format csv --output report.csv data.csv
```

保持简洁 —— 输出会与 agent 正在处理的其他所有内容一起进入它的上下文窗口。

### 编写有帮助的错误消息

当 agent 遇到错误时，错误消息会直接影响它的下一次尝试。一条含糊的 "Error: invalid input" 会浪费一轮。相反，应说明出了什么问题、期望什么，以及可以尝试什么：

```
Error: --format must be one of: json, csv, table.
       Received: "xml"
```

### 使用结构化输出

相比自由格式文本，优先使用结构化格式 —— JSON、CSV、TSV。结构化格式既能被 agent 消费，也能被标准工具（`jq`、`cut`、`awk`）消费，使你的脚本可在管道中组合使用。

```
# Whitespace-aligned — hard to parse programmatically
NAME          STATUS    CREATED
my-service    running   2025-01-15

# Delimited — unambiguous field boundaries
{"name": "my-service", "status": "running", "created": "2025-01-15"}
```

**将数据与诊断信息分离：** 把结构化数据发送到 stdout，把进度消息、警告和其他诊断信息发送到 stderr。这样 agent 既能捕获干净、可解析的输出，又能在需要时访问诊断信息。

### 其他注意事项

- **幂等性。** agent 可能会重试命令。"不存在则创建" 比 "创建并在重复时失败" 更安全。
- **输入约束。** 对含糊的输入给出明确的错误而不是猜测。尽可能使用枚举和封闭集合。
- **试运行支持。** 对于破坏性或带状态的操作，`--dry-run` 标志让 agent 预览将会发生什么。
- **有意义的退出码。** 为不同的失败类型（未找到、参数无效、认证失败）使用不同的退出码，并在 `--help` 输出中记录它们，让 agent 知道每个代码的含义。
- **安全的默认值。** 考虑破坏性操作是否应要求显式确认标志（`--confirm`、`--force`）或其他与风险级别相称的保护措施。
- **可预测的输出大小。** 许多 agent 框架会自动截断超过阈值（例如 10-30K 字符）的工具输出，可能丢失关键信息。如果你的脚本可能产生大量输出，默认返回摘要或合理的限制，并支持 `--offset` 之类的标志，让 agent 在需要时请求更多信息。或者，如果输出很大且不适合分页，则要求 agent 传入 `--output` 标志，指定输出文件或 `-` 来显式选择输出到 stdout。
