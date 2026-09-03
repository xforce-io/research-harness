---
title: "Can Language Models Actually Retrieve In-Context? Drowning in Documents at Million Token Scale"
authors: ["Siddharth Gollapudi","Nilesh Gupta","Prasann Singhal","Sewon Min"]
paper_id: "paper_arxiv_2607_01538"
source_kind: "arxiv"
source_id: "arxiv:2607.01538"
source_url: "https://arxiv.org/abs/2607.01538"
pdf_url: "https://arxiv.org/pdf/2607.01538"
read_id: "read_paper_arxiv_2607_01538"
kind: library-read
tags: []
---

# Can Language Models Actually Retrieve In-Context? Drowning in Documents at Million Token Scale

> 首个在百万 token 规模语料上对 LM 上下文检索的系统研究，揭示并缓解了 attention dilution 效应。

## Brief

本文首次系统研究了语言模型 (LM) 在百万 token 语料规模下进行上下文检索 (In-Context Retrieval, ICR) 的能力。作者提出 BlockSearch 模型，发现其虽能泛化至训练长度 10 倍的上下文，但在百万 token 规模下检索性能崩溃；通过机制分析将崩溃归因为 attention dilution 效应，并提出 SSMax 和文档级稀疏注意力两种缓解方法。最终模型在传统检索基准上匹配稠密检索性能，在需要不同相似性定义的 LIMIT 基准上则以 3 倍优势超越稠密检索。

## Claims

- BlockSearch 是一个 0.6B 参数的 LM 检索器，其架构和训练修改使其能够泛化至训练上下文长度 10 倍以上的规模 [§3.4]。
- 随着语料规模增长，即使 gold 文档的 pre-softmax 分数保持较高，无关文档主导 softmax 分母导致 gold 文档的归一化注意力质量崩溃，此为 attention dilution 效应 [§4]。
- 即使在 N=10,000 时，至少有一个注意力头仍能将 gold 文档排在首位 (R_any=1.00)，但所有头的聚合一致性 (R_sum) 崩溃至 0.01，表明检索信号在单头层面保留但在输出读取层面丢失 [§4.2, Figure 3]。
- 在 L19 层，注意力输出的总幅度仅缩小约 36%，但 gold 驱动的份额从 0.91 降至 0.01（约 130 倍），表明注意力输出从 gold-token 平均值被改写为幅度相近的非 gold-token 平均值 [§4.2, Table 1]。
- SSMax（将 pre-softmax 分数乘以 s·log N）在全部 N 扫描范围内有效，将 MS MARCO N=10k 的 Recall@1 从 0.2% 提升至 16.5% [§5.2, Table 2]。
- Top-B=256 文档级稀疏注意力在 MS MARCO N=10k 达到 18.8% Recall@1，接近稠密检索基线的 20.2% [§5.2, Table 2]。
- SSMax 与 routing 组合在 MS MARCO N=10k 达到 20.5% Recall@1，超越稠密基线；在 HotpotQA N=10k 达到 78.5%，超越 7 倍大的 MSA-4B (75.5%) [§5.2, Table 2]。
- 在需要词法相似性的 LIMIT 基准上，BlockSearch-SSMax-routing 在 N=5,000 时达到 14.9% Recall@1，而同骨干稠密检索器仅 3.5%，接近 3 倍优势 [§6, Table 3]。
- 尽管 MSA-4B 在 RULER NIAH 合成基准上表现近乎完美，但在真实检索任务上急剧退化，表明现有合成长上下文基准高估了真实检索能力 [§3.4]。

## Assumptions

- 文档级独立性假设：文档之间互相独立，因此可以使用 block-sparse attention 使文档 token 仅在自身块内进行因果注意力 [§3.2]。
- 每文档截断至 300 token 不会丢失对检索至关重要的信息 [§3.1]。
- 训练时 256 文档的语料规模足以让模型学习到可泛化至 10,000 文档的检索能力 [§3.2]。
- RLHN（用 LLM judge 剪枝假负例）提供的数据质量足以作为训练检索器的金标准 [§3.2]。
- 评估中每个查询使用 24 个 hard negative 构成的 10,000 文档语料，能够代表真实检索场景的难度 [§3.3]。
- 四位数字编码 (0–9999) 足以为最多 10,000 文档提供唯一标识且模型能有效学习该映射 [§3.1]。

## Method

BlockSearch 基于 Qwen3-0.6B 骨干，采用以下核心设计：

1. **Prompt 格式**：每个文档以 `<bos>Doc {code}: {text} (Doc {code})<eos>` 格式输入，四位数字 code 在每个训练步随机采样，打破 code 与位置/语义的关联；查询块附加在所有文档之后，位于 RoPE 位置 300 [§3.2, §A]。
2. **Block-sparse attention**：文档 token 仅因果注意自身块内，查询块注意整个语料；每个文档起始处重置 RoPE 位置 [§3.2]。
3. **In-batch negative 训练**：batch 中 b 个 (query, 16-doc) 元组的文档统一 prefill，每个 query 对共享语料评分，一次 prefill 产生 b 个训练信号 [§3.2]。
4. **On-policy auxiliary loss**：模型先无梯度 rollout 四位 code，再对 rollout 的每一步用 in-batch 文档分数构建 teacher 分布，有梯度重放四个答案位置并计算交叉熵，总损失 L = L_CE + λL_aux [§3.2, §F]。
5. **Length-aware softmax 修改**：(a) Additive sink 在 softmax 分母附加学习标量 b_L，使注意力分散时层输出被乘性抑制 [§5.1, Eq. 5, §H]；(b) SSMax 将 pre-softmax 分数乘以 s·log N，直接随 N 增大拉大 gold 与 distractor 的分数差距 [§5.1]。
6. **Document-level sparse attention (routing)**：在 L16 层对文档按 QK-MaxSim 评分，仅保留 top-B=256 文档参与 L17 起的密集注意力 [§5.1]。

## Eval

- **数据集**：MS MARCO (dev)、HotpotQA (test)、NQ (test)，各采样 400 queries，语料规模 N=500 至 10,000（NQ 上限 8,607），每个查询含 1 个 gold + 24 个 hard negatives，语料约 948k–1.2M tokens [§3.3, Table 4]。OOD 评估使用 LIMIT（词法相似性，50,000 篇传记，N=46 至 5,000）和 OBLIQ（类比/描述性检索）[§6, §L]。
- **基线**：Qwen3-dense-0.6B（同骨干、同 RLHN 数据训练的稠密检索器）；MSA-4B（7 倍参数、更长训练上下文的并发模型）；BlockSearch-position（顺序编码、无辅助损失，即先前 ICR 方案）；BlockSearch-offpolicy（移除 on-policy 损失）[§3.3]。
- **指标**：Recall@1（beam search 宽度 25、每步保留 top-5、返回 top-5 的四位 code 解码），辅以 Recall@5 [§3.3, §J]。
- **关键结果**：(a) BlockSearch 在 N≤2,500 时匹配或超越 MSA-4B（MS MARCO N=1,000：95.8% vs 93.8%），但在 N=10,000 崩溃至 0.2% [§3.4, Table 2]；(b) SSMax+routing 在 MS MARCO N=10k 达 20.5%（超越 dense 20.2%），HotpotQA N=10k 达 78.5%（超越 MSA 75.5%）[Table 2]；(c) LIMIT N=5,000 上 SSMax+routing 14.9% vs dense 3.5% [Table 3]；(d) OBLIQ 上所有方法绝对值极低（R@1 ≤ 0.008），注意力天花板 R_any 远高于实际输出，表明 attention dilution 仍是未解瓶颈 [§L, Table 15]。

## Weaknesses

- **OBLIQ 结果未在正文充分讨论**：Table 15 显示所有 BlockSearch 变体在 OBLIQ 三个任务上的 R@1 几乎为零（最高 0.008），远低于 R_any 天花板（0.852–0.942），但正文仅在附录 §L 简略提及，主结论声称"ICR 是稠密检索的可行替代"未对此严重失败进行限定 [§L]。
- **Routing 本质上重新引入了 retrieve-then-read 分解**：论文承认 routing 在模型内部重新引入了 ICR 旨在消除的两阶段结构 [§5.2]，但未将其视为对 ICR 核心主张的矛盾，仅以脚注式提及。
- **Sink 方法几乎无效却仍作为主要贡献呈现**：BlockSearch-sink 在所有数据集的大 N 区间仅产生微小改进（MS MARCO N=10k：0.2→2.5），在 LIMIT 上甚至不如基座模型，但论文仍将其与 SSMax 并列为"length-aware adjustments" [§5.2, Table 2, Table 3]。
- **训练-评估域差距未量化**：模型在语义检索数据 (RLHN/BEIR) 上训练，在词法检索 (LIMIT) 和类比检索 (OBLIQ) 上评估，但未报告在域内分布上 attention dilution 的严重程度是否与域外不同，难以判断 attention dilution 是任务固有还是分布偏移导致 [§4, §6]。
- **仅使用 0.6B 模型，未验证规模效应**：所有实验基于单一 0.6B 骨干，attention dilution 是否在更大模型上同样严重或有所缓解未知，但论文以"attention dilution 是根本瓶颈"作为普适结论 [§7]。
- **评估 query 数量有限**：每数据集仅 400 queries，LIMIT 1,000 queries，在 N=10,000 的 0.2% Recall@1 处置信区间极宽，但未报告任何显著性检验或误差棒 [§3.3, §6]。

## Relations

- builds-on 05_scalable_in_context_ranking [med]：BlockSearch 的 block-sparse attention 架构和 per-head MaxSim 评分直接源自 Gupta et al. [5] 的 in-context ranking 工作，论文 §2 明确引用并在此基础上添加随机编码和 on-policy 损失。
- competes-with 16_msa_4b [high]：BlockSearch 与 MSA-4B [16] 均学习从上下文语料生成文档引用，论文 §3.4 直接对比两者性能，BlockSearch 以 1/7 参数量在中小 N 匹配 MSA，但 MSA 在大 N 领先。
- extends 09_scalable_softmax [med]：SSMax 方法直接应用 Nakanishi [9] 的 Scalable-Softmax（pre-softmax 分数乘以 log N），将其从通用长上下文建模扩展到 ICR 的 attention dilution 缓解场景，论文 §5.1 明确引用。
- extends 26_streaming_llm_attention_sinks [med]：Additive sink 方法将 Streaming-LLM [26] 的 attention sink 从流式解码稳定性重新用于缓解大 N 下的 softmax dilution，论文 §2 和 §5.1 明确说明这一转向。
- contradicts 02_gemini_subsume_retrieval [med]：Lee et al. [2] 主张长上下文 LM 可以"subsume"检索/RAG/SQL 等，但本文 §1 和 §3.4 表明在真实百万 token 检索任务中 LM 崩溃至近零，且 MSA 在 RULER NIAH 上近乎完美但真实检索急剧退化，质疑了合成基准支持的主张。
- orthogonal 04_limit_embedding_limitations [med]：LIMIT 基准 [4] 证明嵌入检索存在理论局限，BlockSearch 在 LIMIT 上以 3 倍优势超越稠密检索，从不同角度（ICR vs 理论分析）支持同一结论，但两者方法正交。
