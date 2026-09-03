# Controlling Reasoning Effort in LLMs

`paper_url_2be8751c7abf335d`

## 钉选 · 评论

_2026-08-02T06:01:49.327Z_

# 为什么 Qwen3 等 Hybrid Thinking 模型需要同时保留 Soft Switch 和 Hard Switch？

在支持“思考模式（Thinking Mode）”和“非思考模式（Non-thinking Mode）”的模型中，同时存在两种控制机制：

* **Soft Switch（软控制）：** 如 `/think`、`/no_think`
* **Hard Switch（硬控制）：** 如 `enable_thinking=False`、通过 chat template 注入空 `<think></think>`

二者看起来都是“控制模型是否思考”，但它们实际上处于不同层级，解决的是不同问题，因此不能互相替代。

---

# 一、核心区别：一个控制“模型行为”，一个控制“推理过程”

## 1. Soft Switch：让模型理解用户意图

Soft Switch 本质是：

> 通过自然语言指令，让模型根据训练形成的能力选择是否进行 reasoning。

例如：

```
请简单回答这个问题。

/no_think
```

模型接收到的是普通文本：

```
user:
请简单回答这个问题。
/no_think
```

然后依靠训练得到：

```
看到 /no_think
        ↓
减少或关闭 reasoning
        ↓
直接输出答案
```

它属于：

**模型能力层（Model Behavior Control）**

依赖：

* instruction following
* post-training
* 模型对特殊 token / 指令的理解能力

---

## 2. Hard Switch：让系统确定模型处于什么状态

Hard Switch 不是让模型“理解不要思考”，而是在推理前改变输入上下文结构。

例如：

```python
enable_thinking=False
```

框架可能构造：

```
assistant:

<think>

</think>
```

模型看到：

```
<think> 已经结束
```

于是直接进入：

```
answer generation
```

它不是依赖模型理解指令，而是通过 prompt template 控制模型状态。

因此属于：

**推理控制层（Inference Control）**

特点：

* 确定性强
* 可程序化控制
* 不依赖模型是否理解指令

---

# 二、为什么不能只保留 Hard Switch？

如果只有 Hard Switch，会失去模型的自然交互能力。

原因：

模型需要学会：

```
什么时候应该思考？
什么时候应该快速回答？
```

例如：

用户：

> 帮我写一个朋友圈文案

模型应该：

```
no_think
快速生成
```

用户：

> 分析一个复杂商业战略

模型应该：

```
think
展开推理
```

这种能力必须通过训练建立。

因此在 post-training 阶段，需要让模型看到：

```
输入：
问题 + /think

目标：
<think>
reasoning
</think>
answer
```

以及：

```
输入：
问题 + /no_think

目标：
answer
```

模型才能形成：

```
instruction
        ↓
reasoning mode selection
```

否则模型只是被外部硬切换，而不会真正理解“何时需要思考”。

---

# 三、为什么不能只保留 Soft Switch？

因为 Agent 和生产系统需要确定性。

例如：

一个 Agent workflow：

```
任务规划 Agent
       |
       ↓
需要深度推理
       |
       ↓
Planner
(enable_thinking=True)


执行 Agent
       |
       ↓
简单调用工具
       |
       ↓
Executor
(enable_thinking=False)
```

如果只使用：

```
/no_think
```

那么系统实际上是在赌：

> 模型会不会正确理解这个指令。

但生产系统需要：

* 稳定 token 消耗
* 稳定 latency
* 稳定输出格式
* 可预测成本

因此需要：

```
Hard Switch
```

保证：

> 这个 Agent 一定不会进入长 reasoning。

---

# 四、两种机制的真正分工

|      | Soft Switch          | Hard Switch             |
| ---- | -------------------- | ----------------------- |
| 控制层  | 模型行为层                | 推理系统层                   |
| 本质   | 指令理解                 | 上下文构造                   |
| 依赖   | 模型能力                 | 框架控制                    |
| 典型形式 | `/think` `/no_think` | `enable_thinking=False` |
| 使用对象 | 人类用户、Prompt 用户       | 开发者、Agent 系统            |
| 优点   | 灵活、自然                | 稳定、确定                   |
| 缺点   | 存在一定概率偏差             | 灵活性较低                   |

---

# 五、多轮对话中的 Thinking 保留不是 Switch 问题

需要注意：

“是否保留过去 thinking trace”与 Soft/Hard Switch 是两个不同问题。

它属于：

```
Memory Policy
```

而不是：

```
Thinking Mode Control
```

例如：

是否把：

```
assistant:
<think>
xxx
</think>
answer
```

放入下一轮上下文，需要系统额外决定。

因此：

* Soft Switch ≠ 自动保存 thinking
* Hard Switch ≠ 自动删除 thinking

二者只是控制当前推理模式。

---

# 六、最终理解

可以用一句话总结：

> **Soft Switch 是让模型学会“什么时候应该思考”；Hard Switch 是让系统保证“现在一定思考或一定不思考”。Soft Switch 解决智能交互问题，Hard Switch 解决工程可靠性问题。**

类似 Agent 系统中的：

```
自然语言约束
        +
系统级约束
```

两者不是竞争关系，而是分层协作：

```
              用户意图
                  |
          ----------------
          |              |
      Soft Switch     Hard Switch
          |              |
   模型理解能力       系统确定控制
          |              |
          ------推理行为------
                  |
          Thinking / No Thinking
```

因此，像 Qwen3 这样的 Hybrid Thinking 模型同时保留两种机制，是为了同时满足：

1. **模型能力：让模型理解并适应不同任务需求**
2. **产品体验：让普通用户自然切换**
3. **工程可靠性：让 Agent 和生产系统稳定控制成本与延迟**

这也是未来 Agent 系统中“模型智能”和“系统工程控制”必须同时存在的典型例子。

---
