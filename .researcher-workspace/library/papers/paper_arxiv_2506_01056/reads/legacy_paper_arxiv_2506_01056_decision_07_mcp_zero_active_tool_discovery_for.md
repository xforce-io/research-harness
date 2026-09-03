---
title: "MCP-Zero: Active Tool Discovery for Autonomous LLM Agents"
paper_id: "paper_arxiv_2506_01056"
source_kind: "arxiv"
source_id: "arxiv:2506.01056"
source_url: "https://arxiv.org/abs/2506.01056"
kind: legacy-topic-read
topic: "decision"
source_note: "notes/active/07_mcp_zero_active_tool_discovery_for.md"
---

# Legacy topic read

This Library read artifact was backfilled from an existing topic note. It preserves historical reading state; it was not produced by the current Library deep-read runner.

## Source topic note

---
zone: active
pin: false
score: 0
dwell: 0
paper: "MCP-Zero: Active Tool Discovery for Autonomous LLM Agents"
arxiv: "2506.01056"
authors: ["Xiang Fei", "Xiawu Zheng", "Hao Feng"]
year: 2025
venue: "arXiv preprint v4 (Xiamen University, USTC)"
note_number: 7
---

## Claims

- 让 LLM 主动生成结构化的 `<tool_assistant>{server, tool}` 请求来检索工具，比把全部工具 schema 注入 prompt 在大规模工具池下精度更高、token 成本更低 [7: §1, §3.1]。
- APIBank 全集多轮场景：Claude-3.5-Sonnet 标准注入精度从 Domain 子集 100.00% 跌至 Full 集 60.22%，MCP-Zero 同条件下保持 90.32%；GPT-4.1 同条件下从 91.40% 跌至 92.47%（保持稳定）；多轮 token 消耗从 6402.2 降至 159.0（−97.52%）[7: §5.3, Table 1]。
- 大海捞针实验：在 1–2,797 工具规模、最长 248.1k token 上下文中，MCP-Zero 在 Claude-3.5-Sonnet 与 Gemini-2.5-Flash 上显著超越标准 schema 注入；GPT-4.1 因基线已强而无显著增益 [7: §5.2, Figure 5]。
- 加 1 个 in-context 示例（"ICL-1"）能进一步提升 needle-in-haystack 精度，作者归因于①样式锚定（让模型生成的请求贴近受策展的描述风格），②语义锚定（限定 server / tool 字段含义） [7: §5.2, §7.2]。
- 双层评分函数 `score = (s_server × s_tool) × max(s_server, s_tool)` 用于在 Hierarchical Vector Routing 中合成 server 与 tool 的余弦相似度，使任一维度高分都能贡献最终排名 [7: §3.1, Eq. 1]。
- 仅做"用户查询→工具描述"的检索（Q.Retrieval 基线）在 APIBank 上停留在 65–72% 精度区间，作者由此推断"由模型主笔语义对齐请求"是关键收益来源，而非检索本身 [7: §5.3, Table 1]。
- Top-1 检索在所有报告设置中已足够获得高精度；K 可上调以增强容错 [7: §3.1]。
- 发布 MCP-tools 数据集：从 MCP 官方 servers repo（commit ad2d4e6, tag 2025.4.28）筛选 308 个 servers / 2,797 个 tools，含原始描述 + Qwen2.5-72B-Instruct 生成的 server summary + 预算的 OpenAI text-embedding-3-large 嵌入；规模均值 9.08 / 中位 5.0 工具每 server [7: §4]。

## Assumptions

- 模型自发生成的 `<tool_assistant>` 请求文本与工具/服务器描述处在相近的语义空间，即 `cos(e_r, e_t) > cos(e_q, e_t)`——这一论断在 §3.2 以"嵌入空间一致性"形式陈述，但未独立度量两类相似度的差距 [7: §3.2, Eq. 5]。
- MCP 规范强制每个 server 与 tool 都提供描述性文档，因此该方法跨整个 MCP 生态可用；论文未讨论描述质量参差或缺失场景 [7: §3.1]。
- LLM-as-curator 生成的 server summary 是检索时的有效补充字段（与原始描述取较高相似度），但 summary 本身由 Qwen2.5-72B 生成，被检索的请求由 GPT-4.1/Claude-3.5/Gemini-2.5 生成——作者未审视该"LLM 写、LLM 检"循环对结果代表性的影响 [7: §3.1, §4.1]。
- APIBank Level-1 的 48 个 unique tools 上的多轮对话和 MCP-tools 上的 needle-in-haystack 单步指认，能够外推至真实多步、多智能体的工具发现场景 [7: §5.1, §5.2, §5.3]。
- 工具检索准确性可独立于规划质量来度量——评测假设上游已经分解或单轮即可定位目标工具，未把规划失败计入 [7: §5.2]。
- temperature 等采样设置默认稳定（论文未明示），且不同模型在"自发生成结构化请求块"上行为可比 [7: §5]。

## Method

**核心范式转换。** 不再把所有工具 schema 注入 system prompt，而是让 LLM 在执行中"按需声明能力缺口"，触发外部检索。三个机制：

**❶ Active Tool Request（§3.1）。** Prompt 显式授权模型在解决不了时输出固定结构块：

```
<tool_assistant>
server: ...   # 平台/权限域
tool: ...     # 操作类型 + 目标
</tool_assistant>
```

每出现一次这样的块就触发一次独立检索，模型可在单轮对话中多次发起请求。

**❷ Hierarchical Vector Routing（§3.1）。** 两段式 coarse-to-fine：

1. *Server 阶段*：将 `server` 字段嵌入向量，与 (a) 原始 server description、(b) Qwen2.5-72B 生成的 server summary 两路分别计算余弦相似度，取较高者；过滤出 m 个候选 server。
2. *Tool 阶段*：在候选 server 内将 `tool` 字段与各 tool 描述向量做余弦相似度。
3. *合成评分*：`score = (s_server × s_tool) × max(s_server, s_tool)` 排序，返回 top-k 工具的 JSON-Schema，注入上下文供模型立即调用。

嵌入模型为 OpenAI `text-embedding-3-large`，所有嵌入预先计算并随数据集发布。

**❸ Iterative Active Invocation（§3.1）。** 收到 top-k 工具后，模型自行评估是否够用；不够则改写请求、再次检索，直至（a）找到合用工具或（b）认定无合适工具应回退到参数化知识。论文以"调试 src/train.py"为例展示连续 3 次跨域请求（filesystem→edit→terminal）的执行轨迹（Figure 3）。

**理论框架（§3.2）。** 形式化把工具选择从被动 `P_passive(t* | q, T) ∝ P(q|t*,T)P(t*|T)` 改为主动 `P_active(t* | s_t) = Σ_r P(t*|r)P(r|s_t)`，并把请求生成建模为最大化 `I(T*; r | s_t)` 的主动信息获取过程；声称复杂度从 O(n) 降至 O(m + k)。注意：§3.2 是事后合理化框架，未推导出 §3.1 的 Eq. 1 评分函数。

**MCP-tools 数据集（§4）。** 从 396 个候选 server 经"必须提供 MCP-compliant 工具定义 + 完整文档 + 足够细节"三道过滤后保留 308 个；用 Qwen2.5-72B-Instruct 配 few-shot 示例做结构化抽取，输出 `{server_name, server_description, server_summary, tools[{name, description, parameter}]}` 这一固定 schema（Figure 4）。embedding 一并预计算并发布。

## Eval

**两类评测：**

1. **Needle-in-a-Haystack（MCP-tools 上自构造）。** 工具池规模从 1 到 2,797 阶梯式扩张，最长上下文 248.1k tokens；将目标工具的任务描述作为查询，要求模型从中检出对应工具。三个对比设置：(a) 标准 tool call schema 全注入；(b) MCP-Zero；(c) MCP-Zero + 1 个 ICL 示例。模型：Claude-3.5-Sonnet、Gemini-2.5-Flash、GPT-4.1。指标：是否检中目标工具（success/fail）+ 平均 token 消耗 [7: §5.2, Figure 5–6]。

2. **APIBank（公开 benchmark）。** 取 Level-1 的 48 个 unique tools；按 conversation × tool-scope 二维交叉评测：单轮 vs 多轮、Domain 子集 vs Full 集。三种方法：(a) Q.Retrieval（用户查询直接检索）；(b) Standard Tool Call（全部注入）；(c) MCP-Zero。基于 top-1 命中计精度，并报告平均 token [7: §5.3, Table 1]。

**关键数字（APIBank, Claude-3.5-Sonnet）：**
- 单轮 Domain：Standard 97.60% / Q.Retrieval 71.63% / MCP-Zero 96.62%（token 312.4→111.0，−64.47%）。
- 单轮 Full：Standard 69.23% / Q.Retrieval 71.63% / MCP-Zero 95.19%（token 6308.2→111.0，−98.24%）。
- 多轮 Domain：Standard 100.00% / Q.Retrieval 65.05% / MCP-Zero 91.40%（token 406.4→159.0，−60.84%）。
- 多轮 Full：Standard 60.22% / Q.Retrieval 65.05% / MCP-Zero 90.32%（token 6402.2→159.0，−97.52%）。

**对比基线缺失：** 论文未在 APIBank 上对比 RAG-MCP、ScaleMCP、AnyTool、ToolRerank、COLT 等已发表的工具检索方法，仅与 Standard Tool Call、Query-Retrieval 这两条简单基线作比较 [7: §5.3]。

## Weaknesses

- **核心收益归因模糊。** 论文将 MCP-Zero 优势归于"恢复 agent 自治"，但其实质增益来自三个可分解因素：①模型生成的请求文本嵌入与工具描述更接近（语义对齐）；②减少干扰工具的注意力稀释；③允许多次检索覆盖跨域。论文未做控制实验区分这三者——例如"先注入工具但仍让模型先输出 `<tool_assistant>` 再调用"的中间设定缺失，因此"主动 vs 被动"的二分究竟是哪条机制起作用并不清楚 [7: §3.2 vs §5.3]。
- **"LLM 写、LLM 检"的循环评估。** Server summary 由 Qwen2.5-72B 生成，请求由 Claude/GPT/Gemini 生成，目标工具描述也是 LLM 重写过的——三处都在 LLM 语义空间。语义对齐优势可能很大程度是这种同空间循环的伪信号，而不是面向真实多样化人写工具文档的可迁移结论 [7: §4.1]。
- **APIBank 工具规模严重不足。** Full 集仅 48 个 unique tools；论文反复强调"thousands of tools"动机，但精度对比的全部数据点都在 ≤ 50 工具量级。MCP-tools 的 needle-in-haystack 仅做"是否能检出 1 个目标工具"的单跳指认，未评估多步、跨域链式检索精度。论文宣称的"scales with tool ecosystem growth"在 APIBank 部分无证据 [7: §5.2 vs §5.3]。
- **多轮设置不能等同于"迭代主动发现"。** APIBank "multi-turn" 是 user-assistant 多轮对话，但每轮的工具仍出自同一固定集；§3.1 描绘的"跨域 toolchain 动态构建"（filesystem→edit→terminal）从未在该设置下被独立测量——iterative invocation 的累计 token 与精度衰减没有报告 [7: §5.3 vs Figure 3]。
- **理论部分是事后合理化。** §3.2 的信息论框架（互信息最大化、O(n)→O(m+k) 复杂度）从未被实验证据 instrumented：论文没度量 `I(T*; r | s_t)`，也没度量 `P(r|s_t)`，更未把 Eq. 1 的乘积评分与互信息表达式联系起来。该框架是装饰性的，删除不影响实验结论 [7: §3.2]。
- **Top-1 评估高估实用精度。** 所有指标基于 top-1，作者承认"K 可上调以增强容错"——但若需要 K>1 才能稳定召回正确工具，多步任务里 K 个候选都会消耗下游 prompt 预算，token 节省可能急剧缩水。论文未给出 K=3/5 的退化曲线 [7: §3.1, §5.3]。
- **Server summary 可访问性是隐性依赖。** Hierarchical routing 在 server 阶段需要"扩展 summary"才达到高精度（图 5 显示 ICL-1 进一步显著提升），意味着部署时每个 MCP server 都需经一次 LLM 摘要前处理；论文未审视该预处理在新增 server 流入（动态 catalog）下的运维代价 [7: §3.1, §4.1]。
- **GPT-4.1 无增益的事实被淡化处理。** Figure 5 明确显示 GPT-4.1 baseline 已饱和、MCP-Zero 不带来精度增益（仅省 token），但 §1 与 §6 仍以"98% token reduction with maintained accuracy"作为头条；该说法只在弱模型上成立——强模型基线下"主动检索"对精度的增益边际为零 [7: §5.2, Figure 5]。
- **可复现性缺口：评测协议未完全公开。** APIBank 的 Domain 子集"由谁挑选、按什么标准"未披露；多轮 Q.Retrieval 基线在某些条目上比 Standard 还高（如多轮 Full Claude-3.5：Q.Retrieval 65.05% vs Standard 60.22%），暗示注入更多工具反而干扰决策——但没有控制实验解释这一反直觉现象 [7: §5.3, Table 1]。
- **GitHub 单工具 4,600 token 是孤例。** 文中举的 GitHub MCP server "26 tools 占 4,600 tokens" 是 motivation 的关键数字，但 MCP-tools 数据集里多数 server tool 数 ≤5；论文未给出整体工具池在 schema 注入下的 token 分布，"248.1k tokens"是 2,797 工具堆叠后的极端值，与典型部署相距甚远 [7: §1 vs §4.2]。

## Relations

- **被 05_agent_as_a_graph 显式引用为基线 [high]：** Agent-as-a-Graph (Nizar et al., 2025) 在 LiveMCPBench (70 servers / 527 tools) 上把 MCP-Zero 列为基线，报告 MCP-Zero Recall@5 = 0.70 / nDCG@5 = 0.41，被 Agent-as-a-Graph 的 0.85 / 0.47 超过 [05: §4.2, Table 1]。两者方法论相反：MCP-Zero 让模型主动写请求 + 双层余弦相似度；Agent-as-a-Graph 静态构建 agent-tool 二部图 + 类型加权 RRF。两条路径解决同一问题——大规模工具集中精确召回——但前者押注"LLM 自描述请求"贴近文档，后者押注"显式类型节点"消除多跳查询失败模式。
- **competes-with 05_agent_as_a_graph [high]：** 同一研究问题（MCP catalog tool retrieval）下的两条路径，且 [05] 的实验直接证明在 LiveMCPBench 上 Agent-as-a-Graph 全面胜出 MCP-Zero（Recall@5 0.85 vs 0.70）。但 MCP-Zero 提出的"模型生成请求 vs 用户查询直接检索"对比维度在 [05] 里被吸收为查询前处理（[05] 假定查询/子步已给定）。两者可在工程上组合：MCP-Zero 的请求生成 + Agent-as-a-Graph 的类型化检索后端。
- **competes-with RAG-MCP / ScaleMCP / Tool2Vec / AnyTool [high]：** 论文在 §2 把这些方法归为"retrieval-augmented tool selection"并批评其"single-round, query-only matching"，但在 §5 实验中并未将它们作为基线对比——只比较了简化后的 Q.Retrieval 与 Standard Tool Call。批评仅停留在 §2 文字层面，没有同台数据。
- **builds-on ReAct (Yao et al., 2023) [high]：** Active Tool Request 的 `<tool_assistant>` 块本质是 ReAct "Thought→Action" 模式的细化变体——把"Action"再分一层"Tool Request → Tool Schema → Function Call"。论文承认 ReAct 是"universal protocol"前驱 [7: §2.1]。
- **builds-on Toolformer / Gorilla / HuggingGPT [med]：** §2.1 把 context-based tool injection 谱系归到这条线上，并将 MCP-Zero 定位为对该谱系"上下文开销不可承受"问题的回应。
- **orthogonal to 03_why_reasoning_fails_to_plan_a [med]：** FLARE 关注步进推理在多步规划上的结构性失败；MCP-Zero 关注每步规划已确定后的工具召回。两者层级不同可叠加：planner（FLARE 类前瞻）→ 工具请求生成（MCP-Zero）→ 检索后端。但 MCP-Zero 自身只在单跳检出测试中验证，未触及多步规划；论文 Figure 3 的 3 步链条由模型 in-context 自然分解，规划深度极浅。
- **orthogonal to 06_why_do_multi_agent_llm_systems [low]：** MAST 把"工具 schema 注入"成本与"协同失真"区分开；MCP-Zero 解决前者（FC1 系统设计的子问题：上下文压力），未触及 FC2 智能体间错位。MCP-Zero 减少注入工具数能间接降低 FM-2.6 reasoning-action mismatch 概率（候选少则误选少），但论文未做该方向的失败模式量化。
- **orthogonal to 04_rethinking_the_value_of_multi_agent [low]：** OneFlow 论证同质 MAS 可折叠为单 LLM 顺序执行；MCP-Zero 完全在单 agent 范畴内（一个 LLM + 检索器），与 OneFlow 立场不冲突——它正是单 LLM 模拟的工具供给侧实现。
- **orthogonal to 02_graph_of_agents [low]：** GoA 处理推理时多 agent 输出协调，MCP-Zero 处理单 agent 调用前工具供给；层级正交。
- **orthogonal to 01_co_evolving_llm_decision_and_skill [low]：** COS-PLAY 协同进化技能库（学得能力），MCP-Zero 在静态 catalog 上检索已有工具（选择能力）；问题域不同。

### Relation to thesis

直接命中论题"工具幻觉的语义解耦防御"研究缺口（thesis §设计上下文 #3）。三点对 Decision Agent 的可操作启示：

1. **"模型主笔请求 vs 用户查询"在精度上很重要，且符合 BKN 的解耦思想。** APIBank 上 Q.Retrieval 仅 65–72%，MCP-Zero 高 20–30 个百分点 [7: Table 1]。这表明仅靠用户查询去检索工具远不足够；让模型先按"权限域 + 操作"两段式重写需求，再去检索，本质上是"逻辑（要做什么）/行动（去哪做）"的轻量解耦。这与 BKN 的语义结构化路径完全一致，可作为 ContextLoader 的前置请求改写器的可行范式。

2. **规模缺口仍在原位：APIBank Full 仅 48 个工具，远低于企业目标。** 论文反复用 GitHub MCP server "4,600 tokens / 26 tools" 与 248.1k 总池作为大规模动机，但精度评测全部止步于 ≤50 工具量级；2,797 工具的 needle-in-haystack 只测"能否一次检出 1 个目标工具"。Decision Agent 的真实场景是数千工具 × 多步任务 × 跨域链路，论文证据不能外推。这是与 [05] 一致的"规模未验证"缺口。

3. **强模型上无精度增益是个值得记下的边界条件。** GPT-4.1 在 needle-in-haystack 上对 MCP-Zero 没增益（Figure 5），意味着"主动检索"对精度的杠杆主要在弱模型上。Decision Agent 若以最强基础模型（GPT-4.1+/Claude-4+ 同档）落地，活跃检索的实际增益更可能落在 token 成本侧而非精度侧——这与论题"上下文窗口工程约束"的优先级一致，但需修正"主动检索能提升精度"的潜在过度承诺。

无与论题的直接矛盾。证据强度：APIBank 表 1 的精度数字 [med]——单 benchmark、48 工具规模；MCP-tools 数据集与 [05] 的实验对比 [high]——已被独立第三方在 LiveMCPBench 上复现并基线化。
