---
title: "The Price Reversal Phenomenon: When Cheaper Reasoning Models Cost More"
authors: ["Lingjiao Chen","Chi Zhang","Yeye He","Ion Stoica","Matei Zaharia","James Zou"]
paper_id: "paper_arxiv_2603_23971"
source_kind: "arxiv"
source_id: "arxiv:2603.23971"
source_url: "https://arxiv.org/abs/2603.23971"
pdf_url: "https://arxiv.org/pdf/2603.23971"
read_id: "read_paper_arxiv_2603_23971"
kind: library-read
tags: []
---

# The Price Reversal Phenomenon: When Cheaper Reasoning Models Cost More

> 列价更低的推理模型在 32% 的模型对比较中实际花费更高，因为 thinking token 和交互轮次的异质性可以压倒单位价格优势。

## Brief

本文系统研究了 8 个前沿推理模型在 12 个任务上的实际推理成本与 API 列价之间的差距。核心发现是"价格反转"现象：在 32% 的模型对比较中，列价更低的模型实际花费更高，最高达 28 倍。作者基于 Shapley 值构建了成本归因框架，指出 thinking token 和交互轮次是驱动反转的主要因素，并证明同一查询的重复运行成本变异可达 9.7 倍，使逐查询成本预测成为一个分布预测而非点估计问题。

## Claims

- 在 8 个前沿推理模型、12 个任务上的 336 对模型比较中，32% 的对比出现价格反转，即列价更低的模型实际总成本更高 [§3]。
- 反转幅度最高可达 28 倍；例如 Gemini 3 Flash 的列价比 GPT-5.4 低 80%，但跨所有任务的实际总成本反而高 38% [§3]。
- 反转率因任务而异：ArenaHard 为 11%，MMLUPro 为 57% [§3]。
- 在单轮任务中，thinking token 数量是成本差异的主导因素，占反转模型对成本差异的 95% 以上 [§4.2]。
- 在多轮 agent 任务中，交互轮次是主导因素；例如 Kimi K2.6 与 GPT-5.4 之间，轮次贡献了超过 80% 的成本差异 [§4.2]。
- 同一 (模型, 查询) 对在重复运行下，成本变异最高达 9.7 倍，变异系数常规超过 0.3 [§5.1]。
- 在成本分布的 Q15–Q40 分位区间（用户实际预算触及的分位），反转率从均值的 31.5% 上升至 39–40% [§5.2]。
- 任务难度放大了轮次的归因贡献：困难任务上"更便宜"模型会执行更多过度操作 [§4.2]。
- 多轮 agent 任务中，二次上下文增长（而非 thinking token）主导成本，且低成本列价模型更频繁耗尽消息预算 [附录 E]。

## Assumptions

- 列价以输入价 + 输出价之和（Lm = pi + po）作为排名依据，这是从业者常用的简化比较方式，但未考虑缓存价格等四参数定价的全部细节 [§2]。
- 每个模型使用单一推理配置（如 reasoning effort="high"），未对 thinking effort 与成本/质量之间的权衡进行消融 [附录 G]。
- 成本与质量解耦分析：论文仅关注成本而不考虑质量差异，但声称"更强大的模型通常质量更高，加入质量只会加强反转发现" [§2]——此声明未经验证。
- 定价为 2026 年 5 月 1 日的单一快照，假设在此期间模型行为和定价结构不变 [附录 G]。
- 多轮 agent 使用标准的 ReAct 或 Terminus 2 scaffold 且不修改 agent 循环，假设 scaffold 本身不引入系统性偏差 [附录 A.4]。

## Method

**成本审计框架**：对单轮任务，成本公式为 c_m(q) = p_i · n_i + p_o · n_o，即输入/输出 token 数乘以对应单价。对多轮 agent 任务，扩展为四参数定价（输入、输出、cache write、cache read），按轮次求和。

**成本归因框架**：定义 8 个可观测成本因子（总轮数 T、平均每轮 fresh input tokens、平均每轮 output tokens、平均每轮 cached tokens、平均每轮 thinking tokens、input 单价、output 单价、cached 单价）。对两个模型 A 和 B，成本差 ΔC = C_A − C_B 通过 hybrid cost 函数归因到各因子：将子集 S 中的因子取模型 A 的值、其余取模型 B 的值，构造合作博弈并计算 Shapley 值。由于成本函数对 8 个因子是多线性的，推导出闭式解。

**成本分布分析**：对固定 (模型, 查询) 对进行重复 rollout，记录每次运行的实际成本，绘制经验分布。在分位层面重新计算反转率：对每个查询用分位数 c^(q)_m 替代均值，统计 {均值, Q1, Q5, ..., Q99} 各统计量下的反转率。

## Eval

- **模型**：GPT-5.4、GPT-5.4 Mini、Gemini 3.1 Pro、Gemini 3 Flash、Claude Opus 4.7、Claude Haiku 4.5、Kimi K2.6、MiniMax M2.7。
- **任务**：9 个单轮任务（AIME、ARC-AGI、ArenaHard、GPQA、HLE、LiveCodeBench、LiveMathBench、MMLU-Pro、SimpleQA）+ 3 个多轮 agent 任务（Terminal-Bench 2.0、Cybench、GAIA），共 6877 个唯一任务。
- **Agent scaffold**：Terminal-Bench 2.0 用 Terminus 2；Cybench 和 GAIA 用标准 ReAct，message limit = 1000。
- **指标**：listed price（p_i + p_o）、actual total cost（含缓存定价）、价格反转率（列价更低但实际成本更高的模型对比例）、Shapley 归因贡献值、per-query 成本变异系数、分位反转率。
- **基线**：无传统意义上的基线；论文的比较框架是 listed price ranking vs. actual cost ranking 的系统性偏差。
- **规模**：超过 7.39B tokens 生成，190,000 次轮级 API 调用，总支出超 $5,000。

## Weaknesses

- 论文声称"加入质量只会加强反转发现" [§2]，但未提供任何质量-成本联合分析或 cost-of-correct-answer 指标来验证此论断。如果低成本模型在困难任务上既花更多钱又答错，反转的严重性可能被低估或被高估——两种方向都有可能，但论文无法区分。
- 9.7 倍的 within-query 成本变异来自重复 rollout，但论文未报告 rollout 次数或随机种子控制方案。如果每个 (模型, 查询) 对仅运行少量次数，分布估计本身可能不可靠。
- 8 个模型使用不同 provider 的默认推理配置（如 Opus 用 adaptive thinking，MiniMax 用内置 thinking，OpenAI 用 reasoning effort="high"），这些配置不可直接比较——"高推理努力"对不同 provider 的含义可能不同，归因框架将此差异归入 thinking token 因子，但未分离配置选择与模型本身的影响。
- 多轮 agent 任务的反转分析使用 Terminus 2 和 ReAct 两种不同 scaffold，但未分析 scaffold 选择对反转率的影响。scaffold 的 turn budget 和上下文管理策略可能本身就是多轮成本的主要驱动因素。
- 分位反转率分析（§5.2）将 12 个数据集 pooled 后报告 Q15–Q40 的反转率上升，但不同任务类型的分位曲线形状不同（推理任务倒 U 型，agent 任务双峰），pooled 统计量可能掩盖特定任务类型的极端情况。

## Relations

- **contradicts** FrugalGPT / FrugalML / 模型路由相关工作的隐含假设 `[med]`：本文 §6 明确指出 FrugalGPT [Chen et al., 2024b] 和 RouterBench [Hu et al., 2024] 等路由系统"typically assume that the per-query cost of each model is known or can be estimated from API pricing"，而本文的核心发现是此假设系统性失效。这是直接的概念矛盾，但本文未实证测试路由系统在反转条件下的性能下降。
- **extends** cost-of-pass 框架 `[med]`：Erol et al. [2026] 提出 cost-of-pass 经济框架衡量获取正确答案的财务成本，本文扩展了这一思路，指出即使不考虑质量，仅成本本身也因 thinking token 和 turn 数量的异质性而不可从列价预测 [§6]。
- **builds-on** Shapley 值归因方法 `[high]`：本文将 SHAP [Lundberg and Lee, 2017] 的特征归因范式和 Data Shapley [Ghorbani and Zou, 2019] 的数据估值方法适配到成本差异分解，§4.1 明确引用了 Shapley [1953] 并推导了多线性闭式解。
- **orthogonal** overthinking / efficient reasoning 研究 `[med]`：Chen et al. [2024c] 和 Sui et al. [2025] 研究如何减少 o1 类模型的过度思考以提升效率，本文从用户侧成本视角量化了 overthinking 的财务后果，但未提出缓解方法——两者关注同一现象的不同侧面。
