---
title: "LLM-based Multi-Agent Blackboard System for Information Discovery in Data Science"
paper_id: "paper_arxiv_2510_01285"
source_kind: "arxiv"
source_id: "arxiv:2510.01285"
source_url: "https://arxiv.org/abs/2510.01285"
kind: legacy-topic-read
topic: "decision"
source_note: "notes/active/09_llm_based_multi_agent_blackboard_system.md"
---

# Legacy topic read

This Library read artifact was backfilled from an existing topic note. It preserves historical reading state; it was not produced by the current Library deep-read runner.

## Source topic note

---
zone: active
pin: false
score: 0
dwell: 0
paper: "LLM-based Multi-Agent Blackboard System for Information Discovery in Data Science"
arxiv: "2510.01285"
authors: ["Alireza Salemi", "Mihir Parmar", "Palash Goyal", "Yiwen Song", "Jinsung Yoon", "Hamed Zamani", "Tomas Pfister", "Hamid Palangi"]
year: 2026
venue: "arXiv preprint v2（UMass Amherst + Google Cloud AI Research，2026-01-31）"
note_number: 9
---

## Claims

- 在大规模异构 data lake 上的数据科学问题求解中，把传统 master–slave 多智能体范式替换为 blackboard 架构后，端到端任务成功率相对最强基线获得 13%–57% 的相对提升 [9: §1, §4.2 Table 1]。
- 文件发现（file discovery）任务上，blackboard 架构在 Recall / Precision / F1 三个指标上同时优于 RAG 与 master–slave，相对最强基线最高提升 9% F1 [9: §1, §4.2 Table 2]。
- Blackboard 范式的核心结构差异在于"主智能体不向特定子智能体派发任务，而是在共享 blackboard 上 broadcast 请求；helper agent 独立判断是否参与并写回响应板 β_r" [9: §1, §3]。
- 该设计移除了"中心控制器需要精确知道每个 sub-agent 能力"的强假设，因此在 helper agent 数量增大或能力重叠时仍可工作 [9: §1, §3]。
- 在数据湖规模与 blackboard 相对 master–slave 增益之间存在正相关趋势：在两个最大数据湖（DA-Code 145 文件、KramaBench Astronomy 1556 文件）上分别取得 70%、211% 的相对提升 [9: §4.2 "Scalability Comparison"]。
- 子智能体响应**写入独立的响应板 β_r 而非 blackboard β**，目的是避免 sub-agent 之间相互影响；β_r 由主智能体独占访问 [9: §3 footnote 2]。
- Blackboard 架构的运行时与 RAG / master–slave 基本一致（132.0–145.2 秒/题），但每题美元成本为 RAG 的 ~2.3 倍、master–slave 的 ~1.8 倍；论文将其定位为"有利的 accuracy–cost trade-off" [9: §4.2 "Runtime and Cost Analysis"]。
- 主智能体最大动作上限 T 增大（2 → 10）单调改善性能；T=10 为论文实验默认 [9: §4.2 "Effect of Number of Main Agent's Actions"]。
- 加入 search agent（Google CSE + BeautifulSoup 取页 + 至多 3 轮检索）后端到端表现进一步提升，主智能体在缺少领域知识时主动发起 web 请求 [9: §4.2 "Effect of Search Agent"]。
- 基于 E5-Large 内容嵌入 + KMeans 的语义聚类**优于** Gemini-2.5-Pro 仅按文件名做的默认聚类；增加聚类数量也单调改善性能（实验取 k ∈ {2,4,8,16}）[9: §4.2 "Effect of Clustering Strategy"]。
- 在 Qwen3-Coder-30B / Gemini 2.5 Flash / Gemini 2.5 Pro / Claude 4 Opus 四个 backbone LLM 上结论一致，blackboard 架构均超过对应的三类基线 [9: §4.2 Table 1]。

## Assumptions

- "Helper agent 自主决定是否响应"是通过 prompt 中的指令实现的（"determines whether it can address the request"），论文把它称作"autonomy"——但实质是 LLM 受 prompt 引导的二选一判断，没有任何架构层面的去中心化控制机制（如投票、效用估计、仲裁协议） [9: §3.2 + Appendix C Figure 6]。
- 文件聚类**仅基于文件名**就足以把语义相关的文件分到同一 cluster——这一假设由 Gemini-2.5-Pro 的命名理解能力承担。论文同时承认基于内容嵌入的聚类更好，但默认仍用纯文件名 [9: §3 "Clustering Data Lake"]。
- 每个 file agent 的 offline phase（读取一部分文件 + 生成结构化分析）能为 online phase 的所有可能请求**提前覆盖足够上下文**，不需要在线再次读文件。该假设隐含"helper agent 一次性 sample 即可代表整 cluster" [9: §3.2 "File Agent"]。
- 当多个 helper agent 对同一请求都有响应时，主智能体能正确从冲突或冗余的响应中挑选——论文未给出仲裁机制描述，仅声明"main agent decides whether to use or ignore" [9: §3, Appendix C Figure 5]。
- Search agent 仅服务于"general web-based information retrieval"，本身不响应涉及本地文件的请求；这隐含"领域知识 vs 数据查找"的二分边界清晰可分 [9: §3.2 "Search Agent"]。
- 评测使用的 KramaBench / DSBench / DA-Code 经过人工筛除"无足够 hint 进行 file discovery"的题目（DSBench 保留 253 题、DA-Code 保留 91 题），筛选保留题已构成"discovery 任务的合理子集"——这一筛选本身没有 inter-rater 一致性度量 [9: Appendix A "Filtering"]。
- 用 Gemini 2.5 Pro 当 DSBench 的 LLM-as-Judge 不会引入系统性偏置，即使待评测系统也用 Gemini 2.5 Pro 作为 backbone（实验中确实存在这种重合）[9: §4.1 "Evaluation"]。
- "Blackboard 上的请求 + 响应板 β_r 分离"足以避免 sub-agent 之间的相互影响；helper agent 间不需要相互可见 [9: §3 footnote 2]。

## Method

**核心论点：把多智能体通信从"中心派发"改为"广播 + 自荐"，借此在大规模异构数据湖上免除"中心需知子智能体能力"的强假设。**

**❶ 数据湖分区（offline）。** 数据湖 𝔻 = {d_i}_{i=1..N} 被分为 C 个 cluster，每个 cluster 𝔻_i 交给一个 file agent π_{f_i}。**默认聚类**：把所有文件名喂给 Gemini-2.5-Pro，按命名相似度产出聚类（prompt 见 Figure 4）。**对照聚类**：用 E5-Large 编码文件内容 + KMeans，k ∈ {2,4,8,16}，仅在文件数 ≥ 100 的数据集（Astronomy / Legal / DA-Code）上做。

**❷ File Agent 的两阶段（§3.2）。**
- Offline：从 𝔻_i 中选若干文件读取内容（不必读全部，论文 footnote 4 给出"同类时间序列文件用代表性样本"启发式），生成 cluster 级分析（结构、预处理、加载方式）。
- Online：监听 blackboard，收到主智能体请求 r 后，**先判断能否贡献**；若能，则给出"哪些文件相关 / Python 加载代码 / 库 / 处理步骤 / 数据样本"的详细回应，写入 β_r。

**❸ Search Agent（§3.2）。** 监听 β，收到 r 后**自判可行性**——明确**不响应涉及本地文件的请求**。最多 T_search=3 轮：每轮用 LLM 生成一组 query → Google CSE 各取 k=3 篇网页 → BeautifulSoup 抽正文 → 自评是否足够 → 足够则写 β_r，否则继续生成新 query。CSE 配置上排除了所有数据集相关网站防数据泄漏。

**❹ 主智能体（§3.1，ReAct 框架）。** 主智能体在最多 T=10 步内，每步从 5 个动作中选一个：
- `plan` / `reason`：内部步骤，只更新 history，不影响外部环境。
- `run_code`：调 Python 解释器执行，回收 stdout/error，用于探查文件内容、试加载、实验性处理。
- `request_help`：把请求 r 写入 blackboard β，**等待所有 helper agent 的响应在 β_r 上汇集**后一并回灌主智能体上下文。论文强调请求执行被 helper agent **并行**处理，这是与 ReAct 串行的关键差异。
- `answer`：终止动作，输出最终 Python 程序 p。

**❺ 通信与隔离（§3 footnote 2）。** Helper agent 响应**只写入 β_r**而非 β，目的是防止 sub-agent 间相互影响。β 是 broadcast 通道，β_r 是私有响应板（主智能体独占）。这是与"shared memory MAS"范式的区别点：shared memory 假设所有 agent 共享同一空间，blackboard 强制响应隔离。

**❻ 与 Han & Zhang (2025) 同名工作的差异（Appendix B）。** 论文承认存在"另一篇 LLM blackboard 架构"工作，但对方的 blackboard 实质是 shared memory，主智能体仍负责派发；本文主张"完全不派发"。

**❼ 实验配置（§4.1）。**
- LLM：Gemini 2.5 Pro / 2.5 Flash（Vertex AI）+ Claude 4 Opus + Qwen3-Coder-30B（vLLM, 2×A100 80GB）。
- 采样：top-p（nucleus），温度 0.1，单步至多 8,192 tokens。
- 数据：KramaBench（104q / 1746 files / 27 clusters）、DSBench filtered（253q / 48 files / 12 clusters）、DA-Code filtered（91q / 145 files / 26 clusters）。
- 评测：KramaBench / DA-Code 用各自官方脚本；DSBench 用 Gemini 2.5 Pro 做 LLM-as-Judge（Figure 3 prompt，二值打分）。
- File discovery：从生成的程序中提取实际加载的文件路径，与 ground-truth 文件集合比对得 P/R/F1（Table 2，仅 Gemini 2.5 Pro backbone）。
- 基线：DS-GRU（KramaBench 原作 baseline，把所有文件塞 prompt + 自纠正循环）；RAG（E5-Large top-5 + ReAct，仅保留 search 子动作）；Master–Slave（与主智能体同 ReAct，但 `request_help` 替换为按名直接调用某个 sub-agent）；进阶基线 Data Interpreter（Hong et al., 2025）+ AutoGen（Wu et al., 2023）。

**❽ 关键消融（§4.2 + Figure 2）。** (a) T ∈ {2,4,6,8,10} 单调改善；(b) 去掉 search agent 在 KramaBench 上整体下降；(c) 内容聚类（E5-Large + KMeans）优于文件名聚类（Gemini-2.5-Pro）；(d) k ∈ {2,4,8,16} 单调改善；(e) 50 题随机抽样的成本-时延对比，runtime 132–145 秒/题、cost 为 RAG 的 ~2.3×、master–slave 的 ~1.8×。

## Eval

**实验规模：**
- 3 数据集合计 ~448 题，4 个 backbone LLM（共 16 行结果，Table 1）。
- File discovery 仅在 Gemini 2.5 Pro 上做（Table 2）。
- 进阶基线（Data Interpreter / AutoGen）仅在 KramaBench 上做（Figure 2A）。

**主结果（Table 1，端到端 macro 平均）：**
| Backbone | DS-GRU | RAG | Master–Slave | **Blackboard** |
|---|---|---|---|---|
| Qwen3-Coder | 1.21% | 3.87% | 5.03% | **7.90%** |
| Gemini 2.5 Flash | 4.32% | 14.52% | 14.49% | **16.54%** |
| Gemini 2.5 Pro | 7.71% | 18.51% | 24.01% | **28.53%** |
| Claude 4 Opus | 4.11% | 23.00% | 27.63% | **31.43%** |

最强配置（Claude 4 Opus + Blackboard）的 macro 平均仍只有 31.43%——任务本身远未饱和。

**File discovery（Table 2，Gemini 2.5 Pro，macro F1）：**
- RAG: 0.247 / Master–Slave: 0.513 / **Blackboard: 0.561**
- 在 KramaBench 6 个子任务中，Blackboard 在 4/6 子任务上 F1 最高；Wildfire / Legal 子任务 master–slave 接近或略胜。

**Scalability claim（§4.2 + Figure 20）：** 把"data lake 文件数"对"Blackboard vs Master–Slave 相对增益"做散点回归——正相关。最大两点：DA-Code（145 files, +70%）、Astronomy（1556 files, +211%）。但论文自己承认 master–slave baseline 在 Astronomy 上仅 3.16%–11.52%，分母极小放大相对增益。

**成本对比（§4.2，KramaBench 50 题抽样）：**
- Runtime：RAG 132s / Master–Slave 138s / Blackboard 145s（差距 < 10%）。
- Cost：Blackboard ≈ 2.3× RAG，1.8× Master–Slave，但端到端表现优 54.1% / 18.8%。

**指标缺失：**
- File discovery 仅在 Gemini 2.5 Pro 上做，未做 backbone 健壮性检查——其他三个模型在文件发现上是否同样领先未知。
- 进阶基线（Data Interpreter / AutoGen）只在 KramaBench 单数据集上对比。
- "并行"声明只在 §4.2 文字处给出，未量化"几个 helper agent 平均同时响应"的分布。

## Weaknesses

- **"Autonomy"是 prompt 修辞而非架构属性。** 论文反复强调 helper agent "autonomously decide whether to respond"，但 Figure 6 / 7 / 8 的 prompt 显示这就是一个 LLM 的二选一判断（"first determines whether it is capable of addressing the request"）。这与 master–slave 的"主智能体调谁"在信息论上等价——都是 LLM 通过 prompt 选择是否执行任务，没有引入任何分布式控制、投票、效用估计、或 commitment 协议。论文应称为"自荐式 prompt 模式"而非"autonomous agents" [9: §3.2 vs Appendix C Figure 6]。
- **"主智能体决定使用或忽略响应"机制完全不透明。** §3 仅一句"main agent then decides whether to use or ignore the provided information"，但当多个 helper agent 在 β_r 上给出**冲突或冗余**的响应时（论文 case study Figure 14 显示 8 个 helper 中有 3 个响应同一请求），主智能体如何仲裁、是否会被多余信息分散注意、是否会出现错误响应"压过"正确响应——全部缺失。这是 blackboard 范式的核心风险点，论文回避了 [9: §3 + Appendix E Figure 14]。
- **成本分析隐藏 offline 阶段的 file agent 读取成本。** §4.2 的 cost 对比仅覆盖**单题 online**调用。但每个 file agent 的 offline 阶段需读取 cluster 内代表性文件并生成分析报告——KramaBench 有 27 个 cluster、DA-Code 26 个、合计 ~65 次 file-content 读取 + LLM 分析，成本被均摊到所有题目。当数据湖规模放大到企业 5k–10k 工具/文件时，offline cost 不再可忽略，且**每次数据湖更新都需重做**。论文未做 offline cost 摊销分析 [9: §3.2 vs §4.2 "Runtime and Cost Analysis"]。
- **scalability 回归基于 2 个极端点。** "data lake 越大 blackboard 收益越大"的核心 scaling 论据是 Figure 20 的散点 + 回归线，但**主导斜率的两个点（DA-Code 145 / Astronomy 1556）的 Master–Slave baseline 都很低**（Astronomy Qwen 上 master–slave 仅 3.16%）。相对增益（211%）的分母在小绝对值时被放大，结论易被回归不稳定性破坏。论文未给 bootstrap CI、未做 leave-one-out 回归检验 [9: §4.2 "Scalability Comparison" + Appendix F Figure 20]。
- **Cluster 数量上限远低于企业目标场景。** 实验最大 cluster 数 27（KramaBench）、k ∈ {2,4,8,16} 的内容聚类实验。Decision Agent 的 5k–10k 工具集需要数百乃至上千 cluster，每个 cluster 一个 file agent → online 时需并行评估上千 LLM 是否响应——blackboard 论点的 broadcast 成本随 helper 数量线性放大，论文从未触及这一上界 [9: §3 Table 3 vs thesis goal 4]。
- **默认聚类策略被自家消融实验否定。** §4.2 "Effect of Clustering Strategy" 显示 E5-Large + KMeans 内容聚类**优于** Gemini-2.5-Pro 文件名聚类，但论文主实验全部用文件名聚类做默认配置——这意味着 Table 1 的 Blackboard 数字是 sub-optimal 的。论文没有用最佳聚类配置重做主实验，"端到端 +13–57%" 数字的稳健性受影响 [9: §4.2 Table 1 vs §4.2 "Effect of Clustering Strategy"]。
- **Master–Slave baseline 不是 SOTA orchestration。** Master–Slave baseline 用同样的 ReAct 主智能体，仅把 `request_help` 换成"指名调用某 sub-agent"——这是一个**刻意削弱**的 baseline。真实工业 orchestration（LangGraph supervisor / CrewAI hierarchical / OneFlow 单 agent 折叠）都没有被纳入对比。"blackboard 优于 master–slave"未必意味着"blackboard 优于工业最佳实践" [9: §4.1 "Baselines"]。
- **响应板 β_r 的"防止 cross-influence"声明未被消融验证。** §3 footnote 2 主张"sub-agent 响应不写 β 是为防互相影响"，但论文从未做"响应写 β（共享）vs 写 β_r（隔离）"的消融。该声明停留在设计直觉层面，无实证 [9: §3 footnote 2]。
- **DSBench 的 LLM-as-Judge 与 backbone 重合带来潜在偏置。** DSBench 评测 judge 是 Gemini 2.5 Pro，主实验中 Gemini 2.5 Pro 也是被评 backbone 之一。Table 1 行 12（Gemini 2.5 Pro Blackboard）DSBench 38.73% 是否被自评偏置抬高，论文未做交叉 judge（如让 Claude 4 Opus 复评）[9: §4.1 "Evaluation" + Figure 3]。
- **筛选过的 DSBench / DA-Code 与原版不可比。** 论文承认从 DSBench 与 DA-Code 中**手工筛除**"无足够 hint 进行 discovery"的题目（DSBench 保留 253q、DA-Code 保留 91q），筛选 by hand without inter-rater agreement。这意味着所有 DSBench / DA-Code 上的数字都不能与文献中其他工作直接比较，且筛选可能系统性保留对 Blackboard 友好的题目（如"题目里给了文件名提示"的子集）[9: Appendix A "Filtering"]。
- **Search agent 设计假设"领域知识全在 web"。** §3.2 search agent 仅查 Google CSE。企业场景下大量领域知识在内部 KB / 内部 wiki / 受控数据库中，且这些不是 web-fetchable 的。Search agent 模式无法直接平移到企业治理场景；论文从未讨论 [9: §3.2 vs thesis Design Context]。
- **没有失败模式分类。** 论文未尝试用 MAST（[6]）或类似失败模式分类工具诊断 Blackboard 与 Master–Slave 的失败差异。"Blackboard 减少了什么 / 引入了什么新失败模式"完全未知。Case study（Figure 15）只展示一个 cherry-picked 成功对比 [9: §4.2 "Case Studies"]。
- **"并行"性能优势未量化为 wall-clock 节省。** 论文声称 Blackboard 并行处理 helper agent → runtime 与串行 baseline 持平。但 Figure 2(D,E) 显示三者差距仅 ~10%；如果并行真实生效，Blackboard 应显著快于 Master–Slave（同样 ReAct 但串行）——结果不显著反而暗示"并行收益被多 helper 一起回灌的 prompt 膨胀抵消"。论文未深入分析 [9: §4.2 "Runtime and Cost Analysis"]。

## Relations

- **competes-with 06_why_do_multi_agent_llm_systems [med]**：MAST（n=1642 traces）把 master–slave 类 MAS 的协调失败归类为 FC2 Inter-Agent Misalignment（占 32.3%），并将"信息隐瞒 / 错误任务派发 / 工具误用"列为典型失败模式。本论文的 Blackboard 通过"主智能体不派发，helper 自荐 + 响应隔离"在**设计层面**对应 FC2 的几种触发条件（FM-2.1 Withholding Information / FM-2.4 Information Sharing Errors / FM-2.6 Reasoning-Action Mismatch）。但本论文未引用 MAST、未把 MAST 失败模式作为评测靶点；"Blackboard 缓解了哪些 MAST 失败模式 / 引入了哪些新失败模式"是空白。两者在"master–slave 是结构性瓶颈"上互证，方法论层面 competes（设计修复 vs 失败分类） [high 相互覆盖问题域]。
- **competes-with 04_rethinking_the_value_of_multi_agent (OneFlow) [med]**：OneFlow 论证同质 MAS 可被折叠为单 LLM 顺序执行——但 OneFlow 的实验前提是任务不超出单 LLM 上下文。本论文 DS-GRU baseline（"把所有文件塞 prompt"，对应 OneFlow 的"folded MAS"）在 Qwen3 上仅 1.21%、在 Claude 4 Opus 上仅 4.11%——明确**反证** OneFlow 在"上下文超限的数据发现"场景下的折叠主张。两者并不矛盾，而是边界条件互补：OneFlow 适用于上下文充足任务，Blackboard 适用于上下文必然溢出场景 [med，因为论文未引用 OneFlow，关系是我推断]。
- **builds-on 07_mcp_zero_active_tool_discovery_for [med]**：MCP-Zero 让模型主动写结构化请求驱动工具检索；本论文让主智能体在 blackboard 写结构化请求驱动文件 / 信息发现。两者共享"模型主笔请求 + 类型/语义路由"骨架，但层级不同：MCP-Zero 在工具召回层（候选 ≤ 2,797），Blackboard 在数据湖文件层（候选 ≤ 1,556）。两者在"主动请求改写"上同向证据，但 MCP-Zero 已揭示强模型 baseline 饱和——本论文 Claude 4 Opus 的 Blackboard 仅取得 macro 31.43%，远未饱和，说明"数据发现 vs 工具检索"难度结构不同 [med]。
- **builds-on 05_agent_as_a_graph [med]**：Agent-as-a-Graph 用 KG 类型化在工具/agent catalog 上做 typed retrieval（Recall@5 +14.9%）；本论文用 LLM 文件名聚类做 cluster-typed routing。两者在"用结构化分区改善 flat retrieval"上同向，但本论文聚类不是基于显式 schema 的"类型"而是 LLM 的命名相似性判断——Agent-as-a-Graph 的 BKN 同构性更强。本论文给出的"内容嵌入 + KMeans 优于命名聚类"消融，与 BKN 类型化思路是叠加而非冲突的工程证据 [med]。
- **orthogonal to 03_why_reasoning_fails_to_plan_a (FLARE) [low]**：本论文的主智能体仍是 ReAct（步进推理），按 FLARE 的论证它在多步规划上是贪心策略。但本论文任务（数据发现）的核心难点在"信息检索 + 分区路由"而非"长链规划"，所以 FLARE 的结构性局限论据在此适用度低。两者层级不同：FLARE 改造主智能体单步推理深度，Blackboard 改造跨智能体通信结构 [low]。
- **orthogonal to 02_graph_of_agents [low]**：GoA 处理多智能体输出的图结构协调，本论文处理多智能体请求广播。不同 MAS 拓扑维度。
- **orthogonal to 08_simulating_human_cognition_heartbeat [low]**：HSC 处理"何时进入推理"的时序调度，本论文处理"如何在多 agent 间路由请求"的空间结构。问题域正交。
- **orthogonal to 01_co_evolving_llm_decision_and_skill [low]**：COS-PLAY 关注 skill 库进化，本论文关注 file/info 发现路由。

### Relation to thesis

直接命中 thesis Design Context 的两个 design goals——**goal 3 (Shared Workspace) 与 goal 4 (Composer)**——而且与 thesis 当前关于"协同失真 / 治理优先"的两条核心论点都有正向工程证据。

**1. Blackboard ≅ Shared Workspace 的工程化原型 [high 关联度]。** thesis goal 3（Session/Task 级共享工作区 + 引用传递机制 + ISF 权限控制）的设计目标——"根本性降低多 Agent 协作的上下文压力"——在本论文得到一个具体范式：(a) 主智能体把请求广播到 β（共享区），(b) helper agent 写回 β_r（独占响应板），(c) helper 之间不可见。**关键工程启示：在 Decision Agent 的 Shared Workspace 设计中，"agents 互相不可见 + 主 agent 仲裁"是一个值得复制的 default 选项**——它把共享降到"按需广播 + 隔离响应"两条窄通道，而非"全局共享 memory"。这与 thesis "Shared Workspace + BKN 共享 Memory 对 FC2 的工程级缓解"目标一致，且 §3 footnote 2 给出了"为什么不写 β（防 cross-influence）"的设计动机——可直接进入 Decision Agent 的 Shared Workspace 设计文档作为引用证据。

**2. Composer 的"无中心能力注册表"设计获得初步支持 [med 关联度]。** thesis goal 4（Composer 基于已有 Agent/Template/Skill 的多智能体组合，明确不做端到端 spawn / DAG 引擎 / A2A 协议）的核心反直觉点是"主 agent 不必精确知道每个被组合 agent 的能力 detail"。本论文论据线："blackboard 移除中心控制器需要精确知道每个 sub-agent 能力的强假设"在 4 个 backbone LLM × 3 数据集上得到一致支持。**Decision Agent Composer 可借鉴的具体机制：让 Composer 派生的 Plan 节点以"语义请求"形式发布，候选 Agent/Template 通过自身 BKN-anchored capability summary 自荐，由 Composer plan 节点仲裁选 1–N 个。** 这避免了 Composer 维护"全平台 Agent 能力索引"的负担，与 thesis "在已有能力空间内组合，避免向通用 DAG 编排漂移"的非目标定位互洽。

**3. 治理优先视角下的成本与可审计性 trade-off [low 关联度]。** Blackboard 端到端比 RAG 贵 2.3×、比 Master–Slave 贵 1.8×。Decision Agent 在企业治理场景下需"每次广播 + 每条响应"都进入 TraceAI——可审计性目标与本论文"主 agent 私有 β_r"设计存在小张力：β_r 私有意味着 helper agent 的响应不进入跨 agent trace 视图，这与 thesis "TraceAI-first / action provenance must be explicit" 不直接对齐。**Decision Agent 在落地 Shared Workspace 时需补充设计："β_r 私有" 是否需要替换为 "β_r 私有 + 全量 trace 镜像到 TraceAI"，使治理审计独立于工程隔离**。本论文未涉及该治理维度，需在 Decision Agent 设计文档中显式补齐。

**4. 可证伪点的更新 [med]。** thesis "可证伪点追踪表"中的 "Shared Workspace 能显著降低 FC2 协同失真" 当前状态"未验证"——本论文给出**间接支持**（Blackboard ≅ Shared Workspace 的一个设计点 + 设计层缓解 FC2，但论文本身**不引用 MAST、不做失败模式量化**）。可把状态从"未验证"更新为"间接工程证据，缺直接失败模式量化"。同时补充新的可证伪点：**"Decision Agent Composer 的'无能力注册表'设计在企业 5k–10k Agent 规模仍成立"**——本论文最大 helper agent 数为 27，远低于该规模，证据支持仅适用于 ≤30 helper 的小规模。

**5. Goal 2（Heartbeat + Cron）证据中性 [low]。** 本论文主智能体仍是请求驱动的 ReAct，与"主动调度 / 心跳触发"无关。对 thesis 中"主动 Agent 与治理优先的结构性张力"既不构成支持也不构成证伪。

**证据强度：** 论文整体[med]——经同行评议前的 v2 版本，方法论清晰、结果在 4 backbone × 3 数据集上一致，但默认聚类与最佳聚类不一致、master–slave baseline 较弱、企业级规模未触及、cost 分析忽略 offline 摊销。**应作为 Decision Agent goal 3/4 的工程参考与设计原型来源**，但不应作为 Composer 在企业 5k–10k Agent 规模下"无能力注册表"主张的直接证据。
