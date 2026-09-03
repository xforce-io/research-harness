# The Rise of Verbal Reinforcement Learning

`paper_arxiv_2609_01597`

## 钉选 · 评论

_2026-09-03T04:28:14.976Z_

这篇《**The Rise of Verbal Reinforcement Learning**》刚在 9 月 1 日提交到 arXiv。它不是提出一个新算法，而是一篇**试图重新整理 Agent 学习范式的综述/观点论文**。作者把 Reflexion、自我批评、tool feedback、memory、RLHF/DPO、process reward、reward code generation 等过去看起来分散的技术，统一放进一个概念：

> **Verbal Reinforcement Learning，语言强化学习 / 言语强化学习（VRL）**。

我觉得这篇文章**分类本身不算特别惊艳，但它抓到了一个正在发生的非常重要的结构变化**：

> **Agent 的 feedback 正在从低带宽的“标量 reward”，变成高带宽、结构化、可解释的“语言 feedback”。**

而且这件事情和你最近一直在思考的 **Agent Harness、runtime、state、feedback loop、自我进化**，其实是同一件事情的不同侧面。([arXiv][1])

---

# 一、先用一句话理解这篇论文

经典强化学习大致是：

$$
state \rightarrow action \rightarrow reward\;(scalar) \rightarrow policy\ update
$$

比如：

> 做对了：+1
> 做错了：-1

但对于 LLM Agent，这个反馈完全可以是：

> “你的第三步错了。这个 API 返回 404 不是因为 URL 拼错，而是因为当前用户没有权限。先检查 access token，再决定是否重试。”

这句话携带的信息量显然远远超过：

> reward = -1

所以作者认为：

**自然语言本身正在成为 Agent 最重要的 reinforcement channel。**

而且它不一定非要通过梯度更新才能叫 reinforcement。

语言可以在三个时间尺度发生作用：

**定义问题 → runtime 中纠错 → 训练模型。**

这就是整篇文章的三大 Pillar。([arXiv][2])

---

# 二、论文最核心的那张图

作者把 VRL 分成三类：

| 阶段                        | Language 起什么作用 | 改变什么                           | 是否改模型权重 |
| ------------------------- | -------------- | ------------------------------ | ------- |
| **Pillar 1 Grounding**    | 定义世界           | state / action / goal / reward | ❌       |
| **Pillar 2 Deliberation** | runtime 反馈与纠错  | 当前 trajectory / reasoning      | ❌       |
| **Pillar 3 Learning**     | 学习信号           | data / model weights           | ✅       |

作者特别强调一个分类轴：

> **language 是“什么时候”进入 Agent lifecycle 的？它改变了什么？**

这比按照“human feedback / AI feedback / tool feedback”分类更有意义。([arXiv][2])

论文 Figure 2 本质上就是：

**Problem definition → Inference → Training**

对应：

**Grounding → Deliberation → Learning**



这个分类，我觉得值得记住。

---

# 三、Pillar 1：Language as Grounding Signal

这一部分很容易被低估。

作者说，自然语言首先不是“告诉 Agent 对错”，而是在**定义 Agent 所处的世界**。

经典 MDP：

$$
M = (S,A,P,R)
$$

Agent 必须知道：

* State 是什么；
* Action 是什么；
* Goal 是什么；
* Reward 是什么。

而今天很多 Agent 系统实际上是通过自然语言定义这些东西。

作者进一步分为四种。

### 1. Goal Grounding

例如：

> “修复 GitHub issue #1327，并确保所有 unit tests pass。”

这句话定义了目标。

LLM 需要把它转换成：

```text
Issue description
      ↓
sub-goal 1
sub-goal 2
sub-goal 3
      ↓
success condition
```

真正困难的不是“理解英语”，而是：

> **语言如何映射到一个可执行、可验证的 goal space。**

作者把这个称为 **grounding gap**。([arXiv][2])

---

### 2. State Grounding

这一点和你前两天讨论的 **State / Prime Agent / runtime** 特别相关。

Agent 所谓的 state 经常已经不是原始环境 state，而是：

```text
环境
↓
tool / parser / runtime
↓
语言化 state representation
↓
LLM
```

例如：

真实状态可能是：

```json
{
  "process_exit_code": 1,
  "stderr": "...",
  "changed_files": [...],
  "git_status": ...
}
```

Runtime 最后告诉模型：

> Test failed because `foo.py` line 37 raises TypeError.

这其实是：

> **把巨大环境 state 压缩成 verbal state。**

所以 language 是一种 **state representation / state abstraction**。([arXiv][2])

你之前说：

> Prime Agent 用 Python/runtime 把纯自然语言 Agent 的巨大状态空间压缩了一层；
> Stateful runtime 又进一步约束 state。

这篇文章从另外一个角度证明：

> **决定 Agent 能力的不只是模型如何 reasoning，更是环境怎样被 ground 成模型能够消费的 state。**

两者是高度一致的。

---

### 3. Action Grounding

模型说：

> “重新启动服务。”

这不是 action。

真正 action 是：

```bash
systemctl restart nginx
```

或者：

```python
restart_service("nginx")
```

因此语言必须经历：

$$
language\ intent
\rightarrow skill/tool
\rightarrow executable\ action
$$

这里最关键的问题是 **granularity alignment**。

比如：

> “把咖啡做好”

可能对应：

* 一个高级 skill；
* 10 个 subgoal；
* 100 个 motor actions。

Agent runtime 必须决定：

> **语言在哪个 abstraction level 和 action space 对齐。**

([arXiv][2])

这其实就是你最近反复讨论的：

> Skill / MCP / CLI / Code execution 到底应该暴露在哪一个抽象层。

---

### 4. Reward Code Generation

这一类比较有意思。

过去：

```python
reward = manually_written_function(state)
```

现在可以：

```text
Natural language goal
        ↓
LLM
        ↓
reward function code
```

例如 Eureka、Text2Reward 等工作。

所以 Language 不只是 reward：

> language → **生成 reward function**。

也就是说，人不再必须手工 specification reward。([arXiv][2])

---

# 四、Pillar 2：Language as Deliberative Feedback

这一块就是今天绝大多数 Agent Harness 真正在做的事情。

Agent：

```text
Reason
  ↓
Act
  ↓
Environment feedback
  ↓
Critique
  ↓
Revise
  ↓
Act again
```

但是：

> **模型参数没变。**

所以这是 **inference-time learning / test-time adaptation**，而不是 parameter learning。([arXiv][2])

作者分了五类。

---

## ① Self-Critique

例如 Reflexion：

```text
Generate
↓
Critique yourself
↓
Revise
```

问题也很明显：

> **自己不知道自己不知道什么。**

如果 generator 和 critic 是同一个模型：

```text
Generator blind spot
       ↓
Critic same blind spot
       ↓
confident wrong correction
```

作者认为 self-correction 最大的问题就是这种 **circularity**。

已有研究也表明，独立 critic 往往比自己 critique 自己更有效。([arXiv][2])

---

# 五、这里有一个非常重要的 Agent 设计原则

因此真正可靠的 Agent 不应该主要依赖：

> LLM：“我觉得自己写对了。”

而应该依赖：

```text
LLM
 ↓
真实环境
 ↓
可验证 feedback
 ↓
LLM
```

比如 coding agent：

```text
write code
 ↓
compiler
 ↓
error trace
 ↓
unit tests
 ↓
test failure
 ↓
LLM repair
```

论文把这一类叫：

## Externally Grounded Critique

包括：

* execution trace；
* unit test；
* search results；
* API response；
* simulator；
* external verifier。

([arXiv][2])

我认为这是整篇文章里**非常值得重视的一个观点**：

> **高质量 Agent 的关键不是 reflection，而是 grounded reflection。**

也就是：

$$
LLM\ reasoning + external\ truth\ signal
$$

而不是：

$$
LLM\ reasoning + LLM\ opinion
$$

---

# 六、这恰好解释了为什么 Coding Agent 进步这么快

Coding 是目前最适合 VRL 的领域之一。

因为它天然存在非常强的 feedback channel：

```text
代码
 ↓
compiler
 ↓
runtime
 ↓
test
 ↓
lint
 ↓
type checker
 ↓
git diff
```

这些本身都可以转换成语言反馈。

所以：

```text
Agent generates patch
        ↓
test
        ↓
error message
        ↓
Agent critiques
        ↓
new patch
        ↓
test again
```

这就形成了非常强的 verbal reinforcement loop。

论文自己也专门拿 **GitHub coding agent** 当三层统一案例：

1. Issue + repo + tests → Grounding
2. Error / trace / test → Deliberation
3. 成败 trajectory → 后续 fine-tuning → Learning

([arXiv][2])

---

# 七、Experiential Memory：这里开始和“Stateful Agent”交汇

Pillar 2 里还有一个非常重要的类别：

> **Experiential Memory**

例如：

```text
Episode 1:
失败
↓
总结经验
"以后出现 X 错误先检查 Y"

Episode 2:
retrieve memory
↓
避免重复错误
```

作者把 memory 分为：

* episodic memory：原始经历；
* semantic memory：抽象出来的知识；
* procedural memory：可重复执行的 skill。

([arXiv][2])

这里其实出现了一个很有意思的模糊地带。

作者说 Pillar 2：

> “single episode”。

但是加了 persistent memory 之后：

```text
episode t
 ↓
language feedback
 ↓
memory
 ↓
episode t+1
```

它事实上已经产生了**跨 episode 学习**。

只是：

> 学习发生在 external state，而不是 model weights。

这个区别非常重要。

---

# 八、所以我会把作者的三层稍微改一下

论文是：

```text
Grounding
Deliberation
Learning
```

但如果站在 Agent Architecture 角度，我认为更准确的结构是：

```text
        Environment
             │
             ▼
① World / State Representation
             │
             ▼
② Runtime Policy / Reasoning
             │
             ▼
           Action
             │
             ▼
          Feedback
             │
       ┌─────┴─────┐
       ▼           ▼
③ External     ④ Parameter
   Memory          Learning
```

这里其实存在 **两种 learning**：

### External-state learning

```text
experience
→ memory
→ rules
→ skills
→ harness
```

不改模型。

### Parametric learning

```text
experience
→ dataset
→ SFT/RL/DPO
→ weights
```

改模型。

论文把前者主要放在 Pillar 2，把后者放 Pillar 3。

我认为**这个区别对未来 Agent 系统尤其重要**。

---

# 九、Pillar 3：Language as Learning Signal

这里才进入大家传统理解里的“训练”。

作者提出一个我认为非常漂亮的视角：

## Language feedback 存在一个 compression spectrum。

Figure 5 很值得看。

最上面：

### Full verbal critique

例如：

> “这个答案的问题是没有考虑 edge case X，第二步的假设 Y 也不成立……”

训练数据：

$$
(x,\ verbal\ feedback,\ corrected\ answer)
$$

信息量最大。

---

接下来：

### Filtered trajectory

只保留：

```text
good trajectory
```

把 verbal judgment 丢掉。

例如：

> generate → critique/filter → 保留正确 reasoning → SFT。

STaR 属于这个方向。([arXiv][2])

---

再往下：

### Process supervision

例如：

```text
step 1: 1
step 2: 1
step 3: 0
step 4: 0
```

语言反馈被压缩成 step reward。

优点是：

> credit assignment 更精确。

([arXiv][2])

---

最下面：

### Preference

```text
A > B
```

甚至：

```text
reward = 0.83
```

于是：

$$
rich\ verbal\ feedback
\rightarrow scalar
$$

DPO / RLHF 就属于这一端。([arXiv][2])

---

# 十、这个“compression spectrum”其实非常重要

传统 RL 的基本想法是：

> 把 feedback 压缩成 reward。

例如：

```text
整个复杂世界
↓
+1 / -1
```

这样优点巨大：

* 好计算；
* 好规模化；
* 好优化。

但代价也明显：

> **大量 causal information 被扔掉。**

假设员工给 Agent feedback：

> “结果虽然对了，但你用了一个已经 deprecated 的 API，而且没有考虑 timeout。如果数据量超过 100 万，这个方案会挂。”

压缩成：

```text
reward = -1
```

模型并不知道：

> **为什么错。**

所以 Verbal RL 的一个根本思想其实是：

> **不要过早压缩 feedback。**

这可能是我认为这篇论文最值得延伸的 insight。

---

# 十一、这和你之前问的“为什么一定要评测/围栏”是一个问题

因为：

> **语言 feedback 信息量高 ≠ feedback 就是真的。**

这篇论文反复强调：

### Feedback quality 是瓶颈。

如果：

```text
Generator = LLM
Critic = same LLM
Judge = same LLM
Memory writer = same LLM
```

就很容易变成：

```text
错误
↓
自我合理化
↓
错误 critique
↓
错误 memory
↓
继续强化
```

甚至：

> **compounding error。**

([arXiv][2])

所以自我进化系统最终并不是：

> Generate → Reflect → Improve

这么简单。

而应该是：

```text
Generate
   ↓
External evidence
   ↓
Verifier / critic
   ↓
Attribution
   ↓
Correction
   ↓
Memory / learning
```

这和你之前讨论 Warp self-improving agents 时追问：

> “没有围栏和评测，怎么保证效果提升？”

本质上完全一致。

---

# 十二、我认为这篇文章最有价值的一段，反而在 Section 6.2

作者提出一个很重要的观点：

> **未来 Tool 不应该主要为人设计，而应该为 Agent 设计。**

他们称之为：

> **agent consumability**

很多 API 是给程序员看的：

```text
Error: 400
Invalid request
```

但对 Agent 来说这非常糟糕。

更好的反馈可能应该是：

```json
{
  "status": "failed",
  "error_type": "authorization",
  "recoverable": true,
  "cause": "expired_access_token",
  "recommended_action": "refresh_token",
  "retry_after": null
}
```

因为 Agent 必须区分：

```text
task error
tool error
network error
permission error
environment error
```

否则：

> tool 明明给了正确反馈，
> Agent 却错误 attribution，
> 然后采取错误 recovery。

论文明确提出，Tool API 应该暴露结构化 metadata，让 Agent 能把 task-level failure 和 infrastructure failure 区分开。([arXiv][2])

---

# 十三、这和 Agent Harness 几乎直接接上了

这里论文甚至直接引用了：

**Ning et al., Code as Agent Harness**。([arXiv][2])

它的论点大致是：

> Code / runtime / tools / execution feedback 构成 Agent 的 operational substrate。

而这篇 VRL 论文进一步告诉你：

> **Harness 最重要的作用之一，其实就是构造高质量 feedback channel。**

因此我会把 harness 的价值重新定义成：

$$
Harness =
State\ Compression
+
Action\ Grounding
+
Feedback\ Generation
+
Feedback\ Verification
+
Memory
$$

这已经比：

> “LLM + tools”

深了一层。

---

# 十四、甚至可以重新理解“为什么 Harness 能让同一个模型能力差这么大”

假设完全同一个 Model。

### Harness A

```text
Prompt
↓
Model
↓
Tool
↓
"Error"
↓
Model guess
```

### Harness B

```text
structured state
↓
Model
↓
typed action
↓
execution
↓
structured trace
↓
verifier
↓
causal feedback
↓
memory
↓
retry
```

Model weights 完全一样。

但 Agent capability 可能差非常远。

原因不是：

> Model B 更聪明。

而是：

> **B 获得了更高质量、更高信息密度、更可归因的 reinforcement signal。**

这其实给“Harness 为什么重要”提供了一个非常好的理论视角。

---

# 十五、论文另外一个重要判断：未来 bottleneck 会发生迁移

作者最后有一句非常值得注意：

现在的问题逐渐不是：

> **Can we generate feedback?**

而会变成：

> **Can we verify feedback?**

也就是：

```text
过去
缺反馈
↓
现在
LLM 可以生成海量反馈
↓
未来
反馈太多
↓
不知道哪些可信
```

因此真正重要的基础设施会变成：

* feedback provenance；
* feedback quality evaluation；
* critic calibration；
* verifier；
* adversarial robustness。

([arXiv][2])

我非常认同这一判断。

---

# 十六、还有一个容易被忽略的问题：Feedback 也是攻击面

如果 Agent 会学习 feedback：

```text
feedback
↓
reasoning
↓
memory
↓
future behavior
```

那么恶意 feedback 就不是普通 prompt injection。

它可以变成：

> **policy manipulation。**

尤其 memory agent：

```text
malicious feedback
↓
write into memory
↓
cross-session persistence
↓
future behavior drift
```

作者因此提出：

> feedback provenance 会成为 Agent infrastructure 的核心组件。

即：

```text
Who said this?
↓
Is it trusted?
↓
Is it verified?
↓
How much weight should it receive?
```

([arXiv][2])

这也是一个很重要的安全设计思想。

---

# 十七、不过，我认为论文最大的概念问题也在这里

作者把 **Verbal Reinforcement Learning** 定义得非常宽。

宽到：

* prompt；
* instruction；
* reflection；
* tool error；
* memory；
* debate；
* DPO；
* RLHF；

全部都可以被叫作 VRL。

从统一视角上看，很漂亮。

但从严格的 ML terminology 来说：

> **Pillar 1 和 Pillar 2 很多方法根本不是传统意义上的 reinforcement learning。**

例如：

```text
LLM
↓
看到 unit test error
↓
修改代码
```

没有：

* reward maximization；
* policy gradient；
* value function；
* parameter update。

严格讲更像：

> feedback-conditioned inference / closed-loop control。

作者实际上是在故意扩大 RL 的定义，把：

> **任何利用 feedback 改善 future decision-making 的机制**

都纳入 reinforcement。

论文自己也承认这套 taxonomy 是一个 organizing lens，而不是严格互斥的理论划分。([arXiv][2])

所以我不会把“VRL”这个名字看得特别重。

---

# 十八、真正值得记住的不是 VRL 这个词，而是三个 insight

## Insight 1：Feedback 是 Agent 的第一等公民

过去架构：

```text
Model
↓
Tool
```

未来架构：

```text
State
 ↓
Model
 ↓
Action
 ↓
Environment
 ↓
Feedback
 ↓
Verifier
 ↓
Memory / Learning
 ↺
```

**Agent 是 feedback system，不只是 generation system。**

---

## Insight 2：不要过早把世界压成 scalar reward

未来很可能会更多保留：

```text
why
where
what caused
how to correct
```

而不是只保留：

```text
good / bad
```

也就是：

> **rich feedback → learning**

而不是：

> rich feedback → scalar → learning。

---

## Insight 3：Harness 的核心价值之一，是设计 feedback channel

所以以后评估一个 Agent runtime，我会越来越关注：

```text
State 是否准确？
↓
Action 是否 executable？
↓
Environment 是否给真实反馈？
↓
Feedback 是否 structured？
↓
错误是否 attribution？
↓
Verifier 是否独立？
↓
经验是否能沉淀？
```

而不仅仅是：

> “它支持多少 tools / MCP？”

---

# 十九、如果把这篇论文放进你最近读的几篇工作里，会形成一个非常清晰的结构

我会这样组织：

```text
                   Foundation Model
                         │
                         ▼
                ┌─────────────────┐
                │  Agent Harness  │
                │ Runtime / Code  │
                └─────────────────┘
                  │      │      │
                  ▼      ▼      ▼
                State  Action  Memory
                  │      │
                  └──┬───┘
                     ▼
                 Environment
                     │
                     ▼
              Verbal Feedback
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
       Reasoning    Memory    Training
       Pillar 2     External   Pillar 3
                    learning
```

所以：

**Code as Agent Harness** 解决的是：

> Agent 如何拥有一个可执行、可验证、stateful 的 runtime。([arXiv][3])

你前面讨论的 **Stateful / Prime Agent** 解决的是：

> 如何约束和构造 Agent 所处的 state space。

而这篇 **Verbal RL** 则是在告诉我们：

> **这些 state、action、runtime execution 最终必须形成高质量 feedback，feedback 才是 Agent 自适应和持续提升的燃料。**

---

# 二十、我对这篇文章的最终评价

如果把价值分成三层：

**算法创新：★★☆☆☆**
它基本没有提出新的核心算法，是 survey。

**taxonomy 价值：★★★★☆**
把 grounding / runtime deliberation / training 串在一条时间轴上，是很好的整理框架。([arXiv][2])

**对 Agent architecture 的启发：★★★★★**

尤其是三个观点：

> **Tool output quality 是 iterative agent 的能力上限之一。**

> **未来 tool/interface 要围绕 agent consumability 设计，而不是只围绕 human readability。**

> **未来瓶颈会从 feedback generation 转向 feedback verification。**

([arXiv][2])

这三个判断，我认为甚至比“VRL”这个名字本身重要。

如果再往前推一步，我会得到一个比论文更强的结论：

> **未来真正的 AI-native 系统，本质上不是“给现有系统接一个 Agent”，而是在业务系统内部建立一个可观察、可行动、可验证、可积累的 reinforcement environment。**
>
> 业务系统负责产生 state 和真实 outcome；Harness 将它们转换成 Agent 可消费的 state/action/feedback；Agent 做决策；结果再次回到系统；经验进入 external state 或 parameter learning。
>
> **闭环本身才是资产，模型只是闭环中的 policy。**

这其实和你一直强调的“运行数据 → 指标 → 决策 → write-back → 再产生数据”的 AI-native 业务闭环，已经几乎是同一个架构语言了。

[1]: https://arxiv.org/abs/2609.01597 "[2609.01597] The Rise of Verbal Reinforcement Learning"
[2]: https://arxiv.org/html/2609.01597v1 "The Rise of Verbal Reinforcement Learning"
[3]: https://arxiv.org/abs/2605.18747?utm_source=chatgpt.com "Code as Agent Harness"

---
