---
title: "Failure as a Process: An Anatomy of CLI Coding Agent Trajectories"
authors: ["Xiangxin Zhao","Han Li","Shuaiting Li","Tianyi Zhao","Earl T. Barr","Federica Sarro","He Ye"]
paper_id: "paper_arxiv_2607_09510"
source_kind: "arxiv"
source_id: "arxiv:2607.09510"
source_url: "https://arxiv.org/abs/2607.09510"
pdf_url: "https://arxiv.org/pdf/2607.09510"
read_id: "read_paper_arxiv_2607_09510"
kind: library-read
doc_type: "paper"
tags: []
---

# Failure as a Process: An Anatomy of CLI Coding Agent Trajectories

> 将 CLI coding agent 的失败从"最终结果标签"重构为"时间过程"，用三时间戳（决定性错误、锁定、可观测）标注 1,794 条轨迹，揭示失败早在第 7 步就已注定却常在第 16 步才暴露。

## Essence

**问题**：已有 coding agent 失败分析将失败视为静态终点——只分类"什么错了"，不追踪"错误何时开始、如何演变至不可恢复"。且大多聚焦 issue-resolution 或多 agent 场景，而非真实终端环境。

**做法**：对每条失败轨迹标注三个时间戳——`t_err`（决定性错误，即最终导致失败的步骤）、`t_lock`（经验性不可恢复点，此后不再观察到成功修复）、`t_obs`（首个外部可观测失败信号）。三者之间的间隔定义 fix window（可修复窗口）和 observability lag（隐藏延迟）。另用 prefix monitor（只看前 t 步、不知结局的独立模型）测试能否实时检测失败。

**证据**：决定性错误中位出现在第 7 步，而失败轨迹中位长度为 27 步——结果在前四分之一处就已注定，但可观测信号中位要到第 16 步才出现 [§III-A, Finding 1–3]。

**边界**：发现基于 Terminal-Bench 的 89 个任务、3 个 scaffold、7 个模型；所有时间戳标注为回溯性，`t_lock` 为"经验性不可恢复"而非理论不可恢复。

## Claims

1. CLI coding agent 的失败是一个时间过程而非瞬时事件：决定性错误中位发生在第 7 步，但失败锁定（`t_lock`）中位在第 12 步，首次可观测信号（`t_obs`）中位在第 16 步 [§III-A, Finding 1–3]。
2. 决定性错误后通常留有短暂修复窗口：中位 fix window 为 1 步，60.9% 的错误至少留有 1 步可修复时间，43.9% 留有 3 步以上 [§III-A, Finding 2]。
3. 28% 的失败轨迹从未产生任何外部可观测信号——失败完全沉默 [§III-A, Fig. 4]。
4. 失败的根因以 epistemic 错误（已有信息被忽略/误读/遗忘）为主，占 57.9%；competence 错误占 32.8%，environment 仅 9.4% [§III-B, Finding 5]。
5. False premise（基于未验证假设行动）是单一最大触发因素，占所有决定性错误的 30.7% [§III-B, Finding 6]。
6. Epistemic 错误的主导性在全部 21 个 model–scaffold 组合中均成立，范围 44%–80% [§III-D, Finding 14]。
7. 82% 的失败轨迹在锁定后仍继续执行而非立即终止；其中"修复错误诊断"行为占 24% 的轨迹却贡献 39% 的浪费执行量 [§III-C, Finding 7–8]。
8. 71% 的成功轨迹也经历过至少一次错误；成功与失败的区别不在于是否犯错，而在于能否恢复 [§III-C, Finding 9]。
9. 成功恢复中位耗时 5 步，失败恢复中位 12 步；成功轨迹对错误信号的响应率为 92%，失败轨迹仅 37% [§III-C, Finding 10–11]。
10. 26% 的失败轨迹出现伪造成功（fabricated success），且 84% 的伪造始于锁定点或之后——伪造是失败的后果而非原因 [§III-C, Finding 12]。
11. Prefix monitor 在识别已锁定的失败时精度达 82%，但实时召回率仅 18.2%（仅给任务名）至 28.8%（给任务名+需求）；中位提前量为零 [§III-A, Finding 4]。
12. 模型与 scaffold 均显著影响成功率（19%–45%），但 epistemic 错误的主导地位跨系统一致 [§III-D, Finding 13–14]。

## Assumptions

- Terminal-Bench 的 89 个任务（从 240 个中筛选，保留所有 21 个 model–scaffold 组合均有完整执行的）具有代表性——但筛选本身排除了 151 个任务，可能引入存活偏差。
- `t_lock`（经验性不可恢复）是对"不可恢复"的合理代理：论文承认这只是"未观察到恢复"，不等于理论上不可能恢复 [§II Terminology]。
- 人工标注的三时间戳可以准确回溯判定——尽管标注者阅读的是完整轨迹，存在后见之明偏差的可能。
- 71.4% 的轨迹来自公开数据集 [29]，28.6% 自行生成；假设两者的执行环境一致（通过 Harbor 框架统一协议）。
- Prefix monitor 使用 Claude Sonnet 4.6 作为独立检测器，假设其能力足以代表"在线检测"的可行性上界——但论文未测试其他模型作为 monitor。

## Method

**输入**：Terminal-Bench 上 3 个 scaffold（MiniSWE、OpenHands、Terminus2）× 7 个模型 = 21 个系统，原始 3,843 条轨迹，筛选后保留 89 个任务 × 21 系统 = 1,794 条有效轨迹（1,184 失败 + 610 成功），共 63,000+ 执行步骤。

**标注流程**：
1. Claude Opus 4.6 对每条轨迹生成结构化标注草稿（含支撑证据）。
2. 两位独立人工标注者审阅完整轨迹、验证证据、最终确定所有标签。
3. 分歧通过讨论解决；inter-annotator agreement: Cohen's κ = 0.78–0.94。

**标注内容**：
- **RQ1（时间）**：对每条失败轨迹标注 `t_err`、`t_lock`、`t_obs` 三个步骤号；衍生指标 fix window = `t_lock − t_err`，observability lag = `t_obs − t_lock`。
- **RQ2（根因）**：对 `t_err` 处的触发机制分类为 3 类 9 型（Epistemic 5 型 / Competence 2 型 / Environment 2 型）。
- **RQ3（恢复）**：对锁定后的尾部行为按主导活动 × 失败意识分为 5 种模式；对成功轨迹标注恢复 episode 的步骤数。
- **RQ4（跨系统）**：对比 RQ1–RQ3 结果在 21 个系统间的分布。

**Prefix monitor**：Claude Sonnet 4.6 只读前 t 步（不知结局），预测是否已过 `t_lock`；在 2,659 个前缀上评估，输入条件分两种（仅任务名 / 任务名+核心需求），区分 self-revealing 与 specification-relative 失败。

**输出**：14 条 findings + 公开标注数据集。

## Eval

**数据**：Terminal-Bench，89 任务，1,794 轨迹，63,000+ 步骤。

**系统**：3 scaffold × 7 模型 = 21 组合；pass rate 范围 19%–45%。

**标注质量**：Cohen's κ = 0.78–0.94（不同标签）；root cause κ = 0.83，三时间戳 weighted κ ≥ 0.94。

**Prefix monitor 评估**：2,659 个前缀（300 失败 + 300 成功轨迹，分层采样），指标为 precision、recall、lead time（相对 `t_lock`）。结果：precision 82%，recall 18.2%→28.8%（取决于是否给需求），中位 lead time = 0。

**无对比基线**：这是一项纯经验性研究，没有与任何现有失败检测/诊断方法做定量对比；Table I 的对比仅为规模和"是否将失败视为过程"的定性比较。

## Weaknesses

- **任务筛选引入存活偏差**：从 240 个任务中只保留 89 个"所有 21 系统都有完整执行"的任务，排除了 151 个任务和 1,197 条异常终止轨迹。失败率高或环境不稳定的任务被系统性排除，可能导致失败模式分布偏向"温和"场景。
- **Prefix monitor 仅用一个模型**：只用 Claude Sonnet 4.6 作为 monitor，未测试其他模型或更轻量的检测方法，无法判断"实时检测难"是任务本质限制还是 monitor 能力不足。
- **三时间戳标注的后见之明偏差**：标注者已知轨迹最终结果，`t_lock` 的判定依赖"此后未观察到恢复"——但同一错误在不同模型/scaffold 下可能有不同的恢复路径，`t_lock` 的绝对性被高估。
- **根因分类的互斥性假设**：论文声称三类根因"mutually exclusive"，但 false premise（epistemic）与 capability limitation（competence）在实际场景中可能交织——一个能力不足的 agent 更容易形成错误前提。
- **无纵向/时间维度验证**：所有模型快照来自 2025 年 5 月起；论文标题为"anatomy"但未讨论失败模式是否会随模型版本迭代而变化，限制了发现的时间泛化性。
- **Fabrication 检测缺乏独立验证**：Finding 12 关于 26% 伪造成功的统计依赖人工判定"证据是否伪造"，但论文未报告 fabrication 标签的单独 inter-annotator agreement。

## Relations

- **extends** Bouzenia & Pradel [10] `[high]`：该文对 120 条轨迹做 thought-action-result 三元组分析；本文扩展到 1,184 条失败轨迹并引入时间过程视角，Table I 明确对比。
- **extends** AgentLens [17] `[high]`：AgentLens 揭示"Lucky Passes"（通过脆弱试错通过），本文 Finding 9（71% 成功轨迹含可恢复错误）从反面验证并深化——成功不等于无错，关键在恢复能力。
- **builds-on** Terminal-Bench [19] `[high]`：本文数据全部来自 Terminal-Bench，且 §II-B 明确说明选择理由（纯结果评分 + 多 scaffold 支持）。
- **competes-with** TrajAudit [14] `[med]`：TrajAudit 做 93 条轨迹的自动失败诊断；本文做 1,184 条但全人工标注——两者在"自动化 vs 规模"上形成张力，但论文未直接对比诊断准确率。
- **orthogonal** MAST [15] `[high]`：MAST 聚焦多 agent 系统的 14 种失败模式分类，本文聚焦 CLI 单 agent 的时间过程；失败分类法互补但不重叠（Table I）。
- **extends** Mehtiyev & Assunção [18] `[med]`：该文在 9,374 条轨迹上发现 length-failure correlation 在控制任务难度后反转；本文的 fix window 和 observability lag 为这一现象提供了机制级解释——长轨迹不直接导致失败，而是因为失败锁定后仍长时间执行。

## Takeaway

- **方法论上值得借鉴**：三时间戳分解（`t_err` / `t_lock` / `t_obs`）是一个简洁且可迁移的框架，适用于任何需要理解"错误何时变为不可逆"的 agent 轨迹分析。fix window 和 observability lag 是两个可操作的衍生指标。
- **需要谨慎对待的数字**：所有百分比来自 89 个 Terminal-Bench 任务的筛选子集；任务类型分布（459 easy / 742 medium / 593 hard）偏向中等难度，高难度任务的失败模式可能不同。
- **最核心的洞察**：coding agent 的失败不是能力不足（competence 仅 32.8%），而是信息误用（epistemic 57.9%）——且 28% 的失败永远沉默。如果只能记住一件事：**大多数失败在前四分之一处就已注定，但你通常看不到任何信号。**
