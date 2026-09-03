---
title: "Language Models Need Sleep: Learning to Self-Modify and Consolidate Memories"
authors: ["Ali Behrouz","Farnoosh Hashemi","Adel Javanmard","Vahab Mirrokni"]
paper_id: "paper_arxiv_2606_03979"
source_kind: "arxiv"
source_id: "arxiv:2606.03979"
source_url: "https://arxiv.org/abs/2606.03979"
pdf_url: "https://arxiv.org/pdf/2606.03979"
read_id: "read_paper_arxiv_2606_03979"
kind: library-read
doc_type: "paper"
tags: []
---

# Language Models Need Sleep: Learning to Self-Modify and Consolidate Memories

> LLM 无法将 in-context 短期记忆持久化为长期参数——本文引入"睡眠"范式，通过向上蒸馏（Knowledge Seeding）与 RL 自我改进（Dreaming）将脆弱记忆固化到新激活的低秩参数中，实现持续学习且缓解灾难性遗忘。

## Essence

**问题**：现有 LLM 在预训练后静态不变；fine-tuning/LoRA 等轻量更新在迭代中引发灾难性遗忘（CF），重新预训练代价过高；ICL 虽能即时适应，但会话结束后知识即丢失。

**做法**：提出 Sleep 范式，将持续学习生命周期分为 Wake（接收新数据）与 Sleep（无外部输入、内部自我处理）两阶段。Sleep 又分两步：(1) Memory Consolidation——在由不同更新频率 MLP 块组成的 Continuum Memory System 中，周期性为低频块激活新的低秩专家参数，通过 on-policy 蒸馏 + RL 模仿学习（Knowledge Seeding）将高频块知识向上蒸馏至新参数，随后对高频块的临时专家做突触剪枝；(2) Dreaming——模型基于当前状态自生成合成数据（dreams），经梯度重要性筛选后用 LoRA 做 SFT 微调，以 RL 奖励机制强化能提升性能的 dream。

**证据**：在 BABILong 基准上扩展至 10M token 时接近满分；在数学推理 AIME-24 上 Sleep 达 53.2（Qwen3-1.7B），超过 GRPO 的 51.0 和 SFT 的 47.3 [1: Table 2]。

**边界**：依赖多频率 MLP 链式架构（CMS/Hope），低秩专家需预分配并 mask；Dreaming 阶段计算开销大（每个 dream 需独立 SFT 实例），且语义奖励模型需冻结的外部 reward model。

## Claims

1. 持续学习器不存在训练/测试时间之分，只有 Wake（接收数据）和 Sleep（内部处理）两种状态，传统 train/test 划分不适用于终身学习 [1: §3.1]。
2. Knowledge Seeding（向上蒸馏）允许小模型/低容量状态将知识蒸馏到大模型/高容量状态，与常规蒸馏方向相反，避免 student 因训练在 teacher 生成数据上而次优使用参数 [1: §3.3]。
3. on-policy 蒸馏与 RL 模仿学习（LTI）的组合使学生模型不仅存储知识，还学会模仿教师的采样过程；仅蒸馏而不加 LTI 时学生"有知识但不会用" [1: §3.3]。
4. 参数周期性（de）激活——在低频块中逐步激活新低秩专家、在高频块中剪枝旧专家——在保持可塑性的同时避免新旧知识干扰 [1: §3.2]。
5. 增加记忆层级数（sleep 阶段数）单调提升 in-context learning 和长上下文理解能力；降低最低频率块的持久性会削弱记忆保持 [1: §4.1, Figure 4]。
6. Dreaming 阶段中随机专家选择（在 MoE 路由中额外随机选取无关专家）混合跨域知识，帮助探索隐藏模式 [1: §3.4]。
7. 在 SQuAD 知识整合（单段落设置）中，四层记忆 Sleep 达 48.9，超过 SEAL 的 46.7；移除 Dreaming 降至 35.7 [1: Table 3]。
8. 在 few-shot ARC 抽象推理中 Sleep 达 80% 成功率，超过 SEAL 的 72.5% 和 TTT 的 10% [1: Table 4]。
9. Sleep 的两步设计（先巩固再做梦）比直接迭代自我改进更能抵抗灾难性遗忘 [1: §3.4, Table 3]。

## Assumptions

- 模型架构基于 Continuum Memory System（CMS），即注意力模块后接多个不同更新频率的 MLP 块链；标准 Transformer 需改造才能适用。
- 低秩专家参数可以预分配并在前向/反向传播中 mask，直到 sleep 阶段激活——这要求模型初始即预留足够容量（论文承认这与"大脑固定容量"类比一致，但实际实现中需预设最大参数量）[1: §3.2 Note on Implementation]。
- 语义奖励模型（reward model for r_sem）可用且冻结，其判断学生生成与教师生成语义等价性的能力被视为可靠。
- Dreaming 阶段中每个 dream 的独立 SFT 实例计算量是可接受的——论文未讨论该开销的实际可行性。
- BABILong 等基准的零样本评测与大模型微调评测之间的比较是公平的（论文 Figure 6 混合了 zero-shot 和 fine-tuned 设定）。

## Method

**架构基础对比**：

| 常规 Transformer | Sleep 范式 |
|---|---|
| Attention = 短期记忆（会话结束即丢） | CMS = 多频率 MLP 链，形成记忆频谱 |
| MLP = 冻结的长期记忆（训练后不更新） | MLP 块按不同频率更新，低频块 = 长期记忆 |
| 无显式知识巩固机制 | Sleep 阶段显式蒸馏 + 自我改进 |

**输入 -> 计算 -> 输出**：

**输入**：Wake 阶段累积的 in-context 知识，存储在高频 MLP 块的低秩专家中。

**核心计算——Memory Consolidation（NREM 对应）**：
1. 触发条件：当步数整除某块的 chunk length C^(ℓ) 时触发 sleep。
2. **Compute**：计算高频块（sender, MLP^(f_{ℓ*-1})）的累积梯度更新，得到 prospective 参数，但不立即应用。
3. **Consolidate**：定义 Teacher（更新前的旧状态模型）和 Student（使用 prospective 参数 + 在低频块 MLP^(f_{ℓ*}) 中新增低秩专家 A∈R^{d×d_low}, B∈R^{d_low×d} 的扩展模型）。通过 Knowledge Seeding 目标优化 A, B：
   - On-policy 蒸馏损失：混合 teacher 生成数据 (1-λ) 和 student 生成数据 (λ)，计算输出分布散度 F。
   - RL 模仿学习（LTI）：teacher 生成 dream 序列，随机截取前缀，student 续写；奖励 r = γ·r_sem + (1-γ)·r_abs，其中 r_sem 为语义等价二值奖励，r_abs 基于 Levenshtein 距离。
   - 总目标 L_KS = (1-α)·E[r(y)] - α·E[D(teacher‖student)]。
4. **Update**：应用高频块的基础权重更新；剪枝高频块的旧低秩专家（synaptic pruning）；激活低频块的新专家 {A,B}。

**核心计算——Dreaming（REM 对应）**：
1. 给定上下文 C，从模型 LM_θ 采样 m 个 dreams；MoE 路由器额外随机选取无关专家以混合知识。
2. 计算每个 dream 的梯度重要性分数 g = ∇_θ L_SFT(DREAM, θ)，选 Top-k + b 个随机样本。
3. 对每个 dream 用独立 LoRA 实例做 SFT：θ' ← SFT(θ, DREAM)。
4. 以新模型 θ' 在任务 τ 上的性能提升作为二值奖励（改进=1，否则=0），用 ReSTEM 算法优化采样策略。

**输出**：更新后的模型参数，新知识固化在低频长期记忆块中。

## Eval

**持续学习（Class-Incremental）**：
- 数据：CLINC、Banking、DBpedia。
- Backbone：Llama-3B、Llama3-8B。
- Baseline：ICL、EWC、InCA、Hope（无蒸馏）。
- Metric：增量准确率（Figure 3）。Hope+Sleep 在三数据集上均最优。

**长上下文理解**：
- 数据：LongHealth、QASPER、MK-NIAH (RULER)。
- Baseline：ICL、DuoAttention、Cartridges。
- Metric：QA 准确率 / 多键检索准确率（Figure 4）。Sleep 超过所有 baseline；增加层级数单调提升性能。

**BABILong**：
- Baseline：GPT-4、GPT-4o-mini、Llama-8B+RAG、RMT、ARMT、Titans。
- Metric：跨 token 长度（至 10M）的准确率。Hope 接近满分。

**数学推理**：
- 数据：AIME-24、AIME-25、HMMT-25。
- Backbone：Qwen3-1.7B、Qwen3-8B。
- Baseline：Base、SFT、GRPO、OPSD。
- Metric：average@16。Sleep 在 1.7B 和 8B 上均最优（1.7B: 53.2/40.2/29.3 vs GRPO 51.0/38.6/26.1）。

**知识整合**：
- 数据：SQuAD（单段落 n=1 和持续预训练 n=200）。
- Baseline：Base、Fine-tuned no dreaming、SEAL、Sleep（Transformer 两层/四层）。
- Metric：no-context 准确率。四层 Sleep 最优（n=1: 48.9, n=200: 46.2）。

**Few-shot 抽象推理**：
- 数据：ARC（11 训练任务 + 8 held-out）。
- Backbone：Llama-3.2-1B。
- Baseline：ICL(0%)、TTT(10%)、SEAL(72.5%)。
- Metric：成功率。Sleep 达 80%。

**新语言学习**：
- 数据：MTOB + Manchu（Kalamang 和 Manchu 翻译）。
- Baseline：ICL、Hope-1/2/3、Cartridges、SFT。
- Metric：ChRF 分数。Hope-3 在持续学习设定下几乎恢复单语言性能；ICL 严重退化。

## Weaknesses

- 所有实验的 baseline 中缺少与主流持续学习方法（如 Progressive Neural Networks、PackNet、LwF）的对比；EWC 是 2017 年的方法，InCA 是唯一的 2025 年对比，但未涵盖近年持续学习代表性工作。
- Dreaming 阶段计算代价未量化：每个 dream 需独立 SFT 实例（LoRA），且需梯度计算做重要性筛选——论文未报告 wall-clock 时间或计算量与 baseline 的对比，无法判断实际可行性。
- Figure 6 (BABILong) 混合了 zero-shot 大模型评测与 fine-tuned 小模型评测，blue（zero-shot）与 red（fine-tuned）点的比较存在公平性问题——不同模型规模和不同评测条件下直接比较可能产生误导。
- 语义奖励模型 r_sem 的细节未公开（使用什么模型、训练数据、判断阈值），而该奖励对 LTI 阶段质量有关键影响——不可复现。
- 参数 mask 实现方式要求预分配最大参数量，论文称"与大脑固定容量一致"，但未讨论预设容量上限对长生命周期场景的限制——若超过预分配容量，范式失效。
- 消融实验（Table 1, 3）仅在数学推理和 SQuAD 上进行，未在持续学习和长上下文任务上验证各组件贡献——泛化性不确定。
- 论文声明其方法"更鲁棒于灾难性遗忘"，但在新语言学习实验中 Cartridges 和 SFT "在至少一种语言上"出现 CF——论文将这些结果置于图外而非系统分析，存在选择性报告倾向。

## Relations

- builds-on 08_nested_learning [high]: Sleep 范式直接建立在 Behrouz et al. (2025) 的 Nested Learning 和 Hope 架构之上，使用 CMS 的多频率 MLP 链作为基础设施，并在其在线巩固之上补充离线巩固阶段 [1: §1, §2.2]。
- extends SEAL (Zweiger et al. 2025) [high]: Dreaming 阶段显式构建于 SEAL 框架之上，使用其 SFT+RL 自我改进循环和 ReSTEM 优化，但增加了梯度筛选、随机专家选择和记忆巩固前置步骤 [1: §3.4]。
- competes-with Cartridges (Eyuboglu et al. 2025) [med]: 两者都解决长上下文效率问题——Cartridges 用辅助模型压缩 KV 表示，Sleep 用 on-policy 自蒸馏巩固为参数化知识；论文在长上下文和新语言任务上直接比较并声称 Sleep 更优 [1: §4.1]。
- orthogonal Titans (Behrouz et al. 2024) [med]: Titans 是测试时记忆机制（RNN 式 hidden state 更新），与 Sleep 的参数级巩固正交——两者可潜在组合，论文在 BABILong 上将 Titans 作为 baseline 比较 [1: §4.1]。
- extends GKD (Agarwal et al. 2024) [high]: Knowledge Seeding 的 on-policy 蒸馏目标直接基于 GKD 框架，在其混合 teacher/student 生成数据的基础上增加了 RL 模仿学习组件 [1: §3.3]。

## Takeaway

- **值得借鉴的方法论**：向上蒸馏（小模型→大模型）是对常规蒸馏方向的逆转，在"学生容量大于教师"的持续学习场景下有独特价值；compute-consolidate-update 三步协议确保知识转移在参数更新前完成，避免信息丢失。
- **需要审慎的部分**：Dreaming 的计算开销（每 dream 独立 SFT）未被量化，实际部署可行性存疑；语义奖励模型细节缺失影响可复现性；BABILong 实验中 zero-shot 与 fine-tuned 混合比较需谨慎解读。
- **一句话记忆**：Sleep = 先巩固（向上蒸馏到新激活的低秩参数）再做梦（RL 驱动自生成数据自我改进），两步顺序是抵抗灾难性遗忘的关键设计。
