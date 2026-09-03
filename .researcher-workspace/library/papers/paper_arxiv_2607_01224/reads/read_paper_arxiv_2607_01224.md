---
title: "AutoMem: Automated Learning of Memory as a Cognitive Skill"
authors: ["Shengguang Wu","Hao Zhu","Yuhui Zhang","Xiaohan Wang","Serena Yeung-Levy"]
paper_id: "paper_arxiv_2607_01224"
source_kind: "arxiv"
source_id: "arxiv:2607.01224"
source_url: "https://arxiv.org/abs/2607.01224"
pdf_url: "https://arxiv.org/pdf/2607.01224"
read_id: "read_paper_arxiv_2607_01224"
kind: library-read
tags: []
---

# AutoMem: Automated Learning of Memory as a Cognitive Skill

> 将记忆管理拆分为结构优化与能力训练双轴，用 meta-LLM 自动审查完整长程轨迹来驱动两轴闭环。

## Brief

本文针对 LLM agent 在长程任务中的外部记忆管理问题，提出 AutoMem 框架。该框架通过两个外循环自动优化记忆技能：第一循环由 meta-LLM 审查完整 episode 轨迹并迭代修订 agent 的代码/prompt/文件 schema（结构轴）；第二循环从大量 episode 中筛选优质记忆操作作为监督训练数据，LoRA 微调一个专用的 memory specialist 模型（能力轴），而任务模型权重保持冻结。在 Crafter、MiniHack、NetHack 三个程序化生成的长程游戏上，仅优化记忆即可使 Qwen2.5-32B-Instruct 性能提升约 2×–4×，达到与 Claude Opus 4.5 和 Gemini 3.1 Pro Thinking 相当的水平。

## Claims

- 记忆管理是一种可独立学习的高杠杆技能，无需修改模型的任务行为权重即可显著提升长程任务表现 [§1, §3.2]。
- 仅通过 scaffold 优化（不改模型权重），Crafter 进展率从 25.0% 升至 47.27%（×1.89），MiniHack 从 7.5% 升至 27.5%（×3.67），NetHack 从 0.42% 升至 1.57%（×3.74）[§3.2, Table 1]。
- 在 scaffold 优化基础上叠加 memory specialist 训练，进一步提升 Crafter 至 51.36%、MiniHack 至 30.0%、NetHack 至 1.85% [§3.2, Table 1]。
- 优化后的 32B 模型在三个游戏上全面超越 Qwen2.5-72B-Instruct，表明在长程任务上，结构化外部记忆管理比模型规模扩展更具杠杆 [§3.2]。
- 优化后的 32B 模型达到 Claude Opus 4.5（49.5/27.5/2.0）的同等水平，接近 Gemini 3.1 Pro Thinking（55.0/27.5/2.6）[§3.2, Table 1]。
- scaffold 优化使无用任务动作率下降 32%–65%，重复记忆写入下降 68%–83%，空搜索率下降 13%–50%，每步输入 token 量下降 3%–30% [§3.2, Figure 4]。
- 训练后的 memory specialist 展现出"先查后写"的记忆纪律：LOG 阶段写入与搜索的比值在三个环境中分别下降 54%、72%、72% [§3.2, Table 2]。
- meta-LLM 能够审查长达 10⁴–10⁵ 步的完整 episode 轨迹并诊断记忆失误，在人类审查不可行的场景下实现自动化优化 [§1, §2.2]。

## Assumptions

- 文件系统操作（read/write/search/append/create）作为统一动作空间的一部分，足以表达 agent 所需的全部外部记忆管理行为 [§1]。
- meta-LLM（Claude Opus 4.6/4.7）具备审查数万步完整轨迹并产出有效诊断和代码修订的能力，论文未对此能力的边界做独立验证 [§2.2, §2.3]。
- 记忆技能可从任务能力中干净分离——即可以单独训练 memory specialist 而不损害任务模型的行为能力 [§2.3]。
- 程序化生成的游戏环境（Crafter/MiniHack/NetHack）中的记忆管理需求可代表长程任务中的通用记忆需求 [§1, §3.1]。
- 每个 episode 的记忆为情景性记忆（episode 结束后重置），无需跨 episode 的持久记忆 [§6]。

## Method

**内循环 agent**：单次 episode 中，agent 携带一个磁盘文件目录作为外部记忆。每步执行两个例程：LOG 例程决定记录什么（追加/创建/重写文件），PLAN 例程决定检索什么（搜索/读取文件）并提交下一步任务动作。记忆操作（`<|APPEND|>`、`<|SEARCH|>` 等）与任务动作处于同一动作空间，由同一次 forward pass 决定 [§2.1]。

**外循环 1（结构优化）**：meta-LLM（Claude Opus 4.6）读取完整 episode 轨迹——逐步日志、最终记忆目录内容、agent 代码本身——诊断 scaffold 导致的失败模式，产出代码/prompt/文件 schema 修订。每轮修订在相同固定种子上评估，仅当平均进展率严格提升时才接受修订；失败时允许 1 次重试，再失败则从新 session 重启。实际收敛在 2–5 轮迭代 [§2.2, Appendix A.2]。

**外循环 2（能力训练）**：meta-LLM（Claude Opus 4.7）作为训练引擎，联合决定三个要素：(i) 数据选择标准与组成、(ii) 从 episode 池中筛选优质记忆操作 trace 作为训练样本、(iii) 匹配的 LoRA 超参配置。训练数据全部为 agent 自身 episode 中产出的 verbatim 文本，meta-LLM 仅做筛选不做生成。筛选后经确定性后处理：清理格式、过滤无记忆操作的纯动作样本、裁剪混合样本中的动作部分 [§2.3, Appendix A.2 Stage b]。

**推理部署**：运行两个共享对话历史的模型实例——memory specialist（LoRA 微调副本）处理 LOG 例程和 PLAN 中的记忆查询部分；gameplay model（未修改的基模型）提交世界动作 [§2.3]。

## Eval

**数据**：BALROG 基准的三个程序化生成长程游戏——Crafter（17 动作/22 成就/~10³步）、MiniHack（8 任务/33 动作/~10²步）、NetHack（200+ 动作/10⁴–10⁵步）[§3.1, Figure 2]。

**指标**：游戏进展率（progression rate, 0–100%），Crafter 为 22 项成就完成比例，MiniHack 为 8 任务完成比例，NetHack 为地下城与经验等级进展指标。10 个固定种子 [42–51]，Crafter 10 episodes，MiniHack 40 episodes（5×8），NetHack 5 episodes [§3.1]。

**基线**：(i) BALROG 排行榜前沿模型——Gemini-3-Pro、Gemini-3.1-Pro-Thinking、Claude-Opus-4.5、Gemini-2.5-Pro、DeepSeek-R1、Qwen2.5-72B-Instruct、Qwen2.5-7B-Instruct；(ii) 同基模型 Qwen2.5-32B-Instruct 配合基础上下文管理（sliding window 16 步、±CoT）[§3.1, Table 1]。

**基模型与 meta-LLM**：内循环为 Qwen2.5-32B-Instruct（vLLM 本地部署）；外循环 1 用 Claude Opus 4.6，外循环 2 用 Claude Opus 4.7 [§3.1]。

## Weaknesses

- 三个环境的 memory specialist 各自独立训练和部署，论文未验证 scaffold 或 specialist 在环境间的迁移性，也未报告跨环境联合训练的任何初步结果——但论文主张记忆管理是"通用可学习技能"，这一主张缺乏跨环境泛化的直接证据支撑 [§6 author-acknowledged limitation, but the gap between the general claim and the per-environment evidence is not acknowledged]。
- meta-LLM 审查 10⁴–10⁵ 步轨迹的能力是整个框架的关键依赖，但论文未对 meta-LLM 的诊断准确率做定量评估——没有 ground-truth 标注的"真实记忆失误"来衡量 meta-LLM 的诊断精度和召回率 [§2.2]。
- NetHack 的绝对进展率极低（优化后 1.85%），标准误为 ±0.44，这意味着 5 个 episode 的评估噪声相对于绝对值非常大；论文将此作为核心成功案例之一，但统计可靠性不足 [Table 1]。
- 外循环 2 的训练数据来源于基模型自身在优化 scaffold 下的 episode，存在 self-bootstrapping 的潜在偏差——模型只能强化自己已有的"好"行为，无法获得超出当前能力范围的新记忆策略 [§2.3]。
- 所有实验环境均为游戏，其记忆需求（地图坐标、物品清单、遭遇日志）结构化程度高且可由文件系统自然表达；论文未讨论该方法在记忆需求更抽象或更依赖语义推理的任务（如多轮对话、代码开发、研究规划）中的适用性 [§6]。
- scaffold 优化的"收敛"判定为"平均进展率不再严格提升"，但该准则可能过早停止——meta-LLM 可能在某轮迭代中未能找到有效修订不代表结构空间已穷尽 [Appendix A.2]。

## Relations

- builds-on MemGPT (Packer et al., 2023) [high]: AutoMem 将 MemGPT 的 OS 启发式外部记忆管理思想从固定架构升级为可训练技能；论文 §4 明确引用并区分——MemGPT 的记忆管理机制是预设的，AutoMem 的由模型自主决策并通过 meta-LLM 迭代优化。
- extends ADAS (Hu et al., 2024) [high]: AutoMem 的外循环 1 与 ADAS 的"搜索 agent 架构代码空间"同构，但 AutoMem 将搜索目标收窄到记忆 scaffold 并将更新信号从任务级指标改为完整长程轨迹的诊断分析；论文 §4 明确引用并区分。
- builds-on ReAct (Yao et al., 2022) [high]: AutoMem 将记忆操作提升为与任务动作并列的一类 action，直接采纳 ReAct 的"推理-行动交错"框架；论文 §1 引用 ReAct 作为"动作空间中混合推理与行动"的基础。
- competes-with MemEvolve (Zhang et al., 2025a) [med]: 两者都用 LLM 驱动的自动优化来改进 agent 记忆系统，但 MemEvolve 从模块化设计空间（encode/store/retrieve/manage）演化架构，AutoMem 直接重写 agent 代码和文件 schema；论文 §4 明确区分——MemEvolve 基于 per-question QA 日志，AutoMem 基于完整长程轨迹。
- extends MemLLM (Modarressi et al., 2024) [med]: MemLLM 训练模型使用专用读写记忆模块，对应 AutoMem 的能力轴（外循环 2）；AutoMem 额外覆盖结构轴（外循环 1），论文 §4 明确将 MemLLM 归为"最接近 proficiency axis 的前作"。
- competes-with MeMo (Quek et al., 2026) [med]: 两者都训练独立 memory model 部署在冻结基模型旁；但 MeMo 的 memory model 编码静态文档知识用于 QA，AutoMem 的 memory specialist 处理长程 agent 的动态记忆管理决策；论文 §4 明确区分。
