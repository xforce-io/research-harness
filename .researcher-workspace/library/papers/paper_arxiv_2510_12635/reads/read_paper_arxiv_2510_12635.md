---
title: "Memory as Action: Autonomous Context Curation for Long-Horizon Agentic Tasks"
authors: ["Yuxiang Zhang","Jiangming Shu","Ye Ma","Xueyuan Lin","Shangxi Wu","Jitao Sang"]
paper_id: "paper_arxiv_2510_12635"
source_kind: "arxiv"
source_id: "arxiv:2510.12635"
source_url: "https://arxiv.org/abs/2510.12635"
pdf_url: "https://arxiv.org/pdf/2510.12635"
read_id: "read_paper_arxiv_2510_12635"
kind: library-read
tags: []
---

# Memory as Action: Autonomous Context Curation for Long-Horizon Agentic Tasks

> 将工作记忆管理内化为策略动作，通过端到端强化学习让 Agent 自主学会何时裁剪与压缩上下文。

## Brief

本文针对长时程 Agent 任务中的上下文膨胀导致注意力稀释问题，提出 Memory-as-Action (MemAct) 框架。该框架将记忆管理操作（删除历史记录、插入摘要）与任务动作统一到同一策略空间中，通过端到端强化学习联合优化。实验表明，MemAct-RL-14B 在多目标 QA 任务上以仅 51% 的平均上下文长度匹配了 16 倍大模型 (Qwen3-235B) 的准确率，且学到的策略能跨任务复杂度泛化。

## Claims

- MemAct-RL-14B 在多目标 QA 任务上达到 59.1% 准确率，超过 Qwen3-235B (53.1%) 和 Tongyi-DeepResearch (56.0%)，同时平均每步上下文长度仅 3,500 tokens [§4.4]。
- MemAct-RL-14B 总 token 消耗 (8.2×10⁴) 比 Qwen3-235B (16.7×10⁴) 低 51%，比 Search-R1-14B (19.3×10⁴) 低 57% [§4.4, Table 1]。
- MemAct-RL-7B 相比 Search-R1 总延迟降低 40%，主要来自紧凑上下文减少 prefill 时间和提高 prefix cache 命中率 [§4.4]。
- 强化学习将多目标准确率从 SFT 版本的 0.485 提升至 0.591（14B 模型），表明 RL 对记忆决策优化不可或缺 [§4.5]。
- 7B 和 14B 模型自动学到不同的记忆策略：7B 倾向于更激进的单次裁剪（约 6 条记录），14B 呈双峰分布——推理中细粒度裁剪（约 2 条），子目标完成后粗粒度清理（约 6 条）[§4.6, Fig. 5]。
- 仅在 ≤3 目标任务上训练的策略可有效泛化至 8 目标任务，MemAct-RL-14B 在 8 目标设置下达 54.3% 准确率，远超 Search-R1 的 39.3% [§4.6]。
- DCPO 的轨迹分割机制不依赖于特定 RL 算法——PPO 变体与 GRPO 版本表现相当 [§A.4, Table 7]。

## Assumptions

- 记忆操作通过 function-call 形式实现，隐含假设底层推理引擎支持工具调用的结构化解析与执行 [§3.1]。
- SFT 冷启动数据由 DeepSeek-V3.1 合成，假设该模型的记忆管理行为（在分阶段提示引导下）足以提供有效的初始化监督 [§4.1.2]。
- 稀疏终端奖励（仅任务成功 +1、约束违规 -0.1）足以让策略学会细粒度的记忆编辑决策，无需中间奖励信号 [§3.3.2]。
- 评估任务（多跳 QA、网页浏览搜索）代表长时程 Agent 任务的核心挑战，但未涵盖代码生成、工具使用链等其他 Agent 场景 [§4.1]。
- 基于 ID 的记录寻址在上下文编辑后仍能保持一致性，假设 Agent 框架在执行裁剪后正确维护 ID 映射 [§3.2]。

## Method

MemAct 框架由三个核心组件构成：

**统一动作空间**：将动作空间扩展为 A = A_task ∪ A_mem。任务动作 A_task 为标准环境交互（搜索、浏览等）；记忆动作 A_mem 采用 Prune&Write 算子，参数为 (I_target, c)——I_target 为待删除记录的 ID 集合，c 为生成的记忆内容（包含目标、结论、状态、假设的结构化摘要）。记忆动作执行后，被删除的记录从工作记忆中移除，记忆动作本身作为新记录原地追加，使其后续仍可被寻址和更新。

**MDP 形式化**：状态 s_t 为当前工作记忆 H_t = [z_1, ..., z_k]，每条记录 z_i = (a_i, o_i, id_i)。任务动作的转移为追加新记录；记忆动作的转移为 ID 过滤删除 + 追加记忆记录。目标是学习策略 π_θ(a|H_t) 最大化期望累积奖励。

**DCPO 训练算法**：解决 Prune&Write 引入的非连续轨迹问题。因果 LM 假设上下文单调增长，删除操作破坏此假设导致训练-推理不匹配。DCPO 在每个记忆编辑点将轨迹逻辑分割为独立片段 σ_i = (C_i, y_i)，其中 C_i 为片段起始时的固定上下文前缀，y_i 为后续生成序列。训练时对每条 prompt 生成 N_traj=5 条完整轨迹，按 round-robin 策略采样 N_seg=12 个片段进行优化。每个片段继承轨迹级优势 A(τ)（GRPO 风格的组相对归一化），梯度仅在正确重建的上下文上计算。

## Eval

**数据集**：多目标 QA（基于 HotpotQA 构建，2-8 个子目标，每级 200 样本）；单目标 QA（2WikiMultihopQA、Bamboogle、HotpotQA、Musique、Frames、BrowseComp-Plus）。训练仅用 ≤3 目标任务（SFT 930 例，RL 10,240 轨迹）。

**基线**：Full-Context（Qwen3-235B 无裁剪）；外部管理（Sliding Window、Summarization、A-MEM）；学习型 Agent（MEM1 强制每步压缩、Tongyi-DeepResearch-30B、Search-R1 = MemAct 去掉记忆动作）。所有基线除特别说明外使用 Qwen2.5-14B-Instruct。

**指标**：Task Accuracy（LLM 评估器三pass共识协议）；Solved Sub-objective Count；平均上下文长度；总 token 消耗；工具调用次数；推理延迟（SGLang 引擎，2,000 条轨迹）。

**主要结果**：MemAct-RL-14B 在 Pareto 前沿占优——多目标准确率 59.1% vs Qwen3-235B 53.1%，上下文长度减少约 50%。消融显示固定间隔裁剪（每 5 步）在 8 目标任务上显著落后于学习策略，验证主动记忆管理的必要性。

## Weaknesses

- 评估全部集中于 QA/搜索类任务，未涉及代码生成、多工具组合、真实软件工程等 Agent 场景，泛化性声明缺乏跨任务类型的验证 [§4.1]。
- 稀疏终端奖励下的信用分配问题被作者承认但未解决——论文仅提供 "内在耦合" 的直觉论证，无定量证据表明该问题可被框架自身消解 [§6]。
- SFT 冷启动数据高度依赖 DeepSeek-V3.1 在分阶段提示下的合成质量，但未报告合成数据的错误率或质量分布，无法评估冷启动数据噪声对最终策略的影响 [§4.1.2]。
- 记忆压缩是有损的——一旦原始信息被摘要替代便不可恢复。论文 Table 10 展示了记忆幻觉和歧义塌缩两种失败模式，但未量化这些失败在实际任务中的发生频率和对准确率的影响 [§6, Table 10]。
- LLM 评估器（gpt-oss 系列）的准确率未被人工标注校准，三pass共识协议的一致性率未报告，评估可靠性缺乏独立验证 [§4.2]。
- DCPO 的轨迹分割将全局优势均分给所有片段，但作者承认 "将所有记忆操作视为同等重要" 是次优的——这意味着关键记忆决策和无关决策获得相同梯度权重，可能限制复杂场景下的训练效率 [§6]。

## Relations

- competes-with MEM1 (Zhou et al., 2025) `[med]`：两者均通过 RL 学习记忆管理，但 MEM1 强制每步压缩而 MemAct 允许 agent 自主决策何时编辑。论文直接对比并报告 MemAct 在多目标任务上优于 MEM1（59.1% vs 49.4% avg），但 MEM1 使用相同训练数据重训，对比公平性依赖于实现细节。
- competes-with A-MEM (Xu et al., 2025) `[high]`：A-MEM 作为外部记忆系统基线被直接对比。MemAct 在准确率上大幅领先（59.1% vs 39.9%），但 A-MEM 的 token 消耗更低（3.9×10⁴ vs 8.2×10⁴），说明两者在准确率-效率权衡上定位不同。论文明确讨论此对比。
- extends Search-R1 (Jin et al., 2025) `[high]`：Search-R1 是 MemAct 的直接消融——相同训练流程但无记忆动作。论文使用 Search-R1 作为核心 ablation 验证记忆管理的增益，二者共享 GRPO 训练管线和数据集。
- orthogonal MemGPT (Packer et al., 2023) `[low]`：MemGPT 将 LLM 视为操作系统管理虚拟记忆，属于外部控制器范式；MemAct 将记忆管理内化为策略动作。两者解决相似问题但架构哲学不同，论文在 Related Work 中明确区分这两个范式但未直接对比。
- builds-on GRPO (Shao et al., 2024) `[high]`：DCPO 的策略优化目标直接采用 GRPO 的 clipped surrogate objective 和组相对优势计算，在此基础上增加轨迹分割机制。论文 §3.3.3 明确引用。
