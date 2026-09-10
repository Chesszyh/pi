---
title: 你带不走的 Session
description: 推理 API 正在把 session 填满加密的 reasoning、隐藏的搜索结果、不透明的 compaction，以及加密的 subagent 消息。这正在成为一种日益严重的锁定形式。
template: updates
aria_label: Earendil 文章
from: Earendil Engineering <rfc@earendil.com>
to: 你
date: Thu, 30 Jul 2026 00:00:00 +0200
subject: 你带不走的 Session
---

最初，inference API（推理 API）的承诺简单得令人愉快：发送一些输入，收到一些输出。只要把两者都保存下来，你就拥有了整段对话。你可以检查它、归档它、重放它，或者把它交给另一个模型。

这个抽象从来都不是百分之百真实。例如，[prompt cache（提示缓存）实际存放](/posts/prompt-caching/)在别人的 GPU 上；不同模型使用不同的 tokenization（分词方式）；sampling（采样）也不可复现——而且往往是有意如此。但至少，一段 session 的*语义记录*仍然可以以 transcript（对话记录）的形式属于用户。Transcript 应当包含指令、消息、工具调用和工具结果。另一个足够强的模型也许不会以完全相同的方式继续，但它至少能理解此前发生了什么，然后接手工作。

令人沮丧的是，inference API 正在逐渐远离这一性质，至少部分如此。它们越来越多地返回文本与 provider-bound state（绑定于特定提供商的状态）的混合物，而后者非常明确地不可移植。

* 用户为 reasoning token（推理 token）付了费，但得到的只是不可读、加密的 blob（数据块），最多附带几乎无用的摘要；
* 网页搜索中，模型看到了客户端永远看不到的原始资料；
* 压缩后的上下文只有原提供商能够解密；
* subagent（子智能体）的指令和消息以加密 payload（负载）形式隐藏起来，连实际运行这些 agent 的应用都看不到；
* 文件、vector store（向量存储）、container（容器）和 cache 的引用离开原环境后无法解析；
* response 和 conversation state 完全依赖 ID，而真正的数据完整存放在提供商服务器上。

对每一项功能，提供商都很容易给出一个看似基本合理的理由，也可以提出不少“为什么这对用户有好处”的论点。但把这些东西合在一起，AI session 的所有权现实就被改变了：你机器上的 transcript 已经不再等于你的 session，而只是整个 session 的一个局部视图；真正的运行状态属于 inference provider（推理提供商），而不是你。

我们不喜欢这个方向。我们想谈谈它对作为用户的你意味着什么，也谈谈它对我们这些在这个领域开发工具的人意味着什么。

## 判断 Session 所有权的一个实际测试

所谓 portable session（可移植会话），并不是说从一个模型切换到另一个模型后，下一 token 必须完全一样。模型能力、训练出来的性格、上下文窗口以及使用工具的方式都不同，而且这一切本来就非常非确定。我们所说的可移植性，是一个更克制的要求：

```javascript
const transcript = session.export();
revokeCredentials(oldProvider);
session = newProvider.continueFrom(transcript);
```

导出的归档应当包含足够多、而且人能够理解的信息，使另一个模型可以继续工作。它不应该要求旧提供商去解引用某个 ID、解密一个 blob、记住一次搜索结果，或者替你重建一份摘要。

由此可以得到五项很有用的测试：

1. **检查（Inspection）：** 用户能否看到模型当时看到了什么、调用了哪些工具，以及各个 agent 彼此说了什么？
2. **导出（Export）：** 除了那些同样可以下载的普通 artifact（产物）之外，session 本身是否自包含？
3. **重放（Replay）：** 另一种实现能否重建一个语义等价的上下文？
4. **审计（Audit）：** 事后，人能否解释系统为什么采取某项行动？
5. **删除（Deletion）：** 用户能否识别并删除 session 所依赖的每一份服务器端副本？

Response ID 不是 transcript，因为数据存在服务器上；ciphertext（密文）不是用户可控制的状态，因为用户无法解密它；一串 citation（引用）也不等同于搜索结果实际放入模型上下文中的证据，因为你通常无法重新取得模型当时看到的同一份数据。

## 加密是为谁服务的？

围绕这些功能使用的命名与市场宣传可能会产生误导。
`encrypted_content` 听起来像一种由用户掌控的隐私功能。
但通常，它其实只是一个客户端无法读取、只有提供商才能打开的 capsule（封装数据）。提供商选择密钥，为自己的模型解密内容，并定义这些数据能够在哪里被重放。

更准确的名称应该是 **provider-sealed state（提供商封存状态）**。

Provider sealing 的确可能带来真实的隐私收益。例如 OpenAI 可以在客户端设置 `store: false` 时返回加密 reasoning，然后在下一次请求里只在内存中解密，而不持久化其中间状态。尤其对 Zero Data Retention（零数据保留）客户来说，这比强制使用服务器端 conversation storage（会话存储）要好。但要记住，从一开始其实并没有什么东西非得加密成这样不可。

这种加密并不会把数据藏起来不给 inference provider 看；
它只是把数据藏起来不给你看。

## Stored Conversation 把 Transcript 变成了一个 Pointer

OpenAI 的 Responses API 默认存储 response。其文档称，response object 默认至少保留 30 天。你可以设置 `store: false`，而且应该这么做，因为这样它会更像传统 completions：数据不会被存放在 OpenAI 的服务器上。

新的 Gemini Interactions API 也作出了类似选择，默认 `store: true`。在付费层级中，interaction 会保留 55 天；免费层级则保留一天。

当然，服务器端保存状态这个想法本身非常有吸引力：

```javascript
const first = responses.create({
  model: "frontier-model",
  input: "Investigate this production failure",
  store: true,
});

const second = responses.create({
  model: "frontier-model",
  previousResponseId: first.id,
  input: "Now implement the fix",
  store: true,
});
```

应用要发送的数据更少，提供商可以保留隐藏 reasoning 和 tool state，缓存路由也更容易。但如果本地应用只记录 user message 和最终文本，那么 `first.id` 此时就成了指向一个你无法控制的数据库的 foreign key（外键）。

## 你的 Reasoning？不给你看

所有主要 AI 实验室都声称，它们有正当理由不暴露 raw chain of thought（原始思维链）。因此，在非 open-weight（开放权重）模型上，我们通常都看不到这些 token。

原始 reasoning 无法通过 API 查看。使用 stored response 时，可以通过 `previous_response_id` 恢复此前 reasoning；如果设置 `store: false`，API 则返回 `encrypted_content`，客户端必须保存并在之后原样重放。即使 `reasoning.context: "all_turns"` 能让后续采样继续使用之前 reasoning，被持久保存的 reasoning 仍然是不透明的。

Anthropic 会把加密后的完整 thinking 放在 `signature` 字段中返回。启用 readable thinking 后看到的文本，是另一个模型生成的摘要，并不是原始 chain of thought。在涉及 tool use 的 turn 中，thinking block 必须原样传回。Anthropic 的文档也说明，thinking block 与生成它的模型绑定，切换模型时应当把它们去掉。因此，即便在 Anthropic 内部，这些 reasoning trace 也没有试图做到可移植。

同样的故事在所有 closed-weight（闭源权重）模型上不断重演。

这些加密机制可以实现*同一生态内部*的连续性，但它们并没有创造一份能够带到另一提供商模型上的 portable transcript。一份 session archive 可以保存这些 blob，但另一个模型无法使用其中的语义：

```json
◊{"type": "reasoning", "encrypted_content": "gAAAAAB..."}
◊{"type": "thinking", "thinking": "", "signature": "EqQBCg..."}
◊{"type": "thought", "summary": [], "signature": "EpoGCp..."}
```

## 隐藏的搜索

可以用网页搜索作类比。
Server-side web search（服务端网页搜索）是 transcript 中“缺洞”最明显的例子之一：模型看过内容，而用户的记录里没有。如果使用 client-side search tool（客户端搜索工具），它和其他工具一样工作：

```javascript
const result = search(query);
record({
  query,
  retrievedAt: now(),
  results: result.map((item) => ({
    url: item.url,
    title: item.title,
    passages: item.passages,
  })),
});
model.send({ toolResult: result });
```

用户可以检查搜索排名和 passages（检索片段），重新抓取网页，缓存一份副本，或者把完全相同的证据交给另一个模型。

而使用 hosted search（托管搜索）时，提供商会在内部执行一个私有的 tool loop。OpenAI、Google 和 Anthropic 会暴露搜索动作、citation，以及可选的来源 URL 列表，但不会暴露用于生成回答的完整文本上下文。一个 URL 并不是稳定可重放的记录：页面内容可能已经发生变化，或者在模型看到之前，就已经被压缩成短得多的 snippet（片段）。

最终答案可能完全没有问题。真正的问题会在下一轮出现：

> 比较第三个来源和第一个来源，重新核实那个有争议的数字，
> 然后换一个模型继续这项研究。

新模型得到的是前一个回答和几个 URL。它拿不到结果排名、被抽取的 passages、被过滤掉的材料，也拿不到第一个模型真正使用的精确证据。即便下一次请求已经发往别处，旧提供商仍然事实上是 session 的一部分。就算你手里有 citations 并重新抓取网页，也无法复现当时模型所见的精确数据。

Hosted search 应当提供一种 full-fidelity export（全保真导出）模式，其中包含 query、结果 metadata、检索 passages、时间戳以及被保留的内容。简洁 citation 可以继续作为用户界面，但不应该成为唯一记录。

## 不透明的 Compaction

长期运行的 agent session 最终都需要 compaction（上下文压缩）。由客户端控制、可见的 summary 虽然有损，但至少可以检查和转移。用户能够阅读它、编辑它，或者让另一个模型重新生成一份。

OpenAI 的 server-side compaction 则会产生一个加密的 compaction item。文档把它描述为“opaque and not intended to be human-interpretable（不透明，且并非供人理解）”。独立的 `/responses/compact` endpoint 会返回一个“canonical next context window（规范的下一上下文窗口）”，并要求客户端原样传递。

从概念上看，这个转换大致如下：

```javascript
// Before: expensive but portable
let history = [
  userMessage,
  assistantMessage,
  toolCall,
  fullToolResult,
  // ... 200,000 more tokens of intelligible history
];

// After: cheap to continue only with the original provider
history = [
  {
    type: "compaction",
    encryptedContent: "enc_provider_only_state...",
  },
  ...recentItems,
];
```

OpenAI 可以继续使用压缩后的语义，而另一个提供商看到的只是不可读字符串以及最近的一小段后缀——严格说来它连这个也不会看到，因为 Pi 从不会把这类信息传给另一提供商。

从技术上说，这并不是必须的。Anthropic 的 server-side compaction 会返回一个 `compaction` block，其中有可读的 `content` 字段；客户端还可以提供自定义 summarization instruction（总结指令），而最终 summary 可以被检查，也可以传给另一个模型。对于任何提供商，client-side compaction 也都是可行的。

OpenAI 的 sealed artifact（封存产物）也许确实能比普通 summary 保留更多模型特定状态，并在原模型上表现更好。把它作为可选优化是合理的，但它应该同时附带一份可读的 handoff summary（交接摘要），而不是用密文取代摘要。再一次，这些设计的附带效果之一，就是把你进一步锁进同一套生态。

## Subagent 带着你看不到的指令

Multi-agent system（多智能体系统）会进一步放大问题，因为这时已经不存在唯一的一份 transcript。你面对的是一棵 session 树，以及 agent 之间持续传递的消息流。通常这些消息就像人写给 agent 的 prompt，只不过现在变成了机器写给另一台机器。

OpenAI 托管的 Responses Multi-agent beta 返回三种新 item type：`multi_agent_call`、`multi_agent_call_output` 和 `agent_message`。`spawn_agent` 的示例中包含一个被加密的 `message` 参数，agent 间消息也只包含 `encrypted_content`。一旦启用 Multi-agent，每个 agent 都会隐式启用自动 server-side compaction，即便客户端并没有要求这样做。Reasoning summary 也不受支持，并且 API 还会向 root agent 和 subagent 注入开发者无法编辑或删除的指令。

这实际上是一整套不可转移状态的组合：被封存的 delegation（任务委派）、被封存的 agent 消息、分别自动压缩的上下文、隐藏 reasoning，以及由提供商托管的 orchestration（编排）。

2026 年 6 月，开源 Codex 客户端也合入了一项相关变化。Commit 标题为 [“Encrypt multi-agent v2 message payloads”](https://github.com/openai/codex/commit/5f4d06ef186b896d316620556e561d59206c3ebf)，其中直接说明了这个流程：

```json
◊// Parent model's tool call, as persisted by Codex
{
  "name": "spawn_agent",
  "arguments": {
    "task_name": "worker",
    "message": "<ciphertext>"
  }
}

◊// Child model's input
{
  "type": "agent_message",
  "author": "/root",
  "recipient": "/root/worker",
  "content": [{
    "type": "encrypted_content",
    "encrypted_content": "<ciphertext>"
  }]
}
```

Responses API 会把 parent model 发出的 tool argument 加密，Codex 负责转发，随后 API 在内部为 child 解密。Codex 自己记录的 `InterAgentCommunication.content` 是空的。在它可读的 rollout 和 history 中，真正的任务内容完全缺失。

这大概不只是抽象的“切换模型”问题。很容易想象：如果 child 改错了文件、泄露了 secret、重复了另一个 agent 的工作，或者沿着一个错误假设继续执行，用户甚至无法回答最简单的问题：*那个 agent 当时究竟被要求做什么？*

一个[尚未关闭的 Codex issue](https://github.com/openai/codex/issues/28058) 要求在加密传输之外，另外保留一份可读的审计副本。这是最低限度可以接受的设计。更好的做法，则是让 plaintext（明文）的 agent 间消息继续成为默认。

## “绝大多数人不会在 Session 中途切换模型”

也许确实如此。绝大多数人也不会每周更换操作系统或手机运营商。但即便你从来没有行使这种自由，它仍然重要，因为它会改变你与提供商之间的关系，也改变提供商面对你的方式。

作为用户，你也可能因为很多原因不得不迁移一个 session：某个模型被退役、服务宕机、价格变化、某项策略阻止你的下一次请求（你好，Fable）、某个包含机密信息的阶段必须在本地执行，或者审计人员需要重建此前发生了什么。Agent 也在让 session 越来越长。一次 coding 或 research session 可能累积数天的决策与证据，而一个 personal assistant（个人助理）甚至可能积累跨越数年的 session transcript——当然，现在这些系统还没有存在那么久。

“可以离开”的选项本身，也会制造纪律。如果提供商知道用户可以把工作带到别处继续，它就必须在模型质量、价格、可靠性和信任上竞争。如果用户累积的上下文只有一家提供商能够解释，那么这会形成非常糟糕的激励。

## 一个可移植的 Inference API 应当承诺什么

我们希望 inference provider 和 agent builder 能采用一小组规则。

1. **本地 event log（事件日志）是 canonical source（规范来源）。** 服务器端存储可以镜像它，也可以用来加速，但客户端无需解引用服务器 ID，就应当能够重建 session。
2. **存储必须显式。** `store: false` 应当易于使用、文档清楚，而且最好成为默认值。凡是需要保留数据才能工作的功能，都应在使用点明确说明。
3. **任何 opaque item（不透明条目）都不应成为意义的唯一载体。** 加密 reasoning、compaction 以及 tool signature 可以为了同提供商内的质量而保留，但每一种都应该同时有可读、provider-neutral（与提供商无关）的 handoff representation。
4. **Hosted tool 必须有 full-fidelity log。** 记录精确的输入、输出、证据、过滤过程、provenance（来源链路）、时间戳和内容 hash，而不只是润色后的答案与 citations。
5. **Subagent communication 必须可审计。** 对每一个 agent，持久化它收到的精确可读任务、消息、结果、lineage（谱系）、模型以及工具权限。
6. **Compaction 必须可检查。** 返回可读 summary、用于生成它的指令，以及足够的 lineage，让人知道什么内容被丢弃了。
7. **Artifact 必须可导出。** 文件、container output、search snapshot 和生成媒体，都应当能被下载到 content-addressed（内容寻址）的本地归档中。

## 实际上，Distillation 是件好事

模型层也存在另一种相关的 lock-in（锁定）。

一些最大的美国 closed-weight 实验室，对外部 distillation（蒸馏）正变得越来越敌视。Anthropic 在 2026 年 2 月发布的[文章](https://www.anthropic.com/news/detecting-and-preventing-distillation-attacks)谈到 DeepSeek、Moonshot 和 MiniMax 据称发起的活动，并把它们称为“distillation attacks（蒸馏攻击）”。Anthropic 的商业条款声称客户拥有自己的 outputs，但禁止使用其服务训练与其竞争的 AI 模型。与此同时，Anthropic 自己的文章也承认，当 frontier lab 用 distillation 训练自己的模型时，“distillation is a widely used and legitimate training method（蒸馏是一种广泛使用且正当的训练方法）”。

Anthropic 会使用机器人从公开网页收集数据用于模型开发，也曾因切割纸质书籍后进行扫描而广为人知。OpenAI 同样表示会在可自由访问的公开互联网内容上训练，并主张利用公开可得的互联网材料进行训练属于 fair use（合理使用）。两家公司也都把“在自己体系内部用强模型生成数据来生产更小模型”的 distillation 描述成一种正常做法。OpenAI 还曾提供明确的[官方 API distillation 工作流](https://openai.com/index/api-model-distillation/)，允许用更强 OpenAI 模型的输出去 fine-tune（微调）更小的 OpenAI 模型。

其中的道德不对称很难忽视。这些实验室要求社会接受这样一件事：机器可以学习人类放在互联网上的庞大作品集合——往往没有事先获得每一个个体的许可；但与此同时，它们又坚持认为，其他机器不应该从这些实验室生成的输出中学习。这一原则最宽泛的版本，恰好允许“学习”单向流入闭源模型，却不允许能力再从它们那里向外流动。

我们认为，对 distillation 的默认态度应当从敌意转向支持。Distillation 可以把昂贵的 frontier capability（前沿能力）转化为更小、更便宜、更快的模型，使这些模型能够在本地、离线、受限硬件上，或者完全由用户控制的环境中运行。它能够增加竞争，在某个 API 消失时保存能力，也能降低常见任务所需的计算量与能源。

## 最低限度的自由

用户应当能够注销一个账户，同时保留自己的 session，并把它交给另一个模型。新模型可能不认同前一个模型的结论，可能追问，也可能表现更差。但它不应该在前一个模型看到用户历史、证据、计划和被委派工作的位置，只看到一串 ciphertext。

我们并不反对提供商构建更优秀的 stateful API（有状态 API）。我们反对的是，把更好的性能与更少的用户控制强绑定在一起。Stateful storage 应当是可选的；hosted tool 应当可观察；compaction 应当可读；agent communication 应当可审计；而 opaque reasoning 最理想的情况是根本不应 opaque，至少也应当提供可移植的 handoff。Distillation 应当成为让能力更广泛可得的一条路径，而不是被当成禁忌，用来为越来越高的围墙辩护。
