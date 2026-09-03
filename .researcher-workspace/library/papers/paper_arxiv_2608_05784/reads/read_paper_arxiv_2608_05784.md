---
title: "Activity Frames: Deterministic Screen-Activity Compilation for Agent Memory and Replay"
authors: ["Nossa Iyamu"]
paper_id: "paper_arxiv_2608_05784"
source_kind: "arxiv"
source_id: "arxiv:2608.05784"
source_url: "https://arxiv.org/abs/2608.05784"
pdf_url: "https://arxiv.org/pdf/2608.05784"
read_id: "read_paper_arxiv_2608_05784"
kind: library-read
doc_type: "paper"
tags: []
---

# Activity Frames: Deterministic Screen-Activity Compilation for Agent Memory and Replay

> 屏幕活动记录要么是裸日志（太贵）、要么交给 LLM 摘要（不可复现）——本文用纯确定性代码把捕获流编译为结构化 activity frames，零模型、字节可复现，且同一段编译产物可直接做 replay。

## Essence

**问题：** 计算机使用型 agent 的记忆只记"用户说了什么"（对话），不记"用户做了什么"（屏幕活动）。已有的被动捕获（ActivityWatch、Recall、Chronicle）产出了原始数据，但消费端要么是 token 爆炸的裸行（单日 ~127k tokens），要么是 LLM 摘要（非确定、可能幻觉活动）。

**做法：** 确定性编译器读取本地 SQLite 捕获流，执行会话化分割（dwell credit、session gap、flicker merge）、输入事件富化（最近帧归属、坐标点击解析、键盘布局修正）、以及纯 URL 解析的实体类型化，输出两层文档：tier-1 纯测量层（application/site/时间/输入量/evidence pointer），tier-2 可选推断层（命名空间化、置信度标注、证据关联）。无模型参与编译路径。

**证据：** 单用户 51 活跃日、128,756 frames 语料上，单日编译耗时 68 ms，压缩为 prompt-ready context block 仅 1,469 tokens（86× 压缩）；agent 读该 block 答日间问题准确率 98.4%（Wilson 95% CI 91.7–99.7%），LLM 摘要仅 66–80%。

**边界：** 所有数据来自单一用户（作者本人）的一台机器；OCR 是编译器之前的捕获步骤且为学习组件，确定性从存储文本前向成立但不回溯到像素；dwell 衡量的是屏幕占用（tenure）而非注意力。

## Claims

1. 确定性编译（无模型）可将单日屏幕捕获流转换为字节可复现的结构化文档，两次独立编译产出字节级一致的输出（排除 generation timestamp）[§7.5]。
2. 编译路径的 per-day 成本不随历史增长：均值在后半语料为前半的 0.86×（216.4 ms vs 251.8 ms），即 O(|Δ|) [§7.5]。
3. 确定性编译产生的 compact context block（~1,469 tokens）使 mid-tier 模型（Sonnet 4.5）与 frontier 模型（Opus 4.5）在日间 QA 任务上表现不可区分（均 98.4%），而 raw-row 和 LLM-summary 两种基线下 frontier 模型领先 mid-tier 9–14 个百分点 [§6.2, Table 3]。
4. LLM 摘要在时间量级问题上的错误远大于结构化 block：Sonnet 层摘要将主导应用活跃分钟数平均误报 135.7%，而 activity frames 仅 7.3%（dwell 取整残余，非错误）[§6.2, Table 3]。
5. LLM 摘要每次重新生成产出不同文本（3 次生成 3 个不同结果），而 activity-frames block 跨 3 次生成字节一致 [§6.2]。
6. 在最忙日（257k raw tokens），raw-row 和 summary 两种基线超出模型上下文窗口无法使用；activity-frames block（~2k tokens）不受影响，是唯一能在所有 8 个评测日运行的表示 [§6.2]。
7. Routine Overhead Ratio R（agent 从截图重推导例程 vs. 注入编译例程的 token 比率）中位数为 60×（guarded plan, IQR 59–62×），信息内容上限为 343×（minimal script）[§7.2, Table 4]。
8. 桌面例程可委托重复率 h（delegable recurrence）在样本内为 9.0%、时间留出样本外为 7.7%；全舰队 token 上限约为 8% [§7.4]。
9. 在 guard-matched hit 上，参数化 replay 将模型完全移出循环，恢复约 99% 的单步重推导成本（1 − 1/R ≈ 99.7% at R=343）；但这是 per-covered-step 上限，非全舰队节省 [§7.3]。

## Assumptions

- 捕获引擎的事件驱动 + 心跳（~30s）模型能忠实记录屏幕占用；heartbeat row（19% 语料）被假定为有效注意力代理，但论文承认它无法区分阅读/观看与用户离开 [§4.1, §8.3]。
- 独立 SQL oracle（60s dwell cap, 300s session gap）是 QA 基准的合理 ground truth，且 oracle 与编译器在多显示器日上有意的度量差异（per-monitor vs interleaved）不构成评估偏倚 [§6.2]。
- R 的分子（C_agent）使用 Anthropic 的 wh/750 图片 token 规则 + 固定 350-token 上下文读取 + 180-token 推理写入来建模，假定为无跨步 prompt caching 的截图驱动 agent 上限 [§7.1]。
- 例程 n-gram 的 "≥2 named target" 限制是预先声明的去退化规则，不是事后调参 [§7.1]。
- 确定性编译的增量视图维护不声称方法论新意——论文明确承认 DBSP 和 event sourcing 是成熟领域，贡献在于将其作为 agent memory 的认证保证 [§7.5]。

## Method

**输入：** 本地 SQLite 捕获数据库（timestamp, application, window title, URL, input events, element tree），每显示器独立流。

**核心计算（三层）：**

| 层 | 功能 | 关键规则 |
|---|---|---|
| Segmentation | 将快照流分割为 (app, site) 键控的连续帧 | dwell = min(Δt, 90s)；session gap > 300s 断帧并报告为 coverage gap；flicker merge: A→B→A 且 B ≤ 20s 壁钟则合并为单帧，B 记录为 interruption |
| Enrichment | 修复输入事件可靠性 | 最近快照归属（binary search over stream）修复 stale attribution；坐标点击按 element tree exact→±40px tolerance→screen-zone 三级解析；键盘布局映射表（operator-supplied, identity by default） |
| Entity Typing | URL → typed page reference | 层级解析：~20 bespoke site parser → generic search-param detector → subdomain/path heuristic → generic fallback（total mapping, never loses data） |

**输出：** 两层文档——tier-1 纯测量层（coverage + frames + blind_spots + provenance），tier-2 可选推断层（namespaced + confidence-tagged + evidence-linked，可剥离后保留有效 tier-1）。编译器零运行时依赖，只读打开数据库，支持 JSON/YAML/Markdown/plaintext context block 输出。MCP server 暴露 6 个工具（get_context, get_activity, get_day_summary, get_patterns, get_communications, get_steps）。

**Replay 执行器：** 编译器从重复 n-gram 生成 guarded skill plan（每步携带 expected-element/role/application guard），parametric replay 在 guard-matched 步上将模型完全移出循环，变量槽填充请求新值。

## Eval

**Token 成本：** 单日（2,066 snapshot rows）三种表示对比——raw rows 126,812 tokens；frames JSON 34,815 tokens（3.6× 压缩）；compact context block 1,469 tokens（86× 压缩）。编码 cl100k_base [§6.1, Fig 4]。

**下游 QA：** 8 天（7 个最近连续活跃日 + 1 个 raw 序列化超上下文窗口日），64 题（5 类 + 16 absent-fact probes），独立 SQL oracle 评分。两模型层（Sonnet 4.5, Opus 4.5），三种表示（raw rows, LLM summary, activity frames），数值容差 30%、时间容差 45 分钟。基线在最忙日超上下文窗口仅报告 7/8 天 [§6.2, Table 3]。

**延迟与复现：** 单日编译中位 68 ms（range 65–72, 5 runs, Apple Silicon）；认证语料上 51 活跃日中位 220.9 ms（range 0.1–930 ms）；字节级一致（排除 generated_at）；rebuild = incremental [§6.3, §7.5]。

**实体类型覆盖：** 5,120 distinct URLs，81.3% 获得非泛型 typed reference（46 kinds），18.7% 回退到 generic [§6.4]。

**R 与 h：** R 基于 20 个最频繁 action routine（guarded plan 中位 247.5 tokens, 0.5 ms 编译, 0 tokens）和 minimal script（中位 40 tokens）；h 基于 40 天训练 / 11 天留出时间分割 [§7.2, §7.4, Table 4]。

**Replay 验证：** 1 次 owner-authorized live execution（2 步 compose routine, zero model tokens at execution）；1 次 in-loop 对比（accessibility-tree agent, saved ~14%）[§8.4]。

## Weaknesses

1. **QA 基准的结构性偏倚未充分处理：** activity-frames agent 直接获得 per-application duration ledger（编译器产物），而 raw-row 和 summary 基线必须自行推导——论文承认这一点（"that is exactly what the compiler is for"），但未提供一种让基线也获得等价结构化信息的公平对比，因此 98.4% vs 66–80% 的差距部分度量的是"算术是否已做好"而非纯粹的信息保真度 [§6.2]。

2. **R 的分子是建模上限而非实测值，且唯一的 live in-loop 对比（accessibility-tree agent）仅节省 ~14%，与建模的 83.3% 差距巨大：** 论文诚实地标注了这一点，但 Table 4 和 Table 3 中 R 的数值（60×–343×）在正文叙述中的视觉权重远大于 §8.4 中的 14% 实测值，读者容易高估实际节省 [§7.3, §8.4]。

3. **h 的 "≥2 named target" 限制可能系统性低估真实可委托重复：** 该规则排除了单目标但重复的合法工作流（如在同一应用内重复执行的多步操作），且 9.0% 的 h 严重依赖于这一定义选择——论文声明其为预先声明而非调参，但未提供敏感性分析（如 ≥1 target 的 h 值对比）[§7.1, §7.4]。

4. **OCR 是确定性主张的隐性破裂点：** 捕获阶段 on-device OCR 覆盖 96% frames，是管线中唯一的学习组件。论文将确定性范围限定为"从存储文本前向"，但未量化 OCR 错误对 entity typing（82% 非泛型覆盖）和后续 routine match precision（q < 1）的传播影响 [§8.3, §8.4]。

5. **replay live 验证极度有限：** 仅 1 个 2 步 routine，且其 plan 是从 live accessibility names 种子化而非从 mined routine 生成；click-level grounding 仍在验证中；guard-miss deopt 路径未被执行 [§8.4]。在此基础上，Table 4 的 Rinfo 和三臂对比的 dollar 数字作为"modeled ceiling"的定性不够突出——它们在表格中与实测分母并列展示。

6. **单用户语料的外部效度问题不仅限于 R/h：** entity typing 的 81.3% 覆盖率和 46 kinds 反映的是该用户（独立研究者/开发者）的应用画像；不同职业（设计师、财务、客服）的 URL 分布和 site parser 覆盖率可能差异巨大，但论文未提供 per-site 覆盖率的分母（多少 distinct URLs per site）以评估长尾风险 [§6.4]。

## Relations

- **builds-on** episodic memory position [§2.1, ref 1] `[med]`: 论文将 activity frames 定位为 episodic-memory 倡导者所称"缺失的获取层"的首次确定性桌面实例化，显式引用该 position paper 的论点。
- **contradicts** MIRIX / FOCAL / ProAgentBench / SummAct [§2.2, refs 16–19] `[high]`: 这些系统均在 memory-construction loop 中使用模型（VLM 或 LLM），本文明确反驳该范式，主张无模型编译路径以避免 per-run cost、non-determinism 和 hallucinated episodes——论文显式列出对比。
- **orthogonal** skill induction (Agent Workflow Memory / SkillWeaver / PreAct) [§2.3, refs 15, 38, 40] `[med]`: 技能诱导从 agent rollouts 学习（supply side, post-delegation），activity frames 从被动捕获的人类活动读取（demand side, pre-delegation）——两者观测例程的时机和来源不同，论文称前者"只在 agent 已以全价执行后才观测到例程"。
- **extends** lifelogging threshold segmentation + web sessionization [§2.3, refs 25–27] `[high]`: 论文明确将分割方法论溯源至 lifelogging 的 threshold segmentation 和 web sessionization 的 inactivity timeout，但终点从人类可读的过程模型/散文变为 agent 可消费的 typed memory。
- **orthogonal** agent cost frameworks (FrugalGPT / Cost-of-Pass / AI Agents That Matter / HAL) [§2.3, refs 46–49] `[med]`: 这些框架为 agent 任务定价（supply side），activity frames 提供 R 和 h 两个需求侧参数——论文采用 Eq. 1 作为 cited frame 而非贡献，声称这些框架"假设但不测量"h。

## Takeaway

- **方法层面值得借鉴：** 两层 schema（measured/inferred 边界 + evidence pointer）是对抗 agent memory poisoning 和 hallucination 的结构性方案，且其确定性使 CI 级别的 byte-equality 认证成为可能——这比任何模型层 guardrail 更可审计。
- **需要 distrust 的点：** R 的 60×–343× 数字是建模上限，唯一 live in-loop 对比仅 14% 节省；h 的 9.0% 严重依赖 "≥2 named target" 定义；QA 基准中 activity-frames agent 获得了基线没有的结构化算术产物。引用这些数字时应标注 "modeled ceiling"。
- **如果只记一件事：** 确定性编译用 68 ms 和 0 tokens 把一天的屏幕活动变成 agent 可读的结构化记忆，且字节可复现——"interpretation is not banned but quarantined" 是其信任模型的核心 [§1]。
