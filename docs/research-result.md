# Prompt

详细调研pi(earendil-works/pi)的社区生态：什么才是pi best practice？AI领域的大佬都是如何学习和使用Pi？作为一个codex重度使用者，有必要切换到Pi吗？Pi能给我（准大四，未来想做AI Agent领域工作）带来什么学习和使用价值？

# Result

**你没有必要把主力工具从 Codex 全量切换到 Pi。更合理的策略是：Codex 继续承担 70%–80% 的生产开发，Pi 承担 20%–30% 的 Agent 工程学习、模型横评、轨迹分析与自定义运行时实验。**

Pi 不是“更强的 Codex”，而是一个 agent harness（代理运行框架：负责模型调用、上下文管理、工具执行、会话状态和用户界面，而不是模型本身）。它的核心价值是让这些在 Codex 中主要由产品维护者决定的机制，变成你可以直接观察、修改和测量的代码。Pi 可以直接通过 `/login` 使用 ChatGPT Plus/Pro 的 Codex 订阅，因此你甚至不必先购买额外 API，就可以让同一类 Codex 模型分别运行在原生 Codex 与 Pi 两种 harness 中进行对照。([Pi][1])

截至 2026 年，Codex 本身也已经有 Skills、MCP（Model Context Protocol，模型上下文协议）、subagents（子代理）、Codex SDK（Software Development Kit，软件开发工具包）和操作系统级沙箱。因此，Pi 与 Codex 的真实差异已经不再是“可扩展与不可扩展”，而是：

> **Codex 让你在一个经过模型适配、带安全边界和完整产品体验的运行栈中扩展；Pi 让你直接改造运行栈本身。**

([OpenAI Developers][2])

对你而言，最准确的定位是：

* **Codex：生产工具**
* **Pi：Agent 工程实验室**
* **Pi 源码与公开轨迹：Agent 工程教材**

---

# 一、Pi 的社区生态究竟发展到了什么程度

## 1. Pi 不是单一 CLI，而是一组分层组件

Pi 官方代码库的核心可以分成四层：

| 层级       | 主要组件              | 你能学到什么                                        |
| -------- | ----------------- | --------------------------------------------- |
| 模型抽象层    | `pi-ai`           | 多供应商 API、流式输出、推理内容、工具调用、消息格式转换、成本统计           |
| Agent 循环 | `pi-agent-core`   | 模型—工具循环、工具参数校验、事件流、取消、steering、并发工具执行         |
| 界面层      | `pi-tui`          | TUI（Terminal User Interface，终端用户界面）、增量渲染、交互组件 |
| 成品 Agent | `pi-coding-agent` | 会话、上下文、压缩、Skills、Extensions、RPC、SDK、安全边界      |

`pi-agent-core` 已经把上下文转换、工具前后置回调、串行与并行工具执行、事件订阅、运行中转向和结束后 follow-up 暴露出来；代码库中更进一步的 durable harness（持久化代理运行框架）设计还讨论了操作状态机、意图—副作用—结算、崩溃恢复、重放和幂等性——幂等性指同一个操作被重试时不会重复造成副作用。需要注意，这是非常有价值的设计与源码学习材料，但不代表所有社区 `/goal` 扩展都自动拥有同等级别的持久化保证。

## 2. 社区已经很大，但成熟度分布极不均匀

本次抓取时，Pi 官方 package catalog（扩展包目录）已经收录 **5,374 个** npm 包。下载量靠前的项目包括 MCP 适配器、网页访问、subagent、上下文压缩、结构化提问、Todo、权限控制、Plan Mode、Goal 循环、LSP、内存和 Braintrust/Raindrop 可观测性接入。这个分布揭示了一个很有意思的矛盾：

* Pi 官方刻意不内置 MCP、subagent、Plan Mode、Todo、权限弹窗等功能；
* 社区最热门的工作之一，恰恰是把这些功能重新加回来。

([Pi][3])

因此，Pi 社区实际上存在三种文化：

1. **极简派**：遵循 Mario Zechner 的原始哲学，四个工具、简短 `AGENTS.md`、CLI、tmux、文件化状态，不装或少装扩展。
2. **个人工作台派**：像 Armin Ronacher 一样，针对自己的摩擦点，让 Pi 自己生成少量 Skills 和 Extensions，并不断删除无用部分。
3. **完整 harness 派**：一次性加入 MCP、subagents、memory、planning、permissions、goal loop、worktree orchestration、dashboard，构造类似 Claude Code/Codex 的大型工作流。

第三种最容易在演示中显得强大，也最容易产生依赖冲突、上下文污染、行为不可解释和供应链风险。**目录收录、月下载量和 GitHub Star 都不等于维护者审核或适合你的工作流。**Pi Extensions 是具有当前用户完整系统权限的 TypeScript 模块，官方明确要求只使用可信代码、审计源代码并考虑固定版本。([Pi][4])

## 3. Pi 生态最独特的资产不是扩展，而是真实 Agent 轨迹

Pi 维护者持续公开经过脱敏的真实开发会话。其 JSONL（JSON Lines，逐行 JSON）轨迹不仅包括用户消息和最终回答，还包括：

* reasoning、工具调用与工具结果；
* 模型和推理档位切换；
* context compaction（上下文压缩）；
* session tree 分支；
* branch summary；
* 自定义 Extension 数据。

这些轨迹比“最佳提示词合集”更接近真实 Agent 工程，因为你能观察模型漏读了哪些文件、如何错误归因、何时发生无效探索、工具参数为何失败、压缩后丢了什么、最终如何被人类纠正。官方数据集目前约 225 MB，并明确提示脱敏只是 best-effort，仍需谨慎处理。([Hugging Face][5])

---

# 二、什么才是真正的 Pi best practice

Pi 没有一份公认的“黄金配置”。但从官方设计、Mario 与 Armin 的公开实践、成熟社区项目中，可以归纳出一套相当清晰的原则。

## 1. 先运行“裸 Pi”，不要先安装全家桶

正确的第一步是使用官方的 `--ignore-scripts` 安装方式，通过 `/login` 接入你的 ChatGPT Pro，在同一个熟悉仓库中运行 5–10 个任务，并且暂时不安装社区包：

```bash
npm install -g --ignore-scripts @earendil-works/pi-coding-agent
cd /path/to/project
pi
# 在 Pi 中执行 /login，选择 ChatGPT Plus/Pro (Codex)
```

Pi 默认只向模型提供 `read`、`write`、`edit`、`bash` 四个工具，另有可选的只读 `grep`、`find`、`ls`。先记录裸 Pi 在哪些地方令你不便，再针对真实摩擦补功能。否则你无法区分问题究竟来自模型、Pi 核心、某个扩展，还是扩展之间的交互。([Pi][6])

## 2. 把上下文视为架构资源，而不是“能塞多少塞多少”

推荐的上下文层级是：

**稳定规则写进 `AGENTS.md` → 可复用流程写成 Skill → 大量参考资料按需读取 → 临时任务信息留在当前会话。**

`AGENTS.md` 应主要包含：

* 构建、测试、格式化和验收命令；
* 关键架构约束；
* 禁止修改或高风险路径；
* 提交与验证规则；
* 项目内稳定且长期有效的信息。

不要把完整需求历史、几万字设计文档和所有可能用到的知识都塞进 `AGENTS.md`。Skills 使用 progressive disclosure（渐进披露：模型先看到名称和简短描述，需要时才读取完整说明），Pi 与 Codex 都支持这一思路。([Pi][6])

另外，Pi 中 `!command` 会把命令输出送进模型上下文，而 `!!command` 只执行、不把输出加入上下文。长测试日志、安装输出和你自己查看的信息应优先用后者，避免无意义地消耗上下文。([Pi][6])

## 3. 长期状态必须外置，不能只存在于对话里

适合无人值守和跨会话任务的状态，应写入仓库中的可检查文件，例如：

* `PLAN.md`：目标、当前阶段、下一步；
* `TASKS.md`：可验证任务列表；
* `DECISIONS.md`：架构决策及理由；
* `LEARNINGS.md`：失败尝试和排除项；
* `measure.sh`：可重复执行的指标；
* `checks.sh`：测试、类型检查和 lint；
* `runs.jsonl`：每次尝试、结果和指标的追加日志。

Mario 明确主张用文件代替内置 Plan/Todo；Tobi Lütke 与 David Cortés 的 `pi-autoresearch` 则把这一原则推进到无人值守实验循环：修改、测量、记录、保留或回滚，每个结果追加到 JSONL，长期状态不依赖对话记忆。([Mario Zechner][7])

可以把这一原则概括成：

> **对话是工作内存，仓库文件才是持久状态；compaction summary 是缓存，不是事实源。**

Pi 的自动压缩会保留结构化摘要、最近消息以及累计读写文件列表，机制比普通“总结一下前文”严谨得多，但摘要仍然是模型生成的有损表示。关键要求、当前进度和验证结果仍应落盘。

## 4. 调研、实现和审查尽量使用不同上下文

Mario 对 subagent 的批评并不是“永远不要并行”，而是反对把大量不可观察的探索塞进黑箱子代理。他推荐的典型流程是：

1. 在独立 session 中完成代码库调查或技术调研；
2. 输出一个可检查的 `RESEARCH.md`、`DESIGN.md` 或任务规格；
3. 在新 session 中仅携带该产物进行实现；
4. 在独立 review branch/session 中审查；
5. 把有价值的结论带回主线。

Pi 的会话本身是树结构，`/tree` 可以从早期节点分叉，适合探索替代方案或临时修复 Agent 自己的扩展；`/fork` 和 `/clone` 则创建独立会话。

Armin 的 `/review` 就是利用 session tree，在干净分支中审查当前改动，再把修复带回主线；他也用分支处理“修理 Agent 工具”这种 side quest，避免污染主任务上下文。([Armin Ronacher's Thoughts and Writings][8])

## 5. 按最低必要能力逐级扩展

一个很实用的选择顺序是：

> **Prompt Template → Skill → 普通 CLI → Extension → Pi Package**

含义如下：

* 只是重复一段提示骨架：用 Prompt Template；
* 是一套按需使用的知识或工作流程：用 Skill；
* 是确定性操作、已有成熟命令行工具：直接让 Bash 调 CLI，并提供 README；
* 需要拦截工具调用、注入上下文、保存 session 状态、修改压缩逻辑或增加 TUI：才写 Extension；
* 需要跨项目或向他人分发：最后再打成 Package。

Pi 的 Extension 可以监听生命周期事件、阻止或修改工具调用、动态增减工具、注入上下文、自定义 compaction、保存持久状态和绘制完整 TUI。能力很强，也意味着不应把每一个小需求都升级为模型可见的新工具。([Pi][4])

## 6. 工具集合保持小而稳定

模型看到的工具越多，通常越需要：

* 理解更多 schema（参数结构）；
* 在相似工具间做选择；
* 消耗更多固定上下文；
* 面临更多版本漂移和错误调用。

Mario 因此倾向于 CLI + README，而不是一次性注入几十个 MCP 工具。社区则提供了 `pi-mcp-adapter`，说明 MCP 在一些场景依然有实际需求。正确结论不是“MCP 一律不好”，而是：

* 高频、交互式、需要结构化返回的能力，可以做原生工具或 MCP；
* 低频、确定性、已有优秀 CLI 的能力，优先 CLI + Skill；
* 大型 MCP Server 应考虑工具搜索、动态加载或 CLI 包装，避免全部工具常驻上下文。

([Mario Zechner][7])

## 7. Subagent 先用于只读和低耦合任务

适合并行委派的任务包括：

* 代码库探索；
* 日志分析；
* 测试执行；
* 独立方案比较；
* 代码审查；
* 文档与资料摘要。

并行修改同一代码区、架构尚未稳定的功能开发和存在大量共享状态的任务，往往会增加冲突与协调成本。Codex 官方当前给出的建议也基本一致：先把 subagent 用在 read-heavy（读取为主）的工作，对 write-heavy（写入为主）的并行任务更加谨慎。([OpenAI Developers][9])

对于你正在使用的长时间 `/goal` 工作流，Pi 版本不能只是“不断给自己发继续”。至少要具备：

* 持久任务队列；
* 完成标准和反向验证；
* 每步 checkpoint；
* 测试或指标形成 backpressure（反向约束）；
* 重启后从磁盘恢复；
* 反复失败时停止或换策略；
* 修改范围、预算和最大迭代限制。

`pi-autoresearch` 是较成熟的专项案例；社区中的通用 `/goal`、forever-loop 和多代理包则应逐一审计，不应默认它们具有相同可靠性。([Pi][3])

## 8. 无人值守 Pi 必须运行在真实隔离边界内

Pi 的 project trust（项目可信确认）只决定是否自动加载项目内扩展，不会限制模型之后执行什么命令。Pi 默认没有内置沙箱，Extension 也拥有完整用户权限。因此：

* 可信个人仓库的交互式任务：至少保持 Git clean/checkpoint；
* 长时间无人值守：容器、虚拟机或 Gondolin 微型虚拟机；
* 不可信仓库或外部网页内容：去除宿主机密钥，限制网络和可访问目录；
* API Token 只提供最小权限；
* 完成后人工检查 diff、依赖变化和测试证据。

官方提供了 Gondolin、Docker 和 OpenShell 等隔离路径。相比之下，Codex 本地默认采用操作系统强制沙箱，网络默认关闭，写入通常限于当前 workspace；Codex cloud 则运行在隔离容器中。  ([ChatGPT Learn][10])

---

# 三、公开证据中，资深开发者实际怎样使用 Pi

这里应区分“本人一手公开说明”和“他人转述”。目前没有证据支持“AI 圈大佬都在用 Pi”，但有几类非常有代表性的真实实践。

| 人物                                    | 公开使用方式                                                                                               | 真正值得学习的部分                       |
| ------------------------------------- | ---------------------------------------------------------------------------------------------------- | ------------------------------- |
| **Mario Zechner**，Pi 作者               | 因不满其他 harness 功能和隐藏上下文不断膨胀而创建 Pi；强调四工具、简短系统提示、可观察会话、文件化 Plan/Todo、tmux 和独立调研 session                 | 先控制上下文和可观察性，而不是堆功能              |
| **Armin Ronacher**，Flask/Jinja 作者     | 公开表示几乎只用 Pi；让 Pi 自己写 Extension；只保留一个额外模型工具，其他主要是 Skills/TUI；用 session branch 做 review；不断删除不用的 Skill  | 把 Pi 训练成个人工具，而不是下载一套“万能配置”      |
| **Tobias Lütke + David Cortés**       | 用 Pi 实现 autoresearch 循环，在 Shopify Liquid 中运行约 120 次自动实验、形成 93 个提交；依靠 974 个测试、benchmark 和 keep/revert | 自主 Agent 的基础不是长提示词，而是可测量目标和强验证  |
| **Georgi Gerganov**，llama.cpp/ggml 作者 | 据其公开评论，使用裁剪后的 `pi -nc --offline` 和简短风格提示，配合本地 Qwen 模型处理日常维护小任务                                       | Pi 可作为本地模型的极薄测试壳，不必建设复杂 harness |
| **Peter Steinberger / OpenClaw**      | Armin 记录称 OpenClaw 底层使用 Pi 组件；Pi 也被用于 Telegram bot 等非 CLI 应用                                         | Pi 的组件价值可能高于 `pi` 命令本身          |
| **Simon Willison**                    | 主要是跟踪、验证和整理 Pi 案例，没有充分证据表明他把 Pi 作为日常主力                                                               | 他的价值在于独立分析实践结果和模型—工具适配问题        |

Mario 的直接经验显示，他追求的是精确控制上下文和完整可观察性，而非功能数量。([Mario Zechner][7])

Armin 的配置则说明，成熟用法并不是复制某个大包：他的多数功能由 Pi 根据个人规格生成，包括 `/answer`、Todo、review branch、changed files、通知、`uv` 策略等；他明确表示会丢弃不再需要的 Skills。([Armin Ronacher's Thoughts and Writings][8])

Tobi 的案例尤其重要。最终性能提升本身不是重点；重点是“974 个测试 + 明确 benchmark + 单变量实验 + 追加日志 + 自动回滚”把含糊的“让它更快”变成了 Agent 可以长期优化的闭环。([Simon Willison’s Weblog][11])

Georgi 的案例代表另一端：不是最大化 Agent 能力，而是用极薄 Pi harness 驱动本地模型完成范围明确的维护任务。该信息来自 Simon 对 Georgi 公开评论的转述，不是一篇完整的一手工作流文章。([Simon Willison’s Weblog][12])

还需澄清：`pi-autoresearch` 受到 Andrej Karpathy 的 autoresearch 思路启发，但目前没有充分公开证据说明 Karpathy 本人使用 Pi。不能从“受其项目启发”推导为“Karpathy 也在用 Pi”。

---

# 四、作为 Codex 重度用户，Pi 能否替代 Codex

## 1. 当前已经不能用“Codex 封闭、Pi 开放”简单概括

2026 年的 Codex 已经支持：

* Skills 和渐进披露；
* MCP；
* 可查看线程的 subagent；
* TypeScript/Python Codex SDK；
* 本地、IDE 和云端运行；
* workspace 沙箱、网络控制和审批策略。

([OpenAI Developers][2])

因此，两者更准确的对比如下：

| 维度          | Codex                        | Pi                                   | 对你的判断                     |
| ----------- | ---------------------------- | ------------------------------------ | ------------------------- |
| 开箱生产力       | 完整产品、IDE/App/CLI/Cloud、审查和沙箱 | 基础能力够用，其他由你配置                        | **Codex 胜**               |
| 模型适配        | OpenAI 模型与原生工具链协同            | 多供应商统一抽象，但存在 schema 适配差异             | **Codex 模型用原生 Codex 更稳妥** |
| 多模型与本地模型    | 主要围绕 OpenAI                  | 15+ provider、OpenRouter、Ollama、自定义端点 | **Pi 胜**                  |
| 上下文可控性      | 可配置，但产品仍决定大量底层行为             | 系统提示、工具、压缩、消息转换均可改                   | **Pi 胜**                  |
| 会话可检查性      | 产品化记录与线程                     | 明确 JSONL 树、Extension 数据、可自定义解析       | **Pi 更适合研究**              |
| 安全默认值       | OS 沙箱、网络默认关闭、审批              | 默认完整用户权限，需要外部隔离                      | **Codex 明显胜**             |
| 长时间无人值守     | 原生目标、subagent、云端和安全边界        | 依赖 Extension、外部状态和隔离设计               | **Codex 生产使用更省心**         |
| 自定义运行时      | Skills、MCP、SDK、配置和 hooks     | 可直接修改 Agent loop 周边一切                | **Pi 更底层**                |
| 维护成本        | OpenAI 维护                    | 你要维护自己的包、兼容性和安全                      | **Codex 胜**               |
| 构建非编码 Agent | 可通过 SDK/Agents SDK 构建        | provider-neutral，组件可拆装、RPC/SDK 简单    | 取决于产品需求                   |

## 2. 原生模型—harness 适配是 Pi 的真实风险

Armin 曾观察到，更新的 Claude 模型会向 Pi 的 edit 工具提交不存在的字段，导致原本内容正确的编辑被 schema 校验拒绝。他推测这些模型可能越来越适配 Claude Code 的原生工具格式。这还只是一个有依据的解释假说，而不是完整因果证明，但它说明了一个普遍问题：

> **更强的模型不一定在任意第三方工具 schema 上表现更好。**

Codex 模型同样可能更熟悉原生 Codex 的 `apply_patch` 工作方式。OpenAI 也提供了专门的结构化 `apply_patch` 工具。因此，在 Pi 中运行同一个 Codex 模型，不应默认能复现原生 Codex 的工具可靠性；这恰恰应该通过 A/B 任务测量。([Simon Willison’s Weblog][13])

## 3. 三种“切换”应分别回答

**切换日常编码客户端：不建议。**你已经熟悉 Codex，且经常使用长任务、subagent 和 `/goal`。全量迁移会牺牲原生沙箱、工具适配和产品集成，还会增加大量配置维护。

**切换模型供应商：不需要。**Pi 可以通过你的 ChatGPT Pro 登录使用 Codex，也可以在需要时切换 OpenRouter、本地模型或其他供应商。Pi 的价值之一就是把“换 harness”和“换模型”解耦。([Pi][6])

**增加一个 Agent 工程学习环境：非常值得。**这才是你应该进行的切换——从“只会高强度使用 Codex”，升级为“能够比较和修改两种 Agent runtime”。

---

# 五、Pi 能给你的职业学习带来什么

你当前最需要的不是再学一个命令行客户端，而是形成可展示的 **Agent 工程证据**。仅在简历上写“熟练使用 Pi/Codex”价值有限；能够解释、实现和评测下面这些机制，价值明显更高。

## 1. Agent loop 与工具执行

通过 `pi-agent-core` 可以直接学习：

* 模型输出如何转成工具调用；
* 参数 schema 如何验证；
* 工具失败如何反馈；
* 何时并行、何时串行；
* `AbortSignal` 如何取消；
* steering 如何在当前工具结束后改变后续轨迹；
* follow-up 如何排队；
* 事件如何供 UI、日志和评测系统订阅。

这比单纯调用一个 Agent SDK 更接近运行时工程。

## 2. 多供应商抽象与工具兼容

`pi-ai` 要处理 OpenAI、Anthropic、Google、OpenAI-compatible 服务以及本地推理引擎之间的差异。你可以学习：

* 不同消息角色和 reasoning 字段转换；
* 流式事件归一化；
* tool call ID 和结果配对；
* 模型切换后的上下文兼容；
* token、cost 和缓存统计；
* provider-specific capability negotiation（供应商能力协商）。

这是构建跨模型 Agent 平台时极常见、又很容易被上层框架遮蔽的工作。([Mario Zechner][7])

## 3. 会话、压缩、恢复与持久化

Pi 的 session tree、JSONL、compaction 和 Extension state 很适合研究：

* 长上下文为什么逐步退化；
* 摘要后哪些约束容易丢失；
* 如何把关键状态移出对话；
* 如何恢复中断任务；
* 如何做 replay（重放）；
* 如何防止重复副作用；
* 如何在 branch 间传递最小上下文。

这些问题与真正的长时间 Agent 产品比“提示词怎么写”更接近。

## 4. 可观测性、评测与故障分析

你最近关注 Coding Agent 的完整轨迹记录、能力评测和可靠性分析，Pi 恰好是很适合做实验的平台，因为其会话和事件格式比很多封闭产品更容易解析。

你可以建立 failure taxonomy（失败分类体系），例如：

* 上下文遗漏；
* 工具选择错误；
* 工具 schema 不匹配；
* 编辑冲突或旧文本不匹配；
* 过早宣称完成；
* 未运行验证；
* compaction 后决策漂移；
* subagent 结论未经主 Agent 验证；
* 命令超时、网络失败；
* 重启后重复执行副作用。

这类项目与 Agent Evaluation、Agent Reliability、Applied AI Engineering 岗位直接相关。

## 5. 安全、权限与供应链

Pi 默认缺少沙箱反而形成了一个很好的学习对象：你必须自己思考能力边界，而不能依赖产品预设。可以实践：

* 工具调用前策略检查；
* 受保护路径；
* 网络 allowlist；
* 临时凭证；
* 容器隔离；
* Extension 供应链审计；
* prompt injection 后果；
* 人工审批与自动运行之间的权衡。

但学习价值不等于应在宿主系统上冒险。实验环境仍应隔离。

## 6. 构建自己的 Agent 产品

Pi 有 Interactive、JSON Event Stream、RPC 和 SDK 四种使用方式，可以嵌入其他应用。对你的桌宠、Coding Agent 评测、国际象棋讲解 Agent 等想法，Pi 最合适的角色通常不是“整个产品”，而是：

* 一个可替换的 Agent runtime；
* 一个模型供应商适配层；
* 一个轨迹与状态实验环境；
* 一个自定义工具/策略的快速原型宿主。

Codex SDK 现在也能启动、继续和恢复本地 Codex thread，因此“可嵌入”并非 Pi 独有；Pi 的差异在于更底层、更 provider-neutral，并允许你改变会话和工具语义。([Pi][14])

---

# 六、最适合你的 Pi 项目，不是再造一个 Coding Agent

最值得做的是一个：

## Cross-Harness Agent Reliability Lab

**跨运行框架 Agent 可靠性实验平台**

核心设计如下：

1. 使用 **Codex SDK adapter** 和 **Pi SDK/RPC adapter** 运行同一批任务。
2. 尽量固定同一模型、推理档位、代码版本、`AGENTS.md` 和验收标准。
3. 定义统一 trajectory schema，归一化记录消息、工具调用、耗时、错误、重试、文件修改、测试结果和人工干预。
4. 从你现有项目中提取 15–20 个可自动验收的真实任务，覆盖 bug fix、测试补充、重构、代码审查、仓库理解和长任务。
5. 做 fault injection（故障注入：主动制造受控故障），例如命令超时、无网络、工具返回 malformed JSON、编辑冲突、上下文压缩、进程崩溃和恢复。
6. 比较：

   * 完成率；
   * 测试通过率与回归；
   * 人工 steering 次数；
   * 工具调用失败率；
   * 过早完成率；
   * token、时间和成本；
   * 崩溃恢复正确性；
   * 安全策略违规次数。
7. 编写一个小型 Pi Extension，负责 telemetry、受保护路径和 post-edit verification，而不是安装整个社区 harness。
8. 在容器中运行，输出可复现实验报告和失败案例分析。

这个项目的简历价值远高于“用 Pi 做了一个聊天机器人”，因为它能证明你理解 Agent 的运行时、评测、可靠性、安全和工程实验方法。也能与你现有的 Coding Agent 轨迹评测兴趣、AI Agent 求职方向和 Codex 使用经验连起来。

---

# 七、建议的 14 天验证路线

| 时间        | 工作                                            | 产物                      |
| --------- | --------------------------------------------- | ----------------------- |
| 第 1–2 天   | 裸 Pi，使用 ChatGPT Pro/Codex 登录，在熟悉仓库运行 5–10 个任务 | Pi 与 Codex 的初步差异记录      |
| 第 3–4 天   | 阅读 `pi-ai`、`pi-agent-core` 和基础 Extension 示例   | Agent loop 架构图          |
| 第 5–6 天   | 手写一个最小事件记录 Extension                          | `trace.jsonl`、工具耗时和错误日志 |
| 第 7–9 天   | 固定 10–15 个任务，对 Codex 与 Pi 做尽量同模型 A/B          | 指标表与失败样本                |
| 第 10–11 天 | 加入超时、命令失败、压缩和重启等故障注入                          | 恢复能力报告                  |
| 第 12 天    | 放入 Docker/Gondolin，限制密钥和网络                    | 威胁模型与运行说明               |
| 第 13–14 天 | 整理 README、架构图、实验方法和结论                         | 可作为简历项目的仓库              |

在这 14 天内，不建议安装大型 multi-agent harness、复杂 memory 系统或几十个 Skills。先证明你能解释裸 Pi 的行为，再决定缺什么。

---

# 最终决策标准

你的默认配置应当是：

> **Codex：主力开发与长时间交付。
> Pi：模型横评、Agent runtime 学习、自定义工具和可靠性实验。**

只有在以下条件同时大体成立时，才值得考虑让 Pi 成为主力：

1. 你确实经常需要多供应商、本地模型、自定义压缩、工具策略或嵌入式 Agent；
2. 你自己的同任务 A/B 结果显示，Pi 在至少两个重要指标上持续优于 Codex；
3. 维护 Pi 配置的时间可控；
4. 无人值守任务已经有容器、持久状态、验证和恢复机制；
5. 你能清楚解释每个已安装 Extension 为什么存在。

可以把“两个关键指标改善约 15%，每周 harness 维护少于 1 小时”作为个人工程上的粗略门槛，但这只是决策启发式，不是通行研究标准。

**最不建议的路线**是停止使用已经成熟的 Codex，花数周把 Pi 配成另一个 Codex。
**最有价值的路线**是保留 Codex 的生产效率，利用 Pi 把你从“熟练调用 Agent 的用户”推进为“理解、修改、评测和保障 Agent runtime 的工程人员”。

[1]: https://pi.dev/?utm_source=chatgpt.com "Pi Coding Agent"
[2]: https://developers.openai.com/codex/skills "https://developers.openai.com/codex/skills"
[3]: https://pi.dev/packages "https://pi.dev/packages"
[4]: https://pi.dev/docs/latest/extensions "Extensions · Documentation · Pi"
[5]: https://huggingface.co/datasets/badlogicgames/pi-mono "https://huggingface.co/datasets/badlogicgames/pi-mono"
[6]: https://pi.dev/docs/latest/quickstart "Quickstart · Documentation · Pi"
[7]: https://mariozechner.at/posts/2025-11-30-pi-coding-agent/ "https://mariozechner.at/posts/2025-11-30-pi-coding-agent/"
[8]: https://lucumr.pocoo.org/2026/1/31/pi/ "https://lucumr.pocoo.org/2026/1/31/pi/"
[9]: https://developers.openai.com/codex/subagents "https://developers.openai.com/codex/subagents"
[10]: https://learn.chatgpt.com/codex/agent-approvals-security "https://learn.chatgpt.com/codex/agent-approvals-security"
[11]: https://simonwillison.net/2026/Mar/13/liquid/ "https://simonwillison.net/2026/Mar/13/liquid/"
[12]: https://simonwillison.net/2026/Jun/16/georgi-gerganov/ "https://simonwillison.net/2026/Jun/16/georgi-gerganov/"
[13]: https://feeds.simonwillison.net/2026/Jul/4/better-models-worse-tools/ "Better Models: Worse Tools"
[14]: https://pi.dev/docs/latest/sdk?utm_source=chatgpt.com "SDK · Documentation · Pi"
