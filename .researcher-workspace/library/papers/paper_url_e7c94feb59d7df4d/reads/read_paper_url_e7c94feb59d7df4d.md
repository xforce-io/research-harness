---
title: "Self-Evolving Agent: A Closed-Loop"
authors: []
paper_id: "paper_url_e7c94feb59d7df4d"
source_kind: "url"
source_id: "url:https://alloomi.ai/reports/sea.pdf"
source_url: "https://alloomi.ai/reports/sea.pdf"
pdf_url: "https://alloomi.ai/reports/sea.pdf"
read_id: "read_paper_url_e7c94feb59d7df4d"
kind: library-read
doc_type: "other"
tags: []
---

# Self-Evolving Agent: A Closed-Loop Post-Training Flywheel for Continually Learning LLM Agents

> 部署后的 LLM agent 静态不进化、在线微调又招致灾难性遗忘与更新回归——MelandLabs 的 SEA 报告把质量过滤经验池、三阶段后训练飞轮（LoRA + 教师回放 + 蒸馏）、Engram X 稀疏权重记忆层、MAPE 多指标回滚门闭成一个部署级循环，在同一 backbone 内拿到 +23.1 pt，并让每一轮更新可审计、可回滚。

## Essence

**问题**：企业岗位上的 LLM agent 部署后是静态的（“今天处理再多发票，明天也不比今天更懂任务”）；持续学习（CL）研究集中在视觉/NLP 分类基准，不覆盖工具调用与多步决策的 agent 场景；而在线适应引入两类风险：灾难性遗忘，以及某轮更新悄悄劣化整体能力。

**做法**：每次任务交互产出 三元组，经质量过滤 Q(e)（置信度阈值 τc=0.7、反馈充分性、多样性采样）进入上限 K=50,000 的经验池；每积累 1,000 个新三元组触发三阶段飞轮——Stage 1 LoRA 在线微调（shared-layer-only，~130M 可训参数，0.4%），Stage 2 跨任务回放（回放目标是最强时刻的**教师**轨迹而非学生自身 logits，任务平衡权重 w(e)=(1/|Te|)·f(Te)），Stage 3 对强教师 Qwen3.7-Max（付费 API）做前向 KL 全词表蒸馏。Engram X 把每个被接受的三元组写成 rank-1 cell（Me=ve·ke^T），经 top-k=8 稀疏路由注入 4 个层（{8,16,24,32}）的 pre-Attention 残差流，与 LoRA 联合训练。MAPE 环用三指标 AND 门（reward + 校准误差 + trace 一致性）决定每轮 checkpoint 保留或回滚。

**证据**：SEA-mini（Qwen3.6-35B-A3B，35B/3B active MoE）CL-bench 47.6%，同 backbone FewShot 基线 +23.1 pt、最强同 backbone 检索基线 +8.2 pt（k=3 seeds，bootstrap CI 不含零）；Cont.L.Bench CLG +0.279（CI [+0.196, +0.349]）；10 任务保持率 51.3% vs 33.6%；去 Engram X −3.9 pt；教师目标换学生自 logits −6.1 pt。

**边界**：不主张门控的形式化单调性保证（需不可验证的验证分布稳定性假设）；全部实证在 k=3 seeds 上，BH q 值被明确排除出正式显著性陈述；教师为付费 API、无 API 不可复现；非替代性主张未完成 per-axes-removed sweep；提供的正文在 §3.6 中途截断，§4/§5 仅以引用出现。

## Key takeaways

- **归因纪律是本文最可移植的做法**：headline 严格定义为 within-backbone 对比；公开排行榜排名（最强 frontier LM GPT-5.4 (xhigh) 27.9%）"reported for external context only"，因与 SEA-mini 在 backbone、参数量、训练数据、推理预算上同时不同（Abstract/§1）。
- **记忆 = 模型内稀疏权重，而非检索侧信道**：Engram X per-triple 写入仅 2·dh 参数（~16 KB BF16，dh=4096），比 per-experience LoRA（需整轮 fwd/bwd）便宜 3–4 个数量级；rank-1 + 稀疏路由保证未激活 cell 不被污染，实现内容（sparse cells）/技能（shared LoRA）分离；pre-Attention 放置比 post-MoE 高 2.0 CL-bench pt（Table 16）。
- **回放教师而非学生**：Stage 2 回放最强时刻的教师轨迹；换成学生自 logits 目标 −6.1 pt——对 DER 式 dark replay（自 logits 随饱和熵收缩）在 agent 场景的反例主张。
- **三指标 AND 回滚门**：仅当 reward（δ=1%）、ECE（δ=0.02）、trace 一致性（δ=5%，token-level Jaccard 剥离 tool-call 后）三者同时劣化才回滚，压掉单指标噪声；20 轮 burn-in 实测门触发 4/20（1 真 3 假，Wilson CI [7.3%, 38.5%]）、假阴 0/20（Wilson 上界 ~14%）。门控是 harness 层运行时属性，文档特意命名为 "operations-layer self-validation" 而非 "self-validating rollback"（§3.4）。
- **循环边界四分契约**（§3.5）：固定（base 权重 W0、门/过滤/评测定义）/ 可变（θ(t)、M(t)、池内容）/ 不可修改 / 评测与训练分离（池源与 CL-bench 不相交；验证 split 为 10% 分层切出且每轮在三个 split 间轮换）——这是“增益归因于飞轮而非固定元素漂移”成立的最小声明。
- **蒸馏选型有数字支撑**：前向 KL（Hinton 约定、mode-seeking、全词表）比 logit-only +1.8 pt、比小教师（Qwen3-7B-Instruct）+2.6 pt（§4.4.1 引用）。
- **对抗面量化且预注册**：5% 注入 × 10 轮下，签名来源 + 教师二遍防御把注入成功率 0.94→≤0.06（k=3，CL-bench 代价 −0.8 pt）；不声称残差形式化上界，k=30 复跑的验收标准已预注册（残差 ≤0.05、Wilson 95% 上界 <0.10）。

## Decisions / claims

- 贡献定位：headline = within-backbone gap（47.6% vs 24.5% FewShot）；leaderboard 排名仅作外部语境，不用于量化框架贡献（Abstract；§1；§4.2.1 引用）。
- Engram X 定位：数学原语非原创——"the cells-per-token activation pattern, the key-value-product form … and the sparse routing mechanism are unchanged from Lample et al. [2024]"（§2.3）；新颖性仅在部署语境：pre-Attention 放置、单部署范围、与 LoRA 在同一 MAPE-gated 更新契约下联合训练（§2.9）。
- MAPE 门控自我定性：是 "a direct lift of Shukla et al.'s design into the LLM-agent setting"（§2.1），差异仅工程性：三指标 AND（vs 二指标）、held-out 验证集每轮重抽（vs 复用部署遥测）、显式放弃单调性保证并上报 burn-in + Wilson 界。
- LoRA 挂载策略：仅 attention Q/K/V/O 与专家 FFN 输入/输出投影，路由与 gate 冻结；依据是 top-2 专家熵从 base 到微调变化 <5%，per-expert LoRA 只多 ≤0.8 pt 但 adapter 大 ~9×（§3.3.1；§4.4.1 引用）。
- 超参与成本：L_total = L_LoRA + λr·L_replay + λd·L_distill，λr=λd=0.5（grid search）；AdamW lr=2e-4、batch 64、3 epochs、8×A100 约 2.3 小时/轮；触发阈值 N=1,000 新三元组；池填充至 K=50,000 需 ~28,000 次教师 rollout（§3.3）。
- 质量过滤操作点：τc=0.7 在 5% hold-out 校准；扫 {0.5, 0.7, 0.85} 内 retention 波动 ≤1.2 pt、CL-bench ≤0.9 pt，故不声称全局最优（§3.1；§4.5.1 引用）。
- Engram X 容量账：K=50,000 cells ≈0.82 GB（BF16），与 mini 级 LoRA adapter（~0.26 GB）同量级；per-token 开销 O(k·dh)≈32,768 FLOPs，<1% FFN（§3.2）。
- 消融主张：replay-only / distillation-only / replay-no-MAPE 变体比全系统低 1.5–7.0 pt（pass rate）与 14.8–17.6 pt（retention），跨两个 backbone（§2.8；§4 引用）；但非替代性主张明确限定于“已测变体”，per-axes-removed sweep 未完成。
- 迁移主张：post-trained adapter 无再训练严格迁移到 AppWorld 与完整 CL-bench（1,899 tasks，§4.5.3 引用）；SWE-Bench-CL in-harness 序列（273 tasks）测得 80.6%（Fig.1 caption；§4.2.5 引用）。
- 对抗防御组合与威胁模型参数对齐声明：α=0.05、T=10 轮在威胁模型与实验中同值（§3.6）。

## Constraints & assumptions

- 全部实证（headline、retention、CLG、消融、对抗残差）采样范围为 k=3 seeds；BH q 值在 k=3 resamples 上计算，文档自认非正式显著性。
- 教师 Qwen3.7-Max 为托管商业 API、按 token 计费，除 API 行为外不可审计；headline 依赖教师输出分布，no-teacher-anywhere 消融被推迟（§5.3 引用）——教师贡献与环路自身贡献的分解目前不可得。
- 门控无形式化单调性保证；20 轮 burn-in + Wilson 界是全部经验依据，且明确不外推到 burn-in 之外或验证分布漂移情形（§3.4）。
- 门控信号链本身教师锚定：trace 一致性 Cv 以教师 trace 为参照，验证 reward 也来自同一闭环——教师若有系统性偏差，门控继承之。
- Engram X 容量受 K=50,000 硬上限与稀疏预算 k 约束，依赖多样性驱逐规则维持覆盖。
- 注入层选择 {8,16,24,32} 由 held-out layer-importance sweep 定出，且明确绑定 Qwen3.5/3.6 家族；路由稳定性假设（专家熵 <5% 变化）是 backbone 特定的经验假设。
- 门控为运行时属性：验证集配置错误、度量实现被篡改或 harness 被攻破即失效——文档自己声明这是有意的设计取舍。
- 行为抽取威胁未被签名来源防御覆盖，教师二遍仅部分缓解（§3.6）。
- 海马体–新皮层类比被明确降格为启发性词汇：Stage 2 是梯度回放，非模式补全（§3.2 disclaimer）。
- **文本完整性约束**：所提供正文止于 §3.6 "rate limiting on the deployment API to bound the number of behavioural-extr" 处截断；§4（全部实验）与 §5（结论/推迟实验）仅以引用形式出现，Appendix A.1 同样未见。

## Open questions

- 残差注入率的形式化上界：k=30 对抗复跑为第一优先推迟实验，预注册验收标准已给出（§4.5.2 引用；§5.3 引用）。
- no-teacher-anywhere 消融：分解 headline 增益中教师 bootstrapping 的占比——文档自认这是“分解环路自身贡献”的最干净测试。
- k=60 门控 burn-in 扩展：20 轮不足以收紧假阴性上界（0/20 仅约束到 ~14%）。
- per-axes-removed sweep：非替代性主张目前只覆盖已测变体，不覆盖四要素的任意重排。
- 门控单调性的形式证明：需要部署中不可验证的验证分布稳定性假设，被列为 open problem。
- 更长时程部署（>10 任务）、跨架构泛化（dense vs MoE、Qwen 家族之外）、更强对抗者情形（结论段自列）。
- 学习式置信度阈值（对 held-out 人工标注集校准）替代固定 τc。
- 行为抽取的完整防御（截断处正在讨论 rate limiting 等最小生产原语，未完成）。

## Relations

- builds-on Memory Layers at Scale (Lample et al., 2024) [high]：Engram X 的 rank-1 key-value-product 与 top-k 稀疏路由原语逐字取自该工作，差异仅在部署语境（§2.3/§2.9 明示）。
- builds-on Cheng et al., 2026 (Engram) [high]：模块名与 n-gram-hash lookup 约定的来源（§2.3）。
- builds-on Li, 2026 (User as Engram) [high]：内容/技能分离的 per-user 版本；SEA 继承为较弱的 within-deployment 可组合性主张，并引用其“per-user Engram cells 比 per-user LoRA 小约 4 个数量级”作为外部验证（§2.3/§3.2）。
- builds-on Shukla et al., 2025 (Adaptive Data Flywheel / MAPE for AI agents) [high]：文档自认门控为其设计的直接移植，仅作工程改造（§2.1）。
- extends LoRA (Hu et al., 2022) [high]：从一次性微调改为每 MAPE 周期更新、以当前经验池为条件（§2.3）。
- contradicts dark experience replay (Buzzega et al., 2020) [med]：DER 主张回放模型自身 logits 强于原始样本；SEA 在 agent 轨迹层实测学生自 logits 目标 −6.1 pt，方向相反（§2.2/§2.9）。
- orthogonal Lu et al., 2026 自适应 memory-aware sampler [high]：文档自称可直接插入 Stage 2；同理 orthogonal：specialist-agent 路由（FlyRoute）、Ragen（提供环境基质）（§2.3/§2.6）。
- competes-with Voyager / Generative Agents / Sage（Wang et al., 2023a；Park et al., 2023；Wang et al., 2024）[med]：Table 1 定位表声称无一系统同时具备五轴（持久记忆/轨迹回放/蒸馏/MAPE 环/agent 任务）——作者自评，非独立验证。
- competes-with Self-Rewarding (Yuan et al., 2024) [med]：奖励来源分歧——task-grounded evaluator 信号 vs LLM-as-judge 偏好（§2.5）。
- 可组合 SoRFT (Ma et al., 2025) / Agent Lightning (Zhang et al., 2025) [high]：SEA 定位为 verifier-RL 之上的自进化外环（§2.6）。
- 注：文中全部 2025–2026 引用对象（Qwen3.7-Max、GPT-5.4、CL-bench、Cheng et al. 2026 等）在本次 read 内无法独立核实；本文为行业报告，非同行评审文献。

## Takeaway

- **可搬用**：within-backbone 归因纪律与“排行榜数字仅作 external context”的处理；三指标 AND 门 + 如实上报 Wilson 假阳/假阴区间；固定/可变/不可改/评测分离的循环契约；把记忆做成 in-graph 稀疏权重（rank-1 cell + top-k 路由 + 与 LoRA 联合训练）而非外部检索。
- **需存疑**：k=3 种子贯穿全部 headline；教师为付费 API 且 no-teacher 消融推迟，+23.1 pt 中教师与环路各自的占比未知（Engram X 自身仅贡献 3.9 pt）；门控的 trace 一致性以教师为参照，“自验证”实为教师锚定；行业报告、未经同行评审、正文 §3.6 截断且 §4/§5 未随文提供。
- **一句话钩子**：把“经验池 → 稀疏权重写入 → 教师回放与蒸馏 → 可回滚门控”闭成一个飞轮，并且只拿同 backbone 对比说话。
