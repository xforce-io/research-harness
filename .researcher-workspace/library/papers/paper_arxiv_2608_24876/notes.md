# Recursive Experiential-Working Memory Evolution for Long-Horizon Agent Harnesses

`paper_arxiv_2608_24876`

## 钉选 · 评论

_2026-08-29T15:49:32.393Z_

可以，整段讨论可以压缩成下面这套核心框架：

## 1. Recuris 在解决什么问题

Recuris 关注的是 **长时程 Agent 为什么会越跑越乱**。

关键问题并不总是“模型不知道怎么做”，而是：

* 不知道当前做到哪一步；
* 忘了还有哪些目标没完成；
* 把“准备做”误当成“已经做”；
* skill 调用时机错；
* 任务是否真的完成缺少可靠验证。

所以长任务的核心瓶颈之一，是 **execution state management**，而不只是知识检索。

---

## 2. Recuris 的核心架构

它把 Agent 的记忆拆成四类：

* **Working Memory**：当前任务状态、哪些 goal pending/done/blocked；
* **Experiential Memory**：SOP、经验、技能；
* **Invocation Policy**：什么状态、什么事件下该调用哪个 skill；
* **Checker**：什么证据才算真正完成。

整个执行链是：

**当前状态 → 选择 Skill → 执行动作 → 环境返回证据 → 校验 → 更新状态**

最重要的一点是：

> **不是 conversation history 决定下一步，而是 verified task state 决定下一步。**

因此它本质上是在做 **state-conditioned procedural retrieval**，而不仅是普通 RAG。

---

## 3. Working Memory 是论文最关键的发现

论文的消融实验非常说明问题：

* 只加 Experience Memory，提升很小；
* 只加 Working Memory，提升非常大。

这说明很多长任务里，Agent 最大的问题不是：

> “不会做。”

而是：

> **“不知道自己现在做到哪了。”**

所以未来 Agent Memory 不能只理解成向量数据库或长期记忆，至少要分成：

**过去学到了什么**
和
**现在进行到哪里**

这两类完全不同的 memory。

---

## 4. 为什么普通长上下文和 RAG 不够

长 conversation 里同时混着：

* 已完成目标；
* 未完成目标；
* 旧计划；
* 新要求；
* tool result；
* 用户确认；
* Agent 自己的描述。

RAG 解决的是：

> “什么内容和现在最相似？”

但 Agent 真正需要回答的是：

> **“现在还有什么事情没有完成？”**

所以“更多 context”不等于更强的 Agent。

Recuris 的思路是：

> **Context accumulation → Context/state selection**

而且实验显示，把所有 skill 永久塞进上下文，反而 token 更多、效果更差。

---

## 5. 它为什么能做 Self-Improvement

Recuris 不只是记录最终成功或失败。

它记录完整的结构化执行轨迹：

**状态 → 调用了什么 Skill → 做了什么 Action → Tool 返回什么 → 模型想怎么更新状态 → Checker 怎么判断 → 最终状态**

这非常关键。

因为只有“任务失败”这个结果，你不知道到底哪里错了。

可能是：

* Skill 内容错；
* Working Memory schema 错；
* Invocation Policy 没触发；
* Checker 错误判断。

Recuris 因此可以把 failure **定位到具体组件**。

也就是：

> **execution trace → failure attribution → localized patch**

而不是简单：

> failure → reflection → 重写整个 prompt。

---

## 6. 真正的 RSI 闭环

它形成了一个很清晰的循环：

**Execution
→ Structured Trace
→ Diagnosis
→ Local Patch
→ Validation
→ New Memory
→ New Execution**

需要强调的是：

它不修改：

* Base LLM；
* Meta-Agent；
* 外层 Harness；
* Tool。

真正变化的是那套：

**Working Memory + Skill + Invocation Policy + Checker**

所以它更准确地说是：

> **bounded externalized RSI**

而不是模型自己无限改写自己。

---

## 7. Validation Gate 是整个系统能成立的关键

Self-evolving 系统最危险的问题不是“不会改”，而是：

> **改好 A，弄坏 B。**

所以 Recuris 不允许 patch 直接上线。

必须验证：

* 当前失败是否被修好；
* 已经会做的旧任务有没有退化。

可以把它理解成 Agent 世界里的：

**debug → patch → regression test → CI → deploy**

因此未来 RSI 未必依赖一个永远正确的超级 Meta-Agent。

更现实的是：

> **允许产生很多 candidate patch，但用 verifier 和 regression test 淘汰坏 patch。**

---

## 8. 实验最值得关注的结论

几个结论比单纯涨了多少分更重要：

### 第一，越长的任务，收益越明显

说明显式状态管理主要解决的是 long-horizon degradation。

### 第二，强模型同样受益很大

GPT-5.6 Sol、Claude Opus 5 仍然有十几个点的提升。

说明很多 Agent 问题属于：

> **system problem，而不是纯 model intelligence problem。**

### 第三，Memory 可以跨模型迁移

一个模型上积累出的 operational memory，可以给其他模型使用。

说明它学到的不只是 prompt hack，而是某种：

> **domain-level operational knowledge / SOP。**

---

## 9. Terminal-Bench 的结果要谨慎看

论文也证明了一点：

> 不能看到“多轮自我改进成功率提高”，就直接认为 Agent 学会了。

Terminal-Bench 中，大部分提升来自 retry，多给几次机会本身就会提升结果。

真正 memory adaptation 的额外收益比较有限，而且统计上并不显著。

所以这篇论文最强的证据不是：

> “同一个任务失败后不断自我修复。”

而是：

> **一个任务中提炼出来的 operational knowledge，可以帮助之后同类任务。**

也就是 **cross-task evolution** 比单任务 test-time RSI 更可信。

---

## 10. 这篇论文真正改变的，是对 Agent Harness 的理解

过去 Harness 很容易被理解成：

> tool routing、context、memory、retry 这些外围胶水代码。

Recuris 展示的是另一种可能：

> **Harness 本身可以成为学习系统。**

真实生产执行不断产生：

**state + action + tool observation + verification + failure**

这些不只是日志，而可以成为：

> **下一版 Agent runtime 的训练数据。**

所以 Harness 可以形成：

**运行 → 数据 → 诊断 → 改进 → 验证 → 再运行**

的飞轮。

---

# 最值得保留的 5 个 Insight

**1. 长时程 Agent 的关键瓶颈之一不是知识不足，而是状态漂移。**

**2. Working Memory 很可能比传统 Experience Memory 更重要。**

**3. Agent 想自我改进，首先必须有结构化 observability，而不是只记录对话。**

**4. Self-evolution 最现实的路径不是重写整个 Agent，而是 localized patch + regression gate。**

**5. RSI 很可能首先发生在 Harness / operational memory，而不是模型权重。**

---

最终可以把 Recuris 的核心浓缩成一条链：

> **State → Skill → Action → Evidence → State → Trace → Diagnosis → Patch → Validation → New Memory**

它最重要的意义，不是证明“Agent 已经会真正自我进化”，而是提出了一条很现实的工程路线：

> **让 Agent 每一次真实工作，都沉淀成可验证、可归因、可复用、可迭代的 operational knowledge。**

一旦这个闭环跑起来，Agent 的壁垒就可能不再只是“用了哪个模型”，而是 **在真实业务里积累和进化了多少运行经验**。

---

## 钉选 · 评论

_2026-08-29T16:03:00.309Z_

对，这基本是论文最重要的 takeaway；但我会把它表述得更严格一些：

> 长任务智能体需要一个“可验证的状态跟踪与状态转换控制器”，持续维护未完成目标，并只根据环境证据提交状态变化。

它不只是一个任务清单或 LLM 摘要。至少需要四件事：

1. **结构化状态**：把任务拆成目标，记录 `pending / done / blocked`、证据和阻塞原因。
2. **状态转换提议**：行动和环境反馈之后，提出“哪些目标应改变状态”。
3. **证据检查器**：核验工具回执或环境结果，而非相信模型自己的完成宣称。
4. **提交规则**：只有检查通过，才把状态从 `pending` 写成 `done`。

这会让长任务从：

```text
模型依据长历史“感觉”进展到哪里了
```

变成：

```text
当前有哪些未闭环目标？
→ 此刻该调用什么能力？
→ 行动后是否拿到了可验证的结果？
→ 哪些状态能安全地更新？
```

不过论文还有第二层主张：这个状态跟踪器不应只是“防遗忘”。它还应成为**技能检索的控制信号**——根据“尚未完成什么”和“正要做什么操作”取回相应 skill；并留下状态、动作、回执和检查结果的轨迹，便于失败后定位究竟该改技能、状态结构、检索时机，还是检查规则。

所以一句话总结：

> 长任务 agent 的可靠性，核心不在于给模型塞更多历史，而在于把进度做成可验证、可驱动下一步决策、可复盘的外部状态机。 :codex-file-citation{path="/var/folders/jb/tscbsx51573chq8g__11qvyc0000gn/T/codex-tab-context-assets/fa2db756-b1f3-4c8e-97e2-bdda75c831ad-chrome-tab-259734734-c918b67c-dbe6-47a0-96d2-2630154b40a3.pdf" purpose="source"}

---
