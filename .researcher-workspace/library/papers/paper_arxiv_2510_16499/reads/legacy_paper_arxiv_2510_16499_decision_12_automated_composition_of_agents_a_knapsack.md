---
title: "Automated Composition of Agents: A Knapsack Approach for Agentic Component Selection"
paper_id: "paper_arxiv_2510_16499"
source_kind: "arxiv"
source_id: "arxiv:2510.16499"
source_url: "https://arxiv.org/abs/2510.16499"
kind: legacy-topic-read
topic: "decision"
source_note: "notes/active/12_automated_composition_of_agents_a_knapsack.md"
---

# Legacy topic read

This Library read artifact was backfilled from an existing topic note. It preserves historical reading state; it was not produced by the current Library deep-read runner.

## Source topic note

---
zone: active
pin: false
score: 0
dwell: 0
paper: "Automated Composition of Agents: A Knapsack Approach for Agentic Component Selection"
arxiv: "2510.16499"
authors: ["Michelle Yuan", "Khushbu Pahwa", "Shuaichen Chang", "Mustafa Kaba", "Jiarong Jiang", "Xiaofei Ma", "Yi Zhang", "Monica Sunkara"]
year: 2025
venue: "NeurIPS 2025（AWS Agentic AI；Yuan 完稿时已转 Oracle AI）"
note_number: 12
---

## Claims

- 把"从已有组件库中选最优子集组装智能体"形式化为带预算约束的 knapsack 问题：S* = argmax pτ(S) s.t. Σ ci ≤ B；并提出 composer agent 通过"生成 skill → 检索候选 → 沙箱测试 → ZCL 在线阈值收 / 弃"的工作流求解 [12: §3, §4, Algorithm 1]。
- 单 agent 设置下，online knapsack composer 相对纯检索基线最多取得 31.6% 的成功率绝对提升；在 GAIA / SimpleQA / MedQA 三个数据集上均落在 success–cost Pareto 前沿 [12: Abstract, §5.1, Figure 3]。
- 多 agent 设置下，从 117 个 sub-agent 的 inventory 中选 sub-agent 组装 hierarchical MAC 团队，online knapsack ($6 预算) 把 mortgage 域 overall goal success rate 从 identity 全选的 0.07 拉升到 0.87；travel 域从 0.07 拉升到 0.40 [12: §5.2, Figure 4, Table 5]。
- 纯 retrieval composer 在 GAIA / SimpleQA 上未选中任何 web search 工具，因为 skill description "Web Browsing" 与 `get_article_content` 描述语义最近、与 `web_search_paid` 反而较远；这是描述-能力错配导致检索失败的典型实例 [12: §6, Table 2]。
- 多 agent 域中 retrieval-based 方法系统性挑选 "distractor" agent —— 这些 agent 描述与原 MAC agent 重叠但缺少 tools 和 instructions；offline knapsack ($6) 在 travel 域因此选了 5 个无能力 agent [12: §6 "Improvements with online knapsack in multi-agent composition"]。
- Identity composer（不做选择，把整个 inventory 都装上）在多 agent 设置下崩盘（travel 0.07 / mortgage 0.07），因为 supervisor agent 无法在 100+ diverse agent 上做正确的 delegation [12: §5.2, Figure 4]。
- ZCL 在线阈值算法 Ψ = (U/L·e)^z · L·e（z = 已用预算占比）保证 ln(U/L)+1 的竞争比；论文把它直接接到组件选择上 [12: §2 Knapsack Algorithms, Algorithm 1]。
- 在 online knapsack 之上叠加 AvaTaR prompt 优化（基于沙箱 trial 轨迹做 prompt 反馈），在 SimpleQA 上把 Claude 3.5 Sonnet 从 0.92 推到 0.92（持平最高）但 Haiku 从 0.78 推到 0.86；prompt 优化在部分设置下出现回退 [12: §5.1, §6, Appendix A.8, Table 7]。
- 跨 8 个 backbone（Claude 3.5/3.7 Sonnet、Claude 3.5 Haiku、Llama 3.3 70B、Llama 4 Scout/Maverick、Qwen 2.5 72B）trend 一致：online knapsack ≥ offline knapsack ≥ retrieval；3 次重复跑 SimpleQA 标准差 ≤ 0.034 [12: Appendix A.9, Tables 4–12]。
- Sandbox trial 的判别采用 LLM-as-judge：composer 自身判断"工具是否帮助 agent 答对 query / 是否 broken"；broken 工具被永久排除，已覆盖的 skill 跳过后续测试 [12: §4 Online Knapsack Composer, Algorithm 2, Appendix A.2–A.3]。

## Assumptions

- 任务描述 x 已经"足够明确"（well-defined task description），论文在 Limitations 段显式承认这一前提，并将 ambiguous-goal 场景列为 future work [12: §6 Limitations]。
- Composer agent 自己生成的 skill 列表（≤6 个）能覆盖任务所需的"required, core abilities"；skill 列表是单次 LLM 调用产出，不与人工 ground truth 比对 [12: §4 Retrieval Composer, Appendix A.1]。
- 每个 skill 的 2 条 LLM 生成测试 query + 对应 reference plan 足以判别工具的真实效用；同一 LLM 既生成 query 又作 judge，未独立分离生成器与判别器 [12: §4, Appendix A.1–A.2]。
- 工具/sub-agent 价格可静态估计：单 agent 用 "API 调用费 + schema input token 费"近似（free $3, paid $5 / $8 per 5k 调用），multi-agent 直接给所有 sub-agent 统一定价 $1（论文承认在 simulated-tool 设置下"无 meaningful runtime/API cost variation"）[12: §5.1 Models, §5.2 Setup, Appendix A.5]。
- ZCL 算法的 (L, U) 价值-代价比上下界从 inventory 全局 cost 极值推出：L = 1/max(c)，U = Σwj / min(c)；这隐含"未来到达组件的价值/代价比仍落在该范围内" [12: Algorithm 1, §2 Knapsack Algorithms]。
- 组件之间的非加性交互（synergy / conflict）虽在 §3 被点名为"二次耦合项"，但实际算法把组件按 skill 独立测试、独立累加值；隐含"实践中独立测试已是足够近似" [12: §3 vs Algorithm 1]。
- "Distractor" agent 的能力错配（描述与原 agent 重叠但无工具）足以代表"现实中 retrieval 会被 fooled"的失败模式；synthetic 构造用于多 agent 实验中扩展 inventory 从 ~20 到 117 [12: Appendix A.6]。
- Sandbox 测试一次得到的 helpful/broken 二元判断能稳定刻画组件未来部署中的真实表现；不考虑组件回归、上下游 prompt 改动后失效等运行期漂移 [12: §4, Appendix A.2]。

## Method

**核心论点：组件选择应该由"语义检索（描述匹配）"升级为"语义检索 + 沙箱实测（能力验证）+ 预算感知（在线 knapsack 阈值）"的三段式 pipeline，并在装配阶段一次性闭环。**

**❶ 形式化（§3）。** 给定任务 τ 与描述 x、组件库 A = {ai}（每个组件有 cost ci 与描述 di）、预算 B；目标 S* = argmax pτ(S) s.t. Σci ≤ B。论文指出该问题相对经典 knapsack 有三处偏离：(i) 真实成功率 pτ 未知，需迭代估计 → 在线变体；(ii) 组件间存在非加性交互（耦合项）；(iii) inventory 动态演化。

**❷ 四档 composer 阶梯（§4）。**
- **Identity composer**：f(A,τ,B)=A，全选。作为 cost-blind 的上界基线。
- **Retrieval composer**：先让 composer 把任务描述解析成 ≤6 个 skill（name + description + importance + 测试 query 列表，详见 Appendix A.1 prompt），按 skill description 在 inventory 上做 top-1 语义检索（BGE-Large-English），结果取并集。
- **Offline knapsack composer**：在 retrieval 的 top-K=10 候选上跑多选 knapsack 线性规划：viOFF = Σm SIM(ai, m)，约束 = 预算 + skill 覆盖（每个 skill 至少一个候选）[12: Eq. 2]。
- **Online knapsack composer（主张）**：把 retrieval 候选按到达顺序 stream-test，套 ZCL 算法。

**❸ ZCL 阈值（§4 + Algorithm 1/2）。** 价值-代价比下界 L = 1 / maxa ca、上界 U = Σj wj / mina ca。已用预算占比 z = (B − B̂) / B。当前阈值
Ψ(z) = (U / (L·e))^z · L·e
当候选 ai 的实测 ratio ρi = vi / ci ≥ Ψ(z) 时收下，否则放弃。论文称该选择"理论上 ln(U/L)+1 竞争"。

**❹ 实测值 vi 的来源（§4 + Appendix A.2）。** 对每个候选组件 ai：
1. 取 skill 列表 M = {(mj, dj, Qj, wj)} 的所有测试 query；
2. 把 ai 装到 base agent（CodeAct / ReAct）上跑这些 query；
3. composer 用 LLM-as-judge prompt 判断每条 trajectory 是否"helpful"或"broken"——单 agent 时 prompt 看 "工具是否帮助 agent 答对 query"，多 agent 时看 "sub-agent 是否能调用对的工具完成请求"；
4. vi = 当前 round 内"未被 cover 的 skill 中、ai 被判 helpful 的那些 skill 的 importance wk 之和"。
- **早停优化**：被判 broken 的组件入 broken 集合，永不再测试。
- **跳过覆盖**：本 round 已被某组件覆盖的 skill 不再为后续候选测试。

**❺ 单 agent 实验（§5.1）。**
- Inventory：120 工具，混合 LangChain 公开 API（arxiv、PubMed、Wikipedia、SemanticScholar、web_search_free/paid）+ ToolRet 数据集中可用的子集；分 23 个功能类，35 个工具（28.5%）跨多类。
- 价格：free $3 / free-rate-limited $5 / paid $8（折算"5k 调用 + schema 输入 token"）。
- Agent 框架：CodeAct（让 agent 用 Python 代码触发工具调用，相对 JSON tool-calling 更强 [Wang et al. ICML'24]）。
- Backbone：Claude 3.5 Sonnet/Haiku、Claude 3.7 Sonnet、Llama 3.3 70B、Llama 4 Scout/Maverick、Qwen 2.5 72B。
- 数据集：GAIA（"smolagents" 版）、SimpleQA、MedQA。
- 预算：$10 / $30。Top-K=10 检索；每 skill 2 条测试 query。
- 一个变体把 online knapsack 选出的工具 + sandbox 轨迹喂给 AvaTaR [Wu et al. NeurIPS'24] 做 prompt 优化。

**❻ 多 agent 实验（§5.2）。**
- Setup：直接复用 Shu et al. (2024) 的 hierarchical MAC 评测框架（一个 supervisor + 若干 specialist sub-agent）。原 inventory ~20 个 sub-agent，被 synthetic 扩展到 117 —— 加入 healthcare / e-commerce / education / job / sustainability 等横跨多域的"分散"agent，外加 10 个"profile 与原 MAC agent 重叠但无工具"的 distractor。
- 域：travel + mortgage。每个 sub-agent 价格统一设为 $1。
- 预算：$3 / $6。
- 评测：goal success rate（Overall + Partial）；端到端通过 user–MAC 模拟对话 + assertion 列表判通过。

**❼ 与既有组件选择工作的关系（§2）。** 论文把自己定位在 ADAS（Hu et al. 2024）的子问题"automated agent composition"上，并把已有的 tool retrieval（ToolLLM、ToolRet、RAG-MCP）、agent selection（DyLAN、AgentPrune、Multi-agent Architecture Search）、joint prompt+tool optimization（Wu et al. 2025）共同纳入"prior work 仍以静态 metadata / 语义检索为主"的批评对象，主张其差异在引入预算约束与实测验证。

## Eval

- **指标 + 基线**。单 agent 用任务级 success rate（GAIA / SimpleQA / MedQA 各自 benchmark 的 pass 判定）+ 总 tool 价格（美元）作 Pareto 前沿；多 agent 用 Overall / Partial GSR。基线四档：identity / top-1 retrieval / offline knapsack / online knapsack（再加一个 online knapsack + AvaTaR）。
- **Pareto 主结果（Claude 3.5 Sonnet, §5.1, Figure 3）**。GAIA：online knapsack+OPT($30) 0.47 success @ $14 vs identity 0.47 @ $398 vs retrieval 0.19 @ $23；SimpleQA：online knapsack+OPT($30) 0.92 @ $12 vs identity 0.80 @ $398；MedQA：online knapsack($30) 0.93 @ $6 vs identity 0.92 @ $398。Online knapsack 系列均落在 Pareto 前沿。
- **多 agent 主结果（§5.2, Figure 4, Table 5）**。Travel：online knapsack($6) 0.40 / 0.69 (overall/partial) vs offline knapsack($6) 0.17 / 0.38 vs retrieval 0.23 / 0.57 vs identity 0.07 / 0.09。Mortgage：online knapsack($6) 0.87 / 0.93 vs offline($6) 0.70 / 0.89 vs retrieval 0.37 / 0.68 vs identity 0.07 / 0.02。
- **跨模型一致性（Appendix A.9, Tables 4–11）**。Online knapsack 在 Claude 3.5/3.7 Sonnet、Claude 3.5 Haiku、Llama 3.3/4、Qwen 2.5 上 dominance 趋势一致；个别低端 backbone 上 GAIA 分数低（如 Llama 4 Scout online knapsack($30) GAIA 0.31）但相对自身 retrieval 仍领先。
- **稳定性（Appendix A.9, Table 12）**。SimpleQA + Claude 3.5 Sonnet 三次重复：online knapsack($30) 0.873 ± 0.034、offline($30) 0.207 ± 0.009、identity 0.840 ± 0.033。
- **定性证据（§6, Table 2）**。Retrieval 在 GAIA 选了 pub_med / read_file / wolfram_alpha / job_title_autocomplete / get_article_content / number_fact，没选任何 web search 工具；offline knapsack 选 web_search_free（rate-limited）；online knapsack 选 web_search_paid。

## Weaknesses

- **多 agent 实验里 sub-agent 价格全 $1，让 ZCL 的核心理论卖点失效**。在 cost-uniform 设置下 L = 1/$1、U = Σwj/$1，ratio ρi = vi 退化为绝对 value；ZCL 的"价格-价值阈值"等价于"value 阈值"。论文也承认这是为了"isolate 选择策略"而做的简化，但同时把 multi-agent 上 0.07→0.87 的巨大提升都归因于 knapsack 算法 —— 该归因在算法已经退化为"在已 cover skills 之外按 value 贪心收"时不成立 [12: §5.2 Setup]。
- **"distractor agent" 是论文自己合成的（描述重叠 + 无工具）**，并且原 MAC inventory 仅 ~20 个就被拉到 117 个，扩展是 synthetic 而非真实运营 inventory。retrieval-based 方法在这个被故意构造来欺骗 retrieval 的 inventory 上失败，并不能直接外推到"真实大规模 agent inventory 中 retrieval 同样会失败"——论文未与现实大规模 inventory（如 LangChain Hub、HuggingFace Spaces、MCP servers）做对照 [12: Appendix A.6]。
- **沙箱判别是 self-judging**：composer agent 既出 skill 与 test query、又当 LLM-as-judge 决定组件 helpful/broken。Test query 由同一 LLM 生成、由同一 LLM 评估自己的产出，闭环没有人类 ground truth；Appendix A.2 的 prompt 直接让 LLM 二元决定，未做 calibration 或多评判一致性度量 [12: §4, Appendix A.2]。
- **inventory 规模仅 120 工具 / 117 agent，远低于 ContextLoader 目标的 5k–10k**。ZCL 的竞争比 ln(U/L)+1 在 cost 范围拉开（Decision Agent 多租户工具 cost 跨 free→paid→infra 资源消耗多档）时迅速变大；论文未做规模 scan，沙箱总耗时 10–30 min（论文自承）也未公布 inventory size × budget 的扩展曲线 [12: §6 Limitations]。
- **Sandbox 是 design-time 一次性测试**，不在生产期重测；与 [9] Blackboard helper 在每个请求上 self-nominate 形成结构性对照——这意味着工具 / sub-agent schema 漂移、上游 prompt 改动、模型升级都会让一次性沙箱过期，但论文未量化"沙箱判别多久后失效"。
- **Skill 列表上限 6 个 + 每 skill 2 条 query**，对长链业务任务（"查询-分析-撰写-校验-审批"等）的 skill decomposition 是否充分未做敏感性分析；skill 数 / 测试 query 数 / top-K 三者的 ablation 在论文正文与附录中均缺失。
- **prompt 优化（AvaTaR）出现回退而无机理解释**。论文明示"in a few settings"prompt 优化反而降分（Table 6 GAIA Claude 3.7 Sonnet online knapsack ($30) 0.50 → 0.44；Llama 4 Scout SimpleQA 0.82 → 0.82 持平等），但只在 §6 Limitations 一笔带过"emphasizes need for more robust optimization method"——未提供 trial 数 / 信号筛选 / 终止判据等可复现细节。
- **组件间非加性交互被形式化点名（§3）但被算法忽略**。Algorithm 1 的 vi 仅按"未覆盖 skill 的 importance 之和"计算，假设组件值可加；论文未给出 synergy / conflict 真出现时算法的失败案例或 fallback。
- **GAIA 上 online knapsack 与 identity 持平（0.47 vs 0.47）但成本从 $398 降到 $14**——论文据此宣称 cost 优势，但未排除 identity 的失败是因为 122 个工具撑爆 prompt 上下文导致 agent 困惑；如果 identity 失败的根因是 token-bloat 而非工具选择本身，则 cost 节省的核心解释偏向 "context bloat 缓解"而非 "knapsack 最优"，论文未做该控制实验（如随机抽样 4 工具子集）。

## Relations

- builds-on 11_retrieval_models_aren_t_tool_savvy [high]：论文 §2 与 §5.1 显式引用 Shi et al. ([11]) 的 ToolRet，并使用其工具集构造 inventory；Identity & Retrieval 在 GAIA / SimpleQA 上的失败被论文用作 [11] 主张"retrieval 不够工具感知"的进一步证据 [12: §2 Tool Retrieval, §5.1 Inventory, §5.1 Results]。
- competes-with 07_mcp_zero_active_tool_discovery_for [med]：[7] MCP-Zero 把"克服检索失败"的方案放在 inference-time 由模型主动笔写结构化请求；本论文把方案放在 design-time 由 composer 沙箱测试 + ZCL 阈值。两条路径都拒绝纯静态语义检索，但选择不同的层（运行时 vs 装配时）介入；本文未引用 [7]。
- competes-with 05_agent_as_a_graph_knowledge_graph [med]：[5] AaaG 的类型化召回（BKN-同构）用 graph 结构而非测试来防止描述-能力错配；本论文用 sandbox 实测来防止同一类错配。两者解决同一问题（GAIA 上 retrieval 漏选 web_search 类）的两条独立证据线。
- contradicts 04_rethinking_the_value_of_multi_agent [med]：OneFlow 主张"多智能体系统常可被等价折叠为单一 LLM"；本论文 §5.2 在 117 sub-agent + supervisor 设置下 identity 0.07 vs online knapsack 0.87，反向表明：当 inventory 足够大且任务跨域时，"折叠"等价构造会被 supervisor 的 delegation capacity 绑死，多智能体反而是必要的——但需小心：本论文是从已有 sub-agent inventory 选子集而非真做 OneFlow 式 prompt 折叠，所以严格说是"在选择存在的前提下，选择策略至关重要"，并未直接复测 OneFlow 的折叠假设。
- orthogonal 09_llm_based_multi_agent_blackboard_system [med]：[9] Blackboard helper 在每个请求上 self-nominate（runtime self-discovery），本论文 composer 在装配前一次性沙箱（design-time pre-validation）。二者在"如何对抗中央 capability registry 失准"上互补：design-time 给静态 inventory 做能力清洗，runtime 给动态请求做 last-mile 路由。
- builds-on 02_graph_of_agents_a_graph_based [low]：GoA 优化 agent 间通信图与节点能力；本论文优化 agent 选取 + 预算约束。两者同属"多 agent 系统设计自动化"大类（均归到 ADAS Hu et al. 2024 框架），但本论文未引用 GoA。
- orthogonal 06_why_do_multi_agent_llm_systems [med]：MAST 的 14 类失败模式分类（FC1/FC2/FC3）刻画系统已组装后的运行期失败；本论文刻画的是组装期的"选错组件"失败，与 MAST 的失败轴不直接对应——但本论文 §6 关于 distractor agent 的描述（被选中后 supervisor delegation 失败）实际可被 MAST FC1（specification）与 FC3（verification）部分覆盖，论文未做该映射。
- orthogonal 01_co_evolving_llm_decision_and_skill [low]：[1] 让 skill 库随 trace 反馈增长；本论文让 composer 在装配时清洗 skill 库。两者一个 runtime 学习、一个 design-time 选择，方向相反但共享"skill / 组件库不是静态资产"的判断 [low]。
