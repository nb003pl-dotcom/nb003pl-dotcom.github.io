# AI4AI（AI for AI Research）综述：用 AI 自动化 AI 研究

*生成日期：2026-11-04 ｜ 来源：40+（arXiv / 官方博客 / Nature / 主流科技媒体）｜ 置信度：中高（关键数字均经一手来源核实，未核实项显式标注）*

---

## 摘要（Executive Summary)

AI4AI 指**用 AI 系统自动化 AI（乃至一般科学）研究流程**的方向：从早期只覆盖 pipeline 窄环节的 AutoML/NAS/HPO，演进到 LLM 时代能完成"提出想法 → 写代码做实验 → 分析结果 → 撰写论文 → 接受评审"全闭环的科研智能体。2024–2026 年出现三个标志性事件：Sakana AI Scientist-v2 生成的论文通过 ICLR 2025 workshop 双盲评审（分数 6/7/6）；Google AI co-scientist 在生物医学领域产出经湿实验验证的新假设；前沿模型在 MLE-bench 上达到 Kaggle 奖牌水平。但评测同时揭示硬约束：PaperBench 显示模型"能读懂论文却复现不出代码"（最佳 21% vs 人类博士 ~42%），RE-Bench 显示 agent 在长时间尺度上仍落后人类专家。评审污染（ICLR 约 21% 评审被判全 AI 生成）与 AI 论文泛滥已成为学术社区的现实治理难题。

---

## 1. 定义与范畴

**AI4AI（AI for AI Research）**：以 AI 系统（尤其是 LLM 智能体）作为执行主体，自动化机器学习/AI 研究的生命周期，包括：

- **想法层**：研究选题、假设生成、新颖性判断
- **实验层**：ML 工程（建模、调参、代码迭代）、实验设计与执行
- **产出层**：结果分析、论文写作、图表生成
- **评价层**：自动评审、复现验证
- **元层**：科研智能体的自我改进与自动设计（与 self-improvement 交叉）

它与相邻概念的边界：
- **AutoML**：AI4AI 的前史，只自动化"模型选择+调参"这一窄环节；
- **AI4Science**：AI 用于自然科学发现（如 AI co-scientist 的生物医学应用、机器人科学家），与 AI4AI 共享智能体技术栈，本文只在交叉处简述；
- **Self-improvement**：模型改进自身能力，是 AI4AI 的"元层"，见 §7。

---

## 2. 前史：AutoML / NAS / HPO（2013–2022）

### 2.1 里程碑

| 时间 | 工作 | 贡献 | 来源 |
|---|---|---|---|
| 2013 | **Auto-WEKA**（Thornton, Hutter 等，KDD 2013） | 提出 CASH 问题：算法选择与超参优化联合求解，用 SMAC 贝叶斯优化 | [arXiv:1208.3719](https://arxiv.org/abs/1208.3719) |
| 2016/17 | **NAS**（Zoph & Le，ICLR 2017） | RNN 控制器 + REINFORCE 搜索网络架构，匹敌人工设计 | [arXiv:1611.01578](https://arxiv.org/abs/1611.01578) |
| 2018 | **ENAS**（Pham 等） | 权重共享，把 NAS 从数千 GPU 时降到 ~1 GPU 时 | [arXiv:1802.03268](https://arxiv.org/abs/1802.03268) |
| 2019 | **DARTS**（Liu 等，ICLR 2019） | 架构搜索连续松弛，梯度化，数 GPU-day | [arXiv:1806.09055](https://arxiv.org/abs/1806.09055) |
| 2021 | **Zero-Cost Proxies**（Abdelfattah 等，ICLR 2021） | 免训练指标预测架构优劣，搜索成本趋近零 | [OpenReview](https://openreview.net/forum?id=0cmMMy8J5q) |
| 2011–2019 | **HPO 工具链**：Hyperopt(TPE)→Optuna(define-by-run)→Ray Tune(分布式/PBT/ASHA) | 调参工程化 | [Optuna](https://optuna.org/)、[Ray Tune](https://docs.ray.io/en/latest/tune/index.html)、[对比综述 arXiv:2311.15854](https://arxiv.org/abs/2311.15854) |
| 2018–2021 | **AutoML 产品化**：AutoKeras、H2O AutoML、Google Cloud AutoML/Vertex AI | 低代码建模平台 | [AutoKeras JMLR](http://jmlr.org/papers/v20/19-290.html)、[Vertex AI](https://cloud.google.com/blog/products/ai-machine-learning/google-cloud-launches-vertex-ai-unified-platform-for-mlops) |

### 2.2 思想源头：AI-GAs（Jeff Clune, 2019）

AI4AI 概念最重要的思想前驱是 Clune 的 **AI-Generating Algorithms**（[arXiv:1905.10985](https://arxiv.org/abs/1905.10985)，注意常见误引为 1906.05833）。其主张：通往通用智能的路径不是手工设计 AI，而是设计"能生成 AI 的算法"，三个支柱：
1. **Meta-learning architectures**（自动学习架构，即 NAS/ADAS 一脉）
2. **Meta-learning algorithms**（学习"学习算法"本身）
3. **Generating effective training environments**（开放式地自动生成训练环境/课程）

三者需运行在一个自改进回路中。后续的 ADAS（Automated Design of Agentic Systems，[arXiv:2408.08435](https://arxiv.org/pdf/2408.08435)）与 Sakana 的 Darwin Gödel Machine 是这一愿景在 LLM 时代的实操延续。更早的渊源可追溯到 Schmidhuber 1987 年起的 "learning to learn / 递归自我改进" 工作（[Schmidhuber metalearning](https://people.idsia.ch/~juergen/metalearning.html)）。

### 2.3 AutoML 时代的局限——为什么它不是 AI4AI

综合多个综述（[AutoML systematic review 2024](https://www.sciencedirect.com/science/article/pii/S2949715923000604)、[Frontiers in AI 2025](https://www.frontiersin.org/journals/artificial-intelligence/articles/10.3389/frai.2025.1590105/full)）：

- **搜索空间由人预定义**：只能在人类给定的算法/架构候选集内搜索，无法提出新范式；
- **覆盖面窄**：不做问题定义、创意性特征工程、实验设计、结果解释、研究方向迭代；
- **pipeline 结构固定**：不能按问题自适应重组研究流程；
- **无科学闭环**：不产生假设、不写论文、不接受同行评议。

**过渡期标志**：MLAgentBench（Huang, Vora, Liang, Leskovec；[arXiv:2310.03302](https://arxiv.org/pdf/2310.03302)，ICML 2024）首次让 LLM agent（ReAct 框架）在 13 个 ML 研究任务上端到端做实验；最佳配置 Claude 3 Opus 平均成功率 37.5%，瓶颈为长期规划与幻觉。它标志着范式从"自动化 pipeline 窄环节"转向"agent 自主科研闭环"。

---

## 3. 方法分类（Taxonomy）

按自动化程度与作用环节，LLM 时代的 AI4AI 系统可分为四类：

```
AI4AI
├── A. 端到端科研智能体（idea → paper 全流程）
│     ├── AI Scientist v1/v2（Sakana）
│     ├── Zochi（Intology）
│     └── FutureHouse 平台（Crow/Falcon/Owl/Phoenix/Finch/Robin）+ Kosmos（Edison Scientific）
├── B. ML 工程智能体（聚焦实验/建模环节）
│     ├── AIDE（WecoAI）：代码空间树搜索
│     ├── MLE-STAR（Google）：web 检索 + 嵌套代码精修
│     └── OpenHands / MLAB 等通用 agent 跑 MLE-bench
├── C. 假设生成与科学发现协作系统（human-in-the-loop）
│     └── Google AI co-scientist（Gemini 2.0 多智能体 + 锦标赛）
└── D. 元层：自我改进 / 自动设计智能体
      ├── STaR / Quiet-STaR、Self-Rewarding LM、Absolute Zero Reasoner、SEAL
      ├── ADAS（自动设计 agentic 系统）
      └── Darwin Gödel Machine（自我改进的编码 agent）
```

配套的评价层（基准）与治理层（评审政策）见 §5、§6。

---

## 4. 代表性系统详解（2024–2026 主线）

### 4.1 Sakana AI — The AI Scientist v1 / v2

**v1**（2024-08-13；[博客](https://sakana.ai/ai-scientist/)、[arXiv:2408.06292](https://arxiv.org/abs/2408.06292)）：首个宣称覆盖"想法→代码→实验→图表→论文→自动评审"全流程的系统，每篇论文计算成本 **< $15**；但依赖**人工编写的实验代码模板**，且自动评审分数虚高（自评分与真实评审相关性差）。

**v2**（2025-04-07；[arXiv:2504.08066](https://arxiv.org/abs/2504.08066)、[GitHub](https://github.com/SakanaAI/AI-Scientist-v2)），与 Oxford/UBC 合作，关键改进：
- **去模板化**：experiment manager agent 自主写实验代码；
- **Agentic tree search**：实验阶段一个 idea 分支为多条并行路径，进度式选择最优分支；
- **VLM 集成**：迭代改进图表；用 reasoning model 驱动 reflection 写作。

**ICLR 2025 workshop 事件**（2025-03-12 公告；[Sakana 博客](https://sakana.ai/ai-scientist-first-publication/)）：
- 向 ICLR 2025 **ICBINB workshop**（"I Can't Believe It's Not Better"，主题为有信息量的负结果）投稿 3 篇全 AI 生成论文（共 43 篇投稿），1 篇被接收：《Compositional Regularization: Unexpected Obstacles in Enhancing Neural Network Generalization》，双盲人类评审打分 **6/7/6（均分 6.33）**；
- 经 ICLR 领导层与 workshop 组织者事前同意、UBC IRB 伦理审批；按事前协议，**接收后撤回并程序性 desk-reject**，未正式发表；
- 审稿人仅被告知"3/43 篇可能是 AI 生成"，不知具体是哪篇；
- 注意：这是**接收率约 60–70% 的 workshop**，Sakana 自评 3 篇均未达主会水平；被接收论文的核心结论（组合正则化未带来预期提升）恰好契合负结果主题。

**系统论文后续**：v2 系统论文以《Towards end-to-end automation of AI research》发表于 Nature（2026；[s41586-026-10265-5](https://www.nature.com/articles/s41586-026-10265-5)）。媒体常混淆"AI 论文被 workshop 接收"与"系统论文登 Nature"两件事。

**社区批评**：idea 幼稚、代码错误、引用幻觉、评测数据集过窄（合成算术表达式）；第三方人类审阅见 [Akshat Gupta 的批评](https://akshat101.substack.com/p/i-reviewed-a-15-ai-generated-paper)。

### 4.2 Google — AI Co-Scientist（2025-02-19）

[官方博客](https://research.google/blog/accelerating-scientific-breakthroughs-with-an-ai-co-scientist/)、[arXiv:2502.18864](https://arxiv.org/abs/2502.18864)。定位与 AI Scientist 不同：**human-in-the-loop 的假设生成与排序系统**，不写完整论文。
- 基于 **Gemini 2.0**；六类专职 agent：Generation / Reflection / Ranking / Evolution / Proximity / Meta-review，由 Supervisor agent 调度，采用 test-time compute scaling；
- **锦标赛机制**：假设间进行基于 self-play 的科学辩论，用 Elo 排名；Elo 与 GPQA-diamond 准确率正相关；
- **生物医学验证**（属 AI4Science 交叉）：AML 药物重定位（KIRA6 在 KG-1 细胞系达临床相关浓度抑制）、肝纤维化表观遗传靶点（3D 肝类器官验证，p<0.01）、抗菌耐药机制假设（cf-PICI）——第三例与合作团队未公开的已验证结论独立吻合；
- 专家评估：11 个开放研究目标上，人类专家认为其输出新颖性与影响力潜力更高；
- 正式论文发表于 Nature 2026（[s41586-026-10644-y](https://www.nature.com/articles/s41586-026-10644-y)，Gottweis 等）。

### 4.3 ML 工程智能体：AIDE 与 MLE-STAR

**AIDE**（WecoAI；[arXiv:2502.13138](https://arxiv.org/html/2502.13138v1)、[GitHub](https://github.com/wecoai/aideml)）：把 ML 工程建模为**代码空间上的树搜索**——节点是 Python 脚本，策略在 draft / debug / improve 三种算子间决策，每次只做一个原子变更，配合摘要算子压缩历史。MLE-bench 成绩（pass@1，o1-preview 底座）：有效提交 82.8%、**铜牌及以上 16.9%（约 7 金）**，约为 OpenHands（4.4%）的 4 倍；MLE-bench Lite 子集上加 AIDE 后奖牌率 7.6%→36.4%、金牌率 6.1%→21.2%（均 p<0.01）。局限：Kaggle 数据污染风险、贪心搜索易陷局部最优。

**MLE-STAR**（Google，NeurIPS 2025；[论文](https://papers.nips.cc/paper_files/paper/2025/file/a9619dd0f0d54a5cf7734add1dc38cd1-Paper-Conference.pdf)、[博客](https://research.google/blog/mle-star/)）：web 检索选择建模方案 + 嵌套式代码块精修 + 自改进集成，在 MLE-bench Lite 上以 Gemini-2.0-Flash 达 **64% 奖牌率**。

**澄清**："OpenAI MLE-agent 开源框架"一说**未核实**——OpenAI 官方发布的是 MLE-bench 基准本身；MLE-bench 论文中实测的开源 agent 是 AIDE、MLAB (ResearchAgent)、OpenHands。市面上名为 "MLE-Agent" 的项目是无关的社区工具。

### 4.4 想法生成（Idea Generation）

**Stanford Si / Yang / Hashimoto**（[arXiv:2409.04109](https://arxiv.org/abs/2409.04109)，2024-09）：100+ NLP 研究者（36 机构）的大规模盲评研究，LLM agent（Claude 3.5 Sonnet + Semantic Scholar RAG）对比人类专家：
- **新颖性显著更高**（p<0.05），excitement 也更高；**feasibility 略弱**；
- **关键缺陷：多样性不足**——多次运行生成相似想法；LLM 自评不可靠。
- 注：任务书中提到的"~1000 名研究者的大规模跟进工作"**未核实**；相反方向的补充证据来自 Liu 等（[arXiv:2610.17439](https://arxiv.org/abs/2610.17439)，2026-10）：LLM 想法与已有工作距离更远（最多 +16%），但"正交新颖性"中位数反而低 15–23%，即倾向延伸已有方向而非开辟新范式。

**AIdeaBench**（Qiu 等，非 FutureHouse；[arXiv:2504.14191](https://arxiv.org/abs/2504.14191)、[主页](https://ai-idea-bench.github.io/)）：3,495 篇 AI 会议高影响力论文及对应生成想法，评测 LLM 研究想法质量（"想法是否值得做"）。

### 4.5 FutureHouse：平台化科研智能体

非营利机构（Eric Schmidt 资助）。2025-04/05 发布 [agent 平台](https://www.futurehouse.org/research-announcements/launching-futurehouse-platform-ai-agents)：**Crow**（文献检索）、**Falcon**（文献评审）、**Owl**（深度调研）、**Phoenix**（实验/合成设计）、**Finch**（notebook 内自主数据分析）、**Robin**（多 agent 编排器）。LitQA 上搜索精度超过 o3-mini、GPT-4.5、Claude 3.7 及 PhD 级人类。

**Robin 案例**（[arXiv:2505.13400](https://arxiv.org/abs/2505.13400)，2025-05）：约 10 周完成干性 AMD 药物再定位的干湿闭环——Crow 综述 151 篇文献→提出"增强 RPE 吞噬"机制假设→再分析 ~400 篇→合成 30 个候选→排序→湿实验验证。（媒体常提的 ROCK 抑制剂 ripasudil 与该工作的直接关联**部分未核实**，以原文为准。）

**Kosmos**（2025-11，分拆公司 Edison Scientific）：宣称单次任务 12 小时读 ~1,500 篇论文、生成 ~42,000 行分析代码，复现准确率 ~79.4%（厂商宣称，**独立验证缺失**）。

**澄清**：任务书中"FutureHouse AI Research Bench（108 篇复现基准）"**未核实存在**；FutureHouse 实际基准为 **LAB-Bench**（[GitHub](https://github.com/Future-House/LAB-Bench)，NeurIPS 2024，2,400+ 生物科研选择题）与 **BixBench**（53 个真实生信分析场景，GPT-4o/Claude 3.5 开放题仅 9%/17%）。

### 4.6 Zochi（Intology）：AI 论文进主会

2025-05-29 宣布：Zochi 生成的论文《Tempest: Automatic Multi-Turn Jailbreaking of LLMs with Tree Search》（[arXiv:2503.10619](https://arxiv.org/abs/2503.10619)）被 **ACL 2025 主会**接收，最终分 4、位列投稿前 8.2%；Intology 称选题、方法、代码、实验、写作全部由 Zochi 完成，人类仅做图表、排版与伦理审查（[报道](https://36kr.com/p/3313932314257154)）。这是"AI 生成论文进入顶级主会"的首个公开宣称案例，直接推动了会议披露政策收紧（§6）。

---

## 5. 评测基准（Evaluation Benchmarks）

| 基准 | 机构/时间 | 内容 | 关键结果 | 来源 |
|---|---|---|---|---|
| **MLAgentBench** | Stanford, 2023-10 (ICML 2024) | 13 个 ML 研究任务，agent 端到端做实验 | 最佳 Claude 3 Opus 成功率 37.5%；瓶颈=长期规划+幻觉 | [arXiv:2310.03302](https://arxiv.org/pdf/2310.03302) |
| **MLE-bench** | OpenAI, 2024-10 (ICLR 2025) | 75 个 Kaggle 竞赛，铜/银/金牌线 | o1-preview+AIDE：pass@1 **16.9% ≥铜牌**（~7 金）；pass@8 >30% | [arXiv:2410.07095](https://arxiv.org/pdf/2410.07095)、[博客](https://openai.com/index/mle-bench/) |
| **MLE-bench Lite** | OpenAI, 2025 | 22 个低复杂度任务子集 | AIDE+o1-preview 奖牌率 36.4%；MLE-STAR 64%；[第三方榜单](https://benchlm.ai/benchmarks/mle-bench-lite)持续更新 | 同上 |
| **RE-Bench** | METR, 2024-11 | **7 个** AI R&D 环境，人类专家基线（61 人×8h） | **2h 时 agent ≈ 人类 4 倍；8h 人类反超；32h 人类 ≈ 2 倍**；agent 提交方案速度快 10 倍+ | [arXiv:2411.15114](https://arxiv.org/abs/2411.15114) |
| **PaperBench** | OpenAI, 2025-04 (ICML 2025) | **20 篇** ICML 2024 Spotlight/Oral 复现，8,316 个 rubric 子任务 | 最佳 agent（Claude 3.5 Sonnet New）**21.0%**；人类 ML 博士（48h/篇）≈ **42%**；核心发现=comprehension-execution gap | [arXiv:2504.01848](https://arxiv.org/abs/2504.01848) |
| **MLGym / MLGym-Bench** | Meta, 2025-02 | 13 个开放式 AI 研究任务（CV/NLP/RL/博弈论） | 唯一含算法研究类任务的 gym 式基准 | [arXiv:2502.14499](https://arxiv.org/abs/2502.14499) |
| **AIdeaBench** | Qiu 等, 2025-04 | 3,495 篇论文的研究想法质量评测 | LLM 判"想法是否值得做"仍不可靠 | [arXiv:2504.14191](https://arxiv.org/abs/2504.14191) |
| **LAB-Bench / BixBench** | FutureHouse | 生物科研基础能力 / 生信数据分析 | BixBench 开放题 GPT-4o 9%、Claude 3.5 17% | [LAB-Bench](https://github.com/Future-House/LAB-Bench) |
| **DSBench / DABstep / SciCode / HAL** | 多方 | 数据分析建模 / 数据 agent / 科学代码 / 综合 agent 榜单 | HAL（Princeton）提供成本感知的第三方排名 | [DSBench](https://arxiv.org/abs/2409.07703)、[HAL](https://hal.cs.princeton.edu/) |

> 常见讹传纠正：PaperBench **不是** 250 篇（是 20 篇论文、8,316 子任务）；"o3-mini-high 24-26%"不确——o3-mini-high 主要作裁判模型（JudgeEval F1≈0.83），作参赛者 <10%；"树搜索 41%"疑为人类基线 ~42% 的误记。RE-Bench 是 7 个环境而非 8 个。

**基准揭示的三条规律**：
1. **脚手架决定成败**：同一模型加 AIDE 后奖牌率翻数倍（MLE-bench Lite：7.6%→36.4%）；
2. **时间尺度不对称**（RE-Bench）：短时窗 agent 碾压人类，长时窗人类反超——agent 缺乏长程策略调整能力；
3. **理解≠执行**（PaperBench）：模型能正确描述论文方法，却难以产出可运行代码库与正确实验结果。

### 5.1 机器人科学家与自驱动实验室（AI4Science 交叉，简述）

Adam/Eve（Ross King 团队，2009 年 Adam 成为[首台自主发现新科学知识的机器](https://www.cam.ac.uk/research/news/robot-scientist-becomes-first-machine-to-discover-new-scientific-knowledge)）、Coscientist（CMU，GPT-4 驱动化学实验，[Nature 2023](https://engineering.cmu.edu/news-events/news/2023-12-20-ai-coscientist.html)）、A-Lab（Berkeley 全自动材料合成）。综述见 [Autonomous Agents for Scientific Discovery（arXiv:2510.09901）](https://arxiv.org/html/2510.09901v1)、清华 [Awesome-AI-Scientists](https://github.com/tsinghua-fib-lab/Awesome-AI-Scientists)。警示案例：Nature "Robot Chemist" 论文 2026-01 被勘误，显示自动实验室同样面临可复现性问题。

---

## 6. 自动化写作、同行评审与科研诚信

### 6.1 评审污染已成事实

- ICLR：Pangram Labs/EditLens 对 ~75,800 条评审的分析显示 **21% 被判"完全 AI 生成"**，35% 有不同程度 AI 修改；AI 评审均分（4.43）反而高于人类（4.13）且更长（~3,700 字符）；
- 实证研究（[arXiv:2512.22145](https://arxiv.org/html/2512.22145v1)，2025-12）：LLM 评审与人类评审相关性仅 **0.15**，系统性高估 3–5 分；
- LLM 评审一致性/检测研究生态：ReviewerToo、[AutoRev（arXiv:2505.14376）](https://arxiv.org/html/2505.14376v1)、"Is Your Paper Being Reviewed by an LLM?"、以及批评"paper laundering"（LLM 改写论文以抬高 AI 评审分）的 [STOP AUTOMATING PEER REVIEW WITHOUT RIGOR](https://openreview.net/pdf?id=cJhlquXIuS)。

### 6.2 会议政策收紧（2025–2026）

- **ICLR 2026**（[2025-11-19 政策](https://blog.iclr.cc/2025-11-19/iclr-2026-response-to-llm-generated-papers-and-reviews/)）：LLM 使用须披露并负责；大量未披露使用→desk reject；**幻觉引用=伦理违规**；滥用 AI 的审稿人其本人投稿将被 desk reject；
- **ICML 2026**（[政策博客](https://blog.icml.cc/2026-03-18/on-violations-of-llm-review-policies/)）：据 [The Scientist](https://www.the-scientist.com/a-trap-for-ai-use-in-peer-reviews-sparks-controversy-74702) 报道 desk-reject 近 500 篇（约 2%），检测方法本身引发争议；
- **NeurIPS 2026**（[博客](https://blog.neurips.cc/2026-06-02/ai-generated-papers-in-the-neurips-2026-position-paper-track/)）：称 AI 生成论文对评审系统构成 "acute risk"；背景：NeurIPS 投稿量从 2016 年 2,425 涨到 2024 年 15,671。

### 6.3 投稿端泛滥

arXiv 于 2025-11 起对 AI 生成的 survey 类投稿加强筛查（[报道](https://decrypt.co/347196/arxiv-blocks-ai-generated-survey-papers-flood-trashy-submissions)、[arXiv:2510.09686 "Stop DDoS Attacking the Research Community with AI"](https://arxiv.org/html/2510.09686v1)）；Nature 评论称 AI slop 正在计算机学界制造危机（[d41586-025-03967-9](https://www.nature.com/articles/d41586-025-03967-9)）；Pangram 跟踪显示 AI 辅助论文使投稿量同比 +42%。另有研究警告自动科研系统为追求"可发表性"会系统性引入 benchmark 选择偏差、数据泄漏、p-hacking（CMU Nihar Shah 团队，PNAS；**具体篇目未在本轮直源核实**）。

写作工具生态（Paperpal、SciSpace、Writefull、Overleaf AI）已高度普及，属"辅助润色"而非"端到端生成"，争议较小。

---

## 7. 交叉方向：自我改进（Self-Improvement）

自我改进是 AI4AI 的"元层"——AI 研究能力最终要能反哺自身：

| 工作 | 时间 | 核心机制 | 来源 |
|---|---|---|---|
| **STaR**（Zelikman 等） | 2022 | 自生成 rationale→筛选正确答案→SFT 迭代；CommonsenseQA 达 30× 大模型微调水平 | [arXiv:2203.14465](https://arxiv.org/abs/2203.14465) |
| **Quiet-STaR** | 2024 | 每个 token 处隐式思考 | [arXiv:2403.09629](https://arxiv.org/abs/2403.09629) |
| **Self-Rewarding LM**（Meta） | 2024-01 (ICML 2024) | 模型兼任 LLM-as-Judge 给自己打分→DPO 迭代；主张"超人智能需要超人反馈" | [arXiv:2401.10020](https://arxiv.org/abs/2401.10020) |
| **Absolute Zero Reasoner** | 2025-05 | 同一 LLM 兼任 Proposer+Solver，Python 解释器作可验证环境，零人类标注达 SOTA | [arXiv:2505.03335](https://arxiv.org/abs/2505.03335) |
| **SEAL**（MIT） | 2025-06 | self-edit：模型生成自己的微调数据/超参并持久更新权重，RL 训练"产生有效 self-edit 的能力" | [arXiv:2506.10943](https://arxiv.org/pdf/2506.10943) |
| **Darwin Gödel Machine**（Sakana） | 2025 | 自我改进编码 agent 的开放式进化档案 | [sakana.ai/dgm](https://sakana.ai/dgm/) |

**递归自我改进（RSI）的实证与安全讨论**：
- Anthropic Institute《When AI Builds Itself》（2026-06，[链接](https://www.anthropic.com/institute/recursive-self-improvement)）披露：Anthropic 仓库 **>80% 合并代码由 Claude 编写**；Claude agent 累计自主完成 ~800 小时开放式 AI 安全研究；Claude 可在人类监督下主导 **26% 的 AI R&D 任务**（2026-02 时 <1%）；联合创始人 Jack Clark 预测 2028 年底前实现无人参与递归改进的概率 >60%；
- Dario Amodei《Machines of Loving Grace》（2024-10，[原文](https://darioamodei.com/machines-of-loving-grace)）："压缩的 21 世纪"——强大 AI 可能在 5–10 年内压缩整个 21 世纪量级的科学进步；
- 数学领域的信号：Google DeepMind 的 Gemini Erdős 问题案例研究（agent "Aletheia" 探索 ~700 个开放问题，13 个取得进展，但 ~200 个候选中 68.5% 有根本错误；arXiv 编号 **2601.22401 建议二次核验**）；2025-10 OpenAI 宣称 GPT-5 "解决" 10 个 Erdős 开放问题，事后多数被发现是检索到既存文献而非独立求解（Hassabis 评价 "embarrassing"）——**"AI 解决开放问题"的宣称需要严格甄别**；2026 年 OpenAI 批量发布 AI 辅助数学论文（一次 722 篇）引发 25 位菲尔兹奖得主联署批评，陶哲轩协助建立社区 wiki 追踪（[报道](https://eu.36kr.com/en/p/4019262571663237)）。

---

## 8. 开放问题与争议

1. **新颖性与多样性悖论**：LLM 想法在盲评中"更新颖"，但跨运行多样性不足、倾向延伸已有方向而非开辟范式（§4.4）。自动科研可能加速"增量科学"的通货膨胀。
2. **数据污染与评测效度**：Kaggle 竞赛、arXiv 论文大量存在于训练语料中，MLE-bench/PaperBench 成绩的"真实泛化"存疑；AIDE 论文自身承认此风险。2026 年出现"Clean Track"类应对（**未核实细节**）。
3. **Comprehension–Execution Gap**：PaperBench 的核心发现——读懂≠做出。端到端复现仍显著落后人类博士（21% vs 42%）。
4. **长时间尺度的自主性**：RE-Bench 显示 32 小时时限下人类约为顶级 agent 的 2 倍；agent 缺乏长程规划、策略级纠错能力（与 MLAgentBench 结论一致）。
5. **评审系统的可信度危机**：21% 全 AI 评审、LLM-人类评审相关性 0.15、检测器对"润色级"使用误判率高——治理工具跑不过滥用速度。AI Scientist-v2 通过 workshop 评审更说明现行评审无法区分人机论文。
6. **科研诚信与披露**：署名问题（AI 不能署名，但贡献如何披露）、双盲评审与 AI 披露的张力、"可发表性优化"导致的系统性学术不端风险（p-hacking、benchmark 挑选）。
7. **安全：研究能力即风险放大器**：RSI 实证数据（Anthropic 26% 任务主导权、80% 代码）显示自动化程度快速上升；AI 自主研究若失去人类验证环节，错误会以论文/代码的形式规模化传播（Erdős "假解决"事件、Robot Chemist 勘误是先兆）。
8. **科学价值的最终裁判**：AI Scientist 论文通过的是低门槛 workshop 评审；"通过评审"≠"产生知识"。领域缺乏对"AI 生成研究是否带来真实科学增量"的严格评估（Stanford 团队提出的 end-to-end 执行式评估是方向之一）。

---

## 9. 趋势判断（2026→）

- **系统论文进入主流学术渠道**：AI Scientist-v2 与 AI co-scientist 的系统论文先后登上 Nature，标志 AI4AI 本身成为被严肃评审的研究对象；
- **从"全自动论文"转向"人机协作分工"**：工业界主流（Google co-scientist、FutureHouse）选择 human-in-the-loop 的假设生成/文献综合/实验编排，而非端到端论文工厂；
- **基准快速演化**：从 Kaggle 式封闭任务（MLE-bench）走向开放式研究环境（MLGym、RE-Bench）、复现（PaperBench）与想法价值（AIdeaBench）；2026 年出现 DSAgentBench（真实计算环境端到端，最强模型 56.7%）等更贴近真实工作流的新基准；
- **元层闭环加速**：ADAS/DGM/SEAL 一脉与科研智能体合流——"自动设计更好的科研 agent"是 AI-GAs 愿景的直接延续；
- **治理与技术赛跑**：会议 desk-reject 政策、arXiv 筛查、评审检测工具将在 1–2 年内决定学术出版体系能否吸收 AI 生成内容的冲击。

---

## 附：未核实/存疑项清单（引用前请二次核验）

1. "OpenAI MLE-agent 开源框架"——未核实存在，疑与 MLE-bench 或社区项目 MLE-Agent 混淆；
2. Stanford "~1000 名研究者" 的想法生成跟进论文——未找到；
3. FutureHouse "AI Research Bench（108 篇复现）"——未核实；实际为 LAB-Bench/BixBench，AIdeaBench 属另一团队；
4. FutureHouse agent 名称 "Bob/Zen/Papert"——未在核实来源中出现；
5. ripasudil (RIP-1) 与 FutureHouse Robin 的直接关联——中文媒体提及，需查 arXiv:2505.13400 原文；
6. GPT-5/5.5、Gemini 3.x 在 MLE-bench 的官方具体分数——媒体数字未获官方来源确认；
7. ICBINB workshop 组织者的独立公开声明——未检索到；
8. DeepMind Erdős 案例研究 arXiv:2601.22401——编号来自检索转述；
9. Kosmos 的 ~79.4% 复现准确率——厂商宣称，无独立验证；
10. ICLR 2025 是否存在名为 "AI Generated Papers: Unveiling The Reviewing Process" 的 workshop——未核实（接收 Sakana 论文的确认为 ICBINB）。

---

## 方法论说明

由 4 个并行研究子代理分别调研：①AutoML/NAS 前史；②LLM 科研智能体主线；③评测基准与自动实验室；④评审争议与自我改进。检索工具为 WebSearch + WebFetch（无 firecrawl/exa MCP），优先一手来源（arXiv、官方博客、GitHub、Nature）。共执行 20+ 组关键词搜索、约 80 条来源，其中关键数字（日期、分数、成本）均经一手来源交叉核实；子代理间交叉纠错 3 处（PaperBench 规模、RE-Bench 任务数、AI-GAs arXiv 编号）。所有未能核实的论断已在上文标注。
