---
title: "Controlling Reasoning Effort in LLMs"
authors: []
paper_id: "paper_url_2be8751c7abf335d"
source_kind: "url"
source_id: "url:https://magazine.sebastianraschka.com/p/controlling-reasoning-effort-in-llms"
source_url: "https://magazine.sebastianraschka.com/p/controlling-reasoning-effort-in-llms"
pdf_url: ""
read_id: "read_paper_url_2be8751c7abf335d"
kind: library-read
doc_type: "other"
tags: []
---

# Controlling Reasoning Effort in LLMs

> 推理模型在推理时输出长度无法按需控制 -> 通过 SFT 混合训练 + 条件化 RL + 硬预算截断让单一模型支持多档推理努力 -> 同一模型可在成本与精度间动态权衡，无需部署多个检查点。

## Essence

1. **问题**：第一代推理模型（如 DeepSeek-R1）无差别输出冗长推理链，即使简单问题也消耗大量 token，且无法关闭推理模式；用户缺乏按任务难度调节推理预算的手段。
2. **做法**：文章梳理了六种旗舰开源模型（DeepSeek V4、Nemotron 3 Ultra、Kimi K2.5、GLM-5、Qwen3、Inkling）实现多档推理努力的真实训练管线，归纳出三个共享组件——(a) SFT 阶段引入 effort 标签与混合数据让模型学会模式切换，(b) RL 阶段用 context window / length penalty 做条件化奖励，(c) 硬预算截断训练让模型在被强制中断后仍能产出有效答案。
3. **证据**：Inkling 的奖励公式 `R(e) = R_task - λ(e) · N_tokens` 是最具体的案例——effort 值直接调节 per-token cost，低 effort 用大 λ 鼓励短输出，高 effort 用小 λ 允许长推理 [§5.3]。Nemotron 则通过随机截断推理链 + 保留原始答案来训练 SFT，使 `</think>` 被外部关闭后模型仍能续写答案 [§6.2.2]。
4. **边界**：OpenAI GPT-5.6 的训练细节未公开；文章基于 gpt-oss chat template 和开源模型报告做推测。各模型报告省略了大量超参，无法做受控对比。自动 effort 选择（GPT-5 Auto 模式）已被移除，说明该问题尚未解决。

## Key takeaways

- `<think></think>` 标签是纯格式标记，不赋予模型推理能力，更换其他定界符效果相同；训练时通过 format reward 鼓励其使用 [§3]。
- Qwen3 的 Thinking Mode Fusion 是最简单的 on/off 方案：SFT 阶段混合 `/think` 和 `/no_think` 示例，`/no_think` 以空 `<think></think>` 开头；推理时 `enable_thinking=False` 硬注入空标签块作为"硬开关" [§4]。
- DeepSeek V4 为三种 effort 模式分别训练 specialist（不同 context window + length penalty），再蒸馏为单一检查点——比 Qwen3 重得多但每种模式独立优化 [§6.1]。
- Kimi K2.5 的 Toggle 方法交替执行"有预算 RL"和"无约束 RL"两阶段，避免模型因固定预算过拟合到短解而丧失长推理能力；在 K2 Thinking 上减少 25–30% token 且基准性能几乎不变 [§6.3.1]。
- Nemotron 3 Ultra 将 learned effort mode 和硬推理预算解耦：mode 决定模型如何使用推理 token，budget 决定推理链能跑多久，两者可自由搭配 [§6.2.1]。
- 训练 scaling（选不同大小的模型）和推理 scaling（调 reasoning effort）是正交的两个轴；小模型 + 高 effort 有时能逼近大模型 + 低 effort 的性能 [§5.4]。

## Decisions / claims

- 推理模型的推理链（reasoning trace）在 RLVR 训练中不参与奖励计算，仅最终答案和格式决定 reward [§2.1]。
- DeepSeek-R1-Zero 证明纯 RLVR（无 SFT）即可让模型学会生成推理链和自我纠错 [§2.2]。
- gpt-oss 通过 system prompt 中的 `"Reasoning effort: low/medium/high"` 控制 effort，推理时映射为 system message [§5.1]。
- effort-conditioned 训练的两种主要实现路径：(1) RLVR 阶段根据 effort 标签施加不同 length penalty；(2) RLVR 后用 SFT 让模型学习 effort 标签到目标推理长度的映射；两者可组合 [§5.2]。
- Inkling 使用连续 effort 值（0.0–1.0）而非离散标签，effort conditioning 主要放在 RL 阶段而非 SFT [§5.3]。
- GLM-5 将 on/off 扩展为三种行为：interleaved thinking（每次工具调用前插入推理块）、preserved thinking（跨轮保留推理块）、turn-level thinking（逐轮开关）[§6.4]。
- Qwen3 的硬思考预算行为（在阈值处停止推理并插入 stop-thinking 指令）并非显式训练，而是在 Thinking Mode Fusion 后涌现 [§6.5]。
- 六个开源模型的共享框架：SFT + chat template 引入模式控制 → mode-conditioned RL 调整 context window 和 length penalty → 预算鲁棒性训练（截断/交替/停止指令）[§6.7]。
- GPT-5 的 Auto 模式"probably more miss than hit"已被移除，自动 effort 选择仍是未解难题 [§7]。

## Constraints & assumptions

- 所有 effort 控制方法均要求训练阶段引入 effort 标签或预算信号；对未经此类训练的任意模型，仅在 system prompt 中添加 effort 指令不会产生相同效果 [§5.2, §6.1]。
- 可验证奖励域（数学、代码）是 RLVR 的前提；非可验证域无法直接应用此范式 [§2.1]。
- 各模型报告省略了 teacher 分配、reward 超参、数据配比等关键细节，无法做受控横向对比 [§6.1, §6.7]。
- 推理努力的收益存在饱和点——GPT-5.6 Sol 在最高 effort 档位出现边际递减甚至不经济 [§5.1]。
- Toggle 的预算约束仅在问题平均准确率超过阈值后激活，以避免在模型尚未学会解题时强制缩短推理 [§6.3.1]。
- 文中 GPT-5.6、gpt-oss 的训练管线描述均为基于公开信息的推测，非官方确认 [§5.1–5.2]。

## Open questions

- 自动 effort 选择（根据任务状态、剩余预算推断合适模式）仍无可靠方案；GPT-5 Auto 模式的失败表明该问题难度被低估 [§7]。
- DeepSeek V4 的三种 effort specialist 如何映射到十多个 domain specialist，报告未披露 [§6.1]。
- Kimi K2.5 的 instant mode 是否有独立的 RL 训练配方，报告未说明 [§6.3.2]。
- 不同 effort 实现方法之间无受控对比，无法判断哪种方法在特定场景（交互助手 vs. 长时编码 agent）下更优 [§7]。
- Inkling 未公开确切的 reward 公式、token-cost 系数，以及 effort conditioning 是否也出现在 SFT 中 [§5.3]。

## Relations

- builds-on DeepSeek-R1 [high]: 文章以 R1 的 RLVR 训练范式为基线，R1-Zero 证明纯 RL 可产生推理行为，是后续所有 effort 控制方法的前提 [§2.1–2.2]。
- extends Qwen3 technical report [high]: 文章直接引用 Qwen3 的 Thinking Mode Fusion 作为 on/off 开关的核心案例，并将其与 Nemotron 的硬预算做对比 [§4, §6.5]。
- extends Kimi K1.5 / K2.5 [high]: Toggle 方法在 K2 Thinking 上的评估被直接引用作为 token-efficient RL 的旗舰案例 [§6.3]。
- orthogonal DeepSeekMath-V2 [med]: 文章引用其 self-consistency + self-refinement 作为推理 scaling 技术的独立路径，与 effort 控制正交——前者是多次采样投票，后者是单次生成的长度控制 [§2.3]。
- competes-with Tülu 3 [med]: Tülu 3 在 SFT 模型上做 RL，R1-Zero 在 base model 上做纯 RL；文章指出两者对"是否需要 SFT 前置"给出不同答案 [§2.2]。

## Takeaway

- **可直接复用**：effort 控制的最小可行实现是 Qwen3 式 SFT 混合（thinking + non-thinking 示例）+ chat template 硬开关；需要更细粒度控制时加入 RL 阶段的 length penalty 条件化。
- **需要警惕**：固定 token 预算会让模型过拟合到短解、丧失长推理能力——Kimi 的 Toggle 交替训练是目前唯一公开的缓解方案。
- **不可信处**：GPT-5.6 / gpt-oss 的训练管线均为推测，非官方确认；不应将文章中的"possible implementation"当作事实引用。
- **记忆钩**：effort 标签只是 system prompt 里的几个字，但它背后的训练——SFT 混合、条件化 RL、预算截断——才是让它真正生效的东西。
