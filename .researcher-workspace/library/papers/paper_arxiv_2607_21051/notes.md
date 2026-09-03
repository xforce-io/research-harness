# Sample-Efficient Learning from Agent Experience

`paper_arxiv_2607_21051`

## 钉选 · 评论

_2026-07-28T07:13:39.640Z_

# Molt vs Experience Distillation：Agent 学习闭环总结

两篇论文放在一起看，核心讨论的是：

> **未来 Agent 如何从“执行任务”走向“持续成长”。**

其中：

* **Molt：解决 Agent 如何通过大量交互学习（Learning by Doing）**
* **Experience Distillation：解决 Agent 如何把经历转化为长期能力（Learning from Experience）**

二者不是替代关系，而是 Agent 自我进化闭环中的两个阶段。

---

# 1. Molt：Agent RL 训练基础设施

## 核心定位

Molt 不是简单的 experience 生成器，而是：

> **一个 Agentic Reinforcement Learning 训练框架。**

它负责完整训练流程：

```
Environment

↓

Agent rollout

↓

Trajectory

↓

Reward / Advantage

↓

RL Optimization

↓

Updated Agent
```

也就是说：

Molt 会：

* 运行 Agent
* 与环境交互
* 收集 trajectory
* 计算 reward
* 进行 RL 更新
* 得到更强的 Agent

---

## Molt 解决的问题

传统 Agent RL 最大的问题：

### 1. 环境交互成本高

例如：

* Coding Agent
* Robot Agent
* 企业流程 Agent

每一次失败都很贵。

### 2. Agent RL 工程复杂

需要处理：

* 大规模 rollout
* 并行环境
* trajectory storage
* policy update
* distributed training

Molt 类似：

> Agent RL 时代的 PyTorch。

---

# 2. Experience Distillation：经验如何变成能力

## 核心问题

Agent 已经经历很多事情：

```
Agent Run

↓

Trajectory

↓

成功经验 / 失败经验
```

问题：

这些经验怎么办？

传统：

存 memory：

```
Memory

↓

Retrieval

↓

Context
```

但是：

经验没有进入模型。

---

Experience Distillation：

目标：

```
Experience

↓

Distillation

↓

Model weights

↓

Permanent capability
```

即：

> 把一次经历压缩成模型能力。

---

# 3. 两者最大的区别

|        | Molt      | Experience Distillation |
| ------ | --------- | ----------------------- |
| 解决问题   | 如何学习      | 如何吸收经验                  |
| 学习来源   | 环境交互      | 已有 trajectory           |
| 是否需要环境 | 需要        | 不需要                     |
| 核心方法   | RL        | 蒸馏/模仿/监督学习              |
| 输出     | 更优 policy | 长期能力                    |
| 类似人类   | 练习        | 复盘总结                    |

---

# 4. Experience Distillation 是监督学习吗？

答案：

## 表面上：是

最简单形式：

```
state → action
```

类似：

Behavior Cloning / SFT。

例如：

```
当前状态：

发现空指针


专家动作：

检查初始化代码
```

训练：

```
P(action | state)
```

---

但是：

## 本质上不是普通 SFT

原因：

普通 SFT：

```
问题 → 答案
```

而 Agent experience：

```
Observation

↓

Reasoning

↓

Action

↓

Feedback

↓

Correction

↓

Outcome
```

包含：

* 决策过程
* 探索过程
* 错误修正
* 结果反馈

所以更接近：

* Policy Distillation
* Behavior Cloning
* Preference Learning
* Outcome-based learning

的组合。

---

# 5. Experience Distillation 的训练形式

可能包含：

## 方式 1：Trajectory imitation

学习：

```
state → action
```

类似 SFT。

---

## 方式 2：Preference Distillation

比较：

```
成功 trajectory

>

失败 trajectory
```

训练：

DPO / preference learning。

---

## 方式 3：Outcome-conditioned learning

学习：

```
状态

+

目标结果

↓

行动策略
```

例如：

```
目标：
降低能耗10%

当前设备状态：

↓

调整策略
```

---

# 6. 两者结合后的完整 Agent 学习闭环

未来 Agent 可能：

```
                 Environment

                      ↓

                 Agent Run

                      ↓

                 Experience

                      ↓

        ┌─────────────┴─────────────┐

        ↓                           ↓

   Molt / Agent RL          Experience Distillation

        ↓                           ↓

  探索新的策略                固化已有经验


        └─────────────┬─────────────┘

                      ↓

              Better Agent
```

---

# 7. 类比人的学习

## Molt = 练习

比如学习打篮球：

```
投篮

↓

失败

↓

调整

↓

继续投
```

通过大量尝试发现规律。

---

## Experience Distillation = 复盘

比赛结束：

```
分析：

为什么这个球成功？

为什么那个球失败？

↓

形成习惯
```

经验进入能力。

---

# 8. 和数字分身 / 角色模型的关系

这与你之前讨论的“角色模型 Adapter”高度相关。

未来：

一个专业 Agent：

不是：

```
基础模型

+

知识库

+

Workflow
```

而是：

```
Base Model

+

Role Adapter
```

这个 Adapter 来自：

```
专家执行过程

↓

Agent trajectory

↓

Experience Distillation

↓

角色能力
```

---

例如：

能源专家 Agent：

上线后：

执行：

```
10000 次能源优化任务
```

产生：

```
状态

↓

判断

↓

策略

↓

结果
```

然后：

蒸馏：

```
能源专家 Adapter
```

以后：

新的 Agent 加载：

```
GPT

+

能源专家能力
```

---

# 9. Molt 和 Experience Distillation 在企业 AI 中的分工

可以对应：

## Molt：

负责：

> 探索能力

回答：

“还有没有更好的策略？”

例如：

* 新方案探索
* 新 workflow 探索
* 新决策策略探索

---

## Experience Distillation：

负责：

> 能力沉淀

回答：

“已经验证过的经验如何变成组织能力？”

例如：

* 专家经验
* 最佳实践
* 异常处理经验

---

# 10. 最终形成 AI Native 产品闭环

未来企业 Agent：

```
业务运行

↓

Agent 执行

↓

产生 Experience

↓

评价 / Reward

↓

RL 探索优化

↓

Experience Distillation

↓

角色能力升级

↓

下一轮业务运行
```

---

## 最核心结论

两篇论文共同说明：

> Agent 的未来竞争力，不只是模型大小，而是“经历 → 学习 → 能力沉淀”的闭环能力。

其中：

* **Molt 让 Agent 会通过经历成长**
* **Experience Distillation 让经历真正成为能力**

如果对应你之前提出的：

```
Agent Run
 ↓
Monitor
 ↓
Failure Attribution
 ↓
Retry / Fork / Learn
```

那么：

* Molt 更偏 **Learn 的探索侧**
* Experience Distillation 更偏 **Learn 的内化侧**

二者结合，才接近真正的“会成长的数字分身”。

---
