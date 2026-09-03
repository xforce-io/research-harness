---
title: "ToolOmni: Enabling Open-World Tool Use via Agentic Learning with Proactive Retrieval and Grounded Execution"
paper_id: "paper_arxiv_2604_13787"
source_kind: "arxiv"
source_id: "arxiv:2604.13787"
source_url: "https://arxiv.org/abs/2604.13787"
kind: legacy-topic-read
topic: "decision"
source_note: "notes/active/13_toolomni_enabling_open_world_tool_use.md"
---

# Legacy topic read

This Library read artifact was backfilled from an existing topic note. It preserves historical reading state; it was not produced by the current Library deep-read runner.

## Source topic note

---
zone: active
pin: false
score: 0
dwell: 0
paper: "ToolOmni: Enabling Open-World Tool Use via Agentic Learning with Proactive Retrieval and Grounded Execution"
arxiv: "2604.13787"
authors: ["Shouzheng Huang", "Meishan Zhang", "Baotian Hu", "Min Zhang"]
year: 2026
venue: "arXiv preprint v1 (Harbin Institute of Technology, Shenzhen)"
note_number: 13
---

## Claims

- 主动迭代检索（agent 自主生成多轮搜索查询）优于单轮被动检索：多域（Multi-Domain）平均 NDCG@5 78.29%，对比最强 IterFeedback 基线 73.16%，+5.13 个绝对点；消融中，迭代检索比一次性检索（one-shot）平均 NDCG@5 提升 +4.5% [13: §4.2, §4.5, Figure 3]。
- 端到端执行 SoPR 54.13%，超过 GPT-3.5 pipeline（42.35%）+11.78 个点、ToolGen-7B（36.10%）+18 个点；SoWR 50.16%，超过 ToolLlama-v2（35.90%）+14.26 个点 [13: §4.3, Table 2]。
- 类别泛化（Category Generalization，全新领域工具）SoPR 55.95%，超过 GPT-3.5 pipeline（42.10%）+13.85 个点，显示 ToolOmni 学到了"工具使用元技能"而非领域记忆 [13: §4.4, Table 3]。
- 解耦 GRPO（分独立 reward + 分步梯度更新）比合并梯度更新（Combined Update）高 52.5% vs 42.6% SoPR；比 Vanilla GRPO（单一 reward）高 52.5% vs 50.8% [13: §4.5, Figure 5]。
- 轨迹过滤（仅在检索阶段成功召回全部 ground-truth 工具后才启动执行 rollout）是关键约束：去除后 SoPR 从 52.5% 跌至 38.5%（-14 个点）[13: §4.5, Figure 5]。
- RL 主导检索质量（去除 RL：NDCG -28.99%；去除 SFT 仅 -0.51%）；SFT 主导执行质量（去除 SFT：SoPR -9.1%；去除 RL：SoPR -28.7%），两阶段训练缺一不可 [13: §4.5, Table 4]。
- 噪声鲁棒性：注入 15 个对抗工具后 ToolOmni SoPR 从 39.3% 反弹至 58.2%，ToolLlama-v2 单调下降至 20.5%；扩展候选池反而使模型找到功能等价的替代工具 [13: §4.4, Figure 4]。
- k=5 是检索数量最优值（倒 U 型：k=1→62.8%、k=5→78.3%、k=9→73.2% NDCG@5），过多候选引入"噪声工具"稀释注意力 [13: §4.6, Table 5]。
- 主动检索以 3.02 次嵌入检索（便宜）换取 2.47 次 API 调用（昂贵），低于 ChatGPT 的 3.98 次 API 调用，检索开销被执行侧节省抵消 [13: §4.7, Table 6]。

## Assumptions

- 工具库已预先建立嵌入索引（ToolRetriever 作为 backend embedding model）；运行时增量工具不在评测范围内 [13: §3.1]。
- 训练时可获取 ground-truth 工具标签（用于检索 reward `rrec` 和执行轨迹过滤 `Tgold ⊆ Tret`）；部署时 ground-truth 不可知，但检索策略已通过 RL 内化 [13: §3.3.2]。
- LLM-based Tool Simulator（MirrorAPI）能足够忠实地模拟真实 API 反馈，包括错误类型（缺参、认证失败、超时）；论文假设该仿真足以支撑执行策略学习 [13: §3.3.2, Appendix B.4]。
- ToolBench（16,464 APIs / 49 类别）足以代表 open-world 工具环境；评测结论可外推至其它工具分布 [13: §4.1]。
- 检索-执行强制串行（cascaded）是正确的序列；不存在"执行中间结果驱动的新工具发现"需求——作者在 Limitations 节承认该假设为当前架构约束 [13: §Limitations]。
- Qwen3-4B 作为 base model 规模充足；更大模型的增益边界未验证，作者显式列为局限 [13: §Limitations]。

## Method

**框架概述（§3.1）。** ToolOmni 在单一 reasoning loop 中串联两阶段：先 Proactive Retrieval（迭代检索），后 Grounded Execution（接地执行）。策略 πθ 驱动整个轨迹 τ。

**Phase 1：Proactive Retrieval（§3.3.1）。** 给定用户指令 Q，agent 分析意图后输出 `<search>qt</search>` 标签封装的查询；检索服务器用预训练 embedding 模型 E(·) 计算余弦相似度，返回 top-k 候选：`Tret = top-k(cos(Eτ, Eqt))`。Agent 可迭代发出多条查询（最多 4 轮），直至收集到足够的工具后在 `<tool_call>` / `</tool_call>` 内输出排序后的最终工具子集 Tsub。

**Phase 2：Grounded Execution（§3.3.2）。** 获得 Tsub 后，agent 进入执行阶段：在 `<reasoning>` 标签内先推理，再调用具体工具（指定 tool_name + 参数）；LLM-based Tool Simulator 返回 `<information>` 块；循环直至输出 `<answer>`。执行训练只在检索阶段成功 `Tgold ⊆ Tret` 的轨迹上进行（Selective Rollout）。

**两阶段训练（§3.2、§3.3.3、§3.3.4）：**

*Stage 1 — 冷启动 SFT：* 从 ToolBench 构建 28k 检索轨迹（rejection sampling 过滤低质）+ 33k 执行轨迹（Qwen-2.5-72B 生成推理步骤，Qwen-2.5-32B 验证；70% 正样本 + 30% 负样本）；标准交叉熵，token-level masking 只计 agent 生成 token。

*Stage 2 — Open-World Tool Learning（Decoupled Multi-Objective GRPO）：*
- 检索 reward：`Rret = α1·rfmt + α2·rrec·rconv`（格式合法 + 对 Tgold 的 recall × 有效转化率；α1=0.2, α2=0.8）
- 执行 reward：`Rexec = β1·rfmt + β2·rans`（格式合法 + 答案正确性；β1=0.2, β2=0.8）
- 组内相对优势归一化独立计算：`A_task = (R_task - mean) / (std + ε)`，隔离两子任务学习信号
- 分步梯度更新（先 retrieval update 再 execution update），而非两路梯度直接求和，避免量级冲突

## Eval

- **Benchmark：** ToolBench（16,464 APIs / 49 类别）
- **检索评测：** NDCG@k（k∈{1,3,5}）；In-Domain（工具子集受限）与 Multi-Domain（全量 16k+ API）两种设置；1587 条测试查询（I1: 796 / I2: 573 / I3: 218）
- **执行评测：** StableToolBench Solvable 子集 765 条；SoPR（查询解决率）与 SoWR（答案胜率）；GPT-5 Mini 作为 judge（Kendall τ=0.847 与人类对齐；跨 GPT-5 / Gemini-3 / Qwen-32B 三 judge 结论稳定）
- **泛化评测：** Tool Generalization（已知类别内未见工具 SoPR 52.20%）+ Category Generalization（全新领域 55.95%）
- **对比基线：** 检索侧：BM25、EmbSim、ToolRetriever、IterFeedback、Re-Invoke、ToolGen；执行侧：GPT-3.5、ToolLlama-v1/v2、ToolGen-3B/7B
- **模型：** Qwen3-4B-Instruct，8×NVIDIA H100，训练单 epoch；GRPO G=5，温度 1.0

**关键数字：** Multi-Domain 平均 NDCG@5 78.29%（第一）；端到端 SoPR 54.13% / SoWR 50.16%（均第一）；Category Gen. SoPR 55.95%（超 GPT-3.5 +13.85%）。

## Weaknesses

- **级联顺序是硬约束，不可在执行中途发现新工具。** 若执行阶段的中间 API 返回结果暗示需要一个检索阶段未收录的工具（如动态 API 引用），ToolOmni 无法重入检索；作者承认这是当前"rigid cascaded paradigm"的局限，但未量化此情形在 ToolBench 测试集中的发生频率 [13: §Limitations]。
- **仅在 ToolBench 单一 benchmark 上完整评测。** 端到端 SoPR/SoWR 只在 StableToolBench（765 条）上测；论文未在 APIBench / Gorilla / ToolACE 上重复端到端实验——"ToolOmni 学到了通用工具使用机制"的结论缺乏跨 benchmark 佐证 [13: §4.4]。
- **SFT 冷启动与 RL 数据均来源于 ToolBench，执行质量存在 benchmark 过拟合风险。** 执行训练集 33k 条来自 ToolBench 训练集；Category Generalization 虽引入新领域，但底层工具文档格式与 ToolBench 一致（RapidAPI JSON schema）；在文档格式差异更大的企业内部 API 环境下泛化性未知 [13: §3.2, Appendix A.4]。
- **"主动检索"backend 仍是 embedding 检索，继承 [11] 揭示的 dense 检索瓶颈。** ToolOmni 的检索后端是 ToolRetriever（dense embedding 模型）；[11] 已量化 dense 检索器在 ToolBench 上 NDCG@10 ≤ 33.83（未训练），[13] 通过迭代查询改写把精度推到 78.29%，但若工具文档与查询存在词面重叠极低的结构差异（[11] §4.1 ROUGE-L 0.06），多轮改写能否突破这一上限未独立验证 [13: §3.3.1]。
- **执行 reward 依赖 LLM-as-Judge（GPT-5 Mini），引入循环依赖。** 训练时用 GPT-5 Mini 评判答案正确性（`rans`），测试时也用 GPT-5 Mini 计算 SoPR/SoWR；判断模型与 judge 是否共享偏好空间未做独立消融 [13: §4.1, §3.3.3]。
- **噪声鲁棒性实验只在 I3 单一难度档测试。** Figure 4 的对抗噪声曲线仅报告 "complex I3 tasks"；I1 / I2 的鲁棒性数据缺失，而大多数企业实际查询更接近 I1/I2 的单工具 / 同类多工具场景 [13: §4.4, Figure 4]。
- **执行 budget 固定上限 6 轮，无法感知任务复杂度。** 所有模型统一限制最多 6 次工具调用；Decision Agent 的 AutoFlow 长链任务（查询-分析-撰写-校验-审批-落地）可能需要更深的序列决策，该预算限制在企业场景下能否被动态放宽未讨论 [13: §4.3]。
- **仅用 Qwen3-4B 训练，更大模型的解耦 GRPO 收益曲线未知。** 论文承认"upper bound of performance and emergent reasoning from scaling remain unexplored"——从 [7] 的 GPT-4.1 baseline 饱和来看，若强模型基线本身已接近上限，解耦 GRPO 带来的增量可能缩小 [13: §Limitations; cf. 07: §5.2]。

## Relations

- **builds-on 07_mcp_zero_active_tool_discovery_for [high]：** 两者共享核心前提——让模型主动生成语义化请求（而非直接用 user query）能显著提升工具召回精度。MCP-Zero 无需训练，用 `<tool_assistant>{server, tool}` 提示模型生成请求，在 APIBank Full 集上 +25–30 个百分点；ToolOmni 通过 GRPO 端到端训练 query 生成策略，在 ToolBench 16k 工具池上多域 NDCG@5 78.29%。ToolOmni 是"训练版 MCP-Zero"的工程延伸：用 RL 代替 prompt 来内化主动请求生成能力，并额外联合优化执行阶段。两者共同佐证"query 改写是工具检索的必要前置"；ToolOmni 进一步量化了 RL fine-tuning 相对纯 prompt 方案的增量（消融：w/o RL 检索 NDCG -28.99%，而 MCP-Zero 是 prompt-only）[13: §4.5; 07: §5.3]。
- **extends 11_retrieval_models_aren_t_tool_savvy [high]：** [11] 确立了 dense 检索器在 ToolBench 上的上限（NV-Embed-v1 nDCG@10 ~33–42，Completeness@10 ≤ 43%），并证明 instruction-augmented 查询能带来 +7–17 nDCG 点。ToolOmni 沿着"让模型改写查询"这一 [11] 指向的路径，以迭代 RL 训练把 NDCG@5 推到 78.29%——相当于把 [11] 里 offline instruction 离线生成的增益内化为 online RL 能力。[11] 的"instruction 在训练与推理两侧独立有效"与 ToolOmni 的"SFT 驱动执行、RL 驱动检索"两阶段设计相互映证 [13: §4.5; 11: §7.2]。
- **extends 05_agent_as_a_graph_knowledge_graph [med]：** [05]（Agent-as-a-Graph）在 LiveMCPBench（70 servers / 527 tools）上用 KG 类型节点 + 加权 RRF 胜出 MCP-Zero（Recall@5 0.85 vs 0.70）；ToolOmni 在量级更大的 ToolBench（16k 工具）上以 RL 迭代查询路径达到 78.29% NDCG@5。两者选用了不同的归纳偏置：[05] 类型化结构显式约束候选空间，ToolOmni 直接优化查询生成策略；两者原则上可叠加——ToolOmni 式查询生成 + AgentGraph 式类型化后端。[13] 的工具规模（16k）更接近企业目标，但同样未报告 5k+ 规模下多步链式调用的 Completeness 曲线 [13: §4.2; 05: §4.2]。
- **orthogonal to 03_why_reasoning_fails_to_plan_a [med]：** FLARE [03] 证明 ReAct 步进推理在需要超过若干步序列决策的任务上系统性退化；ToolOmni 的执行阶段是 ReAct-based（Thought-Action-Observation 循环上限 6 轮），直接继承该结构性局限。ToolOmni 的检索阶段多轮 query 规划引入了有限的前瞻性，但执行深度仍被 6 轮 budget 硬限——在 Decision Agent 的长链 AutoFlow（>6 步）场景下，该约束直接制约任务完成率 [13: §4.3; 03: §4]。
- **orthogonal to 06_why_do_multi_agent_llm_systems [low]：** ToolOmni 在单 agent 范畴内操作，不涉及多 agent 协同失真（MAST FC2）。但减少工具检索噪声能间接降低 FM-2.6（reasoning-action mismatch）：检索召回率越低，下游 agent 越容易调用错误工具触发幻觉链；ToolOmni 将每查询 API 调用从 3.98 降至 2.47 次，执行错误率相应下降 [13: §4.7; 06: §3.2]。
- **orthogonal to 12_automated_composition_of_agents_a_knapsack [low]：** [12] 把 agent/skill 组合问题建模为背包优化；ToolOmni 的 Tsub 最终工具子集选择（从候选 k 工具中选最有用的）结构上与背包约束问题相似（有限上下文 budget），但 [13] 未用资源约束优化框架，而是通过 RL rconv reward 隐式鼓励精简选择。两者处理不同粒度（agent 组合 vs 工具子集选取）但可在 Composer 层面联合设计 [13: §3.3.1; 12]。

### Relation to thesis

直接命中论题"工具幻觉的语义解耦防御 / ContextLoader 5k–10k 工具规模有效性"研究缺口（thesis §设计上下文 #3）。三点对 Decision Agent 的可操作启示：

1. **RL 端到端训练查询改写比 prompt-only 多 28.99% NDCG——ContextLoader 的查询前置改写必须进入训练 pipeline。** ToolOmni 的消融明确：仅 SFT（对标 [11] 的 instruction-augmented 静态改写路径）在检索 NDCG 上远不及加 RL 的全量模型（NDCG 49.3% vs 78.3%）。BKN 的"逻辑/行动解耦"是正确方向，但若只做离线语义结构化而不做在线 RL 优化，检索精度上限被严重低估。Decision Agent 的 ContextLoader 若要在 5k–10k 工具规模达到可用精度，需考虑 ToolOmni 式的检索 reward 细粒度设计（recall × conversion）[13: §4.5, Table 4]。

2. **执行成败与检索质量强耦合：Golden Tools 条件下 SoPR 仅 54.7%，而端到端执行同样 54.1%——说明当前瓶颈不在检索而在执行推理。** ToolOmni 的 Golden Tool 实验（已给出正确工具但仍只有 54.7% SoPR）揭示：即便召回完美，Agent 的执行推理本身还有相当的错误率。Decision Agent 不能把提升工具召回等同于提升任务通过率——Composer 的规划质量和 Execution Factory 的执行健壮性是独立瓶颈 [13: §4.3, Table 2]。

3. **ToolBench 16k API 规模接近企业 5k–10k 目标，但仍只做了单 benchmark——需自家 BKN 工具集复测。** 这是目前工具检索领域验证规模最接近 Decision Agent 目标的论文；但 ToolBench 工具文档均为 RapidAPI JSON schema（标准化、英文、完整），与企业内部 KWeaver BKN 工具（可能存在描述质量参差、多语言、内部依赖）有结构差异。thesis 可证伪点"BKN 语义解耦在 5k+ 工具规模仍有效"的实验设计可借鉴 ToolOmni 的 Decoupled GRPO 框架，但必须在自家 BKN 工具集上从头复测 Recall@5 与 Completeness@10 [13: §4.1; thesis §可证伪点追踪]。
