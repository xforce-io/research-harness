---
title: "SmartGen: Seamless Disaggregated LLM Inference with Selective KV Cache Transfer"
authors: ["Xuchuan Luo","Jiacheng Shen","Xin Wang","Yangfan Zhou"]
paper_id: "paper_arxiv_2607_28150"
source_kind: "arxiv"
source_id: "arxiv:2607.28150"
source_url: "https://arxiv.org/abs/2607.28150"
pdf_url: "https://arxiv.org/pdf/2607.28150"
read_id: "read_paper_arxiv_2607_28150"
kind: library-read
doc_type: "paper"
tags: []
---

# SmartGen: Seamless Disaggregated LLM Inference with Selective KV Cache Transfer

> Prefill/Decoding 分离推理中全量 KV cache 传输饱和网络带宽 -> 按重要性分三类选择性传输 KV，仅预推关键块、按需并行拉取缺失块、投机补全剩余块 -> TTST 降低 4.3× 且不损精度。

## Essence

**问题：** P/D 分离架构下，KV cache 从 prefill 节点传到 decoding 节点的时间可达 prefill 计算时间的 6.5×（48K tokens, 25 Gbps RDMA），导致用户看到第二个 token 前出现"stage-transition stall"。

**做法：** 不是压缩 KV cache（如量化），而是利用 KV cache 的动态稀疏性——只传输解码阶段真正被 attention 选中的 KV 条目。三条路径：(1) 离线 profiling 找出"跨 prompt 普遍重要"的 KV block，prefill 阶段提前 RDMA WRITE 推送到 decoding 节点；(2) 解码时将 KV index 拆分为 local/remote，local 走 PCIe、remote 走 RDMA 并行拉取，网络延迟与本地加载重叠；(3) attention 计算期间网络空闲时投机传输剩余 KV block，10 轮内全部传完。

**证据：** 相比全量传输，TTST 降低最高 4.3×（GovReport），TBT 接近全量传输的理想值，LongBench 精度保持 98%-103%。

**边界：** 依赖 RDMA 网络（GDR 可选但有回退）；profiling 假设 KV 重要性分布在 4K-16K 范围内有位置稳定性；低带宽（15 Gbps）下 TTST 反而超过理想值因前两层 KV 传输已饱和带宽。

## Claims

- 在 25 Gbps RDMA 网络下，48K tokens 的 KV cache 传输耗时是 prefill 计算的 6.5×，KV 传输可占 job completion time 的 42.2% [§1, §3.1]。
- 重要 KV 条目在相同模型的不同 prompt 长度（4K-16K）下呈现一致的位置分布（positional similarity），因此离线 calibration 可近似在线请求的 KV 重要性 [§4.1, Figure 5]。
- Profile-based proactive transfer 的 on-demand ratio 比随机选择降低 51%，比顺序选择降低 38%，TBT 加速最高 1.3× [§5.3, Figure 15]。
- Parallel on-demand transfer 通过 mask 矩阵拆分 KV index 为 local/remote 两路并行加载，将网络往返从关键路径移除，TBT 降低 1.1-1.2× [§4.2, §5.3]。
- Speculative transfer 利用 attention 计算期间的空闲网络/CPU 资源，以 10% speculative ratio 在 10 轮内传完全部剩余 KV block，TBT 进一步降低 1.2-1.3× [§4.3, §5.3]。
- SmartGen 在 Llama-3.1-8B、Qwen3-8B/14B、Gemma-3-12B、Phi-4-14B 上的 LongBench 精度保持 98%-103%（相对 full-cache baseline），而 HACK（2-bit 量化）精度仅 13%-77% [§5.5, Figure 9/17]。
- TTST 相比全量传输降低最高 4.3×，随网络带宽从 32 Gbps 降至 15 Gbps，优势从 2.5× 扩大到 3.3× [§5.2, Figure 12]。
- SmartGen 与 InfiniGen 和 HATA 两种 KV 选择算法均兼容，验证了设计的通用性 [§5.2]。

## Assumptions

- Prefix cache hit ratio 假设为 75%（论文取 60%-90% 报告范围的代表值），影响 prefill 性能但不是 SmartGen 的核心机制 [§3.1]。
- KV 重要性的位置分布在 4K-16K prompt 长度范围内稳定；超出此范围的 prompt 未验证 [§4.1]。
- 输出长度固定为 64 tokens 用于评测；更长输出下的 speculative transfer 完成时机与 TBT 趋势未报告 [§5.1]。
- RDMA NIC 的 scatter-gather 上限（如 20）足以支撑 remote KV 的 row-wise 传输优化 [§4.2]。
- P/D 分离架构下 prefill 节点 batch size 小、decoding 节点 batch size 大，多 prefill 到单 decoding 的 fan-in 是典型部署 [§3.1]。

## Method

**对比：全量传输 vs SmartGen 三路径**

| | 全量传输 | SmartGen |
|---|---|---|
| Prefill 阶段 | 逐层传全部 KV | 仅传 top-Kr 重要 KV block |
| Decoding 阶段 | 全部本地 | local KV 走 PCIe + remote KV 走 RDMA 并行 |
| 后续轮次 | 无额外开销 | 投机传输剩余 KV，网络空闲时填充 |

**输入：** Prefill 节点的 KV cache（按 layer offload 到 CPU memory），离线 profiling 得到的重要性矩阵 I，prefill 时间 Tp 与传输时间 Tt。

**核心计算：**

1. **Profile-based proactive transfer：** 离线在校准数据集上运行推理，统计每个 KV block 被选中的频率，计算 importance score I_{l,m}（首两层恒为∞）。根据 Kr = L·M·(Tp/Tt)·((L-1)/L) 选出 top-Kr 个 block，prefill 阶段逐层 offload 时通过 RDMA WRITE 推送。同时通过 doorbell batching 的额外 WRITE 更新 decoding 端 int8 KV mask 矩阵。

2. **Parallel on-demand transfer：** 解码时 KV selection 算法生成 KV index（shape BH×k）。Algorithm 1 用 KV mask 将 index 拆分为 local_index（mask=1）和 remote_index（mask=2）。local 走 PCIe 从 host memory 加载，remote 通过 GDR SEND index 到 prefill 节点、prefill 节点 SEND 回 KV entries。利用 attention 对 KV 顺序不敏感的特性，对 mask 降序排序使 remote entries 连续，RDMA scatter-gather 批量写入。

3. **Speculative transfer：** 每层 attention 计算开始时 decoding 节点通知 prefill 节点网络空闲，prefill 节点按重要性顺序 RDMA WRITE 剩余 KV block，speculative ratio = 10%，10 轮内传完。

**输出：** Decoding 节点 GPU 上就绪的 KV cache，attention 计算可直接执行。

## Eval

- **指标：** TTST（KV 传输性能）、TBT（解码开销，排除 TTST）、CTL（用户感知延迟）、LongBench accuracy metrics（F1/ROUGE-L/Edit Sim.）。
- **Baseline：** Full transfer（SOTA 逐层全传）、Partial transfer（顺序选 Kr 块 + on-demand）、HACK（2-bit 同态量化）。
- **模型：** Llama-3.1-8B、Qwen3-8B、Qwen3-14B、Gemma-3-12B、Phi-4-14B。
- **数据：** LongBench（MultiFieldQA、GovReport、SAMSum、LCC），校准用 2WikiMultihopQA。
- **硬件：** 3× Alibaba ecs.gn8is-2x.8xlarge（2× L20, 32 Gbps）做 prefill，1× ecs.gn8is.4xlarge（1× L20, 25 Gbps）做 decoding；另测 CloudLab V100S 物理机。
- **关键结果：** TTST 降低 4.3×（GovReport）；TBT 接近 full transfer；精度 98%-103%；HACK 精度 13%-77%。Factor analysis 显示三个组件叠加贡献 TBT 降低 1.5×→1.03×→1.1×→1.3×（L20）。

## Weaknesses

- 评测输出长度仅 64 tokens，speculative transfer 的"10 轮内传完"假设短输出场景；长输出（数百 tokens）下 speculative 与 on-demand 的干扰模式未验证。
- Profile-based approach 的 positional similarity 仅在 4K-16K 验证，但实际长上下文场景（32K-128K）的 KV 重要性分布是否仍稳定未测试——论文标题暗示更广适用性但实验未覆盖。
- 假设 75% prefix cache hit ratio，但未报告不同 hit ratio 下 SmartGen 的性能变化；低 hit ratio 时 prefill 时间变长、可传输窗口增大，可能改变三条路径的相对贡献。
- 15 Gbps 带宽下 SmartGen 的 TTST 超过理想值，论文承认但未量化这个 overhead 的构成（前两层 KV + metadata tensors），也未提供缓解方案。
- 所有实验 DP=6、单 decoding 节点；多 decoding 节点的 fan-out 场景下 speculative transfer 的网络竞争未评估。
- KV selection ratio 上限设为 20%，但 Figure 16c 显示 60% 时 TTST 增长 1.8×——高稀疏度场景（需要更多 KV）的 scalability 未深入讨论。

## Relations

- builds-on InfiniGen [high]: SmartGen 直接采用 InfiniGen 作为默认 KV selection 算法，并复用其 prefetch 技术实现 attention 与 KV selection 的并行 [§4.3]。同时扩展 InfiniGen 的动态 KV 选择从单机 GPU memory offloading 场景到跨节点 RDMA 传输场景。
- builds-on HATA [high]: SmartGen 将 HATA 作为第二种 KV selection 后端验证通用性 [§5.2]。
- competes-with HACK [high]: 两者均解决 self-hosted disaggregated inference 的网络瓶颈，但 SmartGen 用稀疏性选择、HACK 用 2-bit 量化；在同一实验框架下 SmartGen 精度 98%-103% vs HACK 13%-77% [§5.5]。
- orthogonal-to KVQuant [med]: 论文明确指出量化方法与 selective transfer 正交，可叠加使用 [§6.2]。
- extends Mooncake [med]: Mooncake 依赖 800 Gbps 高带宽 RDMA 做 KV 传输，SmartGen 针对低带宽云实例优化同一传输环节，可集成进 Mooncake 架构 [§6.1]。

## Takeaway

- **方法论借鉴：** 离线 profiling + 在线自适应的分层传输策略——将"什么重要"的判断与"何时传输"的调度解耦，是处理网络受限场景的通用思路。
- **需复验：** Positional similarity 在 32K+ prompt 长度下是否成立——这是 profile-based 方法的根基，但实验仅到 16K。
- **注意区分：** SmartGen 不改变 KV selection 算法本身，精度保持来自"选择算法不变 + on-demand 补全"——profiling 只决定传输优先级，不决定最终使用的 KV 集合。
- **一句话：** 用 KV 稀疏性重写传输策略而非压缩 KV 本身，在低带宽云实例上把 stage-transition stall 从 6.5× 降到接近零。
