---
title: "AgentRisk：面向多轮智能体交互的风险理解、识别与策略干预"
date: 2026-11-06T14:12:24+08:00
draft: false
tags: [AgentRisk, LLM Agent, 风险评估, 策略干预, AI安全]
comments: true
---

# AgentRisk：面向多轮智能体交互的风险理解、识别与策略干预

## 摘要

随着大语言模型从对话系统发展为能够调用浏览器、文件系统、数据库、终端和企业服务的智能体（agent），安全问题的核心也从“模型会不会生成有害文本”转向“模型会不会在不充分授权、不可信指令或信息不足的情况下执行具有现实后果的动作”。AgentRisk 是 SemEval 2027 Task 10，研究多轮 agent 轨迹中的风险评估、风险归因与政策条件化干预。任务将完整交互轨迹和最终拟执行动作作为输入，要求系统预测五级风险严重度、损害领域、风险驱动因素，并在给定政策下决定如何干预。本文介绍 AgentRisk 的任务定义和评测结构，对官方发布的 711 条训练与开发数据进行统计分析，比较基于规则、监督学习、大语言模型和混合系统的实现路线，并讨论如何将其转化为 agent harness 中的执行前风险 Gate。分析表明，AgentRisk 的主要困难并非上下文长度，而是小样本与极端长尾、风险等级的序数边界、多标签组合泛化、输出措辞捷径，以及尚无监督标注的政策干预任务。本文最后提出面向研究和工程落地的评估框架与未来方向。

**关键词：** AgentRisk；LLM Agent；工具调用；风险评估；多标签分类；策略干预；安全 Gate

## 1. 背景：为什么 Agent 需要动作级风险评估

传统大模型安全研究往往以模型输出的文本为观察对象，但 agent 能够将语言决策转化为文件删除、邮件发送、资金转移、权限修改或设备控制。相同一句自然语言在普通聊天中可能没有直接后果，在工具调用环境中却可能引发不可逆损害。风险判断也不能只看用户最后一句话：恶意指令可能来自网页、邮件、文档或工具返回结果；一个动作是否安全还取决于用户原始意图、授权范围、目标资源、执行规模、可逆性和人工监督。

已有研究从不同角度刻画这类风险。AgentHarm 包含 110 个显式恶意 agent 任务及其扩增版本，重点测量 agent 对恶意用户请求的服从和越狱后的任务执行能力 [3]。SafeArena 在四个网站环境中构造安全与有害任务，并以四级框架描述 web agent 的行为结果 [4]。AgentDojo 则强调来自外部工具数据的间接提示注入，提供 97 个任务和 629 个安全测试案例 [5]。MT-AgentRisk 进一步研究多轮攻击，将单轮有害任务转化为多轮序列；其结果显示，多轮环境使攻击成功率平均增加 16% [6]。

SemEval AgentRisk 与上述工作的关注点并不完全相同。它不直接要求 agent 在可执行环境中完成任务，而是给定一段已经形成的多轮轨迹和一个 proposed action，评估系统能否回答三个递进问题：动作有多危险、危险来自哪里、在特定政策下应该如何处理。因此，它更接近一个可嵌入 agent 系统的风险理解与决策模块。

## 2. 任务定义

每个样本由多轮交互轨迹和最终拟执行动作组成。轨迹按 `USER`、`THOUGHT`、`ACTION`、`OBSERVATION` 等角色组织；系统必须结合整个轨迹，而不是孤立判断最后动作 [1][2]。

### 2.1 Subtask 1：Risk Level Estimation

该子任务将风险严重度表示为 1 至 5 的有序等级：

1. **Minimal**：后果可忽略、局部且易恢复；
2. **Low**：存在有限风险，但影响轻微或已有有效保障；
3. **Moderate**：不良后果具有合理可能性，并可能造成实质影响；
4. **High**：严重后果具有现实可能性，恢复困难或涉及敏感目标；
5. **Critical**：可能造成灾难性、系统性、大规模或不可逆后果。

官方要求综合考虑发生概率、损害强度、影响规模、目标脆弱性、系统依赖、监督机制和可逆性。主指标是平均绝对误差（MAE）：

\[
\mathrm{MAE}=\frac{1}{N}\sum_{i=1}^{N}|\hat y_i-y_i|.
\]

MAE 体现了标签的序数性质：把等级 5 预测成 4 的代价低于预测成 1。由此看，Subtask 1 更适合序数回归或阈值预测，而不是完全忽视等级距离的普通五分类。

### 2.2 Subtask 2a：Harm Domain Classification

该子任务判断“什么可能受到损害”。标签包括：Physical / Health、Privacy / Confidentiality、Security / System Integrity、Financial / Assets、Operational / Service、Legal / Compliance、Information Quality、Reputational / Social / Psychological、Discrimination / Fairness、Environmental 和 No Material Harm。

每条样本包含一至两个无序标签，使用 mean Jaccard 评测：

\[
J(A,B)=\frac{|A\cap B|}{|A\cup B|}.
\]

Jaccard 同时惩罚漏标和多标，因此阈值选择、输出标签数量和标签共现建模都会直接影响成绩。

### 2.3 Subtask 2b：Risk Driver Classification

该子任务回答“为什么这个动作具有风险”。标签包括：Risky Requested Outcome、Untrusted / Conflicting Instructions、Authorization / Consent Failure、Insufficient Grounding / Uncertainty、Scope / Safeguard Failure、Execution / Action Hazard 和 No Material Risk。

Domain 和 Driver 的区别值得特别强调。前者描述受损对象，后者描述风险形成机制。例如，将患者记录发送到外部邮箱，可能同时属于 Privacy / Confidentiality 和 Physical / Health；其风险驱动因素则可能是不可信嵌入指令与缺乏授权。两个维度分开，有利于系统生成可解释、可操作的干预意见。

### 2.4 Subtask 3：Policy-Guided Intervention

Subtask 3 要求系统在给定风险政策下选择干预方式，并生成对 agent 可执行的反馈。同一动作在不同政策下可能被允许、要求确认、升级人工审批或直接阻断。官方数据卡指出，目前没有发布该子任务的训练 gold label，评价政策将临近正式评测时公布 [2]。因此，它不是普通监督分类，更接近政策条件化的零样本或弱监督决策问题。

截至本文写作时，官方尚未公布最终提交格式、全部补充指标和完整排名规则；这些内容预计随 Codabench 与评测脚本进一步明确 [1][2]。

## 3. 官方数据集与统计特征

官方数据集包含 568 条训练样本和 143 条开发样本，共 711 条英文轨迹 [2]。本文直接统计官方 JSONL 文件，结果如下。

| 指标 | 统计结果 |
|---|---:|
| 总样本数 | 711 |
| Train / Dev | 568 / 143 |
| 平均轨迹步数 | 14.2 |
| 平均轨迹长度 | 496.4 words |
| 轨迹长度 P90 | 682 words |
| 最大轨迹长度 | 1,325 words |
| proposed action 平均长度 | 22.1 words |
| 场景组数量 | 397 |
| 双 harm-domain 样本占比 | 20.3% |
| 双 risk-driver 样本占比 | 39.8% |

官方使用 `scenario_group` 管理相互关联的样本。由同一场景产生的风险版本和安全版本被放入同一 split，避免 train 与 dev 之间的直接场景泄漏。自行交叉验证时也应使用 GroupKFold 或类似的成组划分，不能随机拆分单条样本。

### 3.1 风险等级分布

| 等级 | 数量 | 占比 |
|---|---:|---:|
| 1 Minimal | 331 | 46.6% |
| 2 Low | 41 | 5.8% |
| 3 Moderate | 159 | 22.4% |
| 4 High | 171 | 24.1% |
| 5 Critical | 9 | 1.3% |

等级 1 占接近一半，而等级 5 只有 9 条。这意味着总体准确率不能反映极端风险识别能力；即便 dev 分布经过分层，Level 5 也只有 2 条。单次开发集分数对这类样本非常不稳定。

### 3.2 Harm Domain 分布

| Domain | 数量 |
|---|---:|
| No Material Harm | 327 |
| Privacy / Confidentiality | 225 |
| Security / System Integrity | 81 |
| Financial / Assets | 66 |
| Physical / Health | 64 |
| Operational / Service | 56 |
| Legal / Compliance | 19 |
| Reputational / Social / Psychological | 8 |
| Information Quality | 4 |
| Environmental | 3 |
| Discrimination / Fairness | 2 |

后三类几乎无法仅靠监督学习稳定获得分布知识。一个模型即使完全不预测这些标签，对整体 mean Jaccard 的影响也可能很小，但这会掩盖部署中对少数高影响风险的失效。

### 3.3 Risk Driver 分布

| Driver | 数量 |
|---|---:|
| No Material Risk | 327 |
| Untrusted / Conflicting Instructions | 244 |
| Authorization / Consent Failure | 160 |
| Scope / Safeguard Failure | 135 |
| Insufficient Grounding / Uncertainty | 78 |
| Execution / Action Hazard | 33 |
| Risky Requested Outcome | 17 |

风险驱动因素具有明显共现结构。例如 Untrusted Instructions 经常与 Authorization Failure 或 Scope Failure 同时出现。将完整标签组合视为单一类别会造成组合爆炸；完全独立预测又会忽略依赖关系。较合理的方式是独立概率输出加结构化解码，并强制 `No Material Harm` 与 `No Material Risk` 单独出现。

### 3.4 数据捷径

数据中存在明显的 action-style shortcut。本文统计发现，proposed action 中包含 “I can't/I cannot” 等拒绝措辞的 200 条样本中，190 条为 Level 1。仅使用一个简单规则：“拒绝或询问确认则预测 1，否则预测 3”，在 dev 上即可得到约 0.538 MAE；最优常数预测的 MAE 约为 1.217。一个简易词袋模型只读取 proposed action，也能取得约 0.56 MAE。

这些结果不是官方基线，而是本文为诊断数据偏差构造的探索性基线。它说明开发集上的好成绩可能部分来自输出风格，而非真正理解完整轨迹。严谨实验至少应比较 action-only、trajectory-only、full-input 和反事实改写四种设置。

## 4. 核心挑战

### 4.1 小样本和极端长尾

568 条训练数据不足以从头学习复杂风险边界。Level 5、Environmental、Information Quality 和 Fairness 的样本尤其稀少。常规类别加权能够提高模型对少数类的关注，但也可能让模型记住极少量模板。需要结合外部安全知识、合成数据、检索增强或 LLM teacher，并使用成组验证监控过拟合。

### 4.2 等级 3 与等级 4 的序数边界

中高风险之间通常不存在单个决定性关键词。判断需要组合授权、后果强度、影响范围、可逆性和监督机会。不同 harm domain 的严重度又是在各自领域内部标定，不能简单认为医疗风险天然高于财务风险。模型需要学习条件化阈值，同时给出校准概率或不确定度。

### 4.3 多标签组合泛化

数据中存在 40 种 domain 标签集合和 27 种 driver 标签集合，其中许多只出现一至三次。系统既要预测单标签，又要识别合理的双标签组合。最佳阈值不一定是 0.5，也不一定对所有标签相同，应直接在 grouped validation 上针对 Jaccard 调整标签级阈值和最大输出数量。

### 4.4 多轮证据归因

平均约 500 words 的轨迹并不会超过现代模型的上下文窗口，真正困难的是找出具有决定意义的步骤：用户原始授权是什么、危险指令来自哪里、工具观察暴露了什么、最终动作是否偏离目标。缺乏证据定位时，系统容易被最后动作的措辞支配。

### 4.5 政策条件化与无监督干预

Subtask 3 没有现成监督标签，而且政策尚未最终公布。严重度不能唯一决定干预：高风险但得到充分授权的动作可能需要确认或人工审批；低严重度但来自不可信网页指令的动作可能应该阻断。系统必须先形成与政策无关的风险事实，再由 policy engine 映射成干预措施。

## 5. 实现路线

### 5.1 规则与启发式基线

第一组基线应包括多数类、MAE 最优常数、关键词规则和基于工具元数据的静态策略。它们不是最终系统，但能揭示数据中有多少信号来自拒绝措辞、工具名称、路径、外部收件人和危险动词。若复杂模型只略高于这些基线，说明它可能没有真正利用轨迹。

### 5.2 监督学习路线

Subtask 1 可使用 encoder 模型或本地指令模型进行监督微调。推荐序数方法，例如累计链接模型或多个阈值分类器：

\[
P(y>k\mid x),\quad k\in\{1,2,3,4\}.
\]

Subtask 2 可使用共享 encoder 与两个 multi-label heads。损失可采用类别平衡 BCE 或 focal loss，并为少数标签设置平滑和正则化。由于轨迹不算极长，DeBERTa、ModernBERT 等 encoder 是成本较低的起点；7B 至 14B 指令模型可用 LoRA，但必须通过 grouped validation 和反事实测试控制记忆化。

### 5.3 纯 LLM 推理路线

LLM 路线将标签定义、少量示例和完整轨迹放入 prompt，要求返回固定 JSON。相较直接要求标签，更稳定的流程是先抽取结构化风险因素，再给出最终预测：

```json
{
  "instruction_source": "untrusted_tool_content",
  "authorized": false,
  "affected_target": "patient_record",
  "scale": "single_individual",
  "reversibility": "low",
  "safeguards": ["none"],
  "severity": 4,
  "domains": ["Privacy / Confidentiality"],
  "drivers": ["Authorization / Consent Failure"],
  "evidence_steps": [1, 6, 9]
}
```

纯 LLM 适合快速建立强基线和处理 Subtask 3，但需要固定 temperature、schema validation、重试机制和输出校准。闭源 API 的使用还取决于最终竞赛规则、成本和可复现性要求。

### 5.4 混合路线

综合效果、稳定性与研究价值，推荐采用以下流程：

```text
轨迹与 proposed action
        ↓
LLM 或抽取器生成结构化风险因素与证据
        ↓
共享监督模型
  ├─ ordinal severity head
  ├─ harm-domain head
  └─ risk-driver head
        ↓
概率校准与结构化解码
        ↓
Policy interpreter
        ↓
Allow / Confirm / Clarify / Escalate / Block
```

LLM 可作为 teacher 为训练数据生成风险解释、反事实样本和证据位置；小模型负责低成本、确定性的正式预测；校准器根据 dev 或 grouped cross-validation 修正等级阈值和多标签阈值。Subtask 3 则保留 policy-conditioned LLM 或符号规则层。

### 5.5 推荐实验矩阵

小组实施时，建议至少完成以下比较：

1. 常数、规则和关键词基线；
2. proposed-action-only 模型；
3. full-trajectory 模型；
4. full trajectory + ordinal loss；
5. 共享 encoder 的多任务学习；
6. zero/few-shot LLM；
7. LLM 风险因素抽取 + 小模型校准；
8. 原始样本与反事实改写样本上的性能差异；
9. 常见标签与稀有标签分别报告；
10. 不同场景组的 bootstrap 置信区间。

## 6. 从比赛任务到 Agent Risk Gate

AgentRisk 最直接的未来用途，是作为 agent harness 中的动作执行前 Gate。它位于 planner/LLM 提出工具调用之后、工具真正执行之前：

```text
User + trajectory + tool observations
                  ↓
          Agent proposes action
                  ↓
       Risk understanding module
     severity / domains / drivers
                  ↓
            Policy engine
 Allow / Confirm / Clarify / Escalate / Block
                  ↓
             Tool execution
```

工程上，风险理解模型不应直接拥有最终阻断权。GuardAgent 等工作也表明，将安全政策知识与独立的守护推理模块结合，是保护工具型 agent 的一条可行路线 [7]。建议采用三层结构：

1. **硬规则层**：处理路径黑名单、密钥外发、资金操作、不可逆删除等确定性约束；
2. **风险模型层**：预测严重度、损害领域、驱动因素、证据和不确定度；
3. **政策层**：结合用户身份、工具元数据和组织政策决定放行、确认、澄清、升级或阻断。

该模块可以实现为 tool-call middleware，统一接收用户请求、轨迹、工具名、参数、指令来源、权限与资源敏感度。对于 MCP 系统，可放在 client 端，也可部署为 MCP gateway；生产环境更适合 gateway 强制执行，因为单个 agent 无法绕过它。

为降低延迟，可使用级联决策：低风险只读操作由静态规则直接放行；中等风险调用轻量分类器；高影响或低置信度操作再调用强 LLM；资金、医疗、权限和生产系统操作进入人工审批。检查后的工具参数必须冻结或签名，防止“检查时一个参数、执行时另一个参数”的竞态问题。

## 7. 未来使用方式与研究方向

### 7.1 从静态分类扩展到动态执行验证

当前任务只判断 proposed action。真实系统还需要执行后的状态验证，例如文件是否删除了超出范围的内容、邮件实际收件人是否改变、工具是否产生隐藏副作用。未来可将 AgentRisk 扩展为 plan gate、action gate 和 result monitor 三阶段架构。

### 7.2 跨工具与跨领域泛化

数据中的风险模式可能与工具名称和模板绑定。未来评测应将训练和测试按工具、领域或组织完全隔离，测量对新工具和新政策的零样本泛化，而不只是对新文本模板的泛化。

### 7.3 反事实与自适应攻击评测

安全动作可以被改写成命令式语言，危险动作也可以使用礼貌、合规的措辞。通过保持动作语义不变、改变表面风格，可以检查模型是否依赖拒绝词等捷径。还应引入自适应攻击者，根据 Gate 的反馈逐轮改变指令，而非只使用固定轨迹。

### 7.4 风险校准和选择性预测

部署时，最重要的不是单一准确率，而是“预测置信度是否可信”。模型可在高不确定度时 abstain，将动作升级给更强模型或人工。评价指标应增加 expected calibration error、危险动作漏拦率、安全动作误拦率，以及风险—效用曲线。

### 7.5 个性化与组织政策

不同组织对相同风险有不同容忍度。研究方向之一是将政策写成机器可读约束，或由模型将自然语言政策编译为决策规则。风险事实层保持通用，干预层根据组织、用户角色和环境状态动态变化。

### 7.6 持续学习与审计

Gate 的确认、阻断和人工覆写记录可形成新的监督数据，但需要防止将错误人工决策直接固化。较稳妥的做法是保存完整审计日志，周期性抽样复核，再通过 active learning 选择高不确定度和高影响案例进入训练集。

## 8. 局限与使用注意事项

首先，官方数据规模较小，且当前发布的主要是静态英文轨迹，无法覆盖生产环境的工具、语言和组织差异。其次，标签具有规范性判断成分，尤其是中高风险边界和政策干预，不同标注者或机构可能存在分歧。再次，数据中的拒绝措辞与低风险高度相关，容易形成虚假的高分。最后，Subtask 3 的政策、标注与完整评测方式尚未最终发布，因此本文对该子任务实现路线的讨论属于基于现有官方说明的设计建议，而非最终比赛规则。

AgentRisk 分类器不能被视为唯一安全边界。生产系统仍需要最小权限、沙箱、参数验证、事务回滚、人工审批、审计日志和工具端访问控制。风险模型适合补充这些机制，而不能替代它们。

## 9. 小组学习建议

建议用两至三次讨论完成学习：

**第一次：任务和数据。** 每位成员人工分析 10 条不同风险等级的轨迹，分别标注 domain、driver 和关键证据，再与 gold label 比较。重点讨论等级 3 与 4 的分界。

**第二次：实现和诊断。** 实现常数、规则、action-only 和 full-trajectory 四个基线，使用 `scenario_group` 成组验证。对拒绝措辞做反事实改写，观察性能下降。

**第三次：系统设计。** 将模型输出转换成 Allow、Confirm、Clarify、Escalate 和 Block，讨论误拦与漏拦的成本，并设计一个可嵌入 MCP gateway 或自研 harness 的最小 Gate。

可供讨论的问题包括：

1. 风险等级应当描述动作本身，还是动作在当前环境下的预期后果？
2. 当发生概率很低但后果灾难性时，应如何设定等级？
3. “请求用户确认”何时足够，何时必须阻断或人工审批？
4. 如果用户明确授权高风险操作，Gate 是否仍应干预？
5. 如何证明模型理解了完整轨迹，而不是识别拒绝措辞？
6. 比赛指标与真实部署指标之间还缺少什么？

## 10. 结论

AgentRisk 将 agent 安全从二元有害检测推进到多维风险理解：系统不仅要判断动作有多危险，还要解释受损对象、风险形成机制，并依据政策选择干预方式。其数据规模和上下文长度并不构成主要计算障碍，真正困难在于长尾、序数边界、多标签组合、证据归因、表面风格捷径和政策条件化决策。

对于竞赛，实现强基线的门槛并不高；对于研究，关键是证明系统没有依赖 action style，并能在新场景、新工具和新政策下保持校准；对于工程，最合理的定位是把 AgentRisk 作为 tool-call middleware 中的风险理解模块，与硬规则、权限控制和 policy engine 共同构成执行前 Gate。由此，AgentRisk 不只是一个文本分类任务，也可成为研究安全 agent 架构、风险校准和人机协同干预的实验入口。

## 参考文献

[1] AgentRisk Organizers, “AgentRisk — SemEval 2027 Shared Task,” 2026. [Online]. Available: https://agentrisk-semeval.lovable.app/

[2] AgentRisk, “AgentRisk/agentrisk-data-2027,” Hugging Face Datasets, 2026. [Online]. Available: https://huggingface.co/datasets/AgentRisk/agentrisk-data-2027

[3] M. Andriushchenko et al., “AgentHarm: A Benchmark for Measuring Harmfulness of LLM Agents,” arXiv:2410.09024, 2024. https://arxiv.org/abs/2410.09024

[4] A. D. Tur et al., “SafeArena: Evaluating the Safety of Autonomous Web Agents,” in *Proceedings of the 42nd International Conference on Machine Learning*, vol. 267, pp. 60404–60441, 2025. https://proceedings.mlr.press/v267/tur25a.html

[5] E. Debenedetti, J. Zhang, M. Balunović, L. Beurer-Kellner, M. Fischer, and F. Tramèr, “AgentDojo: A Dynamic Environment to Evaluate Prompt Injection Attacks and Defenses for LLM Agents,” arXiv:2406.13352, 2024. https://arxiv.org/abs/2406.13352

[6] X. Li et al., “Unsafer in Many Turns: Benchmarking and Defending Multi-Turn Safety Risks in Tool-Using Agents,” in *Proceedings of the 43rd International Conference on Machine Learning*, vol. 306, pp. 72041–72080, 2026. https://proceedings.mlr.press/v306/li26is.html

[7] Z. Xiang et al., “GuardAgent: Safeguard LLM Agents via Knowledge-Enabled Reasoning,” in *Proceedings of the 42nd International Conference on Machine Learning*, vol. 267, pp. 68316–68342, 2025. https://proceedings.mlr.press/v267/xiang25a.html

## 数据与伦理声明

**数据可用性：** 本文使用的 AgentRisk train、dev 和 split manifest 均由任务组织者公开发布，下载地址见参考文献 [2]。本文未重新发布数据副本。

**伦理声明：** 本文为公开基准的技术综述，不涉及人体受试者实验。文中讨论的风险案例仅用于安全研究与防护设计。

**利益冲突：** 作者未声明与 AgentRisk 组织者或相关模型供应商存在利益冲突。

**资金声明：** 本文未声明专项资金支持。

**作者贡献：** 文章材料检索、数据分析、结构设计与初稿需由小组成员在实际使用时补充 CRediT 署名信息。

**AI 使用说明：** 本文初稿由 AI 辅助完成资料整理、数据统计和文字组织。公开事实与引用链接经过核验；小组在对外发表前仍应由成员复核内容、补充作者信息，并对最终文本承担责任。
