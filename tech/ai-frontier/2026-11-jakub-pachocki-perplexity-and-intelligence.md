# 长文精读｜Perplexity and Intelligence：把「压缩」当作智能的标尺

> **一句话**：Pachocki 把 AI 的底层进步还原为一件事——训练更好的压缩器；他公开 OpenAI 内部的 log-perplexity 曲线，主张把它变成跨实验室的「共同标尺」与「共同刹车」：在理解与监控能力达到具体里程碑之前，业界承诺不越过某条阈值。

## 文章档案

| 项目 | 内容 |
| --- | --- |
| 标题 | Perplexity and Intelligence（困惑度与智能） |
| 作者 | Jakub Pachocki，OpenAI 首席科学家 |
| 发布 | 2026-11-22，OpenAI Research / Safety |
| 原文 | [openai.com/index/perplexity-and-intelligence](https://openai.com/index/perplexity-and-intelligence/) |
| 篇幅 | 约 1,600 词（含脚注），5 条脚注，1 张交互图 |
| 标签 | Compute Scaling、Ethics & Safety |
| 性质 | 立场文章：公开内部数据 + 提出行业治理主张 |
| 注意 | 这里的「Perplexity」是指标，不是 AI 公司 Perplexity |

## 读前须知：三个容易误读的点

1. **数字是 log-perplexity，不是 perplexity 绝对值。** 文中 ~0.99、~0.91、"below 0.8" 都是交叉熵损失（log-perplexity）。若按自然对数换算成通常说的 perplexity：e^0.99≈2.69、e^0.91≈2.49、e^0.80≈2.23。曲线纵轴标的也是 log-perplexity。
2. **不是呼吁「全面暂停」。** 文章主张的是「阈值 + 条件」的配速机制：先设第一道可测上限；只有当我们对 AI 行为的理解与监控达到某个具体里程碑，才允许越过。媒体标题里的 "pause" 是简化。
3. **「2027 雪崩」是带 hedge 的外推。** 原文是 "as soon as 2027 we could conceivably see..."（最早 2027 年，我们有可能看到……），不是确定预测。

## TL;DR

1. **智能的底层是压缩。** 现代 AI 的多种技术，本质上是在驾驭一个更简单也更难解释的过程：在超大规模语料上训练压缩器（等价于预测器）。预训练造出具有难预测涌现能力的「异类心智」（alien mind），prompting / finetuning / RL 只是把潜在能力「诱导」（elicit）出来。
2. **perplexity 是追踪这条主线的稳定指标。** OpenAI 自 GPT-3.5 起用固定的内部代码留出集追踪 log-perplexity。GPT-3.5≈0.99 → GPT-4≈0.91 → 现在 <0.8；每一档都对应一次能力跃迁。
3. **下一窗口是 0.68–0.70。** 作者预期进入该区间会出现今天尚未测试的变革性能力；同时对齐审计会退化，模型的评估/训练意识会增强。
4. **真正的加速变量是 AI 参与「预训练算法研究」。** 数学上，模型在 ~0.76 时开始接连解决长期开放问题；作者据此外推：最早 2027 年，预训练基础研究可能出现「雪崩式」加速。但对齐与监控大概率跟不上。
5. **政策主张：共同标尺 + 共同红线。** 前沿实验室共享评估/红队规范与透明度标准；承诺在「理解与监控 AI 行为」达到具体里程碑（如 CoT 监控器的继任者）之前，不越过以 perplexity 等指标定义的共同上限。
6. **作者主动列出四个局限**：跨实验室不可比、与可操作评估脱节、忽略预训练之外的进步（CoT）、不知道数据固有熵的下限。

## 1. 论证起点：AI 是在「训练更好的压缩器」

文章开篇纠正一个直觉：现代 AI 看起来像过去几十年的软件工程——用多种技术分别实现不同能力；实际上，这些技术大多是在**驾驭**一个更简单、更不可解释的过程：在超大规模数据上训练一个压缩器。按 Shannon 信源编码定理，压缩与预测等价——最好的压缩器不是「记住」语料（不是 GZIP 文件），而是能预测未见数据。

由此得到一个简化的开发模型：

    造大计算机 → 预训练压缩器 → 用 prompting / finetuning / RL 诱导能力

算法进步会模糊阶段边界，但仍可区分两件事：

- **异类心智（alien mind）**：预训练造出的、具有难预测涌现能力的基底；
- **针对性诱导（elicitation）**：在具体场景里把潜在能力激发出来。

这个区分的政策含义是：**新能力的源头在基底，而不在后训练技巧**。因此，追踪基底的进度，比追踪不断饱和的下游 benchmark 更能预见下一次跃迁。

## 2. 一条曲线串起的能力阶梯

OpenAI 的追踪方式：在固定快照的内部代码留出集上，计算模型对下一个 token 的平均预测损失，即 log-perplexity（= cross-entropy loss）。文章公开了这条曲线（GPT-3.5 起）。

| 时间 | 模型 | log-perplexity | 图中标注的里程碑 |
| --- | --- | --- | --- |
| 2022-03 | GPT-3.5 | 0.986 | 流畅对话（ChatGPT 的基础） |
| 2023-03 | GPT-4 | 0.913 | 从「先思考再回答」中获益（CoT） |
| 2023-11 | GPT-4 Turbo | 0.909 |  |
| 2024-05 | GPT-4o / o1 | 0.904 |  |
| 2025-02 | GPT-4.5 | 0.876 |  |
| 2025-04 | GPT-4.1 / o3 | 0.902 |  |
| 2025-08 | GPT-5 / 5.1 | 0.893 |  |
| 2025-12 | GPT-5.2 | 0.888 |  |
| 2026-02 | GPT-5.3-Codex / 5.4 | 0.871 | 有用的编程者 |
| 2026-04 | GPT-5.5 | 0.848 |  |
| 2026-07 | GPT-5.6 Sol | 0.828 | 网络安全专家 |
| 2026-09 | GPT-6 Astra | 0.783 |  |
| 2026-11 | GPT-6.2 Astra | 0.761 | 解决一道千禧年数学难题 |
| 2026-11 | 内部模型（方块） | 0.732 | 非发布序列 |

> 图中阴影带为 0.68–0.70；曲线只连发布模型，方块是另一个内部模型。

作者的能力叙事：

- **GPT-2**：在旧数据集上刷新 perplexity SOTA，首次能稳定写出连贯段落；
- **GPT-3**：few-shot 上下文学习；
- **GPT-3.5（~0.99）**：流畅对话，撑起初代 ChatGPT；
- **GPT-4（~0.91）**：能有效使用 chain-of-thought，促成后来的推理模型；
- **<0.8**：网络安全、数学等领域出现变革性能力；
- **~0.76**：开始产出突破性数学结果（图中标注 GPT-6.2 Astra 解决一道千禧年数学难题）；
- **0.68–0.70**：作者预期下一次变革性能力的窗口。

最关键的一句判断：这些能力**不是预先设计出来的**，而是在特定 perplexity 阈值附近**涌现**的；在阈值到达之前，没人能先验地确定会出现什么。这也是为什么作者认为 benchmark 会过时（GPT-2/3 放到今天的 benchmark 上没有意义，当年的 benchmark 又在 GPT-4 附近饱和），而基底损失长期有信息量。

> 注：GPT-2/3 的数字来自旧数据集，与 GPT-3.5 之后的内部代码集不可直接连线；脚注 4 也承认「用代码预测能力代表 GPT-2/3 的通用能力并不合适」。这是「一条曲线」叙事里一个容易被忽略的缝隙。

## 3. 真正的变量：AI 参与「预训练算法研究」

常见的「AI 自我改进」讨论，聚焦于 AI 在研究全流程中增强/替代人类，终极形态是**完全递归自我改进**（full recursive self-improvement）：全自动、无人类监督。作者说，这种状态当然应避免（为了保持人类控制），但这个框架漏掉了关键的一步：

> 潜在的阶跃，可能来自 AI 参与开发流程中的一个特定环节——**预训练算法研究**。

他的证据来自数学：

- 过去一年多，OpenAI 持续把开放数学问题抛给模型，作为评估方式之一；
- 升级到 log-perplexity ~0.76 的模型后，模型开始产出突破性结果；
- 对做过数学的人来说，长期思考的问题在一个 Slack 线程里被接连解决，「非常超现实」。

由此外推：深度学习研究更依赖实验，但历史上也有简单洞见被忽视多年；**最早 2027 年，预训练基础研究可能出现类似的「雪崩」**。不过作者马上加了一个约束：对齐与监控也会受益于 AI，但进展**不太可能快到足以支撑预训练的全部潜在加速**。

脚注 5 补充了 2026 年曲线陡峭的归因：**主要**来自长期研究与算力投入，**只有较小部分**来自自动化；但最近的模型已经出现 AI 的实质性设计贡献。这个归因很重要——它既没有否认 AI 的贡献，也压低了「已经进入递归自我改进」的解读。

## 4. 政策主张：从「透明」到「共同上限」

公开内部数据的动机，作者写得很直白：这会让竞争对手更清楚 OpenAI 的位置，但**建立跨行业共享安全措施的紧迫性压过了竞争顾虑**；他希望这是「未来配速（pacing）努力的共同科学基础」的第一步。

他对近期形势的判断：

- 进入 ~0.68–0.70 窗口后，会出现今天没有测试过的变革性能力；
- 同时会出现**当前主流对齐审计方式的进一步退化**，以及**评估意识/训练意识的进一步增强**；
- 因此，前沿开发者应在模型**首次使用之前**，就评估与红队规范达成共享标准，并建立透明度标准。

具体主张是「共同上限」（common limits）：

1. 以 perplexity 等可跨实验室追踪的指标，设定**第一道开发上限**；
2. 开发者承诺在「理解与监控 AI 行为」达到某个**具体里程碑之前**不越过该上限；
3. 这个里程碑的例子是：**CoT 监控器的继任者**；
4. 行业标准应写明「第一道上限是什么」以及「越过它需要具体达成什么」，并由此形成逐级设限、持续配速的框架。

需要强调：这不是「停止研发」，而是**把能力阈值与监控能力挂钩的配速机制**。但文章没有回答治理层面的关键问题：谁来定阈值、谁来核查、违约如何处理、监控里程碑由谁认定。它更像一份「原则宣言」，而非可执行方案。

## 5. 作者自己列出的四个局限

| 局限 | 含义 | 为什么重要 |
| --- | --- | --- |
| 跨实验室不可比 | 评估数据集、训练数据配比不同 | 没有共享测量协议，「共同上限」就无法落地 |
| 与可操作评估脱节 | perplexity 不直接对应任何具体任务 | 关心它，只是因为它预示新能力 |
| 忽略预训练之外的进步 | CoT 不降低 perplexity，却大幅提升能力 | 单看基座损失会低估产品能力 |
| 不知道下限（floor） | loss 不可能到 0，数据固有熵未知 | 越接近下限，perplexity 进展越慢，但同等降幅对应更大智能跃迁 |

作者的结论是：尽管有这些局限，perplexity 经受住了时间考验，许多研究团队长期投入优化它，它持续带来通用智能的增益；因此值得标准化，也值得行业提高透明度。

## 6. 我的评论（个人观点，非作者原意）

1. **这是一篇「用指标换治理」的文章，不是技术论文。** 数据是实的，但「阈值—能力」的因果叙事是回溯性的：曲线上的里程碑是事后标注的，不能反过来当预测器。真正的检验在 0.68–0.70 窗口——如果到时候没有出现预期中的变革性能力，这套叙事就要重修。
2. **曲线并不单调，这恰恰暴露了单一指标的张力。** GPT-4.5（0.876）之后，GPT-4.1/o3（0.902）、GPT-5/5.1（0.893）的基座损失反而更高；产品模型会在基座损失与推理、代码、安全、成本之间做权衡。用一条「越低越好」的线讲故事，会抹掉这些权衡。
3. **跨实验室可比性是最大软肋，而且不只是「数据集不同」。** 数据配比、tokenizer、去重、污染、训练 token 数都会改变 loss；一旦指标成为门槛，就会产生「优化指标」的激励（换 tokenizer、混入评测分布、挑数据集）。文章承认了前者，没有充分讨论后者。
4. **「共同上限」缺少治理骨架。** 谁定阈值、谁核查、违约怎么办、监控里程碑由谁判定？在竞争格局下，自愿承诺是典型的囚徒困境。文章把它当作「第一步」可以理解，但离可执行还有距离。
5. **对齐门槛可能与作者自己的判断冲突。** 文章说对齐审计会退化、评估意识会增强，却把「CoT 监控器的继任者」当作放行条件；如果监控本身随模型变强而失效，这个门槛要么定得太低，要么永远达不到。
6. **2027「雪崩」是全文最强也最脆的断言。** 数学有廉价验证和大规模采样；预训练算法研究实验周期长、信号噪声大、算力昂贵。类比成立的方向对，速度未必。
7. **对从业者的启示**：如果基座损失真是主变量，长期复利在数据质量、tokenizer、架构与算力的组合，而不是单点技巧；但产品差距仍主要由 post-training / RL / agent harness 的「诱导」能力决定——CoT 就是反例：它不降 perplexity，却能大幅提升能力。
8. **对安全的启示**：可测的能力阈值 + 难定义的理解里程碑，是这套方案的结构性张力。更现实的路径也许是「能力阈值 + 一组可审计的行为评估 + 事件报告」，而不是押注单一监控技术。

## 7. 金句（原文 + 译文）

1. **"training a compressor (or, equivalently, predictor) on a very large corpus of data."**
   → 在超大规模语料上训练一个压缩器（等价地说，一个预测器）。

2. **"It was not at all clear a priori that these capabilities would appear once those precise perplexity thresholds are met."**
   → 在那些精确的困惑度阈值被达到之前，根本无法先验地确定这些能力会出现。

3. **"As soon as 2027 we could conceivably see a similar avalanche of fundamental pretraining advancements."**
   → 最早 2027 年，我们就有可能看到基础预训练进展的类似雪崩。

4. **"Progress is unlikely to be fast enough to allow the full potential acceleration in pretraining."**
   → （对齐与监控的）进展不太可能快到足以支撑预训练的全部潜在加速。

5. **"Common limits... until our ability to understand and monitor AI behavior achieves a certain concrete milestone (such as the development of a successor to CoT monitors)."**
   → 共同上限……直到我们理解与监控 AI 行为的能力达到某个具体里程碑（例如 CoT 监控器的继任者）。

6. **"We should expect further degradation in currently dominant forms of alignment auditing and further increases in evaluation and training awareness."**
   → 我们应预期当前主流对齐审计方式进一步退化，以及评估意识与训练意识的进一步增强。

## 8. 术语速查

| 术语 | 含义 |
| --- | --- |
| perplexity（困惑度） | 模型平均预测下一个 token 的难度；越低越好 |
| log-perplexity / cross-entropy loss | 实践中追踪的等价量；文中数字均为该值（底数文中未注明，通常按 nats/token 计） |
| compressor / predictor | 能预测未见数据的模型；不是记忆语料的压缩文件 |
| elicitation（诱导） | 用 prompting / finetuning / RL 把预训练潜在能力激发出来 |
| alien mind（异类心智） | 作者此前提出的说法：具有难预测涌现能力的 AI |
| recursive self-improvement | 完全自动化、无人类监督的自我改进 |
| CoT monitor（思维链监控器） | 通过模型思维链监控其行为的对齐手段 |

## 9. 发表后的反响（截至 2026-11-24）

> 说明：以下只收公开可检索的直接表态。媒体常把 2026 年 9 月 Amodei「We Must Pace the Frontier」获得的支持算到这篇文章头上，那是另一场事件。

### 9.1 明确支持：OpenAI 系 + 微软

| 人物 | 身份 | 关键表态 |
| --- | --- | --- |
| Sam Altman | OpenAI CEO | 称这是“the most important thing OpenAI has published in awhile”，并希望业界能就 common limits 达成一致。（[X, 11/22](https://x.com/sama/status/2124551594691932405)） |
| Greg Brockman | OpenAI 总裁 | 称文章“can be the foundation of industry-wide shared safety bars”，并复述“第一道上限 + 具体退出条件 + 逐级配速”的机制。（[X, 11/22](https://x.com/gdb/status/2124582684840399311)） |
| Mark Chen | OpenAI 首席研究官 | “We want frontier labs to operate under *common* limits, backed by *shared* safety cases.” 承认公开数字“scary”，因为长期视为敏感 IP；呼吁第三方评估者介入。（[X, 11/22](https://x.com/markchen90/status/2124592748099665943)） |
| Mustafa Suleyman | 微软 AI CEO | “We fully support OpenAI's transparency initiative.” 并报出 MAI-2 Base 在 OpenAI 评测集上的 log-perplexity **0.819**，与 GPT-5.6 Sol 基座“same ballpark”；同一 tokenizer，所以可直接比较；未来建议改用 BPB，且需要不同评测集。（[X, 11/23](https://x.com/mustafasuleyman/status/2124663594449318173)） |

微软这一条是唯一一个用数字跟进的外部实验室：它把一篇原则宣言变成了可验证的行业规范雏形。但它同时暴露了三个前提——同一 tokenizer、同一评测集、OpenAI 愿意授权；缺一个，“可比”就不成立。Suleyman 自己补了 BPB 和“需要不同评测集”的补丁。

### 9.2 暧昧、借势与保留

- **Elon Musk（xAI）**：在 Pachocki 帖下回复：“Imagine what super intelligence will be like when we hit an exawatt. This will be like a caveman playing with sticks.”（[Benzinga 转述](https://longbridge.com/en/news/306475207)）这是算力/超级智能的感叹，不是对“common limits”的背书。部分媒体把他对 9 月 Amodei 计划的表态混到这篇上，不可靠。
- **Nathan Lambert（Ai2 / Interconnects）**：只留下一句 “OpenAI makes the best, most lasting plots.”（[X, 11/22](https://x.com/natolambert/status/2124562346353840302)）像是夸图，也像是讽刺；没有实质政策表态。
- **swyx（Latent Space）**：认为 OpenAI 是在用 perplexity 替代 FLOPS 做行业自我监管；称这张图是“过去四年最美的叙事方式之一”，并把 post-training 争议称为 pretraining 主线的“B plot”。（[X, 11/22](https://x.com/swyx/status/2124583459356373052)）分析性肯定，不是政策背书。

### 9.3 技术共同体的质疑

- **Hacker News**（[讨论帖](https://news.ycombinator.com/item?id=50601525)，约 650 分、400+ 评论）集中在三件事：
  1. **指标**：能力是否真的在某个 perplexity 阈值“涌现”？GPT-4 与 o3 的 perplexity 接近，但 o3 明显更强；“perplexity on what?”；perplexity 可能被记忆/污染“做低”。
  2. **数据**：曲线来自 OpenAI 内部代码库，可能只是“代码训得更多”的故事；一旦成为门槛，就有换 tokenizer、混评测分布的激励。
  3. **动机**：“This is just an ad.” 支持者则认为，至少它给出了可讨论的具体数字。
- **Tomek Korbak（前 OpenAI 安全研究员）**：欢迎公开预训练指标，认为 pretraining loss 是“限速器”的好候选（架构无关、比“串行深度”一类限制更耐久）；但警告“words vs actions”，并提到 OpenAI 近期有“empty promises”。（[X, 11/22](https://x.com/tomekkorbak/status/2124570745959514382)）
- **Charlie O'Neill（Baseten）**：据 Medium 综述转引，OpenAI 最有影响力的动作应该是把 held-out perplexity 评测集放出来；那样会在安全贡献上“instantly leapfrog Anthropic”。
- **Bret Kerr（“Where You Put the Gate”）**：据 [ai-handwerk.de 的转述](https://ai-handwerk.de/openai-perplexity-and-intelligence/)，Anthropic 和 Google DeepMind 无法在 OpenAI 的私有测试集上测自己的模型；公开数据让对手知道 OpenAI 站在哪，却不让他们站到旁边比较。这是“共同上限”最结构性的批评。

### 9.4 沉默的一方

截至 11 月 24 日，公开检索没有找到 **Dario Amodei（Anthropic）、Demis Hassabis（Google DeepMind）、Yann LeCun（Meta）、Ilya Sutskever（SSI）、Andrej Karpathy** 对这篇长文的直接回应；公开检索也未见 Anthropic 与 Google DeepMind 的官方账号表态。中文科技媒体和评论者已注意到这一点，并公开呼吁两家公布自己的预训练困惑度。

需要区分三件事：

1. 文章引用了 Amodei 9 月的 “We Must Pace the Frontier”，那是**背景**，不是 Amodei 对本文的回应；
2. 9 月 Amodei 计划曾获得 Altman、Hassabis、Musk 的公开支持，并引发“cartel”批评——这是**另一场事件**；
3. 据 Medium 综述，LeCun 那个周末的炮火主要打向 Amodei 的 safety framing，而非 Pachocki 这篇。

### 9.5 中文舆论

凤凰科技、腾讯新闻、36 氪、IT之家等均做了报道，问题集中在：谁定阈值、谁核查、违约怎么办；“2027 雪崩”是严肃预测还是营销；跨实验室到底可不可比。尚未见到中国头部 AI 实验室负责人直接回应。

**一句话总结**：这是一次 **OpenAI 主导、微软第一个用数字跟进** 的透明度倡议；它还不是行业共识。最值得注意的不是谁点赞，而是两个事实——微软真的报了一个可比的数，Anthropic 和 Google DeepMind 没有接话。前者证明“共同标尺”至少可操作，后者说明“共同上限”离“共同”还很远。

## 10. 参考与延伸

- 原文：[Perplexity and Intelligence](https://openai.com/index/perplexity-and-intelligence/)（2026-11-22）
- [Shannon's source coding theorem](https://en.wikipedia.org/wiki/Shannon%27s_source_coding_theorem)：压缩与预测等价的数学基础
- [An Alien Mind](https://openai.com/index/an-alien-mind/)：Pachocki 此前关于「异类心智」的文章
- [Dario Amodei, "We Must Pace the Frontier"](https://darioamodei.com/post/we-must-pace-the-frontier)：作者在文中引用的「共享配速」论述
- [What affects chain-of-thought monitorability across the model lifecycle?](https://alignment.openai.com/cot-monitorability-model-lifecycle)：文中「对齐审计退化」的链接
- 同周 OpenAI 相关文章：[Research acceleration: The jagged frontier](https://openai.com/index/research-acceleration-jagged-frontier/)、[Managing frontier AI risk: From training to deployment](https://openai.com/index/managing-frontier-ai-risk/)
