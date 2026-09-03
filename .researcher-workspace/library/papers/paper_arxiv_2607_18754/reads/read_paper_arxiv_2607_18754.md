---
title: "AgentDebugX: An Open-Source Toolkit for Failure Observability, Attribution, and Recovery in LLM Agents"
authors: ["Kunlun Zhu","Xuyan Ye","Zhiguang Han","Yuchen Zhao","Bingxuan Li","Weijia Zhang","Muxin Tian","Xiangru Tang","Pan Lu","James Zou","Jiaxuan You","Heng Ji"]
paper_id: "paper_arxiv_2607_18754"
source_kind: "arxiv"
source_id: "arxiv:2607.18754"
source_url: "https://arxiv.org/abs/2607.18754"
pdf_url: "https://arxiv.org/pdf/2607.18754"
read_id: "read_paper_arxiv_2607_18754"
kind: library-read
doc_type: "paper"
tags: []
---

# AgentDebugX: An Open-Source Toolkit for Failure Observability, Attribution, and Recovery in LLM Agents

> 失败步骤≠根因步骤——AgentDebugX 用多轮诊断代理把"回放 trace"升级为"定位根因→生成修复→重跑验证"的闭环。

## Essence

**问题**：现有可观测性工具（Langfuse 等）能回放 trace，但不定位根因也不修复；归因 benchmark 只评局部能力不提供可部署基础设施；self-correction 在不知道错误位置时效果差。三者之间缺少一个闭环。

**做法**：AgentDebugX 将调试组织为 Detect→Attribute→Recover→Rerun 四阶段闭环。核心是 DeepDebug 多轮诊断代理：先全局通读 trace 标初始候选步，再用结构化探查（多代理→沿 handoff 级联回溯；单代理→二分 bisect）得独立第二候选，两候选冲突时做 cross-examination 裁决，最终输出含证据和修复建议的结构化报告。所有诊断叠加在 trace 之上，不写回原始记录。

**证据**：Who&When 上 strict agent-and-step 准确率 28.8% vs 最强单遍基线 21.7%（qwen3.5-9b）；GAIA 上单次 rerun 修复 13/73 失败任务 vs decoupled self-correction 的 4–6 个。

**边界**：归因增益是 model-dependent 的——在 hosted backbone 上单遍全局阅读已足够、adjudication 无额外收益；Error Hub 检索和 taxonomy 归纳已实现但未评测。

## Claims

- 失败可见步骤与根因步骤的错位是 agent 调试的核心困难，回放 trace 本身不足以定位根因 [§1]。
- DeepDebug 的两轮读+裁决设计将根因搜索从"遍历整条 trace"缩减为"两个候选假设之间的聚焦裁决" [§3.3]。
- 结构化探查轮（cascade/bisect）与全局通读轮互补：全局读保留任务上下文但锚定最显眼的下游症状，结构化读丢失全局但能定位被遮蔽的上游根因 [§3.3]。
- 将结构化探查轮替换为第二次全局搜索，在 gpt-5.4-mini 上 strict 准确率下降 4.8 个百分点（0.310→0.262），证明结构化探查不是冗余而是增益来源 [§C, ablation]。
- 归因增益集中在 >40 事件的 trace 上——短 trace 各方法差异不大，长 trace 才是 multi-turn 诊断的目标区间 [§4, Figure 3]。
- DeepDebug 的多轮调用成本仅为单遍阅读的 1.6×（12.8K vs 8.1K tokens），因为后续轮只读聚焦窗口 [§4]。
- 在 qwen3.5-9b 上，DeepDebug 在 Who&When 全部五项指标上超过所有被测单策略定位器 [§4, Table 2]。
- 在 GAIA 上，DeepDebug 原生修复路径单次 rerun 修复 13/73 任务，将整体准确率从 55.8% 提升至 63.6%，是三个 decoupled 基线（4–6 个）的 2–3 倍 [§4, Table 3]。

## Assumptions

- 被调试 agent 的执行可被转换为框架无关的 AgentTrajectory 表示（通过 adapter 或离线导入），诊断质量不依赖于原始框架格式 [§3.1]。
- Who&When 的 reference-answer protocol（提供参考答案但不给 gold label）能反映真实调试场景中开发者可能拥有的信息水平 [§C]。
- 诊断代理（gemini-2.5-flash, temperature 0, thinking disabled）的判断不受 thinking 能力关闭的显著影响 [§4 Setup]。
- GAIA recovery 实验中，DeepDebug 路径与 decoupled 基线的唯一关键差异是"是否获得 DeepDebug 的局部化诊断"——但论文承认实验评测的是完整 recipe 而非隔离归因效果 [§4]。
- Error Hub 的 scrubber 默认剥离 event inputs 并做已知 pattern 的凭证/PII 脱敏，但论文承认无法保证移除任意敏感内容 [§3.5]。

## Method

**输入**：live execution（通过 LangGraph/CrewAI/OpenAI Agents SDK/OpenTelemetry/raw ReAct adapter）或导出日志（离线 importer），统一转为 AgentTrajectory——有序 AgentEvent 序列，每个 event 记录 agent、module、step index、parent、inputs/outputs、error、artifact。

**闭环四阶段**：

| 阶段 | 机制 | 输出 |
|------|------|------|
| Detect | 确定性 rule pack（无模型调用）→ 不够时 LLM judge 读 bounded window | typed findings（event, failure mode, evidence, confidence），19 种 seed failure mode |
| Attribute | 成本递增策略族：heuristic→single-pass→binary search→per-step→budgeted ensemble；ambiguous 案例升级 DeepDebug | ranked hypotheses with confidence & provenance |
| Recover | DeepDebug 诊断直接作为 retry directive（无需额外模型调用）；Reflexion/CRITIC/AutoManual 作为替代策略 | suggest-only 修复建议，human/policy gate |
| Rerun | 诊断+checkpoint+retry directive 打包为 rerun request；executor 生成新 trajectory | 新 branch 与原始并列保留；成功→resolved case，失败→重新进入 Detect |

**DeepDebug 四阶段**（核心差异化）：

1. **全局通读**：读整条 trace，重建目标与历史，标初始候选步——区分"因果错误"与"局部异常但有效的动作"。
2. **结构化探查**：多代理→沿 handoff 级联从可见失败向上游回溯至最早致命步；单代理→二分 step range 重读存活区间——产生独立第二候选。
3. **交叉审查**：两候选一致→接受；冲突→并排检查上下文/输入/输出/下游影响，选更强因果解释。
4. **诊断输出**：structured report（responsible agent+step, plain-language explanation, quoted evidence, one concrete fix），每次检查均有审计记录。

**可扩展 taxonomy**：judge 遇到 seed 外的 recurrent failure → 记录 novel-mode candidate → inducer 聚类（label/lexical/embedding similarity, support threshold gating）→ 提案新 mode → 维护者审批，提案不覆写 curated taxonomy。

## Eval

**归因评测**：
- 数据：Who&When 全量 184 traces（126 算法生成 + 58 人工构造），每条标注 gold responsible agent + mistake step。
- 基线：Rule heuristic, All-at-Once（单遍全局）, Step-by-Step, Binary-Search。
- 指标：responsible agent accuracy, exact step, ±1 step, strict agent-and-step (exact), strict agent-and-step (±1)。
- 结果（qwen3.5-9b）：DeepDebug strict A+S exact 28.8% vs 最强基线 21.7%；agent accuracy 56.0 vs 47.8。qwen3.6-27b 上 strict 38.0 vs 36.4。
- 模型依赖性：在 hosted backbone（gpt-5.4-mini, gemini-3.5-flash）上单遍全局阅读已最强，adjudication 无增益——论文据此建议 per-model routing 而非无条件调用 multi-call。

**端到端修复评测**：
- 数据：GAIA validation（165 tasks, 3 难度级别），vanilla qwen3.5-9b Open-DeepResearch agent 首跑失败 73 tasks。
- 基线：Reflexion, CRITIC, AutoManual（decoupled——看 generic judge summary，不看 DeepDebug 局部化诊断）。
- 指标：repaired cases / 73；各难度级别及整体 accuracy（official scorer）。
- 结果：DeepDebug 13/73（+7.8pp overall），CRITIC 4/73, AutoManual 5/73, Reflexion 6/73。最大增益在 Level-2 multi-hop（48.8→61.6）。

**消融**：stratified 42-trace sample 上，结构化探查轮→第二次全局搜索，gpt-5.4-mini strict 从 0.310 降至 0.262。

## Weaknesses

- GAIA recovery 实验比较的是"DeepDebug 完整 recipe" vs "decoupled 基线的通用 summary"——基线未获得局部化信息，这不是归因效果的隔离测试而是完整 pipeline 的非对称比较；论文承认这一点但未设计控制实验隔离 attribution 单独贡献 [§4]。
- Who&When 上 >40 事件的 trace 仅 26 条（14%），DeepDebug 的核心增益证据来自这个很小的子集，论文自己标注为 "descriptive evidence"——但 Claims 和 Abstract 未反映这一不确定性 [§4, Figure 3]。
- hosted backbone 上 adjudication 无增益甚至有害（ablation 显示 gpt-5.4-mini 上替换后降 4.8pp），意味着 DeepDebug 的价值可能仅限于较弱的 open-weight 模型——论文建议 per-model routing 但未提供路由决策的具体规则或阈值 [§C]。
- Error Hub 检索增强和 taxonomy 归纳是论文叙事的重要组成部分（"debugging memory"、"cross-team corpus"），但两者均"implemented but not yet evaluated"——这些功能的效果是未验证的假设而非已证实的能力 [§E]。
- 诊断 LLM（gemini-2.5-flash）与被调试 policy（qwen3.5-9b）来自不同模型家族，诊断能力可能主要来自 gemini 的更强推理而非方法本身的优势——未测试同模型自诊断场景。
- scrubber 默认剥离 event inputs 并做已知 pattern 脱敏，但论文承认"pattern redaction cannot guarantee removal of arbitrary sensitive content"——Error Hub 的共享安全性依赖于人工审查，这在规模化场景下是否可行未讨论 [§3.5]。
- GAIA 只测试了一个 policy model（qwen3.5-9b）在一个 benchmark 上的固定 73-task 失败子集——73 个任务中修复 13 个（17.8%），样本量有限，未报告 confidence interval 或多次 seed 的方差。

## Relations

- builds-on Who&When benchmark (Zhang et al., 2025) [high]：AgentDebugX 直接使用 Who&When 作为归因评测基准，DeepDebug 与该 benchmark 的单策略基线直接比较。
- extends AgentDebug (Zhu et al., 2025) [high]：论文在 Table 1 中将 AgentDebug 列为部分覆盖 Attribution 且缺失 Recovery 和 Error Hub 的前序工作；AgentDebugX 显式扩展为闭环 pipeline。
- competes-with AgentDiagnose (Ou et al., 2025) [high]：两者均为开源 agent 诊断工具包，但 AgentDiagnose 专注于轨迹评分和训练数据策展，不连接 step-level attribution 到 verified recovery——Table 1 直接对比。
- competes-with MAST (Cemri et al., 2026) [med]：MAST 提供 multi-agent failure taxonomy，AgentDebugX 在同一维度上提供可部署的 attribution + recovery，覆盖范围更广但 MAST 的 taxonomy 分析深度未做直接比较。
- extends Reflexion (Shinn et al., 2023) / CRITIC (Gou et al., 2024) / AutoManual (Chen et al., 2024) [high]：三者作为 AgentDebugX 的 alternative recovery 策略和基线被集成和比较；AgentDebugX 的核心论点是 self-correction 在已知错误位置时更可靠——这些方法被定位为缺少 attribution 的 decoupled recovery。
- orthogonal-to Langfuse (Langfuse, 2023) [med]：Langfuse 提供执行追踪和可观测性但不做根因分析或修复；AgentDebugX 的 portable trace 格式可与这类平台互补而非替代——Table 1 将 Langfuse 标为仅 partial 在 portability schema。

## Takeaway

- **值得借鉴的方法论**：两轮读+裁决的归因设计——全局读保上下文、结构化读保精度、冲突时聚焦裁决——是一个可迁移的"搜索→缩减→裁决"模式，不限于 agent debugging。
- **需要存疑的数字**：核心归因增益的证据来自 26 条 >40-event trace（"descriptive evidence"），GAIA 修复率来自 73-task 单 policy 单 benchmark 子集——两者样本量均有限。
- **适用边界**：DeepDebug 的多轮增益是 model-dependent 的，在强 hosted 模型上可能无效；Error Hub 的检索增强和 taxonomy 归纳是叙事组件而非已验证能力。
- **一句话记忆**：不是"更好的 trace 回放"，而是"把 trace 上的根因搜索从遍历变成两个候选之间的裁决"——前提是 trace 已被转换为统一格式且你愿意为长 trace 付 1.6× 的 token。
