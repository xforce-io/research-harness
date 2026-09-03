---
title: "Harness-Delta Attribution (HDA)"
authors: []
paper_id: "paper_url_cbebf719668f137d"
source_kind: "url"
source_id: "url:https://github.com/Wenwen-D/HarnessDeltaAttribution"
source_url: "https://github.com/Wenwen-D/HarnessDeltaAttribution"
pdf_url: ""
read_id: "read_paper_url_cbebf719668f137d"
kind: library-read
doc_type: "other"
tags: []
---

# Harness-Delta Attribution (HDA)

> 自动化 harness 演化能把基准分数抬高，但一个总分说不清改进来自捷径、算力还是真本事;HDA 通过构造两个受控变体(compute-matched 的 `B_cc` 与去过拟合的 `E_neutral`)把每次增益拆成 O/T/G 三项可加账目，使“harness 变强了”这类主张第一次可以被审计。

## Essence

**问题**：自动 harness 演化——proposer LLM 围绕冻结的 executor 反复编辑 prompts、tools、memory 与控制流——能提升基准分数，但分数上升本身无法区分改进机制：可能是吃了数据集捷径，可能是多烧了推理算力，也可能是真的沉淀了可迁移结构。

**做法**：HDA 构造两个受控变体并在同一测试集上重新打分——给基线配平算力的 `B_cc`,以及剥去过拟合成分的 `E_neutral`——然后读差:`T = S(B_cc) − S(B)`、`O = S(E) − S(E_neutral)`、`G = S(E_neutral) − S(B_cc)`,且 `O + T + G = S(E) − S(B)` 恒等封闭 [README: intro]。

**证据**：跨四个基准(ALFWorld、LiveMath、CREATE、SWE-bench Verified)的分析结论是"most search-set gains turn out to be Overfitting or Test-Time Scaling; the Generalizable residual is often small" [README: intro];方法以任务无关的 `HDA_SKILL.md` 六步流程发布，其中 Step 3(compute-matching 计划)与 Step 4(neutralization diff)两处强制人工审批，因为"`B_cc` construction and the 'what counts as overfitting' call are the two judgment-heavy operations" [README: Using the HDA skill on a new task]。

**边界**：README 本身不含任何具体 O/T/G 数值(量化结论在配套博客与 runs 工件中)；两个受控变体的构造质量依赖人工判断而非全自动化程序；原始 per-item 日志与完整执行轨迹(多 GB)未发布，第三方验证只能到 summary 层级。

## Key takeaways

- **账本设计可移植**：任何“harness/agent 改进让分数涨了 X”的主张都可拆为 O + T + G;恒等式保证拆完无余数，争论从“涨没涨”变成“涨在哪一项”。
- **三机制的操作性定义**：O = dataset shortcuts / hardcoded task knowledge / "answering with code instead of the model"(不迁移)；T = 更多采样、重试、验证(真实但付算力代价)；G = 可复用结构与 guardrails(目标)。
- **判断密集处显式不自动化**：方法不假装有客观算法判定“什么算 overfitting”与算力配平，而是把这两步设计成人工审批闸口——这是一个值得复制的工程决策。
- **发布形态是 skill 而非库**:`HDA_SKILL.md` 任务无关，通过 per-task hook block(解析 item、scoring aggregator 的可分性、compute meter)接入新任务，配套 `sig_test.py` 的 paired-bootstrap 显著性闸门。
- **仓库是分析伴随物**：包含每个 run 的全部演化 harness、per-candidate 分数摘要、proposer 假设、赢家、数据 splits 与一份演化引擎；量化叙事在博客中。

## Decisions / claims

- 断言：四个基准上大多数 search-set 增益属于 O 或 T;G 的残差往往很小 [README: intro]。
- 决策：分解必须可加且封闭(`O + T + G = S(E) − S(B)`),三个量由两个受控变体在同一测试集上的重打分读出，而非拟合或估计 [README: intro 公式]。
- 决策:`src/hda/HDA_SKILL.md` 是方法的唯一事实源，`sig_test.py` 提供流程内建的 paired-bootstrap 显著性门槛 [README: Repository map]。
- 决策：六步流程中 Step 3 与 Step 4 暂停等待人工批准 [README: Using the HDA skill on a new task]。
- 范围声明：ALFWorld/LiveMath 覆盖 Qwen3.5-0.8B/2B/4B、Qwen3-8B、Qwen3.6-35B-A3B;CREATE 覆盖 Claude Haiku 4.5、Qwen3.6-35B-A3B;SWE-bench Verified 覆盖 Qwen3-4B/30B-A3B-Instruct、Qwen3.6-27B、Qwen3-Coder-30B-A3B;proposer 全程为 Claude Code (Opus 4.8) [README: Tasks & executors]。
- 发布取舍：包含全部演化产物与 per-candidate 摘要，不包含原始 per-item 日志与完整轨迹 [README: What's here vs. not]。

## Constraints & assumptions

- **executor 冻结假设**：整个分解的前提是 executor 不变、全部 delta 归因于 harness;executor 变化则账本失效。
- **可加性假设**:O、T、G 能在两个构造变体上正交读出；若机制间存在交互(例如 hardcoded 捷径同时降低所需采样数)，读数可能互相污染——README 未讨论此风险。
- **compute 可计量假设**：构造 `B_cc` 需要一个能把“推理算力”压成可比数字的 compute meter;该 meter 的定义是 per-task 判断，无统一标准。
- **scoring aggregator 可分性**：hook block 要求基准的打分聚合可分离，隐含对基准形态的限制，不满足者无法直接套用 skill。
- **人工审批依赖**：结果可信度上限等于审批者在 compute-matching 与 neutralization 上的判断质量；方法对这两步不提供客观程序。
- **复现边界**：无 per-item 原始日志，显著性检验输入为 summary 级分数；且所有演化出自单一 proposer(Claude Code Opus 4.8),结论可能与该 proposer 的偏差倾向耦合(文档未触及)。

## Open questions

- README 未给出任何量化 O/T/G 比例；“多数是 O 或 T”在各基准上的具体分布需要到博客与 `runs/` 工件中核实。
- compute-matching 的具体度量口径(采样数、token 数、验证调用数)在 README 层面未定义，只存在于 skill 的 per-task hook 中。
- neutralization 若过严会把真实结构改进误归为 O,系统性低估 G——文档未说明敏感性分析或防呆机制。
- 同一演化结果在不同审批者、不同 `B_cc` 构造下 O/T/G 读数的方差有多大，未见报告。

## Relations

- extends ADAS / DSPy / GEPA 类自动 agent 与 scaffold 搜索 [med]:HDA 不提出新搜索算法，而是对该类系统产出的 harness 做增益归因审计；README 开篇即以“proposer LLM 迭代编辑 harness”的场景为对象。
- builds-on test-time scaling 文献(Snell et al. 2024;"Are More LLM Calls All You Need?" 等)[med]:T 分量把“更多采样、重试、验证”显式建模为待扣除的算力混淆，与该文献“涨分来自算力还是能力”的追问同构。
- builds-on benchmark contamination / shortcut learning 文献 [med]:O 分量的定义(dataset shortcuts、hardcoded task knowledge、以代码代答)直接对应基准污染与捷径学习研究的关切。
- orthogonal to agent 评测基准构建(τ-bench、SWE-bench 等)[low]:HDA 消费这些基准而不改造它们；SWE-bench Verified 在此是研究对象而非贡献对象。
- method-analogous to 消融与因果中介分析 [low]:通过构造两个中间受控点(`B_cc`、`E_neutral`)做差分归因，形式上与两步中介分解同构。

## Takeaway

- 可搬走:O/T/G 可加账本 + “两个受控变体读差”的设计，是对任何 harness/scaffold 优化结果的最简审计框架；把 judgment-heavy 步骤显式设计为人工审批闸口，是可复用的方法论决策。
- 需存疑：README 层面零数值，方向性结论(“G 往往小”)依赖博客；O/T/G 读数的质量受制于两个未经客观化的判断点，且全部演化出自单一 proposer。
- 记忆钩子：分数涨了 ≠ 变强了——先扣算力(T),再扣捷径(O),剩下的才是真增益(G)。
