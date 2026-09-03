---
title: "Collaborative Memory: Multi-User Memory Sharing in LLM Agents with Dynamic Access Control"
paper_id: "paper_arxiv_2505_18279"
source_kind: "arxiv"
source_id: "arxiv:2505.18279"
source_url: "https://arxiv.org/abs/2505.18279"
kind: legacy-topic-read
topic: "decision"
source_note: "notes/active/10_collaborative_memory_multi_user_memory_sharing.md"
---

# Legacy topic read

This Library read artifact was backfilled from an existing topic note. It preserves historical reading state; it was not produced by the current Library deep-read runner.

## Source topic note

---
zone: active
pin: false
score: 0
dwell: 0
paper: "Collaborative Memory: Multi-User Memory Sharing in LLM Agents with Dynamic Access Control"
arxiv: "2505.18279"
authors: ["Alireza Rezazadeh", "Zichao Li", "Ange Lou", "Yuying Zhao", "Wei Wei", "Yujia Bao"]
year: 2025
venue: "arXiv preprint v1（Center for Advanced AI, Accenture，2025-05-23）"
note_number: 10
---

## Claims

- 在多用户多 agent 场景下，把单用户 monolithic memory 替换为带访问控制的双层（private + shared）memory 后，5 用户 / 6 specialist agent / 6 domain knowledge base 的全协作配置在 MultiHop-RAG 上：(a) 准确率维持在 0.90+ 与 isolated memory 相当；(b) 资源调用数下降——overlap 50% 时降 61%、overlap 75% 时降 59% [10: §5.1, Figure 3]。
- 时变访问关系可被两张 bipartite graph 完整刻画：G_UA(t) ⊆ U×A 表示用户-agent 授权、G_AR(t) ⊆ A×R 表示 agent-resource 授权；动态增删边即对应授权与吊销 [10: §3.1, Eq. 1-2]。
- 每条 memory fragment m 携带不可变 provenance 四元组 (T(m), U(m), A(m), R(m))——分别为创建时间、贡献用户、贡献 agent 集合、被访问 resource 集合；provenance 用于支持 retrospective 权限重检 [10: §3.2]。
- 可访问 memory 由 fragment provenance 与当前授权的交集决定：M(u,a,t) = {m | A(m) ⊆ A(u,t) ∧ R(m) ⊆ R(a,t)}——即只有 fragment 所涉 agent 与 resource 当前都仍在该用户授权范围内才可读 [10: §3.2, Eq. 3]。
- Read 与 write 是两条独立 policy，可在四个粒度（global / per-user / per-agent / per-time）独立配置；本文实验采用 simple read（直接返回符合权限的 fragment）+ transformation write（用 LLM 系统级 prompt 在写入前做 redact / anonymize / paraphrase）[10: §3.4 "Granularity and Dynamics", §4 "Policy Instantiation"]。
- 在 4 个用户角色 / 4 个 specialist agent 的非对称授权场景（200 条业务查询，仅 Strategy Director 可见全部 agent）下，asymmetric collaboration 比 fully isolated 显著降低每用户 resource 调用数；Strategy Director 由于已经吸收前置用户的部分共享 fragment，在最终决策合成阶段冗余调用进一步减少 [10: §5.2, Figure 4]。
- 在动态访问图场景（5 用户 / 5 RAG agent / 5 科学领域 / t0–t8 共 9 时段，先授 0→25 边再吊回）下：(a) 准确率随授权扩张上升、随吊销下降，与可用资源量强耦合；(b) agent 调用数随授权同步起伏；(c) **resource 调用数随时间整体下降**，因 memory fragment 复用替代了重复外部检索 [10: §5.3, Figure 5]。
- 访问矩阵（Figure 6）的逐格使用计数证实在所有 t1–t8 时段内**没有出现授权图之外的 user-agent / agent-resource 调用**——即 access graph 的硬约束被严格遵守 [10: §5.3 + Figure 6]。
- 该框架与 RBAC / ABAC 在策略模块化上同构，但显式建模 user / agent / resource 三方之间的时变 bipartite 关系，作者主张在已有 LLM memory 系统中尚无同时整合"provenance-aware fragment + 时变 access policy"的工作 [10: §2 "Access-control models"]。

## Assumptions

- "Provenance 不可变（immutable）"是设计性声明，论文实现层用普通 metadata 字段承载 (T, U, A, R)——没有任何加密、签名、链式校验或防篡改机制；隐含"系统内部所有写入路径可信"假设 [10: §3.2 vs §4 "Memory Encoder"]。
- Write policy 用 LLM 完成 redact / anonymize / paraphrase，隐含"LLM 转写正确执行隐私控制"——但 LLM 本质概率性，本文 §6 自陈"probabilistic nature of these models can lead to occasional hallucinations or policy breaches"。即"安全保证由 LLM 概率性输出承担"被作为可接受假设 [10: §4 "Policy Instantiation", §6]。
- Coordinator LLM（gpt-4o）能正确根据查询语义和当前用户授权列表选出合适 agent，agent selection 的失败模式（错选 / 漏选 / 过度调用）未被独立评测——评测体系直接把 coordinator 作为系统的一部分，而非独立模块 [10: §4 "Multi-Agent Interaction Loop", §B.1]。
- top-k retrieval（kuser=10, kcross=10 在 S1；kuser=20, kcross=20 在 S2）足以覆盖相关 fragment——但隐含"shared memory 增长不会让 top-k 检索精度退化"，本文未做 fragment 数量增长下的检索精度曲线 [10: §5.1, §5.2]。
- 同一查询在不同 user 视角下检索得到的"权限过滤后 fragment 视图"之间无冲突或冲突不重要——论文未涉及 fragment 间冲突合并（例如同一事实由两个 user 写入但内容矛盾）的处理机制 [10: §3 整章]。
- Memory 单调增长，无 decay / forgetting / 过期机制；隐含"在评测时长内 fragment 总量可控"——这与 thesis 在企业生产环境下"layered memory + 治理与遗忘"的需求是错位的 [10: §3.4 vs thesis Design Context goal 5]。
- Scenario 2 的 200 条业务查询由 GPT-4o 合成（§D.1 prompt 公开），评测所用 agent / coordinator / aggregator 也都是 GPT-4o——存在"生成数据 与 评测系统 共享底座 LLM"的潜在自一致偏置；论文未做交叉模型 sanity check [10: §D.1, §4]。
- Scenario 1 用 LLM-as-Judge（gpt-4o）打分 MultiHop-RAG 答案的正确性；judge 与 system 同模型，且 §4 未给出 inter-judge 一致性度量 [10: §4 "Metrics", §5.1 "Task"]。
- Scenario 3 把"resource 调用数下降"解读为"fragment 复用替代外部检索"——但论文未做 ablation（关闭 memory 后 resource 调用数会怎样）来直接量化复用率；解读基于 narrative 推理 [10: §5.3]。

## Method

**核心论点：把"单用户 monolithic memory"扩展为"多用户 + 时变 bipartite 授权 + 双层 memory + provenance-aware policy"，以在保留单用户 long-term memory 收益的同时支持跨用户知识转移而不违反权限。**

**❶ 双层 memory（§3.2）。** 总 memory M = M_private ∪ M_shared。
- M_private(u, t)：仅 user u 的私有 fragment，按贡献用户隔离。
- M_shared(a, t)：跨用户的可共享 fragment，按贡献 agent 组织（"agent a 在为多个用户服务过程中产出的可共享内容"）。
- 每条 fragment 携带不可变 provenance：T(m) 时间戳、U(m) 贡献用户、A(m) 贡献 agent 集合、R(m) 创建时使用过的 resource 集合。

**❷ 时变 bipartite 授权图（§3.1）。** G_UA(t) ⊆ U×A、G_AR(t) ⊆ A×R。每个时刻 t 由这两张图导出 A(u, t)（用户 u 可调用的 agent 集合）与 R(a, t)（agent a 可访问的 resource 集合）。授权变化通过加边 / 删边表达。

**❸ 可读 fragment 集合（§3.2 Eq. 3）。** 当 agent a 在 t 时刻为 user u 服务时：
- M(u, a, t) = { m | A(m) ⊆ A(u, t) ∧ R(m) ⊆ R(a, t) }
- 含义：只有 fragment 所涉的所有 agent 当前仍在 u 的授权列表内，且所涉的所有 resource 当前仍在 a 的授权列表内，该 fragment 才可见。
- 该可见集可包含三类来源：(i) u 自己与其他 agent 的私有历史；(ii) a 自己服务其他 user 时产出的共享 fragment；(iii) 其他 user 与 u 共同授权的 agent 互动时产生的、且通过 a 的 resource 权限筛选的 fragment。

**❹ Read / Write policy（§3.3, §3.4）。**
- Read：π^read_{u,a,t}(M(u,a,t))——再做一层基于内容的筛选（如 keyword filter、top-k embedding similarity）；本文实验取"simple read"，直接返回符合权限的 top-k fragment（kuser + kcross）。
- Write：拆 π^{write/private}_{u,a,t} 与 π^{write/shared}_{u,a,t} 两支独立。前者把 (yu,a,t, M_private(u,t)) 映射到 M_private 新版本；后者把 (yu,a,t, M_shared(u,t)) 映射到 M_shared 新版本。
- Policy 粒度：global / per-user / per-agent / per-time 四级，可独立配置；本文实验取"transformation write"——用 LLM 配 prompt 把 raw response 转写为 KV 形式，private 与 shared 用不同 prompt（Table 1：shared 版强制移除"用户特定细节"和"个性化示例"，旨在抽出通用知识）。

**❺ 多 agent 交互回路（§4）。**
- Coordinator（gpt-4o）：每轮接收 (用户查询, 用户可见 agent 的 specialization 字符串, 历史) → 输出 JSON {agent: ID, subquery: ...}，可输出 {stop: true} 终止。
- Agent：用 OpenAI function-calling 调外部 resource（如 KB 检索）；先查 memory，不足再调 resource。
- Memory Encoder：对话 trace → LLM 抽 KV fragment → 标注 provenance → 入库。
- Memory Retrieval：subquery 嵌入（text-embedding-3-large）+ cosine similarity，top-kuser 私有 + top-kcross 跨用户（仅取 provenance 通过权限过滤的）。
- Aggregator：所有子查询响应序列 → 合成最终回答。

**❻ 三场景实验（§5）。**
- S1（Fully Collaborative，§5.1）：MultiHop-RAG，609 篇英文新闻 / 6 domain（entertainment / business / sports / technology / health / science）/ 2,556 条 multi-hop 查询。5 用户全权限连接 6 domain agent，每 agent 独占一个 domain KB。用 KMeans 把查询聚 10 簇，按 overlap rate ρ ∈ {0%, 25%, 50%, 75%} 选"全局共享查询"，剩余唯一分配给单用户。kuser = kcross = 10。对照：collaborative vs isolated。
- S2（Asymmetric，§5.2）：GPT-4o 合成 200 条业务项目查询。4 用户角色（Market Researcher / Financial Analyst / Logistics Lead / Strategy Director）非对称连接 4 specialist agent（market / finance / logistics / decision），仅 Strategy Director 可调全部 agent。每个 agent 接独立 resource（market_kb / finance_forecaster / logistics_comparator / strategic_computation）。kuser = kcross = 20。无 ground truth，只测 resource utilization。
- S3（Dynamic，§5.3）：SciQAG，5 科学类目（chemistry analytical / energy & fuels / mat. sci. ceramics / mat. sci. paper & wood / phys. math），每类目 1 个 RAG agent + 1 个独占 corpus。5 用户每人每类目 4 题，共 100 题。授权图 t0=空 → t1..t4 逐步加边到 25 → t5..t8 逐步吊销；每 100 题为一个时段。

**❼ 评测指标（§4）。**
- Accuracy：与 ground truth 比对的 LLM-as-Judge（gpt-4o）平均得分。
- Agent Utilization：每查询调用的 distinct agent 数。
- Resource Utilization：每查询的 KB / API 调用次数（论文用此替代延迟，因 deep research 类 agent 延迟波动 5–30 分钟）。

## Eval

**S1 主结果（§5.1, Figure 3，5 用户 / 6 agent / 2556 查询）：**
- Collaborative memory accuracy ≥ 0.90（与 isolated 相近），所有 overlap rate 上稳定。
- Resource utilization：collaborative 在 50% overlap 下比 isolated 降 **61%**、75% overlap 降 **59%**；0% / 25% overlap 仍有降幅但较小（曲线收敛慢）。
- Agent utilization：两种配置都 ~1.5–1.8 agent/query，差距 < 10%。
- 按查询类型（§C.4, Figure 8）拆分：inference / comparison / temporal 三类的 collaborative 收益排序为 temporal > comparison > inference——temporal 类对 fragment 复用最敏感。

**S2 结果（§5.2, Figure 4，4 用户 / 4 agent / 200 查询，无准确率）：**
- 4 个用户角色的 resource usage 直方图均向左移（用 collaborative 的均值低于 isolated）；Strategy Director 由于在最后整合，复用前 3 个用户产生的 shared fragment 收益最大（但论文未给出具体数字）。
- 没有给出"协作降低多少 percentage 的 resource calls"的汇总值，只给出 4 张直方图。

**S3 结果（§5.3, Figure 5–6，5 用户 / 5 agent / 100 查询，t0–t8 共 9 时段）：**
- Accuracy 曲线：t0~0.30 → t4 峰值 ~0.62 → t8 回落到 ~0.40，与"User-Agent 授权图边数"曲线（t0=0, t4=25, t8=0）形态吻合。
- Agent utilization 与边数曲线同构。
- Resource utilization 整体下降（即使授权吊销期），归因为 memory fragment 复用。
- Memory fragment utilization：cross-user fragment 数从 t0 起逐时段累积，t8 时仍可被使用（即使其贡献 agent 已被吊销访问权——只要 fragment provenance 仍被当前授权满足）。
- 访问矩阵 Figure 6（t1 与 t7）证实所有 user × agent / user × resource 调用均落在授权格内，零越界。

**未覆盖的指标：**
- 没有 latency 数据（论文 §6 自陈"unpredictable API latencies"，用 resource calls 做 proxy）。
- 没有 fragment 数 vs 检索精度退化曲线。
- 没有 ablation："关闭 shared memory 但保留 private memory" 的细粒度对照。
- 没有 cross-judge：S1 的 LLM-as-Judge 与 system 同为 gpt-4o，未做交叉模型复评。
- S2 没有 accuracy（"open-ended 无 ground truth"），只能间接论证"协作不损害质量"。

## Weaknesses

- **"Immutable provenance" 是声明而非机制。** §3.2 反复强调 provenance 四元组 (T, U, A, R) 不可变，但实现层（§4 "Memory Encoder"）只是普通 metadata 字段——没有签名、哈希链、append-only log 或任何防篡改机制。若任一系统内部组件（write policy LLM、memory store DAO）有 bug 或被注入，provenance 可被静默修改，整个 retrospective 权限重检机制失效。论文将此当作"理所当然的安全属性"而无任何 enforcement 评测 [10: §3.2 vs §4]。
- **安全保证依赖 LLM 概率输出。** Write policy 的 redact / anonymize / paraphrase 由 LLM 配 prompt 完成（Table 1），即"敏感信息是否被正确去除"完全依赖 LLM 的概率性行为。论文 §6 自陈这点，但**没有任何"redaction 失败率"评测**——例如系统提示要求移除"用户特定细节"，但 LLM 仍把姓名 / 公司 / 邮件嵌在 paraphrase 里的概率有多大？论文未做攻击性测试或合成对抗测试。在企业治理场景下，这是不可接受的安全 gap [10: §4 "Policy Instantiation", §6]。
- **跨用户冲突合并机制完全缺失。** §3 整章未触及"两个用户对同一事实写入互相矛盾的 fragment 时如何处理"。当 user A 的 fragment 说"X 公司 Q3 营收增长 5%"、user B 的 fragment 说"X 公司 Q3 营收下滑 3%"，第三方用户检索时这两条 fragment 都会被返回——既无版本号、无权威性投票、无新近性优先、亦无 LLM 仲裁。这对 thesis goal 5（User/Role/Org 三层记忆 + 合并冲突治理）来说是**关键空缺** [10: §3 整章]。
- **没有衰减 / 遗忘 / 容量管理。** Memory 单调增长。论文实验时长有限（最大 2,556 查询），但企业 7×24 场景下 fragment 数会达到 10⁵–10⁷，top-k retrieval 在大规模下的精度退化未被讨论。无 TTL、无 LRU、无 importance-based pruning——直接平移到生产是不可行的。论文将此作为"future work"轻描淡写 [10: §3.4, §6]。
- **Coordinator 的失败模式被掩盖在端到端指标里。** Coordinator LLM（gpt-4o）的 agent selection 没有独立评测——错选 / 漏选 / 重复调用的失败率全部混入 accuracy 与 agent utilization 数字。例如"agent utilization ~1.5–1.8" 既可能是 coordinator 精确调度，也可能是 coordinator 频繁重选同一 agent。这与 [6] MAST FC1 Specification Issues / FC2 Inter-Agent Misalignment 的归因诊断完全无对应 [10: §4 "Multi-Agent Interaction Loop"]。
- **Top-k retrieval 在 fragment 数增长时的精度曲线缺失。** S1 用 kuser = kcross = 10、S2 用 20。但 shared memory 在 S1 跑完 2556 查询后会累积大量 fragment，cosine similarity top-10 是否仍能精准命中相关 fragment 没有评测。这是 memory 系统最核心的工程指标之一，论文回避 [10: §4 "Memory Retrieval", §5.1]。
- **Scenario 2 的资源使用降幅未量化。** §5.2 仅给出 4 张直方图（Figure 4）显示"collaborative 直方图整体左移"，但**没有具体百分比降幅**。S1 给出"61% / 59%"具体数字，S2 仅文字论断"reduces overall resource calls"——评测严谨度不一致 [10: §5.2]。
- **Synthetic data 与 evaluator 共享底座。** S2 的 200 条业务查询由 GPT-4o 合成（§D.1），评测中 coordinator / agent / aggregator 也是 GPT-4o——同一 LLM 既出题、又作答、又评判，存在自一致偏置。论文未做 cross-model sanity check（如 Claude / Gemini 出题但 GPT-4o 答）[10: §D.1, §4]。
- **LLM-as-Judge 与 backbone 同模型（S1）。** Accuracy 用 gpt-4o 当 judge，但 system 也用 gpt-4o——系统倾向 judge 偏好的回答风格，可能存在自评偏置。无 cross-judge 复评（如让 Claude 复评同一答案）[10: §4 "Metrics"]。
- **"Resource utilization 下降 = memory 复用"是 narrative 推理，无 ablation。** S3 §5.3 把 t1–t8 的 resource 调用下降归因为 fragment 复用，但缺一个关键对照："关闭 memory 但保留 access graph" 的 resource 调用曲线——若该曲线也下降（例如因为模型 prompt 学会了），则归因不成立。论文未做 [10: §5.3]。
- **Bipartite 图规模小。** 最大配置：5 用户 × 6 agent × 6 resource（S1）。Decision Agent 目标 5k–10k 工具 + 数百 agent + 数百用户角色，bipartite 图节点数差 2–3 个数量级。fragment 检索、provenance 计算、access set M(u,a,t) 求交在大规模下的工程成本论文未触及——尤其 Eq. 3 的 "A(m) ⊆ A(u,t) ∧ R(m) ⊆ R(a,t)" 在 fragment 数 × 授权数 大规模下是 O(|fragments| × |permissions|)，需要索引设计 [10: §3.2 vs thesis Design Context]。
- **没有 RBAC / ABAC baseline 对比。** §2 主张框架"继承 ABAC 的 policy modularity"，但实验**从未与"naive ABAC + flat memory"做对比**。无法判定 bipartite-graph + provenance 设计是否相对 ABAC 提供了新价值，还是只是同一抽象的 LLM-friendly 包装 [10: §2 vs §5]。
- **三层 hierarchy（User/Role/Org）的论文是两层（User/Shared）。** Shared memory 在论文中按"贡献 agent" 分组（M_shared(a, t)），没有 Role 维度（4 用户角色在 S2 是模拟的，但 memory 没有"Market Researcher 共享但 Logistics Lead 不共享"的中间粒度）。要扩展到 thesis goal 5 的 User/Role/Org 三层，需要把 shared memory 进一步按 role 分区 + 引入 role hierarchy——非平凡延伸 [10: §3.2 vs thesis goal 5]。
- **"Asymmetric 协作降低 redundant work" 的因果链未拆解。** S2 中 Strategy Director 的 resource 调用数因前置用户的共享 fragment 而下降——但具体是"Director 直接读到了 Logistics Lead 的最终结论 fragment"还是"Director 读到了 Market Researcher 的中间分析 fragment"？论文未做 fragment-level 引用追踪。Decision Agent 在生产中需要这种引用链审计能力，论文示例（§D.3）只给一个手挑的 success case [10: §5.2, §D.3]。
- **没有失败模式分类。** 与 [9] 同样问题——论文没用 MAST 类工具诊断 collaborative memory 的失败模式（错误共享 / 错误隔离 / fragment 噪声 / coordinator 错调度）。"协作降低 resource calls" 与"协作引入了什么新失败" 完全不对称报道 [10: §5 整章]。

## Relations

- **builds-on 09_llm_based_multi_agent_blackboard_system [high]**：[9] 的 β / β_r 双板设计——共享黑板 β 接受请求广播、私有响应板 β_r 隔离 helper 响应——与本论文 M_shared / M_private 的双层 memory 在结构上同构：β ↔ M_shared 都是跨主体可见的共享通道，β_r ↔ M_private 都是按"贡献者"隔离的私有空间。两文角度互补：[9] 处理"任务执行期间多 agent 通信结构"（按任务划分），本文处理"跨任务跨用户长期 memory 累积"（按用户划分 + 时变授权）。本文为 [9] 的 β_r 私有性补上了**长期持久化 + 重访权限校验**机制，是 [9] 在跨会话维度上的自然扩展 [high]。
- **competes-with 09_llm_based_multi_agent_blackboard_system [med]**：但两者在"helper / agent 之间是否相互可见"上对应不同设计选择。[9] 强制"helper agent 互不可见"是为执行期防 cross-influence；本论文的 shared memory 中"agent 互相产出的 fragment 是可被其他 agent 读到的"——若一个 agent 在为 user A 服务时读到 agent B 为 user A' 写的 fragment，则跨 agent 信息流是允许的（只要权限通过）。这是"执行期严格隔离"vs"持久期受控共享" 的设计分歧。Decision Agent 在落地 Shared Workspace 时必须在两者间显式选择 [med]。
- **competes-with 06_why_do_multi_agent_llm_systems [med]**：MAST 把 MAS 失败按 FC1 / FC2 / FC3 分类，FC2 Inter-Agent Misalignment 占 32.3%。本论文的 collaborative memory 在设计层面对若干 FM 是潜在缓解——FM-2.1 信息隐瞒（shared memory 让先前结果跨 user 可见）、FM-2.4 信息共享错误（write policy 强制结构化转写降低自由对话漂移）。但**论文未引用 MAST、未做失败模式量化**——"我修复了 FC2 中哪几条"完全空白。两者在"MAS 协调失败需要工程修复"上互证，但本文不能作为 FC2 缓解的实证证据 [med]。
- **competes-with 04_rethinking_the_value_of_multi_agent (OneFlow) [low]**：OneFlow 论证同质 MAS 可被折叠为单 LLM 顺序调用——前提是任务上下文 fit 单 LLM。本论文 S1 / S3 的查询本身大多 fit 单 LLM 上下文（multi-hop QA 和 SciQAG 单题），但**论文论点不在"多 agent 必要性"而在"多用户 memory 共享"**——即使把多 agent 折叠为单 LLM，多用户场景的 memory 隔离 / 共享 / 权限约束仍然存在。两者论域错位但不冲突：OneFlow 处理空间结构（agent 数量），本文处理时间维度（用户跨会话）。Decision Agent 在 thesis 中已记入"OneFlow 折叠适用于上下文 fit 子情形"，本文额外补充"即使在折叠后，跨用户 memory 仍是独立工程维度" [low]。
- **builds-on 01_co_evolving_llm_decision_and_skill [med]**：[1] COS-PLAY 关注 skill 库随对话演化（skill = 可复用决策技能），本论文关注 memory fragment 库随多用户交互演化（fragment = KV 形式的事实/总结）。两者结构同构——一个不断增长的可复用知识库 + 检索增强；本文引入了 [1] 缺失的 access control 与 provenance 维度。Decision Agent 的"User/Role/Org 三层记忆"可同时借鉴 [1]（如何沉淀） + 本文（如何隔离与共享）[med]。
- **builds-on 05_agent_as_a_graph [low]**：Agent-as-a-Graph 用 KG 类型化 + 图查询做工具召回（Recall@5 +14.9%）；本论文用 cosine similarity top-k 做 fragment 召回。两者目标都是"在大候选集中精准命中相关条目"，但本文没用任何结构化类型 / KG——纯 embedding 检索。在 Decision Agent 的 BKN-anchored memory 设计中，本文 fragment 的 provenance (U, A, R) 元组本身就是天然的"类型化标签"，可与 [5] 的 typed retrieval 叠加（先按 provenance 类型过滤、再做 embedding 相似度）。本文为 [5] 提供了"类型来源"的现成结构 [low]。
- **builds-on 07_mcp_zero_active_tool_discovery_for [low]**：MCP-Zero 让模型主动写结构化请求驱动工具检索；本文 write policy 让 LLM 主动按 prompt 转写 fragment（结构化 KV）。两者共享"模型主笔结构化输出"骨架，但层级不同：MCP-Zero 在工具召回层，本文在 memory 写入层。两者方法论同向 [low]。
- **orthogonal to 03_why_reasoning_fails_to_plan_a (FLARE) [low]**：FLARE 处理单 agent 推理深度不足问题，本文处理多 agent 多用户 memory 共享问题，论域不重叠。但本文的 coordinator + agent + aggregator 整体仍是步进推理，按 FLARE 论证在长链规划上是贪心策略——若 collaborative memory 被用于长链规划场景（即 Decision Agent 的核心目标），FLARE 的局限仍然适用 [low]。
- **orthogonal to 02_graph_of_agents [low]**：GoA 处理单 task 内多 agent 输出协调的图结构，本文处理跨任务跨用户的 memory 共享。问题域正交。
- **orthogonal to 08_simulating_human_cognition_heartbeat [low]**：HSC 处理"何时进入推理"的时序调度，本文处理"如何跨用户共享 memory"——后者是请求驱动而非心跳驱动。问题域正交。

### Relation to thesis

直接命中 thesis Design Context 的 **goal 5（跨会话层级化 Memory）** 与 **goal 3（Shared Workspace 的 ISF 权限控制维度）**——是当前 corpus 中**第一份**直接处理"多用户 memory 共享 + 时变访问控制"的工作，对应 thesis "现有 build_memory/search_memory 基础链路上构建 User/Role/Org 三层记忆 + 写入双通道 + 治理与遗忘机制"的明确支柱。

**1. Goal 5 三层记忆的两条工程化原型 [high 关联度，但需补强]。**
- thesis goal 5 需要 **User / Role / Org** 三层；本文给出 **User / Shared** 两层，缺 Role 中间层。但"shared memory 按贡献 agent 分组"（M_shared(a, t)）实质是按 agent 角色分区，可被解读为 Role 维度的雏形。Decision Agent 的 Role 层可类比为"按贡献 agent 集合切分的 shared memory 子区间"，再加 Org 层"全 Org 共享 + 跨 Role 公共知识"作为第三层。
- thesis 写入双通道（"private 直写 + shared 经审 / 转写"）——本文的 π^{write/private} 与 π^{write/shared} 双 policy 是直接同构。Table 1 给出的两条系统级 prompt（private 保留具体细节、shared 强制移除用户特定信息）是 Decision Agent 双通道 prompt 设计的可直接借鉴起点。
- thesis 治理与遗忘——本文 **完全缺失** decay / TTL / pruning。Decision Agent 必须在借鉴 provenance + bipartite 授权框架的同时**自行补齐遗忘机制**，这是 thesis 必须扩展而本文未触及的工程空缺。

**2. Provenance 与 TraceAI 的对齐 [high 关联度]。**
- 本文的 (T, U, A, R) provenance 四元组与 thesis "TraceAI-first / action provenance must be explicit" 治理约束**结构上完全对齐**。每条 fragment 携带"何时 / 谁的指令下 / 哪些 agent 参与 / 调了哪些 resource"——这恰是 TraceAI 需要审计的维度。
- 但本文 provenance 是普通 metadata，**无防篡改机制**（§ Weaknesses 第一条）。Decision Agent 在借鉴本文设计时**必须把 provenance 升级为 append-only + 哈希链 / 签名**，使其满足"事后审计可信"的治理要求。

**3. Goal 3 Shared Workspace 的 ISF 权限模型 [high 关联度]。**
- thesis goal 3 要求"Session/Task 级共享工作区 + ISF 权限控制"。本文 G_UA(t) 与 G_AR(t) 双 bipartite graph 是 ISF 权限模型的具体数学形式：用户-agent 授权 + agent-resource 授权两层。Eq. 3 的 M(u,a,t) 求交规则给出"在两层授权下何时何条 memory 可见"的可计算判据。
- 与 [9] β_r 私有响应板的张力（thesis 已记录）：本文给出了一个"私有 vs 共享"在跨会话维度的清晰分割准则。Decision Agent 落地时应**采用 thesis 已决议的分轨策略**——内部信息处理走 private + TraceAI 镜像、涉及外部副作用走 shared + Human-in-the-loop——并在本文的"按贡献 agent 分组共享" 之外**额外引入按动作类型分轨**。

**4. 可证伪点更新建议 [med]。**
- 新增可证伪点（建议加入 thesis 表）：**"User/Role/Org 三层记忆设计在 5k+ 工具 / 数百用户角色规模下，bipartite 授权图的求交计算（Eq. 3）保持亚线性"**——本文最大配置仅 5 用户 × 6 agent，远低于该规模，证据支持仅适用于 ≤ 数十节点小规模。Decision Agent 落地需自行验证。
- 新增可证伪点（建议加入 thesis 表）：**"LLM-based write policy（redact / anonymize）在企业敏感数据场景下的失败率 ≤ 1%"**——本文未做该评测，但安全要求决定该假设若被证伪（>5% 泄漏率）则整个 transformation write 路径不可接受，需替换为规则引擎。
- 现有可证伪点"BKN 语义解耦在 5k+ 工具规模仍有效" 与本文检索机制的关系：本文用纯 cosine similarity 做 fragment 检索，**没有 BKN 类型化 / 语义结构化** ——与 thesis 的"BKN-anchored 检索优于纯 embedding"主张并不冲突，反而本文 provenance 元组提供了 BKN 类型化 retrieval 的天然落点。

**5. Goal 4 Composer 的间接支持 [low 关联度]。** 本文 coordinator + multi-agent 编排（§4 "Multi-Agent Interaction Loop"）是 Composer 在执行期的对应 runtime——coordinator 输出 {agent: ID, subquery: ...} 是"自然语言任务 → Plan → 调度执行"链条的一个最小化实例。但本文的 coordinator 仅做单步 agent 选择、不做 plan 规划，与 thesis goal 4 "Composer 输出可审查 Agent Team Plan" 的要求差距大，仅可作为 runtime 调度细节参考。

**6. Goal 1 / 2 证据中性 [low]。** 本文不涉及"开箱即用模板"也不涉及"主动 Agent / Heartbeat"，对 thesis 这两个 design goal 既不构成支持也不构成证伪。

**证据强度：** 论文整体[med]——v1 preprint，方法论清晰、形式化干净（bipartite graph + Eq. 3 求交规则）、3 个递进场景设计合理；但实验规模小（最大 5 用户 × 6 agent）、provenance 无防篡改 enforcement、安全保证依赖 LLM 概率输出、跨用户 fragment 冲突未处理、无衰减机制、关键 ablation（如关 memory 仅留授权图、cross-judge 复评）缺失。**应作为 Decision Agent goal 5 的 access control / provenance 形式化原型来源**，但**不应作为 LLM-based redaction 在企业治理场景下安全性的直接证据**——后者必须自行做对抗测试。Decision Agent 三层记忆需在本文双层基础上**补齐 Role 层、衰减机制、provenance 防篡改、冲突合并** 四项工程能力。
