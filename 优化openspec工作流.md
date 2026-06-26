# TDD&SDD应用：AI开发工作流的融合与插件化

**做了什么：参考现有openspec框架，规范、强化ai开发工作流，作为cursor插件（在IDE中使用命令/opsx superapply）使用。**
项目地址：[choeyyy/opsx-superapply](https://github.com/choeyyy/opsx-superapply)

（**DD**:driven-development）

任务（**T**est）驱动开发和规范（**S**pec）驱动开发

先写测试，再写代码||先写规范再写代码。

这个例子也用作SDD、TDD开发范式的说明。

## 一、例子：superpowers&openspec

superpowers是agent+skill框架，重点在完成任务，包括设计和实践检查;
目的是让agent写高质代码；
*关键词：TDD、子agent编排、code review。*

openspec是设计驱动框架，重点在设置规范和设计任务清单、需求整理和持久化。
目的是根据需求设计编码规范，按照规范文档开发。面对每次开发需求都有

*关键词：SDD、需求规划、持久化。*

### 1.1 superpowers

#### 1.1.1 安装配置

[obra/superpowers: An agentic skills framework & software development methodology that works.](https://github.com/obra/superpowers)

它实际是一套给（Claude Code、Cursor、Codex、Gemini CLI 等） AI IDE软件用的软件开发插件。本质上是一组skills文件。安装之后会让agent在开发过程中遵循一套工作流：设计、编码、测试、子代理并行执行、代码审查。

安装方式根据使用的IDE软件有不同：

> 在 Cursor ：直接在 Agent 对话框输入 `/add-plugin superpowers` 或在插件市场搜索 "superpowers" 安装即可，无需手动配置。
>
> 在 Claude Code ：可以通过官方插件市场一行命令安装：
>
> /plugin install superpowers@claude-plugins-official
>
> 在 Gemini CLI ：
>
> gemini extensions install [https://github.com/obra/superpowers](https://github.com/obra/superpowers)
>
> 在 Codex CLI ：输入 `/plugins` 打开插件界面，搜索 superpowers 后选择 Install 即可。

安装之后插件会通过hooks机制在合适的实际自动触发对应的skill，不用手动调动。

> hook（钩子），插件在`hooks/session-start` 里放了一个 shell 脚本，每次ai会话启动这个脚本自动运行，把superpower的skill注入到ai上下文（在会话启动时注入一次而已）。

#### 1.1.2 使用时工作流

设计、编码、测试、子代理并行执行、代码审查。

详细的有七步，每一步

1. **头脑风暴**
    和ai讨论，ai探索项目现状后问开发者问题理解需求，提出分析和方案设计给开发者确认。
   **产生文件**：设计写入 `docs/superpowers/specs/` 下的 Markdown 文件并提交到 git，这是唯一的文字设计产物。
  > 从这一步研究**superpowers的使用场景**：
  >
  > ##### 适合从0到1建项目，还是改现有代码？
  >
  > 都可以。
  >
  > 头脑风暴Skill里有专门的说明：在进入现有代码库前，先探索当前的文件结构、文档和最近的提交，遵循已有模式，不要提出无关的重构。计划阶段也强调遵循现有代码的模式，不擅自改变结构。
  >
  > **从0到1建项目是Superpowers最适用的场景**：ai 可以完全按自己规划的架构来做。
  >
  > **关于改现有代码可能不够充分，需要提高上下文上限，对于需求也需要描述得更详细**：为了了解现有项目结构，AI 需要更多上下文，设计文档也需要描述清楚"改什么、改到什么程度、不改什么"，否则子代理容易越界改动无关代码。
  >
  > 头脑风暴Skill把大项目拆成子项目，一个子项目一个循环，但这个拆解过程本身就需要 AI 先理解整个代码库的结构。如果代码库大到超出上下文窗口，AI 在头脑风暴阶段就已经看不完全貌，拆解出来的子项目边界很可能不准确。
  >
  > 以及没有解决"持久化代码库知识"这个问题的机制，它的设计假设是代码库规模在一次上下文内可以基本理解。
2. **创建隔离工作区**
    开发者同意方案设计之后，ai在新的git分支上创建隔离工作目录，初始化。
3. **编写实现计划**
    ai把方案设计拆解为多个任务，每个任务包含：文件路径、完整代码、测试命令和预期输出，不会有模糊描述。
   完成后询问选择哪种执行方式。
   **产生文件**：带 checkbox 的任务清单写入 `docs/superpowers/plans/`，子代理执行时会勾选这些 checkbox。
4. **子agent开发**
    执行阶段。
   主调度ai agent对每个任务派出一个[开发子agent]，[开发子agent]完成之后再依次派发[规格审查子agent]和[代码审查子agent]进行两轮审查，直到审查通过，才进入下一步。
   两个审查子代理在每个子任务内部是串行的，而且不同子任务之间也是串行的。
   整个执行流程是：完成子任务1（实现→规格审查→质量审查，全部通过）→ 完成子任务2（同样三步）→ 以此类推。
   **产生文件**：代码
5. **测试**
    每个子agent实现任务时，先写一个注定失败的测试，运行确认它失败，再写最少的代码让它通过。运行确认通过之后提交。（如果没写测试代码，先写了开发相关的代码，则这段开发相关的代码会被删除）
  > “先写一个注定失败的测试，运行确认它失败，再写最少的代码让它通过” 这个叫RED-GREEN-REFACTOR 循环，这里体现测试驱动开发（test-driven-development） 。
  >
  > 大概就是：先写一个注定失败的测试（RED），运行并亲眼看到它失败，再写最少的代码让它通过（GREEN），最后清理重复和命名（REFACTOR），然后开始下一个测试。
  >
  > 没有先写失败的测试，就不允许存在任何生产代码。如果子代理在没有测试的情况下写了代码，要求把这段代码删掉，从测试开始重写。
  >
  > TDD的目的在于强制先定义行为再定义实现：先写测试意味着必须先想清楚这段代码对外应该表现什么，API 长什么样，输入输出是什么。会让设计更干净。
  >
  > 如果测试写起来很费劲，通常说明设计本身耦合太重。先看到测试失败这一步也很关键，它证明测试确实在检验需求中提到的功能，而不是测试了假命题。
  >  **产生文件**：测试代码
  >  （PS:第四第五步每完成一个任务就提交一次git commit，包含测试和实现两部分一起提交。）
6. **代码审查**
    子任务完成，ai对照计划逐条检查；
   全部任务完成，主 Agent自己做对照计划逐条检查。
7. **完成开发分支**

任务完成，ai选择清理第二步产生的隔离工作目录本身，合并、创建 PR、保留分支还是丢弃，然后整理工作目录。

#### 1.1.3 选择superpowers作为TDD例子的理由

​      superpowers的执行本身依照TDD规则，并且在skill文件中也强调了必须要先生成测试才能写功能代码，所有不测试的理由会被驳回。TDD执行力度大。

​     另一个原因是它的子agent编排架构让TDD的执行变得可观测：实现子agent必须按RED-GREEN-REFACTOR顺序工作，完成之后规格审查子agent和代码审查子agent会分别验证测试是否存在、是否覆盖了需求，形成三道检查。

如果实现子agent跳过了TDD，在审查阶段就会被拦住要求修复。agent分工设计让编码具有客观性，不受其他代码功能影响。

### 1.2 openspec

#### 1.2.1 安装配置

见：openspec npm安装
[OpenSpec — A lightweight spec‑driven framework](https://openspec.dev/#)

#### 1.2.2 使用时工作流

在这里按照openspec的命令使用顺序，扩展来说。

1. **探索 explore**
    探索模式，可以选择运行（如果方案/需求足够清晰，可以跳过这一步）
   主要是探索模糊想法/需求，最后不会生成文件或者变更，相当于比较规范化的聊天，如果聊出来的结果足够清晰，会建议用户下次直接输出/opsx-propose进行第二步。
2. **提案 propose**
    说明：运行之后，设计构思方案，形成设计的架构文件夹/change。
  > （ps：该文件夹下的子文件夹结构差不多每次都一样，可能根据需求有个数上的差别；每次propose就创建一次，生成方案规范任务文档，等任务实施结束，按需选择是否归档。归档的时候，整个change文件夹会加上时间，转移到/archive目录下。所以可以在/archive文件夹下看历史变更，也会有个文档统一介绍变更记录和结果。做到持久化）
  >  创建的文件有哪些：
  >  **.openspec.yaml（配置文件）**：记录日期，前期可以不用管，后期要对整个openspec工作模式进行修改，可在这改配置；
  >  **proposal.md（全局决策文件）**：why（为什么做）、what changes（变更有什么）、capabilities（如果要达成需求，要完成的功能们）、impact（列表：预计新增的文件、需要改的文件、要做持久化的文件、项目级文件。会做一个相对路径索引，也相当于初步构建文件架构了）；
  >  **design.md（约束决策风险文件）**：相当于给proposal文件提到的全局方案提出兜底。约束和决策让后续任务不偏离目标，风险让后续任务避坑。
  >  /spec/*(按照需求有不同子文件夹)/spec.md（每个需求对应一个仔细方案）（记录仔细的需求、方案、验收条件）：【按照需求有不同子文件夹】这里指按照proposal全局决策文件下的capabilities（功能们），每个功能创造一个子文件夹，子文件夹下一个spec文档（spec即方案、规划），记录要完成此功能需要做的详细方案；
  >  **tasks.md(任务清单)**：按照方案列出步骤，说明每步的任务具体内容、产出物、预期结果。
  >  有checkbox[]（每步做完的话:[]会变成[x]，标记做完了，才能进行下一步任务）
3. **执行 apply**
    说明：按照第二个，即propose阶段生成的任务tasks文档进行开发。
   有下面这些运行原则：
   逐条做tasks，做完才可做下一条；
   若任务不清楚，ai可询问开发者；
   实施时发现任务有问题，就暂停问开发者是否要修改，然后更新第二步生成过的约束文件和仔细方案文件，即design、spec文档；
   保持最小改动化，每步骤完成立即同步当前task进度。
   创建的文件有哪些：开发过程中的项目级文件。（不在openspec目录下）
   上述运行原则保证了开发任务执行不偏离。（Guardrails，护栏原则）
4. **归档 archive**
    说明：归档。就是记录、持久化当前需求对应的任务以及执行情况以及日期。
   可以选择是否归档、归档哪些内容，剩下的文件怎么处理。
   会把当前需求在propose阶段产生的change文件夹转移到archive目录下，加上日期前缀进行归档。
   主要步骤是：
   用户输入命令，选择需要归档的内容；
   ai检查完成状态：按照propose阶段生成的文件，检查propose、designs、spec、tasks，确认有没有偏离，有没有遗留任务等；
   继续查tasks，再次校验；
   评估归档的价值，告知用户，确认归档之后，转移整个change文件夹，加上日期。变更内容记录到全局
   创建的文件有哪些：无

#### 1.2.3 选择openspec作为SDD例子的理由

​    openspec的核心设计就是围绕spec（规格）构建的。整个开发循环___从explore到propose到apply到archive，每一步都以spec为基线，代码是spec的实现。这是SDD最直接的体现（规格先行，代码跟随）。

​    另一个原因是openspec解决了SDD在实际落地中最大的难题：规格的持久化和演进。openspec通过archive机制保留每次变更的完整上下文（proposal、design、spec、tasks），通过specs目录持续维护当前系统的行为契约，让AI在每次新会话中都能从一个准确的基线出发，而不需要重新阅读全部源码。

## 二、TDD&SDD融合

​    由第一章针对两个ai工具的分析，可看出openspec具有项目规范管理的优势：面对新旧项目的理解都比较深入，并且能持续记录代码迭代。在开发中可以负责探索规划以及持久化管理的部分。

   而superpower特点在于子agent编排，先测试再开发，最后是评审。优点在于多agent执行可以缩短开发时间，同时增强各个功能解耦性，各个功能代码独立不会受其他影响。同时测试和分阶段review也增加了代码的可信度。在测试时同时也能把需求再检查一次，保证后期开发时字段、方法的完整性。在开发中可以负责执行，即编码测试评审的部分。

  二者都是ai工具，实际编排上也比较像，都是skill+调用配置；同时都可以在各个软件用，并且使用率相对其他插件也比较高。所以考虑融合。融合的方式是在openspec的apply层基础上，加一句命令，要求ai以superpowers的方式执行子agent编排开发测试评审，并且最终对所有子任务进行合并评估。

这样能在保有openspec原有的功能基础上，让开发者多一个功能选择。

### 2.1 准备：梳理源码结构

在实践之前对superpowers、openspec的结构源码，以及目标平台（这里以cursor为例）的插件写法配置使用进行研究。

> 关于TDD和SDD的概念在开头说了：TDD先写测试，再写代码||SDD先写规范再写代码。
>
> 以及根据英文单词Test、Spec也比较好理解这两个开发方式的侧重点是什么，网上也有概念就不赘述了。

#### 2.1.1 superpowers 源码结构

路径：`https://github.com/obra/superpowers`

```
superpowers/
├── .cursor-plugin/
│   └── plugin.json              ← Cursor插件清单（名称、版本、入口路径）
├── .claude-plugin/
│   ├── plugin.json              ← Claude Code插件清单
│   └── marketplace.json         ← 市场上架信息
├── .codex-plugin/
│   └── plugin.json              ← Codex插件清单
├── hooks/
│   ├── hooks-cursor.json        ← Cursor读这个，声明sessionStart钩子
│   ├── hooks.json               ← Claude Code读这个，同样声明SessionStart
│   ├── run-hook.cmd             ← 跨平台适配脚本，找Git Bash来执行hook
│   └── session-start            ← 实际注入逻辑，读using-superpowers内容→JSON注入上下文
├── skills/                      ← 14个技能子目录，每个含一个SKILL.md
│   ├── brainstorming/           ← 头脑风暴（含visual-companion、scripts/server等）
│   ├── writing-plans/           ← 写实现计划
│   ├── subagent-driven-development/  ← 子agent编排（含三个prompt模板文件）
│   │   ├── SKILL.md
│   │   ├── implementer-prompt.md      ← 实现子agent的提示词模板
│   │   ├── spec-reviewer-prompt.md    ← 规格审查子agent模板
│   │   └── code-quality-reviewer-prompt.md  ← 质量审查子agent模板
│   ├── executing-plans/         ← 内联执行（不用子agent的替代方式）
│   ├── test-driven-development/ ← TDD纪律（含testing-anti-patterns.md）
│   ├── using-git-worktrees/     ← 隔离工作区
│   ├── requesting-code-review/  ← 代码审查
│   ├── finishing-a-development-branch/  ← 完成分支
│   ├── systematic-debugging/    ← 调试方法论
│   ├── verification-before-completion/  ← 完工验证
│   ├── using-superpowers/       ← 核心引导（sessionStart注入的就是这个）
│   ├── dispatching-parallel-agents/
│   ├── receiving-code-review/
│   └── writing-skills/          ← 写新skill的规范
├── package.json                 ← 只声明名称和版本，无npm依赖
├── CLAUDE.md / AGENTS.md / GEMINI.md  ← 各平台的贡献者指南
└── docs/ tests/ assets/ scripts/
```

整个仓库没有任何 npm 依赖，`package.json` 里只有名称和版本号。它本质就是一堆 Markdown 文件，通过各平台的 `.xxx-plugin/plugin.json` 告诉 IDE 在哪找 skills 和 hooks。运行时的核心机制是 `hooks/session-start` 这个 bash 脚本在会话启动时把 `using-superpowers/SKILL.md` 的全文以 JSON 格式注入到 AI 上下文，之后 AI 按需通过 Skill 工具加载其他技能。

其中，`.cursor-plugin/plugin.json` 的关键字段就三个：`"skills": "./skills/"` 指向技能目录，`"hooks": "./hooks/hooks-cursor.json"` 指向钩子配置，`"agents": "./agents/"` 指向代理定义（当前为空目录）。这就是 Cursor 插件的最小结构。

#### 2.1.2 openspec 源码结构

地址：[Fission-AI/OpenSpec: Spec-driven development (SDD) for AI coding assistants.](https://github.com/Fission-AI/OpenSpec)

另一种获取源码的情况：openspec官网推荐通过npm全局安装（`npm install -g @fission-ai/openspec`），此时源码不在项目目录里，而在 npm 的全局模块路径下：

`C:\Users\你的名字\AppData\Roaming\npm\node_modules\@fission-ai\openspec`

```
@fission-ai/openspec/                    ← npm全局安装位置
├── bin/
│   └── openspec.js                      ← CLI入口（命令行敲openspec时执行的）
├── dist/
│   ├── commands/                        ← CLI命令处理（change、config、validate等）
│   ├── core/
│   │   ├── command-generation/
│   │   │   └── adapters/                ← 25个IDE适配器（cursor.js、claude.js、codex.js…）
│   │   ├── shared/
│   │   │   └── skill-generation.js      ← 生成skill文件的逻辑
│   │   ├── templates/
│   │   │   └── workflows/               ← 内嵌的skill和command模板（JS，不是Markdown）
│   │   └── schemas/ parsers/ validation/
│   └── ui/ utils/
├── schemas/
│   └── spec-driven/
│       ├── schema.yaml
│       └── templates/                   ← proposal.md、design.md、spec.md、tasks.md模板
└── package.json                         ← @fission-ai/openspec v1.3.1
```

和 superpowers 有一个根本区别：openspec 的 skills 和 commands **不以 Markdown 文件的形式存在于 npm 包里**，而是以 JavaScript 代码的形式内嵌在 `dist/core/templates/workflows/` 和 `dist/core/shared/skill-generation.js` 中。当在项目里运行 `openspec init` 时，这些 JS 模板才会被渲染成 Markdown 文件，写入到项目的 `.cursor/skills/`（技能）和 `.cursor/commands/`（斜杠命令）目录下。

生成出来的技能有 12 个：`openspec-explore`、`openspec-propose`、`openspec-apply-change`、`openspec-new-change`、`openspec-continue-change`、`openspec-ff-change`、`openspec-sync-specs`、`openspec-archive-change`、`openspec-bulk-archive-change`、`openspec-verify-change`、`openspec-onboard`、`openspec-feedback`。对应的斜杠命令有 4 个（`/opsx-explore`、`/opsx-propose`、`/opsx-apply`、`/opsx-archive`），命令使用 `opsx-` 前缀，技能使用 `openspec-` 前缀，并非同名，且并非每个技能都有对应的斜杠命令。

#### 2.1.3 Cursor 插件写法

地址：[cursor/plugins: Cursor plugin specification and official plugins](https://github.com/cursor/plugins)

一个 Cursor 插件的最小结构如下：

```
插件名/
├── .cursor-plugin/
│   └── plugin.json          ← 必需：插件清单
├── skills/
│   └── 技能名/
│       └── SKILL.md         ← 技能文件
├── commands/                ← 可选：斜杠命令（/xxx触发）
│   └── 命令名.md
├── agents/                  ← 可选：代理定义
│   └── 代理名.md
├── hooks/                   ← 可选：钩子（sessionStart等事件触发）
│   └── hooks.json
└── README.md
```

`plugin.json` 是唯一必需的配置文件，关键字段是 `name`、`version`、`description`，然后通过 `"skills"`、`"commands"`、`"agents"`、`"hooks"` 四个路径字段指向对应目录。以 superpowers 为例，它的 `plugin.json` 就是：`"skills": "./skills/"`、`"hooks": "./hooks/hooks-cursor.json"`。以自建的 code-check 插件为例，它还多了 `"agents": "./agents/"`（10 个审查代理）和 `references/`（评审标准文档）。

从 cursor-plugins 仓库里的 16 个插件可以看到几种典型模式：纯 skill 型（cli-for-agent，只有一个 SKILL.md）；skill + hook 型（continual-learning、ralph-loop，通过 hook 在会话中自动触发行为）；skill + agent + command 完整型（code-check、orchestrate，有多个子代理和斜杠命令协同）；大型技能集型（pstack，40+个 skill 文件覆盖方方面面）。

superpowers 属于 skill + hook 型：14 个 skill 文件 + 1 个 sessionStart hook，没有斜杠命令，因为它的设计理念是"技能自动触发，不需要用户手动调用"。openspec 生成的则是 skill + command 型：每个工作流既有 skill 也有对应的斜杠命令（`/opsx-propose`、`/opsx-apply` 等），因为它需要用户主动选择进入哪个阶段。

### 2.2 融合：自定义opsx-superapply

#### 2.2.1 开发路径选择

已知openspec一条命令有对应的skill。如果要在本地openspec的基础上扩充功能，比如扩充一条命令，需要找到本地路径。在2.1.2 openspec 源码结构有提到大致的路径，在此路径新建文件夹：

 `C:\Users\你的名字\.cursor\skills\opsx-superapply\`

```
opsx-superapply/
├── SKILL.md                     ← 主编排器（146行）
├── implementer-prompt.md        ← 实现子agent提示词模板
├── spec-reviewer-prompt.md      ← 规格审查子agent模板
└── quality-reviewer-prompt.md   ← 代码质量审查子agent模板
```

它的执行流程是这样的：用户输入 `/opsx-superapply`（或在对话中触发），skill 先通过 `openspec status` 和 `openspec instructions apply` 两条 CLI 命令拿到当前 change 的状态和上下文（proposal、spec、design 这些 openspec 产出的文件），然后解析 tasks.md 里的任务列表，对每个未完成的任务走 superpowers 式的三步循环：派出实现子 agent → 派出规格审查子 agent → 派出质量审查子 agent，全部通过后把 tasks.md 里的 checkbox 勾上，继续下一个任务。如果超过 3 个任务，全部做完后还会对整个 git 范围做一次集成审查。

#### 2.2.2 参考和开发

**上游来自 openspec 的部分**：任务列表来自 openspec 的 `tasks.md`（SDD 的产物），实现子 agent 收到的上下文通过 `{PROPOSAL_EXCERPT}`、`{SPEC_CONTENT}`、`{DESIGN_CONSTRAINTS}` 三个占位符注入 openspec 的 proposal、spec、design 内容，审查子 agent 也是拿 openspec 的 spec 作为"对照什么审查"的基准。这样每个子 agent 都能看到规格约束，而不是凭空写代码。

**下游来自 superpowers 的部分**：三个 prompt 模板（implementer、spec-reviewer、quality-reviewer）的结构直接参考了 superpowers 的 `skills/subagent-driven-development/` 下的三个同名模板，包括四种状态码（DONE / DONE_WITH_CONCERNS / BLOCKED / NEEDS_CONTEXT）、自审检查清单、escalation 机制、fix-then-re-review 循环都保持一致。TDD 也写在实现子 agent 的模板里："Follow TDD: write failing test → run and confirm failure → write minimal code → run and confirm pass → commit"。

所以实际融合的方式是：openspec 管"对照什么开发"（spec 作为基线、tasks 作为清单、archive 做持久化），superpowers 管"怎么开发"（子 agent 隔离、TDD 纪律、两轮审查），opsx-superapply 是中间的编排层，把 openspec 的数据喂给 superpowers 式的执行引擎。开发者在使用时，前半段（explore → propose）走 openspec 原有流程生成规格和任务，到执行阶段选择 `/opsx-superapply` 替代原来的 `/opsx-apply`，就能获得子 agent 编排 + TDD + 两轮审查的执行质量，执行完之后再走 openspec 的 `/opsx-archive` 归档。

#### 2.2.3 /opsx-superapply源码

[choeyyy/opsx-superapply](https://github.com/choeyyy/opsx-superapply)

## 三、FAQ

#### **TDD&SDD？**

> TDD，测试驱动开发，英文全称Test-Driven Development，简称TDD，是一种不同于传统[软件开发流程](https://baike.baidu.com/item/软件开发流程/3430246?fromModule=lemma_inlink)的新型的开发方法。它要求在[编写](https://baike.baidu.com/item/编写/1517598?fromModule=lemma_inlink)某个[功能](https://baike.baidu.com/item/功能/10346898?fromModule=lemma_inlink)的[代码](https://baike.baidu.com/item/代码/86048?fromModule=lemma_inlink)之前先编写测试代码，然后只编写使测试通过的功能代码，通过测试来推动整个开发的进行。这有助于编写简洁可用和高质量的代码，并加速开发过程。（来自百度百科）
>
> SDD，规范驱动开发，英文全称Spec-Driven Development，简称SDD，是一种将规范而非代码作为真相来源的开发方法。它要求在编写功能代码之前先编写完整的需求规范，然后从规范生成实现代码，通过规范来推动整个开发的进行。维护软件即演进规范，调试即修正规范，生产环境的反馈持续写回规范再生成新代码，形成规范与实现之间的闭环迭代。（详细可见[spec-kit/spec-driven.md at main · github/spec-kit](https://github.com/github/spec-kit/blob/main/spec-driven.md)可以看到这里有spec-kit，这是一个和openspec有类似功能的工具，但本文选择了openspec，区别和选型后面有空说）

#### **skill网站？**

[搜索 170 万 Agent Skills | SkillsMP](https://skillsmp.com/zh/search)

#### opsx-superapply 和原来的 opsx-apply 区别？

 /opsx-apply 是单 agent 逐条执行任务，/opsx-superapply 是每个任务派独立子 agent + 两轮审查。前者简单快，后者质量高但耗 token 多。子agent审查很费token，每个任务至少三次子 agent 调用（实现+规格审查+质量审查），审查不通过还要循环。

#### 怎么写自己的 skill / agent 编排？有框架或语言要求吗？

没有编程语言要求。skill 本质就是一个 Markdown 文件（SKILL.md），里面用自然语言写 AI 应该遵循的流程、规则和输出格式，AI 读到这份文件之后就会按照里面的指令行事。

agent 编排也是在 Markdown 里写：在 skill 文件中描述"对每个任务派出一个子 agent 执行，完成后派另一个子 agent 审查"，AI 就会调用平台提供的 Task 工具去创建子 agent。所以写 skill 不需要会任何编程语言，只需要把流程写清楚。子 agent 的提示词模板也是 Markdown 文件，用 `{占位符}` 标记需要动态填入的内容（比如任务描述、项目路径），主编排 skill 在运行时把实际内容填进去再派发。

本文的做法就是先看目标 IDE 平台，再参照已有的 skill 格式（superpowers 的子 agent 编排、openspec 的工作流），新建文件夹写一套自己的 SKILL.md 和 prompt 模板。没有需要编译、安装依赖或注册的步骤，放对位置 AI 就能读到。

#### skill 和 agent 除了在 IDE 里用，还有什么应用方式？

IDE 插件只是一种载体。

同样的 skill 文件可以在 CLI 工具中使用——Claude Code、Codex CLI、Gemini CLI 都支持读取 skill 文件，用法和 IDE 里一样，只是界面从编辑器变成了终端。

还有一种方式是直接把 skill 内容当作 system prompt 喂给 API 调用，比如用 Claude API 或 OpenAI API 做自动化脚本时，把 SKILL.md 的内容拼到 system message 里，AI 的行为就会被这份 skill 约束。

CI/CD 流水线里也能这样用，比如在 GitHub Actions 里调 AI 做代码审查，把审查 skill 塞进 prompt 就行。本质上 skill 就是一段结构化的指令文本，能送到 AI 面前的地方就能用。

#### 提示词优化？

很重要，可以用来减少token、明确ai任务需求和边界等。

上上个问题提到skill和agent编排都是用自然语言写在markdown文件的，可以看成提示词。所以优化提示词其实也可以优化skill编写。

常用的优化方式：用英文省token；用 XML 标签把角色、任务、上下文、规则、输出格式分成独立区块，让 AI 不会把任务描述和执行规则混淆；拆分角色时每个子 agent 只看到自己需要的上下文，不继承主会话历史；用明确的门禁（"在 X 完成之前不允许做 Y"）代替建议性语言（"建议先做 X"）；还有很多。

可以参考claude封装skill插件的源码是怎么约束的。以及Anthropic 官方也有提供提示词工程交互式教程[anthropics/prompt-eng-interactive-tutorial](https://github.com/anthropics/prompt-eng-interactive-tutorial)（可以下载配置交互学习，也可以只看源码）。