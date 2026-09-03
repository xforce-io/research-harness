# Harness-R1: Learning to Edit Executable Runtime Harnesses from Agent Failure Trajectories

`paper_arxiv_2608_02276`

## 钉选 · 评论

_2026-08-05T12:18:58.829Z_

我们刚才围绕 **Harness-R1 这篇论文** 讨论了很多层，我帮你整理成一个结构化总结。

---

## 1. Harness-R1 解决的问题是什么？

核心思想：

> 不直接训练 Agent 本身，而是训练一个“改进 Agent 运行系统”的 Agent。

传统思路：

```
任务
 ↓
LLM Agent
 ↓
工具执行
 ↓
结果
```

如果失败，通常：

* 换更大的模型
* Fine-tune 模型
* 改 prompt

而 Harness-R1 认为：

很多失败不是模型能力不足，而是 **运行机制不好**。

例如：

* context 给得不对
* 工具调用策略不好
* 没有错误恢复
* memory 注入时机不好
* workflow 不合理

所以它优化的是：

```
Agent Runtime / Harness
```

---

# 2. 什么是 Harness？

你提出了一个很关键的问题：

> 广义上 Harness 包含 Agent framework、runtime、数据层、MCP、sandbox 等，那论文到底改什么？

答案：

论文里的 Harness 不是“所有外部系统”。

更准确：

> Harness = Agent 运行时中的可优化策略层。

它主要改：

* hook 逻辑
* context 注入策略
* tool 调用策略
* failure recovery
* retry 策略
* observation 处理方式

例如：

原来：

```
LLM
 ↓
调用工具
 ↓
失败
 ↓
结束
```

优化后：

```
LLM
 ↓
调用工具前检查
 ↓
失败分析
 ↓
换策略
 ↓
重新执行
```

---

# 3. Harness Engineer 到底改什么？

这是我们重点讨论的地方。

你的理解：

> 它是不是有一些 API，然后 Engineer 写代码调用 API？

答案：

基本正确。

架构类似：

```
Harness Runtime

提供可修改 API：

- add_context()
- modify_tool_policy()
- retry()
- validate_action()
- change_memory()

          ↑

Harness Engineer

生成 patch code

          ↑

失败轨迹
```

---

关键区别：

### API ≠ Action Space

API 是积木。

例如：

```
add_context()
retry()
switch_tool()
```

但是动作空间不是：

```
动作1 = add_context
动作2 = retry
```

而是：

> 用这些 API 组合出来的一段程序 patch。

例如：

```python
if tool_failure_rate > threshold:
    add_context("previous failures")
    retry(max_times=3)
```

所以：

动作空间 = 可生成的策略代码。

---

# 4. 它训练过程是什么？

我们最后确认的流程：

不是：

```
边跑边改
```

而是：

一个循环：

---

### Step 1

准备任务 / test cases：

```
Task A
Task B
Task C
...
```

---

### Step 2

用当前 Agent + Harness 执行：

产生：

```
trajectory

观察：
- action
- tool call
- error
- final result
```

---

### Step 3

发现失败：

例如：

```
Agent 总是在搜索阶段死循环
```

---

### Step 4

Harness Engineer 生成 patch：

例如：

```python
if repeated_search > 3:
    change_strategy()
```

---

### Step 5

加载 patch：

新的 Harness：

```
Agent
+
new runtime policy
```

重新跑。

---

### Step 6

根据效果给 reward：

例如：

成功率：

```
50%
 ↓
65%
```

奖励增加。

然后用 GRPO 训练 Harness Engineer。

---

# 5. 为什么不用普通 Coding Agent？

这是我们讨论最有价值的问题。

你的观点：

> 我直接让 GPT/Claude 看失败记录，然后写 patch 不行吗？

答案：

可以。

甚至早期可能更好。

区别：

---

## 方法 A：普通 Coding Agent

每次：

```
失败
 ↓
大模型分析
 ↓
写 patch
 ↓
运行
```

优势：

* 灵活
* 不需要训练
* 可以快速探索

缺点：

* 每次都要重新推理
* 成本高
* 经验主要存在 context 里

---

## 方法 B：Harness Engineer

训练后：

```
失败
 ↓
Engineer
 ↓
patch
```

优势：

经验进入模型参数：

* 更快
* 更便宜
* 对重复失败模式更强

本质：

类似：

> 把超级强 Coding Agent 的能力蒸馏下来。

---

所以更现实路线：

```
阶段1：

GPT-5级 Coding Agent
+
历史记录


阶段2：

积累大量失败-修复数据


阶段3：

训练 Harness Engineer
```

---

# 6. 它需要什么环境？

你提出：

> 是否需要 simulator？

答案：

不一定。

论文：

使用真实 environment：

例如：

* Web
* Database

流程：

```
patch
 ↓
真实执行
 ↓
reward
```

---

但是你提出：

> 如果结合 World Model，可以用 simulator。

这个想法非常重要。

未来可能：

```
World Model Simulator

        ↓

预测 patch效果

        ↓

选择最佳 patch

        ↓

少量真实验证
```

类似：

Agent Runtime 的 RL 可以从：

real-world RL

变成：

model-based RL。

---

# 7. 最大风险：Overfit

你提出：

> 这会不会 overfit？

非常正确。

风险：

Harness Engineer 可能学到：

```
某几个 test case 的 hack
```

而不是：

```
通用 runtime improvement
```

解决：

* 更多任务分布
* unseen test
* 多环境验证
* 限制 patch 空间

---

# 8. 最终我们形成的三层 Agent 架构

这是我觉得最值得保留的抽象：

```
                Learning Layer

          Harness Engineer
          （学习如何改进）


                    ↓


                Strategy Layer

          Harness / Runtime Policy

          - context
          - tools
          - hooks
          - recovery


                    ↓


                Capability Layer

          Foundation Model

          - reasoning
          - knowledge
          - generation
```

---

对应关系：

### 能力层

“模型会不会做”

例如：

GPT-5 能力。

---

### 策略层

“怎么让模型发挥能力”

例如：

什么时候调用工具。

---

### 学习层

“怎么自动改进策略”

例如：

Harness Engineer。

---

# 9. 我认为这篇真正大的意义

不是 Harness-R1 这个具体方法。

而是提出一个方向：

未来 AI 系统可能不是：

```
越来越大的模型
```

而是：

```
越来越会自我改进的运行系统
```

演化路径可能：

```
Agent

↓

Agent learns tasks

↓

Agent improves runtime

↓

Agent builds better Agents
```

Harness-R1 处于第三层：

**让 Agent 学会改造自己的执行系统。**

---

一句话总结：

> Harness-R1 把 Agent 的“自我进化”从训练模型参数，转移到了训练一个能够修改运行策略的工程师 Agent；它通过受限 API 生成 runtime patch，并用环境反馈进行 RL 优化。未来如果结合世界模型，它可能成为一种“模拟环境中自动进化 Agent 架构”的基础。

---
