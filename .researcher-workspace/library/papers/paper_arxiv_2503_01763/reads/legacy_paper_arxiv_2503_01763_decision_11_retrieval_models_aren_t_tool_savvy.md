---
title: "Retrieval Models Aren't Tool-Savvy: Benchmarking Tool Retrieval for Large Language Models"
paper_id: "paper_arxiv_2503_01763"
source_kind: "arxiv"
source_id: "arxiv:2503.01763"
source_url: "https://arxiv.org/abs/2503.01763"
kind: legacy-topic-read
topic: "decision"
source_note: "notes/active/11_retrieval_models_aren_t_tool_savvy.md"
---

# Legacy topic read

This Library read artifact was backfilled from an existing topic note. It preserves historical reading state; it was not produced by the current Library deep-read runner.

## Source topic note

---
zone: active
pin: false
score: 0
dwell: 0
paper: "Retrieval Models Aren't Tool-Savvy: Benchmarking Tool Retrieval for Large Language Models"
arxiv: "2503.01763"
authors: ["Zhengliang Shi", "Yuhan Wang", "Lingyong Yan", "Pengjie Ren", "Shuaiqiang Wang", "Dawei Yin", "Zhaochun Ren"]
year: 2025
venue: "arXiv preprint v2 (Shandong University, Baidu, Leiden)"
note_number: 11
---

## Claims

- 现有 IR 模型即便在传统 IR benchmark（MTEB / BEIR）上表现强，到工具检索任务上整体崩盘：6.4B 参数级 NV-Embed-v1 在 ToolRet 上 nDCG@10 仅 33.83，Completeness@10 ≤ 35%，Recall@10 ≤ 52% [11: §6.1, Table 4]。
- 工具检索性能直接拖垮下游 tool-use agent 的任务通过率：ToolBench-G1 上把官方人工标注 toolset 换成 IR 检索结果后，GPT-3.5 pass rate 从 62.0 降至 50.6（−11.4），且在 G2/G3 上趋势一致；ColBERTv2 等强检索器 Recall@10 仅 27.3 [11: §1, Figure 1, §7.1, Figure 6]。
- 通用 re-ranking 技术在工具检索上提供有限甚至负向收益：用 MonoT5 对 NV-Embed-v1 召回结果重排，平均 nDCG@10 从 33.83 跌至 28.92；mxbai-rerank 同样退化；最强 bge-reranker-v2-gemma 也仅带来 ~4.7% 的 Completeness@10 提升 [11: §6.1, Table 4]。
- 工具检索任务难在两点：（i）query–target 词面重叠极低（ROUGE-L 0.06，对比 NQ 0.31 / MS-MARCO 0.34），需更强语义表征；（ii）工具检索与传统 information-seeking 任务存在分布漂移，未显式优化的模型表现普遍下滑 [11: §1, §4.1, Table 2]。
- 同时，ToolRet 上的得分与 MTEB 得分之间仍呈中–强相关（Pearson 0.790, Spearman 0.441），即"工具检索是 IR 的子任务但非简单延伸"：方向一致、绝对水平显著低于 IR [11: §6.3, Figure 5]。
- 加上指令（instruction）显著提升所有 IR 模型在 ToolRet 上的表现，且对指令微调过的模型（NV-Embed-v1、e5-mistral-7b、GritLM-7B、gte-Qwen2-1.5B-inst.）增益最大；NV-Embed-v1 平均 nDCG@10 从 33.83 提升到 42.71 [11: §6.2, Table 5]。
- 提出 ToolRet-train（200k+ 指令式训练实例，配 K=10 负采样）；在该数据上微调 IR 模型可显著提升 nDCG@10，并把下游 tool-use LLM（GPT-3.5 / ToolLlama）的 pass rate 提升 10%–20% [11: §7.2, Figure 6]。
- 移除指令做消融训练后，模型仍优于未训练版本，但低于指令微调版本——指令既在评测时有用、在训练时也起独立作用 [11: §7.2, Appendix D]。

## Assumptions

- "工具检索可独立于工具调用规划单独评测"：ToolRet 把 retrieve-then-call 拆成单步 IR 问题，每个 query 配静态目标工具集；论文承认未来应做"interleaved retrieval-and-calling"，因此当前评测把规划失败和检索失败的耦合排除在外 [11: §8 limitation, Future Work (ii)]。
- "原数据集的 ground-truth 工具就是该 query 的最合适工具"——这是 one-to-many 问题的处理立场。论文承认合并多源数据后存在功能近似的多个工具，但坚持只用源标注做标签，假设模型应学到"细粒度功能区分能力" [11: §8 fine-grained functional differences]。
- 三类工具格式（Web API / Code Function / Customized App）足以覆盖工具检索的结构性差异；论文承认这是 format-based 而非 capability-based 划分，留待未来扩展按 query 长度 / target 数量 / 词面重叠等维度切分 [11: §8 beyond format-based]。
- LLM 自动生成的 instruction（GPT-4o + 100 条人写 seed + in-context learning）在 89.2% 的样本上"覆盖目标工具特征且忠于原查询"，且剩余 10.8% 由专家修订；论文假定该流程产出的 instruction 质量足以作为评测协议 [11: §4.3, Table 3]。
- 实验默认所有评测在英文文本检索上完成，不涉及多语种或多模态——这是 limitation 节明确承认的边界 [11: §Limitation]。

## Method

**数据收集（§3.1）。** 自 30+ 公开 tool-use 数据集汇总，覆盖：(i) ACL/NeurIPS 等会议主流 tool-use benchmark（ToolBench、ToolACE、APIGen、ToolEyes、ToolAlpaca、Gorilla、CRAFT、UltraTool、T-Eval、TaskBench…）；(ii) CIKM/EMNLP resource track（AppBench、ToolLens、COLT）；(iii) HuggingFace 公开数据集。时间窗 2023.08–2024.12。原始格式异质，统一对齐为 BEIR/MTEB 风格——每条任务 = `<query, target_tool_ids>`，每个工具 = `<unique_id, documentation>`。

**采样（§3.2）。** 每数据集用 NV-Embed-v1 嵌入 + K-means 聚类（k = min(query_count, toolset_size)），每簇取 1 条，去除冗余。toolset 端跨数据集做手工归并（如 COLT 与 ToolBench 重叠工具合并）。最终：7.6k 任务（Web 4.9k / Code 0.95k / Custom 1.75k），43k 工具（Web API 37.0k / Code 3.79k / Custom 2.44k）。avg query/instruction 长度 46.87/43.43 tokens，avg tool doc 174.56 tokens。

**Instruction 生成（§3.3）。** 三位 NLP/IR 专家先手写 100 条 seed instruction（按 Asai et al. 的 task-aware 格式：桥接 query intent 与 target tool 功能）；以 GPT-4o 为 instruction generator，从 instruction pool（seed + 已生成）随机抽 in-context 样例为每条任务自动生成 instruction，新生成 instruction 反哺 pool（详见 Appendix B Algorithm 1）。最后 5 名专家按四个维度审查：与原 query 相关 / 描述 target tool 特征 / 覆盖所有 target tool / 是否含幻觉。结果：89.2% 通过，10.8% 由专家修订。Kappa 0.743。

**评测协议（§5）。** 指标：NDCG@K、Recall@K、Precision@K、Completeness@K（COLT 提出，全部 target 工具被命中才计 1，否则 0）。两套设置：`w/o inst.`（仅 query）与 `w/ inst.`（query + instruction 拼接）。

**评测的模型（§5.2）。** 五类基线：(i) sparse（BM25s）；(ii) single-task dense 检索器（gtr / contriever / ColBERTv2 / COLT）；(iii) multi-task embedding（all-MiniLM、e5、bge、gte、e5-mistral-7b、GritLM-7B、gte-Qwen2-1.5B-inst.、NV-Embed-v1）；(iv) cross-encoder re-ranker（MonoT5、mxbai-rerank、jina-reranker-v2、bge-reranker-v2-{m3,gemma}）；(v) LLM agent re-ranking（RankGPT 框架 + Mixtral-8x22B / GPT-3.5）。re-ranking 与 LLM agent 的初始召回都来自 NV-Embed-v1。

**ToolRet-train（§7.2）。** 把 §3 的数据收集流程延伸至 ToolACE/APIGen/ToolBench 的训练集，得到 200k+ 训练实例；用 NV-Embed-v1 为每 query 召回 top-10 负样本 `T̂`，构造 `(query, instruction, T, T̂)`。对比损失：

```
L = -(1/|T|) Σ_{t_i ∈ T} log[ exp(sim(I⊕q, t_i)) / (exp(sim(I⊕q, t_i)) + Σ_{t̂_j ∈ T̂} exp(sim(I⊕q, t̂_j))) ]
```

`I ⊕ q` 用特殊 token 拼接；K=10，learning rate 5e-5。消融时去掉 instruction（仅用 query）训练做对照。

**端到端评测（§7）。** 在 ToolBench (G1/G2/G3) 上替换官方 toolset 为 IR 召回结果，用 ToolBench 官方 Pass Rate 度量 GPT-3.5 与 ToolLlama 的端到端表现。

## Eval

**主结果——retrieval 端（Tables 4–5）：**
- 平均 nDCG@10（w/o inst. → w/ inst.）：BM25s 22.32→36.46；ColBERTv2 19.46→23.79；NV-Embed-v1 33.83→42.71；GritLM-7B 30.02→41.13；gte-Qwen2-1.5B-inst. 28.96→45.96。指令相对增益 ~7–17 个 nDCG 点。
- 平均 Completeness@10：最优 NV-Embed-v1 w/ inst. 43.41，即仍有 ~57% 的任务无法把所有目标工具召回到 top-10。
- Re-ranking：bge-reranker-v2-gemma w/ inst. 平均 nDCG@10 = 47.52、Completeness@10 = 48.90，是全表最高，明显高于纯 embedding 模型；但 MonoT5、mxbai-rerank 在 w/o inst. 下相对 NV-Embed-v1 检出退化（28.92 / 24.84 vs 33.83）。
- LLM agent re-ranking：GPT-3.5-turbo-1106 w/ inst. 平均 nDCG@10 38.77 / Completeness@10 38.63，未超过最强 cross-encoder。

**子集差异：** Code 与 Customized 子集普遍优于 Web API 子集——后者因 OpenAPI JSON 文档更长、参数细节更多，召回更难（NV-Embed-v1 w/o inst. 在 Web/Code/Custom 三子集 nDCG@10 分别 31.30 / 29.64 / 40.54）。

**MTEB 相关性（§6.3, Figure 5）：** ToolRet 与 MTEB Retrieval 子集 Pearson 0.790，但绝对分数普遍低 20+ 点；contriever 这类 relevance-oriented 训练的模型在 ToolRet 上表现尤差，被作者解读为"工具检索更需 target-aware 推理"。

**端到端（§7, Figure 6）：**
- Pre-train baseline：bge-large w/ ToolBench-G1 → GPT-3.5 pass rate 50.60（oracle 62.00）。
- ToolRet-train 后：相同模型 NDCG@10 从 ~30 提到 50+，GPT-3.5 与 ToolLlama 的 pass rate 同步抬升 10–20 点（具体数字按子集不同）。
- 消融（Appendix D）：去掉 instruction 训练版仍超过未训练版，但低于带 instruction 版本——instruction 在训练侧也独立有效。

**评测覆盖：** 仅在 ToolBench 子集上做端到端 pass rate（其它 27+ 数据集仅做 retrieval 端评测）；未在 APIBench / Gorilla / ToolACE 上跑端到端。

## Weaknesses

- **One-to-many 标签噪声未量化。** §8 承认跨数据集合并后存在功能近似的多个工具，但只用源数据集的标注作为唯一 ground truth；论文以"现实场景就是要选最合适的"作为辩护，未提供该噪声对绝对分数的影响估计——例如取 top-3 中"任意一个语义等价工具"算命中后的 Completeness@10 退化曲线。这是 33% 这种极低 Completeness 数字的可疑信号 [11: §8]。
- **Re-rank 退化原因未机理化。** MonoT5、mxbai 在 w/o inst. 下使 nDCG 反而下降，bge-reranker-v2-gemma 却带来明显提升；论文给出"limited improvement"的描述但未做错误案例分析（语义对齐 vs 词面对齐？训练域差异？输入长度截断？），读者无法判断哪类 re-ranker 适合工具检索 [11: §6.1]。
- **Instruction 自动化生成的"质量"评估自闭环。** 89.2% 通过率由 5 名专家打 yes/no 得到，但每条任务只评了 ~1 名专家（未显示是否独立打分多次）；GPT-4o 既是生成器也几乎是上游许多 query 的间接生成器（ToolBench、ToolACE 都是 LLM 生成数据），instruction 与 query 处于同一 LLM 语义空间，可能高估真实 user instruction 设计下的检索难度 [11: §3.3, §4.3]。
- **训练侧负采样固定 K=10、负样本由 NV-Embed-v1 召回。** 这意味着所有训练后的模型都在"NV-Embed-v1 认为难"的负样本上学习；其它检索器若有不同失败模式，可能被这套负样本忽略。论文未做 K 扫描或多源负采样消融 [11: §7.2]。
- **端到端只在 ToolBench 上评测。** ToolRet 收录了 30+ 数据集，但 §7 只挑 ToolBench (G1/G2/G3) 做 pass rate；ToolACE、Gorilla、CRAFT 等子集都未上端到端表，使得"提升 IR 模型 → 提升下游 pass rate"的因果链只在 ToolBench 上成立。ToolBench 本身因稳定性问题已被 StableToolBench 修订，论文未说明用的是哪个版本/快照 [11: §7]。
- **Completeness@10 普遍 ≤ 50% 的工程含义被低估。** 即便最强 reranker bge-reranker-v2-gemma 也只 48.90% Completeness@10，意味着一半以上多目标查询无法在 top-10 内全部召回，实际部署若每步只取 top-5 工具进 LLM 上下文将更悲观。论文未给出 K=5 的 Completeness 衰减曲线，读者无法直接外推 ContextLoader 等 5k+ 工具部署的可用性 [11: §5, §6.1]。
- **指令是否泄漏 target tool 信息存在风险。** Instruction 由 GPT-4o 在"知道 target tool"前提下生成（target-aware），即便人写 seed 也是基于知道答案。这种 oracle-aware instruction 在评测时让 IR 模型获得超出部署条件的提示——论文未做"盲写 instruction"（仅给 query 而不告知 target）的 sanity check [11: §3.3, §4.3]。
- **跨数据集的工具数量极不平衡。** ToolBench 单独贡献 13,862 个工具（占 43k 的 ~32%），其余数据集多在百量级；但论文报告的子集平均分按 Web/Code/Custom 三类聚合，没有按"主导数据集 vs 长尾数据集"切分——读者无法判断分数是否被 ToolBench 拉偏 [11: §3.2, Table 6]。
- **ColBERTv2 w/o inst. 表现差于 BM25s 的反直觉现象未深挖。** ColBERTv2 是 late-interaction 强基线，平均 nDCG@10 = 19.46 低于 BM25s 22.32；论文以"domain shift"一笔带过，未做语义/词面分量化，难以指导工程选型 [11: §6.1, Table 4]。
- **训练数据与测试数据的潜在重叠。** ToolRet-train 来自 ToolACE/APIGen/ToolBench 训练集；ToolRet 测试侧也包括 ToolBench 等同源数据集的 test split。论文称已 deduplicate，但未给出 query-level / tool-level 的去重审计指标（如最近邻嵌入相似度阈值）[11: §3, §7.2]。

## Relations

- **builds-on 05_agent_as_a_graph_knowledge_graph [high]：** [05] (Agent-as-a-Graph) 在 LiveMCPBench 上把 ToolRet 类的 dense 检索器（包括 BGE、E5）与 MCP-Zero 都列为基线，并以图结构 + 类型加权 RRF 全面胜出；本文则把同一谱系（dense / cross-encoder / LLM re-rank）拉到统一 7.6k 任务 / 43k 工具的横向对照，量化"现成 IR 模型在工具检索上不堪用"的空间。两者从不同方向佐证：[05] 给出"图结构胜出"的局部上界，[11] 给出"非图基线全集崩盘"的下界，可被联合解读为"工具检索需要专用归纳偏置"。
- **competes-with 07_mcp_zero_active_tool_discovery_for [high]：** 同样针对工具检索，但路径相反——MCP-Zero [07] 押注"让模型主笔结构化请求"取代 query 直接检索，并报告 APIBank Full 集 Q.Retrieval 65–72% / Standard 60–69% / MCP-Zero 90–95%；本文则在 query/query+instruction 两种"被动检索"协议下系统评估 IR 模型，最强 NV-Embed-v1+inst. 平均 nDCG@10 = 42.71（Recall@10 ~46%）。两者证据结合：(a) 仅靠 query 的 dense 检索瓶颈在 ~30 nDCG@10，必须要么训练（[11] 路径）、要么改写请求（[07] 路径）；(b) [11] 的 instruction-augmented 协议本质是把"target-aware 重写"放到查询侧，与 [07] 的"模型主笔请求"是同一思想的两种实现——instruction 由 GPT-4o 离线生成 vs 在推理时由 agent 在线生成。
- **competes-with COLT (Qu et al., 2024) [high]：** 论文显式把 COLT 列为基线（ad-hoc 工具检索数据上训练的 dense 模型），结果在 Web/Code/Custom 三子集上 COLT 都被新训练的 ToolRet-train 模型超过；同时 Completeness@K 这一指标本身来自 COLT。本文把 COLT 的"局部数据集"思路推到"跨 30+ 数据集异质统一格式"。
- **builds-on RAG-MCP / ScaleMCP / Tool2Vec / AnyTool / ToolRerank [med]：** §2 把这些方法纳入 "retrieval-augmented tool selection" 谱系，但实验中并未把它们作为 IR 基线复现——这与 [07] 的批评结构一致：MCP-Zero 也只与 Standard/Q.Retrieval 比，未与 RAG-MCP 同台。两篇论文同时缺这块对比，是该研究方向公认的实验空缺。
- **orthogonal to 03_why_reasoning_fails_to_plan_a [med]：** FLARE [03] 关注步进推理在多步规划上的结构性失败，本文关注"已分解为单步检索"后的 IR 性能。两者层级正交可叠加：planner（FLARE 类前瞻）→ 子步生成 instruction → ToolRet 类 IR 模型召回工具。本文 §8 Future Work (ii) 明确点名希望评估"interleaved retrieval-and-calling"，即把规划与检索耦合评测——正是 FLARE 路径打开的方向。
- **orthogonal to 06_why_do_multi_agent_llm_systems [med]：** MAST [06] 把"工具 schema 注入成本"与"协同失真 FC2"区分；ToolRet 解决前者中的"召回精度"维度（FC1 系统设计子问题），未触及 FC2 跨 agent 错位。但 ToolRet 数据可以作为 FM-2.6 reasoning-action mismatch 的上游归因证据：当 IR Recall@10 < 50% 时，下游 LLM 即便完美决策也找不到工具——这给 MAST 的失败模式分类提供了"上游噪声基线"。
- **orthogonal to 04_rethinking_the_value_of_multi_agent [low]：** OneFlow 论证同质 MAS 可折叠；ToolRet 在单 agent 工具召回内部，与多/单 agent 折叠等价命题不冲突。可被读为"折叠后单 LLM 仍需要面对工具召回这一独立子问题"。
- **orthogonal to 02_graph_of_agents [low]：** GoA 处理 multi-agent 推理输出协调；ToolRet 处理单 agent 工具供给侧召回；层级正交。
- **orthogonal to 01_co_evolving_llm_decision_and_skill [low]：** COS-PLAY 协同进化技能库（学得能力），ToolRet 在静态 tool catalog 上检索已有工具（选择能力）；问题域不同。
- **orthogonal to 08_simulating_human_cognition_heartbeat_driven_autonomous [low]：** HSC 处理常驻 runtime 的认知调度；ToolRet 在被动检索协议下评测，与主动 / 心跳层级无交集。
- **orthogonal to 09_llm_based_multi_agent_blackboard_system [low]：** Blackboard 处理 helper 间响应隔离；ToolRet 处理工具召回，两者层级不同。
- **orthogonal to 10_collaborative_memory_multi_user_memory_sharing [low]：** 协同记忆与工具检索可正交叠加（共享记忆喂给 ContextLoader，ContextLoader 调用 IR 检索工具），但本文未涉及任何记忆机制。

### Relation to thesis

直接命中论题"工具幻觉的语义解耦防御 / ContextLoader 5k–10k 工具规模有效性"研究缺口（thesis §设计上下文 #3）。四点对 Decision Agent 的可操作启示：

1. **现成 dense 检索器不足以支撑 ContextLoader 的 5k–10k 工具规模——必须做工具检索专用训练。** 论文最强的 NV-Embed-v1 在 43k 工具池上 nDCG@10 仅 33.83、Completeness@10 ≤ 35%；即便加 instruction 也只到 42.71 / 43.41。Decision Agent 若直接挂用 BGE/E5 等通用 embedding 服务工具召回，召回上限严重低于业务可接受阈值。**ToolRet-train 这种"指令式 + 多源 tool-use 数据 + K=10 负采样"的微调路径是工程必需，不是 nice-to-have。**

2. **Instruction-augmented 协议与 BKN 解耦 / [07] 主动请求路径同源，证据互相加强。** 本文 instruction = "面向 target tool 的功能描述 + 任务约束"；[07] 的 `<tool_assistant>{server, tool}` 块 = "主动权限域 + 操作类型"。两条独立证据线——[11] 在 30+ 数据集上量化 instruction 对 IR 的增益（+7~17 nDCG 点），[07] 在 APIBank Full 上量化 LLM 主笔请求对精度的增益（+25 个百分点）——共同支持 thesis 的论点："仅靠用户原始 query 做工具检索是错误的设计点；BKN 类的语义结构化前置改写应作为 ContextLoader 的默认 pipeline。"

3. **Completeness@10 普遍 ≤ 50% 是 ContextLoader 设计的一个具体警钟。** ToolRet 测得最强 reranker 在 multi-target query 上 Completeness@10 也只到 48.90%。Decision Agent 的真实业务任务通常需要"检索 → 规划 → 多工具组合调用"，若 IR 阶段就丢掉一半 multi-target，下游 Composer / Plan-Execute 无论怎么努力都补不回来。**这说明 ContextLoader 必须支持 K > 10 的召回 + 二段筛选（语义 retrieval → 结构化 BKN 过滤），而不是单段 top-10。**

4. **强相关但绝对值更低（Pearson 0.79、绝对值低 ~20 点）— "工具检索 ≠ 文档检索"是 testable 的边界。** 论文给出可证伪点的具体数字：若 Decision Agent 的 BKN 工具集复测中，相关性≈0.79 + 绝对值≈低 20 点的关系不再成立（如 BKN 解耦后绝对值反而提升），则可作为 BKN "语义结构化是工具检索专属归纳偏置"的工程级证据；反之若仍相关同档下降，则 BKN 并未带来本质突破，需要回到训练侧（如 ToolRet-train 路径）寻找增益。**这是 thesis "BKN 语义解耦在 5k+ 工具规模下有效性"可证伪点的一个直接实验设计。**

**与 thesis 的张力 / 可证伪点新增：**

- thesis 当前列"BKN 语义解耦在 5k+ 工具规模仍有效"为待验证，证伪条件是"自家 BKN 工具集复测 Recall@5 < 拼接基线"。本文进一步暗示：即便 BKN 给出类型化结构，若不结合 ToolRet-train 这种工具检索专用微调，绝对召回仍可能停留在 ~30–40 nDCG@10 量级。**建议在 thesis 可证伪点表中新增一行**：「ContextLoader 仅做 BKN 类型化 + 通用 embedding（无工具检索专用微调）已足以支撑 5k+ 工具规模」——证伪条件：BGE/E5 等通用 embedding 在 BKN 增强后 Recall@10 仍 < 50%；当前状态：[11] 间接支持该证伪点（通用 embedding 在 43k 工具池 Recall@10 ≤ 52%，且无 BKN 类型化辅助）。

- 证据强度：retrieval 端绝对数字 [high]（30+ 数据集横向、metric 标准化）；instruction 增益 [high]（所有模型一致提升）；端到端 pass rate 增益 [med]（仅 ToolBench 一家、ToolBench 本身有稳定性问题，未在 ToolACE/Gorilla 等上交叉验证）；ToolRet-train 微调可迁移性 [med]（训练数据与 ToolBench 测试同源，去重审计未公开）。
