---
title: 什么是 Harness？
description: Agent harness 是一种为 AI 模型提供运行环境的软件。
template: updates
aria_label: Earendil 文章
from: Earendil Product <rfc@earendil.com>
to: 你
date: Thu, 20 Aug 2026 12:03:55 +0200
subject: 什么是 Harness？
---

**Harness** —— 剑桥词典释义

*名词。* 一种带有带子和腰带的装备，用来控制或固定人、动物或物体

*动词。* 控制某物，通常是为了利用它的力量

–

当我想到 harness 时，我首先想到的是中学时攀爬学校墙面之前穿在身上的那套带子和腰带。说得客气一点，我也只是个水平很普通的攀岩者。

<figure class="post-figure">
  <img
    src="/static/posts/what-is-a-harness/royal-robbins-el-capitan-climbing-05.png"
    width="1440"
    height="2182"
    alt="Royal Robbins 正在攀登 El Capitan，安全带上挂满攀登工具。"
    loading="lazy"
    decoding="async">
  <figcaption>Royal Robbins 正在攀登 El Capitan，安全带上挂满了攀登所需的工具。照片由 <a href="https://www.frostworksclimbing.com/cool_aid.htm">Tom Frost</a> 拍摄。</figcaption>
</figure>

不过，如果这些天你一直高强度摄入 AI 新闻，那么你脑海里最典型的 harness 可能早已是 agent harness（智能体运行框架）了。那么，这篇文章不是写给你的。

这篇文章写给那些好奇 agent harness 到底是什么、却并不了解，而且一直因为不好意思而没敢问的人。

还是回到攀岩。

为什么攀岩时要穿 harness？首先，它会支撑你、保护你的安全。它把你连接到登山扣与绳索上，防止你坠落、控制你的移动节奏，也约束你能走的路线。你还可以把别的工具挂在 harness 上，比如粉袋、岩塞取出器和快挂。

而当你去爬不同的山，或者选择不同的攀登线路时，harness 也可以跟着你一起去。根据地形，你甚至可以修改自己的 harness，调整装备环上挂着什么。攀岩 harness 是可适配的。杂技演员和树艺师也会使用类似装备。拥有它的人，可以把它变成真正属于自己的东西。

无论从结构还是功能来看，攀岩 harness 与 agent harness 都有相似之处。

## Agent Harness

有人曾经用一个相当简化的公式来表达：Agent = Model + Harness。这里的 Harness，指的就是 Agent Harness。但 agent harness 究竟是什么？Agent harness 使用 AI 模型来构建 AI agent（人工智能智能体），它最早大规模落地的用途是编程。如今，agent harness 已经处于各种 AI agent 的核心。理解 agent harness 如何工作，也会帮助你理解 AI agent 到底是什么。

Agent harness 是一种软件，它为 AI 模型提供一个可以在其中运行的环境。与大多数 AI 模型不同，作为最终用户，你可以真正拥有自己的 agent harness。

软件工程师等用户经常通过自己电脑上的 Terminal（终端）应用，直接与 [Pi](https://pi.dev/) 这样的 harness 交互。不过，[OpenClaw](https://openclaw.ai/) 这样的 harness 也会使用其他用户界面，比如 iMessage、聊天应用或电子邮件。我们的 harness [Lefos](https://www.lefos.com/about) 则主要围绕电子邮件交互构建。无论界面是什么，harness 通常都会做四件事。第一，它提供一组指令，用来约束和引导 AI 模型如何响应；这组指令通常叫作“system prompt（系统提示词）”。第二，它描述并提供一组工具，让 AI 模型可以使用这些工具来完成用户请求。第三，harness 建立一套约束模型行为的框架。这套框架会做许多事情，其中最重要的一项，是建立“agentic loop（智能体循环）”。最后，大多数 harness 都提供一个关键的 translation layer（转换层），使同一个 harness 能够与不同 AI 模型配合工作。

### I. System Prompt（系统提示词）

多数 AI 模型本身都带有一组在训练过程中逐步形成并反复完善的内嵌规则与指导原则。最著名的例子之一，是 Claude Opus 4.5 那份广为流传的“[soul document（灵魂文档）](https://gist.github.com/Richard-Weiss/efe157692991535403bd7e7fb20b6695)”，它向 AI 模型解释自己是什么、应该如何行动。AI harness 里的 System Prompt 与之类似，但它并没有那么深地嵌入模型本身。它更像新员工上班第一天收到的一套工作说明：员工并没有把这些说明内化成自己的本能，但知道做这份工作时应该遵守它们。System prompt 会随着每一次 prompt 一同注入对话，对确保 AI 模型在特定 harness 的语境中做出合适行为起着重要作用。

### II. Tools（工具）

工具是一组用代码实现、可供模型“调用”的能力。Harness 会描述这些工具，也会实际提供实现工具功能的软件。例如，工具可能包括网页搜索工具、让模型编写并执行软件代码的工具，或者让模型撰写邮件的工具。关键在于，harness 通常不会规定 AI 模型必须在什么时候、以什么方式使用工具。它只是把工具提供出来，清晰描述它们，然后让 AI 模型自己决定何时以及如何使用。

### III. Agentic Loops（智能体循环）

现在，我们有一个位于 agent harness 中的 AI 模型，同时给它一组指令和一组工具。假设这个 harness 是围绕电子邮件工作的，拥有刚才提到的工具（WebSearch、WriteCode、ComposeEmail），用户要求 agent 比较本地小学的排名和考试成绩，然后给出推荐。Agent 会怎样行动？首先，它会尝试理解请求，也就是“prompt（提示）”。它会利用预训练所得的知识与权重，理解“primary school（小学）”是什么、“local area（本地范围）”指什么，以及用户大概关心哪些排名。接着，它会构造网页搜索查询，获取最新数据。拿到结果之后呢？因为模型位于 harness 中，它可以把这些结果放回最初请求的上下文中审视。它也许会判断第一次搜索并没有找到正确的信息，或者信息不够，于是自行决定再搜索一次。模型根据自己的判断再次调用工具，这就是“循环”最清晰的第一个例子。

现在假设它已经收集了所有相关数据。AI 模型决定使用“write code”工具制作一个电子表格——毕竟，从某种意义上说，电子表格也只是代码。它可以借助这个工具做计算，并把结果格式化成容易理解的形式。然后，它把电子表格与原始 prompt 对照。如果数据还不能令它满意，它就可能再次“loop”，回头继续搜索。当它判断信息已经足够时，会调用 ComposeEmail：这个工具让 AI 可以审阅自己的发现、总结结果、撰写邮件，并加入电子表格这样的附件。模型检查最终成果，判断任务已经完成。于是“agentic loop”闭合。几秒钟内，用户就会收到一封邮件：正文里有总结和建议，附件里有展示调查结果的电子表格。想看看真实的 agentic loop 是什么样，可以探索[这个 Pi session](https://pi.dev/session/#b23f2459599f8439327f65c90ee95d06)。

### IV. Translation Layer（转换层）

转换层让同一个 harness 能够配合不同 AI 模型工作。有时，一个 harness 甚至可能在同一轮 agentic loop 中决定使用不同模型，因为不同 AI 模型可能分别擅长不同任务。转换层也是 harness 非常关键的一部分，因为它把控制权交给最终用户。这意味着，一个人可以拿自己的 AI harness 去使用 Anthropic 的模型、OpenAI 的模型，或者尝试那些常常能提供很高性价比（按每项任务成本衡量）的 open-weight（开放权重）AI 模型。

这个转换层有助于把一部分权力与杠杆从 AI 实验室手中转移到最终用户手中。如果人们能够拥有自己的 harness，并在自己的电脑上本地运行，就意味着他们保留了自己的能动性，也保留了把工具真正变成自己工具的自由；同时，他们还能在本地保留 session（会话）的副本，而这些 session 随着时间会逐渐构成人与机器之间的通信记录。与其使用某个 AI 实验室发布的封闭应用，不如围绕一个 harness 建立关系并使用它，这样用户能保留更多自由与选择。在前面的示例里，用户完全可以把同一封邮件分别交给 OpenAI 的模型、Anthropic 的模型以及一个开放权重模型，然后比较它们的结果、结果的成本，并把所有答案保存在同一个地方，而不是让三个答案分别困在三个应用里。

## 把 Harness 变成你的

与 AI 模型本身不同，harness 是你可以拥有并改造的。就像攀岩 harness 一样，你可以把它变成真正属于自己的东西。这正是人们喜欢 Pi 的原因之一。Pi 是一个极简的 agent harness：system prompt 很短，默认工具集很小，开箱即用时它的设计目标就是尽可能“不挡路”。但随着人们使用 Pi，他们会按照自己的需求扩展和塑造它。他们会修改 system prompt，或者设计一个适配某种工作流的[扩展](https://pi.dev/packages)，然后再把这些扩展分享给别人。Pi 用户已经彼此分享了超过 5,000 个扩展。Pi 同时还是免费、开源的，并且就运行在你自己的笔记本电脑上。这意味着，人们如今拥有了一件真正属于自己的工具：它运行在自己的硬件上，让他们能够驾驭 AI。

## 作为能动性工具的中立开源 Harness

Harness 并不是从一开始就开源、中立。第一个广泛流行的 agent harness——Claude Code——并不是为了提供一个与模型厂商无关的 AI 转换层而构建的；它最初是一款应用，让用户可以在本地电脑上用 Claude 模型编程。此后，我们欣喜地看到，OpenClaw、OpenCode、Hermes 和 Pi 等免费开源 agent harness 不断增长。在 Earendil，我们正在把 Pi 构建成一个中立的 harness，为 Pi 用户提供能力选择与自由。我们也在探索，怎样把 harness 所提供的好处和能动性带给更广泛的人群。

现在，许多人都在担忧越来越庞大的 AI 公司所拥有的力量与影响。有些人也许会因此完全避开 AI。但在 Earendil，我们相信，可以通过打造软件与开放协议来增强人类的能动性；这些软件与协议能够弥合分歧与无知，并培育持久的快乐与理解。要做到这一点，我们不能忽视今天已经存在的技术，而应当睁大眼睛、牢牢握住把手去驾驭（harness）它们；确保是我们挥动锤子，而不是让锤子挥动我们。
