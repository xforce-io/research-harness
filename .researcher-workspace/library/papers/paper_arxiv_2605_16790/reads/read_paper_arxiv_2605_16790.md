---
title: "TIER: Trajectory-Invariant Execution Rewards for Multi-Step Tool Composition"
authors: ["Anay Kulkarni","ChiaEn Lu","Dheeraj Mekala","Jayanth Srinivasa","Gaowen Liu","Jingbo Shang"]
paper_id: "paper_arxiv_2605_16790"
source_kind: "arxiv"
source_id: "arxiv:2605.16790"
source_url: "https://arxiv.org/abs/2605.16790"
pdf_url: "https://arxiv.org/pdf/2605.16790"
read_id: "read_paper_arxiv_2605_16790"
kind: library-read
doc_type: "paper"
tags: []
---

# TIER: Trajectory-Invariant Execution Rewards for Multi-Step Tool Composition

> 多步工具组合中，旧奖励要么稀疏要么绑死参考轨迹——TIER 直接从函数 schema 和运行时执行结果派生多级奖励，无需标注轨迹即可在深度 6 的组合任务上保持 ≥90% 准确率。

## Essence

**问题**：现有 RL 方法在多步工具组合（multi-step tool composition）上性能骤降——outcome-based 奖励在 2-step 即跌至 ~1%，而 trajectory-supervised 奖励（如 ToolRL）依赖标注轨迹，会惩罚合法的替代路径，在深度 ≥5 时崩溃至 0%。

**做法**：将工具调用表示为 JSON AST，奖励不与参考轨迹比对，而是沿四个维度逐级验证整个序列：格式合法性（R_format）、schema 符合度（R_parse，含工具名/参数/类型）、执行成功（R_exec，全有或全无）、答案正确性（R_answer，权重×5）。四项在序列级聚合后归一化到 [0,1]，任何能产出正确答案且通过中间校验的路径均获满分。

**证据**：在 DepthBench（1–6 步深度分层基准）上，TIER 在所有深度保持 ≥90% 准确率（深度 6 = 90%），而 Simple-RL 和 ToolRL 在深度 ≥5 均为 0% [Table 1]。

**边界**：假设工具执行是廉价、可重复且无副作用的；全部实验仅用 Qwen3-8B 单一模型、单轮设定、确定性合成后端。

## Claims

- 多步工具组合的性能瓶颈在奖励设计而非优化算法：在相同 TIER 奖励下，GRPO、Batch-Normalized GRPO、DAPO 均达到 >93% 总体准确率 [Table 7, §F]。
- Trajectory-supervised 奖励在高深度崩溃的原因不是训练不充分而是奖励误设：正确但偏离参考轨迹的执行获低分，将策略推离合法解 [§4.2]。
- 奖励组件不可互换：仅当 format + parsing + execution + correctness 同时存在时才在深度 ≥5 维持 >90%；移除任一中间组件均导致深度 5–6 归零 [Table 2, §4.3]。
- 缺少 parsing 时加入 execution 奖励反而降低总体准确率：模型学会将查询路由到少数"安全"工具以获取执行奖励，同时给出错误答案（reward hacking）[§4.3]。
- R_exec 采用全有或全无规则（非按成功调用比例打分），因为组合依赖结构中第 k 步失败会使下游调用失效，按比例打分会奖励无可用结果的工作 [§2.2]。
- R_answer 权重设为 5（远高于其他组件），因为低权重会导致模型满足格式和执行校验但不产出正确答案，训练在答案信号生效前停滞 [§2.2]。
- DepthBench 训练的模型（1.7K 样本）在 BFCL v3 上总体准确率 68.92%，优于 ToolACE（10K 样本，64.66%）和 xLAM（60K 样本，63.02%）[Table 6]。
- 在 NestFUL 上，TIER 后训练的 Qwen3-8B（8B）3-shot ICL 达到 0.75 EM Acc，超过 DeepSeek-V3（685B，0.60）和 GPT-4o（0.60）[Table 4b]。

## Assumptions

- 工具执行环境是确定性的、廉价的、可安全重复执行的——论文在 Limitations 中承认真实环境存在延迟、噪声、部分失败、速率限制和不可逆副作用 [§7]。
- Answer correctness 可以被确定性规则验证（因此适用 RLVR），但论文未说明非确定性任务（如开放式生成）如何适配。
- 单轮（single-turn）设定足以研究多步组合；多轮对话中跨轮的组合未被训练或评估。
- 模型规模效应可忽略：所有实验仅用 Qwen3-8B，未验证更大或更小模型的趋势 [§7]。
- 奖励权重（λ_p=0.25, R_answer=5）基于初步调优固定，未探索跨领域/深度的敏感性 [§7]。

## Method

**Before vs After 对比**：

| 方面 | Simple (outcome) | ToolRL (trajectory) | TIER |
|------|-------------------|---------------------|------|
| 反馈粒度 | 二值（格式+答案） | 细粒度，但锚定参考轨迹 | 细粒度，从 schema + 执行派生 |
| 路径不变 | ✓（但不区分错误类型） | ✗（惩罚合法替代路径） | ✓（所有合法路径等权） |
| 中间步骤信号 | 无 | 有（但路径依赖） | 有（schema + 执行校验） |
| 工具变更成本 | 无 | 需重新生成参考轨迹 | 自动跟踪 schema 变更 |

**输入**：用户查询 + 可用工具 schema 列表（含 distractors）。模型生成 `<think>` 推理 + JSON AST 格式的工具调用序列，通过 `<tool_call>` 标签输出。

**奖励计算**（每条生成的轨迹计算一次，序列级聚合）：

1. **R_format ∈ {0,1}**：AST 是否格式良好可解析。失败则下游奖励全归零。
2. **R_parse = R_name + R_param + R_dtype ∈ [0,3]**：
   - R_name：所有工具名有效→1，否则 0（且 R_parse=0）。
   - R_param / R_dtype：`clip(1 − λ_p · p, 0, 1)`，p 为全部调用中参数/类型不匹配总数，λ_p=0.25。
3. **R_exec ∈ {0,1}**：全部调用执行成功→1，否则 0（全有或全无）。
4. **R_answer ∈ {0,5}**：最终答案正确→5，否则 0。
5. 总和 R_total 归一化到 [0,1]。

**RL 训练**：GRPO-style 策略梯度，组内归一化优势 A = (R − μ_G) / (σ_G + ε)，PPO clipped surrogate（单次更新退化为 REINFORCE），token 级 KL 正则（k3 估计器，λ_KL=0.04）。每 prompt 采样 8 条轨迹。

## Eval

- **基准 1 — DepthBench**（766 验证样本，1,710 总样本，163 工具，深度 0–6 分层）：TIER 总体 98.57%，深度 5–6 均 90%；Simple-RL 总体 66.15%，深度 ≥5 = 0%；ToolRL 总体 67.49%，深度 5–6 = 0% [Table 1]。还对比了 Qwen3-8B zero-shot（35.51%）、3-shot ICL（79.56%）、GPT-5 3-shot（92.16%）。
- **基准 2 — BFCL v3**：TIER 68.92% overall，优于 Simple（66.30%）、ToolRL（37.27%）、SFT（61.47%）、Base（64.31%）。ToolRL 在 Multiturn（0.38）和 Irrelevance（0.81）崩溃 [Table 3]。
- **基准 3 — NestFUL**：受控对比中 TIER 0.684 vs ToolRL 0.476 EM Acc [Table 4a]；3-shot ICL 对比中 TIER 8B 达 0.75，超过 DeepSeek-V3 685B（0.60）和 GPT-4o（0.60）[Table 4b]。
- **消融**：逐一移除 parsing / execution，深度 5–6 均降至 0%；仅 format+correctness 在深度 2 即降至 1.25% [Table 2]。
- **数据集对比**：DepthBench（1.2K 训练样本）在 BFCL 总体优于 ToolACE（10K）和 xLAM（60K），差距集中在 Multiturn 指标 [Table 6]。
- **RL 算法消融**：GRPO（97.17%）、BN-GRPO（98.56%）、DAPO（93.21%）均达 >90%，验证收益来自奖励设计而非优化器 [Table 7]。
- **中间表示消融**：JSON AST（68.92%）优于 XML（66.06%）和直接生成（63.80%）[Table 5]。

## Weaknesses

- **Answer correctness 验证机制未公开**：R_answer 是权重最高的组件（×5），但论文未说明"答案正确"如何判定——是精确匹配、规则匹配还是 LLM 判定？对于非确定性问题（如"推荐一家餐厅"）如何验证？这直接影响可复现性和适用范围。
- **DepthBench 工具与查询均为手工构建**：163 个工具和 1,710 条查询由作者手工设计，可能隐含与 TIER 奖励结构对齐的偏差（如工具命名规范、参数类型简单），使得 TIER 在此环境上的高表现可能不完全迁移到真实 API。
- **训练/测试工具分割的泛化性存疑**：虽然 target tools 和 distractors 在训练/测试间不相交，但 163 个工具的总量较小，测试集覆盖的工具多样性有限，"未见工具泛化"的声明未经大规模工具池验证。
- **GPT-5 对比设定不公平但未充分讨论**：GPT-5 在 DepthBench 上 3-shot ICL 达 92.16%，而 TIER 是 RL 后训练结果；论文称"not a head-to-head comparison"但仍将其列为基线，可能误导读者。更重要的是，GPT-5 在深度 6（60%）已开始下降，说明 TIER 的深度优势可能部分来自对 DepthBench 分布的过拟合而非真正的组合泛化。
- **单模型、单规模、单种子**：所有实验仅用 Qwen3-8B，无方差报告，无 scaling 趋势分析。论文虽在 Appendix F 提到"gains primarily arise from reward design"，但未提供不同模型大小下奖励组件是否仍互补的证据。
- **R_exec 全有或全无规则在部分可恢复场景下过于严苛**：论文论证组合依赖中第 k 步失败使下游失效，但真实场景中存在 try-catch、fallback 路径或独立子目标的部分成功，该规则会将其全部归零，可能抑制有用探索。

## Relations

- competes-with ToolRL [Qian et al., 2025] [high]：TIER 与 ToolRL 直接对比，核心区别在于 TIER 从 schema+执行派生奖励（路径不变），ToolRL 从参考轨迹匹配派生奖励（路径依赖）；论文在 DepthBench、BFCL、NestFUL 三个基准上均报告 TIER 优于 ToolRL。
- builds-on GRPO / DeepSeek-R1 [Shao et al., 2024; Guo et al., 2025] [high]：TIER 使用 GRPO-style 策略梯度 + KL 正则作为优化框架，贡献在奖励设计而非优化算法；Appendix F 显式验证不同优化器下收益保持。
- extends BFCL v3 [Patil et al., 2025] [high]：TIER 在 BFCL v3 上评估迁移性，并指出 BFCL 缺乏可执行后端和深度分层——DepthBench 被设计为补充这些缺失维度。
- contradicts NestFUL [Basu et al., 2025] [med]：NestFUL 不按组合深度分层且使用精确轨迹匹配评估，TIER 指出这无法诊断奖励在深度上的失败模式，并在 NestFUL 上以路径不变奖励超越 ToolRL 的精确匹配方法。
- orthogonal PORTool [Wu et al., 2025] [med]：PORTool 通过 rollout-tree 比较重分配轨迹级奖励，仍为路径依赖；TIER 不与之直接对比但论文指出其信号"remains path-dependent and tied to specific exploration trajectories"。

## Takeaway

- **方法层面值得借鉴**：将工具使用奖励分解为 format/schema/execution/correctness 四级、从函数 schema 和运行时直接派生而非依赖参考轨迹，是一个可迁移的奖励设计范式——适用于任何可确定性验证的领域（code generation、NL-to-SQL 等论文自身也指出）。
- **需要警惕的部分**：R_answer 的验证机制不透明；DepthBench 规模小且手工构建，高数字可能含环境偏差；单模型单种子无方差，奖励权重的泛化性未验证。
- **如果只记住一件事**：多步组合能力的涌现不在优化器，而在奖励是否同时提供"路径不变"和"多级错误区分"——缺少任一属性，深度 ≥5 即崩溃到 0%。
