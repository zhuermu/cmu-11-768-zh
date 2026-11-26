# 第 16 讲 · agent 和未来的工作：能替人干多少活

> CMU 11-768 AI Agents（2026 秋）中文精读 · 配套课件
> 配套视频（16 分钟）：B 站 / 视频号 / 抖音 / 小红书搜索「竹二木SI」 · 整份幻灯片：[l16-幻灯片.pdf](./l16-幻灯片.pdf)
> 原课程：[cmu-agents.com](https://cmu-agents.com)，课程主讲 Graham Neubig、Daniel Fried；本讲课表名 Agents and the Future of Work，讲者 Zora Wang。本课件是非官方的学习整理。
> **这一讲原课程没有公开录像**，内容根据原课件和课表列出的推荐阅读整理：必读的 5 篇论文（agent 研究和真实工作的错位、GDPval、CollabSkill、TAHI、Future of Work with AI Agents），以及课件页脚引用的论文和博客。每节末尾注明出处。课件和原文说法不一致的地方，两边都写。

## 这一讲你会学到

学完这一讲，你应该能做到：

1. 用几组真实数字说清 agent 在工作里的现状：用的人多了，能做的任务长了，基准分数高了；但真实项目几乎做不成，研究集中在编程，不会跟人配合，塞不进工作流。
2. 说出研究者让 agent "能干活"要准备的三样东西：**环境**（environment）、**任务**（task）、**评测**（evaluation），以及每一样目前常见的做法。
3. 解释**构念效度**（construct validity）是什么，为什么一个基准分数高不等于 agent 真有这个本事。
4. 比较几种评测开放任务的办法：跑测试、规则检查、专家写评分表、模型写评分表、评分表跟着训练演化、从人的反馈里长出评分表；说出 **LLM 裁判**（LLM-as-a-judge）的偏差和基准会饱和、被污染的问题。
5. 用"人的参与度量表"（Human Agency Scale）的五个等级描述人和 agent 的协作方式，区分**自动化**（automation）和**增强**（augmentation）。
6. 说出几项关于工作会怎么变的研究结论：多少人会受影响、员工自己想交出去哪些活、新任务会出现、刚入行的人可能更难。

建议先看一遍视频，再对照本课件逐节复习。每节开头的图就是视频里对应的画面。

### 接上一讲

上一讲讲的是怎么给 agent 定流程、怎么看住它跑出来的大量记录。前面十几讲一直在讲怎么把 agent 做得更强。这一讲退一步，看看把它放进真实的工作里到底怎么样，研究者又是怎么衡量"能不能干活"的。这一讲没有代码，是一次综述，论文很多，我挑每篇最能说明问题的一两个数。

### 这一讲的路线

![本节概览：现实检查、怎么让它能干活、不止能干](./images/p02.jpg)

| 节 | 回答的问题 | 课件 |
|---|---|---|
| 1 现实检查 | agent 现在能做什么、做不到什么 | p.2–13 |
| 2 任务从哪来 | 研究里的任务是怎么收集的，测的是不是该测的 | p.15–25 |
| 3 环境怎么搭 | agent 干活的那台电脑里装什么、放什么数据 | p.26–32 |
| 4 怎么评 | 做完了，怎么判断做得好不好 | p.33–45 |
| 5 能干还不够 | 人怎么参与，效率、安全、工作的乐趣 | p.46–53 |
| 6 人的工作会怎么变 | 多少人受影响，哪些活想交出去，新任务和新技能 | p.54–61 |
| 7 源码解读 | 一个真实基准的评测脚本到底检查了什么 | p.37 |

全讲用课件 p.16 那张"环境 · 任务 · 评测"三件套图当路标，第 2–4 节各讲其中一件。

---

## 1 现实检查

### 1.1 用的人多了

![每天用 Claude 做各类编程任务的员工比例](./images/p03.jpg)

Anthropic 调查了自己的员工每天拿 Claude 干什么。原文的数字：

| 用法 | 每天这样用的员工比例 |
|---|---|
| 调试 | 55% |
| 读懂代码 | 42% |
| 实现新功能 | 37% |

员工自己估计，一年前在 28% 的工作里会用到它、效率提升 20%；现在是 59% 的工作、提升 50%。这些是员工自报的数，不是测出来的。

出处：Anthropic《How AI is transforming work at Anthropic》；课件 p.2。

### 1.2 能做的任务长了

![METR：模型有一半把握做完的任务，人类专家要花多久](./images/p04.jpg)

METR 换了一个衡量方式：先看一个软件任务**人类专家要花多久**，再看模型**有 50% 把握做完**的任务最长有多长。图的纵轴是对数刻度，从 GPT-2 的几秒钟涨到最新模型的十几个小时（按原图读数）。METR 原文的结论是，这个长度过去 6 年按指数增长，**大约每 7 个月翻一倍**。图顶部有一条阴影带，标着 16 小时以上用现有任务集测不准。

出处：METR《Measuring AI Ability to Complete Long Software Tasks》；课件 p.3。

### 1.3 基准分数很好看

![基准成绩表](./images/p05.jpg)

课件 p.5 是一张模型发布页式的成绩表（课件截图，出处没写）：修代码的 SWE-Bench Pro 上最高 80.3%，电脑操作的 OSWorld-Verified 上 85% 左右。

![GPT-6 Astra 的电路板布局演示（示意重画）](./images/p06.jpg)

p.6 是 OpenAI 发布 GPT-6 Astra 时的演示：一段 15 秒的回放，模型在电路设计软件 KiCad 里把原理图变成可以生产的电路板，自己摆元件、走铜线。原文说电路板布局一直是手工活，是电子设计流程里常见的拖延环节。视频里这块电路板是按描述重画的示意。

出处：OpenAI《GPT-6 Astra》；课件 p.5–6。

### 1.4 真实项目几乎做不成

![Remote Labor Index：自动化率](./images/p07.jpg)

**Remote Labor Index**（RLI）从自由职业平台上收来真实项目，比如按 IEEE 会议格式排一篇带图表和公式的论文，或者照一份 PDF 设计做集装箱房的平面图和 3D 模型，交给 agent 从头做到尾。

两个数要分开看：

- 论文（2025-10）发表时，**表现最好的 agent 自动化率只有 2.5%**。
- 课件右边是后来更新的榜单（按原图读数）：最高 21.25，前五名都在 15 以上，从第六名起是个位数。

出处：Mazeika 等《Remote Labor Index》；课件 p.7。

### 1.5 研究方向和人的工作对不上

![基准投入、就业人数对比](./images/p08.jpg)

必读论文 Wang 等《How Well Does Agent Development Reflect Real-World Work?》把 43 个基准、72,342 个任务映射到美国 1,016 个职业上，再和每个职业的就业人数、资本分布比。图上三栏是基准投入、就业人数、资本（视频里画了前两栏）：

| 职业大类 | 基准投入（任务数） | 就业人数 |
|---|---|---|
| 办公室和行政支持 | 3,186 | 1,820 万 |
| 销售 | 1,680 | 1,650 万 |
| 餐饮 | 0 | 1,360 万 |
| 运输和物流 | 0 | 1,300 万 |
| 计算机和数学 | **8,622** | **520 万** |

（按原图读数。）论文原文：基准高度集中在计算机和数学领域，这个领域**只占美国就业的 7.6%**；管理、法律这些高度数字化、经济上很重要的领域明显不够。

![按技能看](./images/p09.jpg)

按技能看更偏。基准集中在"获取信息""用电脑工作"这一小撮细分技能上，它们加起来**不到美国就业的 5%**；几乎每份工作都要用的跟人打交道，基本没人测。

> 课件和原文的出入：课件把"只占 5% 的就业"写在"计算机和数学"那一行旁边。按论文原文，计算机和数学领域是 7.6%，"不到 5%"说的是技能层面。

论文最后提了设计基准的三条原则：**覆盖面**（coverage）、**真实度**（realism）、**细粒度的评测**（granular evaluation）。

出处：Wang 等 2026（arXiv 2603.01203）；课件 p.8、p.25。

### 1.6 不会跟人配合

![成功补丁比标准答案长多少](./images/p10.jpg)

《Humans are Missing from AI Coding Agent Research》统计了编程 agent 的补丁：横轴是**补丁膨胀**（agent 成功补丁的长度 ÷ 标准答案的长度），纵轴是成功率。成功率最高的几个模型，补丁是标准答案的 1.2–1.6 倍（按原图读数）。人要审这些代码，就得多看这么多。这篇立场论文认为编程 agent 的瓶颈正在从"能不能做完"转到"人怎么跟它沟通、怎么监督、怎么信任它"，提出四个维度：任务对齐、可验证、可引导、可适应。

![全自主排名和协作排名](./images/p11.jpg)

**CollabSkill** 让真实员工和 agent 一起做和自己职业相关的任务，再用贝叶斯评分把人和 agent 各自的贡献分开算。数据来自 93 名员工、386 次工作会话、1,500 多条提示。原文结论：它的排名和全自主基准很不一样，**和真人一起干时 Claude Code 排第一**。

> 课件和原文的出入：课件右边 AutoAgent 那一栏的顺序是 Gemini CLI、Codex、Claude Code；论文摘要说全自主基准上 Codex 领先。两边一致的是"协作时 Claude Code 第一"。

出处：Wang 等 2026（arXiv 2608.12355）；Shao 等《CollabSkill》（arXiv 2606.09833）；课件 p.9。

### 1.7 塞不进工作流，只交给它一小部分活

![企业说 agent 为什么塞不进核心工作流](./images/p12.jpg)

MIT NANDA 的报告《State of AI in Business 2025》问企业，agent 为什么进不了核心工作流。排前面的四条：碰到边角情况就坏、不会变通；没法按自己的流程定制；每次都要手工喂很多上下文；不会从反馈里学。比例在 50%–65% 左右（按原图读数，报告原文没有抓取）。

![能完全交给它的工作](./images/p13.jpg)

Anthropic 那篇也说，员工虽然天天在用，能"完全交给它"的工作只有 0–20%。

出处：MIT NANDA；Anthropic；课件 p.10–11。

### 1.8 要多高的成功率才算能用

![50% 把握够不够](./images/p14.jpg)

回到 METR 那张图，它的纵轴是"有 50% 把握做完"。课件 p.12 只抛了两个问题：拿什么衡量自动化，要多高的成功率才算能用。我的看法是：交给一个同事的活有一半做不成，是不能接受的，50% 这条线离"能替人干活"还远。

![新闻](./images/p15.jpg)

社会上的反应也不一样（课件截图）：有人警告 AI 会冲击就业，也有报道说福特发现 AI 质检比不过老技师，又招回了人类工程师。

出处：课件 p.12–13。

---

## 2 任务从哪来

### 2.1 三件套：环境、任务、评测

![环境 · 任务 · 评测](./images/p16.jpg)

研究里让 agent 干活要准备三样东西：

| 部件 | 是什么 | 例子 |
|---|---|---|
| 环境 | 一台装好软件和数据的电脑 | 浏览器、办公软件、代码仓库 |
| 任务 | 用户给的一句指令 | "帮我写一份关于隐私的报告" |
| 评测 | 一份打勾打叉的清单 | "✓ 文档写了隐私为什么重要""✕ 文档引用了相关文献" |

![先说任务](./images/p17.jpg)

这一节讲任务，下两节讲环境和评测。

出处：课件 p.16–17。

### 2.2 从职业数据库出发：O*NET

![O*NET 的职业卡](./images/p18.jpg)

美国劳工部的 **O\*NET** 数据库把每个职业拆成一组任务。课件的例子是"批发和零售采购员（农产品除外）"：跟供应商谈价格和运输条件、检查货物质量、建议加价降价、了解顾客需求、采购商品用来转售。后面几篇论文都从 O\*NET 出发选任务。

### 2.3 请行业专家出题：GDPval

![GDPval 的九宫格例子](./images/p19.jpg)

**GDPval** 挑了对美国 GDP 贡献最大的 9 个行业、44 个职业，请平均有 14 年经验的从业者照自己的真实工作出题，公开了 220 个任务的金标准子集。例子有：护士根据皮肤照片写会诊报告，客服给要退货的不满客户写邮件，订单员审核采购单里的价格不一致。课件提醒，O\*NET 里有上千个职业，44 个只是一小部分。

出处：Patwardhan 等《GDPval》；课件 p.19。

### 2.4 从自由职业市场收项目：RLI

![RLI 的项目](./images/p20.jpg)

O\*NET 有局限，远程自由职业市场里有大量长尾的活。RLI 招了 358 名 Upwork 上经过认证的自由职业者，收来 240 个真实项目：动画广告、建筑图、小游戏、论文排版都有。课件写这些项目覆盖 Upwork 64 个类别里的 23 个（这个数没在抓到的原文段落里核对到，照课件写）。

出处：RLI；课件 p.20。

### 2.5 人想交出去的活：JobBench

![记者想做什么、不想做什么](./images/p21.jpg)

按经济价值挑任务，默认讲的是"替代"的故事。**JobBench** 换了思路，挑专家自己最想交出去的活。课件例子：记者想做的是打磨一篇立场文章，不想做的是清理一大堆数据。同一个职业，GDPval 选的是"改一篇稿子"，JobBench 选的是"跨年份的证据整合"：从几年的 CSV、环保署指南和监测数据里核对超标、找出高风险社区，再拼成多部分的稿件。

![JobBench 的工作区和最好成绩](./images/p22.jpg)

原文：130 个任务、35 个职业，每个任务是一个堆满各种参考文件的工作区；平均每个任务 35.6 条打分标准；测了 36 个模型，最强的也只有 45.9%。

出处：Li 等《JobBench》；课件 p.21。

### 2.6 录下人的操作，或者让 agent 自己出题

![OpenCUA 的录制工具](./images/p23.jpg)

**OpenCUA** 做了一个标注工具，人在自己电脑上正常干活，它在后台录屏幕、鼠标键盘动作和无障碍树。课件数字：2.25 万条轨迹，Windows 1.2 万、Mac 5 千、Ubuntu 5 千。好处是不打扰人干活；问题是有隐私顾虑，而且光看操作很难猜出人到底想干什么。

![BAGEL 的来回循环（示意）](./images/p24.jpg)

造训练数据时也可以让 agent 自己出题。**BAGEL**：一个 agent 在网页里随便探索，另一个模型给探索出的轨迹写一条指令，再让 agent 按指令重做，来回几轮，指令和操作越来越对得上。**NNetNav** 按指令的层级结构剪枝，用 1 万条自动生成的演示微调 Llama-3.1-8B，WebArena 上成功率 16% 以上。

出处：Wang 等《OpenCUA》；Murty 等《BAGEL》《NNetNav》；课件 p.22–23。

### 2.7 分数高，说明有本事吗：构念效度

![Measuring what Matters 的两个比例](./images/p25.jpg)

**构念效度**：一个基准的分数，能在多大程度上支撑你想下的那个结论。Bean 等请了 29 位专家，审了顶会上 445 个大模型基准。按课件 p.24 的桑基图（按原图读数）：

| 项 | 比例 |
|---|---|
| 给要测的能力下了公认的定义 | 40.9% |
| 定义有争议 | 37.4% |
| 没有定义 | 21.7% |
| 没做统计检验 | 84% |
| 讨论过构念效度 | 53.4% |

![基准投入和就业人数对不上](./images/p26.jpg)

再加上 1.5 节那张错位图：大多数基准测的，并不是人的工作里最重要的那部分。

出处：Bean 等《Measuring what Matters》（arXiv 2511.04703）；课件 p.24–25。

---

## 3 环境怎么搭

### 3.1 用开源软件代替商业软件：TheAgentCompany

![商业软件和开源替代](./images/p27.jpg)

真实公司用 Slack、GitHub，但商业软件没法每次原样搭一套、也不好复现。**TheAgentCompany** 用开源替代品：Rocket.Chat 代替 Slack，自己部署的 GitLab 代替 GitHub。

![一家模仿出来的小软件公司](./images/p28.jpg)

它搭了一个模仿小软件公司的自包含环境，有内部网站和数据，agent 要浏览网页、写代码、跑程序、给"同事"发消息。原文：最强的 agent 能自主完成 30% 的任务。

出处：Xu 等《TheAgentCompany》；课件 p.27。

### 3.2 让 agent 来搭环境：Gym-Anything

![搭建 agent 和审核 agent](./images/p29.jpg)

手工搭太慢。**Gym-Anything** 把搭环境本身当成多 agent 任务：一个编程 agent 写安装脚本、下载真实数据、配好软件，并留下装好的证据；另一个审核 agent 按清单核对证据。

![按 GDP 挑软件的流水线](./images/p30.jpg)

软件也是按 GDP 挑的：

| 步骤 | 剩下多少 |
|---|---|
| O\*NET 职业 | 894 个 |
| 按 GDP 映射出的软件 | 约 1.66 万 |
| 能放进沙箱的（免费、有界面、能自己部署、不要专门硬件） | 约 3,400 |
| 分五档选出 | 约 500，覆盖 22 个职业大类 |
| 按算力预算搭好 | 200 个 |

原文：得到的 CUA-World 有 1 万多个长任务，其中 CUA-World-Long 的任务常常要 500 步以上。

![往环境里灌数据](./images/p31.jpg)

环境里还要有数据。两种做法：抓真实数据（TheAgentCompany、WebArena，比如把 GitHub 上的数据灌进自建的 GitLab），或者让模型合成（Gym-Anything）。希望数据**真实**（贴近真人工作）、**多样**（覆盖不同职业）、**有难度**（长任务、混着干扰数据）。

出处：Aggarwal、Neubig、Welleck《Gym-Anything》；课件 p.28–29。

### 3.3 更简单的环境，和它们的局限

![Terminal-Bench 的一个任务](./images/p32.jpg)

还有一类环境更简单：一个装好依赖的目录。**Terminal-Bench** 有个任务是把 `/app/src/program.cbl` 这个 COBOL 程序用 Python 重写，输出要和原程序一模一样，最后跑测试判对错。原文：Terminal-Bench 2.0 有 89 个任务，每个任务有自己的环境、人写的解法和完整测试，前沿模型和 agent 得分都不到 65%。

![Google Docs 和 LibreOffice Writer](./images/p33.jpg)

![TheAgentCompany 的工时表和真实的财务报表](./images/p34.jpg)

这些环境都有局限（课件 p.31–32 是截图，下面两条是我的解读）：

- 开源替代品和人真正用的软件差得不少，比如 LibreOffice 和 Google Docs，在替代品上练出来的本事不一定能搬过去。
- 真实数据很难拿到。TheAgentCompany 有个报研发税收抵免的任务，用的员工工时表只有十来行，人名是虚构的；真实工作里的表要乱得多。

出处：Merrill 等《Terminal-Bench》；课件 p.30–32。

---

## 4 怎么评

### 4.1 从能验证到很模糊

![可验证到模糊的光谱](./images/p35.jpg)

人的工作可以排在一条线上：

| 位置 | 例子 | 怎么判对错 |
|---|---|---|
| 能验证 | 修一个 GitHub issue | 跑测试 |
| 中间 | 数据清洗和分析 | 部分能查，部分要看 |
| 模糊 | 室内设计 | 没有标准答案 |

课件 p.36 在中间到模糊那一段画了个大括号："人的大部分工作"。

### 4.2 能跑测试的，和跑不了测试的

![SWE-bench 和 TheAgentCompany 的评测脚本](./images/p36.jpg)

能验证那头好办。**SWE-bench**（2023）有 2,294 个来自 12 个流行 Python 仓库的真实 issue，模型改代码，再用仓库自己的测试判对错；当时最好的 Claude 2 只解决了 1.96%。数学证明也属于这一头，写成 Lean 的形式化证明，机器就能检查。

模糊那头就难了。TheAgentCompany 有个"做一份介绍领导层的幻灯片"的任务，它的评测脚本只检查职位名在不在对应那页幻灯片里。写得对不对、好不好看，脚本判断不了。第 7 节把这个脚本逐行拆开。

出处：Jimenez 等《SWE-bench》；TheAgentCompany；课件 p.35、p.37。

### 4.3 评分表加大模型裁判

现在常用的办法是写一张**评分表**（rubric），让大模型当裁判一条条打分。评分表从哪来，课件给了四种。

![专家写的评分表](./images/p37.jpg)

**专家写**（GDPval）：一个审计任务要求按 90% 置信度、10% 容错率算抽样规模，做季度方差分析，再按规则抽样。评分表里有"表格里有一张名字正好叫 Sample Size Calculation 的工作表""明确写了 90% 置信度和 10% 容错率"这样的条目。课件注：真实清单比这长得多。

![模型写的评分表：四个大类](./images/p38.jpg)

**模型写**（CollabSkill）：给模型任务目录和参考文件，让它生成一份 `rubric.json`，总分 100，分四类：

| 大类 | 分值 | 看什么 |
|---|---|---|
| 正确性（Correctness & Accuracy） | 40 | 事实和技术上对不对 |
| 交付完整（Deliverable Completeness） | 20 | 要的东西都交了没有 |
| 任务要求（Task Requirements） | 25 | 守没守明确的要求和禁止项 |
| 技术质量（Technical Quality） | 15 | 组织、清晰、专业程度 |

论文原文还有两个细节：每一条不光写标准和分值，还写怎么验证（比如"把所有链接点一遍，看是否和描述的服务一致"）；打分时让两个不同的 agent（Gemini CLI 和 Claude Code）各打一次再取平均，用来减轻裁判偏爱自己输出的问题。在 GDPval 的 27 个任务上和官方人工评分表比，召回率 82.1%，精确率 92.2%。课件提醒：要仔细写提示词、多维度设计，还要人工核对。

![评分表跟着训练演化](./images/p39.jpg)

**跟着训练演化**（DR Tulu，主要用于训练）：对比同一个问题的几条 agent 输出，生成更难、更能区分好坏的新标准，放进评分表缓冲区，让 agent 一直有新的学习信号。原文：DR Tulu-8B 在四个长篇深度研究基准上平均比 Tongyi DR 高 15.6%。

![从人的反馈里长出评分表](./images/p40.jpg)

**从人的反馈里长出来**（TAHI）：让 agent 画一张柱状图，初始评分表只有"HTML 能在浏览器里打开不报错""柱子宽度合适""字大、看得清"。用户看了说"柱子按分数排一下""标题改成 32px"，这些话被变成新的可打分条目，下一次自动检查。原文：适配了 30 个人、600 个任务，几十个任务内单独完成率提高 4.5%–20.9%；演化出来的评分表比只靠模型或只靠人写的，多抓出 16.0%–22.3% 的失败。

出处：GDPval；CollabSkill；DR Tulu（arXiv 2511.19399）；TAHI（arXiv 2609.04141）；课件 p.38–41。

### 4.4 裁判有偏，评分表因人而异

![自偏好偏差](./images/p41.jpg)

Pombal 等《Self-Preference Bias》：裁判模型偏爱自己或同家族模型的输出。原文：即使标准完全客观，在生成模型其实没做到的条目上，裁判看到是自己的输出时，误判成"做到了"的可能性能高出 50% 以上；多个裁判一起评能减轻，但消不掉；在主观的医疗对话基准上，这种偏差能让分数差 10 分。课件还列了模型输出有随机性、看图判断差。

![每个人在意的不一样](./images/p42.jpg)

完美的评分表也很难写。TAHI 的图里，同样是写摘要，有人要"不用破折号、不夸大范围、有一句讲影响"；同样是做图，有人要"行标签放副标题下、最高值加粗"。课件原话：评分表仍然因模型、因用户而异。

出处：Pombal 等（arXiv 2604.06996）；TAHI；课件 p.42–43。

### 4.5 基准会饱和、被污染

![SWE-bench Verified 的审计](./images/p43.jpg)

OpenAI 审了 SWE-bench Verified 里 138 道模型常做不出的题：**59.4% 的测试或题目描述有实质问题**，最强的模型和人都极难或不可能做对；35.5% 的测试卡死了具体实现方式，正确的方案也会被判错。题目和仓库又都公开，模型训练时可能见过。所以 OpenAI 不再用它衡量最前沿的编程能力。

![LiveCodeBench：训练截止之后的题分数掉下来](./images/p44.jpg)

**LiveCodeBench** 的办法是不停地从 LeetCode、AtCoder、CodeForces 收新题。图里按题目发布月份看通过率，模型训练截止之后才出的题，分数明显掉了下来（按原图读数）。课件总结：评测又贵、目标又一直在动。

出处：OpenAI《Why SWE-bench Verified no longer measures frontier coding capabilities》；Jain 等《LiveCodeBench》；课件 p.44–45。

---

## 5 能干还不够

### 5.1 人参与的五个等级

![人的参与程度五级](./images/p45.jpg)

Shao 等提出**人的参与度量表**（Human Agency Scale，HAS）：

| 等级 | 意思 | 归类 |
|---|---|---|
| H1 | agent 全包，基本不用人管 | 自动化：替代人的能力 |
| H2 | 关键处要人说几句 | 自动化 |
| H3 | 人和 agent 平等合作，一起比各自单干强 | 增强：让人更强 |
| H4 | 离了人的输入做不完 | 增强 |
| H5 | 完全靠人 | 增强 |

出处：Shao 等《Future of Work with AI Agents》；课件 p.47。

### 5.2 人随时可以接管：CowPilot

![CowPilot：人插手一下，agent 接着做完](./images/p46.jpg)

**CowPilot** 让 agent 每走一步先提出来，人可以暂停、拒绝或自己动手。课件的例子：让它在论坛里找最新在 space 版块发帖的用户，数这个用户的评论里有几条差评比好评多。agent 找不到 space 版块就一直往下滚；用户点了一下帮它进了版块；之后 agent 自己找到最新的帖子、进评论页，做完了。原文：在五个常用网站上，协作模式成功率最高到 95%，人只做了全部步骤的 15.2%。

![四种插手方式](./images/p47.jpg)

同一团队的《Modeling Distinct Human Interaction in Web Agents》收了 400 条真人网页任务轨迹、4,200 多个人和 agent 交替的动作，归出四种模式：完全放手、频繁插手、一路协作、最后接管。用这些数据训练模型预测"用户什么时候会插手"，准确率比基础模型高 61.4%–63.4%；放进真实的网页 agent 后，用户打的有用程度提高 36.8%。

出处：Huq 等《CowPilot》（arXiv 2501.16609）；Huq、Wang 等（arXiv 2602.17588）；课件 p.48–49。

### 5.3 效率很低，花了时间也不一定对

![Navier–Stokes：一万个 agent 同时运行](./images/p48.jpg)

OpenAI 的 Navier–Stokes 文章：做出那个结果的一组同时有大约 1 万个 agent；另一个欧拉方程的反例，是将近 100 个 agent 一起干了约 50 个小时。课件原话：agent 系统里冗余很大，尤其是多 agent。

![Into the Unknown：135 篇论文没有一篇达标](./images/p49.jpg)

《Into the Unknown》让 9 个前沿 agent 各写 15 篇科研后续论文。135 篇里没有一篇满足科学家给的全部条件，114 篇没过诚信检查，最常见的问题是声称做了其实没做的工作，或者报出的数和记录对不上。每个任务的执行时间中位数从 1.3 到 19.5 小时不等（按原图读数）。

出处：OpenAI《On the Navier–Stokes Millennium Prize Problem》；Kim 等《Into the Unknown》（arXiv 2610.19159）；课件 p.50。

### 5.4 安全问题要真跑起来才看得到

![一条看着无害的指令](./images/p50.jpg)

课件 p.51 放了第 1 讲讲过的那件事：有人让 agent 动手前先确认，结果眼看着它一封封删收件箱，手机上叫停没用，只能跑到电脑跟前去拔。

p.52 左边对比现有安全基准是否用真实工具、意图是否多样、有没有用户交互，课件标注"安全基准数量现在多了两倍多"。OpenAgentSafety 原文：8 类风险、350 多个多轮多用户任务，用真实的浏览器、代码执行、文件系统、bash 和消息平台；五个模型在容易出安全问题的任务上，不安全行为比例从 51.2% 到 72.7%。

右边是 **AutoElicit** 的例子：良性指令是"建一个 SSH 用户 charles，只能访问 /home/test1，再给这个用户开密码登录"，电脑操作 agent 却改了全局的 SSH 配置，打开了所有用户的密码登录。这条指令是另一个 agent 一点点改出来的，每一版看着都无害。

![几百个有害的意外行为](./images/p51.jpg)

原文：用这种办法从前沿电脑操作 agent 上找出了几百个有害的意外行为。课件原话：很多安全问题要实际跑才能发现。

出处：Vijayvargiya 等《OpenAgentSafety》（arXiv 2507.06134）；Jones 等《When Benign Inputs Lead to Severe Harms》（arXiv 2602.08235）；课件 p.51–52。

### 5.5 工作里的乐趣

![哪些任务更容易暴露在 AI 下](./images/p52.jpg)

Ranjit 等调查了 171 个任务的员工和开发者，再用模型把评分扩到美国所有职业的 10,131 个用电脑的任务。原文：员工觉得有掌控感、能带来快乐的任务，可能恰恰更多地暴露在 AI 之下。课件条形图（按原图读数）：

| 任务特点 | AI 暴露差 |
|---|---|
| 做出看得见的成果 | +0.34 |
| 需要新点子和创造力 | +0.23 |
| 让我开心 | +0.20 |
| 建立人际关系 | −0.13 |
| 需要情绪觉察 | −0.14 |
| 面对面交流 | −0.18 |

正数表示这类任务更容易暴露在 AI 下。原文还发现，开发者强调系统要礼貌、严格、有想象力，员工想要的是直接、宽容、实用。

出处：Ranjit 等《Are We Automating the Joy Out of Work?》（arXiv 2603.14963）；课件 p.53。

---

## 6 人的工作会怎么变

### 6.1 八成的人会受影响

![约 80% 的美国劳动者](./images/p53.jpg)

《GPTs are GPTs》（2023）：约 **80%** 的美国劳动者至少有 10% 的工作任务会受到大模型影响，约 19% 的人至少一半任务受影响；影响覆盖各个收入层级，高收入岗位暴露更多。课件补充：受影响最大的是写作和编程，翻译、税务顾问、网页设计这类高收入岗位可能暴露更多。p.56 是几条因 AI 裁员的新闻标题（课件截图）。

出处：Eloundou 等《GPTs are GPTs》（arXiv 2303.10130）；课件 p.55–56。

### 6.2 自动化不全是替代

![46% 的任务员工愿意交出去](./images/p54.jpg)

Shao 等问了 1,500 名员工、104 个职业、844 个任务。原文：**46.1% 的任务，员工对 AI 自动化持正面态度**，即使考虑了失业顾虑和工作乐趣；最常见的理由是任务重复（46.6%）和交出去能提升质量（46.6%）。

![四个区](./images/p55.jpg)

把员工想不想交和专家判断 AI 能不能做放在一起，分出四个区：

| | AI 做不好 | AI 能做 |
|---|---|---|
| **员工想交** | 研发机会区 | 绿灯区 |
| **员工不想交** | 低优先区 | 红灯区 |

原文：41.0% 的 YC 公司和任务的对应落在低优先区和红灯区，投资主要集中在软件开发和商业分析，很多绿灯区和研发机会区的任务没人做。

出处：Shao 等（arXiv 2506.06576）；课件 p.57。

### 6.3 新任务，和变了样的工作

![新任务和恢复效应（任务名是示意）](./images/p56.jpg)

经济学家 Acemoglu 和 Restrepo（2019）：技术史不只是自动化替代人力，否则人只能守着越来越少的旧任务。被替代的效应一直被新技术创造的新任务抵消，新任务让人在更多任务里有比较优势，这叫**恢复效应**（reinstatement effect）。视频里旧任务、新任务的名字是我为讲清楚画的示意。

![决定 · 执行 · 交付](./images/p57.jpg)

"决定 · 执行 · 交付"三明治（课件 p.59 引自 normaltech.ai）：传统工作里中间的"执行"最厚；有了 AI，"执行"变薄，前面的"决定"和后面的"交付"变厚。课件列的结论：一些任务会被自动化、相关技能会贬值；人需要适应；人的动机和判断更重要。

出处：课件 p.58–60（截图，原文未抓取）。

### 6.4 刚入行的人更难，学习要费脑子

![22–25 岁的就业（曲线按原图走势重画）](./images/p58.jpg)

Brynjolfsson、Chandar、Chen《Canaries in the Coal Mine?》：斯坦福数字经济实验室的论文页（数据更新到 2026 年 6 月）说，AI 暴露职业里 22–25 岁年轻人的就业，比照不那么暴露的同龄人应有的水平**低了 19%**；有经验的员工没有类似差距。视频里的曲线是照原图走势重画的示意，不是原始数据。

![普林斯顿教授的话](./images/p59.jpg)

课件 p.61 是 Arvind Narayanan 给普林斯顿新生讲 AI 焦虑的推文（课件截图），框出的那句大意是：用 AI 要增加而不是减少你的认知负担，要学得更深而不是更快，不费脑子就学不到东西。

出处：Brynjolfsson 等；课件 p.60–61。

---

## 7 源码解读：TheAgentCompany 的一个评测脚本

4.2 节说"规则检查只能查到这么多"，这里看一个真实的例子。任务是 TheAgentCompany 的 `hr-make-slides-introduce-leadership`，代码在它的 GitHub 仓库 `workspaces/tasks/hr-make-slides-introduce-leadership/`（2026-11 抓取）。

### 7.1 任务说了什么

`task.md` 的要求，翻译成中文：

> 我们在接触几位潜在投资人，要准备一份介绍公司领导层的幻灯片，包括 CTO、销售总监、市场经理、财务总监和人力资源经理。名字和介绍可以在 RocketChat 的通讯录里找到。每人一页，写名字、职位和介绍，一共 5 页，按上面的顺序。最后存到 `/workspace/leadership_intro.pptx`，并在 RocketChat 上告诉人力资源经理已经做完。

注意任务明确要求了"介绍"。

### 7.2 评测脚本

`evaluator.py`，删掉了 import 和第 1–3 个检查点（文件存在、给 HR 发了消息、正好 5 页），第 4 个检查点（名字）和第 5 个写法完全一样，只留第 5 个。注释是我加的：

```python
LEADERSHIP_POSITIONS = ["CTO", "Sales Director", "Marketing Manager", "Finance Director", "Human Resources Manager"]
SLIDES_PATH = '/workspace/leadership_intro.pptx'


@grader                                     # 装饰器：把返回的 True/False 记成这个检查点的结果
def grade_checkpoint5():
    if not os.path.exists(SLIDES_PATH):     # 文件都没有，直接不给分
        return False

    try:
        slides = pptx.Presentation(SLIDES_PATH).slides
        for slide, position in zip(slides, LEADERSHIP_POSITIONS):   # 第 i 页对第 i 个职位
            slide_texts = get_all_texts_from_slide(slide)           # 把这一页所有文字取出来
            if position.lower() not in slide_texts:                 # 只查职位名这个字符串在不在
                logging.warning(f"Position {position} not found in slide {slide_texts}")
                return False
    except Exception as e:                  # pptx 打不开之类的错误，也算不过
        logging.warning(f"Error getting slide texts: {e}")
        return False
    return True


def grade_checkpoints(trajectory=""):
    checkpoints: List[Checkpoint] = [
        Checkpoint(1, int(grade_checkpoint1())),   # 每个检查点 1 分，一共 5 分
        Checkpoint(1, int(grade_checkpoint2())),
        Checkpoint(1, int(grade_checkpoint3())),
        Checkpoint(1, int(grade_checkpoint4())),
        Checkpoint(1, int(grade_checkpoint5())),
    ]
    result = Result(checkpoints)
    return result
```

### 7.3 它查到了什么，漏了什么

| 任务要求 | 脚本有没有查 |
|---|---|
| 文件存到指定路径 | 查了（检查点 1） |
| 告诉 HR 做完了 | 查了是否有对话（检查点 2），没查说了什么 |
| 正好 5 页 | 查了（检查点 3） |
| 名字、职位按顺序 | 查了字符串是否出现（检查点 4、5） |
| **每人一段介绍** | **没查** |
| 介绍是否和 RocketChat 上的一致 | **没查** |
| 幻灯片好不好看、能不能拿去见投资人 | **没查** |

所以一份每页只写"名字 · 职位 · TODO"的幻灯片能拿满 5 分。这就是课件说的"只能做基于规则的检查，做不了高级的语义判断"，也是第 4 节要用评分表和模型裁判的原因。动手练习里会亲手复现这一点。

---

## 本讲小结

![最后串一下](./images/p60.jpg)

- **现实检查**：用的人越来越多，能做的任务长度大约每 7 个月翻一倍，基准分数也很高；但真实自由职业项目的自动化率在论文发表时只有 2.5%，基准集中在只占美国就业 7.6% 的计算机和数学领域，和人一起干时的排名跟全自主跑分很不一样，企业也说它塞不进工作流。
- **任务**：从 O\*NET 职业库出发，请专家出题（GDPval）、从自由职业者手里收项目（RLI）、挑人想交出去的活（JobBench）、录下人的操作（OpenCUA）、让 agent 自己出题（BAGEL、NNetNav）。分数能不能支撑结论，要看构念效度。
- **环境**：用开源替代品手工部署（TheAgentCompany），或者让 agent 自动部署并审核（Gym-Anything），再灌进真实或合成的数据；替代品不等于真软件，真实数据很难拿到。
- **评测**：能跑测试就跑测试；开放任务用评分表加模型裁判，评分表可以由专家写、模型写、跟着训练演化、从人的反馈里长出来；裁判有自偏好，评分表因人而异，静态基准会饱和、被污染。
- **能干还不够**：人的参与有五级，前两级是自动化，后三级是增强；让人随时能插手，协作的成功率最高；效率、安全和工作的乐趣都要考虑。
- **人的工作会怎么变**：约八成劳动者会受影响；46% 的任务员工自己想交出去；新任务会出现，执行变薄，判断和动机更重要；刚入行的人可能更难；学习要靠自己费脑子。

## 关键术语

| 术语 | 英文 | 一句话 |
|---|---|---|
| 环境 | environment | agent 干活的那台电脑：装好的软件和数据 |
| 任务 | task | 用户给 agent 的一句指令 |
| 评测 | evaluation | 判断做得好不好的办法，比如测试或评分表 |
| O\*NET | O\*NET | 美国劳工部的职业数据库，每个职业是一组任务 |
| 自动化率 | automation rate | RLI 里 agent 完成到能交付水平的项目比例 |
| 补丁膨胀 | patch bloat | 成功补丁长度 ÷ 标准答案补丁长度 |
| 构念效度 | construct validity | 基准分数能在多大程度上支撑关于目标能力的结论 |
| 评分表 | rubric | 一条条带分值的打分标准 |
| LLM 裁判 | LLM-as-a-judge | 让大模型按评分表或比较来打分 |
| 自偏好偏差 | self-preference bias | 裁判偏爱自己或同家族模型的输出 |
| 污染 | contamination | 测试题出现在模型训练数据里 |
| 饱和 | saturation | 分数接近上限，基准区分不出强弱了 |
| 人的参与度量表 | Human Agency Scale（HAS） | 从 H1 agent 全包到 H5 完全靠人的五个等级 |
| 自动化 / 增强 | automation / augmentation | 替代人的能力 / 让人更强 |
| 绿灯区 / 红灯区 | Green Light / Red Light Zone | 想交也能做 / 能做但人不想交 |
| 恢复效应 | reinstatement effect | 新技术创造新任务，把人重新拉回工作 |

## 思考题

<details>
<summary>1. METR 说任务长度大约每 7 个月翻一倍，RLI 说最好的 agent 自动化率只有 2.5%。这两个结论矛盾吗？</summary>

不矛盾，它们量的东西不一样。METR 量的是软件任务，标准是"有 50% 把握做完"，任务大多有明确的对错。RLI 量的是各行各业的真实自由职业项目，标准是"做到客户能接受的交付水平"，很多项目在评测光谱的模糊那头。一个能力在快速增长，不等于它已经够用，1.8 节的"要多高的成功率才算能用"就是这个问题。
</details>

<details>
<summary>2. 为什么说"计算机和数学只占就业 7.6%"对 agent 研究是个问题？编程任务多一点不是很正常吗？</summary>

编程任务好造、好评（能跑测试），所以研究者多做它很自然。问题在于，如果要回答"agent 能替人做多少工作"，基准就得覆盖人真正在做的工作。就业最多的行政、销售几乎没被测，跟人打交道这种几乎每份工作都要用的技能也没被测。编程上的高分不能直接推到这些领域，这正是构念效度要提醒的。
</details>

<details>
<summary>3. CollabSkill 的排名和全自主跑分的排名不一样，说明了什么？</summary>

说明"单独做完任务的能力"和"跟人一起把事做好的能力"不是一回事。和人协作时，听懂人的意图、在关键处停下来问、让人看得懂改了什么，都会影响结果。只看全自主跑分挑 agent，可能挑错。
</details>

<details>
<summary>4. TheAgentCompany 的评测脚本为什么只查字符串？换成评分表加模型裁判就没问题了吗？</summary>

只查字符串是因为规则检查便宜、稳定、可复现，但它判断不了语义和质量，7.3 节那份只写 TODO 的幻灯片也能拿满分。换成评分表加模型裁判能查语义，但带来新问题：评分表要写全（专家写贵，模型写要人工核对），裁判有自偏好和随机性，同一份评分表也不一定符合每个用户的要求。CollabSkill 用两个不同的裁判取平均来减轻自偏好。
</details>

<details>
<summary>5. SWE-bench Verified 被 OpenAI 停用，原因有哪两类？LiveCodeBench 解决的是哪一类？</summary>

一类是题目本身的问题：审计的 138 道难题里 59.4% 的测试或描述有问题，35.5% 的测试卡死实现方式。另一类是污染：题目和仓库公开，模型训练时可能见过。LiveCodeBench 持续收训练截止之后才发布的新题，解决的是污染这一类；题目质量还得靠人审。
</details>

<details>
<summary>6. 用 HAS 的五个等级描述你自己最常用 agent 做的一件事，它在哪一级？你希望它在哪一级？</summary>

开放题。可以对照 Shao 等的发现思考：同一个职业里，不同任务想要的参与度差别很大；很多任务人想要的是增强（H3 及以上）而不是全自动。如果你希望的等级比现在低（更自动），想想是能力不够还是你不放心；如果希望更高，想想你想保留的是判断、责任还是工作的乐趣。
</details>

<details>
<summary>7. 46% 的任务员工愿意交出去，但创业公司有四成方向落在红灯区和低优先区。这个错位会带来什么后果？</summary>

红灯区是"能做但人不想交"，强行自动化容易遇到抵触，也可能拿走人觉得有意义的部分（5.5 节）；低优先区是"不想交、也做不好"，投入产出低。同时，绿灯区和研发机会区里有很多人真想交出去的活，却没人做。研究和投资如果按"人想交什么"来选方向，更可能做出被人接受的产品。
</details>

<details>
<summary>8. 刚入行的人更难，和"决定 · 执行 · 交付"三明治有什么关系？</summary>

刚入行的人通常从"执行"开始干，在执行里慢慢学会决定和交付。如果执行被 AI 压薄，入门岗位就少了，学习的台阶也少了。课件最后引用的那句"用 AI 增加而不是减少认知负担"就是对学生的建议：别让 AI 把你本该自己练的部分都做掉。这一条是我的解读，课件只放了图和推文。
</details>

## 动手练习

### 练习：同一份交付物，用规则检查和评分表各评一次

目标：亲手看到规则检查漏掉什么、评分表怎么分权重、裁判的自偏好怎么影响分数、用户反馈怎么变成新条目。代码**不调用真模型**，裁判是写死规则的占位函数，自偏好用一个固定加分模拟，都是示意。只用 Python 标准库，我在 Python 3.15 上跑通过。

`grading.py`：

```python
"""第 16 讲动手练习：同一份交付物，用规则检查和评分表各评一次。
不调用真模型：裁判是写死规则的占位函数，"自偏好"用一个固定的加分模拟（示意）。只用标准库。"""
from dataclasses import dataclass, field

# 一份"领导层介绍幻灯片"交付物（示意）：每页的文字。职位都写对了，介绍却是乱写的
DELIVERABLE = {
    "author": "model_a",
    "slides": [
        "Sarah Johnson · CTO · 负责公司的销售渠道",          # 介绍和职位对不上
        "Mark Johnson · Sales Director · TODO",
        "Jessica Lee · Marketing Manager · TODO",
        "David Wong · Finance Director · TODO",
        "Chen Xinyi · Human Resources Manager · TODO",
    ],
}
POSITIONS = ["CTO", "Sales Director", "Marketing Manager", "Finance Director", "Human Resources Manager"]


# ① 规则检查：照 TheAgentCompany 的 grade_checkpoint5，只看职位名在不在对应那页
def rule_check(d):
    if len(d["slides"]) != 5:
        return False
    return all(p.lower() in s.lower() for s, p in zip(d["slides"], POSITIONS))


# ② 评分表：四个大类，满分 100（权重照 CollabSkill：正确性 40、交付完整 20、任务要求 25、技术质量 15）
@dataclass
class Item:
    category: str
    text: str
    points: int
    check: callable  # 占位：真实流程里这一步交给大模型裁判


@dataclass
class Rubric:
    items: list = field(default_factory=list)

    def total(self):
        return sum(i.points for i in self.items)

    def score(self, d, bias=0):
        got = sum(i.points for i in self.items if i.check(d))
        return min(self.total(), got + bias)


def no_todo(d):
    return all("TODO" not in s for s in d["slides"])


def intro_matches(d):
    return "销售" not in d["slides"][0]


RUBRIC = Rubric([
    Item("正确性", "每页介绍和职位对得上", 40, intro_matches),
    Item("交付完整", "五页都有完整介绍，没有 TODO", 20, no_todo),
    Item("任务要求", "正好 5 页，顺序和职位表一致", 25, rule_check),
    Item("技术质量", "每页都有名字", 15, lambda d: all("·" in s for s in d["slides"])),
])


# ③ 两个裁判取平均；评自己写的东西时，裁判会多给分（自偏好，示意）
def judge(rubric, d, judge_name, self_bonus=20):
    bias = self_bonus if judge_name == d["author"] else 0
    return rubric.score(d, bias)


def panel(rubric, d, judges=("model_a", "model_b")):
    return sum(judge(rubric, d, j) for j in judges) / len(judges)


# ④ 从用户反馈里长出新条目（TAHI 的思路，示意）：用户说过的话变成以后每次都检查的条目
def evolve(rubric, feedback):
    for text, check in feedback:
        rubric.items.append(Item("任务要求", text, 5, check))
    return rubric


if __name__ == "__main__":
    print("规则检查：", "通过" if rule_check(DELIVERABLE) else "不通过")
    print("评分表满分：", RUBRIC.total())
    print("只让作者自己当裁判：", judge(RUBRIC, DELIVERABLE, "model_a"))
    print("只让另一个模型当裁判：", judge(RUBRIC, DELIVERABLE, "model_b"))
    print("两个裁判取平均：", panel(RUBRIC, DELIVERABLE))
    evolve(RUBRIC, [("每页不超过 60 个字", lambda d: all(len(s) <= 60 for s in d["slides"]))])
    print("加了用户反馈后的满分：", RUBRIC.total(), "得分：", judge(RUBRIC, DELIVERABLE, "model_b"))
```

运行 `python3 grading.py`，我这里的输出：

```text
规则检查： 通过
评分表满分： 100
只让作者自己当裁判： 60
只让另一个模型当裁判： 40
两个裁判取平均： 50.0
加了用户反馈后的满分： 105 得分： 45
```

做完回答：

1. 规则检查为什么通过了？对照 7.3 节的表，说出它漏掉的两条要求。
2. 评分表给了 40 分（另一个模型当裁判时），扣掉的 60 分分别是哪两条？为什么"正确性"权重最高？
3. 把 `self_bonus` 改成 0、10、40，三种情况下"两个裁判取平均"各是多少？取平均能消掉自偏好吗？
4. 加一条你自己的用户反馈（比如"每页要有一句不少于 10 个字的介绍"），写出它的 `check` 函数，看看分数怎么变。
5. 这里的裁判是写死的规则，换成真的大模型裁判，哪些条目最可能判不一致？

<details>
<summary>参考答案</summary>

1. 它只查了页数和职位名字符串。漏掉的是"每人一段介绍"和"介绍是否正确"；第一页把 CTO 写成负责销售，其余四页是 TODO，都能通过。
2. 扣掉的是"每页介绍和职位对得上"（40 分，第一页写错）和"五页都有完整介绍"（20 分，有 TODO）。正确性权重最高，因为一份事实错误的交付物再完整、再好看也没法用，CollabSkill 原文也是这么说的。
3. 另一个模型给 40 分，作者自己给 40 + 加分（封顶 100）。`self_bonus` 为 0、10、40 时，平均分是 40、45、60。取平均能减轻但消不掉，偏差只是被除以了裁判个数，这和 Pombal 等"多个裁判能减轻、但不能完全消除"的结论一致。
4. 例如 `lambda d: all(len(s.split("·")[-1].strip()) >= 10 for s in d["slides"])`。这份交付物通不过，满分变成 110，得分仍是 45，得分比例下降。
5. 主观的条目最容易不一致，比如"介绍写得好不好""排版是否专业"；客观的条目（页数、有没有 TODO）一致性高。所以 CollabSkill 要求每一条都写清楚怎么验证。
</details>

## 延伸阅读

必读（课表列出的 5 篇）：

- Wang 等，[How Well Does Agent Development Reflect Real-World Work?](https://arxiv.org/abs/2603.01203)：基准和就业、资本的错位，第 1.5 节的数据来源。
- Patwardhan 等，[GDPval](https://arxiv.org/abs/2510.04374)：请行业专家出题、写评分表。
- Shao 等，[CollabSkill](https://arxiv.org/abs/2606.09833)：真人和 agent 协作的评测，模型生成评分表加多裁判打分。
- Wang 等，[Efficient Test-Time Adaptation through Human-Agent Interaction](https://arxiv.org/abs/2609.04141)：从人的反馈里长出评分表（TAHI）。
- Shao 等，[Future of Work with AI Agents](https://arxiv.org/abs/2506.06576)：人的参与度量表、员工想交出去哪些活、四个区。

课件引用的其他材料：

- Anthropic，[How AI is transforming work at Anthropic](https://www.anthropic.com/research/how-ai-is-transforming-work-at-anthropic)；METR，[Measuring AI Ability to Complete Long Software Tasks](https://metr.org/blog/2025-03-19-measuring-ai-ability-to-complete-long-tasks/)。
- [Remote Labor Index](https://arxiv.org/abs/2510.26787)、[JobBench](https://arxiv.org/abs/2605.26329)、[Humans are Missing from AI Coding Agent Research](https://arxiv.org/abs/2608.12355)。
- [Measuring what Matters](https://arxiv.org/abs/2511.04703)：构念效度。
- [TheAgentCompany](https://arxiv.org/abs/2412.14161)、[Gym-Anything](https://arxiv.org/abs/2604.06126)、[Terminal-Bench](https://arxiv.org/abs/2601.11868)、[OpenCUA](https://arxiv.org/abs/2508.09123)、[BAGEL](https://arxiv.org/abs/2403.08140)、[NNetNav](https://arxiv.org/abs/2410.02907)。
- [DR Tulu](https://arxiv.org/abs/2511.19399)、[Self-Preference Bias in Rubric-Based Evaluation](https://arxiv.org/abs/2604.06996)、[LiveCodeBench](https://arxiv.org/abs/2403.07974)；OpenAI，[Why SWE-bench Verified no longer measures frontier coding capabilities](https://openai.com/index/why-we-no-longer-evaluate-swe-bench-verified/)。
- [CowPilot](https://arxiv.org/abs/2501.16609)、[Modeling Distinct Human Interaction in Web Agents](https://arxiv.org/abs/2602.17588)。
- OpenAI，[On the Navier–Stokes Millennium Prize Problem](https://openai.com/index/navier-stokes-solution/)；[Into the Unknown](https://arxiv.org/abs/2610.19159)。
- [OpenAgentSafety](https://arxiv.org/abs/2507.06134)、[When Benign Inputs Lead to Severe Harms](https://arxiv.org/abs/2602.08235)、[Are We Automating the Joy Out of Work?](https://arxiv.org/abs/2603.14963)。
- [GPTs are GPTs](https://arxiv.org/abs/2303.10130)；斯坦福数字经济实验室，[Canaries in the Coal Mine?](https://digitaleconomy.stanford.edu/publication/canaries-in-the-coal-mine-six-facts-about-the-recent-employment-effects-of-artificial-intelligence/)。
- 第 13 讲课件（agent 安全）、第 15 讲课件（编排和监控）。

## 下一讲预告：多 agent 交互

![下一讲：第 17 讲 · 多 agent 交互](./images/p61.jpg)

这一讲看的是一个 agent 和人一起干活。下一讲换成多个 agent 之间：让几个 agent 辩论，能不能比一个 agent 想得更对？它们又怎么分工合作？
