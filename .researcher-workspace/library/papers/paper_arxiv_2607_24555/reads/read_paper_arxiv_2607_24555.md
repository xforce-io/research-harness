---
title: "LOCKS: Page-Local Compact Key Summaries for Efficient Long-Context Decoding"
authors: ["Junsung Hwang"]
paper_id: "paper_arxiv_2607_24555"
source_kind: "arxiv"
source_id: "arxiv:2607.24555"
source_url: "https://arxiv.org/abs/2607.24555"
pdf_url: "https://arxiv.org/pdf/2607.24555"
read_id: "read_paper_arxiv_2607_24555"
kind: library-read
doc_type: "paper"
tags: []
---

# LOCKS: Page-Local Compact Key Summaries for Efficient Long-Context Decoding

> 旧做法用全局/序列级低秩基近似 KV cache 做稀疏注意力，丢失了页面特异性方向；本文给每个页面算自己的谱摘要，仅据此选 top-k 页面做注意力，在 100K+ 上下文匹配 FullKV 质量的同时将 decode 延迟减半。

## Essence

**问题**：长上下文推理瓶颈在 KV cache 的读取带宽——每步 decode 要全量读 KV。已有稀疏注意力选择器依赖全局或序列级低秩基（ShadowKV、Loki），或包络统计（Quest），但共享基对页面内容存在结构性盲区（Prop. 1 证明），包络保留总质量却丢失关键 carrier 页面。

**做法**：不读任何候选 key/value，仅从常驻摘要选页。每个 16-token 页面完成时做一次小特征分解（B×B Gram），保留 rank-8 基 + 系数 + 质心（int4/int8 量化，约 KV 字节的 9.4%）。decode 时重建页内 logits、用 log-sum-exp 估计每页注意力质量、跨 GQA 组做 share-average 排序，选 top-k 页面。整个 batched decode 步跑在 CUDA graph 里。

**证据**：在 1M token 上下文，LOCKS 以 b=2048 预算（约 2% token）匹配 FullKV 聚合质量，per-token decode 延迟降至 dense attention 的 2.0×，KV 读取量降 9.8×。

**边界**：保留全量 KV cache 在显存（不省显存），只省读取带宽；量化摘要的保留保证是经验测量而非逐步有界证明；页大小固定 16。

## Claims

1. 注意力 key 局部低秩而全局高秩：页面自身的 rank-8 特征基捕获大部分页内 key 能量，而序列级基（ShadowKV）捕获很小一部分，全局基（Loki）几乎为零 [§3, Table D.11]。
2. 任何固定共享低秩线性投影对页面内容存在盲方向：Prop. 1 证明，对于任意 W∈R^{d×ρ}（ρ<d），存在一族页面内容差异使共享基评分完全相同而精确 log-mass 无界分离 [§3, Prop. 1]。
3. 包络统计（Quest）在小预算下保留总质量但丢失 carrier 页面：在 0.5% 预算时 carrier retention 仅 39%（Llama-3.1-8B），而 LOCKS r8i4 达 93% [Fig. 1b, Table B.1]。
4. 精确 log-sum-exp 页面质量排序是 value-blind 选择中的 minimax 最优规则（Lemma 1），但计算它需读全部 key；LOCKS 从摘要重建 logits 近似该排序，无需读候选 key/value [§2, §4]。
5. Thm. 1 对未量化摘要给出从谱尾到 attention-output 误差的组合保留保证：group-average output error ≤ 2R(1−e^{−4η}γ̄*)，其中 η 由页面谱尾控制 [§4, App. A.3]。
6. int4 量化基相比未量化基在 recall 上损失数个百分点但 carrier retention 几乎不变；int2 在所有 rank 下退化为质心级排序 [§5.4, Fig. 5]。
7. 在 LongBench-v1 上 LOCKS 在各预算下均比 Quest、KVzip、ShadowKV、RocketKV 更接近 FullKV，差距随预算缩小而拉大 [§5.1, Fig. 2a]。
8. 在 RULER-16K/32K 上 LOCKS 在最小预算（b=64/128）下追踪 exact-LSE oracle，而四个基线选择器在低预算时崩塌 [§5.1, Fig. 2b,c]。
9. 在长链推理（AIME26, MATH-500）上基线选择器（Quest、R-KV、TriAttention、LazyEviction）跨预算均落后，LOCKS 追踪 oracle；在大预算下 AIME26 上 LOCKS 名义上匹配或超过 oracle 和 FullKV [§5.2, Fig. 3]。
10. 在 1M token 上下文（InfiniteBench, GLM-4-9B-Chat-1M），b=2048 时 LOCKS 以 43.6 平均分在可部署选择器中最高，超过 RocketKV（42.3）和 FullKV（43.0）[§5.1, Table 1]。
11. 在 H200 NVL 上，b=2048 时 LOCKS 的 per-token decode 延迟在 1M 上下文降至 dense baseline（FA3/FlashInfer）的 2.0×，KV 读取量降 9.8×；批量 decode 加速随 batch 和上下文增长，256K/batch=4 达 1.80× [§5.3, Fig. 4, Table 2]。

## Assumptions

- 页面大小固定 B=16，rank 固定 r=8——选择基于此参数化，未探索其他页面尺寸的端到端质量。
- GQA 组内共享选择（share-average）替代每头独立选择，牺牲最差头覆盖以换取统一 KV 读取——假设组内头注意力分布相似。
- 全量 KV cache 常驻显存——LOCKS 只省读取带宽，不省显存容量；论文明确声明减少显存占用是正交方向。
- 摘要的量化保留保证是经验测量（Table B.2）而非逐步解析界——Thm. 1 仅覆盖未量化摘要。
- 选择目标为 value-blind mass-ranking（Lemma 1）——不考虑 value 信息在选择阶段的使用（VATP 等方向被排除）。
- 页面谱集中性是 key 本身的性质，不依赖 RoPE 是否施加——论文测量了两种情况，但假设模型架构不会破坏此结构。

## Method

**与已有方法的对比**：

| 维度 | Quest (包络) | ShadowKV (序列级基) | Loki (全局基) | **LOCKS (页级基)** |
|------|------------|-------------------|-------------|-------------------|
| 表示范围 | 页级 min/max | 序列级低秩 | 全局低秩 | **页级 rank-8 谱基** |
| 选择读 key? | 是（读候选页） | 否 | 否 | **否** |
| 内容盲区 | 极端值相同即无法区分 | Prop. 1 盲方向 | Prop. 1 盲方向 | **无（每页独立基）** |
| 训练需求 | 无 | 无 | 离线校准 | **无** |

**构建（每页完成时一次）**：
1. 页面 key 中心化：δ^i = k_i − μ_j
2. 堆叠偏差为 D_j ∈ R^{B×d}，计算小 Gram 矩阵 D_j D_j^T（B×B）
3. 特征分解，保留 top-r 奇异值/向量：V_j = [v_{j,1}···v_{j,r}]，系数 c_i = V_j^T δ^i
4. 量化：基存 int4，系数和质心存 int8（带 per-column/per-row scale）
5. 逻辑载荷约 KV 字节的 9.4%（r=8, B=16）

**打分（每步 decode）**：
1. 对每页重建 B 个页内 logits：q^Tμ_j + (V_j^T q)^T c_i
2. log-sum-exp 归约得页面估计质量 ŝ_j(q)（式 1）
3. 组内归一化为 mass share，跨 GQA 头做 share-average：m̄_j = (1/G)Σ_g m̂_g(j)
4. 排序选 top-(k−2) 页面（sink + recent 占 2 槽），所有头共享一张选择表
5. 仅读取选中页面的 KV 做 attention——选择本身不读候选 key/value

**部署**：作为 vLLM 0.24 的 pip-installable 插件加载，无需 fork/patch；r、B、b 固定使所有 kernel shape 固定，整个 batched decode 步在 CUDA graph 内回放。

## Eval

**测量内容**：固定 per-head token budget b ∈ {64, ..., 2048} 下的任务质量和 decode 效率。

**基线**：FullKV（全量 attention 参考）、exact-LSE oracle（读全部 key 的精确选择上界）、Quest、KVzip、ShadowKV、RocketKV（检索/QA）；R-KV、TriAttention、LazyEviction（推理）。

**数据集**：LongBench-v1（14 子集，~11K 上下文）、RULER-16K/32K（13 任务）、InfiniteBench（100K+，GLM-4-9B-Chat-1M）、AIME26 + MATH-500（Qwen3-4B, thinking on）。

**模型**：Llama-3.1-8B（检索/QA + 显微镜）、Qwen3-4B（推理 + 显微镜）、GLM-4-9B-Chat-1M（长上下文 + 效率）。

**指标**：任务分数（各 benchmark 原生指标）、carrier retention（条件于 exact ranking 保留 carrier 的比例）、mass/recall（vs exact-LSE）、per-token decode latency、KV bytes read/step、throughput speedup（H200 NVL, vLLM 0.24）。

**关键结果**：LongBench-v1 全预算追踪 oracle；RULER-16K b=256 时 LOCKS 追踪 FullKV 而 Quest 等崩塌（Fig. 1d）；1M 上下文 b=2048 匹配 FullKV 聚合质量且 2.0× 加速；批量 256K/batch=4 达 1.80× throughput speedup。

## Weaknesses

- Thm. 1 的保留保证仅覆盖未量化摘要；部署的 int4/int8 摘要依赖经验测量（Table B.2），且 max error 极大（Llama 上 |ŝ−s| max=10.87），说明尾部页面的最坏情况保证是定性的而非定量的——论文承认但未在评测中隔离这些尾部页面对任务的下游影响。
- 全量 KV cache 必须常驻显存（"keeps the full cache resident"），意味着显存容量瓶颈未解决；在 1M 上下文，这仍是部署门槛——论文将其留给"future tiering or offload"但未评估任何分层方案的开销。
- 显微镜追踪（App. B）仅覆盖 RULER-32K 的 5 条记录/模型，且全部正确回答——carrier 页面分析建立在这极小样本上，未覆盖错误案例中选择失败的统计特性。
- 效率测量（Fig. 4, Table 2）仅在 GLM-4-9B-Chat-1M 上进行，未报告 Llama-3.1-8B 或 Qwen3-4B 的端到端延迟——不同模型的 KV 大小/GQA 结构可能改变加速比。
- 与 Prism [57]（同为谱选择块）和 SPLA [56] 的比较仅在 Related Work 中提及为 concurrent work，未在评测中对齐——论文的 "empirically strongest among evaluated deployable summaries" 声明不含这些谱方法。
- Table 2 中 32K/batch=1 的 speedup 为 0.96（LOCKS 比 dense 更慢），说明短上下文低并发时选择开销抵消了读取节省——但论文未给出 break-even 点（上下文 × batch 的等值线），使部署者难以判断适用边界。
- 推理实验（Fig. 3）中 LOCKS 在大预算下"nominally matches or exceeds both the oracle and FullKV on AIME26"——超过 oracle 和 FullKV 这一结果缺乏置信区间或统计检验，且与"value-blind 选择上界"的定位矛盾，可能是噪声或采样设置差异。

## Relations

- competes-with Quest [52] [high]: 同为 query-aware 稀疏注意力选择器、保留全量 cache，但 LOCKS 用页级谱摘要替代 Quest 的包络统计；论文在 Fig. 1b 和 Table B.1 直接展示 carrier retention 差距（93% vs 39% at 0.5% budget）。
- competes-with ShadowKV [51] [high]: 同为低秩 KV 压缩、training-free，但 ShadowKV 用序列级基而 LOCKS 用页级基；论文在 §3 和 App. D 的 matched-bytes 前沿表直接对比，页级基在相同字节数下 recall 更高。
- competes-with Loki [49] [high]: 同为低秩 key 压缩，但 Loki 用全局离线校准基而 LOCKS 用每页在线基；Prop. 1 直接针对 Loki 的共享投影类，Table D.11 实证对比。
- builds-on Tzachristas et al. [55] [high]: 论文采用其 top-k attention truncation 的精确误差恒等式作为选择目标（Lemma 1），明确声明"we claim neither as a contribution"。
- extends TokenSelect [58] [med]: LOCKS 的 share-average combine 的归一化形式即 TokenSelect 的 head soft vote，但 LOCKS 贡献了最优性证明（Cor. 3）和 worst-head 测量——§4 明确引用。
- competes-with COBS [54] [high]: 同为 block-level mass 估计、concurrent work，但 COBS 用二阶矩截断而 LOCKS 用完整 logit 重建；Prop. 3 分析 COBS 的二阶截断在 peaky carrier 页面上 worst-case 控制最弱。
- orthogonal KVQuant [23] / ZipCache [20] [med]: 量化全量 KV 的每条目比特数，与 LOCKS 选择哪些页面读取正交；论文 §6 明确归类为"orthogonal axis"。
- competes-with Prism [57] [med]: 同为谱方法选择块、concurrent work，但 Prism 在块级做谱选择而 LOCKS 在页级做完整 logit 重建加 LSE；论文仅在 Related Work 提及，未实验对齐 [low]。

## Takeaway

- 方法论上最值得借鉴的是"表示范围决定选择保真度"这一洞察：匹配字节数后页级基仍优于序列/全局基，且共享投影的内容盲区是结构性而非定量退化——这对任何用低秩近似做 KV 压缩的工作都有约束力。
- 部署友好的设计模式值得参考：固定 shape 使全 decode 步跑 CUDA graph、pip-installable 插件不 fork vLLM、摘要构建在 prefill–decode 边界批量完成、增量构建在 decode 关键路径外。
- 需要警惕的：量化摘要的最坏情况误差极大（max ~17），保留保证是定性的；效率 break-even 点未给出，短上下文低并发时 LOCKS 可能更慢；推理基准上"超过 oracle"的结果缺乏统计支撑。
- 如果只记一件事：**每个页面自己的 rank-8 谱基是 KV 读取带宽瓶颈下做稀疏选择的正确粒度——全局/序列级基有结构性盲区，包络统计丢失 carrier 页面，页级谱基两者都避免。**
