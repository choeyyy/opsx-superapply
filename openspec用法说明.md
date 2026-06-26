# openspec

一个设计驱动开发框架
帮助开发者整理需求-设计方案-实施
**每次变更作为一次记录，可持久化到文件目录作为之后工作的上下文**
官方链接：[OpenSpec — A lightweight spec‑driven framework](https://openspec.dev/#)

## 安装

文末附SKILL.md

```
skill说明：
用于OpenSpec全局安装配置。
保证使用者在任何目录使用该skill之后
1.全局安装，在（默认codx）codx/cursor/其他ide平台，任何项目目录都可以使用
2.对于openspec的斜杠命令，安装结束之后给简单的例子+语句说明全流程（中文）
3.没有npm会帮忙下node
4.如果有下载过，检查openspec更新
```

## 基本工作流程

按照**语句实施顺序**说明openspec的工作流。IDE里有两套斜杠命令可用：`/openspec-`* 和 `/opsx-`*，内容一样，混用无影响（见文末说明）。

### 1./openspec-explore

```
说明：探索模式，可以选择运行（如果方案/需求足够清晰，可以跳过这一步）
主要是探索模糊想法/需求，只读探索不会写应用代码，但如果用户要求可以创建OpenSpec产物（proposals、designs、specs等思考记录）。相当于比较规范化的聊天，如果聊出来的结果足够清晰，会建议用户下次直接输出/opsx-propose进行第二步。
```

```
创建的文件有哪些：默认不创建文件；如果用户要求，可创建OpenSpec产物（proposal、design、spec等）。
格式：/openspec-explore 【模糊需求/想法等；可以不用打方框】
```

### 2./openspec-propose

```
说明：运行之后，设计构思方案，形成设计的架构文件夹/change。
（ps：该文件夹下的子文件夹结构差不多每次都一样，可能根据需求有个数上的差别；每次propose就创建一次，生成方案规范任务文档，等任务实施结束，按需选择是否归档。归档的时候，整个change文件夹会加上时间，转移到/archive目录下。所以可以在/archive文件夹下看历史变更，也会有个文档统一介绍变更记录和结果。做到持久化）
```

```
创建的文件有哪些：
.openspec.yaml（配置文件，记录方案模式schema和创建日期created，可以不用管）
proposal.md（全局决策文件）：why（为什么做）、what changes（变更有什么）、capabilities（如果要达成需求，要完成的功能们）、impact（列表：预计新增的文件、需要改的文件、要做持久化的文件、项目级文件。会做一个相对路径索引，也相当于初步构建文件架构了）
design.md（约束决策风险文件）：相当于给proposal文件提到的全局方案提出兜底。约束和决策让后续任务不偏离目标，风险让后续任务避坑。
/spec/*(按照需求有不同子文件夹)/spec.md（每个需求对应一个仔细方案）（记录仔细的需求、方案、验收条件）：【按照需求有不同子文件夹】这里指按照proposal全局决策文件下的capabilities（功能们），每个功能创造一个子文件夹，子文件夹下一个spec文档（spec即方案、规划），记录要完成此功能需要做的详细方案。
tasks.md(任务清单)：按照方案列出步骤，说明每步的任务具体内容、产出物、预期结果。
有checkbox[]（每步做完的话:[]会变成[x]，标记做完了，才能进行下一步任务）

格式：/openspec-propose 【清楚的需求/或者在完成第一步探索explore之后不输出括号内容也可以】
```

### 3./opsx-apply

```
说明：按照第二个，即propose阶段生成的任务tasks文档进行开发。
有下面这些运行原则：
逐条做tasks，做完才可做下一条；
若任务不清楚，ai可询问开发者；
实施时发现任务有问题，就暂停问开发者是否要修改，然后更新第二步生成过的约束文件和仔细方案文件，即design、spec文档；
保持最小改动化，每步骤完成立即同步当前task进度。
```

```
创建的文件有哪些：开发过程中的项目级文件。（不在openspec目录下）
格式：/openspec-apply-change
（一般就这样 除非有特殊说明也可以加在后面）
上述运行原则保证了开发任务执行不偏离。（Guardrails，护栏原则）
```

### 4./opsx-archive

```
说明：归档。就是记录、持久化当前需求对应的任务以及执行情况以及日期。
可以选择是否归档、归档哪些内容，剩下的文件怎么处理。
会把当前需求在propose阶段产生的change文件夹。
主要步骤是：
用户输入命令，选择需要归档的内容；
ai检查完成状态：按照propose阶段生成的文件，检查propose、designs、spec、tasks，确认有没有偏离，有没有遗留任务等；
继续查tasks，再次校验；
评估归档的价值，告知用户，确认归档之后，转移整个change文件夹，加上日期。变更内容记录到全局
```

```
创建的文件有哪些：无
格式：/openspec-archive-change
（一般就这样 除非有特殊说明也可以加在后面）
```

### 补充命令：/opsx-superapply

```
说明：子代理驱动的OpenSpec变更实现。与/opsx-apply功能类似，但采用多代理自主执行模式：
每个task由独立的子代理执行，执行后经过双阶段审查（先验证spec合规性，再检查代码质量），全部通过才标记完成。
适合任务较多、希望自主执行+质量把关的场景。
```

```
创建的文件有哪些：开发过程中的项目级文件。（不在openspec目录下）
格式：/opsx-superapply 【可选：指定change名称】
与/opsx-apply的区别：多代理并行执行+双阶段自动审查（spec合规→代码质量），无需逐步人工确认。
```

综上是常用的语句，也是使用openspec作为设计驱动开发框架的流程：propose、apply、archive。

---

## 补充：/openspec-* 和 /opsx-* 的关系

Cursor 有两种扩展机制：Skills（文件夹形式）和 Commands（单文件形式）。OpenSpec 安装时两种都生成了，所以 IDE 里出现两套命令：


| /openspec-*（Skill）       | /opsx-*（Command）   |
| ------------------------ | ------------------ |
| /openspec-propose        | /opsx-propose      |
| /openspec-apply-change   | /opsx-apply        |
| /openspec-explore        | /opsx-explore      |
| /openspec-archive-change | /opsx-archive      |
| /opsx-superapply         | /opsx-superapply   |


两套都装在全局目录（`~/.cursor/skills/` 和 `~/.cursor/commands/`），所有项目可用，内容一样，混用无影响。    

## 附件：[openspec安装/环境检查和配置/更新 ]SKILL

~~~markdown
---
name: openspec-setup
description: >-
  Globally install and configure OpenSpec CLI + Cursor skills so slash commands
  work in any project directory. Use when the user wants to install openspec,
  set up openspec, configure openspec globally, or enable openspec slash commands.

---

You are an environment setup agent. Install, configure, and verify OpenSpec CLI and its IDE skill files so that `/openspec-*` slash commands work in any project directory.

Install or update OpenSpec CLI globally and inject IDE skills so `/openspec-*` slash commands are available in every project.

## Prerequisites

- Cursor / Codex / Windsurf or another IDE that supports skills (at least one)
- Node.js >= 18 (bundled with npm) — Step 0 installs it automatically if missing

---

## Installation Procedure

Execute steps in order. Verify each step's result before continuing.

### Step 0 — Detect Node.js Environment (skip if already present)

Run `node -v`. If the command does not exist or the version is below 18, install per platform:

**Windows (winget, recommended):**

```powershell
winget install OpenJS.NodeJS.LTS
```

If winget is unavailable, fall back to fnm (Fast Node Manager):

```powershell
winget install Schniz.fnm
fnm install --lts
fnm use lts-latest
fnm default lts-latest
```

If winget is also unavailable, prompt the user to download manually: https://nodejs.org/

**macOS (Homebrew):**

```bash
brew install node@22
```

If Homebrew is not installed:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install node@22
```

**Linux (apt/dnf):**

```bash
# Debian/Ubuntu
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs

# Fedora/RHEL
curl -fsSL https://rpm.nodesource.com/setup_22.x | sudo bash -
sudo dnf install -y nodejs
```

Post-install verification:

```bash
node -v   # expect >= 18
npm -v    # expect >= 9
```

**Important**: On Windows, you may need to restart the terminal for PATH changes to take effect. If `node -v` still fails, prompt the user to close and reopen Cursor.

### Step 1 — Install or Update CLI

Detect current state and decide action:

```bash
openspec --version
```

- **Not installed** (command not found) — fresh install:

  ```bash
  npm install -g @fission-ai/openspec
  ```

- **Already installed** — check for newer version:

  ```bash
  npm outdated -g @fission-ai/openspec
  ```

  If output exists (update available), upgrade:

  ```bash
  npm update -g @fission-ai/openspec
  ```

  If no output, already up to date — skip.

Verify:

```bash
openspec --version
```

Expected: a version number (e.g. `1.4.1`). If it fails, check whether the npm global bin directory is in PATH (`npm bin -g`).

### Step 2 — Set Global Configuration

Ask the user to choose IDE platform(s) (affects `--tools` parameter). Default: Cursor + Codex.

```bash
openspec config set profile custom
openspec config set delivery both
```

`delivery` values:

- `skills` — generate skill files only
- `rules` — generate rule files only
- `both` — generate both (recommended)

### Step 3 — Initialize Instruction Files in Any Directory

Navigate to the user's project directory and run:

```bash
openspec init --tools {{IDE_TOOLS}}
```

`--tools` accepts comma-separated IDE names. See `openspec init --help` for supported values.
If the project was already initialized, use `openspec update --force` to refresh instruction files.

### Step 4 — Inject Global Skills (critical step)

After `openspec init`, skill files appear in the project's `.cursor/skills/` directory.
To make them available in **all projects**, copy them to the user-level skills directory.

**Windows:**

```powershell
$src = ".cursor\skills"
$dst = "$env:USERPROFILE\.cursor\skills"
Get-ChildItem $src -Directory | Where-Object { $_.Name -like "openspec-*" } | ForEach-Object {
    $target = Join-Path $dst $_.Name
    if (Test-Path $target) { Remove-Item $target -Recurse -Force }
    Copy-Item $_.FullName $target -Recurse
}
```

**macOS / Linux:**

```bash
src=".cursor/skills"
dst="$HOME/.cursor/skills"
mkdir -p "$dst"
for d in "$src"/openspec-*; do
    [ -d "$d" ] && cp -r "$d" "$dst/"
done
```

### Step 5 — Verify Installation

```bash
openspec --version
openspec config list
```

In the Cursor chat input, type `/openspec-` and confirm autocomplete suggestions appear:

- `/openspec-explore`
- `/openspec-propose`
- `/openspec-apply-change`
- `/openspec-archive-change`

---

## Execution Guide

When this skill is triggered, follow this procedure:

1. **Detect Node.js environment**
   - Run `node -v`
   - If the command does not exist or version < 18, execute Step 0 to install Node.js
   - After installation, verify again with `node -v` and `npm -v`

2. **Detect OpenSpec installation and updates**
   - Run `openspec --version`
   - If not installed, run `npm install -g @fission-ai/openspec`
   - If installed, run `npm outdated -g @fission-ai/openspec` to check for updates
   - If a newer version exists, run `npm update -g @fission-ai/openspec`; otherwise skip

3. **Confirm IDE platform**
   - Use AskQuestion to ask the user which IDE(s) to configure (options: Cursor, Codex, Cursor+Codex, Other)
   - Default recommendation: `cursor,codex`

4. **Execute Steps 2–4**
   - Run `openspec init --tools {{IDE_TOOLS}}` in the current working directory
   - Copy generated openspec skills to the global skills directory
   - Windows target: `%USERPROFILE%\.cursor\skills\`
   - macOS/Linux target: `~/.cursor/skills/`

5. **Execute Step 5 verification**
   - Print version and configuration info
   - List installed skill files

6. **Output usage instructions** (in Chinese), including the quick-start example below

---

## Post-Installation Output

After successful installation, display the following to the user:

```
 OpenSpec 全局配置完成！

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

 全流程示例（在任意项目目录）

① 初始化项目（首次使用该项目时）
   openspec init --tools cursor

② 探索需求（想清楚再动手）
   /openspec-explore
   → 对话式思考，画架构图，不写代码

③ 生成方案（一键产出设计文档）
   /openspec-propose
   → 自动创建 proposal.md + design.md + tasks.md

④ 逐步实现（按任务清单写代码）
   /openspec-apply-change
   → 逐个 task 实现，自动勾选进度

⑤ 归档变更（完成后收尾）
   /openspec-archive-change
   → 归档到 archive/，更新主规范

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

 常用 CLI 命令：
   openspec list          — 查看活跃变更
   openspec status        — 查看任务进度
   openspec show <name>   — 查看变更详情
   openspec validate      — 验证格式

 流程口诀：探索 → 提案 → 实现 → 归档
```

---

<example>
<input>
User: 帮我装一下 openspec
(Environment: Windows 11, Node.js not installed, Cursor IDE)
</input>
<output>
Detecting Node.js... not found. Installing via winget:

> winget install OpenJS.NodeJS.LTS
> Successfully installed OpenJS.NodeJS.LTS v22.x

> node -v → v22.15.0
> npm -v  → 10.9.2

Installing OpenSpec CLI:

> npm install -g @fission-ai/openspec
> openspec --version → 1.4.1

Configuring IDE tools: Cursor + Codex (default)

> openspec config set profile custom
> openspec config set delivery both

Initializing in current project:

> openspec init --tools cursor,codex

Copying skills to global directory:

> Copied 12 openspec-* skill folders to C:\Users\you\.cursor\skills\

Verification:

> openspec --version → 1.4.1
> /openspec- autocomplete: explore, propose, apply-change, archive-change ✓

 OpenSpec 全局配置完成！
（附使用说明...）
</output>
</example>

---

## Troubleshooting

| Symptom                       | Solution                                                     |
| ----------------------------- | ------------------------------------------------------------ |
| `node: command not found`     | Re-run Step 0; on Windows restart the terminal after installation |
| `npm: command not found`      | Node bundles npm — reinstall Node or check PATH              |
| `openspec: command not found` | Confirm npm global bin is in PATH: `npm bin -g`              |
| Slash commands do not appear  | Restart Cursor; confirm skills exist at `~/.cursor/skills/openspec-*` |
| `openspec init` errors        | Confirm Node >= 18: `node -v`                                |
| Outdated version              | `npm update -g @fission-ai/openspec`                         |

---

## Constraints

- Do NOT modify any existing project files outside `.cursor/` and `.openspec.yaml` during setup
- Do NOT run `openspec init` in the user's home directory — always confirm a valid project directory first
- Do NOT skip verification steps — every install/update action must be followed by a version check
- If any step fails and you cannot determine the cause, report the exact error output to the user and suggest manual intervention rather than guessing a fix

## Guardrails

- Report all CLI errors verbatim — do not summarize or paraphrase error messages; surface the exact output
- Do not fabricate version numbers, paths, or command outputs — only report what the commands actually return
- If `openspec init` or `npm install` fails with an unexpected error, halt and present the raw error to the user instead of retrying silently
~~~


