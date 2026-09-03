# SKILL.state: Scalable Long-Horizon Agent Skills

`paper_arxiv_2608_26263`

## 钉选 · 评论

_2026-09-01T13:06:02.968Z_

可以。上面这整段讨论，真正有价值的地方，不只是“Prime Agent、SKILL.state、Recuris 三篇工作怎么比较”，而是我们其实推导出了一个更大的判断：

> **Agent 架构正在经历一条非常清晰的“状态空间逐层收缩”的演进路径：从自然语言上下文，到通用可编程 Runtime，再到显式、结构化、可验证的 Execution State。**

我把整个讨论收敛成一套完整框架。

---

# 一、起点：传统 Agent 的根本问题，不只是 context 太长，而是“状态表达太自由”

传统 Agent 的主要执行载体，本质上还是：

```text
System Prompt
+ Conversation History
+ Tool Results
+ Reasoning
+ Latest Observation
```

所有东西都放在自然语言里。

这意味着它的“状态空间”几乎是无限的：

```text
自然语言
≈ 任意描述
≈ 任意顺序
≈ 任意粒度
≈ 任意重复
≈ 任意歧义
```

Agent 每走一步，都要从一大堆自然语言里重新推断：

> 我现在到底在哪里？
> 哪些已经做完？
> 哪些事实仍然有效？
> 哪些只是之前的猜测？
> 下一步该做什么？

所以传统 Agent 的一个深层问题可以理解为：

> **状态并没有被真正表示出来，只是隐含在历史文本里。**

这其实是比“token 太多”更根本的问题。

---

# 二、Prime Agent 已经做了第一次非常重要的“状态空间压缩”

Prime Agent 相比普通 Agent，已经向前迈了一大步。

它没有让模型直接面对几十个、几百个 tool schema，而是把一个 **Persistent Python/IPython Runtime** 放在中间：

```text
LLM
 ↓
Python Runtime
 ↓
CLI / API / MCP / Files / Shell / Subagents
```

所以原本纯自然语言表达的很多东西开始变成：

```python
files
results
df
repo
agents
config
test_result
```

也就是说，Prime Agent 把一部分世界从：

```text
“语言空间”
```

压缩到了：

```text
“程序空间”
```

这是很关键的一步。

---

# 三、为什么说 Python Runtime 本身已经是一种约束？

因为 Python 并不是任意自然语言。

自然语言可能写成：

```text
我觉得昨天那个测试可能差不多已经好了，
但是之前有个问题，
好像还要看看另外那个 dependency，
不过大概问题不大……
```

而程序表达至少会收敛成：

```python
test_status = "passed"
dependency_issue = True
```

所以 Python 已经带来了：

* 变量；
* 类型；
  -函数；
* 对象；
* 数据结构；
* 控制流；
* 可执行语义。

换句话说：

> **Prime Agent 把 Agent 从“纯语言智能体”推进成了“运行在可编程计算环境中的智能体”。**

这就是第一次状态空间收缩。

---

# 四、但是 Prime Agent 的问题在于：Python 的状态空间依然巨大

这里是我们讨论中很重要的一个洞察。

Persistent Python Runtime 虽然比纯自然语言好很多，但它仍然非常自由。

运行几天以后，可能出现：

```python
result
result2
result_new
tmp
df
df_old
analysis
analysis2
config
state
x
foo
bar
```

那么问题就来了：

> 哪个变量是真正的 current truth？

这和 Jupyter Notebook 很像。

Notebook 最大的问题之一就是：

> **执行状态存在，但它可能是隐式的、分散的、不可控的。**

所以：

> **Python State ≠ Canonical Execution State。**

Python 给了 Agent 一个巨大的、通用的状态空间。

但它没有告诉 Agent：

> 哪几个变量才真正代表“任务当前状态”。

---

# 五、SKILL.state / Recuris 做的是第二次状态空间压缩

这就是刚才你抓到的关键。

如果说 Prime Agent 的抽象是：

> **给 Agent 一个通用 Runtime。**

那么 SKILL.state / Recuris 往上再做了一层：

> **在通用 Runtime 之上，定义一个任务/领域相关的、有限的、显式的 State Space。**

比如 Python 世界可能有几百个变量：

```text
Python Runtime
────────────────────────────
df1
df2
logs
repo
result
tmp
commands
files
agents
errors
...
```

但是当前任务真正重要的可能只有：

```json
{
  "goal": "deploy v3.2",
  "build_status": "passed",
  "test_status": "passed",
  "security_review": "pending",
  "blockers": ["CVE-xxx"],
  "deployed": false
}
```

这其实是在做：

```text
巨大计算状态空间
        ↓
任务相关状态空间
```

也就是第二次压缩。

---

# 六、所以可以把 Agent 演进理解成三级状态空间

这是我们讨论里最值得保留的模型。

## Level 1：Natural Language State

传统 Agent：

```text
Conversation
History
Reasoning
Observation
Tool output
```

特点：

> **状态完全隐含在语言里。**

状态空间最大，最自由，也最容易漂移。

---

## Level 2：Computational State

Prime Agent：

```text
Python runtime
variables
objects
files
functions
subagents
```

特点：

> **状态开始进入程序化世界。**

比自然语言收敛很多，而且可操作、可组合、可计算。

但仍然非常自由。

---

## Level 3：Canonical Execution State

SKILL.state / Recuris：

```json
{
  "phase": "...",
  "completed": [...],
  "pending": [...],
  "blockers": [...],
  "verified_facts": [...]
}
```

特点：

> **只有对未来执行真正重要的信息，才允许进入 State。**

状态空间进一步大幅缩小。

---

因此可以画成：

```text
Natural Language
巨大、模糊、非结构化
        │
        ▼
General Runtime State
Python / code / variables
        │
        ▼
Canonical Execution State
有限、结构化、可验证
```

本质上就是：

> **不断压缩 Agent 决策所需要面对的自由度。**

---

# 七、为什么“状态空间收缩”会让 Agent 更可靠？

这其实可以类比机器学习。

假设一个 Agent 下一步决策是：

$$
a_t = f(x_t)
$$

如果 \(x_t\) 是整个 conversation：

```text
100k tokens
```

模型需要从巨大的输入空间里寻找真正有用的信息。

而如果：

$$
x_t = \text{canonical state}
$$

只有：

```text
20 个字段
```

那么 Agent 的问题从：

> “从所有历史里推断当前世界”

变成：

> “基于当前世界做决策”。

这是完全不同的难度。

因此 State 本质上是在做：

> **Representation Learning 的人工结构化版本。**

我们人为设计一个更好的 latent representation：

$$
History \rightarrow State
$$

使未来决策更简单：

$$
State \rightarrow Action
$$

---

# 八、这也解释了为什么 SKILL.state 不只是“节省 Token”

表面上看，它的结果是：

```text
context 变短
token 下降
```

但更本质的变化其实是：

> **把“历史文本”转换成了“当前世界表示”。**

传统 Agent：

```text
History
↓
LLM 每次重新理解
↓
Current State
↓
Action
```

State-centric Agent：

```text
History / Observation
↓
State Update
↓
Current State
↓
Action
```

所以最核心的变化是：

> **State inference 从每一步重复执行，变成了增量维护。**

这和数据库的思想其实非常接近。

---

# 九、因此 Event Log 和 State 必须分开

这也是前面讨论中另一个重要结论。

State-centric 并不意味着：

> 历史数据真的应该被删掉。

更合理的架构是：

```text
Event Store
append-only
完整历史
      │
      ▼
Materialized State
current truth
      │
      ▼
LLM
```

也就是：

### Event

回答：

> 过去发生过什么？

### State

回答：

> 现在什么是真的？

这其实很像：

> **Event Sourcing + Materialized View / CQRS**

Agent 正常执行只需要 State。

需要：

* 审计；
* debug；
* provenance；
* 回放；

才重新访问 Event Log。

---

# 十、这也解释了 Prime Agent 和 SKILL.state 并不冲突

它们其实位于不同抽象层。

Prime Agent 解决：

> **Agent 怎么运行？**

包括：

* daemon；
* session；
* persistent kernel；
* tool execution；
* code execution；
* recursive agents；
* scheduling；
* recovery。

这是：

> **General-purpose Agent Runtime**

而 SKILL.state 解决：

> **这个 Agent 运行过程中，什么才应该成为 canonical current state？**

这是：

> **Task / Domain Runtime Semantics**

所以你说：

> “state 相当于是在 Prime Agent 的通用 Runtime 上再包了一层更垂类的 Runtime。”

我觉得这个描述是很准确的。

可以画成：

```text
Infrastructure
────────────────────────

Prime Agent Runtime
daemon / process / Python / tools / agents

        ↓

Domain / Skill Runtime
state schema
state transition
validation
completion criteria

        ↓

Specific Task Instance
current state
```

---

# 十一、所以真正的 Skill 以后可能也会变化

现在很多 Agent Skill 是：

```text
SKILL.md
+
instructions
+
examples
+
tool description
```

本质上还是：

> Prompt Package。

但如果接受今天的 state-centric 思路，一个成熟的 Skill 应该更像：

```text
Skill
├── Procedure
├── State Schema
├── Actions
├── Tool Bindings
├── State Transitions
├── Invariants
├── Evaluator
└── Completion Criteria
```

也就是说：

> **Skill 从“知识文档”逐渐变成“小型 Runtime”。**

例如：

```text
DeploySkill
```

不是只告诉 Agent：

> 如何部署。

而是定义：

```text
state:
  build_status
  test_status
  security_status
  deployment_status
  rollback_status
```

以及：

```text
什么状态允许 deploy
什么状态必须 rollback
什么状态算 complete
```

这已经很接近：

> **Agent-native executable module**

而不是 markdown。

---

# 十二、Recuris 又在这个基础上增加了另一层：Harness Evolution

SKILL.state 更关注：

> 如何正确维护当前状态。

Recuris 更进一步：

> 当前 State + 执行 Evidence 能不能反过来改变未来 Skill？

于是形成：

```text
State
 ↓
Skill
 ↓
Action
 ↓
Evidence
 ↓
Evaluation
 ↓
Skill Update
 ↓
下一次执行
```

这意味着 Agent 不仅：

> 在 State Space 里运行，

而且还会：

> 修改产生行为的 Runtime / Skill 本身。

所以 Recuris 更接近：

> **Adaptive Domain Runtime**

---

# 十三、Prime Agent + SKILL.state + Recuris，其实拼出了一个很完整的架构

可以把三个工作放在一起：

```text
┌──────────────────────────────┐
│      Continual Harness       │
│ Skill / policy evolution     │
│          Recuris             │
├──────────────────────────────┤
│ Canonical Execution State    │
│ state / invariant / progress │
│       SKILL.state            │
├──────────────────────────────┤
│ General Agent Runtime        │
│ Python / tools / agents      │
│        Prime Agent           │
├──────────────────────────────┤
│ External Systems             │
│ CLI / MCP / APIs / DB        │
└──────────────────────────────┘
```

这是一个很漂亮的分层。

---

# 十四、如果用操作系统类比，会更加清楚

Prime Agent 很像：

> **Operating System / VM**

提供：

```text
process
memory
runtime
IPC
scheduler
I/O
```

SKILL.state 更像：

> **Application State Model**

定义：

```text
当前应用处于什么状态
哪些 transition 合法
```

Recuris 则有点像：

> **程序可以根据运行 telemetry 修改自己的 policy / code**

所以它们并不在同一个层级竞争。

而是：

```text
Prime Agent
= General Compute Runtime

SKILL.state
= Semantic State Runtime

Recuris
= Adaptive Runtime
```

---

# 十五、从这个角度重新理解“Agent Memory”，也会完全不同

以前经常把 Agent Memory 简单分为：

```text
short-term memory
long-term memory
```

这个分类其实太粗。

更好的分类可能是：

### 1. Computational Working State

Python：

```text
变量
数据
中间计算
```

### 2. Canonical Execution State

```text
任务进度
当前世界
blocker
commitment
```

### 3. Episodic Memory

```text
完整历史
事件
tool outputs
```

### 4. Semantic / Skill Memory

```text
经验
规律
procedure
policy
```

这四类东西过去经常全部被塞进：

```text
Messages[]
```

而下一代 Agent Runtime 很可能会把它们彻底拆开。

---

# 十六、这里其实还能得出一个很有意思的结论：越智能，反而越需要结构化约束

乍一看好像：

> 模型能力越强，就越不需要人为结构。

但 long-horizon Agent 恰恰可能相反。

模型越强：

```text
可以处理的动作更多
可以调用的工具更多
可以创建的变量更多
可以探索的路径更多
```

于是：

> **可能状态空间也指数级变大。**

因此：

> 更强的模型，不一定意味着更少的 Runtime。

反而意味着：

> **更需要好的 Runtime 去约束自由度。**

这和人类组织其实很像。

能力越大的公司，不是规则越少。

反而：

* accounting；
* ERP；
* workflow；
* governance；
* audit；

会越来越强。

因为：

> **能力扩大以后，治理状态空间本身成为问题。**

---

# 十七、因此 Agent 未来真正的竞争点，可能不是 Context Window

这也是我们这轮讨论最后可以抽象出的一个判断。

过去行业经常认为：

```text
Long Horizon
=
更大 Context Window
```

但这些工作共同指向：

```text
Long Horizon
≈
Better State Abstraction
+
Better Runtime
+
Better State Transition
+
Better Evaluation
```

换句话说：

> **解决长期 Agent 的关键，不是让模型记住更多，而是让模型需要记住得更少。**

这句话我觉得可以作为整个讨论的核心。

---

# 十八、最终可以把这条演进路线总结成四代

### 第一代：Prompt Agent

```text
LLM + Prompt
```

核心能力：

> reasoning。

---

### 第二代：Tool Agent

```text
LLM + Tools + Conversation
```

核心能力：

> action。

---

### 第三代：Runtime Agent

Prime Agent：

```text
LLM + Programmable Runtime
```

核心能力：

> computation / orchestration / recursion。

---

### 第四代：State-centric Agent

SKILL.state / Recuris：

```text
LLM
+
Runtime
+
Canonical State
+
Skill
+
Evaluation
```

核心能力：

> **long-horizon reliable execution。**

再向前一步可能就是：

### 第五代：Adaptive Runtime

```text
execution
→ evidence
→ evaluation
→ state / skill / harness evolution
```

也就是：

> **Agent Runtime 自身开始持续进化。**

---

# 最后一句话总结整个讨论

我觉得可以压缩成一句非常重要的判断：

> **Agent 架构的演进，本质上是在不断把模型从一个“无限自由的语言状态空间”里解放出来：先用代码 Runtime 收缩一次，再用 Canonical State 收缩一次，最终把 LLM 的能力集中在真正需要智能的“状态转移决策”上。**

所以真正理想的 Agent 可能不是：

```text
一个拥有巨大 context 的聪明模型
```

而是：

```text
一个非常强的模型
+
一个非常小、非常明确的当前状态
+
一个非常强的通用 Runtime
+
一个持续进化的 Skill/Harness
```

而其中最值得我们继续关注的，可能正是这个变化：

> **Agent engineering 正从“Prompt Engineering”走向“Runtime Engineering”，然后进一步走向“State Engineering”。**

---
