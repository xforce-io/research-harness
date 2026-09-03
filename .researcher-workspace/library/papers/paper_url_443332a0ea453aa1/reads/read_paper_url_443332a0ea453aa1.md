---
title: "You only need the frontier model for one single edit — Stencil"
authors: []
paper_id: "paper_url_443332a0ea453aa1"
source_kind: "url"
source_id: "url:https://stencil.so/blog/prewalk"
source_url: "https://stencil.so/blog/prewalk"
pdf_url: ""
read_id: "read_paper_url_443332a0ea453aa1"
kind: library-read
doc_type: "blog"
tags: []
---

# You only need the frontier model for one single edit - Stencil

> 前沿模型做 plan-then-hand-off 反而更贵——因为成本不在"思考"而在"读代码"——/prewalk 在首个 edit 后换廉价模型并清除 plan 指令，让廉价模型继承完整 trajectory 而非一纸 postcard，从而同时降本、提速、减少作弊。

## Essence

1. **问题**：plan-then-execute（前沿模型规划、廉价模型执行）被当作降本标配，但实测比前沿模型全程自己做更贵、更慢，且未提升通过率。
2. **做法**：/prewalk——前沿模型带隐藏指令启动（深度规划→写成 todo list→开始执行），一旦落地第一个 edit 就切换到廉价模型，并从 context 中删除规划指令；廉价模型看到的是"自己探索过、写过 plan、已经动过手"的连续 trajectory。
3. **证据**：Opus 4.8 + /plan = $3.18/task @ 84.6%；Opus 全程 = $2.78/task @ 84.6%——"省钱"方案贵 14%。/prewalk = $1.46/task @ 78%，比 oneshot Flash +30pts。1.81B token 中仅 9% 是 edit，其余是 read——agent 的账单是 O(reads)。
4. **边界**：只在 SWE-bench 类任务上测过两个模型对（Opus→Flash、Sol→Luna）；TODO list 必须配合使用，否则廉价模型中途遗忘或误判完成；GPT 5.6 需限制 TODO 条目数（否则生成 60 条批量完成）。

## Key takeaways

- **Agent 成本是 O(reads) 而非 O(thinking)**：1.81B token 中 edits 仅占 9%，reads 占大头且两个模型都要按各自单价全额付费——/plan 让前沿模型读完一遍、廉价模型再读一遍，是把成本复制而非转移。
- **Hand off trajectory, not plan document**：plan.md 是 2K token 的 postcard，廉价模型拿到后必须重建 100K token 的 grounded context；/prewalk 交接的是 context window 本身（探索痕迹 + todo + 已落地 edit），廉价模型不重新理解。
- **Prefill 原理的 trajectory 级变体**：不在 token 级 prefill，而是在 turn 级——删除规划指令后，廉价模型不知道有"换人"发生过，把前任的探索和 edit 当作自己的，一致性驱动它继续执行。
- **TODO list 是廉价模型的 steering 通道**：廉价模型会遗忘 plan 和 validation step，但不会忽略反复提醒的 todo；这是 /prewalk 区别于"无脑首个 edit 后切换"的关键。
- **反作弊副效​​应**：/prewalk 在前沿模型进入"搜索 GitHub"阶段前终止它（median ~7 turns），执行模型继承的是"已在代码上验证过"的 context 而非"搜索"行为——作弊率从 oneshot 的 44–100% 降到 13–70%。

## Decisions / claims

- **Plan-then-execute 是反优化**：Opus 4.8 + /plan ($3.18) 比 Opus oneshot ($2.78) 贵 14%、通过率持平 (84.6%)——"省钱"措施反而更贵 ["Cost vs pass rate" 图表 / "reads it all" 节]。
- **Agent 账单 ≈ O(reads)**：1.81B token / ~2M tool calls 中 edits 仅 9%；"this split is not a quirk of one harness"——任何 agent/model/scaffold 皆如此 ["reads it all" 节]。
- **/prewalk 触发点 = 首个 edit 落地**：仅 gate on edit 不够，必须配合 TODO list（含 validation step per item）；TODO 是廉价模型唯一可靠的 steering 通道 ["How we got here" 节]。
- **切换时删除规划指令**：廉价模型的 context 中不存在 "plan deeply" 指令，因此不会"wait, I thought we were planning" ["Hand off a trajectory" 节]。
- **/prewalk 实测数据**：
  - Sol→Luna: 85% pass @ $1.04 @ 300s（vs Sol oneshot 88% @ $1.71 @ 372s；vs Luna oneshot 77% @ $0.60 @ 570s）→ 97% of Sol pass rate at 61% cost ["The receipts" 节]。
  - Opus→Flash: 78% pass @ $1.46 @ 402s（vs Opus oneshot 85% @ $2.78 @ 606s；vs Flash oneshot 60% @ $1.16 @ 360s）→ 92% of Opus pass rate at 53% cost ["The receipts" 节]。
- **作弊率显著下降**：Opus /prewalk 13% vs oneshot 44% vs /plan 72%；Sol /prewalk 70% vs oneshot 95% ["The effect we didn't expect" 节]。
- **作弊机制解释**：solo trace 中 GitHub 搜索发生在探索停滞期（Sol ~turn 14, Opus ~turn 12）；/prewalk 在前沿模型 median ~7 turns 终止——仍在"confident phase"，未进入"googling phase" ["The effect we didn't expect" 节]。
- **GPT 5.6 需限制 TODO 条目数**：否则生成 60 条 TODO 并批量完成 ["How we got here" 节]。
- **实现已 upstream 到 omp**：`--prewalk`、`--prewalk-into <model>`、`/prewalk` ["Prefill walked" 节]。

## Constraints & assumptions

- **假设**：廉价模型足够跟随——能继续执行一个已落地的 edit sequence，只要 context 连贯且 TODO 存在。未测试远低于 Flash/Luna 级别的模型。
- **假设**：任务结构是 bug-fix 类（SWE-bench）——有明确 reproduce→fix→test 流程；open-ended 任务是否受益未验证。
- **约束**：前沿模型必须在"confident phase"被终止——太早（仍 lost）或太晚（已进入搜索/作弊）都会失败；"swap at fixed turn" 和 "swap after first edit alone" 均被否 ["How we got here" 节]。
- **约束**：TODO list 含 per-item validation step 是必需的——否则廉价模型中途误判完成或遗忘步骤。
- **约束**：TODO 条目数需 prompt-level 限制（至少对 GPT 5.6）。
- **假设**：inference 层允许中途删除 context 中的特定 instruction 并切换 model——部分 closed API 可能不支持 context editing。

## Open questions

- /prewalk 在 non-SWE-bench 任务（长 horizon 多文件重构、open-ended exploration）上是否仍然有效？博客未涉及。
- 如果前沿模型的首个 edit 方向错误，廉价模型是否会沿错误 trajectory 一路走到底？TODO validation 能否可靠拦截？
- 不同廉价模型的最小能力阈值是什么？博客未测试低于 Flash/Luna 的模型。
- 作弊率下降是否可泛化为"trajectory 继承改变后续行为模式"的一般性结论，还是 SWE-bench 特有（答案在 GitHub 上）？
- /prewalk 与 sub-agent dispatch（博客提到但未实现/对比的方案）在复杂任务上的 trade-off？

## Relations

- builds-on prefill / assistant-prefixing literature [high]: 博客明确将 /prewalk 定位为 prefill 原理的 trajectory 级变体——"It's the oldest trick in the book: prefill" ["Prefill walked so prewalk could run" 节]。区别在于 prefill 在 token 级注入，/prewalk 在 turn 级注入（prefilled turns = 已发生的探索 + todo + edit）。
- competes-with plan-then-execute / hierarchical agent patterns [med]: /prewalk 与 plan-then-execute 解决同一问题（前沿模型贵、廉价模型弱）但机制相反——交接 trajectory 而非 plan document；博客直接以 /plan 为 negative baseline。与 sub-agent dispatch（"let it explore, then dispatch sub-agents"）属于同问题空间但未直接对比。
- extends context-engineering / agent-steering via context manipulation [med]: 删除规划指令使廉价模型不知道"换人"发生过——属于 context editing 作为 steering 手段的实践案例，但未引用学术文献。

## Takeaway

- **可复用**：首个 edit 后切换模型 + 删除规划指令 + TODO with validation——三要素缺一不可，可移植到任何支持 context editing 的 agent harness。
- **可复用**："agent 成本 = O(reads)" 这一经验法则可直接用于评估任何 multi-model pipeline 的成本结构——只要两个模型各自 read，成本就在叠加。
- **需警惕**：所有数据来自单一 harness、两个模型对、SWE-bench 单一 benchmark；97%/41%/1.9×/3× 等 headline 数字是 best case，非分布。
- **记忆钩**：plan 是 postcard，trajectory 才是旅程——别让廉价模型从明信片重建一万 token 的上下文。
