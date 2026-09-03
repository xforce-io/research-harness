---
title: "Can a Language Model Learn Facts Continually in Its Weights?"
authors: ["Charles O'Neill"]
paper_id: "paper_arxiv_2607_11020"
source_kind: "arxiv"
source_id: "arxiv:2607.11020"
source_url: "https://arxiv.org/abs/2607.11020"
pdf_url: "https://arxiv.org/pdf/2607.11020"
read_id: "read_paper_arxiv_2607_11020"
kind: library-read
doc_type: "paper"
tags: []
---

# Can a Language Model Learn Facts Continually in Its Weights?

> 语言模型写新事实入权重时，训练数据广度决定其创造的是"可复述知识"还是"可应用知识"；而遗忘的本质是路由丢失而非存储抹除，当事实需被组合或存活于后续写入时，上下文才是可靠信道。

## Essence

**问题**：持续学习期望模型通过权重写入不断获取新知识，但已有工作未统一解释为何某些写入产生可用知识、为何某些能存活、以及被遗忘的事实留下了什么。

**做法**：用虚构事实写入 Qwen3 模型并追踪其从创建到经历 20–100 次后续写入的全过程。对比两种数据条件（裸语句 vs. 24 条多样化改述），通过五类问题（复述/改述/应用/组合/反事实）和双评分策略（严格/宽松）区分"复述"与"使用"。同时以原模型和 fact-in-prompt 为地板/天花板锚点。

**证据**：20 次连续写入后，裸语句事实仅保留 1% 准确率，而广数据事实保留 46%；但被遗忘的事实仍保留写入时 log-probability 增益的 57–67%——存储仍在，只是路由被新写入劫持。

**边界**：所有实验仅在 Qwen3-4B（含 8B 复制）上进行，使用 LoRA 适配器为主；结论限于离散事实，技能类持续学习未测试。

## Claims

1. 训练数据广度决定写入知识的类型：裸语句训练产生"复述知识"（entailment gap 27.4 分），多样化改述将其降至 5.4 分且不向模型展示任何推导结论 [§3, §7.1]。
2. 知识类型预测存活率：20 次连续写入后，裸语句事实保留 1%，广数据事实保留 46%（配对差异 45.6 分，95% CI [38.8, 52.6]）；100 次写入后广数据事实稳定在 25–28% 平台 [§4.1, §4.3]。
3. 遗忘破坏访问而非存储：行为性遗忘的事实仍保留写入时 log-probability 增益的 57–67%（drift-corrected），裸语句条件下 70% 的错误答案包含最近写入事实的内容 [§5.1]。
4. 两个已写入事实在覆盖前已无法可靠联合使用：双事实问题在两者均入权重时准确率 32%，均入上下文时 91%（配对差异 −58.1 分）[§5.2]。
5. 上下文信道在持续写入中不比通用能力退化更快：差异中的差异 +9.2 分；被遗忘事实在上下文恢复后达到 77–80% 准确率 [§5.3]。
6. 能力损伤与 KL 散度正相关（Spearman ρ = 0.83 跨 12 条件，ρ = 0.946 跨 16 条件），但 KL penalty 能保护能力却不能降低实际 KL 距离 [§6.1, §6.3]。
7. 冻结教师蒸馏可在近零能力代价下顺序写入 20 个事实（+2 分能力，54% 保留率），而自身累积合并教师导致 −28 分能力、11% 保留率 [§6.2]。
8. 干扰由新写入而非存储方式引起：2×2 交叉实验中，改入写为 study vs. bare 改善保留率 37.6 分，存储方式效应仅 −2.4 分 [§7.2]。
9. 线性化 Adam 更新可预测下一步对旧知识的影响（ρ = 0.795）但无法预测最终遗忘轨迹（ρ = −0.258）；所有三种局部干预（bridging data、activation-guided projection、梯度投影）均未达到阈值 [§7.3]。

## Assumptions

- 虚构事实与真实事实在持续学习机制上行为等价（论文未直接论证此泛化性）。
- 五类问题 + 双评分策略构成"可用知识"的充分表征（隐含于整个评估框架）。
- LoRA 适配器合并是权重写入的合理代理；全参数微调的验证仅在少数条件下完成。
- 模型生成的训练数据、问题和评分构成有效工具（通过 §2.1 认证流程验证，但认证本身依赖模型判断）。
- GRPO 的失败归因于信号不足而非 RL 目标本身的根本限制 [§3.3]。

## Method

**对比基线**：

| 维度 | 裸语句训练 | Study 训练 |
|------|-----------|-----------|
| 训练数据 | 事实句 + 2 条 trivial framing | 24 条改述、QA、推导、对比 |
| 优化步数 | 24/96/192 | 24/96 |
| 知识类型 | 复述为主（gap 27.4） | 可应用（gap 5.4） |

**写入流程**：(1) 对每个事实训练一个 LoRA 适配器（rank 16，192 步）；(2) 顺序条件下，合并当前适配器到模型后再训练下一个；(3) 在每 k 个写入后用 5 类问题评估所有先前事实。

**蒸馏目标**：教师 π_T = 原模型加 fact-in-prompt，学生不见事实。Offline 蒸馏用教师采样序列，online 蒸馏用学生采样（forward/reverse KL 两种方向）。

**存储探针**：对事实语句 s 在写入前 θ₀、写入后 θ_j、写入 k 步后 θ_{j+k} 分别测量 log-probability，定义保留分数 R(k) = (log p_{θ_{j+k}}(s) − log p_{θ₀}(s)) / (log p_{θ_j}(s) − log p_{θ₀}(s))。

**梯度投影干预**：将 Adam 更新投影到与旧事实梯度正交的方向，∆θ_⊥ = ∆θ − max(0, ĝ_use^T ∆θ) ĝ_use，期望在不损害旧知识的前提下应用新写入。

## Eval

- **数据**：247 条虚构实体单句事实（primary eval）；prior-conflict eval 含 prior-neutral 和 prior-inverting 事实；5 类问题 × 每事实多题；14 个事实对用于联合使用测试（47 题）。
- **模型**：Qwen3-4B（主），Qwen3-8B（复制 entailment gap）；LoRA rank 16（主），rank 4 / full FT（对照）。
- **基线/锚点**：原模型（floor, 1–7%）+ fact-in-prompt（ceiling, 75–99%）；5 方法对比（bare SFT / study SFT / offline distill / online fwd-KL / online rev-KL）。
- **指标**：5 类问题 strict/lenient 准确率；entailment gap = lenient − strict；retention@k（第 k 次写入后旧事实准确率）；capability loss（100 项规则评分测试，基线 ~80%）；KL divergence from original。
- **规模**：sequential 20–100 writes × 3 seeds；factorial 16 条件 × 3 seeds × 40 facts；crossed 2×2 × 3 seeds × 24 stored facts × 10 later writes。
- **评分**：确定性 checker + GPT-5.4-mini judge（reasoning off, temp 0）+ 第二 judge 家族交叉验证；bootstrap 95% CI over facts。

## Weaknesses

- 所有结论依赖单一模型家族（Qwen3-4B），8B 仅复制了 entailment gap 而非核心保留率或遗忘机制；无法排除这是 Qwen3 特有的参数组织结构。
- 联合使用实验仅覆盖 14 个事实对（47 题），论文据此声称"两个写入事实无法可靠组合"，但样本量对如此强的结论偏小。
- 评分管道依赖模型 judge（GPT-5.4-mini），认证中的 pass-audit 为 29/30–30/30，但 60 项审计发现 7 个 judge 错误（约 12% 错误率），对几个百分点的差异比较构成潜在混淆。
- GRPO 条件"几乎没收到信号"即被放弃 [§3.3]，但未尝试替代 RL 目标设计（如 reasoning-length-aware reward），仅笼统留给 future work；不能据此排除 RL 范式写入知识的可能性。
- 存储探针 R(k) 仅读取事实语句本身的 log-probability，而非其推导结论的概率；论文承认这一点但仍将 R(k) 作为"存储仍在"的证据，逻辑链对 study 事实成立（因 use questions 曾通过）但对 bare-statement 事实的推断更弱。
- frozen-teacher 顺序蒸馏的 54% 保留率是全文最佳，但该条件下保留率测量仅基于 3 seeds × 20 facts，且未在 prior-conflict eval 或 100-write 延展中验证，稳健性不明。
- §7.2 的 2×2 交叉实验中，bare-statement 写入流产生 4.2–3.2% 的重复生成失败和 fact-in-prompt 参考下降 45.5 分，虽然排除失败项后效应仍在（36.8 分），但 bare-statement 流的退化是否代表"选择性干扰"还是"全面退化"仍存疑。

## Relations

- **builds-on** Allen-Zhu & Li (2024) [high]：§3 中 paraphrase 使事实更可提取的结论直接扩展为训练数据广度的因果变量分离；论文 §8 明确引用并将此从单次写入扩展到连续写入场景。
- **builds-on** Shenfeld et al. (2025) [high]：KL-from-base-policy 预测遗忘的结论被扩展到知识写入领域；§6.1 明确引用并将其从 RL 场景推广到 SFT/distillation 写入。
- **contradicts** knowledge editing 文献（Meng et al. 2022/2023; Zhong et al. 2023）[med]：编辑方法在单跳问题上 >90% 成功，但论文 §8 指出其在多跳使用和持续写入下失败；论文的 question-keyed 框架为这些失败提供了统一解释（地址缺失而非存储缺失）。
- **extends** Tulving & Pearlstone (1966) 的 availability vs. accessibility 区分 [high]：论文 §8 明确借用此词汇框架，将行为性遗忘重新概念化为"路由丢失"而非"存储抹除"，并提供了权重层面的定量证据。
- **orthogonal** Dai et al. (2026) [med]：Dai 的机制性工作定位记忆事实停留在直接回忆可达但中层多跳计算不可达的层，论文 §8 引用此作为 question-keyed 框架的机制层面佐证，但两者方法论不同（activation patching vs. 行为实验）。

## Takeaway

- **方法论值得借鉴**：双评分策略（strict/lenient → entailment gap）将"复述"与"使用"量化为一个可测属性，而非仅依赖单一准确率；fact-in-prompt 天花板 + 原模型地板的双锚设计为所有比较提供了校准基准。
- **需谨慎对待的结论**：存储探针（R(k)）仅测量语句 log-probability，将其作为"存储未损"的证据对 study 事实较强、对 bare-statement 事实较弱；54% 保留率（frozen teacher）的样本量和验证范围有限。
- **核心洞察**：权重写入存储内容但不创建地址（address）——模型能回答碰巧路由到该事实的问题，却无法按需检索或组合多个已写入事实。遗忘是新写入劫持了旧问题的路由，而非抹除了旧知识的痕迹。
- **一句话**：权重是错误的事实系统记录——它存内容不给地址；当事实需被组合或存活于后续训练时，上下文才是可靠信道。
