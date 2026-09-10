---
title: Pi 中的 Compaction 是如何工作的
description: 为什么大型语言模型需要 compaction，以及 Pi 如何实现它
template: updates
aria_label: Earendil 文章
from: Earendil Engineering <rfc@earendil.com>
to: 你
date: Thu, 13 Aug 2026 8:30:00 +0200
subject: Pi 中的 Compaction 是如何工作的
---

如果你曾经在 [Pi](https://pi.dev)、Claude Code 或 Codex 这样的 coding agent（编码智能体）中进行过一次很长的编程会话，那么你一定触发过 compaction（上下文压缩）。
本文会解释 compaction 如何工作，以及 Pi 在什么时候需要进行压缩。

## 一次 LLM 对话
大型语言模型（LLM，Large Language Model）的[上下文窗口](https://en.wikipedia.org/wiki/Context_window)是有限的。
上下文窗口，就是模型在生成回答时能够“看到”的内容范围。
LLM 所使用的 [Transformer 架构](https://en.wikipedia.org/wiki/Transformer_(deep_learning))会限制模型一次能够处理多少输入。
一次 coding agent 会话的输入包含此前所有消息和工具调用，并且会随着你的工作不断增长。
一旦输入超过上下文窗口，LLM 就会拒绝这次请求。

当你以交互方式使用 Pi 这样的 coding agent 时，agent 会向 LLM 发送请求并接收响应。
每个请求都包含 system prompt（系统提示词）、已加载的文件（例如 [`AGENTS.md`](https://agents.md/)）、工具定义，以及对话历史。

Coding agent 向 LLM 发出的第一个请求，包含这份初始上下文以及用户的第一条消息。

```text
request 1:
[system][tools][user]
```

这会开始一个 turn（轮次）。
LLM 可能首先返回一条包含工具调用的 assistant 消息。
Agent 程序执行这些工具调用，然后向 LLM 发出一个新请求，其中包含完整对话，并新增工具执行结果。
随后我们又会得到一条 assistant 消息。
当 assistant 完成输出生成后，这个 turn 就结束了。

```text
after request 1:
[system][tools][user][assistant: tool call][tool result][assistant]
                     <------------------->     ^        <--------->
                     returned by LLM           |        returned by LLM
                                               |
                                     produced by the agent
```

我们继续工作，再发一条消息。
```text
request 2:
[system][tools][user][assistant: tool call][tool result][assistant][user]
                                                                     ^
                                                               new user message
```

每个 turn 都会让对话继续膨胀。
最终，历史记录会超过上下文限制。
这时，下一个请求就会返回类似 `Request exceeds the maximum size` 的错误。

```text
[system][tools][user][assistant][....][tool result][user]
                                                      ^
                                             exceeds context window
```

## 处理上下文溢出
当我们已经无法按原样继续现有对话时，有两种选择。

1. 可以开始一段全新的空白对话，不再携带已经累积的上下文。
这样会丢弃历史，其中也包括此前作出的决策和尚未解决的工作。
这么做有时仍然是个好主意，因为[随着上下文长度增长，LLM 输出性能会下降](https://www.trychroma.com/research/context-rot)。
2. 如果我们希望把这段对话继续下去，就可以为当前对话上下文创建一个更小的表示。
这就是 compaction 所做的事情。

## Compaction

从理论上说，compaction 有很多种实现方式。
例如，我们可以写一个确定性的函数，保留对话中的一部分内容，并丢弃其余部分。
但在实际系统中，compaction 的实现通常会额外发起一次 LLM 请求，让模型对对话历史进行总结。

Compaction 会把一部分历史替换成压缩后的表示，从而为后续消息和工具调用腾出空间。

```text
[system][tools][compaction result][user]
                                    ^
                               new message
```

## Pi 的实现

下面更具体地看看 Pi 是怎样[实现 compaction](https://pi.dev/docs/latest/compaction#summary-format) 的。

当对话变得过长时，Pi 会使用 compaction 总结较旧的内容，同时原样保留最近的工作。
当当前上下文大小已经接近上下文窗口总容量时，会触发 compaction。
你也可以用 `/compact` 命令手动触发。

Pi 会在一个 turn 结束后检查是否需要自动 compaction。
在此之前，每一个请求都只是在已有 prompt 后继续追加，因此能够复用其缓存前缀。
如果在某个 turn 进行过程中遇到上下文溢出错误，Pi 也可能在 turn 中途进行 compaction。

执行压缩时，Pi 会保留一定数量的最近消息，完全不作修改。
```text
before compaction:
[system + tools][older turns][recent retained messages]
```

具体保留多少条消息并不固定，因为 Pi 使用的是一个[可配置的 token 预算](https://pi.dev/docs/latest/compaction#when-it-triggers)。
Pi 当前默认预算为 2 万 token，大致相当于保留最近 5 到 20 个 turn。
分界点之前的所有消息都会被提取、序列化，然后用于生成摘要。

## Pi 的 compaction prompt
对于 coding agent 来说，一份理想的压缩摘要，应该像一次从上一班次交接给下一班次的交接简报。
Pi 的 compaction prompt 着眼于一个事实：现有上下文中，有大量内容已经不再相关。
我们真正应该保留的，只是对下一次 LLM 请求仍然重要的上下文。

因此，Pi 发起 compaction 时使用的请求，与常规对话请求不同。

1. 独立 compaction 请求使用不同的 system prompt。
它不会再告诉 LLM“你是一个专家级编码助手”，而会告诉它：[“你是一个上下文总结助手。”](https://github.com/earendil-works/pi/blob/47610217098d9ba8f22d223fa7c1413f9f5fd759/packages/coding-agent/src/core/compaction/utils.ts#L152-L158)
2. Compaction 请求里的 user message 也不同。
它要求模型生成[“这条对话分支的结构化摘要，以便稍后返回时作为上下文。”](https://github.com/earendil-works/pi/blob/47610217098d9ba8f22d223fa7c1413f9f5fd759/packages/coding-agent/src/core/compaction/compaction.ts#L463-L498)
Prompt 明确要求摘要包含目标、进展和关键决策等部分。
3. 这是一次独立请求，不会使用现有任何对话历史。因此它可以改用另一个 LLM 模型，也不会因此产生不必要的额外成本。

Compaction 的结果会以一条 compaction entry（压缩条目）的形式追加进 Pi session（会话），之后这个 session 就可以继续进行。
完成这次 compaction 请求后，上下文已经被压缩。

```text
after compaction:
[system][tools][summary][recent turns][new user message]
```

现在，对话上下文中又有空间容纳许多新的消息。

Pi 会把 compaction summary（压缩摘要）以纯文本形式存入 session。
这样，压缩后的上下文仍然易于阅读，也保持[可移植性](../session-portability)：我们可以在 Pi 中切换模型，并继续使用这份摘要。

## Compaction 与 prompt caching
LLM 提供商会使用 [prompt caching（提示缓存）](../prompt-caching)，降低同一对话中重复请求的成本。
在一个活跃的 coding session 里，对于模型此前已经处理过的上下文，我们支付的费用会更低。
这种缓存依赖完全一致的 prefix（前缀）匹配，因此对 session 做 compaction 会打破现有 prompt cache。

```text
cached before compaction:
[system][tools][older history][recent retained turns]
<-------------------- cached prefix -------------------->

first request after compaction:
[system][tools][summary][recent retained turns][new user message]
<-- reusable -->^
                |
        first changed token
                |
                +-- everything after this point must be recomputed
```

虽然被保留的那些 turn 本身仍然包含完全相同的 token，但现在它们前面接着的是另一个前缀。
因此，先前为它们保存的缓存状态无法继续复用。

Compaction 之后的新请求，会重新开始从 prompt caching 中受益。

## 实验

由于 Pi 是可扩展、可塑造的，你完全可以把它默认的 compaction 换成自己的实现。
如果想测试另一种 compaction 机制，可以直接让 Pi 创建一个使用自定义 compaction prompt 的 extension（扩展）。
