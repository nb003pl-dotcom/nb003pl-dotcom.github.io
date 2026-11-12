---
title: "EPSBench 学习报告"
date: 2026-11-12T15:17:03+08:00
draft: false
tags: [EPSBench, LLM Agent, 盈利预测, 金融基准, 不确定性校准]
comments: true
---

# EPSBench 学习报告

> 论文：*EPSBench: Benchmarking LLM Agents for Earnings Forecasting*\
> arXiv：2610.38293\
> 本报告按 PPT 顺序整理。每节包含原 slide 图片与补充注释，可直接复制到支持 Markdown 的笔记工具中。

## 一、核心结论

EPSBench 将盈利预测拆成“证据检索、driver 估计、会计计算和不确定性报告”四个可审计环节。实验显示，Opus 4.6、GPT-5.5 和 GPT-5.4 的点预测准确度接近同期分析师共识，但所有模型给出的 80% prediction interval 都存在明显欠覆盖。模型的主要误差来自 operating income 估计，尤其在盈利大幅变化或出现复杂交易的季度。

---

## Slide 1｜EPSBench：面向盈利预测的 LLM Agent 基准

![Slide 1](/learning_report_slides/slide-01.png)

### 注释

- 研究问题：LLM Agent 能否把公开金融信息转化成可解释、可复核的 EPS 预测？
- 核心发现：领先模型的点预测接近 analyst consensus，但所有模型都低估预测不确定性。
- EPSBench 不只比较最终 EPS，还保存 drivers、evidence、calculator 输出和预测区间。

---

## Slide 2｜盈利预测考验“证据到数字”的完整链条

![Slide 2](/learning_report_slides/slide-02.png)

### 注释

- 盈利预测并非简单判断 EPS 上升或下降，而是要把不完整、可能冲突的信息映射到利润表项目。
- 收入增长不一定带来利润增长，因为成本、费用、税项和股数可能同时变化。
- 只要求最终数字的 benchmark 无法比较中间假设，也无法判断错误发生在哪个会计环节。
- EPSBench 要求模型预测 company-specific drivers，再由统一 calculator 计算 GAAP diluted EPS。

---

## Slide 3｜EPSBench 将盈利预测变成可审计的结构化任务

![Slide 3](/learning_report_slides/slide-03.png)

### 注释

- 数据集包含 86 家美国上市公司、8 个行业和 149 个 firm-quarter。
- 每个任务的信息截止日设在财报发布日前一天，避免模型使用正式公布的目标结果。
- 可用信息包括 10-K/10-Q、上一季度 earnings-call transcript 和截止日前新闻。
- 每个配置重复运行 3 次；输出包括 driver 数值及 evidence、calculator EPS 和 p10–p90 区间。

---

## Slide 4｜EPSBench workflow

![Slide 4](/learning_report_slides/slide-04.png)

### 注释

完整流程分为三段：

1. 从历史财报构建 company-specific accounting schema。
2. Agent 检索截止日前信息，逐项估计 drivers 并提交证据。
3. 统一 calculator 计算 EPS，再与实际值和同期分析师共识比较。

Schema 由三部分组成：Q-4 baseline、允许模型预测的 drivers、固定 accounting DAG。最终 EPS 可以逐层回溯到具体输入和证据。

---

## Slide 5｜最终预测 Prompt 分离证据、计算与输出

![Slide 5](/learning_report_slides/slide-05.png)

### 注释

- Prompt 先提供 Q-4 baseline 和唯一允许预测的字段。
- Agent 必须检索 filings、earnings-call transcript 和截止日前新闻。
- 每个 driver 都要提供一行具体 evidence，并明确 chosen value。
- Agent 不能直接预测 total revenue 或 EPS，不能手算，也不能从 analyst consensus 反推 drivers。
- `financial_calculate` 负责生成完整利润表和点预测 EPS；模型随后输出严格 JSON 及 p10–p90 区间。
- Point forecast 必须保留 calculator 的原始结果，不能为了让区间居中而修改。

---

## Slide 6｜Company-specific accounting schema

![Slide 6](/learning_report_slides/slide-06.png)

### 注释

- 不同行业和公司采用不同利润表结构，因此不能用一套通用模板覆盖所有任务。
- Baseline 保存去年同期的利润表数值；drivers 描述收入增长、成本率、费用、税项和股数变化。
- Accounting DAG 用固定会计恒等式逐层计算 diluted EPS。
- 历史回放验证中，146/149 个 schema 能在 1 美分内复现报告 EPS，其余 3 个在 4 美分以内。
- 固定计算过程后，误差可以分解为 operating、non-operating、tax 和 shares 等层级。

---

## Slide 7｜MSFT 2026Q3 完整预测案例

![Slide 7](/learning_report_slides/slide-07.png)

### 注释

这是 GPT-5.5、OpenCode、default reasoning、full information、seed 1 的真实预测记录。

#### 模型预测的 9 个 drivers

| Driver | 预测值 | Q-4 baseline |
|---|---:|---:|
| Product revenue YoY | −7% | $15,319M |
| Service & other revenue YoY | +22% | $54,747M |
| COGS margin change | +165 bps | 31.28% |
| R&D YoY | +14% | $8,198M |
| Sales & marketing YoY | +7% | $6,212M |
| G&A YoY | +8% | $1,737M |
| Other expense, net | $700M | — |
| Income taxes | $7,100M | — |
| Diluted shares | 7,456M | 7,461M |

#### Accounting schema 的计算过程

```text
Product revenue = 15,319 × 0.93 = 14,246.67
Service revenue = 54,747 × 1.22 = 66,791.34
Total revenue   = 81,038.01
Total COGS      = 26,688.54
Gross profit    = 54,349.47
Total OpEx      = 17,868.52
Operating income = 36,480.95
Pretax income    = 35,780.95
Net income       = 28,680.95
Diluted EPS      = 28,680.95 / 7,456 = 3.8467
```

#### 预测表现

- 点预测：$3.85
- P10–P90：$3.55–$4.25
- 实际 EPS：$4.27
- 误差：−$0.42，约 −9.9%
- 实际值略高于 P90，说明点预测和区间上界都偏低。
- 预测 operating income 为 $36.48B，实际为 $38.40B；pretax income 低估约 $3.56B。Schema 因而将 EPS 误差定位到偏保守的利润端假设。

---

## Slide 8｜实验设置

![Slide 8](/learning_report_slides/slide-08.png)

### 注释

- 主实验比较 Opus 4.6、GPT-5.5、GPT-5.4、GPT-5-mini、Qwen 3.5 和 GLM 4.5 Air。
- 所有模型使用相同的 149 个任务、accounting schemas、OpenCode harness、默认 reasoning effort 和完整信息环境。
- 每个模型配置运行 447 次。
- Shell、file 和开放 web 被关闭，仅保留 6 个受控 MCP 工具。
- 扩展实验分别考察信息源移除、agent harness 和 reasoning effort 的影响。

---

## Slide 9｜模型预测 leakage 风险

![Slide 9](/learning_report_slides/slide-09.png)

### 注释

- Opus 4.6、GPT-5.5、GPT-5.4 和 GPT-5-mini 披露的 knowledge cutoff 均早于 benchmark 的全部目标公告。
- Qwen 3.5 在评测窗口内发布且 cutoff 未披露，因此风险最高。
- GLM 4.5 Air 的 cutoff 同样未披露，但发布时间早于正式预测窗口。
- Memorization probe 使用同一批 86 个 ticker 的 247 个历史 firm-quarter，公告范围为 2024-06 至 2026-06。
- 正式 forecast benchmark 包含 149 个 target firm-quarter，公告范围为 2025-12-02 至 2026-06-24。
- Qwen 在 149 个 benchmark 实例上的 memorization probe 为 0 hits。该结果降低了直接记忆的担忧，但不能完全排除未披露训练语料造成的 leakage。

---

## Slide 10｜点预测结果

![Slide 10](/learning_report_slides/slide-10.png)

### 注释

- Analyst consensus 的 median price-deflated error 为 0.111%。
- Opus 4.6、GPT-5.5 和 GPT-5.4 的 PDE 为 0.113%–0.131%，在约 50% 的实例上击败 consensus。
- 三个领先模型相对 consensus 的 paired difference 之 90% confidence interval 均包含 0。
- 因此论文的结论是领先模型与分析师共识“无显著差异”，而不是模型已经优于分析师。
- GPT-5-mini、Qwen 3.5 和 GLM 4.5 Air 的误差明显更高。

---

## Slide 11｜所有模型都低估预测不确定性

![Slide 11](/learning_report_slides/slide-11.png)

### 注释

- 名义 80% prediction interval 理论上应覆盖约 80% 的实际 EPS。
- 实际覆盖率只有 37%–70%，没有模型达到目标。
- 即使表现最好的 Opus 4.6，覆盖率也只有 70%。
- Miss 存在明显方向性：实际 EPS 经常高于 p90。
- 原因不是单纯区间太窄，还包括点预测整体偏低，导致整个区间随点预测一起下移。

---

## Slide 12｜大幅盈利变化拉开模型差距

![Slide 12](/learning_report_slides/slide-12.png)

### 注释

- Near instances 中，各模型与分析师的差距相对有限。
- Far instances 指 EPS 大幅偏离去年同期的季度，共 50 个实例。
- 在 far instances 上，领先模型的 PDE 约为 0.39%，接近 analyst consensus 的 0.409%；trailing models 为 0.599%–0.716%。
- 领先模型在 95%–96% 的关键 driver 调整中引用具体数字，而 trailing models 只有 35%–72%。
- 关键启示：面对大幅变化，定性叙述不够，模型需要找到能够支持调整幅度的具体数值证据。

---

## Slide 13｜信息源消融

![Slide 13](/learning_report_slides/slide-13.png)

### 注释

- 仅保留 news 时，没有检测到显著的 paired error 变化。
- 保留 filings 与 transcripts、移除 news 时，误差小幅上升。
- 同时移除两类来源时，PDE 从 0.131% 上升到 0.210%，win rate 从 50% 降到 31%。
- 实例层面存在异质性：24 个实例主要依赖 news，17 个主要依赖 filings/transcripts，9 个对任一来源的移除都敏感。
- 这说明不同来源可以部分替代，但 Agent 仍需判断特定公司和季度的决定性证据在哪里。

---

## Slide 14｜误差来源与 Boeing 案例

![Slide 14](/learning_report_slides/slide-14.png)

### 注释

- 会计误差分解显示，模型差距主要集中在 operating income，而不是 non-operating 或 tax/shares。
- Boeing 2025Q4 出售 Jeppesen 和 ForeFlight，交易额为 $10.55B。
- 真正难点不是发现这笔交易，而是预测当季应确认的会计收益。
- 领先模型估计确认收益约 $8.0B–$9.6B，最终 EPS 接近实际 GAAP diluted EPS $10.23。
- Qwen 3.5 和 GLM 4.5 Air 几乎未确认该收益，因此产生巨大的 EPS 偏差。
- 案例说明：信息检索成功不等于预测成功，模型还要正确判断交易进入利润表的金额与时点。

---

## Slide 15｜总结与限制

![Slide 15](/learning_report_slides/slide-15.png)

### 注释

1. Company-specific schema 使不同模型的预测可比较、可审计。
2. 三个领先模型的点预测准确度已接近分析师共识。
3. 所有模型的 80% prediction interval 都存在欠覆盖。
4. 改进重点是 operating drivers 的数值估计和独立的不确定性校准。

研究限制包括：样本集中于美国大型公司；分析师可能拥有更广的信息来源；benchmark 结果不能直接证明模型预测适合真实投资决策。

---

## 二、个人学习总结
### 1. EPSBench 的核心创新

它把难以比较的端到端 EPS 预测，转化为统一的 driver prediction 和 accounting calculation。模型保留研究与判断空间，但不能任意选择会计口径或计算方法。这使最终误差具备可诊断性。

### 2. 点预测准确不等于风险预测可靠

领先模型可以接近分析师的点预测误差，但区间覆盖率仍明显不足。真实应用若只关注 MAE 或 PDE，可能高估模型的风险管理能力。Point accuracy 与 uncertainty calibration 应当分别训练和评估。

### 3. 复杂季度需要数值化证据

模型之间最大的差距出现在盈利大幅变化的季度。领先模型不仅发现事件，还能找到支持 driver 调整幅度的数字。Boeing 案例进一步说明，交易金额、会计确认金额和确认时点是不同概念。

### 4. Leakage 结论应保持克制

已披露 cutoff 且早于目标公告的模型风险相对较低。对于 cutoff 未披露的模型，0-hit memorization probe 是缓解证据，而不是无泄漏证明。评估结论应区分“没有发现直接记忆”与“确定不存在训练语料污染”。

## 三、资料来源

- 论文：Yin et al., *EPSBench: Benchmarking LLM Agents for Earnings Forecasting*, arXiv:2610.38293。
- Slide 图表：论文 LaTeX source package 中的原始图表或按论文数值重建的可编辑表格。
- MSFT 2026Q3 案例：EPSBench 官方仓库中的 `MSFT_2026Q3` schema 与 `gpt-5.5__opencode__default__full/s1` prediction record。
