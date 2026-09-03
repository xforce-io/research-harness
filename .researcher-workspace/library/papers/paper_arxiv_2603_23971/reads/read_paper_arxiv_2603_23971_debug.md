---
title: "The Price Reversal Phenomenon: When Cheaper Reasoning Models Cost More"
authors: ["Lingjiao Chen","Chi Zhang","Yeye He","Ion Stoica","Matei Zaharia","James Zou"]
paper_id: "paper_arxiv_2603_23971"
source_kind: "arxiv"
source_id: "arxiv:2603.23971"
source_url: "https://arxiv.org/abs/2603.23971"
pdf_url: "https://arxiv.org/pdf/2603.23971"
read_id: "read_paper_arxiv_2603_23971_debug"
kind: library-read
tags: []
---

# The Price Reversal Phenomenon: When Cheaper Reasoning Models Cost More

> Listed API prices for reasoning models are an unreliable proxy for actual cost: in 32% of model-pair comparisons, the cheaper-listed model costs more.

## Brief

本文系统研究了 8 个前沿推理模型（RM）在 12 个任务上的实际推理成本，发现"价格反转"现象：在 32% 的模型对比较中，标价更低的模型实际总成本更高，反转幅度最高达 28 倍。论文构建了基于 Shapley 值的成本归因框架，揭示 thinking token 消耗量和多轮交互轮数是成本反转的主要驱动因素。论文进一步证明同一查询的重复运行成本变化可达 9.7 倍，使逐查询成本预测本质上是一个分布估计问题。

## Claims

- 在 8 个前沿 RM 和 12 个任务的 336 次成对比较中，32% 存在价格反转现象（标价更低的模型实际成本更高），反转率在不同任务间从 11%（ArenaHard）到 57%（MMLUPro）不等 [§3]。
- 价格反转的幅度可以极大：Gemini 3 Flash 的标价比 Claude Haiku 4.5 便宜 1.7 倍，但在 MMLUPro 上实际成本高出 4 倍 [§3]。
- 没有任何一个模型在所有任务上始终最便宜或最昂贵；模型间的成本排名高度依赖任务 [§3]。
- 基于Shapley值的成本归因表明，单轮任务中 thinking token 数量是最主要的成本差异贡献者（>95%）；多轮任务中交互轮数和缓存输入 token 是主要贡献者 [§4.2]。
- 同一（模型, 查询）对在相同配置下的重复运行，成本变化跨度可达 9.7 倍，变异系数经常超过 0.3 [§5.1]。
- 在成本分布的中段分位数（Q15–Q40），反转率从均值的 31.5% 升至 39–40%；高尾分位数 Q99 反而最低（25.0%）[§5.2]。
- 任务难度加剧价格反转：困难任务放大了交互轮数的贡献，简单任务上反转几乎消失 [§4.2]。

## Assumptions

- "列出价格"（listed price）= 输入价格 + 输出价格，这是从业者常用的简化比较方式 [§2]。论文未验证该简化在所有从业者中的普适性。
- 各模型的推理配置（reasoning effort="high"、adaptive thinking 等）设为各自推荐的最优设置，且在所有任务上固定不变 [§A.3]。论文假设这些设置代表了用户的典型使用方式。
- 定价快照为 2026 年 5 月 1 日的数据 [§A.2]；论文假设该快照在分析期间有效，但承认提供商可能重新定价。
- 成本与质量解耦分析：论文明确声明不考虑质量，并认为加入质量只会加强反转结论 [§2]。该假设未经验证。
- 多轮任务使用标准 ReAct 脚手架或数据集默认 agent，message limit 统一设为 1,000；论文假设这些脚手架配置对成本比较是公平的 [§A.4]。

## Method

1. **成本审计框架**：定义单查询成本 $c_m(q) = p_{i,m} \cdot n_{i,m}(q) + p_{o,m} \cdot n_{o,m}(q)$；对于 agentic 任务，扩展为包含 cache write/read 的四参数模型，按轮次累加 [§2, Eq.1-2]。
2. **价格反转检测**：对 8 个模型 × 12 个任务，计算每对模型的列出价格排名与实际总成本排名，统计排名不一致的比例 [§3]。
3. **Shapley 成本归因**：将总成本表达为 8 个因子（总轮数 $T$、平均输入/输出/缓存/thinking token 数、4 种单价）的多线性函数；定义 hybrid cost $v(S)$ 为子集 $S$ 中因子取模型 A 值、其余取模型 B 值时的成本差；证明满足零贡献性、对称性、可加性、完备性的唯一归因为经典 Shapley 值，并给出多线性函数下的闭式解 [§4.1, Eq.3-6]。
4. **成本分布分析**：对固定（模型, 查询）对进行多次 rollout，绘制成本直方图；将逐查询均值成本替换为各分位数（Q1–Q99），重新计算反转率 [§5.1-5.2]。

## Eval

- **模型**：GPT-5.4, GPT-5.4 Mini, Gemini 3.1 Pro, Gemini 3 Flash, Claude Opus 4.7, Claude Haiku 4.5, Kimi K2.6, MiniMax M2.7（共 8 个）[§2]。
- **数据集**：9 个单轮任务（AIME, ARC-AGI, ArenaHard, GPQA, HLE, LiveCodeBench, LiveMathBench, MMLUPro, SimpleQA）+ 3 个多轮 agentic 任务（TerminalBench 2.0, Cybench, GAIA），共 6877 个唯一任务 [§2, Table 2]。
- **指标**：列出价格 vs. 实际总成本（USD）；反转率（反转对数 / 总对数）；Shapley 归因值（USD）；逐查询成本分布的变异系数和分位数跨度。
- **基线**：列出价格排名作为隐含基线；无外部预测器基线——论文论证预测本身是 open challenge。
- **规模**：总计消耗 >$5,000 USD，>190,000 次 turn-level API 调用，>7.39B tokens [§A.6]。

## Weaknesses

- 论文的核心反转结论基于每个（模型, 查询）对的单次运行成本，但 §5 自己证明同查询重复运行成本变化达 9.7 倍。这意味着 §3 中 32% 的反转率本身可能在高方差噪声下不稳定——论文未报告反转率在重复采样下的置信区间或翻转概率。
- 成本与质量完全解耦，但从业者的实际决策是 cost-per-correct-answer。论文声称加入质量只会加强反转结论 [§2]，但未提供任何证据支持该断言；如果高标价模型在困难任务上正确率显著更高，cost-of-pass 排名可能与纯成本排名不同。
- 多轮任务仅使用 ReAct 和 Terminus 2 两种脚手架，且 message limit 固定为 1,000。不同脚手架（如 Reflexion、multi-agent debate）可能改变轮数分布和缓存模式，从而改变归因结论——论文未讨论脚手架选择对成本归因的敏感度。
- 定价快照为单一时间点（2026-05-01），但 RM 市场价格变动频繁；论文虽在 Limitations 中承认这一点，但未评估历史价格变动下反转率的稳定性，使结论的时间泛化性不明。
- Shapley 归因的 8 个因子选择缺乏消融：论文称这些因子"完全决定成本"并满足可独立替换性 [§4.1]，但 thinking token 和 output token 在所有提供商处均按输出价格计费 [§A.2]，二者在公式中线性耦合，实际可替换性存疑。

## Relations

- `contradicts` FrugalGPT / FrugalML / 模型路由文献 [§6, high]：论文 §6 Related Work 明确指出 FrugalGPT [Chen et al., 2024b]、FrugalML [Chen et al., 2020] 及 LLM 路由工作"typically assume that the per-query cost of each model is known or can be estimated from API pricing"，而本文的核心发现正是该假设在推理模型上系统性失效。同一作者群（Chen, Zaharia, Zou）既写了 FrugalGPT 又写了本文，使该 contradiction 尤为显著。
- `builds-on` cost-of-pass 框架 [§6, high]：Erol et al. [2026] 提出 cost-of-pass 经济框架衡量获取正确答案的实际费用；本文 §6 明确引用并指出该框架"measuring the actual financial expense required to obtain a correct answer"，本文将这一思路从质量感知成本扩展到纯成本审计，揭示即使不考虑质量，成本本身也难以从标价推断。
- `orthogonal` overthinking / 高效推理文献 [§6, high]：Sui et al. [2025] 和 Chen et al. [2024c] 研究推理模型的 overthinking 问题及其优化方法；本文 §6 明确区分——这些工作关注减少不必要推理以提升效率，而本文关注 overthinking 的用户侧成本后果。两者共享 thinking token 过量消耗这一现象，但从不同角度切入。
- `extends` compound AI systems 视角 [§6, high]：Zaharia et al. [2024] 提出 compound AI systems 是部署的基本单元；本文 §6 明确引用并指出这些系统"amplify the cost question"——每轮触发一次 RM 调用，模型间的微小差异会大幅放大总账单，将 compound systems 的成本不可预测性量化为具体数字。
