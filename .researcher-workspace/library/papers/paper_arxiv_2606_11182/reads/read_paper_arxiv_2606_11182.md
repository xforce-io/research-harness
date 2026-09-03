---
title: "EEVEE: Towards Test-time Prompt Learning in the Real World for Self-Improving Agents"
authors: ["Weixian Xu","Shilong Liu","Mengdi Wang"]
paper_id: "paper_arxiv_2606_11182"
source_kind: "arxiv"
source_id: "arxiv:2606.11182"
source_url: "https://arxiv.org/abs/2606.11182"
pdf_url: "https://arxiv.org/pdf/2606.11182"
read_id: "read_paper_arxiv_2606_11182"
kind: library-read
tags: []
---

# EEVEE: Towards Test-time Prompt Learning in the Real World for Self-Improving Agents

> 首个多数据集测试时 prompt 学习框架，通过 router-prompt 协同进化消除跨数据集干扰。

## Brief

现有测试时 prompt 学习方法在单一数据集上有效，但面对真实世界中异构多领域任务流时，单一 prompt 会因跨数据集干扰而丢失任务特化行为。EEVEE 引入路由器将输入流划分为任务簇并分配到专用 prompt 配置，通过三阶段协同进化策略交替优化路由器与 prompt。在四基准混合测试中，EEVEE 在 Qwen3-4B-Instruct 和 DeepSeek-V3.2 上分别提升 10.38 和 24.32 分，并实现 +41.53 的累积留存增益，而基线 GEPA 和 ACE 均为负值。

## Claims

- 多数据集测试时 prompt 学习中，单一 prompt 会因跨数据集干扰而丢失任务特化行为：在四任务增量设置下，GEPA 和 ACE 的累积留存分别为 -15.36 和 -18.58，而 EEVEE 保持 +41.53 [1: §1, Fig.1]。
- EEVEE 在四基准套件上将 Qwen3-4B-Instruct 的平均分从 41.37 提升至 51.75（+10.38），将 DeepSeek-V3.2 从 39.75 提升至 64.07（+24.32）[1: §3.2, Table 1]。
- EEVEE 分别比 GEPA 和 ACE 高出最多 37.2% 和 48.2% [1: §1]。
- 学习型路由器和协同进化均为必要：默认路由器仅比基线提升 2.21 分，手动路由器下降 4.19 分，无协同进化仅提升 1.51 分，完整方法提升 10.38 分 [1: §3.3, Table 2]。
- 在单基准设置下 EEVEE 仍具竞争力，说明 prompt 学习设计本身有效，路由器特化不损害单任务性能 [1: §3.4, Fig.4]。
- 跨模型迁移：在 Qwen3-4B-Instruct 上学到的 prompt 直接应用于 DeepSeek-V3.2 可将平均分从 39.75 提升至 54.10 [1: §3.5, Table 3]。
- Token 开销方面，EEVEE 平均每例使用 4.32k tokens，接近 GEPA 的 3.47k，远低于 ACE 的 21.30k（约 4.9× 节省）[1: §3.6, Fig.5]。
- Prompt 学习在可转化为可复用程序的任务（代码、公式）上增益最大，在知识密集型 QA（GPQA Diamond）上反而可能有害（6 次运行中仅 1 次正向）[1: §3.7, Table 4]。
- 超参数鲁棒性：八种超参数配置的平均分跨度仅 5.92 分，标准差 1.73 分，无配置崩溃 [1: §B.1, Table 6]。

## Assumptions

- 目标模型 M 的权重固定，适配仅通过 prompt 实现，不更新模型参数 [1: §2.1]。
- 反馈依赖 ground-truth 或规则标签，而非纯反思式学习；需要准备好的适配数据集而非完全在线流 [1: §6]。
- 每个基准的数据量上限为 500 条，训练/测试 0.5/0.5 划分，训练集再分出一半作为验证集 [1: §B]。
- 路由器 prompt 和模型 prompt 均为自然语言文本，通过 LLM 生成和变异 [1: §2.2]。
- 基准测试使用规则判分器（rule-based judges），假设其可靠 [1: §B]。

## Method

EEVEE 维护一组专用 prompt $P = \{p_1, \ldots, p_K\}$ 和路由器 $R$，目标模型 $M$ 固定。推理时路由器先为输入 $x$ 选择 prompt 槽位 $z = R(x; P) \in \{1,\ldots,K\}$，模型再用对应 prompt 生成答案 $\hat{y} = M(x; p_z)$。

**路由器-prompt 协同进化**交替两个阶段：
- **RouterEvolve**：固定 prompt 集，搜索更好的路由器。从可被至少一个现有槽位 prompt 解决的训练样本中采样 mini-batch，对参考路由器进行变异得到 $R_{mut}$，分析路由失败但其他槽位成功的案例，通过反思生成 $R_{ref}$，按下游准确率选择最优候选。评分函数 $S_R = \lambda_{acc}A + \lambda_{con}C + \lambda_{bal}B$，权重从一致性/平衡性退火到下游准确率。
- **PromptEvolve**：固定路由器，将数据路由到各槽位后独立并行进化每个槽位 prompt。使用变异和反思生成候选，按 Pareto 前沿池管理互补 prompt，候选需优于空 prompt 且位于 Pareto 前沿方可入池。

**三阶段训练**：
1. **初始化**：在混合训练集上运行 prompt 学习，通过贪心覆盖规则从 Pareto 前沿池中选择 Top-K 互补 prompt，为路由器提供可区分的初始行为。
2. **探索**：从 $(R_0, P_0)$ 出发，在轻量预算下交替路由器和 prompt 进化，频繁切换以避免在不稳定路由器上浪费预算或对过时 prompt 过拟合。
3. **收敛**：固定稳定路由器 $R^\star$，重新路由数据，在每个槽位内投入更大 prompt 学习预算。

## Eval

**数据**：四个基准——GPQA Diamond（闭书知识 QA）、Formula（数学/符号推理）、TheoremQA（定理推理）、HumanEval（代码生成）；额外含 MBPP、MMLU-Pro、FiNER、IFBench 用于泛化和单基准测试。每个基准上限 500 条，训练/测试 0.5/0.5 划分。

**基线**：未适配目标模型、GEPA（反思式 prompt 进化 + Pareto 前沿选择）、ACE（自适应上下文 playbook）。

**模型**：Qwen3-4B-Instruct（temperature 0.7, top-p 0.8）、DeepSeek-V3.2 非思考模式（temperature 1.0, top-p 0.95）。

**主要指标**：基准准确率百分比，三次运行取均值。

**关键结果**：
- Qwen3-4B 上 EEVEE 平均 51.75 vs GEPA 37.73 vs ACE 34.92 vs 基线 41.37 [1: Table 1]。
- DeepSeek-V3.2 上 EEVEE 64.07 vs GEPA 55.83 vs ACE 49.83 vs 基线 39.75 [1: Table 1]。
- 消融：完整方法 51.75 vs 默认路由器 43.58 vs 手动路由器 37.18 vs 无协同进化 42.88 [1: Table 2]。
- 运行间标准差：Qwen3-4B 上 1.62 分，DeepSeek-V3.2 上 1.08 分 [1: Table 7]。

## Weaknesses

- 路由器评分函数中的 $\lambda_{acc}$、$\lambda_{con}$、$\lambda_{bal}$ 权重以及退火策略的具体调度曲线未公开完整公式，论文仅给出初始权重 0.6/0.2/0.2 和最终 1.0/0.0/0.0，中间退火过程不可复现 [1: §2.2, §B]。
- 评估仅使用四个基准，且每个基准截断至 500 条，对于"真实世界异构任务流"的代表性有限——真实部署中任务类型和分布远比四个学术基准复杂 [1: §3.1]。
- 路由器依赖 LLM 作为 prompt researcher/reflector/reasoner，但这些辅助 LLM 调用的具体模型和成本未报告，仅报告了最终测试时的 token 开销，训练阶段开销不透明 [1: §C]。
- GPQA Diamond 上 EEVEE 在 6 次运行中仅 1 次正向（Table 4），且论文承认学习到的推理可能弱化领域知识，但未提出缓解方案，仅作为 case study 观察 [1: §3.7]。
- 跨任务泛化中 MMLU-Pro 下降 1.82 分，虽声称小于 GEPA 的 1.89 分，但与 ACE 的 1.42 分相比实际更差，论文对此差异未充分讨论 [1: §3.5, Table 3]。
- 数据流中任务按固定顺序（GPQA Diamond → Formula → TheoremQA → HumanEval）引入，但未测试任务顺序敏感性，增量留存结果可能依赖该特定顺序 [1: §1, Fig.1]。

## Relations

- builds-on GEPA [high]: EEVEE 的 prompt 进化直接采用 GEPA 的 Pareto 前沿选择机制和自然语言反思策略，并在实验中以 GEPA 作为主要基线；论文 §4 明确引用并对比 [1: §4]。
- builds-on ACE [high]: EEVEE 以 ACE 的自适应上下文 playbook 为基线，且 token 开销分析（§3.6）直接针对 ACE 的 prompt 膨胀问题；论文 §4 明确引用 [1: §4]。
- competes-with Combee [med]: Combee 通过并行 trace 聚合扩展 prompt 学习，EEVEE 通过路由器分区处理多数据集，两者均解决 prompt 学习的可扩展性但路径不同——Combee 扩展单任务规模，EEVEE 扩展任务多样性；论文 §4 将 Combee 列为相关工作 [1: §4]。
- extends Reflexion [med]: Reflexion 使用自然语言反馈和言语记忆进行自我改进，EEVEE 将反思式 prompt 学习从单任务扩展到多数据集异构流；论文 §4 明确引用 Reflexion 作为 self-improving agent 的先驱工作 [1: §4]。
