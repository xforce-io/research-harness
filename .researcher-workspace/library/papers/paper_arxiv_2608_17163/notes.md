# Q-Learning With World Models

`paper_arxiv_2608_17163`

## 钉选 · 评论

_2026-08-24T00:39:27.411Z_

## 一句话主线

这篇 QWM 论文的核心是：

> 在**真实数据训练的 off-policy actor-critic**之上，额外用世界模型做**决策时的短期 tree search**；世界模型帮助比较动作后果，但不直接用想象数据训练 policy 或 Q-function。

---

## 1. 基础概念

- **Off-policy RL**：可反复利用 replay buffer 中由旧策略采集的数据，因此更省真实交互样本。
- **Q-function / critic**：

  \[
  Q_\phi(s,a)
  \]

  估计“在状态 \(s\) 做动作 \(a\) 的长期回报”。

- **Actor / policy**：

  \[
  \pi_\theta(a\mid s)
  \]

  负责提出或采样动作。

- **Actor-critic**：critic 用真实数据学习“什么动作好”，actor 按 critic 的评分学习“该做什么动作”。

  \[
  \text{真实数据}\rightarrow Q
  \rightarrow \text{改进 policy}
  \rightarrow \text{新真实数据}
  \]

- **世界模型 / dynamics model**：在本文中基本是同义词：

  \[
  M_\psi(s'\mid s,a)
  \]

  即预测“在 \(s\) 采取 \(a\) 后，会到达什么状态 \(s'\)”。

广义上，预测历史能耗曲线是时序 dynamics 预测；但在本文的 model-based RL 语境中，完整世界模型必须是**动作条件化**的，能回答“若改变控制动作，未来会怎样”。

---

## 2. Model-free 与 model-based RL

- **Model-free RL**：直接学 \(Q(s,a)\) 或 \(\pi(a\mid s)\)，不显式预测环境怎样演化。
- **Model-based RL**：学习/使用 dynamics model，然后：
  - 用 imagined rollout 生成合成训练数据；
  - 在 imagined rollout 上直接训练 policy / value；
  - 或在决策时用 MPC、CEM、MPPI、MCTS / tree search 规划。

它与“生成式 vs 判别式”的直觉有点像：

- model-based：先建模环境如何生成未来，再用它决策；
- model-free：直接预测动作或价值。

但它们不是严格一一对应：一个 actor 可以是 diffusion 生成模型，却仍然是 model-free，只要它不预测环境后果。

---

## 3. 为什么已有世界模型 RL 不易扩展到真实机器人

世界模型通常可用历史 \((s,a,s')\) 数据监督训练，不需要由 RL 训练。

困难在于：若用不完美的模型生成长轨迹，再把这些 imagined transitions 用于更新 policy / critic，会出现闭环偏差：

\[
\text{模型预测偏差}
\rightarrow
\text{错误 policy / Q 更新}
\rightarrow
\text{访问更偏的状态分布}
\rightarrow
\text{模型偏差进一步被放大}
\]

任务越长、视觉越复杂、接触动力学越敏感，这个问题越严重。

论文中 “world models’ success has largely been confined to supervised policy learning” 指的是：世界模型在下游上较成熟的用途，多为辅助模仿/监督策略学习；不是说世界模型本身只能用监督学习训练。

---

## 4. QWM 的方法结构

QWM 需要三类组件：

\[
\pi_\theta(a\mid s)
\quad+\quad
Q_\phi(s,a)
\quad+\quad
M_\psi(s'\mid s,a)
\]

```text
当前真实状态 s₀
      ↓
policy 提出 N 个候选动作
      ↓
world model 预测各动作的 K 种后果
      ↓
短期树搜索至深度 D
      ↓
Q-function 评估、剪枝、叶节点截断
      ↓
按 tree-search score 选择根动作
      ↓
只执行这一步，再获得真实状态并重搜
```

- \(N\)：每个状态采样多少候选动作；
- \(K\)：同一动作采样多少可能后继状态；
- \(D\)：向前预演多少决策步。

不剪枝时，树大致按 \((NK)^D\) 增长，所以会用 Q-function 保留最有希望的 \(J\) 条分支。

---

## 5. QWM 的关键创新

普通 Q-learning 选动作近似是：

\[
a^*=\arg\max_a Q_\phi(s,a)
\]

QWM 则给每个根动作构造更丰富的 tree-search value：

\[
Q_{\mathrm{ts}}(s_0,a_0)
=
\frac12
\left[
Q_\phi(s_0,a_0)
+
\left(
r_\psi(s_0,a_0)+\lambda\operatorname{agg}_k V^1(s_1^k)
\right)
\right]
\]

也就是说，QWM 不是用 tree 取代 critic，而是：

- **Q 值**给出直接的长期判断；
- **世界模型 tree**展示该动作具体的短期后果；
- 两者融合后再选动作。

Q-function 在树中有三种作用：

1. 直接给动作打分；
2. 在叶节点充当长期价值截断；
3. 在树扩展时对分支剪枝。

---

## 6. 为什么两种估值要融合

对节点价值，论文使用：

\[
V^d
=
\alpha V_Q^d+(1-\alpha)V_r^d
\]

其中：

\[
V_Q^d=\operatorname{agg}_nQ_\phi(s_d,a_d^n)
\]

是直接依赖 critic 的估值：

- 优点：不随树深度累积模型误差；
- 缺点：不看本次 rollout 的具体后果。

而：

\[
V_r^d
=
\operatorname{agg}_n
\left[
r_\psi(s_d,a_d^n)
+
\lambda\operatorname{agg}_kV^{d+1}(s_{d+1}^{n,k})
\right]
\]

是世界模型递归展开的估值：

- 优点：能利用具体的短期预测；
- 缺点：深度越大，模型误差越累积。

这是一种“可信 critic + 短期前瞻模型”的折中。

---

## 7. 复杂 tree 不一定更好

树越深不是单调更好。其总误差可粗略理解为：

\[
\sum_{d=0}^{D-1}\lambda^d\epsilon_{\text{model},d}
+
\lambda^D\epsilon_Q
\]

- 树浅：主要相信 Q，世界模型误差少；
- 树适中：可看见关键短期后果；
- 树过深：模型误差、分支剪枝误差与计算成本都迅速增加。

真正重要的是 **control-relevant accuracy**：

> 世界模型是否能把候选动作的未来后果和相对优劣排对，而不只是预测 MSE 低或视频看起来逼真。

实用系统应使用短视搜索、不确定性估计、风险惩罚、动作约束和安全回退策略。

---

## 8. EXPO 与 RLPD

论文将 QWM 加在两种 off-policy Q-learning backbone 上。

- **EXPO**
  - 基础策略 \(\pi_{\text{base}}\) 用监督学习获得动作先验；
  - edit policy 只输出小修正：

    \[
    \tilde a=a_{\text{base}}+\hat a
    \]

  - 用 critic 学“如何在合理动作附近做局部改进”。

- **RLPD**
  - 直接训练一个 Gaussian actor；
  - 每次更新等比例抽取离线和在线 replay 数据；
  - 用高 UTD、多 critic ensemble、随机 target 子集和 LayerNorm，提升样本效率并稳定 critic。

二者都是 actor-critic；差别在于 EXPO 通过“基础动作 + edit”保留先验，RLPD 通过“对称离线/在线回放 + 稳定高 UTD”利用先验。

---

## 9. 与 AlphaGo / AlphaZero 的关系

三者都有：

\[
\text{提出动作}
\rightarrow
\text{评估未来}
\rightarrow
\text{选动作}
\rightarrow
\text{收集数据}
\rightarrow
\text{更新}
\]

但 AlphaGo 在决策时使用已知且精确的围棋规则做 MCTS；QWM 面对的是连续动作和不完美的学习型世界模型，因此必须短视、候选受限，并依赖 Q-function 进行价值锚定。

---

## 10. 你的自适应闭环构想

你提出的方向可以整理为：

\[
D_{\text{real}}
\rightarrow
\{M_\psi,Q_\phi,\pi_\theta\}
\rightarrow
\text{搜索策略}
\rightarrow
\text{部署执行}
\rightarrow
D_{\text{real}}
\]

- 世界模型根据真实数据持续校准；
- policy 提候选，搜索策略做短期前瞻；
- Q / V 补足搜索范围外的长期价值；
- 真正执行后只把**真实 transition**写入训练数据；
- 可选择在线搜索直接部署，或把搜索动作蒸馏成低成本部署策略。

QWM 是这个闭环的保守版本：它只让世界模型参与 test-time search，而让 policy 与 critic 始终由真实环境转移训练，以降低模型偏差进入训练闭环的风险。

---
