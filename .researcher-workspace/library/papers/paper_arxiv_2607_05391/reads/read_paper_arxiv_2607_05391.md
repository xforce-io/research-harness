---
title: "LLM-as-a-Verifier: A General-Purpose Verification Framework"
authors: ["Jacky Kwok","Shulu Li","Pranav Atreya","Yuejiang Liu","Yixing Jiang","Chelsea Finn","Marco Pavone","Ion Stoica","Azalia Mirhoseini"]
paper_id: "paper_arxiv_2607_05391"
source_kind: "arxiv"
source_id: "arxiv:2607.05391"
source_url: "https://arxiv.org/abs/2607.05391"
pdf_url: "https://arxiv.org/pdf/2607.05391"
read_id: "read_paper_arxiv_2607_05391"
kind: library-read
doc_type: "paper"
tags: []
---

# LLM-as-a-Verifier: A General-Purpose Verification Framework

> 标准 LM judge 吐一个离散整数分导致高频平局；本文改为读取打分 token 的完整 logprob 分布算期望，得到连续分数，并将验证沿粒度/重复/分解三个轴 scaling。

## Essence

**问题：** 标准 LLM-as-a-Judge 取 argmax token 作为离散分数，复杂轨迹比较时平局率高达 27%，无法区分好坏方案 [§3.1]。

**做法：** 不改训练，仅改解码方式——提取评分 token 的 top-G logprob，按期望值聚合为连续分数 $R(x,\tau)=\frac{1}{CK}\sum_{c,k}\sum_g p_\theta(v_g|\cdot)\,\varphi(v_g)$；再通过 Bradley-Terry 转偏好概率，用 Probabilistic Pivot Tournament（O(Nk) 而非 O(N²)）选最优轨迹 [§3.2]。

**证据：** 在 Terminal-Bench V2 上，离散 judge 的 tie rate 为 27%，连续 verifier 降至 0%；pairwise 准确率从 73.1%（G=1）提升至 77.5%（G=20）[Fig. 4, Fig. 7]。

**边界：** 核心机制依赖模型暴露 token-level logprobs；对不开放 logprob 的前沿模型需额外两阶段 workaround [§4, Appendix B.6]。

## Claims

1. 将评分从 argmax 离散 token 改为对评分 token 分布取期望，可将 tie rate 从 27% 降至 0% [§3.1, Fig. 7]。
2. 增加评分粒度 G（从 1 到 20）使信噪比 SNR 从 0.775 升至 0.799，pairwise 准确率从 73.1% 升至 77.5% [§4.1, Table 1]。
3. 增加重复评估次数 K（从 1 到 16）将准确率从 74.7% 提升至 77.5%；单次 verifier（K=1）已匹配 K=16 的离散 judge [§4.2, Fig. 7]。
4. 将单一标准分解为 Specification / Output / Errors 三个子标准并 ensemble，准确率从任一单标准的 75.2–76.4% 升至 78.3% [§4.3, Fig. 4]。
5. Probabilistic Pivot Tournament 将候选排序成本从 O(N²) 降至 O(Nk)，通过随机哈密顿环消除位置偏差 [§3.2, Fig. 6]。
6. 在四个 benchmark 上达到 SOTA：Terminal-Bench V2 86.5%、SWE-Bench Verified 78.2%、RoboRewardBench 87.4%、MedAgentBench 73.3% [§5, Fig. 1]。
7. verifier 连续分数与 agent 任务进度强相关：成功轨迹的 Spearman VOC=0.848，失败轨迹 VOC=0.769（Terminal-Bench V2，500 对）[§6, Table 6]。
8. 作为 RL dense reward，在 LIBERO 上将 π₀+DSRL-SAC 的 sample efficiency 提升 ~1.8×，最终成功率 0.76 vs. 0.69 [§7, Fig. 9]。

## Assumptions

- 生成模型在多次采样中至少能产出一次正确解（oracle Pass@K 假设成立）；这是 verification 有意义的前提 [§3.1, Fig. 5]。
- 验证器模型（Gemini 2.5 Flash / Qwen 3.6 35B）的 token logprob 分布与其内部"信念"成正比，即分布是校准的概率表示 [§4.1]。
- 三个分解标准（Specification / Output / Errors）对所有领域（编码/机器人/医疗）均适用，论文在四个 benchmark 上使用了同一套分解策略 [§5]。
- 候选轨迹池质量上限由 oracle Pass@N 决定，verifier 只负责在已有候选中选择而非改进候选 [§3.1]。

## Method

**与传统 judge 的关键对比：**

| | 传统 LM Judge | LLM-as-a-Verifier |
|---|---|---|
| 评分方式 | argmax token → 离散整数 | 对 top-G token logprob 取期望 → 连续分数 |
| 平局率 | 27%（Terminal-Bench） | 0% |
| 排序 | 全配对 O(N²) | Pivot Tournament O(Nk) |

**输入：** 任务 prompt x、两条候选轨迹 τᵢ, τⱼ、评估标准 c、评分 token 集 V={v₁,...,v_G}。

**核心计算：**
1. 构造 pairwise scoring prompt，要求模型在 `<score_A>` / `<score_B>` 标签中输出 1–20 的**字母**评分（用字母而非数字以便提取 logprob）。
2. 提取 top-G 个评分 token 的 logprob，计算期望奖励：$R(x,\tau)=\frac{1}{CK}\sum_{c=1}^{C}\sum_{k=1}^{K}\sum_{g=1}^{G} p_\theta(v_g|x,c,\tau)\,\varphi(v_g)$，其中 C=标准数、K=重复次数、G=粒度。
3. 归一化 R∈[0,1]，用 Bradley-Terry 模型转换为偏好概率：$P(\tau_i\succ\tau_j)=\sigma(R(\tau_i)-R(\tau_j))$。

**候选排序（Probabilistic Pivot Tournament）：**
- Ring pass：随机哈密顿环评分 N 个相邻对，每个候选恰好出现一次在 A 槽、一次在 B 槽，消除位置偏差。
- Pivot selection：按 ring pass 均分排序，取 top-k 作为 pivot。
- Pivot rounds：所有非-pivot vs pivot + pivot vs pivot 对评分，聚合为 win mass wᵢ，选 argmax wᵢ/cᵢ。总比较数 O(Nk)。

**输出：** 最优轨迹索引 i* 及每条轨迹的连续分数（可用于进度追踪或 RL reward）。

## Eval

**Benchmark 结果（§5）：**

| Benchmark | Pass@1 | Oracle Pass@N | LLM-as-a-Verifier | 基线最佳 |
|---|---|---|---|---|
| Terminal-Bench V2 | 83.1% (GPT-5.5) | 92.1% (Pass@5) | **86.5%** | 84.7% |
| SWE-Bench Verified | 76.1% (pool mean) | 84.4% (Pass@3) | **78.2%** | 76.8% (Opus 4.5) |
| RoboRewardBench | — | — | **87.4%** | 81.4% (RoboReward-8B) |
| MedAgentBench | 70.2% (Opus 4.8) | 75.0% (Pass@5) | **73.3%** | 70.2% |

- **验证器：** Gemini 2.5 Flash（编码/医疗），Qwen 3.6 35B（机器人，VLM）。
- **Scaling 消融（§4）：** Terminal-Bench V2 上 200 条随机轨迹的 pairwise accuracy；G: 1→20 给出 73.1%→77.5%；K: 1→16 给出 74.7%→77.5%；C: 1→3 给出 75.2-76.4%→78.3%。
- **Tie rate 对比（Fig. 7）：** Judge 26.7%（K=1）→5.5%（K=16）；Verifier 全部为 0%。
- **RL（§7）：** LIBERO ketchup 任务，π₀+DSRL-SAC，n=5 seeds；MATH+GRPO，Qwen3-8B，n=3 seeds。
- **进度追踪（§6）：** 500 条轨迹的 Spearman VOC，成功=0.848±0.012，失败=0.769±0.016。

## Weaknesses

1. **对 logprob 访问的硬依赖被软处理。** 核心公式依赖 top-G token logprob，但论文仅在 Appendix B.6 简略提及"两阶段 workaround"适用于不开放 logprob 的前沿模型，未给出该 workaround 的准确率对比或误差分析——主实验全部使用 logprob-accessible 模型，实际部署适用性未充分验证 [§4, Appendix B.6]。
2. **Scaling 消融仅在单一 benchmark（Terminal-Bench V2）上完成。** 三个 scaling 轴的增益曲线（Fig. 4）全部来自 Terminal-Bench 的 200 条轨迹采样，未验证该 scaling 规律是否在 SWE-Bench / MedAgentBench / RoboRewardBench 上同样成立。
3. **验证器与生成器的模型选择缺乏系统消融。** 主实验用 Gemini 2.5 Flash 作验证器、GPT-5.5/Opus 4.5 等作生成器，但未测试验证器模型能力对结果的影响——是否弱模型也能当好验证器？更换验证器后 SOTA 是否保持？
4. **SWE-Bench 使用异构候选池但其他 benchmark 用同构池。** SWE-Bench 从三个不同模型族各采一条轨迹（N=3），而 Terminal-Bench 从单一模型采 N=5。候选池构造方式不同使跨 benchmark 的比较不一致，异构池本身可能引入验证器偏好某些模型族的混淆因素 [§5.1-5.2]。
5. **RL 增益规模不对称且未报告方差细节。** LIBERO 上 1.8× 的增益在 success rate 0.2–0.6 区间测量，但最终成功率（0.76 vs 0.69）的方差和显著性未报告；MATH 上仅 1.1× 的增益在 n=3 seeds 下统计可靠性存疑 [§7, Fig. 9]。
6. **Pivot Tournament 的 k 值选择缺乏指导。** 论文展示 PPT 在不同 k 值下的性能（Table 9），但未给出 k 的选择策略—— practitioner 如何根据 N 和预算确定 k？

## Relations

- `builds-on` LLM-as-a-Judge (Zheng et al. 2023) [high]: 本文明确将标准 LM Judge 的离散评分方式作为改进基线，核心创新是将 argmax 评分替换为 logprob 期望 [§3.1, Ref 4]。
- `extends` Generative Verifiers (Zhang et al. 2025) [med]: 两者都将 reward modeling 视为 next-token prediction 并利用 token 概率；本文在此基础上增加三轴 scaling 框架和 pivot tournament 排序算法 [§9, Ref 6]。
- `competes-with` V1 (Singh et al. 2026) [high]: 论文在 Table 9 中直接与 V1 的排序方法对比，PPT 在更少比较次数下优于 V1 [§3.2, Ref 5]。
- `orthogonal` Process Reward Models / Outcome Reward Models [med]: PRM/ORM 需要训练数据且关注步骤级或结果级信号；本文是 training-free 的轨迹级验证，与 PRM/ORM 互补而非直接竞争 [§9]。
- `extends` TOPReward (Chen et al. 2026) [med]: TOPReward 用 token 概率作为机器人零样本 reward；本文将概率化思路推广到更多模态和更长 horizon 的 agentic 任务 [§5.3, Ref 10]。

## Takeaway

- **方法可移植性强：** 核心改动仅在解码层（取期望而非 argmax），不改模型权重、不需训练，任何能暴露 logprob 的模型即可使用。
- **需复验的关键点：** 三个 scaling 轴的增益全部在 Terminal-Bench V2 上测量，迁移到其他任务前应先验证 scaling 曲线是否一致；对不开放 logprob 的前沿模型，"两阶段 workaround"的实际效果未充分报告。
- **部署注意：** 候选池大小 N 和 pivot 数 k 直接影响验证成本（O(Nk)），但论文未给出 k 的选择规则，需自行在成本-准确率曲线上标定。
- **一句话记忆：** 不要让 judge 吐整数——读它的概率分布取期望，平局就消失了，验证就能 scale。
