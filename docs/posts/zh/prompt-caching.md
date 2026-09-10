---
title: Agent 中的 Prompt Caching
description: Prompt caching 如何影响 coding agent 的成本、延迟、工具设计和系统架构，以及 Pi 如何让缓存行为保持可观察。
template: updates
aria_label: Earendil 文章
from: Earendil Engineering <rfc@earendil.com>
to: 你
date: Wed, 22 Jul 2026 20:25:52 +0200
subject: Agent 中的 Prompt Caching
---

人们常常把大型语言模型想象成一个函数：输入一些文本，得到一些文本。这是一个很有用的抽象，但它忽略了运行 coding agent（编码智能体）时最重要的事实之一：绝大部分输入，其实和上一次完全相同。换句话说，我们大多数时候只是在已有内容后面追加。

Coding agent 会把 system prompt（系统提示词）、工具定义、项目指令、对话历史、工具调用以及工具结果发送给模型。到了下一轮，它又会把其中几乎所有东西重新发送一遍，只额外加入少量新内容。一旦一个 session（会话）增长到数万乃至数十万 token，每一轮都从头重新计算整份 prompt，不仅很慢，也很昂贵。

Prompt caching（提示缓存）让这种工作方式在经济上变得勉强可行，但缓存本身也相当脆弱。一次工具定义的修改、一次模型切换，或者一次提供商的路由决策，都可能把本应便宜的增量请求变成对整段上下文的完整重放。

因此，对 coding agent 来说，缓存行为并不只是实现细节或性能优化。它会影响延迟、成本、工具设计、session 设计，甚至影响产品究竟应该提供哪些功能。

## KV Cache 中到底有什么

Transformer 处理一段 prompt，大体分为两个阶段。在 **prefill（预填充）**阶段，它读取输入 token，并为它们计算 attention（注意力）状态；在 **decode（解码）**阶段，它逐个生成新的 token。

在每一层 attention 中，每个已经处理的 token 都会产生一个 key（键）和一个 value（值）。这里的 key-value 并不像哈希表中的键值查找：两者都是数值数组，通常由浮点数或更低精度的量化值组成。模型处理一个新 token 时，会把这个 token 的 **query（查询向量）**同此前的 **keys（键向量）**比较，以判断此前每个 token 与当前 token 的相关程度。随后，它根据这些相关性分数，对对应的 **values（值向量）**进行加权混合。从这个意义上说，key 是模型用来进行匹配的东西，value 则是它据此取回的信息——只不过这种查找是模糊的，而不像字典查询那样“返回唯一的精确匹配”。

这些 key 和 value 会被保留下来，这样在生成下一个 token 时，模型就能关注此前全部内容，而无需重新计算先前 token。被保留下来的这部分状态，就是 **KV cache（Key-Value Cache，键值缓存）**。

从概念上看，一次请求大致是这样：

```text
◊request 1:

◊[system]◊[tools]◊[user]◊[assistant]◊[tool result]◊[user]
◊<--------------------- prefill -------------------->
◊                       |
◊                       K and V tensors per token and layer

◊request 2:

◊[system]◊[tools]◊[user]◊[assistant]◊[tool result]◊[user]◊[new]
◊<---------------- reusable prefix ---------------->◊<--->
◊                                                    |
◊                                                    new work
```

真实的表示形式会更复杂，并且取决于具体模型，而且体积“相当”大。这里最重要的性质是：它们对应某个特定的 token 前缀。两段 prompt 即使语义完全相同，只要 tokenize（分词）结果不同，就不能共享同一份 KV cache。如果中间某个 token 发生变化，那么它之后的一切都会变成另一条 continuation（续接序列）。

Prompt caching 把这种状态的生命周期从“一次生成”延长到了后续请求。当 coding agent 的下一次 API 请求仍然以完全相同的 token 开头时，推理系统就可以复用已经存储的、与该前缀对应的计算结果，只对新增加的后缀执行 prefill。到这里还只是理论。

## Cache 存在哪里

缓存要想工作，就必须存放在某个地方，而且之后还必须能够定位到它。推理系统大体有两种办法，让后续请求能够继续利用 KV cache。

较简单的一种方式是 **session affinity（会话亲和性）**。它的做法是：把 KV cache 保留在最初计算它的 GPU 上或附近，并把同一 session 的下一次请求重新路由到同一个 worker（工作节点）。Session ID 或 prompt-cache key 因而可以成为一种非常简单的路由提示；甚至不需要查看请求负载，只在 HTTP 负载均衡器这一层就有可能处理这个问题。

```text
◊request(session-42) ◊--> router ◊--> worker 7 ◊--> GPU 7 KV cache
◊next(session-42)    ◊--> router ◊--> worker 7 ◊--> GPU 7 KV cache
```

这样可以避免在网络上搬运一份非常大的缓存。工作正常时，它速度很快，但也限制了调度。被选中的 worker 可能过载、重启，或者把这条缓存淘汰掉。Router（路由器）也可能判断，对整个集群进行负载均衡比保住某一个 session 的缓存更重要。尽管如此，这仍然是一种很有吸引力的方案，因为它只需要很少的额外部署基础设施和硬件。

另一种方法是 **分布式缓存**。KV block（KV 数据块）可以被存入另一个内存层级，或者让多个 worker 都能访问，这样一个请求就不必强绑定到某一块 GPU。

```text
◊                         +--------------------+
◊request ◊--> scheduler ◊-->| worker 3 / GPU 3   |
◊             |           +--------------------+
◊             |
◊             +----------> distributed KV blocks
◊             |
◊             +----------> worker 9 / GPU 9
```

这会提升调度灵活性和故障恢复能力，但如何搬运、索引和保留 KV block，本身就是一个系统工程问题。不同实现会以不同方式组合 GPU 显存、主机内存、本地存储、远程存储、前缀感知路由以及淘汰策略。

为了对 KV cache 的体量有个概念：它们确实可能很大，但在某些意义上又没有直觉里那么夸张。通过各种技巧，即便是很长的对话，KV cache 的大小也可以被压缩到数 GB 的量级。

## Cache 与 Prefix

Pi 的 session 是树，而不是列表。`/tree` 可以把当前活跃对话移动回更早的节点，再从那里沿另一条分支继续。Rewind（回退）可以丢弃当前活跃后缀，却不从 session 文件中删除它。新的分支可能与旧上下文共享大部分内容、只共享一点内容，或者实际上完全不共享。这个设计并非 Pi 独有；相当多 coding agent 至少在概念上有类似机制。即使你的 session 并没有被显式表示成树，agent 支持某种形式的回退也并不罕见。

```text
◊                             +-- E -- F  another branch
◊                             |
◊session S: root ◊-- A ◊-- B ◊-- C ◊-- D  current branch
◊                   |
◊                   +-- Z  branch near the start
```

这三条分支完全可以拥有同一个 Pi session ID。从 router 的角度看，它们属于同一场会话；但从 prompt cache 的角度看，它们其实是三条 token 序列，只有部分前缀重合。

如果缓存保留了可复用的 prefix block，那么从 `D` 跳到 `F` 时，仍然可能复用 `root -> C`。但如果系统只保留最“热”的 continuation，或者共享 block 已被淘汰，又或者请求被路由到别处，那么实际命中量可能小得多。跳转到 `Z` 时，即使它是从 `A` 附近分叉出去的，也可能只保住 system prompt 和最初的工具定义。这里精确的缓存管理行为高度取决于提供商。

反过来也可能发生。`/fork` 或新建 session 可能产生一个新的 session ID，却同时携带大量完全相同的上下文。如果路由系统按 session key 严格隔离缓存，它就可能意识不到这里存在可利用的大量重合。

真正决定哪些计算可以缓存的是“可复用前缀”。Session identity（会话标识）只是帮助基础设施更容易找到可能相关的缓存内容。在一些系统中，routing key 对缓存管理至关重要；另一些系统里，它只是一项优化。

## 显式前缀缓存与自动前缀缓存

提供商 API 主要以两种方式暴露缓存能力。

Anthropic 的传统接口使用显式的 `cache_control` 点。客户端会在请求中相对稳定的部分之后标记边界，例如 system prompt、工具定义，或者最近一段适合缓存的对话内容。服务器随后可以写入或查找“截至该边界为止”的前缀。边界虽然是显式指定的，但想要复用，边界之前的内容仍然必须匹配。不仅缓存点是显式的，定价也是显式的：你要为缓存写入付费，并可以选择要保留多久，而不同保留时长对应不同价位。

另一些 API 使用自动前缀缓存。客户端正常发送请求，由提供商自动寻找可复用前缀，不要求客户端设置断点。Prompt-cache key 或 session header 可能有助于路由或分组，但它们不会让两个不同的前缀 magically 变成同一个前缀。

## 为什么工具配置很容易把缓存打碎

工具定义通常位于对话之前，并且会在内部被“折叠”进 system prompt。工具名称、描述以及 JSON schema，与其他任何文本一样，都会成为模型输入。新增一个工具、删除一个工具、修改它的 schema，甚至仅仅以不同顺序序列化工具，都可能让第一个不匹配点提前到 prompt 非常靠前的位置。

```text
◊turn 1: ◊[system]◊[read]◊[write]◊[bash]◊[conversation...........]
◊turn 2: ◊[system]◊[read]◊[write]◊[bash]◊[deploy]◊[conversation...]
◊                                   |
◊                                   old conversation is now
◊                                   after a mismatch
```

在插件系统和 MCP（Model Context Protocol，模型上下文协议）风格的工具目录中，这是一类常见的意外。只在工具真正相关时再加载，看起来很高效，因为一开始可以少发送一些 schema。但对多数模型来说，新扩展出来的工具配置会使其后整段已缓存对话失效。为了节省区区几百或几千个工具 schema token，可能反而导致数万个对话 token 重新处理。

一些较新的模型 API 支持 **additive tool loading（增量式工具加载）**。新工具可以在 transcript（对话记录）内部某个特定 tool result 之后才变得可用，而不是被插回最初的工具列表。这样旧前缀就不会改变：

```text
◊[system]◊[initial tools]◊[conversation]◊[new tool]◊[next turn]
◊<--------- cached prefix ----------->
```

现在，对于原生支持 deferred-tool（延迟工具）机制的模型，Pi 已经支持这种做法。当 extension 通过 `setActiveTools()` 做出纯增量的工具变更时，Pi 会把新增工具名称记录在 tool result 上。对于支持该机制的 Anthropic 模型，它会使用 deferred definitions 和 `tool_reference`；对于支持的 OpenAI 模型，则会发出对应的 tool-search item。其他模型会走安全 fallback（回退方案）：Pi 会在下一次请求中发送完整的当前工具列表。功能上仍然正确，但可能会清空 prompt cache。

这里 **additive（只增不减）**这个词非常重要。删除工具、用另一套工具配置替换当前配置，或者修改 prompt 片段，仍然会改变更早的输入。一个 extension 如果每轮都重新构造 system prompt、打乱工具顺序、注入时间戳或修改 active tools，就可能在无意中让整个 session 的缓存都失效。

Pi 的可扩展性意味着，它无法替所有 extension 保证缓存稳定。我们可以提供 cache-friendly（缓存友好）的机制，但 extension 仍然必须正确使用它们。从我们看到的情况来说，很多 extension 都把缓存效率当作事后才考虑的问题。这部分是因为，当你使用固定订阅套餐时，cache miss（缓存未命中）所对应的实际成本并不会显得那么直观。

## 中断与 TTL

一些重要的 prompt cache 默认生命周期很短。Anthropic 默认五分钟的缓存尤其值得注意，因为五分钟比许多正常的编程活动都短。如果你在使用 Fable 时停下来喝杯咖啡，10 分钟之后再回来，那么仅仅发一句“say hi”，花费都可能比你预期高得多。

原因是：用户可能觉得一段 coding session 一直处于连续活跃状态，但推理提供商看到的其实是一系列彼此分离的请求：

```text
◊model request ◊--> run tests for 7 minutes ◊--> model request
◊                  no cache traffic here
```

一次较长的构建、测试套件、午饭、会议，甚至只是停下来审阅 diff，都可能持续得比缓存寿命更久。下一次请求包含的 prompt 完全相同，但已经保存的 KV 状态消失了，于是这个前缀又会按普通 input 重新计费。

由于 Pi 目前并不是 Anthropic 订阅方案允许使用的 harness，我们遵循 Anthropic 为 API 用户推荐的 5 分钟默认值。不过，通过查看 Claude Code 的代码库，我们知道，对他们自己的订阅用户，Claude Code 会把这个缓存超时提升到一小时。但当你需要按 API token 价格付费时，为此增加的成本往往并不值得。

不过，你可以主动选择更长的保留时间。Anthropic 等一些提供商暴露了更长的 retention（保留）控制。对于支持的直连 API，Pi 用户可以设置 `PI_CACHE_RETENTION=long` 来提出这种请求。但它终究只是一个请求：Pi 无法强迫 gateway（网关）保留某个条目，无法阻止内存压力下的淘汰，也不能在没有任何模型请求发生时强行让缓存一直“活着”。

## Cache Miss 的价格

提供商通常会对未缓存输入、缓存写入和缓存读取采用不同定价。Cache read（缓存读取）通常有折扣，因为昂贵的 prefill 计算已经完成。Cache write（缓存写入）则可能更贵，因为提供商承诺替你保留状态，以供之后复用。

想象一个 coding session 已经有 100,000 token 的历史，随后用户像上面的 Fable 例子一样发出一个很短的新请求。如果缓存正常工作，那么几乎全部历史都会按更低的 cache-read 价格计费。只有少量新增内容需要按普通 input 价格重新处理，并可能再写入缓存。

如果 cache miss，提供商就必须按正常 input 价格把整整 100,000 token 的历史重新处理一次。它还可能另外收取把这些历史重新写入缓存的费用。这就是为什么缓存过期后，一条像 `continue` 这样极短的请求都可能意外昂贵。在长时间 coding session 中，重新读取旧输入的成本可能远远高于生成下一条回答本身。

Caching 还可能制造一些不那么显眼的激励关系。

用户当然希望命中率高，因为这能降低延迟和价格。拥有 GPU 的推理运营商也应该希望命中率高：prefill 工作越少，同样硬件就能服务更多请求。设计得好的 cached-token 折扣能够让双方利益一致，同时给运营商留下更好的利润率。

Gateway 或 reseller（转售商）的激励可能不同。如果它的收入来自按未缓存费率计价的 input token，那么 cache miss 会让客户账单比 cache hit 更大。至于这是否也意味着更高利润，则取决于它的上游成本、合同结构，以及究竟是谁在运营缓存。在激励错配的技术栈里，负责路由的一方可能并不承担 cache miss 的全部成本，而向用户收费的一方却会在 miss 发生时得到更高收入。

这并不意味着提供商会故意破坏缓存；但它意味着，缓存性能应当是可观察的。用户不应该只能从一张异常高昂的账单里反推出发生了什么。能够理解缓存是否出现了异常，是一项重要的信息。

严格坚持缓存也意味着，gateway 在不同 turn 之间将你路由到“最佳选择”的灵活性会降低。你有时可能宁愿承受一次 cache miss，转而使用另一模型，因为从那一刻开始它也许会更经济；也可能在某些情况下，把负载均衡到另一个提供商反而更划算。

## 为什么 Pi 不会激进地 Prune

读到这里，你大概已经明白 Pi 为什么不会持续 prune（裁剪）工具调用了。为了控制成本，不断删除旧 tool result 或重写历史看起来很诱人，而且有些时候确实有必要，尤其接近 context-window（上下文窗口）上限时。但正如我们已经看到的，pruning 本身也有缓存成本。

从中间删除内容，会在删除点改变前缀。它后面所有仍被保留的对话内容，都可能需要重新处理。重写一段很长、原本已经缓存的上下文所产生的一次性成本，可能高于删掉少量低价 cached token 后未来能够节省的费用。

可以用一个粗略的盈亏平衡比较来理解：

```text
◊one-time rewrite cost
◊    ~= surviving tokens after the edit * (uncached price - cache-read price)

◊future savings per turn
◊    ~= pruned tokens * cache-read price
```

而且这不只是一个记账问题。旧的 tool result 往往包含模型后来作出决策时所依赖的证据。即使一份摘要保留了大意，删除这些原始结果仍然可能让行为质量下降。

因此，Pi 更偏好稳定、以追加为主的 transcript，不会把每一个旧 token 都当作垃圾。当上下文压力足以证明一次有损重写值得时，可以使用 compaction（上下文压缩）。由于 compaction 是有意创建新的上下文，而不是意外让同一份未改变 prompt 被重新计费，所以 Pi 在 session 统计中会把它视为一次 cache reset（缓存重置），而不是 cache failure（缓存失败）。

目标并不是把 prompt 压到尽可能小，而是在模型上下文、缓存复用、延迟与价格之间取得最佳权衡。

与此同时，pruning 有时也确实值得做。如果你使用的提供商不会因为良好的缓存使用而给你折扣，或者不管出于什么原因都难以获得较高缓存命中率，那么 prune 可能更合适。由于不同后端之间不能转移缓存，缩小上下文也确实会增加 router 在不同 backend 间负载均衡的空间。

## Pi 能做什么、不能做什么

Pi 会尽量让稳定的输入保持稳定。它会传递一致的 session ID 以及提供商特定的缓存提示；对于需要显式 cache point 的 API，会放置这些缓存点；它会记录 cache read 和 cache write 的用量；在模型支持时，也会使用 message-anchored additive tool loading（锚定在消息位置的增量式工具加载）。Pi 默认的 transcript 行为也会避免无缘无故重写旧上下文。

但请求一旦离开本机，后面的每一层都不是 Pi 能控制的。Pi 不能决定提供商的淘汰策略，不能把缓存寿命延长到 API 允许范围之外，不能保证某块 GPU 永不重启，也不能保证 gateway 一定遵守 affinity。Extension 如果修改了某个前缀，Pi 同样无法替它保住这个前缀。

Pi 能做的是，让 cache health（缓存健康状况）变得可见。

交互式 footer（底部状态栏）会把累计 cache read 和 cache write 显示为 `R` 和 `W`，并以 `CH` 显示最新一次请求的 cache-hit rate（缓存命中率）。`/session` 命令会给出更完整的信息：缓存与未缓存输入总量、累计命中率、成本，以及由[显著 cache miss](https://github.com/earendil-works/pi/blob/34f3719a942ecbf3e6d23e67098f47ba2867de0a/packages/coding-agent/src/core/cache-stats.ts#L50-L90)造成的、被重新计费的 token 数与美元成本估算。

```
◊Messages
◊Total: 178
◊User: 6
◊Assistant: 58
◊Tools: 114 calls, 114 results

◊Tokens
◊Input: 7,129,883
◊  Cached: 6,776,832 (95.0%)
◊  Uncached: 353,051
◊Output: 30,013
◊Total: 7,159,896

◊Cost
◊Total: $6.054
◊Cache Re-billed: $0.728 (161,744 tokens, 2 misses)
```

如果用户希望 cache miss 一发生就立即看到提示，可以在 `/settings` 中启用 **Show cache miss notices**，对应 `settings.json` 中的 `showCacheMissNotices`。发生显著 miss 后，Pi 会插入一条警告，其中包括估算的重新计费 token 数和成本。如果它能够观察到模型切换，或者空闲时间超过常见的短 TTL（Time To Live，生存时间），也会明确说明。对其他 miss，它只报告事实，而不会假装知道提供商内部究竟发生了什么。

## 导致缓存表现变差的常见原因

当一个 session 的 cache-hit rate 看起来不对时，常见原因包括：

1. **长时间空闲。** 一条命令、一次审阅，或者对话暂停超过了提供商的 retention window（保留窗口）。
2. **切换模型或提供商。** KV 状态与具体模型绑定，并且通常不能跨提供商移动。
3. **分支导航。** `/tree`、rewind、fork 和其他分支操作，即使不改变 session ID，也可能改变当前活跃 token 序列。
4. **Compaction 或手动重写历史。** 这些操作会有意替换 prompt 的一部分，并建立新的前缀。
5. **工具与 reasoning level（推理级别）变化。** 新增、删除、重新排序或编辑工具定义，都会改变请求中很靠前的一部分；只有当模型支持 message-anchored loading 且变化完全是增量式添加时例外。修改 reasoning level 通常也会产生同样效果。
6. **动态 system prompt。** 时间戳、随机值、变化的项目上下文，以及 extension 提供的 prompt 片段，都可能让其后全部内容失效。
7. **Extension 的上下文变换。** 如果 extension 修改旧消息或 provider payload，那么在 Pi transcript 里看起来稳定的内容，实际在线路上传输时可能并不稳定。
8. **提供商路由与淘汰。** 即使 prompt 完全一致，相关 KV block 也可能已经不在本次请求落到的位置，因此仍然 miss。
