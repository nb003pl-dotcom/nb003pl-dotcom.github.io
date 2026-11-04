---
title: "从写出答案到直接给概率：Jev 式决策模型与金融应用全景综述"
date: 2026-11-03T09:00:00+08:00
draft: false
tags: [Jev, 决策模型, 概率预测, 金融AI]
comments: true
---

# 从"写出答案"到"直接给概率"：Jev 式决策模型与金融应用全景综述

*生成日期：2026-11-03 | 定位：面向学习者的中文技术博客式综述 | 引用来源约 50 个*
*证据分级沿用本项目 [research_scope.md](./research_scope.md)：A 同行评审/可复现独立评测；B 方法数据较完整的预印本；C 官方文档/厂商实验；D 部分可复现的独立报告；E 博客/社区/宣传材料*

---

## TL;DR

2026 年 9 月，前 OpenAI 研究员 Diogo Almeida 创立的 TypeSafe AI 发布了 **Jev**——一种不生成任何文本、直接在候选项上输出概率分布的"System One 决策模型"。两个月内，它引爆了一个全新赛道：OpenAI、Microsoft、Cloudflare、Liquid AI、蚂蚁、上海 AI Lab、NUS 纷纷下场，arXiv 上出现了几十篇 Jev 式模型论文，开源实现超过 20 个。

但金融领域的证据要冷静得多：**JEVal 基准（11,257 题、10 领域）显示，在金融、医疗、法律这类专业知识域，最佳决策模型仍比最佳生成式 LLM 低 3–6 个百分点**；决策模型虽然 top-1 置信度校准极好（ECE 低至 0.0149——ECE 即期望校准误差，衡量"模型报告的置信度与实际正确率之间的差距"，0 为完美校准，指标详解见 §4.5），却在受控概率任务中**严重高估最可能结果的概率**（把真实 35.6% 的选项标到 89.35%）；放进多步智能体流程后，**系统成功率反而从 77.6% 跌到 66.1%**。

一句话判断：**Jev 式模型是"决策即接口"的范式创新，在金融领域目前处于"值得评测的组件"阶段，而非"已验证的决策方案"**。它最适合的位置是高频、答案空间有限、允许置信度门控的前置分流环节——而不是信贷、投资或合规的最终裁决者。

---

## 一、为什么突然冒出"决策模型"：一场接口革命

### 1.1 问题的起点：大量 LLM 调用的终点根本不是文本

今天生产系统里的 LLM 调用，相当大比例的最终目的不是"一段话"，而是**一个判断**：这条工单该路由到哪个队列？这笔交易是否可疑？这份草稿打几分？传统做法是让 LLM 生成文本（或 JSON），再解析出答案——付出了生成 token 的延迟和成本，还要承担格式解析失败的风险。

TypeSafe AI 的提问是：**当任务的终点是决策时，为什么不从决策本身出发设计模型？**（[jevmodel.org](https://jevmodel.org/)，证据 C）

这就是 Kahneman《思考，快与慢》里 System 1（快思考）的工程化：LLM 是 System Two，负责推理与生成；决策模型是 System One，负责在毫秒级给出类型化判断。"Jev"这个名字取自经济学家 Jevons 的**杰文斯悖论**——智能越便宜，系统调用它的频率越高，总消耗反而越大；决策模型正是要把单次"判断"的价格打下来（[知乎·Jev 从语言生成到概率决策](https://zhuanlan.zhihu.com/p/2093523544483407367)，证据 E）。

### 1.2 两个月的赛道爆发（时间线）

| 时间 | 事件 | 来源/证据 |
|---|---|---|
| 2026-09-15 | TypeSafe 发布 Jev，托管 API；$0.042/百万输入 token，输出免费，端到端延迟 70–500ms | [jevmodel.org](https://jevmodel.org/)（C） |
| 2026-09 | A 轮 $8.7 亿、估值约 $75 亿，a16z 领投，红杉/DCVC 跟投——"今年最贵 A 轮"；公司约 20 人 | [百家号报道](https://baijiahao.baidu.com/s?id=1879532892229523279)（E） |
| 2026-09-29 | Liquid AI 发布 d1-3B（LFM2.5-VL-3B 底座，零输出 token），成为首个在 HuggingFace Decision Index 上超过 Jev 的模型（48.57 分） | [Ollama d1](https://ollama.com/library/d1)（C） |
| 2026-09-30 | arXiv 出现序数偏差研究：JEV 在 36 个序数数据集上系统性压缩评分尺度 | [arXiv 2609.38827](https://arxiv.org/abs/2609.38827)（B） |
| 2026-10-01 | LLM-as-Jev：证明普通 LLM 免训练即可当决策模型用 | [arXiv 2610.02076](https://arxiv.org/abs/2610.02076)（B） |
| 2026-10-09 | Microsoft 发布 Decision-1（Azure Model Catalog），输出校准概率分布 | [Microsoft 公告](https://commandline.microsoft.com/microsoft-decision-1-model-foundry/)（C） |
| 2026-10 | 上海创新研究院发布 JEVal 基准与 SII-Decision；上海 AI Lab 开源 InternDecision；Cloudflare 开源 Clef（27B/9B） | [arXiv 2610.03935](https://arxiv.org/abs/2610.03935)（B） |
| 2026-10-22 | OpenAI Decisions API 在 Microsoft Foundry GA（GPT-6 Luna 底座，$0.10/M 输入）；任务延迟从 1–3s 降到 100–180ms | [OpenAI 文档](https://developers.openai.com/api/docs/guides/decisions)（C） |
| 2026-10 | NUS xML Lab 开源 Jeva-4B：从第一性原理设计的决策原生架构 | [arXiv 2610.18773](https://arxiv.org/html/2610.18773)（B） |
| 2026-10 | AWS（Strands Decider）、Perplexity（Decider v1.1/v1.2）、Snowflake、蚂蚁数科（LingDT-Decision-1.0）、StartLux 等相继入场 | [decisionmodels.io](https://decisionmodels.io/)（C）、CB Insights（E） |

**趋势判断**：这是自"结构化输出"以来 LLM 应用接口最重要的一次收敛——**把"判断"从生成任务中剥离出来，做成一种独立的模型品类和 API 原语**。商业上它对标的是规则引擎、判别式小模型和 LLM 结构化调用三方市场。

---

## 二、方法：Jev 式模型是怎么工作的

### 2.1 统一的类型化接口

尽管实现五花八门，Jev 式模型共享同一个接口契约（[jevmodel.org](https://jevmodel.org/)，C；[Jeva 论文](https://arxiv.org/html/2610.18773)，B）：

- **输入**：`state`（背景文本/对话/环境状态）+ `instruction`(问题) + `candidates`（有限候选集）+ 决策类型
- **输出**：三种类型化结果，全部带概率，**不生成任何 token**
  - **Choice**：从候选集中选一项（含概率分布）
  - **Nul/Noul**：布尔判断（true/false 的概率）
  - **Score**：在有序评分档位上的分布（期望分）

与生成式 LLM 的本质差异：

| 维度 | 生成式 LLM | Jev 式决策模型 |
|---|---|---|
| 输出 | token 序列，需解析 | 候选项上的概率分布，直接可消费 |
| 答案空间 | 开放 | 封闭/类型化（编译期可校验） |
| 延迟 | 秒级（thinking 更久） | 毫秒级（Jev 托管 70–500ms；SII-Decision-4B <0.1s） |
| 成本 | 输入+输出双向计费 | Jev $0.042/M 输入、输出免费 |
| "不知道" | 靠提示词约束，易强行作答 | Null 可作为一等公民输出（Jeva 的 coverage gate） |
| 概率语义 | token logits，受解码策略污染 | 训练目标即分布，ECE 显著更低 |
| 专业知识 | 强（预训练知识+推理链） | **弱（金融/医疗/法律落后 3–6pp）** |
| 解释能力 | 可生成理由 | 无自由文本解释，需外部机制补审计 |

### 2.2 六条技术路线（架构谱系）

这是理解整个赛道最有用的分类框架——同样是"直接出概率"，实现路径至少有六条：

**路线一：商业黑盒 + 校准强化学习（Jev / TypeSafe）**
Jev 核心训练方法为 **RLCD（Reinforcement Learning for Calibrated Decisions，校准决策强化学习）**，主打概率校准与类型安全（C 级证据，官方口径；[CSDN 解析](https://blog.csdn.net/weixin_32181267/article/details/169514969)）。架构未公开。

**路线二：决策原生架构（Jeva / NUS xML Lab）**
[Jeva](https://arxiv.org/html/2610.18773)（B）是目前学术上最完整的"第一性原理"设计。它认为一个合格的决策模型必须满足四个结构性质：

1. **Null-aware**："没有可接受答案"本身是合法输出。Jeva 把"是否存在可接受候选"（coverage）与"哪个候选更好"（competition）拆成两个独立估计：`p⊥ = 1−σ((max sᵢ − b)/τ_cov)`，`pᵢ = (1−p⊥)·qᵢ`。这避免了平铺 softmax 中候选数 K 对 null 概率的强制 `log K` 污染。
2. **Permutation-invariant**：候选重排不改变决策。共享编码器独立编码每个候选（候选模板不含问题、索引和其他选项），对称聚合使概率只跟随候选身份。论文用归纳法证明置换只重排输出、不改变状态向量。**实测 0% 决策翻转**，而 Jev-1.13.0 为 17%、InternDecision-4B 为 22.67%、直接读 Qwen3.5 标签 logits 高达 84.5%。
3. **Option-scalable**：候选集可以从十几个扩展到几千个（ToolRet K=2048 仍保持 69.88% 准确率），成本对 K 线性。
4. **Cacheable**：候选的语义向量与投影可跨请求复用（政策条款、文档库比问题稳定得多），混合缓存下中位延迟从 2102ms 降到 75ms，**27.86 倍加速且不改变任何决策**。

架构上：Qwen3.5-4B 共享编码器 + 仅 **1337 万参数**的决策块（L=1、8 头、熵正则注意力只更新请求状态、读取不可变的候选记忆）。命题 1 证明单个决策块就能实现"独立打分器做不到的事"——同一对候选 A/B 的偏好可随集合中第三个选项变化而反转（集合依赖的竞争）。

**路线三：Pointer head（Kev / Jared Palmer；KEV 系）**
把请求和候选编成单序列，在候选末位 hidden state 上接一个指针头，softmax 直接给出每个候选的概率；单次前向、零生成。开源 Apache-2.0（[github.com/jaredpalmer/kev](https://github.com/jaredpalmer/kev)，0.8B/4B/9B，Qwen3.5 底座）。

**路线四：Decision markers（InternDecision / 上海 AI Lab）**
选项映射到单 token 符号（A–Z/a–z/0–9，最多 62 项），渲染带 `<decision>` 占位符的 JSON 骨架，**从不调用 generate()**，单次前向后读取占位符位置 logits 并做基于 checkpoint 的校准。0.8B/2B/4B，Apache-2.0；4B 在 RTX 4090 上约 44ms/题（C/D）。

**路线五：First-token readout + 自蒸馏（SII-Decision / 上海创新研究院）**
最激进的极简主义：**不加任何新参数**，直接把 LLM 在"The correct answer is ("位置的 next-token 分布限制在合法标签上作为决策输出，再用自蒸馏把 thinking 能力压进这一次前向（详见第四节）。这是 JEVal 论文作者自己的模型，也是当前**综合成绩最强的开权重决策模型**（B）。

**路线六：Encoder decision head（Laya 等小模型）**
不走 decoder 路线，直接用 ModernBERT 类编码器 + 决策头，~421M 参数，T4 上单查询 32.8ms（[github.com/NandhaKishorM/laya](https://github.com/NandhaKishorM/laya)，Apache-2.0）。代价是能力上限低：Jeva 论文测得 Laya-English 置换翻转率 66.83%；独立研究（arXiv 2609.33843）记录其"欠自信"（gap −0.214）（B/D）。

**彩蛋路线：免训练读出（LLM-as-Jev）**
[arXiv 2610.02076](https://arxiv.org/html/2610.02076)（B）证明 Qwen3.5-4B **不做任何微调**、仅用"选项后缀 log-prob 之和 + softmax"就能当决策模型：JevBench 81.4%、ECE 0.057；LoRA 微调（树分解 listwise loss + KL 锚定）提到 84.0%；允许 4096 token 思考的"Thinking Jev"达 94.8%，媲美 GPT-5.6 Sol。**这暗示"决策能力"本来就潜伏在 LLM 的 next-token 分布里，决策模型某种程度上是把它显式化、稳定化、廉价化。**

另有循环架构 **Lev**（arXiv 2610.20951，Kev 的学术后继）：同一份权重按题目难度决定"想几轮"，2.6B 从 1 轮 63.2% 提到 4 轮 76.2%——在 System One 里重新引入可调深度的 System Two（B）。

---

## 三、开源生态全景

社区已经出现了专门的证据综述仓库 [awesome-jev-survey](https://github.com/Eurekaleo/awesome-jev-survey)（v0.3.1，Meng Luo 维护，含 F1–F10 十条跨研究综合结论）。主要玩家：

### 3.1 开权重模型清单（截至 2026-11）

| 模型 | 开发方 | 规模 | 架构路线 | 许可 | 代表成绩 |
|---|---|---|---|---|---|
| **Jeva-4B** | NUS xML Lab | 4B | 决策原生架构 | 开源（[GitHub](https://github.com/LiQiiiii/Jeva)/[HF](https://huggingface.co/LIQIIIII/Jeva-4B)） | BANKING77 K=16 91.62%；null AUROC 98.66%；0% 置换翻转 |
| **SII-Decision-4B/27B** | 上海创新研究院 | 4B/27B | first-token readout + 自蒸馏 | 开权重（[HF: sii-research](https://huggingface.co/sii-research)） | JEVal 总分 79.24%（27B）、ECE 0.0149 全场最低 |
| **Kev-0.8B/4B/9B** | Jared Palmer | 0.8–9B | pointer head | Apache-2.0 | Kev-27B* JEVal 77.17%（*27B 段对应 Clef/SII 线，Kev 官方目录无 27B） |
| **InternDecision-0.8B/2B/4B** | 上海 AI Lab | 0.8–4B | decision markers | Apache-2.0 | 4B JEVal 70.74%，但 ECE 0.0152 极低 |
| **Laya** | 社区（NandhaKishorM） | ~421M | encoder 决策头 | Apache-2.0 | T4 32.8ms；稳定性弱（翻转率 66.83%） |
| **Liquid AI d1-3B / d1-omni-600M** | Liquid AI | 0.6/3B | 零输出 token | 开放权重 | 首个 Decision Index 超 Jev（48.57） |
| **Cloudflare Clef-27B / Clef-Flash-9B** | Cloudflare | 9/27B | Jev 兼容 `/v1/systemone` | 开源 | 多模态（文/JSON/图/视频），~209ms/38.8ms |
| **Nimble-9B** | Bespoke Labs | 9B | LoRA 微调 decoder | Other | JEVal 73.87% |
| **NanoJev-0.6B** | TianyuCodings | 0.6B | 微调 decoder | MIT | 端侧轻量 |
| **openjev 系**（daseinlabs、Zefan-Cai、zhihz、featherless simple-jev 26B-A4B、ekzhang openjev-sglang 35B-A3B 等） | 多个独立团队 | 0.8–35B | 各异 | MIT/Apache 为主 | 社区复现主力 |
| **AnyJev-8B** | Nokia Applied Research | 8B | Qwen3-8B | Apache-2.0 | 研究记录：20 选项时约 1/3 顺序翻转 |
| **Lev-2.6B** | Lehigh/UBC/Vector | 2.6B | 循环决策 | 开源（[HF: songdj/lev](https://huggingface.co/songdj/lev)） | 4 loops 76.2% JevBench |

（*注：网络流传的 "JevK5"（~4.2B，混合线性注意力，Apache-2.0）仅见 CSDN 描述，官方仓库直链未找到；"LightJev"、"JevForge-0.8B" 未找到可靠证据——引用需谨慎，证据 E。*）

### 3.2 大厂与商业服务

- **OpenAI Decisions API**：GPT-6 Luna 底座，$0.10/M 输入，100–180ms；客户案例：语言学习 App Speak 响应从 1.4s 降到 0.13s（C）
- **Microsoft Decision-1**：Azure Model Catalog，主打校准概率 + 延迟/质量平衡（C）
- **[decisionmodels.io](https://decisionmodels.io/)**：Jev 兼容 API 市场，托管 s1-fast/s1-pro/s1-vision/quyet-1-large 等；其 s1-pro 底层 vLLM-SR d3 曾列 HF Decision Index 0.3.2 第一（66.26 分）；提供 EU 层（GDPR/AI Act 对齐）（C）
- **HF Decision Index**（apolinario 维护）：事实上的决策模型公开排行榜（C）
- **蚂蚁数科 LingDT-Decision-1.0**（4B/27B）、**StartLux-Decision-27B**（Decision Index 69.1 分，自称超 Jev）、**Perplexity Decider**、**AWS Strands Decider**（C/E，"自称"类成绩均未经独立复核）

### 3.3 学术研究浪潮（arXiv，2026-09/10 两个月内）

- **[Jev-ToM](https://arxiv.org/html/2610.18610v1)**：11,117 题、9 个心智理论基准评测 5 个决策模型 vs 7 个生成基线；TypeSafe Jev 达 73.2% 源平衡准确率，成本远低于生成式 LLM（B）
- **[序数尺度偏差](https://arxiv.org/abs/2609.38827)**：JEV 在 ANLI 上把 38.8% 预测、51.3% 错误压在"Neutral"档；BA-LoRA 后训练可修复（B）
- **[检索审计](https://arxiv.org/html/2610.14631v1)**、**[实体匹配](https://arxiv.org/html/2610.34626v1)**、**[安全专用 SecJev](https://arxiv.org/abs/2610.03073)**（0.8B 专用超 9B 通用 20.5 分）、**[医疗 MEDJEV](https://github.com/PAI-CUHK/MEDJEV)**、**[Measure Before You Train](https://arxiv.org/html/2610.32955v1)**（泼冷水：简单 logistic 回归常在准确率上打平神经决策模型且快几个数量级）（B）
- 社区基准：JevBench（231 题）、[JEVal 数据集已上 HF](https://huggingface.co/datasets/carlosxiang/JEVal)、DecisionBench 等

**趋势判断**：生态演化路径与当年 LLM 惊人相似——商业闭源引爆（Jev ≈ GPT-3.5 时刻）→ 大厂 API 跟进 → 开权重百花齐放 → 专用化（医疗/安全/泰语/多模态）→ 反思性研究出现（偏差审计、"先测量再训练"）。整个周期只用了约 6 周。

---

## 四、训练方法与数据：如何造一个决策模型

### 4.1 SII-Decision：Reasoning-to-Readout 自蒸馏（当前最有信息量的配方）

来自 [JEVal 论文 §6](https://arxiv.org/html/2610.03935)（B），核心思想：**LLM 同时拥有 System One（next-token 分布）和 System Two（思维链），所以模型可以当自己的老师——不需要人工标签，也不需要更强的 teacher。**

流程：
1. 对每道训练题，开启 thinking 生成**两条独立推理链**；
2. 丢弃推理文本，在推理链末端用 first-token readout 取分布 q(y|x,r)；
3. teacher 分布 = 两条链的平均；**gold 标签从不进入训练目标**（下游消费的是概率而非 argmax，用标签会把分布压成 one-hot）；
4. 学生损失 = teacher-student 交叉熵 + 0.5×score 题期望等级误差 + KL 锚定到冻结原始 readout（κ=0.05，hard-label noul 题提高到 0.2）防遗忘；
5. 冻结 embedding 和输出层，只训 1 个 epoch。

数据：**39,279 题**（27B 版），按八组能力域覆盖选取（语言理解、通用知识、常识、逻辑、中英医学、规则决策与计算、安全与主观判断，27B 加科学/定量/专业/事实边界），作者观察到超过这个量级"增益停滞"——自生成题偏简单，模型也无法可靠校验更难的题。

关键实证：JEVal 上 thinking 给开源 LLM 加 6–9 分，蒸馏**在标签选择类任务上能收回大部分 thinking 增益**，但在 GPQA、MMLU-Pro 这类推理密集任务上只能收回四分之一以下——"单次前向无法复现多步中间计算"。

### 4.2 Jeva：十阶段课程学习 + 四类监督

[Jeva 论文](https://arxiv.org/html/2610.18773)（B）：Qwen3.5-4B 底座 + LoRA rank-64（scaling 128），决策块 13.37M 参数全量训练。监督信号混合**硬标签 + 软概率 + 显式 null + 数值评分**四类；主损失是 coverage 与 competition 的**联合交叉熵**（KL 可分解为"覆盖率误差 + 加权竞争误差"，null 样本只监督可用性、不施加虚假排序）。

十阶段课程（约 113 万样本总量）：基础类型化决策/意图/动作选择（8 万）→ 偏好比较与语义判断（50 万）→ 反事实验证（25 万）→ 证据锚定与开集决策（6 万）→ 阅读与逻辑（2 万）→ **困难样本+回放（6 万，其中 1.5 万按 7:3 混入 teacher 概率，且仅当 teacher 在两次独立打乱候选序下 top 预测都与原目标一致时才接受）** → 定向能力修补（1 万）→ 长文本与大候选池（8 万）→ 反馈驱动保持（0.2 万）→ 最终评分阶段+混合回放（7 万）。

注意这个细节：**teacher 指导要通过"置换一致性"检验才被采纳**——把第二节说的结构性质直接编码进了数据清洗流程。

### 4.3 LLM-as-Jev：免训练 + 树分解损失

（B）免训练读出已可用（81.4%）；若要微调，用树分解 listwise loss（只沿 gold 路径算概率，几百候选也高效）+ 三重 KL 锚定（合法质量/退出 token/位置分布）防止模型退化成强行续写文本——不加锚定时 4B 模型在 231 题里有 69 次直接开始"写小作文"。训练数据全部来自公开人工标注数据集（CLINC150、MASSIVE、ContractNLI、LogiQA 等），13-gram 去污染。

### 4.4 已知缺陷与修复：序数尺度偏差

[arXiv 2609.38827](https://arxiv.org/html/2609.38827v1)（B）发现 JEV/KEV 系在有序评分任务上系统性"压缩尺度"：36 个序数数据集只用了 gold 有效支持的 67–76%（名义分类任务为 87–102%）；档位从 K=2 细化到 K=14 时利用率最低掉到 26%。**BA-LoRA 后训练**（8 个强压缩源各 4000 样本、共 3.2 万）可把 gold 相对利用率从 ~47% 拉回 ~86%，KEV-0.8B 准确率 15.5%→28.5%、QWK 0.188→0.600，且对未见数据集有正迁移——说明**这是学出来的决策边界问题，不是架构天花板**。金融含义直接：信用评分、风险分级这类有序输出任务，裸用决策模型会向中间档塌缩。

### 4.5 校准：决策模型的招牌，也是它的陷阱

> **指标小讲座：什么是 ECE（Expected Calibration Error，期望校准误差）**
>
> 出自 [Guo et al., ICML 2017](https://proceedings.mlr.press/v70/guo17a.html)（A），是衡量模型"自信程度是否名副其实"的标准指标。校准（calibration）与准确率（accuracy）是两回事：准确率问"答对了多少"，校准问"模型说自己有多大把握时，这个把握可不可信"。一个 80% 置信度的预测，长期看就应该有约 80% 是对的——这才叫校准良好。
>
> **计算方法**：把测试集里所有预测按置信度分成若干区间（bin，常用 10–15 个，如 [0.7, 0.8)），每个区间内计算两个量——平均置信度与实际准确率，二者之差的绝对值就是该区间的校准误差；再按各区间样本数加权求和：
>
> $$\mathrm{ECE} = \sum_{b} \frac{|B_b|}{n}\,\bigl|\mathrm{acc}(B_b) - \mathrm{conf}(B_b)\bigr|$$
>
> **怎么读这个数**：ECE = 0 表示完美校准；ECE = 0.0149 意味着模型的置信度报告与实际正确率平均只差约 1.5 个百分点——说"我有 90% 把握"时，实际正确率大概率落在 88.5%–91.5% 附近。作为对照，现代生成式 LLM 普遍在 0.05–0.15 之间（JEVal 实测 GPT-6-Luna 为 0.1159），即置信度平均虚高/虚低 10 个百分点以上，基本不能直接当概率用。姊妹指标 **Brier score**（预测概率与真实标签的均方误差，同样越低越好）度量的是整个概率分布的质量，而 ECE 通常只看 top-1 置信度——这个差别正是下文"陷阱"的伏笔。
>
> **为什么金融场景特别在意校准**：分类模型只取 argmax 时，校准差一点无所谓；但金融下游（期望值计算、风险加权、阈值分流、资本计提）消费的是**概率本身**。校准好的模型才能直接接进"P > 0.85 自动放行、0.6–0.85 转人工"这类门控体系，阈值才有真实语义。

**为什么决策模型 ECE 低**：训练目标就是完整分布（蒸馏 teacher 的 post-reasoning 分布而非 argmax，"校准跟随 teacher 强度"）；不经过 token 采样/解码，没有生成漂移；KL 锚定与 coverage/competition 分解直接约束分布形状。JEVal 实测：SII-Decision-27B ECE 0.0149，约为最佳生成式 LLM（GLM-5.3-Flash thinking，0.0784）的**五分之一**；Jev 0.0532 也优于所有生成式配置（GPT-6-Luna 0.1159）。

**陷阱**：ECE 只度量 top-1 置信度——模型只要"对最有把握的那一项的把握报得准"就能拿到低 ECE，其余候选项上的概率质量如何分布它并不检查。在受控概率估计任务上，Jev"排序对、数值错"（top-1 选对了，但把 89% 的概率压给真实频率仅 35.6% 的选项，见 5.3），ECE 对此完全无感。经典后处理工具（温度缩放，[Guo et al. 2017](https://proceedings.mlr.press/v70/guo17a.html)，A）依然适用，Jeva 也内置了三个独立温度参数（聚合/竞争/覆盖）。

### 4.6 范式小结

| 维度 | 主流做法 |
|---|---|
| 底座 | 几乎全部基于 Qwen 系（3.5/3.8），少数 Gemma/ModernBERT |
| 数据 | 现有 NLP 分类/推理数据集转换为候选集格式为主；SII 完全自蒸馏无 gold |
| 监督 | 硬标签 + 软概率 + null + score 四类混合 |
| 训练 | SFT/蒸馏为主流；DPO 偶见（NeoHorse-Jev）；RLCD 仅 Jev 官方口径；**标准 RL（PPO/GRPO）在公开文献中基本空白** |
| 校准 | 训练期（分布蒸馏、KL 锚定、独立温度）为主，温度缩放等后处理为辅 |
| 去偏 | 候选打乱、置换一致性筛选 teacher、BA-LoRA 后训练 |

---

## 五、金融领域：应用场景与实测效果

### 5.1 JEVal：目前最系统的证据

[JEVal](https://arxiv.org/abs/2610.03935)（B）：中英双语，11,257 实例、36 数据集、58 子任务、10 领域（金融为其中之一）；三种任务类型 choice（85.1%）/noul（12.0%）/score（2.8%）；候选数 2–255（中位数 5）；25 个模型配置（11 个决策模型 + 9 个非思考 LLM + 5 个思考 LLM），开源模型统一 H100 + vLLM 本地部署。数据集在 [HF 公开](https://huggingface.co/datasets/carlosxiang/JEVal)。

**总分榜**（choice+noul 共 10,938 题）：

| 模型 | ACC% | ECE | Brier |
|---|---|---|---|
| GPT-6-Luna（生成式） | **80.93** | 0.1159 | 0.297 |
| GLM-5.3-Flash thinking | 80.37 | 0.0784 | 0.316 |
| **SII-Decision-27B（决策）** | 79.24 | **0.0149** | **0.279** |
| **Jev（决策）** | 78.38 | 0.0532 | 0.287 |
| Kev-27B | 77.17 | 0.0356 | 0.304 |
| Qwen3.8-27B 非思考 | 73.81 | 0.1697 | 0.446 |

前七名差距在 3 分以内，决策模型已挤进第一梯队——**但这是十个领域平均后的结果**。

### 5.2 金融域：专业知识是决策模型的软肋

论文原文（§4.2）：**"顺序在医疗、金融和法律三个领域反转，最佳决策模型比最佳生成式 LLM 低 3–6 个百分点"**，且"thinking 无法解释这个差距——GPT-6-Luna 不开思考就在这三个域排第一或第二"。决策模型内部，**Jev 在金融域领先**（Jev 领跑通用知识、医疗、金融、常识、事实性五域；SII-Decision-27B 领跑其余五域）。（金融子集的逐模型数字在论文 Figure 3/附录 D.4 中以图表呈现，正文未给表格数值——引用时注意这一限制。）

**解读**：金融判断依赖的专业知识存在于预训练语料和推理链中，而决策模型的单次前向 + 有限候选接口"压缩"掉了这部分能力。蒸馏只能收回标签选择类任务的 thinking 增益，专业知识密集域收不回来。这与 [research_scope.md](./research_scope.md) 的预判一致：Jev 的适配条件是"高频、答案空间有限、低延迟、允许置信度门控"，而**专业知识密度高的金融裁决不满足"答案空间有限且证据在输入中"这一前提**。

### 5.3 置信度失真：金融场景最危险的发现

JEVal 设计了 4 个受控任务（数值 2 个 + 概率 2 个，各 100 题）专门考察"概率本身准不准"（§5.1，B）：

- **先验概率任务**：Jev 100 题全部选中了正确的众数选项，**但把平均 89.35% 的概率质量给了它，而真实分布只有 35.60%**；众数概率误差 E_mode = 53.75 个百分点，为所有配置中最大（对比：thinking DeepSeek-V4-Flash 两项误差均为 0.00）。
- **后验概率任务**（贝叶斯更新）：Jev 准确率 67%，E_mode = 28.12pp；thinking LLM 达 82–100%。
- 论文结论："**Jev 排序正确，但把概率压向 top choice**……任何对概率做平均或阈值化的下游用法都会继承这个失真。"

**对金融的含义**（这是本综述认为最需要强调的一点）：金融决策的下游几乎全是"概率消费者"——期望值计算、风险加权、阈值分流、资本计提。一个"选得对但报得虚高"的模型，在 argmax 评测里看起来很好，接进风控阈值体系后会系统性地**高估确定性**：本该转人工复核的边缘案例被自信放行。低 ECE 不能给你安全感，因为它只测 top-1（见 §4.5 指标小讲座）；**必须补测全分布指标（Brier、逐选项可靠性图、受控概率任务）**。

### 5.4 金融相关基准上的横向数据

- **BANKING77（银行客服意图识别，K=16）**：Jeva-4B 91.62% > Jev-1.13.0 84.63% > KEV-4B 82.34% > InternDecision-4B 81.20%（B，[Jeva 论文 Table 1](https://arxiv.org/html/2610.18773)）。这是"证据在输入中"的金融文本分类任务的典型代表——决策模型表现很好。
- **Null 检测（BANKING77 含 588 个"无正确答案"请求）**：Jeva AUROC 98.66%，在 5% 误报阈值下能识别 93.88% 的"无答案"请求（B）。金融含义：**"拒答/转人工"能力可以被显式建模**，这正是置信度门控的基础设施。
- 注意：BANKING77 是意图识别，不是金融专业判断；**目前没有针对真实市场数据、金融机构流程的公开独立评测**——这是证据链上最大的缺口。

### 5.5 长程交互：错误累积吃掉速度红利

JEVal 把 Jev 接入 τ-bench 工具调用智能体（Retail/Airline 两环境；Jev 负责选工具/动作，同一底座 LLM 负责生成参数）（§Finding 7，B）：

| 系统 | 成功率 | P50 时延 | P95 时延 |
|---|---|---|---|
| 纯 LLM 基线 | **77.6%** | 267.1s | 695.5s |
| Jev + LLM | 66.1% | **160.6s** | 816.0s |

单步更快（中位数时间 -40%），但**系统成功率下降 11.5 个百分点，且 P95 尾部反而恶化**——机制是"单步错误复利成慢尾巴"：一次选错工具，轨迹偏离，触发反复查询、补救、重试，部分轨迹直接超时。

**对金融多步流程的含义**：信贷审批、投研工作流、合规检查都是长链条流程。若把决策模型放进每个环节，**单环节的准确率必须按流程长度打折来评估**（N 个环节各 95% ≈ 整体 77%@N=5）。评估必须覆盖错误传播路径，并配套人工复核节点，而不是报告单题准确率和延迟。

另外两个相关发现：长上下文性能衰减（Jev 在 LongBench 上从 4–8k 的 93.5% 掉到 ~20k 的 74.5%，且存在"lost-in-the-middle"，证据在中间时准确率 80.4%）——金融文档（招股书、年报）动辄数万 token，这是硬约束；社会模拟中决策模型的**总量估计误差偏大**（州级选举人票判对 47/51，但 vote-share 误差约为原管线两倍）——类比到金融组合层面的加总风险估计，同样要警惕"个体对、总量偏"。

### 5.6 厂商宣传的金融案例：全部应降权处理

中文技术社区流传一批具体数字诱人的"落地案例"（E 级证据，均为 CSDN/自媒体文章，**无第三方复核、无客户署名、细节无法验证**）：

- 某银行 372 条反欺诈硬编码规则迁移 Jev，"误判率下降 63%"（[CSDN](https://blog.csdn.net/weixin_32181267/article/details/169514969)）
- 某支付公司 3700 行 Java 状态机 + 82 条 Drools 规则压缩为 21 行决策调用（[CSDN](https://blog.csdn.net/weixin_29587015/details/169711627)）
- 某券商风控拆成 23 个决策单元，token -64%、延迟 820ms→97ms（[CSDN](https://blog.csdn.net/weixin_32238157/article/details/169114387)）
- 某银行审批系统日处理 200 万信贷申请、80ms 内三分类（[CSDN](https://blog.csdn.net/weixin_32487557/article/details/169712459)）

**未找到任何一家金融机构公开署名确认部署 Jev 式模型的案例。**这些文章可以作为"厂商叙事与场景想象"来读，但按本项目证据分级只能作线索（E），不能进入结论。

### 5.7 金融场景适配矩阵（综合判断）

| 场景 | 适配度 | 依据 |
|---|---|---|
| 工单/客服意图路由（含银行客服） | ✅ 高 | BANKING77 91%+，证据在输入中，高频低延迟 |
| 交易欺诈**初筛**（前置门控 + 置信度分流） | ⚠️ 中，需门控 | null-aware/低延迟契合；但专业判断弱、全分布校准存疑，必须配阈值转人工 |
| 合规文本分类、文档条款匹配 | ✅ 较高 | 长上下文/文档理解恰是决策模型最强域（JEVal 前两名均为决策模型） |
| 信贷审批**最终裁决** | ❌ 低 | 专业知识域落后 3–6pp；概率失真直接影响风险定价；监管要求可解释性而决策模型无自由文本解释 |
| 投资研究/多步投研工作流 | ❌ 低（现阶段） | 长程错误累积 -11.5pp；数值计算弱（Jev 精确除法仅 44%） |
| 风险分级/信用评分（有序档位） | ⚠️ 需先修偏差 | 序数压缩偏差实证存在；BA-LoRA 类后训练可部分修复 |
| 组合层面总量估计 | ⚠️ 警惕 | "个体对、总量偏"模式已在社会模拟中复现 |

---

## 六、能力边界与研究走向

### 已验证的结论（A/B 级证据）

1. **决策模型作为一个品类是成立的**：总分逼近顶级生成式 LLM（差距 <3pp），校准（top-1）显著更好，延迟低 1–2 个数量级，成本低 1 个数量级以上。
2. **"证据在输入中"的任务是它们的主场**：长文档理解、意图分类、代理动作选择——thinking 给 LLM 的增益在这些域几乎为零。
3. **专业知识域（金融/医疗/法律）系统性落后 3–6pp**，且非 thinking 可弥补——这是预训练知识与单次前向的结构性差距。
4. **"选得对"不等于"报得准"**：top-1 校准优秀与全分布概率失真并存；任何消费概率的下游（金融几乎全是）都继承失真。
5. **单步优势会在长程流程中反转**：τ-bench -11.5pp + P95 恶化。
6. **结构性质可被工程化**：置换不变（0% 翻转）、null 一等公民、候选缓存（27.86× 加速）在 Jeva 上均已实证，且这些性质对金融审计场景（可复现性、拒答、成本）价值直接。
7. **偏差是学出来的、可修的**：序数压缩经 BA-LoRA 后训练大幅缓解并正迁移。

### 值得评估的方向（尚缺证据）

1. **金融专用评测缺位**：JEVal 金融子集是通用基准的一部分，不是针对真实市场数据、机构流程、监管约束的验证。急需：财报事件分类、8-K 分类、信贷文本筛查等任务上的独立可复现实验（正是本项目开题的实证范围）。
2. **全分布校准与选择性预测**：覆盖率-风险曲线（coverage-risk）、拒答率-误判率权衡在金融门控中的定量设计。
3. **混合架构**：Lev 的按需循环、Thinking Jev（94.8%）提示"深度可调的 System One"可能是专业知识域的解法；决策模型选路 + LLM 复核的级联经济性。
4. **可审计性**：无自由文本解释是硬伤。决策路径日志、类型契约、置换不变性带来的可复现性，能否满足监管审计（vs. LLM 事后合理化式解释孰优）尚无研究。
5. **闭源与版本漂移风险**：Jev 是托管 API，`jev-latest` 别名意味着静默升级；金融合规要求模型版本固定——开权重方案（Jeva/SII/Kev/Clef）在治理上天然占优。
6. **RL 训练空白**：公开文献中标准 RL 用于决策模型的工作几乎为零（仅 TypeSafe RLCD 口径与 OpenJev-RLCD 社区复现），这是方法论上的开放地带。

---

## 七、结语：高屋建瓴地看

把镜头拉远，Jev 式模型的意义不在于"又一个更便宜的小模型"，而在于三件事：

**第一，它把"判断"确立为一种独立的模型原语。**过去三年 LLM 应用工程的隐性共识是"一切皆生成"；Jev 证明当终点是决策时，从决策出发设计接口（类型化输出、封闭答案空间、概率直出、null 一等公民）能同时赢下延迟、成本和校准三条曲线。OpenAI/Microsoft/Cloudflare 六周内全部跟进，说明这不是学术好奇心，而是接口层的收敛。

**第二，它暴露了"快思考"的知识天花板，验证了双系统分工的必要性。**金融/医疗/法律 3–6pp 的差距、概率质量的失真、长程流程的错误复利，共同指向同一个结论：System One 不能替代 System Two，只能替代"本该由 System One 干、却被迫用 System Two 干"的那部分工作。金融系统的正确形态大概率是级联：决策模型做高频初筛与分流（置信度门控），LLM 做低频专业裁决与解释，人工做最终复核。

**第三，对学习者而言，这是观察"一个模型品类从 0 到 1"的活教材。**从商业发布、资本定价（20 人公司 $75 亿估值）、大厂跟进、开源涌现、专用化分叉，到偏差审计和"泼冷水"研究出现，完整周期只有六周——所有环节都发生在 arXiv 和 GitHub 上，几乎每篇关键论文都开源了数据与代码（JEVal 数据集、Jeva、SII-Decision、awesome-jev-survey）。想入门，读三篇就够搭起框架：[JEVal](https://arxiv.org/abs/2610.03935)（评测与洞见）、[Jeva](https://arxiv.org/html/2610.18773)（架构第一性原理）、[LLM-as-Jev](https://arxiv.org/html/2610.02076)（决策能力与 LLM 的关系）。

**回到金融**：现有证据支持"值得认真评测"，不支持"可以独立托付"。把它当受测组件，用金融任务、真实使用条件、全分布校准指标和流程级错误传播去检验它——这正是本项目开题报告要做的事。

---

## 参考文献（按证据分级）

### A/B 级：学术论文与可复现评测
1. [General Decision Models: Benchmarking and Insights Beyond Jev (JEVal), arXiv 2610.03935](https://arxiv.org/abs/2610.03935) — 核心基准论文，10 领域 25 配置，金融域 3–6pp 差距、校准失真、τ-bench 长程结果出处；[数据集](https://huggingface.co/datasets/carlosxiang/JEVal)
2. [Jeva: System One Models from First Principles, arXiv 2610.18773](https://arxiv.org/html/2610.18773) — 决策原生架构四性质；[GitHub](https://github.com/LiQiiiii/Jeva)、[HF 模型](https://huggingface.co/LIQIIIII/Jeva-4B)、[项目页](https://liqiiiii.github.io/Jeva/)
3. [Jev-ToM: Evaluating Theory of Mind in Typed Decision Models, arXiv 2610.18610](https://arxiv.org/html/2610.18610v1) — ToM 能力评测，TypeSafe Jev 73.2%
4. [More Choices, Fewer Decisions: Ordinal-Scale Bias in JEV-like Direct-Decision Models, arXiv 2609.38827](https://arxiv.org/html/2609.38827v1) — 序数压缩偏差与 BA-LoRA 修复；[代码](https://github.com/Glax147/jev_ordinal_scale_bia)
5. [LLM-as-Jev, arXiv 2610.02076](https://arxiv.org/abs/2610.02076) — 免训练读出、树分解损失、Thinking Jev 94.8%
6. [Lev: 循环决策模型, arXiv 2610.20951](https://arxiv.org/abs/2610.20951) — [项目页](https://openlair.github.io/lev/)
7. [Guo et al., On Calibration of Modern Neural Networks, ICML 2017](https://proceedings.mlr.press/v70/guo17a.html) — ECE/温度缩放；[实现](https://github.com/gpleiss/temperature_scaling)
8. [Auditing Open Jev-Style Models for Retrieval, arXiv 2610.14631](https://arxiv.org/html/2610.14631v1)；[Entity Matching using Decision Models, arXiv 2610.34626](https://arxiv.org/html/2610.34626v1)；[Measure Before You Train, arXiv 2610.32955](https://arxiv.org/html/2610.32955v1)；[SecJev, arXiv 2610.03073](https://arxiv.org/abs/2610.03073)；[Laya 复现研究, arXiv 2609.33843](https://arxiv.org/abs/2609.33843)
9. [awesome-jev-survey（证据综述仓库）](https://github.com/Eurekaleo/awesome-jev-survey)

### C 级：官方文档与厂商资料
10. [jevmodel.org — Jev 官方站](https://jevmodel.org/) — 定价 $0.042/M、三类输出、API 公开时间
11. [OpenAI Decisions API 文档](https://developers.openai.com/api/docs/guides/decisions)；[LiteLLM 集成](https://docs.litellm.ai/docs/decisions)
12. [Microsoft Decision-1 公告](https://commandline.microsoft.com/microsoft-decision-1-model-foundry/)；[Azure Catalog](https://ai.azure.com/catalog/models/Microsoft-Decision-1)
13. [decisionmodels.io](https://decisionmodels.io/) — Jev 兼容 API 市场、Decision Index 排名
14. [Liquid AI d1（Ollama）](https://ollama.com/library/d1)；[HF Decision Index（Artificial Analysis）](https://artificialanalysis.ai/articles/artificial-analysis-decision-index)
15. [SII 评测公告](https://www.sii.edu.cn/2026/1013/c27a1249/page.htm)；[HF: sii-research](https://huggingface.co/sii-research)
16. 开源仓库：[jaredpalmer/kev](https://github.com/jaredpalmer/kev)、[NandhaKishorM/laya](https://github.com/NandhaKishorM/laya)、[bespokelabsai/nimble](https://github.com/bespokelabsai/nimble)、[TianyuCodings/NanoJev](https://github.com/TianyuCodings/NanoJev)、[daseinlabs/open-jev](https://github.com/daseinlabs/open-jev)、[Zefan-Cai/Open-Jev](https://github.com/Zefan-Cai/Open-Jev)、[typesafe-ai/system-one-adapter-python](https://github.com/typesafe-ai/system-one-adapter-python)、[PAI-CUHK/MEDJEV](https://github.com/PAI-CUHK/MEDJEV)

### D/E 级：媒体报道、社区文章与厂商叙事（仅作线索）
17. [知乎：Jev 从语言生成到概率决策](https://zhuanlan.zhihu.com/p/2093523544483407367)；[知乎：不输出 token 的 Transformer 深度解读](https://zhuanlan.zhihu.com/p/2099285835933814959)
18. [51CTO：Jev 当 AI 不再"写答案"](https://www.51cto.com/article/859516.html)；[InfoQ：Cloudflare Clef](https://www.infoq.cn/article/iqCrZLLizNkRTpfo2Jrb)；[百家号：今年最贵 A 轮](https://baijiahao.baidu.com/s?id=1879532892229523279)
19. CSDN 金融"落地案例"系列（**未经独立验证，勿作结论依据**）：[反欺诈规则迁移](https://blog.csdn.net/weixin_32181267/article/details/169514969)、[支付路由](https://blog.csdn.net/weixin_29587015/details/169711627)、[券商风控 DU 架构](https://blog.csdn.net/weixin_32238157/article/details/169114387)、[银行审批](https://blog.csdn.net/weixin_32487557/article/details/169712459)
20. [搜狐：OpenAI 向所有用户开放 Jev 竞品](https://www.sohu.com/a/1091715056_114984)；[阿里云开发者社区：Jev 决策模型拆解](https://developer.aliyun.com/article/1775160)

## 方法论说明

检索约 10 轮（WebSearch + 网页全文抓取），深读 3 篇核心论文全文（JEVal、Jeva、序数偏差），3 个并行研究代理覆盖生态/训练/金融三条线；累计分析约 50 个来源。已调查的子问题：① Jev 式模型方法与架构谱系；② 开源模型生态与商业格局；③ 训练方法与数据；④ 金融域评测结果与校准风险；⑤ 长程交互与能力边界。**已知缺口**：JEVal 金融子集的逐模型数字仅存于论文图 3/附录 D.4（正文未给表格）；JevK5/LightJev/JevForge 等个别社区模型名无法核实；金融机构真实部署案例为零公开记录。
