# Activity Frames: Deterministic Screen-Activity Compilation for Agent Memory and Replay

`paper_arxiv_2608_05784`

## 钉选 · 评论

_2026-08-12T00:57:57.912Z_

我们前面的讨论其实已经从“如何理解这篇论文”走到了一个更重要的问题：

> **如果把 Human Experience 变成 Agent Skill，应该如何设计一个真正可用的经验系统？**

我重新 refine 一下整个框架。

---

# 1. 核心洞察：Experience 不应该只有一种形态

论文提出：

```
Human
  ↓
Experience
  ↓
Deterministic Compiler
  ↓
Executable Memory
  ↓
Agent
```

它解决的是：

> 如何把人的重复操作经验转化为 Agent 可以直接调用的能力。

但是我们讨论后发现：

**单纯把 experience 全部结构化是不够的。**

因为人的经验有两类：

---

## A. 高频、稳定、可重复经验

例如：

财务每天生成日报：

```
登录 ERP
 ↓
查询销售
 ↓
导出数据
 ↓
生成 PPT
 ↓
发送邮件
```

这种经验：

* 重复很多次
* 成功路径稳定
* 结果明确

适合：

```
Experience
 ↓
Compiler
 ↓
Skill
```

形成：

Executable Skill。

---

## B. 低频、复杂、非结构化经验

例如：

能源专家处理异常：

```
天气异常
+
设备状态变化
+
历史经验
+
人工判断
```

这种经验：

不是固定流程。

如果强行 graph 化：

会产生：

* 错误抽象
* 信息丢失
* 巨大 bias

更适合：

```
Raw Experience
 ↓
Retrieval
 ↓
Few-shot Reference
```

让 Agent 参考，而不是照执行。

---

# 2. 所以未来 Skill 不应该只是一个 Graph

我们提出的更完整 Skill：

不是：

```
Skill = Workflow Graph
```

而应该是：

```
Skill
=
{
 Structured Procedure,
 Raw Experience Memory,
 Decision Context,
 Outcome Feedback
}
```

也就是：

一个 Skill 同时包含：

---

## ① Executable Layer（执行层）

负责：

> 怎么做？

例如：

```
Generate Sales Report Skill

Input:
date

Steps:
1. Query ERP
2. Export CSV
3. Transform Data
4. Generate PPT
5. Send Email
```

这是论文 Activity Frame 的核心。

---

## ② Experience Layer（案例层）

负责：

> 以前类似情况怎么处理？

例如：

历史案例：

```
2026-07-15

销售异常下降

原因:
渠道A库存不足

动作:
调整采购策略

结果:
恢复增长
```

Agent 可以检索：

“有没有类似情况？”

---

## ③ Decision Layer（判断层）

负责：

> 为什么这么做？

这是论文缺少的部分。

例如：

```
如果：

负荷增加
+
COP下降
+
室外温度升高

那么：

优先调整冷机组合
```

这里不是 workflow。

而是 decision policy。

---

## ④ Outcome Layer（反馈层）

负责：

> 做完以后效果怎么样？

例如：

```
Action:
调整参数A

Outcome:
节能8%

Confidence:
0.92
```

这是未来 Agent 自我进化关键。

---

# 3. Skill 粒度怎么控制？

这是我们最后讨论的重点。

不要按：

❌ 人

例如：

“张三的经验 Skill”

太大。

不要按：

❌ 单个动作

例如：

“点击按钮 Skill”

太细。

---

更合理：

## Skill = 可复用任务闭环

判断标准：

一个 Skill 应该满足：

### 1. 有明确目标

例如：

```
生成日报
```

而不是：

```
点击 Excel
```

---

### 2. 有输入

例如：

```
日期
区域
产品
```

---

### 3. 有输出

例如：

```
日报文件
优化策略
决策结果
```

---

### 4. 有反馈

例如：

```
成功率
节省成本
准确率
```

---

所以粒度类似：

软件工程里的：

```
function
```

而不是：

```
instruction
```

---

# 4. Skill 如何形成？

不是一次总结。

而应该是生命周期：

```
Raw Experience

        ↓

Observation

        ↓

Pattern Discovery

        ↓

Candidate Skill

        ↓

Human/Agent Validation

        ↓

Production Skill

        ↓

Feedback Update
```

---

例如：

100 次员工做日报：

前10次：

只是 experience。

50次：

发现稳定模式。

100次：

形成：

```
Daily Sales Report Skill
```

---

# 5. Graph 怎么解决规模问题？

你之前提出：

> 参数空间巨大，会不会导致 graph 爆炸？

这是非常关键的问题。

答案：

不要把状态展开。

错误：

```
Query_Sales_East_ProductA_20260812
```

正确：

```
Query Sales Data

parameters:
{
 date,
 region,
 product
}
```

即：

```
Graph
+
Parameter Space
```

分离。

---

进一步：

采用层级 Skill：

```
Business Report Skill

      |
      |
      ↓

Data Query Skill

      |
      ↓

Visualization Skill

      |
      ↓

Email Skill
```

类似软件架构。

---

# 6. 最终我们形成的 Agent Memory 架构

我认为这是我们讨论后的最终版本：

```
                 Human Experience
                        |
        --------------------------------
        |              |               |
        ↓              ↓               ↓

  Raw Memory     Executable Skill   Model Distillation

  (案例)          (流程)              (能力)

        |              |
        ----------------
                |
                ↓

          Agent Runtime

                |
                ↓

             Outcome

                |
                ↓

          New Experience
```

---

# 7. 对论文的最终评价

这篇论文的重要贡献：

不是提出一个完整 Memory 系统。

而是提出：

> **把经验变成可执行资产，而不是文本记录。**

它解决：

```
Experience → Action
```

的问题。

但是未来完整系统需要补充：

```
Experience → Understanding
Experience → Decision
Experience → Improvement
```

---

# 8. 最终一句话总结

我们最后形成的观点：

> **未来 Agent 的 Skill 不应该是一个固定 workflow，而应该是一个“经验资产包”：里面既有经过编译的可执行程序，也有未压缩的历史案例，还有决策依据和结果反馈。**

这样：

* 高频确定任务 → Skill execution
* 非标准复杂任务 → Experience retrieval
* 长期优秀经验 → Model distillation

三者结合，才可能形成真正会成长的企业 Agent。

---
