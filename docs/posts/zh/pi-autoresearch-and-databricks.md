---
title: Pi：极简而高性能
description: Pi 的极简 harness 如何降低 coding agent 的成本并提升性能，以及 Databricks 与 Shopify 的 pi-autoresearch 扩展案例。
template: updates
aria_label: Earendil 文章
from: Earendil <rfc@earendil.com>
to: 你
date: Tue, 4 Aug 2026 01:00:00 +0200
subject: Pi：极简而高性能
i18n_key: post.pi-autoresearch-and-databricks
---

# Pi 的极简主义正是它的优势

AI 让代码变得廉价，因此许多公司正在追求更强性能的过程中不断把工具做大：更长的 prompt、更多 orchestration（编排）、更多层、更高复杂度。但这也让这些工具从根本上变得更昂贵。Pi 选择了相反的方向。

Pi 是一个有意选择极简主义的 coding harness（编码智能体运行框架）。它开箱只有 4 个工具，而且它的 [system prompt](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/system-prompt.ts#L121-L159) 与工具定义加起来不到 1,000 token。背后的思路是：绝大多数工作都可以靠基础能力完成；如果你还想要更多，那就自己构建。

越来越多的证据表明，Pi 的设计不仅更干净，也更便宜、性能更好。用户发现，即便还没有加入适配个人工作流和需求的 extension（扩展），原版 Pi 就已经能够给出业界领先的结果。正如下面 Databricks 与 Shopify 的案例所显示，Pi 在两种场景中都实现了理想结果。

## 案例研究

### **Databricks 研究：每项任务成本**

Databricks 最近分享了他们的研究结果——“[*Benchmarking Coding Agents on Databricks’ Multi-Million Line Codebase*](https://www.databricks.com/blog/benchmarking-coding-agents-databricks-multi-million-line-codebase)”。他们的研究目标，是理解哪些 coding agent 在真实编程任务上表现最好，以及任务性能如何随价格变化。

为了避免受到那些[已经被过度刷榜的外部 benchmark](https://arxiv.org/html/2602.16763v3) 的偏差影响，他们基于自己工程团队日常真正执行的任务，创建了内部 benchmark。结果符合我们的预期，但可能会让业内很多人感到意外。用他们自己的话说，“……模型通过哪个 harness 被调用，会显著影响成本与质量”；而且，“在很多情况下，Pi 这样的简单 harness 在我们的工作负载上表现最好。”
<figure class="post-figure">
  <img
    src="/static/posts/pi-autoresearch-and-databricks/databricks-cost-per-task.png"
    srcset="/static/posts/pi-autoresearch-and-databricks/databricks-cost-per-task.png 800w, /static/posts/pi-autoresearch-and-databricks/databricks-cost-per-task@2x.png 1600w"
    sizes="(max-width: 520px) calc(100vw - 56px), (max-width: 800px) calc(100vw - 80px), 760px"
    alt="Databricks benchmark 图表，对比不同 coding agent 的通过率与每项任务成本。"
    loading="lazy"
    decoding="async">
  <figcaption>图表由 Databricks 制作。</figcaption>
</figure>

当与 Opus 4.8、xhigh 组合使用时，Pi 的总体 pass rate（通过率）最高，同时成本显著低于 Claude Code 和 Codex。

#### 极简 harness，可以测量的影响

Pi 的优势来自它不会试图用大量默认设置和指令把模型层层包住，再让这些指令淹没在 [instruction hierarchy（指令层级）](https://openai.com/index/the-instruction-hierarchy/) 中。相反，Pi 尽量不挡模型的路，让团队能够只加入自己的工作流真正需要的东西。

Databricks 的研究很有价值，因为它把 model 与 harness 分开比较。

他们报告称，在不同 harness 中以相同 thinking effort（推理强度）运行同一个模型时，“每项任务的成本差异显著（某些情况下超过 2 倍），而质量保持相同”。我们把这称为 Pi 的“context discipline（上下文纪律）”。“Pi 每个 turn 发送的上下文大约少 3 倍。它对上下文管理得更好，维持更紧凑的工作集，并用更少的运行次数完成任务。”

我们同意，应该关注端到端的工程经济性，而不能只看每 token 单价。在模型层也是如此。例如我们观察到，在 Haiku 4.5 上执行复杂工作流，成本往往反而高于 Sonnet 4.6，尤其是涉及代码执行时；原因很简单：为了成功完成任务，较弱的 agent 需要更多 turn。

现在，我们也在 harness 层看到了同样现象：更强、更昂贵的模型配上高性能 harness，可能比反过来的组合更便宜。

### **Shopify 构建 Pi Autoresearch：可扩展胜过臃肿**

极简主义是 Pi 核心哲学的一部分。但这套思路之所以成立，是因为 minimal（极简）并不意味着 inflexible（僵化）。事实上，Pi 是第一批被广泛使用、从一开始就围绕 extensibility（可扩展性）与 self-editability（自我可编辑性）构建的 agent 基础设施之一。

另一个很有启发性的外部验证来自 Shopify。在这篇 [Shopify Engineering](https://shopify.engineering/autoresearch) 文章中，David Cortés 描述了如何直接把 `pi-autoresearch` 构建成一个 Pi extension：他只是向 Pi 提出“Pi，创建一个用于 Autoresearch 的 extension……”。Pi 会读取自己的 extension 文档，然后从那里开始构建新的工作流。

Autoresearch 是一种使用 coding agent 进行优化的 autonomous loop（自主循环）。当你要求进行某项修改时，它会运行实验，找出哪些改动有效、哪些会造成回归。只要目标能够被量化，它就可以丢弃那些导致回归的改动，并持续自我改进。

对于 Shopify [以及其他用户](https://x.com/pidotdev/status/2080616483072225778?s=20)来说，Autoresearch extension 很快就成了重要的内部生产力工具。Shopify 报告的案例包括：单元测试运行“快了 300 倍”、React 组件挂载“快了 20%”、多个项目构建时间下降，甚至 pnpm 的性能也得到了改进。

<figure class="post-figure">
  <img
    src="/static/posts/pi-autoresearch-and-databricks/shopify-autoresearch.png"
    srcset="/static/posts/pi-autoresearch-and-databricks/shopify-autoresearch.png 800w, /static/posts/pi-autoresearch-and-databricks/shopify-autoresearch@2x.png 1600w"
    sizes="(max-width: 520px) calc(100vw - 56px), (max-width: 800px) calc(100vw - 80px), 760px"
    alt="Shopify 的 pi-autoresearch GitHub 仓库截图。"
    loading="lazy"
    decoding="async">
  <figcaption>图片来自 Shopify 的 <a href="https://github.com/davebcn87/pi-autoresearch">pi-autoresearch GitHub 仓库</a>。</figcaption>
</figure>

这里真正重要的一点，是 Pi 开箱并不附带其中任何一种工具。相反，它让你自己构建这些东西变得简单到近乎荒谬。Pi 不会假设 vendor（厂商）比你更懂自己的工作流，然后试图把天底下所有工具都塞进产品；它默认你自己最清楚，并把 extensibility 交到你手中，让你能够驾驭并亲手塑造自己的工作流。

## 为什么极简现在会赢

大约一年前，人们还可以提出一个有力观点：native harness（原生 harness）相比其他所有 harness 都存在结构性优势，因为模型就是围绕它们构建的。但这个论点正在变弱。

如今的 frontier model（前沿模型）通常已经非常擅长理解 terminal（终端）或 terminal-style coding environment（终端式编程环境），并在其中行动。[Anthropic 最近把 Claude Code 的 system prompt 缩短了 80%](https://x.com/petergyang/status/2078895219534438556?s=20)，就是一个清晰信号。因此，问题越来越不在于 harness 与模型有多“原生”，而在于它如何管理上下文、避免冗余，以及能否提供干净的基本原语。模型需要一个简洁的环境接口，也需要一个不会浪费上下文的 harness。

Pi 提供的正是这些：更少的 prompt overhead（提示开销）和重复上下文、更便宜的运行、更少不必要的抽象。因为 Pi 可扩展，你并不会因此失去能力，反而得到更多选择性。只有当复杂性真正“证明自己值得存在”时，你才把它加进来。

我们也看到 local model（本地模型）正在快速发展，而在 [Earendil](https://earendil.com)，我们认为它们非常有前景。Pi 的 context discipline 在这里尤其有价值。本地模型通常拥有更小的上下文窗口，而 [prefill](/posts/prompt-caching/) 可能耗时很长，因此维持稳定的 prompt prefix（提示前缀）非常重要。Context discipline 意味着，除非用户明确要求，否则我们不会随意改变上下文，从而避免分钟级的重新 prefill。再配合极简的默认 system prompt 和工具集，这让 Pi 成为本地模型非常理想的 harness。

Pi 正在证明，它可以同时做到这一切：更便宜、更极简，也更高性能。
