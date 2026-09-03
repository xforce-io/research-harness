---
title: "Benchmarking Coding Agents on Databricks’ Multi-Million Line Codebase | Databricks Blog"
authors: []
paper_id: "paper_url_6fd5a7e05202537d"
source_kind: "url"
source_id: "url:https://www.databricks.com/blog/benchmarking-coding-agents-databricks-multi-million-line-codebase"
source_url: "https://www.databricks.com/blog/benchmarking-coding-agents-databricks-multi-million-line-codebase"
pdf_url: ""
read_id: "read_paper_url_6fd5a7e05202537d"
kind: library-read
doc_type: "blog"
tags: []
---

# Benchmarking Coding Agents on Databricks' Multi-Million Line Codebase

> 在真实多语言生产代码库上自建 PR 级编码 agent 基准，发现 Pareto 前沿由多家厂商混合构成、模型单 token 价格不反映真实任务成本、harness 对效率影响巨大。

## Brief

Databricks 工程团队基于自身合并的 PR 构建了一套内部编码 agent 基准，覆盖 Python、Go、TypeScript、Scala、Rust 等十余种语言的生产代码。该文档阐述了基准构建方法、主要发现及对模型/harness 选择的决策影响。其核心价值在于：证明任何拥有合并 PR 积压的团队都能自建私有、无数据泄露的编码 agent 评测体系，并据此做出数据驱动的工具选择。

## Key takeaways

- 编码任务的 Pareto 前沿由 OpenAI、Anthropic 和开源模型（GLM 5.2）共同构成，没有任何单一来源能全面占优。
- GLM 5.2 与 Opus 4.8 在质量上统计持平，但单任务成本 $1.28 vs $1.94 [blog: "Open models are here for coding"]。
- 模型单 token 价格与端到端任务成本相关性极差：Sonnet 5 每 token 比 Opus 4.8 便宜约 1.7×，但因消耗 1.9× 更多 token，单任务成本反而更高（$2.09 vs $1.94），质量还低 6 分。
- Harness 对成本影响可达 2× 以上：Pi harness 每轮发送约 3× 更少上下文，任务质量相同但成本大幅降低。
- 模型按能力聚类为三个层级，大量日常运维任务（翻 flag、改配置）无需顶级模型，应下沉至 Haiku / GPT 5.4 Mini 级别。

## Decisions / claims

- **不使用 LLM judge 评估正确性**，仅用测试通过/失败判定，因为"LLM judge 奖励听起来对而非真正对" [blog: "How we built the benchmark"]。
- **密封 git 历史**：每个任务运行期间切断工作副本与仓库的 git 连接，防止 agent 从 git history 中直接找回正确实现 [blog: "Additional Guardrails"]。
- **任务描述中删除解决方案**：从 PR 中提取意图时，重写描述仅陈述问题与约束，移除"为什么这个修复是对的"等解释，避免任务过于简单。
- **手工逐条审核候选任务**：尽管用脚本和 AI 生成候选任务，每个样本均经人工评估，必要时手动重写测试以允许多种实现或提高严格度。
- **将更多工作下沉至中小模型**：基于能力分层分析，团队决定将日常低复杂度任务（约占 25%）从默认顶级模型迁移至 Haiku / GPT 5.4 Mini 级别。

## Constraints & assumptions

- 基准任务来源于 Databricks 自身合并的 PR，代表该团队的代码栈和技术栈分布，不代表其他公司的场景。
- 任务筛选条件：近期合并、人工编写（过滤 bot/auto-generated）、含高质量测试、改动局限于少数模块、覆盖全栈技术。
- 约 25% 任务为低复杂度，约 60% 为中等复杂度，高复杂度任务占比较小。
- Agent harness 和模型均使用开箱即用的标准配置，配备 Databricks 工程师常用的全部工具。
- 评估时仅在 agent 明确声明完成时 checkpoint 代码，然后 patch 保留的测试并运行判定。

## Open questions

- 文档未公开具体任务数量、各模型/harness 组合的完整得分表，无法独立验证结论的可推广性。
- 高复杂度任务样本较少，结论"open models can handle even the highest level of task difficulty"是否稳健尚不明确。
- Pi harness 为何能以 3× 更少上下文维持同等质量——是上下文管理策略还是工具调用差异——文档未深入分析。
- 基准是否计划公开发布（不含敏感代码的子集）以支持外部复现，文档未提及。
- 模型版本（如 Opus 4.8、Sonnet 5、GLM 5.2、GPT 5.4 Mini）为内部代号或未来版本，外部无法直接对应验证。

## Relations

- competes-with SWE-Bench / TerminalBench [high]: 文档明确将自身定位为对公开基准的替代方案，理由包括公开基准的任务泄露进训练数据、且不代表性覆盖 10+ 语言的生产代码栈 [blog: "Why build your own benchmark?"]。
- extends coding agent evaluation methodology [med]: 通过 PR 反向构建任务（提取意图、移除解决方案、保留测试、密封 git 历史）的流程，为"从私有 PR 构建内部基准"提供了可复用的方法论模板，任何有 PR 积压的团队均可套用。
- orthogonal to LLM-as-Judge evaluation [high]: 文档明确拒绝使用 LLM judge 评估正确性，仅依赖测试通过率，与当前 LLM-as-Judge 评测趋势形成直接对比——但其前提是每个任务都有高质量测试，这在许多场景下不可满足 [blog: "How we built the benchmark"]。
