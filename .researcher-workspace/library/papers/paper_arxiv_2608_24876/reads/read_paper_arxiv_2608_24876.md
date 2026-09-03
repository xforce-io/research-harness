---
title: "Recursive Experiential-Working Memory Evolution for Long-Horizon Agent Harnesses"
authors: ["Zhaochen Yu","Yingcheng Wu","Zhenfei Yin","Kaiyuan Chen","Zhe Zhao","Mengdi Wang","Shuicheng Yan","Ling Yang"]
paper_id: "paper_arxiv_2608_24876"
source_kind: "arxiv"
source_id: "arxiv:2608.24876"
source_url: "https://arxiv.org/abs/2608.24876"
pdf_url: "https://arxiv.org/pdf/2608.24876"
read_id: "read_paper_arxiv_2608_24876"
kind: library-read
doc_type: "paper"
tags: []
---

继续读取完整 prompt 中的输出路径与现有 Library 笔记结构。继续提取输出规范与论文正文关键实验数字。对照已有 Library read 格式，并补齐论文实验与相关工作细节。再抓 Related Work 与关键实验数字，按 Library 规范写 Relations（只连文献，不连主题笔记）。# Recursive Experiential-Working Memory Evolution for Long-Horizon Agent Harnesses

> 整段历史/初始指令检索技能随交互变长而失准 → Recuris 用可验证工作状态锚定「此刻调哪条经验技能」，再把结构化轨迹交给固定 Meta-Agent 做组件级、验证门控的 Skill Memory 演化 → 基座权重不动，失败即可落成可迁移的记忆补丁。

## Essence

**问题**。长程 agent harness 里，经验记忆若只按初始指令或整段对话检索，会把过时步骤、未完成目标与噪声混在一起，技能调用与当前任务状态脱节；仅靠最终成败信号又无法判断该修技能、状态、调用策略还是校验器。

**做法**。任务内：Working Memory（WM）维护已验证目标状态，在执行事件（如草稿状态变更工具调用）上由调用策略 \(\rho\) 从 Experiential Memory（EM）取匹配技能；环境回执经 checker 才提交状态更新，并写出结构化轨迹 \(\Gamma\)。任务间：固定 Meta-Agent 把失败归因到 \(M=(E,W,\rho,C)\) 的具体组件，写 scoped 补丁，仅在源失败修复且 held-out/anchor 不回归时接纳——外环（基座 LLM、工具、Meta-Agent、门）全程固定。

**证据**。四个长程基准、十个模型上，35/37 个已完成 model–benchmark 对提升任务成功；结构化轨迹把组件故障定位宏准确率从仅 outcome 的 13.0% 提到 64.8%。

**边界**。跨任务演化依赖任务族共享工具/策略/流程结构；无共享结构时（如 Terminal-Bench）跨任务门控可零接纳，且单任务适配的 headline 增益主要由重试预算解释，不宜读成通用无条件 RSI。

## Claims

- 长程 RSI 的瓶颈不在「有没有经验」，而在缺少能持续对齐当前执行需求的紧凑、可信任务状态：仅按初始指令或整段历史检索会随交互变长而失准 [§1]。
- Experiential–Working Memory Coupling 形成闭环：已验证工作状态决定当前需要 → 从 EM 取技能 → 执行反馈经 checker 验证后再更新状态；技能调用在定义好的执行事件上发生，而非一次性灌入或交由模型自决 [§1, §2.2]。
- 可演化对象收窄为 Skill Memory \(M_k=(E_k,W_k,\rho_k,C_k)\)；基座 LLM、工具集、Meta-Agent、定位/打补丁程序与验证门在轮次间固定，递归只发生在 memory-control 层 [§2.1.1, §2.3.4]。
- 结构化轨迹 \(\Gamma_k\) 把每步工作状态、被调技能、动作、观测、提议状态与 checker 判定绑在一起；相对 raw trajectory / 仅 outcome，故障注入实验中组件定位宏准确率分别为 64.8% / 37.0% / 13.0%，增益集中在转录里不可见的 \(\rho\)、\(W\) 故障 [§2.1.2, §3.4.1, Table 4]。
- 候选补丁只改诊断牵涉的组件；须修复源失败并通过含已解 anchor 的 held-out development 回归准则，否则维持原记忆；test 轨迹永不进入定位、打补丁或接纳 [§2.3.2–2.3.3]。
- 消融显示工作状态而非技能内容承载主增益：τ²-Retail 上 EM-only 增益接近噪声、WM-only 与 EM+WM 显著抬升；同库全量注入、由模型自决调用的对照比 Recuris 低约 18 点，且每成功更贵 [§3.3.2–3.3.4, Table 2–3]。
- 关键 harness 机制随域而异（如 Airline 更依赖 write review、Retail 更依赖 status board），故修复目标应从失败轨迹读出，而非设计时固定分配 [§3.3.3]。
- 增益随交互 horizon 扩大而非衰减：最长任务档位相对 base 可达约 +32.2；分离主要在 required-write recall，而非读路径召回 [§1, §3.3.1]。
- 跨任务演化在 held-out 上稳定：约 16 道训练失败蒸馏出的记忆，在 Meta-Agent 未见的 86 题上相对 \(M_0\) 约 +9 至 +17 点；Claude Code 与 DeepSeek Harness 两套 Meta-Agent 实现收敛到相近增益与同类组件修复 [§3.4.2–3.4.3, Table 5–6]。
- 在 mid-sized deployment model 上单源演化的同一包可原样装到未见模型：τ²-Retail 上 GPT-5.6 Sol +17.8、Claude Opus 5 +15.6（至 87.9%）；SkillFlow 上 Qwen3.6-27B/35B 分别 +16.6/+13.5；总表 35/37 对正向 [§3.2, §3.4.6, Table 1, Table 7]。
- Terminal-Bench 2.1 无跨任务共享结构时，跨任务演化 13 轮未接纳任何补丁；within-task adaptation 相对单次 +26.4 几乎全部由 4 次重试预算解释，匹配预算下「学习」项仅 +2.3（\(p=0.774\)）[§3.2, §3.5, Table 8]。

## Assumptions

- 任务族存在可跨题复用的共享结构（共享工具/策略，或 SkillFlow 式流程族）；否则跨任务记忆演化默认无可搬货物，只能退到单任务适配 [§3.2]。
- 冻结的指令调优模型在 temperature 0、共享工具与预算下，配置差可归因于 memory-control 层；τ² 上同一模型还扮演 simulated user [§3.1]。
- 「固定」Meta-Agent（Claude Code / DeepSeek Harness 上的 LLM agent）足以完成组件归因与 scoped patch；学到什么主要由证据池决定，而非某一实现细节 [§3.1, §3.4.3]。
- Checker 完成谓词对着 tool/环境回执（或任务侧 evaluator）判定，不信任模型自述成功；误拒/误接受会写入 \(\Gamma\) 并可归因到 \(C\) [§2.2.3]。
- 单源演化（仅用 mid-sized deployment model 的失败）学到的是模型共享的失败结构，因而同一 \(M\) 可不经再演化迁移到其他模型 [§3.1, §3.4.6]。
- evolve / gate / test 三分法在跑之前固定，且 test 永不进入定位、打补丁或接纳决策 [§2.3.3, §3.1]。

## Method

**前后对比（记忆如何用 / 如何长）。**

| | 常见先验 harness | Recuris |
|---|---|---|
| 检索锚点 | 初始指令或整段 chat history | 已验证工作状态 + 执行事件 |
| 状态更新 | 模型自述或固定规则 | 提议 → checker 对观测判定 → 固定 kernel 只提交被支持变更 |
| 演化粒度 | 常按整次成败重写整库 | 归因到 \(E/W/\rho/C\) 后 scoped patch + held-out 门控 |

**任务内执行环（EM–WM）。**

1. 按 \(W_k\) 初始化工作状态 \(w_0\)：每个 goal 含内容、状态（pending/done/blocked）、证据与可选 blocker。
2. 在执行事件上由 \(\rho_k\) 取技能：τ² 用 call-time（草稿状态变更工具调用时按工具名检索，先返回 synthetic not-executed，再带技能重拟动作）；Terminal-Bench 用 boundary（如 `first_turn`）。
3. 观测返回后，\(U_{W_k}\) 提议下一状态，\(C_k\) 做完成谓词检验，固定 kernel \(K\) 只提交被支持变更。
4. 输出结构化 \(\Gamma_k=(x,\{w_t,E_t,a_t,o_t,\tilde{w}_{t+1},c_t,w_{t+1}\},y)\)，而非仅 \((a_t,o_t)\)。

**跨任务演化环（bounded RSI）。**

1. 定位：\(D_k=A_{\mathrm{fixed}}(\Gamma_k,M_k)\)，把每个诊断失败归因到 \(\{E,W,\rho,C\}\)（修复决策，非严格因果鉴定）。
2. 打补丁：对牵涉组件各写一处编辑，\(M_{k+}=M_k\oplus_{Z_k}\{\Delta m_z\}\)。
3. 门控 \(G_{\mathrm{fixed}}\)：须修复源失败且满足 held-out dev 回归准则才接纳为 \(M_{k+1}\)。
4. 外环不变；递归只改 memory-control 层，使后续状态表示、技能调用与失败观测条件一并改变。

**Test-time adaptation 模式。** 证据池与补丁空间收窄到单任务；隐藏 verifier 只回一 bit；失败后更新经验记忆再试，成功即停；与冻结初始记忆的同预算重试对照，并共享首次 rollout [§2.3.5, §3.5]。

## Eval

- **基准**：τ²-Retail（114）、τ²-Airline（50）— 策略约束工具对话，成功须环境 verifier 全奖励，并分报 read-action / required-write recall；SkillFlow（166 题 / 20 族）— 终身技能发现，程序化 verifier；Terminal-Bench 2.1（87）— 终端任务，主用于 within-task adaptation [§3.1, §3.5]。
- **配置**：benchmark 自带 reference agent；Recuris+\(M_0\)；Recuris+evolved \(M\)；同模型、工具、任务、种子、预算（τ² 含 user simulator）。演化只在 deployment model（doubao-seed-2-0-pro）上跑，再原样装到其他模型 [§3.1–3.2]。
- **指标**：avg@4 任务成功率；required-write / read recall；失败模式相对缩放；held-out Δ vs \(M_0\)；故障注入下 localization 宏准确率；Terminal-Bench 另报 solved-within-budget 与 untruncated avg@4/pass@4 [§3.1–3.5]。
- **主结果**：35/37 对正向；deployment 上 τ²-Retail +23.3†、SkillFlow +16.8†；全库常驻 prompt 对照比 Recuris 多 3111 token 却低约 18 点且每成功贵 46% [§3.2, §D]。
- **机制消融**：EM/WM/model-controlled 调用；域关键组件（write review / status board 等）；fault-injection localization；多 Meta-Agent / 门限；跨任务与跨模型迁移；TTA vs matched-budget retry [§3.3–3.5]。

## Weaknesses

- **「固定 Meta-Agent」淡化了前线成本**：定位与打补丁由 Claude Code / DeepSeek Harness 级 LLM agent 完成，属 agent-judge 成本档；论文强调协议固定与可替换实现，但未系统报告每轮定位/补丁的 token、美元或墙钟，难与「轻量信号 → 更新」叙事对齐 [§3.1, §3.4.3]。
- **τ²/SkillFlow 缺与同预算 sampling/TTS 的硬对照**：主表相对 reference agent 与 memory 消融成立，但未把「多开几次尝试 / parallel sampling」锁进与进化相同的反馈预算；Terminal-Bench 上作者自己把 headline 拆成重试，其余基准的表观 Δ 仍可能混有执行稳定性 [§3.2, §3.5]。
- **跨任务可搬性被任务结构设计预置**：SkillFlow「流程共享」与 τ² 共享工具/策略是正增益主场；无共享结构时 13 轮零接纳——摘要仍把 Recuris 定位为通用长程 RSI 基础，弱化了「共享结构」前置条件 [§3.2, abstract]。
- **演化证据池极小（约 16 训练失败题）却支撑大范围迁移叙事**：held-out +9–17 点令人印象深刻，但也意味着补丁可能编码窄失败族；Airline held-out 因无可改进剩余失败而与 0 不可分，说明「迁移」高度依赖 held-out 是否仍含同类失败 [§3.4.5–3.4.6]。
- **Checker / task-specific evaluator 的 oracle 依赖未量化**：完成谓词可依赖「结构化 tool receipt 或任务侧 evaluator」；若 evaluator 接近官方成功准则，则 WM 验证强度部分来自评测侧信号而非可部署观测，论文未报告重叠度 [§2.2.3]。
- **Figure 1 的失败模式条以 agent alone=100 相对缩放**：正文更扎实的是 write recall 与消融；把相对缩放读成绝对失败率会高估生产收益 [Fig. 1, §3.3.1]。

## Relations

- builds-on Agent Skills (Anthropic, 2025) [high]：EM 条目采用 agent-skill 标准格式；论文在 Related Work 中明确以此为可复用经验的表示载体 [§2.1.1, §4.1]。
- extends StructAgent / Magentic-One 等工作记忆线 [med]：同属显式任务状态/进度账本；Recuris 进一步把状态提议与观测侧 checker 分离，并把已验证状态接到技能检索上，而非只服务下一步动作 [§4.2]。
- competes-with SkillOpt / SkillComposer [med]：三者都从反馈改技能资产；后两者优化或合成 skill 文件本身，Recuris 把可演化面扩成 \(E/W/\rho/C\) 四元组并坚持外环固定、验证门控 [§4.1]。
- competes-with AutoGuide / SGDR [med]：都做执行期条件检索；AutoGuide/SGDR 以原始观测相似度匹配情境，Recuris 以 harness 已确认的任务进度为检索锚点 [§4.1]。
- competes-with Voyager / ExpeL / AWM 等「存什么经验」路线 [med]：同属 experiential memory 增强；Recuris 主张关键缺口是何时/如何调用，而非仅经验条目形态 [§4.1]。
- orthogonal 权重量更新式 RSI / 自改 agent 程序 [high]：论文明确把递归限定在外部 memory-control 层，不改基座权重与完整 agent 程序；与改 θ 或重写整套 harness 的 RSI 路径分轨 [§2.3.4, §4.3]。

## Takeaway

- **值得抄的方法**：把「状态锚定调用」与「组件可观测轨迹」做成同一套闭环——没有 \(\Gamma\)，scoped 演化只是口号；有了 \(\Gamma\)，失败才能落到 \(E/W/\rho/C\) 之一。
- **需要复核**：跨任务增益是否离开共享结构仍成立；主表 Δ 相对 matched-budget 重试/TTS 还剩多少；checker/evaluator 与官方成功准则的重叠有多大。
- **若只记一句**：Recuris 不是让模型自己越改越强，而是冻结外环，让可验证工作状态决定此刻调用哪条经验，再把门控后的记忆补丁变成下一轮执行的条件。
