# Language Models Need Sleep: Learning to Self-Modify and Consolidate Memories

`paper_arxiv_2606_03979`

## 钉选 · 评论

_2026-07-14T01:57:44.856Z_

下面总结我们围绕论文 **《Language Models Need Sleep: Learning to Self-Modify and Consolidate Memories》** 的讨论，以及进一步推导出的观点。论文核心提出 “Sleep” 范式：让 LLM 像人脑一样，在持续运行过程中周期性进入“睡眠”，进行记忆巩固和自我改进。论文将 Sleep 分成两个阶段：**Memory Consolidation（记忆巩固）** 和 **Dreaming（梦境式自我改进）**。

---

# 一、论文的核心思想

当前 LLM 最大的问题：

> 会用，但不会持续学习。

部署后的模型：

```
Pretrain
   |
   v
Frozen LLM
   |
   +---- 新知识只能存在 context window
```

例如：

用户告诉 GPT：

> “公司的采购流程变了，超过100万需要 CFO 审批。”

当前：

* 当前对话知道；
* 下一次对话忘记；
* 模型参数没有改变。

论文认为：

人类学习不是这样。

人类：

```
短期记忆
   |
睡眠巩固
   |
长期记忆
```

因此 LLM 应该有：

```
Wake:
接收新经验

Sleep:
整理经验
抽象知识
修改自身
```

论文强调 continual learning 不应该再分 train/test，而应该是 wake/sleep 生命周期。

---

# 二、Stage 1：为什么“小模型 distill 到大模型”？

这是我们第一个问题。

直觉上：

传统 distillation：

```
Teacher 大模型
       |
       v
Student 小模型
```

论文反过来：

```
Small self
     |
     v
Large self
```

所以你问：

> 小模型哪里来的？

答案：

不是另一个模型。

而是：

**同一个模型的不同时刻/不同容量状态。**

论文定义：

> smaller version of a model（例如部分参数没有激活）
>
> distill 到 larger version（更多参数激活）。

也就是：

```
同一个模型

状态 A:
10B activated parameters

状态 B:
20B activated parameters
```

睡眠时：

```
fast memory
      |
      v
slow memory
```

---

## 为什么需要这样？

因为作者认为 Transformer 只有两种记忆：

### Attention

短期：

```
context window
```

### MLP

长期：

```
pretraining knowledge
```

中间没有。

论文把 Attention 看成高频更新记忆，把 MLP 看成低频甚至不更新记忆。

所以设计：

```
Fast memory

↓

Medium memory

↓

Slow memory
```

类似：

人的：

```
海马体

↓

大脑皮层
```

---

# 三、Stage 1 的技术实现

关键不是增加一个模型，而是：

## 参数扩展

论文使用：

* MoE
* low-rank expert

提前准备：

```
模型容量:

100B

实际激活:

10B
```

睡眠：

激活新的 expert。

论文提出新增 low-rank expert 存储迁移知识。

流程：

```
新增参数

↓

旧知识迁移

↓

冻结旧参数

↓

训练新参数
```

---

# 四、Stage 2：Dreaming 怎么训练？

第二个问题：

> self-improvement 怎么发生？

流程类似进化：

---

## Step 1

模型自己生成 dream：

例如：

```
我最近学会采购流程

生成：

任务：
解释采购审批流程

答案：
...
```

论文把 dream 定义为：

> self-generated synthetic data。

---

## Step 2

产生候选模型：

```
旧模型 M

+

dream 数据

↓

LoRA/SFT

↓

新模型 M'
```

---

## Step 3

评价：

如果：

```
M' > M
```

接受。

否则：

丢弃。

论文具体采用：

生成 dream → 微调 → 看 downstream performance 是否提升。

---

# 五、我们认为论文最大的问题：Evaluation，而不是 Self-improvement

这是最重要的讨论。

论文解决：

> 如何产生新的自己？

但是没有真正解决：

> 如何知道新的自己更好？

---

## Benchmark 场景：

简单：

数学：

```
AIME score ↑
```

代码：

```
pass rate ↑
```

所以：

```
candidate model

      |

benchmark

      |

better?
```

容易。

---

## 现实 Agent：

困难。

例如客服 Agent：

版本 A：

```
回答快
```

版本 B：

```
回答慢
但客户满意度高
```

哪个更好？

需要考虑：

```
accuracy

+
cost

+
latency

+
risk

+
business outcome
```

---

所以真正瓶颈不是：

```
generate improvement
```

而是：

```
define improvement
```

---

# 六、进一步推导：真正的 Self-improving AI 需要反馈闭环

我们最后联系到了之前讨论的 AI-native / Agent。

真正闭环应该是：

```
             Real World

                 |
                 v

Agent action

                 |
                 v

Outcome

                 |
                 v

Evaluation

                 |
                 v

Model update
```

而不是：

```
LLM

↓

生成数据

↓

训练自己

↓

自己评价自己
```

后者容易：

> AI 自嗨、自我强化。

---

# 七、和 AI-native 企业的联系

这篇论文其实说明：

未来 AI 的核心竞争力不是单纯模型能力，而是：

## 1. 数据闭环

模型需要不断获得真实经验。

---

## 2. Evaluation ownership

谁拥有评价标准？

这是关键。

例如：

企业 Agent：

不是：

“回答像不像 GPT”。

而是：

```
有没有降低成本？

有没有提升转化？

有没有减少风险？
```

---

## 3. Runtime + Feedback Loop

未来最强系统：

不是一个 frozen LLM。

而是：

```
AI system

= 

Foundation Model

+

Memory System

+

Agent Runtime

+

Evaluation System

+

Self-improvement Loop
```

---

# 最终一句话总结

这篇论文的重要意义：

> 它提出了 LLM 从“一次训练、永久冻结的软件模型”向“持续学习、周期性睡眠、自我修改的生命系统”转变的方向。

但是：

> Memory consolidation 已经有比较清晰路径；真正的 self-improvement 最大瓶颈不是生成新知识，而是建立可信的 evaluation 和 feedback ownership。

也就是说：

**未来 AI 竞争的核心，不只是“谁能让模型变聪明”，而是谁拥有定义“变聪明”的闭环。**

---
