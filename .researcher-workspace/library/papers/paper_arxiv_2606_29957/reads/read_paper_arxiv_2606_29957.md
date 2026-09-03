---
paper_id: "paper_arxiv_2606_29957"
source_id: "arxiv:2606.29957"
kind: library-read
tags: []
---

# SWE-Together: Evaluating Coding Agents in Interactive User Sessions

> 从 11,260 条真实 user–agent coding 会话中重建出 109 个可验证的多轮交互任务，用 anchored LLM user simulator 回放用户纠偏，在衡量最终 patch 正确性之外追加 "User Correction" 这一交互成本维度。

## Claims

- SWE-Together 从 11,260 条真实记录的 user–agent coding 会话中筛选并转化为 109 个仓库级任务，转化率 0.97% [§2.1, Table 1]。
- User Correction 与模型能力强负相关：与 pass@1 的 Pearson 为 −0.92，与 stable solve rate 为 −0.84，与 mean judge score 为 −0.93 [§3.1]。
- Claude Opus 4.8 在四项正确性指标上均领先（pass@1 63%、SSR 59%、pass2 52%、mean judge 0.801），同时需要最少的纠偏轮次（User Correction 1.38）[§3.1, Table 2]。
- 人类标注者无法可靠区分模拟用户轨迹与真实用户轨迹：Turing pass rate 46%，95% CI [40.5, 51.6]%，区间包含 50% 的随机水平 [§3.3]。
- Intent Coverage 在模型队列间基本稳定（七个队列中六个落在 0.70–0.72），支持模拟器跨 agent 的可比性 [§3.2]。
- reference-patch baseline 仅达到约 78% 通过率（而非 100%），因为约 35% 的未满足 goal 是流程性要求（诊断、回答、解释），最终 patch 无法表达 [§3.1]。
- pass2 ≤ pass@1，因为 pass2 要求两次 replicate 同时超过阈值 τ=0.85，对 run-to-run 方差惩罚最强 [§3.1]。
- 评估框架将 User Correction 与 Intent Coverage 分离：前者是面向 agent 的交互信号（用于排名），后者是模拟器保真度自诊断（不用于排名）[§2.3.2]。
- SWE-Together 在 Table 3 的五个维度（repo-level / agent-env multi-turn / interactive replay / real task source / real user session）上是唯一全部满足的基准 [§4, Table 3]。

## Assumptions

- 四个 Hugging Face 上游来源（DataClaw、Pi-staging、Hyperswitch、SWE-chat）的 11,260 条会话能代表真实世界 coding-agent 使用场景 [§2.1, Table 1]；论文未论证这些来源的代表性或领域覆盖偏差。
- 一个以 session analysis 为条件的 LLM user simulator 能忠实保留原始用户的意图与介入顺序 [§2.2]；该假设仅由 Intent Coverage 自诊断与 46% Turing 通过率佐证，没有 ground-truth 意图标签做校验。
- agentic rubric judge 的 Phase-1 rubric（离线派生、可能查阅 reference patch）是 implementation-agnostic 且跨 agent 可比的 [§2.3.1]；但 rubric 作者 LLM 及其 prompt 未做 bias 审计。
- k=2 replicates 足以可靠估计 pass@1/SSR/pass2 [§3.1]。
- opencode 是一个中性的、不偏向特定模型的公共 harness [§3.1]。
- multi-label tagger 对 simulator message 的 correction/nudge 标注是可靠的 [§2.3.2]；论文未报告该 LLM 标注器的 inter-annotator 或 self-consistency。

## Method

三阶段 pipeline：

1. **Session-to-Task 构造** [§2.1]：(i) 确定性资格过滤——要求多条真实 user message、具体代码改动、可识别的公开成熟仓库、且最终改动主要由 agent 而非人类作者完成；(ii) 可行性筛查——LLM judge 在紧凑 session 摘要（仓库元数据、消息/工具/编辑计数、工具分布、节选 user message、改动文件路径、截断 shell 命令）上判定主交付物是否可在本地复现，拒绝依赖外部状态（PR 管理、部署、私有凭证、live-service）的交付物；(iii) 沙箱任务构造——task-gen agent 在隔离沙箱内 clone pinned commit、识别本地 setup/test 命令、写 verifier 产物与 task-specific user-simulation prompt。输出：原始 session 记录 + 初始 user 指令 + pinned 环境 + 确定性 verifier 产物 + task-specific 模拟 prompt。

2. **User Simulator** [§2.2]：anchored + state-conditional。每完成一个 agent turn，回放流程总结 live trajectory 并询问 simulator 一次，simulator 输出一个结构化动作：no-op（默认，沉默）/ question / redirect / new-requirement / check-external。条件输入包括最近 agent 活动、最新回复、已用时间、观察到的仓库变更、以及 simulator 自身历史决策。每个 task 的 simulator 还锚定一个从原始会话重建的 session analysis（用户目标、约束、各 follow-up 的触发条件），避免两种失败：固定回放在 agent 走不同路径时 timing 错位，通用模拟会偏离原始任务。

3. **Evaluation** [§2.3]：两维。(a) Task correctness = 确定性 verifier + agentic rubric judge。Phase-1 每 task 跑一次派生加权 rubric（可查阅 reference patch 以识别 recorded solution 的行为，但产出 goal 为 behavioral 且跨 agent 复用不变）；Phase-2 把同一冻结 rubric 应用到每个候选仓库状态，对每个 goal 给出二元 met/not-met + 证据，加权求和得 score∈[0,1]（`score = round(Σ w_g·I[g met], 2)`）。host-side validator 检查 goal 覆盖、权重归一化、决策与分数一致性。(b) User-simulator behavior：Intent Coverage = `round(0.70·I_recall + 0.30·I_precision, 2)`，recall 权重更高因为漏掉原始意图会改变交给被评 agent 的任务；User Correction = `N_correction + 0.2·N_nudge`，由 multi-label tagger 对每条 simulator message 标注（三层：corrective 层含 correction/nudge，non-corrective ask 层含 request/question/verification，workflow/approval/context 不计入），先 task 内对 replicate 平均再跨 task 平均。

## Eval

- **数据**：109 个从真实会话重建的仓库级任务，源自 11,260 条真实 user–agent coding 会话（四源：Hyperswitch 支付代码库 9/784、SWE-chat 多 harness 48/5,851、Pi-staging 23/2,397、DataClaw 29/2,228）[§2.1, Table 1]。
- **被评模型**：7 个前沿模型——Claude Opus 4.8/4.6、GPT-5.5、GLM-5.2/5.1、DeepSeek-V4-Pro、MiniMax-2.7；公共 harness `opencode`；每 task k=2 replicates [§3.1]。
- **指标**：四项正确性指标 pass@1、SSR、pass2、MeanJudge（阈值 τ=0.85），交互诊断 User Correction（↓）、Intent Coverage，以及 tokens/task、min/task 效率 [§3.1]。
- **基线**：reference-patch oracle（mean judge 0.90，~78% 通过率，在 93/109 个有可提取 patch 的 task 上评估）[§3.1]。
- **Turing 测试**：4 名标注者、156 个轨迹对、312 个 judgment，forced 2AFC 选"哪条是真实用户"；simulator Turing pass rate 46% [§3.3]。
- **与既有工作对照**：Table 3 在 repo-level / agent-env multi-turn / interactive replay / real task source / real user session 五个维度上对比 SWE-bench family、Terminal-Bench、MINT/ConvCodeWorld、CAB、RECODE-H/FronTalk、BigCodeArena/CodeChat、SWE-chat；SWE-Together 是唯一五项全勾的 [§4, Table 3]。

## Weaknesses

- 模拟器默认 no-op 会让它在真实用户本该纠偏的时机保持沉默，但论文未报告这种 "missed correction window" 的发生频率；Intent Coverage 衡量意图是否被表达，却不衡量反馈 *时机* 是否匹配真实用户 [§2.3.2]。Turing 测试让标注者在配对轨迹中选"哪条是真实的"，测的是风格不可区分性，而非介入时机/覆盖的保真度——一条模拟轨迹读起来自然却可能在系统性错误的时刻介入。
- rubric judge Phase-1 可查阅原始 reference patch 来"识别 recorded solution 的行为"[§2.3.1]，论文声称 goal 是 behavioral 且复用不变，但未审计 rubric goal 是否编码了 reference patch 的实现特定细节（如特定 helper 函数名）从而惩罚选择替代实现的 agent——这恰好是论文自己引用 OpenAI/DeepSWE 批评的 verifier-misalignment 失败模式 [§2.3.1]。论文没有测量 rubric 对替代实现的一致性。
- k=2 replicates 下 pass2（两次都需通过）每 task 是非常嘈杂的二元量；论文把 pass2 报到小数点后一位，但所有指标都没给置信区间或 bootstrap。"GLM-5.2 pass2 高于 GLM-5.1（42% vs 35%）"的稳定性论断 [§3.1] 在 k=2 下可能落在 run-to-run 噪声之内。
- 0.97% 转化率被框定为"刻意高精度"，但论文未刻画选择偏差——哪类会话/任务被系统性排除 [§2.1]。四源偏斜（社区贡献、Pi staging pipeline、单一支付代码库、多 harness session），对其他领域/仓库的泛化性未检验。
- User Correction 权重（correction=1.0、nudge=0.2）与 Intent Coverage 权重（recall=0.70、precision=0.30）是断言设定的，没有敏感性分析；标题性 Pearson −0.92 在替代权重下可能漂移，且无鲁棒性检查 [§2.3.2]。
- User Correction 的 multi-label tagger 是一个 LLM，其标注可靠性（inter-annotator 或 self-consistency）未报告，而 User Correction 是论文核心交互指标 [§2.3.2]。
- 成本与能力"弱耦合"的论断 [§3.1] 未做统计检验（GPT-5.5 在两个成本轴上同时最省且能力第二），可能受各模型推理端点 serving 位置/基础设施影响，论文自己也承认延迟差异可能被 serving 位置影响却未控制 [§3.1]。
- 论文将 7 个模型混合了已发布与未发布版本（Claude Opus 4.8/4.6、GPT-5.5、GLM-5.2/5.1 等），模型版本/日期与 public checkpoint 不可对应，复现窗口极窄，且无法排除在 benchmark 上做针对性适配的可能 [§3.1, Table 2]。

## Relations

作为独立 Library read（无 topic `notes/` 目录），下列关系锚定于论文自身 Related Work [§4, Table 3] 中显式引用的基准与方法，按 §4 的三类划分组织。

- builds-on SWE-bench (Jimenez et al., 2024) [high]：SWE-Together 保留了 SWE-bench 的仓库级、tool-using、可执行验证设定，在其之上追加 interactive user-correction 回放 [§1, §4, Table 3]。论文显式把 SWE-bench family 列为"agent-env multi-turn 但 fixed user request"的前身。
- extends MINT / ConvCodeWorld (Wang et al., 2024; Han et al., 2025) [high]：两者用 LLM-driven simulated user 对标准代码生成任务给反馈；SWE-Together 把模拟用户反馈从短任务（HumanEval/MBPP 级）扩展到仓库级、有 reference patch 与触发条件锚定的 session [§4]。关系由论文 §4 第二组直接声明。
- extends CodeAssistBench (Kim et al., 2025) [high]：CAB 用 GitHub-issue 任务 + satisfaction-driven simulated user；SWE-Together 在 Table 3 中被定位为保留 CAB 的 real task source + interactive replay，但增加 repo-level 与 real user session provenance [§4, Table 3]。
- competes-with SWE-chat (Baumann et al., 2026) [med]：SWE-chat 同时是 SWE-Together 的上游数据源之一（48/109 任务来自它）和定位上的对照——SWE-chat 有 real user session 但缺 replayable repo state + verifier + correction loop [§2.1, §4, Table 3]。既是数据来源又是被超越的对照，故标 competes-with。
- orthogonal BigCodeArena / CodeChat (Zhuo et al., 2025a; Zhong et al., 2025) [med]：这两者收集 code-centric 对话做偏好建模/分析，是描述性/偏好导向的，不做 task-grounded replay [§4]；与 SWE-Together 的可验证正确性评估正交，但都从真实 user–agent 对话取材。
- extends RECODE-H / FronTalk (Miao et al., 2025; Wu et al., 2026b) [med]：两者在后端/研究代码/前端设置下评估 collaborative refinement；SWE-Together 同属"交互式精炼"家族但把场景从 curated feedback policy 推进到从真实会话重建的 correction loop [§4]。
- builds-on HumanLM (Wu et al., 2026a) [med]：simulator 的 session analysis 锚定与 state alignment 思路与 HumanLM "simulating users with state alignment beats response imitation" 一致 [§2.2 引用]；SWE-Together 的 anchored、state-conditional simulator 可视为该原则在 coding-agent 评估中的应用。论文显式引用此依赖。
- orthogonal Terminal-Bench (Merrill et al., 2026) [med]：Terminal-Bench 在交互式终端环境评估但 user request 固定 [§4, Table 3]；与 SWE-Together 共享 agent-env multi-turn 维度但正交于 interactive replay 维度。
