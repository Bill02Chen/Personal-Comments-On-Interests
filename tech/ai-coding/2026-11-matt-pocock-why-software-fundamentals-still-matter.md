# 视频整理｜AI 时代的基本功：代码是 Agent 的第一提示词（Matt Pocock @ React Advanced 2026）

> **一句话**：代码不只是「生产出来的东西」，它是 agent 运行的环境与第一提示词；代码库越健康，agent 产出越好。所以 AI 时代软件基本功的杠杆反而更高——学会「踩刹车」（自动化检查 → 自动化评审 → 人工评审），比一味踩油门更重要。

## 视频档案

| 项目 | 内容 |
| --- | --- |
| 标题 | Matt Pocock 演讲：AI 时代的基本功丨代码是 Agent 的第一提示词 |
| B 站链接 | https://www.bilibili.com/video/BV1gP4n6LEdF |
| UP 主 | ChHsich（AI 辅助转录 + 翻译并经人工校对） |
| 发布时间 | 2026-11-07 |
| 时长 | 30:51（含现场问答） |
| 讲者 | **Matt Pocock**，Total TypeScript / AI Hero 作者，前 XState 核心团队成员 |
| 场合 | **React Advanced 2026，伦敦**（2026.10.23–26） |
| 原始页面 | https://gitnation.com/contents/why-software-fundamentals-still-matter |
| 大会官推 | https://x.com/ReactAdvanced/status/2118658173112287645 |
| 配套仓库 | https://github.com/mattpocock/skills （skills 合集：aihero.dev/skills） |
| 整理依据 | B 站视频音轨全文转写（whisper large-v3-turbo）＋ B 站简介 / 原始页面交叉核对 |

---

## TL;DR

1. **「代码很便宜」只说对了一半**：变便宜的是**生产**，没变便宜的是**维护**。这句话还暗示「代码低质、可抛弃、低价值」——这是错的。
2. **代码是 agent 的第一提示词，也是它运行的环境。** 健康的代码库会限制 agent 的设计空间，把它逼到正确的路径上；代码越烂，agent 越容易乱写。
3. AI 承诺的核心是**速度**，于是出现了「**软件工厂**」：油门（bug 报告、工单、支持请求自动触发 agent 产出）＋ 刹车。学开车先学刹车，只有油门会直接冲进「垃圾内容炮」（slop cannon）。
4. **三层刹车**：自动化检查（确定性的 lint / 类型 / 测试）→ 自动化评审（子代理按编码规范评审 diff）→ 人工评审（最贵、是瓶颈，要优化到最少）。
5. **测试是自动化检查里最重要也最容易骗人的一环**。三种坏测试：同义反复、对结构敏感、根本不可能失败。
6. **代码设计决定测试能不能作弊**：深模块（简单接口、隐藏实现）让测试只能通过接口进行，从结构上减少结构敏感测试；把 React 里散落的状态收进 reducer / 状态机获得**局部性**，让 agent 只看一个文件。
7. **人工评审是最高杠杆的地方**：同一条评论绝不写第二遍，把品味「上游化」成 lint 规则和 coding-standards.md；用**引导词（leading words）**把一页规范压缩成一个术语（error boundary、optimistic update、virtualize）。

---

## 演讲结构

    软件工厂 (Software Factory)
    油门 Accelerators                        刹车 Brakes
    ┌──────────────────┐      ┌──────────────────────────────────────┐
    │ bug 报告 / 工单   │      │ ① 自动化检查（lint / 类型 / 测试）    │
    │ 功能请求 / 支持   │ ───▶ │ ② 自动化评审（子代理 reviewer）        │
    │ → 自动触发 agent  │      │ ③ 人工评审（最贵、是瓶颈、要优化）     │
    └──────────────────┘      └──────────────────────────────────────┘
                                   目标：Stop the slop

## 章节地图（B 站章节，与音轨时间戳一致）

| 时间 | 章节 |
| --- | --- |
| 0:00 | 开场与论点 |
| 0:43 | 「代码很便宜」的陷阱 |
| 2:01 | 软件工厂与三个刹车 |
| 4:37 | lint 规则与摩擦 |
| 6:03 | 三种坏测试 |
| 9:15 | 深模块设计 |
| 10:44 | 状态收拢与局部性 |
| 13:19 | 自动化评审 |
| 16:48 | 人工评审与引导词 |
| 20:28 | 现场问答 |

---

## 一、开场：AI 都干活了，我们为什么还来大会？

> "My thesis is that software fundamentals have actually never been higher leverage, and particularly understanding React has never been higher leverage than it is today."

- 大家花自己的钱或公司的钱来参会，而「AI 把活都干了」——那学习还有什么意义？
- 讲者的论点：**软件基本功的杠杆从未这么高**，尤其是理解 React。
- 因为：代码不只是产出物，它还是 **agent 的提示词与环境**。

---

## 二、「代码很便宜」这句话错在哪（0:43）

- 流行说法：**Code is cheap.** 代码产量前所未有地高——**这一点是真的**。
- 但他「讨厌这句话」，因为：
  1. 代码不只是**生产**，还有**维护**；
  2. 「便宜」暗含「低质、一次性、低价值」的意味；
  3. 更关键的是：**代码是 agent 的首要提示词（primary prompt）**，是 agent 运行的环境。
- 推论：如果代码库很健康，它会**约束** agent，只允许走某些路径，**限制它的设计选择**，产出就会更好；而且 agent 可以按预算横向扩展，于是价值可以放大。
- 所以他强调：**正因为代码重要，软件基本功才重要；理解 React 比以往任何时候都值钱。**

> "If you have a very healthy code base that constrains the agent into only certain paths, restricts its design choices, you're going to get better output from your agents."

---

## 三、软件工厂：油门与三个刹车（2:01）

- AI 的核心承诺是**速度**，于是有了当下的热词「**软件工厂（software factory）**」。
- 软件工厂分两部分：
  - **油门（accelerators）**：不是你在提示 agent，而是**系统在提示 agent**——bug 报告、功能请求、支持工单自动进来，触发 agent 产出代码或复现。
  - **刹车（brakes）**：让 agent 慢下来、提升产出质量的东西。
- 学开车先学什么？**先学刹车。** 只有油门，你会直接冲进 "slop cannon"（垃圾内容炮）。
- 反直觉的地方：**刹车让你更快**。减少垃圾产出、让进入人工评审的代码质量更高，整体流速才会提升。

### 三层刹车

| 层级 | 是什么 | 成本 | 作用 |
| --- | --- | --- | --- |
| ① 自动化检查 | lint、类型检查、测试等确定性检查 | 极低（花 CPU，不花 token） | 上线前抓掉大量 bug |
| ② 自动化评审 | 用 agent 检查 diff，执行编码规范 | 低 | 抓住「漏网之鱼」，承接编码规范 |
| ③ 人工评审 | 真正的人看代码 | 最高，且是当前**瓶颈** | 品味、判断、系统级思考 |

> 目标一句话：**Stop the slop.** 减少 agent 垃圾产出，让人工评审只处理高质量内容。

---

## 四、刹车一：自动化检查（4:37）

### 4.1 检查便宜，所以可以堆满

- 自动化检查花的是 **CPU 周期**，不是 token，因此可以「堆得很高」。
- 而且过去几年这些工具越来越快：TypeScript 被移植到 Go（后又出现 Rust 版本）、**OXLint** 等工具出现——现在完全可以把大量检查塞进应用里。

### 4.2 把 lint 规则开到最大

- 以前 lint 规则有一种「**社会契约**」：不要写刁难同事的规则。讲者自嘲：早年给老板写了一条自定义 lint 规则，每次他提交都报错，老板问「这是给我写的吗？」——「是的。」这违反了社会契约。
- **但 agent 不一样：agent 从摩擦中受益。** 减少 agent 可选的设计路径，是巨大的收益。
- 他自己在用的：**OXLint、React Doctor、React Compiler 的规则**，以及仍在写的自定义 lint 规则。而且「现在写自定义规则不用觉得内疚了」。

### 4.3 类型检查很好，但测试才是关键

- 类型检查很棒，但**测试真正执行了代码**，而不是只看源码。
- 测试很难，agent 并没有让它变简单，因为**测试非常容易撒谎**：
  - 问：CI 全绿，是否意味着代码可以合并？——**不是。**
  - 自动化检查一直有洞（holes）。

---

## 五、三种坏测试（6:03）

### 5.1 同义反复（tautological tests）

- 测试只是**重申实现**。例如实现里定义了常量 X_POST_CHAR_LIMIT = 280，测试就写 expect(X_POST_CHAR_LIMIT).toBe(280)。
- 问题：
  - 对代码结构敏感——改个常量名、挪个位置，测试就要改；
  - 实际上没有断言任何行为，只是在描述代码的形状；
  - 没有真正执行什么。
- 讲者吐槽：「**Opus 5 特别沉迷写这种测试**，不知道为什么。」

### 5.2 对结构过度敏感（too structure sensitive）

- 例子：在没有配置 React Testing Library / JSDOM 的环境里，测试干脆**读取源文件**，断言两个按钮在源码中的顺序（"Content plan" 在 "Title video" 前面）。
- 任何写过 React 的人都知道这是垃圾：把内容抽成常量、调换渲染顺序，测试就挂了——但它根本没验证页面行为。

### 5.3 根本不可能失败（tests that simply cannot fail）

- 例子：测试一个真实的 useAudioBoost hook，把 AudioContext 直接 **stub** 掉，忽略它所有复杂的校验规则。
- 结果：**生产中会失败的路径，在测试里永远不会失败**。这种测试给了虚假的安全感。

### 5.4 可直接复制的自检 prompt

> "Explore my test suite and identify tests or groups of tests that are tautological, too structure sensitive, or whether mocks don't match reality."

（大意：扫描我的测试套件，找出同义反复的、对结构过度敏感的、以及 mock 与现实不符的测试。）

---

## 六、深模块：让自动化检查「难以作弊」（9:15）

> "If automated checks can be cheated, how do we make them harder to cheat? ... This is where codebase design comes in."

- 光靠 prompt 约束不够，**要用代码设计让坏测试在结构上难以存在**。
- 引用 John Ousterhout《A Philosophy of Software Design》（软件设计的哲学）的**深模块（deep modules）**：
  - **深模块**：极简接口 + 大量被隐藏的实现。典型例子 JSON.parse——人人会用、接口简单，内部庞杂但你从不需要关心。
  - **浅模块**：agent 最爱写的那种——导出所有函数、导出所有常量、内部一切可测。
- 为什么深模块能治坏测试？**测试只能通过接口观察模块**，无法伸手进去断言内部结构，于是「结构敏感测试」自然减少。
- 而 React 的默认写法**天然倾向浅模块**：组件可以任意嵌套、任意组合——这是 React 这些年的巨大优势，但代价是「**状态到底住在哪里？**」

---

## 七、状态收拢：reducer / 状态机换来局部性（10:44）

- 讲者背景：XState 核心团队，多年沉迷状态管理。「现在终于能在 AI 时代再讲一次这个话题。」
- 问题：React 组件里蓝色（有状态）的部分散落在应用各处；**agent 很难跟踪随时间变化的多处状态**。
- 解法：把所有状态**抽出来放进 reducer**（「好的老 Redux」——不一定是 Redux，任何集中式状态管理都行）。
- 收益是**局部性（locality）**：
  - 所有状态都在一处 → **agent 只需要看一个文件**；
  - 不需要浏览器就能测试；
  - 变成纯函数，独立存在；
  - agent 容易加 case、容易理解整个流程。
- 配套 skill：**improve-codebase-architecture**（他最喜欢的 skill 之一），可以把你「从自己手里继承来的」糟糕 vibe-coded 应用重构出深模块，并给出改进报告（HTML），你可以挑着执行。
- 推荐 prompt：

> "Improve codebase architecture. Look for opportunities to turn my messy front-end state into reducers or state machines."

---

## 八、刹车二：自动化评审（13:19）

### 8.1 核心原则：不要信任实现者来执行编码规范

- **实现者 agent 严重超载**：它要探索代码库、写实现、调试自己写的代码、修被弄挂的自动化检查。
- **评审者 agent 严重欠载**：它只接收 diff（几乎不需要探索），不需要实现、不需要调试——像「《记忆碎片》里醒来的主角」，带着干净的上下文看代码。
- 所以应该把**编码规范的执行**交给评审，形成两阶段分工：
  - 实现阶段只求「能跑」（相当于 red-green 里的 red、green）；
  - 评审阶段再把它变「好」（相当于 refactor）。
- 结论：**不要把编码规范放进 AGENTS.md**（每次会话都加载的「方向盘」文件，实现者会直接忽略，因为它已经太忙了）；要放进 **coding-standards.md**，由 code review skill 读取。这样「每一分投入的回报」高得多。
- 配套 skill：**code-review**——在一个或多个子代理里跑 reviewer，给它们干净的上下文窗口。

### 8.2 不要外包给通用工具

- 不要用 Cursor 的 BugBot、CodeRabbit 这类通用代码评审工具：它们**不适配你的应用**。
- 自动化评审的价值在于：**防止 agent 反复犯你亲眼见过的那些错**。所以规则要基于你自己踩过的坑。
- coding-standards.md 里可以写 linter 抓不到的东西：
  - useEffect 用得是否合理；
  - 状态是否放进 reducer；
  - server components / client components 的复杂规则。

---

## 九、刹车三：人工评审与引导词（16:48）

### 9.1 人工评审的杠杆：只写一次评论

- 人工评审不只评审代码，**也在评审「生产代码的系统」**。
- 原则：**同一条评论永远不要写第二遍**。写第二遍就是浪费——应该把这条评论里的品味、直觉**上游化**到自动化检查和自动化评审里。
- 这是一个**正循环**：每次人工评审都往系统里注入一点你的品味 → 下次人工评审更快更少。
- 配套 skill：**retro**——对一次 coding agent 会话做复盘，建议新增哪些自动化检查和自动化评审，让下一次人工评审更轻松。
- 成果形态：自定义 lint 规则、coding-standards.md 里那些「你非常在意但再也不想跟 agent 重复说」的规则。

### 9.2 引导词（leading words）：一个词顶一页规范

对比三组「啰嗦规范 vs 引导词」：

| 啰嗦写法 | 引导词写法 |
| --- | --- |
| 「当用户点赞时，立即更新计数，不要等服务器返回，然后……」 | 「点赞、关注、收藏都是 **optimistic updates**（乐观更新）」 |
| 「列表很长时滚动很慢……」 | 「**Virtualize the list**」（虚拟化列表） |
| 「一个 widget 抛错时，不要让整页变空白，要在每个 widget 外面捕获……」 | 「每个 dashboard widget 都在自己的 **error boundary** 内」 |

- 原理：**领域词是你知识的极致压缩**。agent 认识 "optimistic updates"，但不认识你那一大段自造描述。
- 更关键的是：**agent 会在自己的思考轨迹里使用这些词，并不断强化它们**——这就是「引导词」（words the agent thinks with）。
- 所以你的 React 词汇量，直接决定了你能写出多紧的规范、把 agent 引导到多正确的路上。

### 9.3 从历史数据挖规则

> "Take a look at as many GitHub PR comments as you have access to from my team. Create suggestions for automated checks and updates to coding-standards.md."

- 把团队过去的 PR 评论当作数据矿，自动生成自动化检查与规范更新——让系统越用越聪明。

---

## 十、现场问答（20:28）

### Q1：AI 会不会压制 AI 之外的创新（新语言、新概念）？

- 反例就是证据：TypeScript 团队花约 **两年**把编译器移植到 Go，带来约 **10 倍**提速；**Theo** 几周内就做了自己的 Rust 版本；**OXLint 背后的团队**也出了自己的版本。
- 生产代码的劳动力成本大幅下降，没有理由不出现大规模创新；尤其 dev tool 领域都在追求「更快」，而更快就意味着更快的自动化检查，甚至尝试**更适合 AI 的新语言**（他提到一门内置形式化验证、面向 AI 的新语言，录制中名字听辨不清）。
- 结论：**现在创新比过去更多，不是更少。**

### Q2：新术语从哪来？怎么保证新人学到？

- **来自对话。** 过去几年大量讨论 AI，讨论的**数量**和**质量**都在提升。
- 他自己「痴迷于 process」（如何让软件工程的流程更好），在讨论过程中学到了大量新词与新思考。
- 新词会不断涌现：loop engineering、prompt engineering、context engineering……

### Q3：AI 检查 AI，信任够吗？

- 先定义「够」——是指可以**取消人工评审**吗？有些变更类型可以，有些不行。
- 用**单向门 / 双向门（one-way door / two-way door）** 区分：
  - 金融、医疗、安全关键、对外暴露的变更 → **单向门**：合并后就回不去了，**必须有人类评审**；
  - 可轻易回滚的变更（比如按钮改名）→ **双向门**：可以让 agent 自动合并，信任自动化检查与评审。
- 他甚至有一个 **PR skill**，专门判断一个 PR 是单向门还是双向门。
- 更高阶的做法：**好的代码设计可以把单向门变成双向门**——比如不要直接删数据库字段，而是先 **deprecate**、保留数据，于是随时可回滚。
- 关键变量不是「评审工具多强」，而是**这次改动的影响半径（blast radius）**。

### Q4：CI 全绿、一切看起来都正常，怎么发现问题并转成 prompt/规则？

- 靠 **retro skill**：对 coding agent 会话做复盘。
  - 即使是一次**很成功**的运行，也能挖出很多问题；
  - 它不只看自动化检查与评审，还看**工具经济性（tool economy）**（工具调用是否高效）和**仓库可导航性（navigation）**；
  - 它会读 agent 实际发出的 tool calls——人自己逐条读这些 trace 非常痛苦，交给 agent 读则价值巨大；
  - retro agent 能发现你**根本想不到要去看**的问题，然后给出「一个很简单的改动，让之后所有任务都受益」。

### Q5：AI 承担了这么多，新人怎么练基本功？

- 编程一直有两个面向（借用 Ousterhout 的术语）：
  - **战术面（tactical）**：日常、落地、写代码；
  - **战略面（strategic）**：长期思考、判断下一次改动、维护代码库的长期健康。
- agent 是「**战术龙卷风（tactical tornadoes）**」：只盯当下，不考虑长期。所以人更需要战略思维。
- 给新人的路径：
  1. 先写**个人软件**（但人对自己写的软件太宽容，容易掩盖问题）；
  2. 再做**内部软件 / 团队 dev tool**：拥有较大范围的自主权，用户基数小，犯错伤害有限；
  3. 最后才上生产面向公众的软件。
- 一句话：**把新人当作团队（和 agent）的平台来培养**——给他们大面积的责任，但放在伤害可控的地方。

---

## 关键概念表

| 英文 | 中文 | 含义 |
| --- | --- | --- |
| software factory | 软件工厂 | 由系统自动触发 agent 产出代码的流水线 |
| accelerator / brake | 油门 / 刹车 | 提升产出速度的机制 / 提升产出质量的机制 |
| slop | 垃圾内容 | agent 批量生成的、低质量代码与 PR |
| tautological test | 同义反复测试 | 只是重申实现、不验证行为的测试 |
| structure sensitive test | 结构敏感测试 | 断言源码结构而非行为的测试 |
| deep module | 深模块 | 简单接口隐藏大量实现的模块设计（Ousterhout） |
| locality | 局部性 | 状态集中在一处，改一处即可理解全局 |
| leading words | 引导词 | agent 会在思考中复用的领域术语，可压缩规范 |
| one-way / two-way door | 单向门 / 双向门 | 不可回滚的改动 / 可回滚的改动 |
| blast radius | 影响半径 | 一次改动出问题时波及的范围 |
| tactical tornado | 战术龙卷风 | 只顾当下、不管长期健康的工程师（或 agent） |

---

## 金句（原文 + 译文）

1. **"Code is a primary prompt for your agent. It's the environment your agent operates in."**
   → 代码是 agent 的首要提示词，是它运行的环境。

2. **"If you have a very healthy code base that constrains the agent into only certain paths... you're going to get better output from your agents."**
   → 健康的代码库把 agent 限制在少数正确路径上，产出自然更好。

3. **"What's the most important thing when you're learning to drive? Do you learn about the accelerator first or do you learn about the brakes?"**
   → 学开车先学什么？油门还是刹车？

4. **"If you only have accelerators, you're just going to drive yourself into a slop cannon."**
   → 只有油门，你会一头冲进垃圾内容炮。

5. **"Human review is basically the bottleneck at the moment."**
   → 人工评审就是当下的瓶颈。

6. **"You should never write the same comment twice."**
   → 同一条评审意见，永远不要写第二遍。

7. **"Leading words are words that the agent thinks with."**
   → 引导词，是 agent 用来思考的词。

---

## 我的思考与落地清单（个人观点，非讲者原意）

### 三个判断

1. **「代码便宜」是产出侧的真相，不是资产侧的真相。** 代码一旦进入长期维护，成本结构就完全反过来。AI 放大的是「生成」，没有放大「理解」。
2. **三层刹车的价值排序会变。** 自动化检查便宜到可无限堆；自动化评审是新的「规范执行层」；人工评审的稀缺性反而上升——因为它是唯一能判断「影响半径」和「品味」的层。
3. **深模块 + 状态收拢，本质是给 agent 降低「可作弊面」。** 这比写更多 prompt 更根本：prompt 是请求，结构是约束。

### 可直接照做的清单

- [ ] 把 lint 开到最大（oxlint + 类型 + React Compiler 规则），接受「少数误报」的成本。
- [ ] 把 coding-standards.md 从 AGENTS.md 里拆出来，只给评审子代理读。
- [ ] 建一个 reviewer 子代理：只吃 diff、干净上下文、只做评审。
- [ ] 用一次 retro：让它读最近几次 agent 会话的 tool calls，找出「每次都卡住的地方」。
- [ ] 把团队 PR 评论喂给 agent，生成 lint 规则与规范条目。
- [ ] 检查自己项目里的测试：同义反复 / 结构敏感 / 不可能失败，各清理一批。
- [ ] 把散落在组件里的状态收进 reducer / 状态机，先从一个页面开始。
- [ ] 把规范里的长句改写成**术语**（error boundary、optimistic update、virtualize、idempotent……），术语表就是最好的 prompt。

> 与 Max Kanat-Alexander 那场的关系：Max 回答「**该投入多少严格度**」（一次性 vs 持久软件的光谱），Matt 回答「**严格度怎么落地**」（三层刹车 + 深模块 + 引导词）。两篇连读，是一套完整的 AI 时代工程质量方法论，见 [另一篇笔记](./2026-11-max-kanat-alexander-does-code-quality-even-matter.md)。

---

## 参考链接

- B 站视频：https://www.bilibili.com/video/BV1gP4n6LEdF
- 原始演讲页（GitNation）：https://gitnation.com/contents/why-software-fundamentals-still-matter
- 大会官推：https://x.com/ReactAdvanced/status/2118658173112287645
- 讲者转发：https://x.com/mattpocockuk/status/2118663407616606637
- Skills 仓库：https://github.com/mattpocock/skills
- 讲者：Matt Pocock（Total TypeScript / AI Hero）
- 提到的方法论来源：John Ousterhout《A Philosophy of Software Design》（深模块、战术 vs 战略）
