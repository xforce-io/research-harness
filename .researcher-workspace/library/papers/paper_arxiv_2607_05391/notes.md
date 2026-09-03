# LLM-as-a-Verifier: A General-Purpose Verification Framework

`paper_arxiv_2607_05391`

## 钉选 · 澄清

_2026-07-11T08:33:22.094Z_

LM judge 提升的是 selection 不是 generation

---

## 钉选 · 评论

_2026-07-11T08:42:22.829Z_

我们上面的讨论可以总结为一个核心问题：

> **为什么一个“只是打分的 LM Judge / Verifier”，能够提升 Terminal-Bench 这类 agent benchmark 的准确率？它到底贡献在哪里？**

---

## 1. 首先澄清：Verifier 不提升生成能力，而提升选择能力

LM Judge / Verifier **不是让 agent 写出更好的代码**。

它解决的问题是：

> 当 agent 可以生成多个候选轨迹（trajectory）时，如何从里面挑出最可能成功的那个。

过程：

```
任务
 |
 |---- Agent rollout 1 → 失败
 |
 |---- Agent rollout 2 → 成功
 |
 |---- Agent rollout 3 → 部分正确
 |
 ↓

Verifier 评价每条 trajectory

 ↓

选择最高质量 trajectory

 ↓

最终结果
```

所以提升来自：

```
生成能力不变
+
更好的候选利用率
=
最终 Pass@1 提升
```

而不是：

```
Judge → Agent 智能提升
```

---

# 2. Terminal-Bench 为什么适合这种方法？

因为 terminal agent 的特点：

## 单次生成成功率低

例如：

一个 coding task：

```
理解需求
 ↓
修改代码
 ↓
运行测试
 ↓
发现错误
 ↓
继续修改
 ↓
提交
```

一条 trajectory 很长。

失败可能只是：

* 漏了一个 edge case
* 测试没覆盖
* 某一步判断错误

所以：

```
一次 rollout:
成功率 70%
```

但是：

```
5 次 rollout:

T1 ❌
T2 ❌
T3 ✅
T4 ❌
T5 ❌
```

实际上：

```
Oracle Pass@5 = 100%
```

问题：

> 已经有正确答案，但不知道是哪一个。

这就是 verifier 的价值。

---

# 3. 论文真正的技术点：不要让 LLM 输出分数，而读取概率分布

传统 LM Judge：

Prompt：

```
给这个 trajectory 打 1-5 分
```

模型输出：

```
4
```

但是模型内部其实有：

```
P(1)=0.01
P(2)=0.05
P(3)=0.20
P(4)=0.35
P(5)=0.39
```

传统方法：

取最大概率：

```
argmax(P)

=5
```

问题：

丢失大量信息。

---

论文方法：

保留完整分布：

[
score=\sum_i P(i)\times i
]

例如：

```
0.01×1
+0.05×2
+0.20×3
+0.35×4
+0.39×5

=4.06
```

得到连续分数：

```
trajectory score = 4.06
```

---

所以你的判断：

> “这不就是期望吗？”

完全正确。

数学上就是：

**概率分布的期望。**

---

# 4. 那创新在哪里？

不是数学创新。

不是提出 expectation。

而是提出：

> LLM judge 应该被看作一个概率模型，而不是一个离散分类器。

传统：

```
LLM
 |
输出一个数字
 |
排序
```

论文：

```
LLM
 |
读取 token probability distribution
 |
连续 reward
 |
排序
```

本质类似：

机器学习里面：

分类：

```
cat 0.51
dog 0.49
```

不会只看：

```
cat
```

而会利用概率。

论文认为：

LLM judge 以前浪费了这个信息。

---

# 5. 为什么连续分数有效？

因为离散分数容易产生 tie。

例如：

真实质量：

```
A = 4.51
B = 4.49
```

离散：

```
A → 5
B → 5
```

无法区分。

连续：

```
A = 4.51
B = 4.49
```

可以排序。

论文结果：

```
传统 LM Judge:
tie rate ≈ 27%

LLM-as-a-Verifier:
tie rate = 0
```

---

# 6. 论文真正的大思想：Verification 成为 Agent Scaling 维度

传统 AI scaling：

```
更大模型
更多参数
更多训练数据
```

论文提出另一条：

```
更多候选生成
+
更强验证
+
搜索
=
更强 agent
```

类似 AlphaGo：

AlphaGo：

```
Policy network
生成棋步

Value network
评价棋步
```

Agent：

```
LLM
生成 trajectory

Verifier
评价 trajectory
```

---

# 7. 但是论文也有值得怀疑的地方

## （1）提升部分来自 candidate pool

比如：

如果：

```
单 agent:
76%
```

多个 agent：

```
Oracle:
84%
```

那么 verifier 的贡献只是：

> 找到已有的好答案。

不是创造能力。

---

## （2）依赖 logprob

方法需要：

```
token probability
```

但是很多 frontier API 不提供。

如果只能：

```
让模型生成数字
```

优势消失。

---

## （3）Verifier 是否最终会被 agent 自己吸收？

未来 agent 可能：

```
生成
 ↓
测试
 ↓
反思
 ↓
修改
```

自己形成闭环。

那么：

独立 verifier 是否必要？

这是开放问题。

---

# 最终一句话总结

这篇论文的核心不是：

> “提出了一个复杂的新算法。”

而是：

> **发现 LLM 本身已经包含一个隐式价值判断能力，以前 LM Judge 只取 argmax 输出浪费了概率信息；通过把离散评分变成连续概率期望，可以构造更好的 verifier，从而让 agent 在多个候选轨迹中更准确地选择正确答案。**

从 agent 架构角度看，它最大的意义是：

> **未来 agent 的能力可能不仅来自生成模型，而来自 Generation + Verification + Search 的闭环。**

这和你之前一直讨论的 **harness、eval、control loop、verification gate** 是同一个方向。

---

## 澄清

_2026-07-11T08:43:15.316Z_

**Selection** not `generation`

- multi-candidate
- not smarter agent

---
