# Pi 系统理解

**Pi 最准确的定位不是“另一个 Codex/Claude Code CLI”，而是一个可编程、模型无关、以终端为主要界面的 coding-agent harness（编码代理运行支架）**。这里的 harness 指负责把模型、上下文、工具、代理循环、会话持久化和用户界面连接起来的运行层；它不是模型本身。

可以用下面这条公式理解：

```text
Pi
= pi-ai：统一模型与供应商
+ pi-agent-core：代理循环与工具执行
+ AgentSession：编码任务、会话、压缩、重试
+ Resource/Extension：上下文与可编程工作流
+ TUI / Print / JSON / RPC / SDK：不同交互入口
```

与 Codex 相比，Codex 更像一台安全策略、工作流和模型都调校好的“整机”；Pi 更像一套公开、可拆解、可重组的“代理底盘”。Pi 的核心竞争力不是默认功能最多，而是**你可以精确控制模型看到什么、能调用什么、每个生命周期阶段发生什么，以及如何记录和评测这些行为**。

以下分析基于我刚刚读取的 2026 年 8 月 20 日 `main` 分支；当前 `pi-coding-agent` 包版本为 `0.84.2`，要求 Node.js 22.19.0 或更高版本。

---

## 一、先区分仓库中的几种“Pi”

| 层次       | 主要包                                                        | 实际职责                                                       |
| -------- | ---------------------------------------------------------- | ---------------------------------------------------------- |
| 模型层      | `pi-ai`                                                    | 模型目录、鉴权、流式输出、推理块、工具调用、token 与费用统计、跨供应商上下文转换                |
| 通用代理层    | `pi-agent-core`                                            | 状态机、LLM 调用循环、工具执行、消息队列、事件流                                 |
| 编码产品层    | `pi-coding-agent`                                          | 实际的 `pi` CLI、编码工具、会话树、压缩、资源加载、扩展系统                         |
| 界面层      | `pi-tui`                                                   | TUI（Terminal User Interface，终端用户界面）组件、编辑器、Markdown 渲染、交互控件 |
| 可观测与评测   | `pi-telemetry`、`evals`                                     | span（操作区间）遥测接口、真实 AgentSession 行为评测                        |
| 新一代耐久运行层 | `AgentHarness`、`protocol`、`client`、`server`、SQLite backend | 可崩溃恢复、多 lane、远程多会话服务；目前部分明确标为实验性                           |

日常启动的 `pi` CLI，其成熟主路径仍然是：

```text
Interactive / Print / JSON / RPC / SDK
                    │
                    ▼
              AgentSession
     会话、资源、压缩、重试、模型状态
                    │
                    ▼
                  Agent
        消息、工具、队列、事件、循环
                    │
                    ▼
                  pi-ai
       provider、鉴权、模型 API 适配
                    │
                    ▼
    OpenAI / Anthropic / Google / 本地模型……
```

`Agent` 是通用的代理状态机；`AgentSession` 才加入“编码代理产品”的语义，例如自动保存、会话树、模型切换、上下文压缩、扩展钩子和自动重试。交互式界面、打印模式和当前的 RPC 模式，本质上都是 `AgentSession` 的不同输入输出外壳。

仓库里同时正在建设更复杂的 `AgentHarness`。它不是简单重命名，而是一套面向耐久执行的新运行时：用不可变 entry tree（条目树）、可变 register（寄存状态）和追加式 usage ledger（用量账本）保存完整状态，再通过持久化 operation state（操作状态）实现崩溃后恢复。配套的 `PiServer` 文档明确标注为 experimental，因此阅读仓库时不要把这条新路线和今天 CLI 已经稳定提供的全部行为混为一谈。

---

## 二、一次用户输入在 Pi 内部到底如何运行

理解 Pi 的最佳方式不是背命令，而是追踪一条输入的生命周期。

### 1. 输入预处理

当你输入一句话时，`AgentSession.prompt()` 首先检查它是不是扩展注册的命令，例如 `/mycommand`。扩展命令优先级最高，可以直接处理输入而不进入普通代理循环。

如果不是扩展命令，Pi 依次进行：

1. 触发 `input` 扩展事件，允许扩展阻止、接管或改写输入；
2. 展开 `/skill:name`；
3. 展开 Markdown prompt template（提示词模板）；
4. 检查模型、鉴权和是否需要提前压缩上下文；
5. 创建用户消息；
6. 触发 `before_agent_start`，允许扩展注入自定义消息或临时修改本回合系统提示词；
7. 交给底层 `Agent`。

这个顺序非常重要。例如：

* 想把用户的自然语言改写为统一格式，应使用 `input`；
* 想添加静态工作流，应使用 skill 或 prompt template；
* 想在模型启动前动态注入环境状态，应使用 `before_agent_start`；
* 想修改真正发送给供应商的消息，应使用 `context` 或 provider 相关钩子。

### 2. 一个 run 由多个 turn 组成

在 Pi 的术语里：

* **turn（回合）**：一次模型请求，加上该回复触发的整批工具调用；
* **run（运行）**：从用户输入开始，经过若干模型—工具回合，直到代理真正停止。

典型循环如下：

```text
用户消息
  ↓
模型回复，要求调用工具
  ↓
执行工具并加入 toolResult
  ↓
再次调用模型
  ↓
模型继续调用工具或给出最终回答
  ↓
run 结束
```

在真正调用模型前，Pi 会先执行 `transformContext()`，供应用层剪枝、压缩或注入消息；然后通过 `convertToLlm()`，把 Pi 内部的 `AgentMessage` 转换成供应商能理解的 `user / assistant / toolResult`。这使得 Pi 可以在会话中保存 Bash 执行记录、扩展消息、压缩摘要等内部类型，而不必把所有 UI 或状态数据原样发送给模型。

### 3. 工具调用

模型返回工具调用后，Pi 会：

1. 解析并按 TypeBox schema（运行时参数结构定义）验证参数；
2. 触发 `tool_execution_start`；
3. 依模型原始顺序执行 `beforeToolCall` / `tool_call` 预检；
4. 被阻止的调用直接产生错误结果；
5. 允许的调用进入实际执行；
6. 执行后触发 `afterToolCall` / `tool_result`，扩展可修改结果；
7. 将最终 `toolResult` 加入上下文；
8. 开始下一次模型请求。

底层 Agent 默认允许同一批工具并发执行，但预检按模型给出的顺序进行；完成事件按真实完成顺序出现，最终写入会话和提交给下一回合的结果仍保持模型原始调用顺序。如果批次中有任何工具被标为 sequential（顺序执行），整批都会顺序运行。

编码层对同一个文件的 `edit`、`write` 还增加了 file mutation queue（文件修改队列），因此同一文件的修改会串行化，不同文件仍可并行。需要注意的是，多个 Bash 工具调用本身可以并发；不要把“模型通常一次只发一个 Bash”误认为运行时强制串行。

### 4. Steering 和 follow-up

这是 Pi 中很值得单独理解的一组原语。

* **Steering（转向消息）**：当前 assistant turn 及其已经发出的工具调用全部结束后，在下一次模型请求前插入。它不会强行终止正在运行的 shell 命令。
* **Follow-up（后续任务）**：只有当代理已经没有工具调用、也没有 steering，本来准备结束 run 时才插入。

交互模式中，普通 Enter 默认提交 steering，Alt+Enter 提交 follow-up。底层又支持 `one-at-a-time` 和 `all` 两种队列消费方式。

因此：

* “别继续这个方向，改查另一个模块”适合 steering；
* “完成以后再写一份总结”适合 follow-up；
* “立即杀掉正在运行的进程”应使用 abort，而不是 steering。

### 5. Run 结束以后

模型循环表面结束后，`AgentSession` 仍会检查：

* 是否发生可重试错误；
* 是否需要自动压缩；
* 是否有扩展在 `agent_end` 中新加入消息；
* 是否还有队列；
* 最后才发出 `agent_settled`。

所以 `agent_end` 更接近“底层代理循环刚结束”，而 `agent_settled` 才表示自动重试、压缩与队列都已处理完毕。

---

## 三、Pi 的真正中心是上下文工程

维护者最初对 Pi 的描述强调 minimal system prompt（极简系统提示词）。当前源码中的默认提示词确实很短，而且是动态构造的：只描述当前实际启用的工具，附加少量工具指南、项目上下文、skill 索引和工作目录。关闭某些工具后，对应描述也会从提示词中消失。

Pi 提供了几类不同的定制机制，它们不应混用：

| 机制                        |       是否执行代码 | 进入上下文的方式       | 最适合解决什么           |
| ------------------------- | -----------: | -------------- | ----------------- |
| `AGENTS.md` / `CLAUDE.md` |            否 | 长期加入系统上下文      | 项目规范、构建命令、代码约束    |
| `SYSTEM.md`               |            否 | 替换默认系统提示词      | 彻底改变代理角色和基础行为     |
| `APPEND_SYSTEM.md`        |            否 | 追加系统提示词        | 在默认行为上增加固定政策      |
| Prompt Template           |            否 | 用户显式调用时展开      | 重复使用的任务模板         |
| Skill                     |        可包含脚本 | 元数据常驻，正文按需读取   | 可复用的专业工作流和资料包     |
| Extension                 | 是，TypeScript | 通过事件、工具和消息动态作用 | 权限、工具、UI、状态、集成、编排 |
| Pi Package                |           可能 | 分发上述资源的容器      | 安装、版本化和共享         |

### AGENTS.md 的继承方式

资源加载器会先加载全局上下文，再从文件系统祖先目录到当前工作目录逐级加载。在每一个目录内，候选优先级是：

```text
AGENTS.override.md
AGENTS.md
AGENTS.MD
CLAUDE.md
CLAUDE.MD
```

每个目录只取第一个存在的候选，但不同祖先目录的上下文可以同时存在。内容会被包进带路径的 XML 节点，因此模型能够区分每条规则的来源。

一个容易忽略的安全细节是：**项目 trust 被拒绝时，AGENTS.md 和 CLAUDE.md 仍会加载，除非你显式使用 `--no-context-files`。** Project trust 主要阻止项目自动加载 `.pi/settings.json`、项目扩展、skills、packages 和系统提示词文件，不是对所有仓库文本的信任隔离。

### Skill 的 progressive disclosure

Progressive disclosure（渐进式披露）指：启动时只把 skill 的名称和描述加入系统提示词；模型认为任务匹配时，再用 `read` 读取完整 `SKILL.md`、引用文档和脚本。这样可以拥有很多能力包，而不必每回合为所有说明支付上下文成本。你也可以用 `/skill:name` 强制展开正文。

Pi 实现 Agent Skills 标准，并且能够在设置中复用 `~/.claude/skills`、`~/.codex/skills` 等目录。因此你现有的 Codex skills 不必全部重写；但要检查其中是否假设了 Codex 特有工具或安全模型。

我建议采用这条分层原则：

* **稳定的项目事实和约束**放进 AGENTS.md；
* **偶尔触发的专业流程**做成 skill；
* **纯文本的重复任务入口**做成 prompt template；
* **需要动态行为、外部状态、拦截、工具或 UI**时才写 extension。

---

## 四、工具、扩展和“极简”哲学

Pi 默认激活四个工具：

```text
read
bash
edit
write
```

另外还内置 `grep`、`find`、`ls`，可以通过 `--tools` 启用。只读审查模式可以直接使用：

```bash
pi --tools read,grep,find,ls
```

完整内置工具集合及 CLI allowlist（允许列表）/exclude 机制都由编码层统一管理。

### Extension 是 Pi 最强、也最危险的机制

Extension 是直接运行的 TypeScript 模块，可以：

* 注册模型可调用的工具；
* 注册 slash command、快捷键和 CLI flag；
* 拦截或修改用户输入；
* 修改发送给模型的上下文；
* 检查甚至替换 provider request；
* 阻止 Bash、edit、write 等工具调用；
* 修改工具结果；
* 自定义压缩和分支摘要；
* 存储会话状态；
* 注册新模型供应商；
* 更换 TUI 编辑器、组件、渲染器和弹窗。

事件链覆盖了从：

```text
project_trust
→ input
→ before_agent_start
→ context
→ provider request / response
→ tool_call / tool_result
→ turn_end
→ agent_end / agent_settled
→ compaction / tree / session switch
```

的几乎整个生命周期。这意味着 Pi 的“插件”不是外围小功能，而是能够修改代理运行语义的 middleware（中间件）。

### “没有内置”不等于“不支持”

Pi 明确不内置：

* MCP（Model Context Protocol，模型上下文协议）；
* 子代理；
* plan mode；
* todo 系统；
  -权限弹窗；
* 后台 Bash。

这是架构取舍，不是技术缺口。仓库的官方 examples 中已经包含 plan-mode 和 subagent 示例；你也可以安装第三方包或自己写扩展。

维护者对 MCP 的核心批评是：大型通用 MCP server 往往暴露很多长工具描述，长期占用上下文；而 CLI、README 和脚本可以按需加载、写入文件并通过 shell 组合。这个判断不意味着 MCP 一定低效，而是解释了 Pi 为什么更倾向于“让模型调用小型 CLI，再用 skill 告诉模型怎么用”。([Mario Zechner][1])

因此，Pi 所谓 minimal（极简）的真正含义是：

> 核心只提供可组合原语；工作流政策由用户、项目和扩展决定。

代价则是：你必须自己承担更多架构、安全和维护责任。

---

## 五、会话不是线性聊天，而是一棵树

当前 CLI 会把会话保存为 JSONL（JSON Lines，每行一个 JSON 对象）。除文件头外，每个条目都有 `id` 和 `parentId`，所以单个文件内部就能形成树，而不是只保存一条线性历史。

条目不只包括普通消息，还包括：

* 模型切换；
* thinking level 切换；
* Bash execution；
* custom message；
* extension custom state；
* compaction summary；
* branch summary；
* label；
* session name。

三个容易混淆的操作是：

* **`/tree`**：在当前 JSONL 文件中切换叶节点，旧分支仍在同一文件；
* **`/fork`**：从之前的用户消息创建新的会话文件，并允许修改那条提示；
* **`/clone`**：把当前活动分支完整复制到新会话文件，编辑器为空。

### Compaction 的真实含义

Compaction（上下文压缩）不是删除历史，而是：

1. 找到一个切分点；
2. 对较老部分生成结构化摘要；
3. 保留最近一段原始消息；
4. 在会话树末端追加 compaction entry；
5. 后续模型看到“摘要 + 最近消息”。

完整历史仍然保存在 JSONL 中，使用 `/tree` 仍能回到原始条目。压缩对于模型上下文是有损的，但对于会话文件不是删除。默认规则还会避免从孤立的 tool result 中间切开，以免工具调用和结果失配。

当你从一个分支切到另一个分支时，Pi 还可以总结即将离开的路径，并把 branch summary 注入目标分支。这解决了“代码已经被旧分支修改，但新分支模型完全不知道旧分支做过什么”的一部分连续性问题。

从研究角度看，当前 JSONL 会话已经是一份相当好的**规范化高层轨迹**：它包含用户消息、模型回复、工具调用、工具结果、模型和用量信息。但它不是完整的原始 HTTP/wire trace；若要记录供应商请求体、响应头、每个流式 delta 和扩展内部状态，需要另外使用 provider hooks、AgentSession 事件或遥测扩展。

---

## 六、模型层：Pi 不是简单套一层 OpenAI-compatible API

`pi-ai` 明确区分两个概念：

* **Provider（供应商）**：拥有模型目录、鉴权方式、流式行为和模型归属；
* **API implementation / wire protocol（底层请求协议实现）**：例如 Anthropic Messages、OpenAI Responses、OpenAI Completions。

多个供应商可能复用同一个底层协议，例如许多平台都提供 OpenAI-compatible endpoint，但它们仍可以有不同的鉴权、模型目录和能力元数据。

Pi 统一处理：

* 文本、图像与工具调用；
* reasoning/thinking level；
* 输入、输出、cache read、cache write token；
* 费用估算；
* OAuth 和 API key；
* 模型目录；
* 中途切换模型和供应商；
* 序列化上下文。

它只收录支持工具调用的模型，因为工具调用被视为 agentic workflow（代理式工作流）的必要条件。当前 CLI 文档列出的订阅登录包括 Claude Pro/Max、ChatGPT Plus/Pro 的 Codex 接口和 GitHub Copilot，因此你可以在 Pi 中使用 ChatGPT 订阅鉴权，而不必只走 OpenAI API key。

### 跨供应商切换不是数学意义上的无损

Pi 允许在同一会话中从 Anthropic 切到 OpenAI 或其他供应商，但应把它理解为 best-effort handoff（尽力转换）。不同供应商的推理块、工具调用关联、签名数据和重放要求并不相同；Pi 会把它们转换为共同消息表示，但不能保证另一个模型以完全相同的语义理解。维护者也明确把这一层描述为尽力而为。([Mario Zechner][2])

所以：

* 普通文本、工具调用和结果通常可以较好迁移；
* 隐式推理状态、供应商特有缓存和签名数据更容易泄漏抽象；
* 长会话中频繁跨供应商切换，可能产生语义漂移。

Pi 的价值不是把所有模型差异“消灭”，而是把差异集中在一个可审查的适配层中。

---

## 七、四种运行方式，以及两个不同的 RPC

Pi 用户侧主要有四类运行方式：

1. **Interactive**：默认 TUI；
2. **Print / JSON**：单次命令或机器可读事件流；
3. **RPC（Remote Procedure Call，远程过程调用）**：由其他进程控制 Pi；
4. **SDK**：在 TypeScript 应用中直接嵌入 `AgentSession`。

选择原则很简单：

* TypeScript 应用中嵌入代理：直接用 SDK；
* Python、Rust 或其他语言控制 Pi：启动 `pi --mode rpc`；
* shell pipeline：使用 `pi -p`；
* 要观察完整交互：使用 TUI 或 JSON event stream。

当前 `pi --mode rpc` 使用 stdin/stdout 上的 LF 分隔 JSONL。仓库中的 `pi-protocol`、`PiClient` 和 `PiServer` 则是另一条实验性远程服务路线：使用四字节长度前缀加 CBOR（Concise Binary Object Representation，简洁二进制对象表示）消息，支持一个连接管理多个远程 session lease（会话租约）。不要把这两个协议当成同一个东西。

---

## 八、安全模型：这是 Pi 与 Codex 最大的行为差异

Pi 没有内置 sandbox（沙箱），也没有默认工具确认弹窗。`read`、`write`、`edit`、`bash` 和所有 TypeScript 扩展，都继承启动 Pi 的系统用户权限。

特别需要记住：

> **`--approve` 的意思是信任并加载项目级 Pi 配置与扩展，不是批准模型执行 shell 命令。**

Project trust 只是 input-loading guard（输入资源加载保护），不能阻止：

* 模型读取其他目录；
* Bash 使用你的网络和凭据；
* 恶意扩展执行任意代码；
* 仓库文档中的 prompt injection；
* build/test 命令触发不可信脚本。

你可以写一个 `tool_call` 扩展，在执行 `rm -rf`、`sudo` 或写入敏感路径前询问确认。但这只是应用层政策，不是操作系统强制边界；扩展本身仍拥有完整权限。

对不可信仓库、无人值守开发或长时间自动任务，合理方式是：

* 把整个 Pi 放入容器、VM 或 micro-VM；
* 只挂载必要工作区；
* 尽可能只读挂载；
* 不挂载主机的 `~/.pi/agent`；
* 只注入必要且短期的凭据；
* 不需要网络时关闭网络；
* 结束后检查 diff，再把结果带回可信环境。

Codex 当前的官方设计则把系统级沙箱、网络边界和 approval policy（批准策略）集成在产品中，并对越过沙箱边界的操作请求确认；Codex 还内置了 todo、web search 和 MCP 等工作流功能。([OpenAI][3])

因此，**不要把 Pi 的“无弹窗”理解成比 Codex 更自由但同样安全；它实际上是把安全边界交回给操作系统和用户。**

---

## 九、Pi 与 Codex 应该怎样比较

对 Codex 重度用户，最准确的比较不是“哪个模型编码更强”，因为 Pi 本身不是模型。

**Codex 是经过整体调校的编码代理产品。** 模型、系统提示、工具、进度管理、MCP、搜索、沙箱、批准策略、CLI/IDE/桌面端协作都属于同一产品栈。它更适合希望开箱即用、默认安全边界明确、长期自主工作的人。([OpenAI][3])

**Pi 是面向工作流所有权的代理支架。** 它允许你更换模型供应商、中途切模型、替换系统提示词、拦截 provider payload、修改上下文、定制工具、重写压缩、改变 TUI 和保存自定义会话状态。代价是很多政策需要你自己建立。

所以不建议把问题设成“是否彻底从 Codex 切换到 Pi”。更合理的分工是：

* **Codex**：默认生产力工具、需要成熟安全策略和高自治的日常开发；
* **Pi**：研究 harness 行为、比较不同模型、开发自定义 agent workflow、做深层轨迹记录和评测；
* **两者复用**：AGENTS.md、skills、CLI 工具、项目文档和测试环境可以尽量共享。

Pi 支持使用 ChatGPT Plus/Pro 的 Codex 登录，因此用同一模型分别跑 Codex harness 和 Pi harness，是研究“模型能力”和“代理支架能力”差异的一个很干净的实验。

---

## 十、Pi 对 Coding Agent 研究尤其有价值的部分

对轨迹记录、可靠性分析和能力评测而言，Pi 的价值明显高于普通封闭 CLI，原因有四个。

### 1. 运行循环完全公开

你可以直接检查：

* 上下文何时变换；
* 系统提示词如何构造；
* 工具参数如何验证；
* 工具调用如何并行；
* steering 和 follow-up 在何处入队；
* 会话何时落盘；
* retry 与 compaction 怎样接管控制流。

这比只观察终端输出更适合做因果分析。

### 2. 扩展事件覆盖完整轨迹

一个 trace extension（轨迹扩展）可以记录：

* 原始输入与输入改写；
* 每回合模型上下文；
* provider request/response 元数据；
* token-by-token message update；
* 工具参数、流式输出和最终结果；
* turn、run、retry、compaction、branch 和 session 生命周期。

### 3. 官方已有评测支架

`packages/evals` 使用真实 `AgentSession`，在隔离的临时项目与 agent 目录中运行任务，并保存原生 Pi JSONL 会话作为 artifact（评测产物）。它支持比较不同模型、系统提示、工具、skill 和 harness 配置，还提供 baseline/candidate、重复实验和 judge score 的组织方式。

这意味着你可以直接做：

```text
默认 Pi
vs
加入某个 AGENTS.md
vs
加入 skill
vs
加入 extension
vs
更换工具定义
vs
更换模型
```

而不必先自己造完整代理评测框架。

### 4. 遥测与未来耐久运行时

`pi-telemetry` 提供供应商无关的 span、attribute、event 和 status 接口，但不捆绑 exporter；应用可以接 OpenTelemetry、Sentry、日志系统或自定义存储。它适合作为性能与可靠性遥测基础，而不是现成的 SaaS 仪表盘。

新 `AgentHarness` 更进一步提出“effect sandwich（外部副作用三明治）”：

```text
先提交：准备执行某个外部副作用
执行：模型请求或真实工具调用
再提交：副作用结果与下一状态
```

如果进程在中间崩溃，恢复时至少知道某个副作用“可能已经发生但尚未结算”，再根据工具声明的 replay policy（重放策略）决定重试还是写入 synthetic interrupted result。这不能实现真正的 exactly-once external effect（外部副作用恰好一次），但能把不可消除的不确定窗口显式建模。

这部分目前更适合研究源码和架构方向，不应直接视为当前普通 CLI 已经具备完整的 crash-safe autonomy。

---

## 十一、建议的系统学习路径

不要一开始安装大量第三方包或复制别人的复杂配置。那会让你无法判断行为究竟来自 Pi、模型、prompt、skill 还是 extension。

### 第一阶段：只使用默认 Pi

在一个可丢弃的 Git 仓库或容器中：

```bash
npm install -g --ignore-scripts @earendil-works/pi-coding-agent
pi
/login
```

先掌握：

* `/model` 和 thinking level；
* `@file`；
* `!command` 与 `!!command`；
* steering 和 follow-up；
* `/session`、`/tree`、`/fork`、`/clone`；
* `/compact`；
* 只读模式 `--tools read,grep,find,ls`。

### 第二阶段：只做上下文工程

依次添加：

1. 一个简洁的 AGENTS.md；
2. 一个 prompt template；
3. 一个专用 skill。

每次只改变一个变量，用相同任务比较轨迹、工具调用数、token、耗时和结果质量。不要立刻替换整个系统提示词。

### 第三阶段：写第一个 extension

第一个扩展不应是复杂子代理。更适合从以下二选一开始：

* **工具安全拦截器**：记录并阻止危险 Bash；
* **轨迹记录器**：把 input、context、tool_call、tool_result、turn_end、agent_end 写入独立 JSONL。

这样可以同时理解 extension API、事件顺序和 AgentSession。

### 第四阶段：做 harness 对照实验

选择同一个 OpenAI 模型和同一仓库，分别运行：

```text
Codex 默认配置
Pi 默认配置
Pi + 同一个 AGENTS.md
Pi + skill
Pi + 自定义系统提示
```

评测：

* 是否完成任务；
* 修改是否正确；
* 测试通过率；
* 工具调用数量；
* 无效探索比例；
* 输入/输出/cache token；
* 用户干预次数；
* 是否产生不可恢复副作用。

这会比泛泛讨论“Pi 或 Codex 谁聪明”更有学习价值。

### 第五阶段：再进入 SDK、RPC 和 evals

* TypeScript 内嵌：`createAgentSession()`；
* 多次替换活动 session：`AgentSessionRuntime`；
* 非 TypeScript 进程：当前 JSONL RPC；
* 行为实验：`packages/evals`；
* 耐久多会话服务研究：最后再看 `AgentHarness`、protocol、client/server。

---

## 十二、推荐的源码阅读顺序

按这个顺序阅读，认知负担最低：

1. **`packages/coding-agent/README.md`**：产品表面、命令和设计哲学。
2. **`packages/coding-agent/src/core/system-prompt.ts`**：模型究竟被告知了什么。
3. **`packages/agent/src/agent-loop.ts`**：真正的模型—工具循环、队列和并发。
4. **`packages/agent/src/agent.ts`**：状态管理、事件归约、abort 和 queue API。
5. **`packages/coding-agent/src/core/agent-session.ts`**：把通用 Agent 变成编码产品的关键层。
6. **`resource-loader.ts`、`skills.ts`、`extensions/`**：上下文和可编程工作流。
7. **`session-manager.ts` 与 `compaction/`**：会话树、分支和上下文生命周期。
8. **`packages/ai`**：最后再处理各供应商的协议差异。
9. **`packages/evals`、`telemetry`、`agent/docs/harness.md`**：进入评测、可观测性与可靠性研究。

---

## 我的总体判断

Pi 最值得学习的不是它当前某个模型跑 benchmark 的分数，而是它展示了一种清晰的 agent architecture（代理架构）：

1. 模型供应商与代理循环解耦；
2. 应用消息与供应商消息解耦；
3. 工作流政策与核心循环解耦；
4. 会话历史与当前模型上下文解耦；
5. UI、自动化接口和 SDK 共用同一语义层；
6. 观测、干预和评测都能在公开边界上完成。

它的主要弱点也来自同一个选择：

* 默认安全边界弱于 Codex；
* 复杂配置容易演化成个人维护的“第二套 agent platform”；
* 多供应商抽象不可避免会泄漏；
* 扩展具有完整权限，供应链风险高；
* 稳定的 AgentSession 路线与实验性 AgentHarness 路线并存，阅读时需要区分代际；
* 最终表现依然高度依赖模型、提示词、工具环境和用户构建的工作流。

**对普通用户，Pi 不一定比 Codex 更省事；对希望进入 AI Agent 领域、研究代理轨迹、上下文工程、工具系统和可靠性的人，Pi 的学习价值很高。** 最合理的起点不是立刻“迁移”，而是把它作为一个可观测、可干预的实验平台，与 Codex 并行使用。

来源处理上，本文对当前行为以仓库源码和官方文档为准；维护者 2025 年的文章只用于解释设计动机。文章发表时部分功能尚未实现，例如当时缺少 compaction，而当前源码已经有完整压缩与分支摘要，因此不能把旧文章当作当前功能清单。([Mario Zechner][2])

[1]: https://mariozechner.at/posts/2025-11-02-what-if-you-dont-need-mcp/?utm_source=chatgpt.com "What if you don't need MCP at all?"
[2]: https://mariozechner.at/posts/2025-11-30-pi-coding-agent/?utm_source=chatgpt.com "What I learned building an opinionated and minimal coding agent"
[3]: https://openai.com/index/introducing-upgrades-to-codex/?utm_source=chatgpt.com "Introducing upgrades to Codex | OpenAI"
