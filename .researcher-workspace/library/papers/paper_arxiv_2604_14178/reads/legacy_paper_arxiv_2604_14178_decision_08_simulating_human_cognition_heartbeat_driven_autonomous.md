---
title: "Simulating Human Cognition: Heartbeat-Driven Autonomous Thinking Activity Scheduling for LLM-based AI systems"
paper_id: "paper_arxiv_2604_14178"
source_kind: "arxiv"
source_id: "arxiv:2604.14178"
source_url: "https://arxiv.org/abs/2604.14178"
kind: legacy-topic-read
topic: "decision"
source_note: "notes/active/08_simulating_human_cognition_heartbeat_driven_autonomous.md"
---

# Legacy topic read

This Library read artifact was backfilled from an existing topic note. It preserves historical reading state; it was not produced by the current Library deep-read runner.

## Source topic note

---
zone: active
pin: false
score: 0
dwell: 0
paper: "Simulating Human Cognition: Heartbeat-Driven Autonomous Thinking Activity Scheduling for LLM-based AI systems"
arxiv: "2604.14178"
authors: ["Hong Su"]
year: 2026
venue: "arXiv preprint v1（Chengdu University of Information Technology，单作者）"
note_number: 8
---

## Claims

- 现有 LLM agent 框架（AutoGPT、ReAct、CAMEL）在"反应式或目标驱动"范式下被外部 prompt 触发推理循环，无法自主管理认知活动的时序流 [8: §II.A]。
- 元认知工作（Reflexion、Self-Refine、CoT、ToT）的反思机制是"failure-triggered 或 task-completion-triggered"，只在错误发生或终点到达后才介入；何时启动深度推理仍然是启发式或人工预设 [8: §II.B]。
- 提出 Heartbeat Scheduling Controller (HSC)：通过周期性"心跳信号"触发顶层调度器，在内部状态 `s_t^int` 与环境输入 `s_t^env` 之上选择认知模块（Planner / Critic / Recaller / Dreamer），把 agent 从被动响应者转换为自发主动思考者 [8: §I, §IV]。
- 调度策略 `π_θ(s_t, d_t) → z_t` 不是预定义的，而是通过历史交互日志 `D_t` 学习得到，参数 θ 通过最大化 `J(θ) = E[Σ R_t]` 优化 [8: §III, Eq. 1, §III-C, Eq. 12]。
- 双层分层模型：宏观层（Task Scheduling，`π_macro(s_t, d_t) → s_{t+1}`）决定"做什么高层认知任务"，微观层（Detailed Execution，`π_micro(s_t, d_t) → a_t`）决定"如何执行该任务"；两层共同构成完整调度 `Π = {π_macro, π_micro}` [8: §III-A, Eq. 2-3]。
- 引入 Dream Mode：在无外部任务时心跳触发"内部独占思考"，承担三类功能——①记忆整理（数据压缩、语义索引、剪枝）；②合成经验生成（假设测试、反事实推理、技能复演）；③自主目标设定 [8: §IV-C]。
- 状态增强 `s_t' = φ(s_t, h_t)`（其中 `h_t = Encoder(a_{t-k}, …, a_{t-1})`）通过把近期动作历史嵌入到状态表示中防止重复循环，无需显式启发式规则 [8: §V-C, Eq. 19]。
- 复合奖励 `R_total = α · R_external + β · R_internal` 同时优化外部任务完成与内部认知效率（目标对齐、问题解决效能、目标迁移） [8: §IV-B, Eq. 16]。
- 实验：基于 1,800 天合成数据训练 LSTM 多日注意力模型预测 6 类日常认知活动，达成 6/6 类全覆盖、预测分布熵 2.31 bits 接近真值 2.35 bits、稀有动作 recall 78.3%；新增 1 类"Recalling What is Important"（动作 ID 6）后模型仍能预测，但时间错位（真值 12 时，预测 16 时） [8: §VI-A.3, §VI-B.3]。

## Assumptions

- 人类认知活动可被简化为 6（或扩充到 7）个离散动作类别（Idle、Execute、Summarize、Imagine、Recall、Rest），且这些类别在小时粒度上可清晰区分——这一离散化假设直接来自论文的合成数据生成代码，未做认知科学层面的验证 [8: §VI-A.1]。
- 心跳周期 Δt 是一个超参数，论文从未讨论其值（实验中默认按"小时"采样）；心跳频率与认知活动效果的关系完全没被研究 [8: §IV, §VI]。
- §III-IV 中描述的 LLM agent 架构（Planner / Critic / Recaller / Dreamer 等可插拔认知模块）与 §VI 实验中的 LSTM 序列预测模型属于同一框架——这一对应关系是论文隐含的，但实际上实验完全没用 LLM，认知模块也未实例化 [8: §III-D vs §VI]。
- "认知活动调度策略可以独立于具体认知模块的实现"被作为 functional approach 的合理性根据；论文未论证"模块即插即用"在真实系统中的可行性，仅在合成动作序列上做扩类实验 [8: §II.D, §VI-B]。
- Reward 信号 `r_k = R(outcome_k)` 可由系统自评估（self-evaluation）得到——但具体的 self-evaluation 机制（什么样的 LLM-as-judge、什么样的内部度量）从未给出 [8: §III-C, Eq. 9]。
- 实验中 "Recalling the Past" 出现的概率随历史 "Execute a Task" 序列增加而上升的规则被当成"人类行为先验"，但这只是论文作者在合成代码里写入的概率规则，并非任何认知心理学文献的引用 [8: §VI-A.1]。
- §III-C 提到的 "delayed feedback" 与 eligibility traces / TD learning 在实验中未实例化——所有训练都是 supervised learning on synthetic next-action prediction [8: §III-C.3 vs §VI-A.2]。

## Method

**核心论点：把 LLM agent 的"何时思考"从硬编码工作流升级为可学习策略，靠周期心跳触发，靠历史轨迹优化。**

**❶ 心跳调度控制器（HSC, §IV）。** 在每一 tick `t_k = k · Δt`，调度器读取联合状态 `s_k = [s_k^int, s_k^env]`，应用策略 `π : S → A` 选择本拍认知活动 `a_k = π(s_k; Θ)`，其中动作空间 `A = {execute_task, recall, analyze, dream, …}`。心跳的功能：
1. **解耦触发与外部事件**：即使没有用户输入也会周期性评估内部状态、决定是否反思/规划/记忆整理。
2. **同步学习窗口**：为 §III-C 的连续学习提供一致的时间框架，让"动作 → 延迟反馈"的归因有 tick 级时戳。

**❷ 双层调度（§III-A）。** Macro `π_macro(s_t, d_t) → s_{t+1}` 决定状态机层级跳转（解决数学题 / 回忆事件 / 进入 Dream），转移发生在空闲、计划切换或复合任务阶段切换。Micro `π_micro(s_t, d_t) → a_t` 在给定 macro state 下挑选细粒度动作（检索类似题、识别变量）。Micro 层还含辅助活动如经验摘要 `E_t`，更新 `d_{t+1} = d_t ∪ {E_t}`。

**❸ Self-Activity-Driven Autonomous Learning Mechanism（CLSCA, §III-C）。**
- 自数据轨迹 `τ_t = {(o_k, a_k, r_k)}_{k=1..t}`，其中 `r_k = R(outcome_k)` 是 self-evaluated reward。
- 动态扩类：`A_{t+1} = A_t ∪ {a_new | a_new = Gen(τ_t)}`，认知模块可在线增删而无需架构改造。
- 经验数据集 `D_{t+1} = D_t ∪ {C(τ_t, r_t)}` 由 curation 函数 C 自动筛选+打标。
- 参数更新 `Θ_{t+1} = Θ_t + η ∇_Θ E[Σ r_k]`。
- 论文强调"包容次优结果"——次优轨迹也进入训练集，靠 reflective correction 修正策略。
- 延迟反馈处理：当 `R_{t+Δt}` 在动作 `t` 之后到达，按 `∇J(θ) ≈ E[Σ ∇log π_θ(a_t|s_t) · R_{t+Δt}]` 把延迟奖励归因回原动作（论文形式上写出但未实例化）。

**❹ Dream Mode（§IV-C）。** 心跳调度器在外部任务空闲时切到 Dream，承担三件事：(a) 把短期 buffer 经验压缩 / 重新按语义索引 / 剪枝低价值数据，固化进长期记忆；(b) 内部模拟——假设检验、反事实回放、技能复演——产物作为下一轮训练数据；(c) 自主目标设定：Dream 中识别能力缺口、生成 intrinsic goals 进入后续 tick 的调度。Dream 与 Active 的切换由心跳调度（高优先级外部事件可立即唤醒）。

**❺ 防重复 + 环境响应（§V-C）。** 状态增强 `s_t' = φ(s_t, h_t)` 把动作历史 `h_t` 编码进状态，使得即便外部环境不变，内部"effective state"也变，从而 `π(a|s_t')` 自然偏离近期动作。环境动力 `P(s_{t+1}|s_t, a_t) ≠ P(s_{t+1}|s_t, a_t)_{t-Δ}` 由心跳高频采样捕获，资源紧张时调度器自动压制高成本 Dream 活动。

**❻ 实验实现（§VI）。** 与 §III-IV 的 LLM agent 描述完全脱节——实验是一个 Sequence-to-Sequence + 多日注意力的 LSTM 模型 `humanMultiDayAttentionModel`：
- 数据集：合成 1,800 天 × 24 小时 × 6 类动作；含 Weather（4 类）、Temperature（连续）、Day/Night、正余弦时间编码。
- 历史窗口：HISTORY_DAYS = 3，预测下一天 24 小时活动序列。
- 训练划分：70/15/15 顺序切分，Test 集 N=270 天 / 6,480 小时。
- 模型：每天用 2 层 LSTM（hidden=128）独立编码，跨日做带位置编码的多头自注意力；解码端用 additive attention。
- 训练：30 epochs、Adam lr=1e-3、Batch=32、Teacher Forcing 从 0.8 衰减到 0.2。
- 扩类实验：在合成数据生成器里加入"Recalling What is Important for Current"（ID 6），其条件概率仅在 12-14 时 + Sunny + Temperature > 28°C 时为正（增量 ≈0.12，从 Idle 与 Recall 处转移）；模型架构不变，重训。

## Eval

**两个实验：**

1. **6 类活动序列预测（§VI-A）。** 在 N=270 天测试集上聚合预测，度量：
   - **动作覆盖率**：6/6 (100%) 类全覆盖。
   - **熵对齐**：预测熵 H_pred = 2.31 bits，真值熵 H_true = 2.35 bits（ΔH ≈ 0.04）。
   - **稀有动作 recall**：Idle + Imagine 两类合并 recall = 78.3%。
   - **可视化对比**：Figure 1 给出**单日**预测序列 vs 合成真值的离散动作折线图，显示 24 小时内匹配大致节律。

2. **7 类扩类实验（§VI-B）。** 同样架构、同样训练设置，加入动作 6 后：
   - Figure 2 给出**单日**对比：真值动作 6 出现在 12 时，预测出现在 16 时（4 小时错位），但全天频率均为 1 次。
   - 模型保持高层模式（早晨 Rest → 工作 → 傍晚放松），未因新动作引入而退化。
   - 没有报告测试集级聚合指标（覆盖率、熵、recall），仅展示一个单日例子。

**基线缺失：** 论文未与任何 baseline 比较——没有 ReAct / Reflexion / AutoGPT 等 §II 提及的同类系统，也没有"无 attention 的 LSTM"、"单日 history"、"Markov 假设"等消融基线。

**度量缺失：** 经典序列建模指标（per-step accuracy、F1、BLEU、perplexity）一概未给；分布熵作为唯一对照指标对"调度质量"几乎不构成证据——一个均匀随机预测器都能给出接近真值熵的结果。

## Weaknesses

- **"实验"完全不验证论文论点。** §III-IV 通篇讨论 LLM agent + 可插拔认知模块（Planner / Critic / Recaller / Dreamer）+ Dream Mode 的元认知架构，§V 给出形式化策略学习与状态增强，但 §VI 的实验是一个 LSTM 在合成动作序列上的 next-action 预测——既没有 LLM、没有任何认知模块的实例化、也没有 Dream Mode 的运作、更没有 self-activity 驱动的连续学习。论文的核心架构主张全部未被实证 [8: §III-IV vs §VI]。
- **合成数据是循环验证。** 训练数据由作者写的概率规则生成（包括 hour-specific preference、weather-dependent modification、Execute → Summarize 序列依赖），实验"成功捕捉"的所有"人类思维节律"实际上是这些手写规则。模型学到的并不是认知调度，而是数据生成器的反向工程。论文未在任何真实人类行为日志或真实 LLM agent trace 上验证 [8: §VI-A.1]。
- **扩类实验只有单日定性图。** §VI-B 全部论据是 Figure 2 一张单日曲线 + 文字描述"模型成功整合第 7 类"。预测的动作 6 在时间上错位 4 小时（12 时真值 → 16 时预测），论文却归类为成功；缺少测试集 N=270 天的扩类性能聚合（如准确预测动作 6 的天数 / 预测时戳分布 / 与真值时戳的均方误差）[8: §VI-B.3]。
- **基线全部缺失。** §II 列了大量相关系统（AutoGPT、ReAct、CAMEL、Reflexion、Self-Refine、CoT、ToT、ACT-R、SOAR），但实验中没有任何一个被作为基线复现或对比。论文宣称的"超越反应式范式"完全没有量化证据。一个朴素的"按小时频率统计的多项式分类器"都未被作为基线 [8: §II vs §VI]。
- **理论部分与实验完全脱钩。** §III-C 形式化的 `J(θ) = E[Σ R_t]`、§V 的策略梯度 `∇J(θ) ≈ E[Σ ∇log π_θ · R_{t+Δt}]`、§V-C 的状态增强 `s_t' = φ(s_t, h_t)` 在实验中均无实例化——§VI 的训练是 supervised cross-entropy on next-action labels，没有 reward、没有策略梯度、没有 history embedding 的特殊处理。理论与实验是平行宇宙 [8: §III-C, §V vs §VI]。
- **Dream Mode 完全未评估。** §IV-C 用近一页篇幅描述 Dream Mode 的三大功能（记忆整理 / 合成经验生成 / 自主目标设定），但 §VI 实验中既无空闲态识别、也无 Dream 触发、更无任何 Dream 产物（重新索引的记忆 / 反事实回放 / 自主 intrinsic goal）的度量。Dream Mode 是论文最显眼的设计点之一，证据为零 [8: §IV-C vs §VI]。
- **可学习性章节是定义而非证明。** §V "Theoretical Analysis of Schedule Learnability" 标题暗示形式化证明，但实质是策略学习目标函数的复述（Eq. 17 = 标准 RL discounted return）+ 探索-利用混合策略（Eq. 18 = ε-greedy）+ 状态增强的描述（Eq. 19）。没有可学习性的收敛证明、样本复杂度上界、或 PAC 类保证。把"可学习"作为"用 RL 形式表达"的同义词使用 [8: §V]。
- **动作类别选择无理论根据。** 6 个核心类别（Idle / Execute / Summarize / Imagine / Recall / Rest）从未引用任何认知科学文献来论证为何这是"人类认知活动"的合理离散化。Imagine 与 Dream 的关系、Summarize 与 Recall 的边界都未澄清。改成 5 类或 8 类是否会改变结论无从判断 [8: §VI-A.1]。
- **心跳周期完全未敏感性分析。** 心跳是论文核心机制，但 Δt 取何值对调度效果的影响从未被研究。实验默认按"小时"切片是数据生成器的产物，与认知节律无任何论证关系。是否更短（分钟）或更长（数小时）会改变结果，论文沉默 [8: §IV vs §VI]。
- **单作者 + 自引用链。** 引文 [12]、[16] 均为同一作者 Hong Su 的另两篇 arXiv preprint（2602.11516 / 2601.13887），未被任何第三方验证。论文整体未经同行评议（arXiv preprint v1，2026-03-28），且单作者发表降低了交叉审视的可信度 [8: References §12, §16]。
- **未来日期 arXiv ID 与体例错位。** arXiv ID 2604.14178 对应 2026-04 月份提交，但论文中已引用 [4] React-LLM 标注为 AAAI 2026 vol. 40 no. 31 pp. 26337-26345（一个不存在的卷号），引文质量整体可疑——这影响其他引用（包括同行工作）的可核实性 [8: References §4]。
- **声明的"continual / online learning"未实证。** §III-C 反复声称"system can learn from suboptimal trajectories"、"online updates or batch training"、"experience replay"——但实验是固定 1,800 天数据集 + 30 epochs 离线监督训练，没有 online 步骤、没有重放、没有由 reward 驱动的更新。"持续学习"的所有论据停留在描述层面 [8: §III-C vs §VI]。
- **复合奖励无实例化。** Eq. 16 的 `R_total = α · R_external + β · R_internal` 中的 R_internal（target alignment / problem-solving efficacy / goal transfer）三个组件如何度量从未给出。给定 LLM agent，"safety alignment score"、"problem solving novelty score" 这些显然不是直接可观测的标量 [8: §IV-B]。

## Relations

- **competes-with 06_why_do_multi_agent_llm_systems [med]**：MAST 给出 14 类失败模式的实证分类（n=1642 traces），其中 FM-1.3 Step Repetition (15.7%) 与 FM-1.5 Unaware of Termination (12.4%) 都是"何时停 / 何时再思考"的调度问题。本论文 §V-C 的状态增强（防重复）正面对应 FM-1.3，HSC 的"心跳触发反思"对应 FM-1.5。但 MAST 提供的是基于真实 MAS 框架（ChatDev、MetaGPT、AppWorld 等）的诊断证据，本论文提供的是合成 LSTM 实验——前者经验扎实、后者证据为零。两者在"调度问题真实性"上互证，但本论文未把 MAST 失败模式作为评测靶点。
- **orthogonal to 03_why_reasoning_fails_to_plan_a [med]**：FLARE 把 ReAct/CoT 的步进推理形式化为贪心策略并证明其在多步规划上的结构性失败，主张前瞻性 MCTS 类规划。本论文的 HSC 同样不满步进式被动推理，但走的是另一条路——"调度何时启动深度推理"而非"在每步内做更深规划"。两者层级不同：FLARE 改造单次推理的深度，HSC 改造跨次推理的频率。可叠加，但本论文未触及 FLARE 形式化的"贪心策略局限"，也未提供任何证据说明心跳调度能修复多步规划失败。
- **builds-on Reflexion (Shinn et al., 2023) [high]**：本论文 §II-B 把 Reflexion 列为元认知反思的代表，明确承认"reflection on past actions and outcomes"路径，但批评其为"failure-triggered"。HSC 的差异是把反思变成 proactive、由心跳周期驱动、可在空闲时进行。该 builds-on 是 §II-B 显式陈述的。
- **builds-on Self-Refine (Madaan et al., 2023) [high]**：同 §II-B，本论文把 Self-Refine 列为"iterative feedback"的代表，并把 HSC 定位为"跨更高层调度反思"的扩展。
- **builds-on ReAct (Yao et al., 2023) [med]**：§II-A 显式引用 ReAct 作为"reasoning + action interleave"的代表，承认其外部触发循环；HSC 主张内部触发。本论文标题明确把自己定位为对 ReAct 反应式范式的补充。
- **extends 经典认知架构 (ACT-R / SOAR) [med]**：§II-D 显式引用，并主张"functional 而非 structural"——保留人脑认知活动的功能模块（Planner / Recaller / Dreamer）但放弃 ACT-R/SOAR 的产生式规则结构。论文称"pluggable cognitive modules"是核心差异，但从未在实验中实例化任何模块。
- **orthogonal to 07_mcp_zero_active_tool_discovery [low]**：MCP-Zero 的"active tool request"也是"主动驱动"思路，但驱动对象是工具检索，本论文驱动对象是认知活动调度。两者层级互补：HSC 决定何时进入"工具调用阶段"，MCP-Zero 决定该阶段如何写请求。但本论文完全不涉及工具，故关系松散。
- **orthogonal to 05_agent_as_a_graph [low]**：Agent-as-a-Graph 处理工具/agent catalog 上的检索，与认知活动调度层级不同。
- **orthogonal to 04_rethinking_the_value_of_multi_agent [low]**：OneFlow 论证同质 MAS 可折叠为单 LLM 顺序执行；HSC 在单 agent 范畴讨论调度，与 OneFlow 立场不冲突。
- **orthogonal to 02_graph_of_agents [low]**：GoA 处理多 agent 输出协调，本论文专注单 agent 内部时序。
- **orthogonal to 01_co_evolving_llm_decision_and_skill [low]**：COS-PLAY 关注技能库进化（学新能力），HSC 关注"何时调用已有能力"（选择时序），问题域不同。

### Relation to thesis

直接命中论题"常驻 Runtime 的学术对照"研究缺口（thesis Design Context goal 2: Heartbeat + Cron）——但作为**反证案例**而非正面支撑。三点对 Decision Agent 的影响：

1. **Heartbeat 作为"主动 Agent"机制的学术立项已经存在，但当前文献的实证基础非常薄弱。** 本论文是迄今唯一直接以"heartbeat scheduling"为标题的 LLM agent 工作，正面对应 Decision Agent 0.8 路线图 §3.2 的 Heartbeat + Cron 常驻 Runtime + Reflection 设计。但论文的实验（合成 1,800 天动作序列上的 LSTM 监督训练）与论文主张（LLM agent + 可插拔认知模块 + Dream Mode + 自主学习调度）完全脱节。Decision Agent 不能从本论文获得任何工程参数或实证证据；相反，本论文的状态揭示**"主动调度的学术稀缺"是真实的——稀缺到首篇相关论文实质上没做相关实验**。这与 thesis 中"Heartbeat+Cron 学术稀缺，遇到必须收"的预判一致，但同时降低了"通过文献合成获得设计指南"的预期 [8: §VI vs §III-IV]。

2. **论文的状态增强机制 `s_t' = φ(s_t, h_t)` 对 Decision Agent 防重复有形式化参考价值，即使实验未验证 [low]。** §V-C 把"近期动作序列"嵌入到状态表示中以避免循环，是一个清晰的工程化思路。Decision Agent 在 Dolphin 状态机层面已有任务状态持久化，但"避免 step repetition"（MAST FM-1.3, 占 15.7%）的具体机制仍可借鉴本论文的状态增强思路——把 trace 中近 k 步动作哈希注入下一步上下文，比纯启发式黑名单更鲁棒。这一对接是 thesis 的"协同失真量化"研究目标的一个工程化候选方案。

3. **Dream Mode 与 Decision Agent 跨会话 Memory 设计目标 §3.5 的对接 [low]。** §IV-C 的 Dream Mode 三大功能（记忆压缩 / 语义重索引 / 剪枝）与 thesis Design Context goal 5（User/Role/Org 三层记忆 + 治理与遗忘机制）方向一致——尤其"识别并归档低价值数据"的剪枝思路对 Decision Agent 的"记忆库不会无限膨胀"目标有形式化映射。但本论文的 Dream Mode **完全未实现**（§VI 实验中无任何 Dream 触发），无法作为算法参考。Decision Agent 若实现层级化 Memory，需自行设计衰减/合并算法，本论文仅提供问题分解的语言。

**可证伪点检查：** 与 thesis 中"Heartbeat + Cron 是必要 Runtime 能力"无矛盾，也无新的支持；本论文未涉及"治理风险 vs 收益"权衡，因此不构成对该论点的证伪信号。

**证据强度：** 全文 [low]——单作者、未同行评议、合成数据、无基线、理论与实验脱钩。该论文应作为"问题陈述的语言资源"（如 Dream Mode、HSC 这类术语）使用，不应作为机制有效性的证据。在 landscape 中应明确标注其学术成熟度低，避免被后续合成误用为"主动调度有效"的引用。
