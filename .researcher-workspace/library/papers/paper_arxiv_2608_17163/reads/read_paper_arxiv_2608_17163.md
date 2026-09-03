---
title: "Q-Learning With World Models"
authors: ["Perry Dong","Yueru Jia","Chelsea Finn","Dorsa Sadigh"]
paper_id: "paper_arxiv_2608_17163"
source_kind: "arxiv"
source_id: "arxiv:2608.17163"
source_url: "https://arxiv.org/abs/2608.17163"
pdf_url: "https://arxiv.org/pdf/2608.17163"
read_id: "read_paper_arxiv_2608_17163"
kind: library-read
doc_type: "paper"
tags: []
---

# Q-Learning With World Models

> 传统 model-based RL 把想象 rollout 喂进训练、模型偏差随之复合；QWM 让世界模型只在决策时把候选动作的近期后果展开成短视野树、用 Q 函数在想象状态上重新打分，策略与 critic 仍然只用真实转移训练。

## Essence

**问题**：model-based RL 把世界模型的想象 rollout 当训练数据或训练目标，模型偏差随任务视野与视觉复杂度复合，难以扩展到高维机器人场景；而 Q-learning 侧的 best-of-N（IDQL、EXPO 的 OTF）只用当前状态的即时 Q 值挑动作，看不到动作的近期后果。

**做法**：QWM 在标准 Q-learning 外套一层决策时树搜索：策略提出 N 个候选动作，世界模型对每个动作预测 K 个未来状态、递归到深度 D；每个节点的值取两个互补估计的平均——直接查 Q(s,a)（低方差、不吃模型误差）与沿想象 rollout 递归聚合的值（利用模型、误差随深度复合）；根上每个候选按聚合出的 tree-search 值取最大执行。同一搜索既用于在线采样（改善进 replay buffer 的数据）也用于评估，训练本身完全不变。

**证据**：Robomimic 四任务上 QWM 在样本效率与成功率上超过 RLPD/DSRL/QSM/QAM/FQL/IDQL 等 model-free SOTA；对比之下 TD-MPC2 与 EfficientZero V2 在报告训练步数内仅在 Lift 上拿到非零成功率 [§5.1]。

**边界**：收益依赖离线 demo 预训练且冻结的世界模型与可承受的决策时算力（像素设置已被迫砍掉评估期搜索）；它不是“用模型数据加速训练”——恰好相反，训练全程不见一条模型生成的转移。

## Claims

1. 在标准 Q-learning 之上仅于决策时使用世界模型做候选动作树搜索（在线采样与评估两处都用），即可同时提升性能与样本效率，且因策略与 critic 只在真实转移上训练而不产生复合模型偏差 [§1, §4]。
2. 用 Q 函数搜索显著强于只用状态价值 V 搜索：V 版（AWR 式训练）在全部设置下大幅落后 [§B]。
3. 节点值应由两个互补估计量等权组合：即时 Q 估计（深度无关、低方差、完全依赖 Q 精度）与沿想象 rollout 的递归估计（利用模型、误差随深度复合）[§4.2, Eq. 8]。
4. 树被刻意设计为短视野后果预测而非穷举规划，以抑制世界模型预测误差累积 [§4.1]。
5. 采样 + 评估双阶段搜索给出最一致的提升：在线搜索改善 replay buffer 中的经验质量，评估搜索直接改进动作选择，二者互补 [§5.4, Fig 7]。
6. 消融中搜索深度 D=2 总体最佳：过浅不足以区分候选动作，过深暴露模型误差 [§5.4]。
7. 聚合折扣 λ 存在适中甜点（消融中 λ=0.2 最强），过大对深层误差敏感、过小使深层搜索失效 [§5.4]。
8. 候选动作数 N 的最优值依环境而定，过大反而因放大模型与价值估计误差而损害性能 [§5.4]。
9. 性能对保留路径数 J 基本不敏感，贪心保留少量高值分支即可近似全树 [§5.4]。
10. Robomimic（Lift/Can/Square/Tool Hang，state-based，稀疏奖励）上，QWM 在样本效率与成功率上超过 RLPD、DSRL、QSM、QAM、FQL、IDQL 等 model-free 基线 [§5.1]。
11. 同一评测中 TD-MPC2 与 EfficientZero V2 在报告训练步数内仅在 Lift 上取得非零成功率 [§5.1, Fig 3]。
12. QWM 对 EXPO 与 RLPD 两个底座均带来一致提升，任务越难（Tool Hang、Square）增益越大 [§5.2, Fig 4–5]。
13. 像素设置（LIBERO 五任务）下即便因算力只在在线采样时搜索、评估不搜，QWM 仍整体优于 EXPO [§5.3, Fig 6]。

## Assumptions

- 存在可用的离线 demonstration 转移用于预训练世界模型；state-based 模型在 RL 全程冻结、不用在线数据更新 [§3, §C.3]。
- 稀疏奖励（除终止步外为零）设定成立，因此不学习 reward model，rollout 侧价值完全由想象状态上的 Q 估值承担 [§4.4]。
- Q 函数学得足够准——V_Q 估计量被明确描述为完全依赖 Q_φ 的精度 [§4.2]。
- 好动作落在策略采样的候选集合内：搜索只重排 π_θ 提出的 N 个动作，不探索策略支撑集之外 [§4.1]。
- 决策时算力可承载每步 N×K×D 量级的模型查询与 Q 前向；像素设置中该条件收紧为仅采样阶段 [§5.3]。
- 评测域为仿真 manipulation（Robomimic/LIBERO），尽管动机指向真实机器人与 VLA fine-tuning [§1, §5]。

## Method

与熟悉基线的对比：

- 旧（best-of-N / OTF / IDQL）：从策略采 N 个候选 -> 用即时 Q(s,a) 挑最大 -> 执行；只看一步。
- 旧（model-based RL）：在模型 rollout 上训练策略/价值或自举训练目标 -> 模型误差进入训练。
- 新（QWM）：训练保持 model-free 不变；决策时用世界模型把每个候选的近期后果展开成树，用 Q 在想象状态上重新打分。

流程：

1. **输入**：当前状态 s_0、策略 π_θ、critic Q_φ、离线预训练的世界模型 M_ψ(s'|s,a)。
2. **建树**：状态节点从 π_θ 采 N 个动作；每个动作节点向 M_ψ 查询 K 次得 K 个预测下一状态；递归至深度 D；末层每条存活路径再采 N_leaf 个动作。规模由剪枝控制：路径按累计折扣分 Σ(b,d)=Σ_{d'} λ^{d'} Q_φ(s_b^{d'}, a_b^{d'}) 排序，每层仅保留 top-J 条 [§4.1, §4.3]。
3. **节点值聚合**：V_Q(d|s_d)=agg_n Q_φ(s_d,a_n)（不依赖模型与深度）；V_r(d|s_d)=agg_n[r_ψ(s_d,a_n)+λ·agg_k V(s^{n,k}_{d+1})]（利用 rollout 但复合误差）；组合 V=½(V_Q+V_r)，可推广为 α 加权 [§4.2, Eq. 5–8]。
4. **根动作分与选择**：Q_ts(s_0,a_n)=½Q_φ(s_0,a_n)+[r_ψ(s_0,a_n)+agg_k λ V(1|s^k_1)]，按 max/softmax 挑动作执行 [§4.2, Eq. 9]。
5. **世界模型实例化**：state-based 用残差确定性动力学 s+Δ_ψ(s,a)（3 层 MLP、hidden 256、demo 上 MSE 预训练 100k 步后冻结）；像素用动作条件化的 Wan2.2-TI2V-5B 视频 diffusion Transformer（3 层 MLP 动作编码器产 action token 与文本 token 拼接作条件，VAE/umT5XXL 冻结，flow-matching 微调 200k 步；推理 1 步去噪，LIBERO 上生成 5 帧 128×128 clip、取第二帧为下一观测做迭代展开）[§4.4, §C.3]。
6. **默认超参**：N=8、K=8、D=4、λ=0.1、J=1、N_leaf=8；表注称 λ 需乘 1/2 才与公式严格对齐 [Table 2]。

## Eval

- **度量**：online success rate 随环境步数的学习曲线（样本效率 + 最终成功率）。
- **基线**：model-free——EXPO、RLPD、IDQL、DSRL、QSM、QAM、FQL；model-based——TD-MPC2、EfficientZero V2（各含 sparse/dense 奖励变体）[§5, §C.4]。
- **环境/数据**：Robomimic 四任务（Lift 用 10-episode 子集、Can 用 MH split、Square/Tool Hang 用 PH split；低维状态 + 7-DoF OSC 动作；稀疏任务完成奖励）；LIBERO 五任务（Task 60/79/29/28/2；RGB + 语言条件）[§C.2]。
- **协议**：5000 步后开始在线训练、offline data ratio 0.5、无额外离线预训练、UTD 20、batch 256、γ=0.99、τ=0.005；EZ-V2 按其官方推荐 UTD 0.4 [§C.1]。
- **消融**：搜索阶段（仅采样 / 仅评估 / 双开）、深度 D、折扣 λ、候选数 N、保留路径数 J；附录 B 与 search-with-V（AWR 式策略）对照 [§5.4, §B]。
- **像素设置**：因算力限制仅采样期搜索、评估期不搜 [§5.3]。

## Weaknesses

1. 公式与实现脱节：Eq.(6)/(9) 以学到的 reward model r_ψ 为组成部分，但 §4.4 明言"we do not learn a reward model on top of the world model"，且未说明 V_r 在缺失 r_ψ 时的退化形式——复现者只能猜测 rollout 分支的实际价值来源 [§4.2 vs §4.4]。
2. 配置自相矛盾：Table 2 标叶层聚合为 Mean，§C.1 正文却称中间层与叶层均用 max；消融最佳 D=2、λ=0.2 与默认 D=4、λ=0.1 不一致且无按任务取舍的说明；表注还要求 λ 乘 1/2 才与公式对齐 [Table 2, §C.1, §5.4]。
3. 默认 J=1 意味着每层剪枝后仅存单条存活路径，“树搜索”实际接近每根动作一条贪心想象 rollout；叠加“对 J 不敏感”的消融，多分支树本身对收益的贡献存疑 [low：我的推断]——提升可能主要来自用想象状态上的 Q 做深度重打分，与 best-of-N 的距离比论文叙事更近 [Table 2, §4.3, §5.4]。
4. 世界模型仅在 demo 上预训练且冻结，不随在线数据更新，也未定量评估其在在线 RL 后期到达的状态分布上的误差；Fig 10 只是 demo 附近的定性对比。“避免复合偏差”只覆盖训练侧，搜索侧的模型偏差（选错动作）从未被度量 [§C.3]。
5. 统计严谨性缺失：正文不报告种子数、方差或置信区间，全部结论以曲线图承载，文本层面不可验证 [§5]。
6. 基线协议混合：model-free 基线共用为 QWM 系设计的高 UTD=20 协议，EZ-V2 却用官方 UTD=0.4；model-based 基线“仅 Lift 非零”的惨败可能部分源于协议失配而非方法本质差距，且无协议敏感性分析 [§C.1, §5.1]。
7. 动机反复强调 real-world robotics 与 VLA fine-tuning，但实验全部在仿真 manipulation 基准上完成，未提供任何真实机器人或 VLA 相关证据，主张与证据之间存在明显跨度 [§1, §5]。
8. 决策时算力开销随 N×K×D 增长：像素设置已因算力被迫放弃评估期搜索；论文未报告每步搜索的墙钟/计算成本，也未讨论在真实机器人控制频率下的可行性 [§5.3]。

## Relations

- **对 best-of-N / OTF（IDQL、EXPO）**：继承“策略提议、Q 裁决”的决策时重打分范式，但把打分对象从单步动作扩展为动作的短视野想象后果树；可视为一步 Q 排序向深度 D 的推广 [§1, §4]。
- **对以想象数据训练的 model-based RL（TD-MPC2、EfficientZero V2 等）**：正面反对“模型进训练”的路线——QWM 训练全程只用真实转移，模型只在决策时使用，因此不产生训练侧复合偏差；代价是训练侧得不到模型带来的数据增广 [§1, §3]。
- **对 MPC / 决策时树搜索**：同属决策时规划，但刻意做成短视野（小 D）+ 激进剪枝（默认 J=1），节点值取“即时 Q 估计”与“rollout 递归估计”的等权平均，而非单一规划值 [§4.1–4.3]。
- **对 AWR 式 search-with-V**：附录对照显示用 V 而非 Q 搜索显著更差，论证了直接在动作分支上查 Q 的优势 [§B]。
- **对世界模型文献**：state-based 侧用残差确定性动力学；像素侧复用视频 diffusion 模型 Wan2.2-TI2V-5B（冻结 VAE/umT5XXL，flow-matching 微调、单步去噪），把生成式视频模型当作动作条件动力学使用 [§4.4, §C.3]。
- **对 VLA / real-robot 方向**：动机上定位为通向真实机器人与 VLA fine-tuning 的一步（决策时用模型、不动训练流程），但该方向尚无实验支撑 [§1, §5]。
- **底座无关性**：在 EXPO 与 RLPD 两个 model-free 底座上均有效，说明搜索层可作为标准 Q-learning 流程之上的可插拔模块 [§5.2]。

## Takeaway

QWM 的核心一句话：**训练保持 model-free，世界模型只出现在决策时**——用冻结的世界模型把策略候选动作的近期后果展开成（剪枝后近乎贪心的）短视野树，再由 Q 函数在想象状态上对候选动作做深度重打分；训练目标、replay buffer、更新规则一概不变。相比 best-of-N（IDQL/EXPO 的 OTF）它看到了动作的近期后果，相比 model-based RL 它不让模型误差进入训练目标，实验上也在 Robomimic 压过一众 model-free SOTA、远好于训不完的 TD-MPC2/EZ-V2。但公式与实现的脱节（reward model 写进公式却不学）、默认 J=1 使“树”接近贪心 rollout、冻结的 demo-only 世界模型从未被定量评估、无种子/方差报告，使“树搜索本身贡献多少”仍存疑。作为“决策时使用世界模型”这条路线的强参考很有价值；复现或引用前应先厘清 §4.2 与 §4.4 的不一致，并自行补齐统计与模型误差的度量。
