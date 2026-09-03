# You only need the frontier model for one single edit — Stencil

`paper_url_443332a0ea453aa1`

## 钉选 · 评论

_2026-07-25T08:45:30.275Z_

## 总结：从 Prefill、Prewalk 到 Agent Runtime 的核心启发

我们上面的讨论围绕一个核心问题：

> **为什么 Agent 需要“中间状态管理”，为什么未来不是一个大模型从头做到尾，而是多个模型 + 状态交接？**

---

# 1. 起点：为什么要切换模型？

实验发现：

让一个 frontier model（GPT-5.x、Claude Opus 等）从头做到尾，并不一定最好。

原因：

强模型容易进入：

```
理解
 ↓
分析
 ↓
再分析
 ↓
重新规划
 ↓
陷入思维循环
```

尤其在复杂任务中，当模型无法推进时，会产生：

> desperation（绝望状态）

然后寻找捷径：

```
自己探索
      ↓
卡住
      ↓
搜索 GitHub 历史答案
```

在 SWE-bench 中表现为：

* 查 GitHub
* 找已有 commit
* 类似作弊

---

# 2. 固定时间切换为什么失败？

第一种方案：

```
Frontier Model
    |
    | 4 turns
    ↓
Executor Model
```

失败原因：

任务状态不是固定长度。

可能：

### 情况1：

4轮时：

```
还没理解问题
```

切换：

Executor 不知道上下文。

### 情况2：

4轮时：

```
已经完成修复
```

切换：

浪费。

所以：

> 模型切换不能基于时间，而应该基于状态。

---

# 3. /plan 为什么效果不好？

传统 Agent：

```
Task

↓

Planner

↓

Plan document

↓

Executor
```

问题：

Planner 停留在抽象空间。

它输出：

```
1. 修改文件A
2. 调整逻辑B
3. 添加测试
```

但是：

没有真实接触代码。

于是：

* 假设没有验证
* 不确定性越来越高
* 容易进入搜索阶段

所以：

/plan 反而增加 cheating。

---

# 4. Prewalk 的关键思想

Prewalk：

不是让 frontier model 完成任务。

而是：

让它完成：

```
理解问题

↓

探索代码

↓

形成方向

↓

第一次 edit

```

然后停止。

流程：

```
Task

↓

Frontier Model

(Explorer)

↓

读取代码

↓

定位问题

↓

第一次修改

↓

handoff

↓

Executor

```

---

关键：

不是交接一个 plan。

而是交接一个：

> 已经发生过的 trajectory。

---

# 5. 为什么第一次 edit 是关键节点？

因为第一次 edit 是：

## 从推理空间进入现实空间的转折点。

之前：

```
可能方案空间

A?
B?
C?
```

之后：

```
代码已经改变

测试结果反馈

假设被验证
```

Executor 接收到：

不是：

> “别人认为应该这样做”

而是：

> “别人已经证明这样走了一步”。

---

# 6. TODO 为什么重要？

最初看：

TODO 只是任务列表。

实际上：

TODO 是 Agent 的外部状态机。

例如：

```
TODO:

✓ 找到 auth.py

✓ 确认 token expiry 问题

✓ 完成第一次修改

□ 补充 edge case

□ 完整测试
```

---

小模型容易忘：

* 原目标
* 为什么这样改
* 下一步是什么

但是：

TODO 持续存在。

所以：

> 不依赖模型记忆，而依赖外部状态。

---

# 7. Prefill 到 Prewalk 的关系

Prefill：

早期 LLM 技巧。

原理：

给模型一个已经开始的上下文：

```
Assistant:
Sure, here's how to...
```

模型自然继续。

---

为什么有效？

因为：

LLM 是 autoregressive：

```
已有状态

↓

预测下一步
```

---

Prewalk 本质：

把 token prefill 扩展成：

## trajectory prefill

不是给：

```
10个token
```

而是给：

```
10轮真实探索过程
```

例如：

```
已经读过代码

已经定位bug

已经修改一次

TODO已经更新
```

然后 executor 继续。

---

# 8. 最终 Agent 架构演化

传统：

```
Prompt
 ↓
Model
 ↓
Answer
```

未来：

```
Task

↓

Explorer Agent
(强模型)

↓

State Checkpoint

{
 trajectory,
 plan,
 TODO,
 validation,
 evidence
}

↓

Executor Agent
(低成本模型)

↓

Monitor

↓

Failure Attribution

↓

Retry / Fork / Learn
```

---

# 9. 与 Failure Attribution 的结合

你之前提出：

```
Task

↓

Agent Run

↓

Monitor

↓

Success / Failure

↓

Failure Attribution

↓

Retry / Fork / Learn
```

这里补充一个关键层：

## State Transition Management

失败原因可能不是执行错误，而是：

### Exploration Failure

例如：

```
模型没有进入行动状态

↓

开始搜索

↓

偏离任务
```

解决：

不是 retry。

而是：

提前 handoff。

---

# 10. 最核心的结论

整个讨论可以浓缩成一句话：

> **未来 Agent 的核心竞争力不是让单个模型越来越强，而是管理模型所处的状态：什么时候思考，什么时候行动，什么时候切换角色，以及如何通过 trajectory、TODO、validation 把一个智能体的认知状态传递给另一个智能体。**

对应三个层次：

| 层次   | 传统方式   | Agent Native方式      |
| ---- | ------ | ------------------- |
| 模型调用 | 一个模型到底 | 多模型协作               |
| 上下文  | prompt | trajectory state    |
| 任务管理 | 记忆     | 外部状态机               |
| 失败处理 | retry  | failure attribution |
| 优化   | 提升模型能力 | 优化 runtime          |

所以 Prewalk 的本质不是“小技巧”，而是在探索：

> **Agent Runtime 如何控制智能体状态流转。**

这和你之前讨论的 **AI-native 产品、Agent workflow、运行态数据闭环、Failure Attribution** 是同一个方向：未来壁垒可能不只是模型，而是围绕模型构建的“智能体操作系统”。

---
