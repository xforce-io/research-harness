# Runway News | Introducing Solaris

`paper_url_a21426dbb604e489`

## 钉选 · 评论

_2026-09-02T04:15:13.244Z_

你这个疑问是对的，而且我对照原文后，会把我上一轮的说法修正一下：

**Runway 这篇 Solaris 文章根本没有描述“结构化后端服务 + Solaris 画面同步”的机制。**
相反，它更激进：**它声称交互本身也由世界模型直接学出来，而不是把点击先映射成 HTML/DOM/业务 action，再由后端驱动画面。** ([Runway][1])

最关键的是原文这几句：Solaris 把 click、drag 等用户输入直接当作生成下一帧的 conditioning；模型通过训练学习“动作 → 视觉结果”的关系，因此某个东西被点击、拖拽、修改后应该发生什么，**不需要显式编程这些 interaction**。([Runway][1])

所以 Solaris 原型更像这样：

```text
当前视觉状态 Vt
       +
用户动作 At（click / drag / type）
       +
LLM 给出的语义/行为 prompt
       ↓
   Solaris World Model
       ↓
下一视觉状态 Vt+1
```

而不是我上一轮假设的：

```text
点击坐标
  ↓
DOM / 控件 ID
  ↓
结构化业务动作
  ↓
后端 state 更新
  ↓
Solaris 渲染
```

原文甚至明确说：

> “There are no predefined screens and no templates to fall back on.”

以及：点击、拖拽意味着什么，是通过**自然语言 prompt**指定的，而不是传统代码定义的。([Runway][1])

---

### 那么问题就来了：它怎么做“订酒店”这种任务？

这恰恰是 Solaris 当前最大的概念缺口。

假设画面上有：

```text
Hotel A
$180/night
[Book]
```

Agent 点击 Book。

Solaris完全可以生成下一帧：

```text
✓ Booking confirmed
```

但是这里存在两个完全不同的“成功”：

```text
视觉成功：
Solaris 生成了一张
“Booking confirmed”的画面

        ≠

业务成功：
真实 booking API
真的创建了一张订单
```

**Solaris 本身只能天然保证第一种。**

因为其内部 state 本质上仍然是生成模型里的 latent / visual state，并不是：

```python
booking = {
    "hotel_id": 1827,
    "check_in": "2026-09-08",
    "nights": 2,
    "status": "confirmed"
}
```

所以你刚才说：

> “毕竟生成的只是画面，后端是要结构化判断的。”

完全抓到了核心矛盾。

---

## 为什么 Runway 仍然说它可以训练 Computer-use Agent？

因为**训练 Agent**和**真正运行商业软件**其实是两个问题。

训练时未必需要真实 booking backend。

例如训练任务只是：

> “看到界面以后，把红色杯子拖到桌子上。”

Solaris 可以生成：

```text
state_t
   ↓
Agent 看图
   ↓
drag(x1,y1 → x2,y2)
   ↓
Solaris
   ↓
state_t+1：
杯子被拖到桌上
```

然后可以用视觉模型/判别器判断：

```text
杯子现在是不是在桌上？
```

这种环境很像 **generative simulator**。

你训练的是：

> 在大量不同视觉世界中，
> Agent 是否能理解画面并采取合理动作。

所以 Solaris 的价值非常像：

```text
传统 GUI benchmark

固定网站 A
固定网站 B
固定网站 C
      ↓
Agent 容易学布局 shortcut
```

变成：

```text
Solaris

无限生成：
网站 A'
网站 A''
网站 A'''
完全没见过的网站 A''''
      ↓
Agent 被迫学习视觉语义
而不是记按钮位置
```

Runway 原文强调的正是这一点：它希望用不断变化、甚至“从未存在过”的界面训练 Agent，从而减少 Agent 对固定 coded interface 布局的过拟合。([Runway][1])

---

# 但真正重要的是：Solaris 不是完整的软件 runtime

我认为这是理解这篇文章最重要的一层。

Runway 的宣传语言是：

> “image becomes the application itself”

但严格从计算机系统角度说，目前更准确应该是：

> **image becomes the interactive presentation/runtime simulation layer**

而不是：

> image 完全替代数据库、交易系统、权限系统和业务逻辑。

因为软件至少有三层：

```text
① Presentation state
“用户看到了什么”

② Semantic / application state
“现在处于什么业务状态”

③ Ground-truth state
“数据库里真正发生了什么”
```

传统软件：

```text
Database / application state
          ↓
      deterministic code
          ↓
       UI / pixels
```

Solaris目前展示的是：

```text
视觉历史 + interaction + semantic prompt
                ↓
          World Model
                ↓
             pixels
```

它非常强地重构了 **① presentation state**，并且开始让模型隐式承担一部分 **② semantic/application dynamics**。

但是：

**③ ground-truth state 并没有因为 Solaris 而消失。**

---

# 所以未来真正可用的架构，我反而认为会是 Hybrid

例如淘宝/酒店/医院系统不可能这么干：

```text
Solaris：
“我觉得你付款成功了。”

→ 订单成功
```

而应该是：

```text
                    ┌─────────────┐
                    │   LLM       │
                    │ reasoning   │
                    └──────┬──────┘
                           │
                      intent / prompt
                           ↓
User action ───→ Solaris World Model
                    │
                    │ 生成交互视觉
                    ↓
                  Pixels
```

同时另一条线：

```text
User / Agent intent
        ↓
semantic action
        ↓
real API / tool
        ↓
structured backend state
        ↓
数据库 / payment / inventory
```

然后：

```text
structured backend result
        ↓
成为 Solaris 的 conditioning
        ↓
生成对应的视觉世界
```

也就是：

# **World Model 管“experience”，API 管“truth”。**

这个分工我认为非常关键。

---

而且你看 Runway 自己其实已经无意中透露了这一点。

原文说 Solaris 里面还有一个 **LLM**：

> LLM 决定界面应该如何演化，判断是修改当前 scene 还是 transition 到新场景，并生成 prompt 指导 Solaris。([Runway][1])

所以真正结构其实已经不是：

```text
action → video model
```

而更接近：

```text
                     ┌──────────────┐
User intent/action → │     LLM      │
                     │ semantic     │
                     │ reasoning    │
                     └──────┬───────┘
                            │
                 behavior / prompts
                            ↓
current frames ─────→ Solaris
                            ↓
                       next frames
```

**LLM 已经在承担“隐式 application logic”。**

现在缺的就是：

```text
LLM
 ↓
Tool / API / structured state
 ↓
Solaris
```

这一步文章没有公开。

---

## 这其实引出一个非常有意思的判断

Solaris 真正挑战的未必是：

> **“HTML 会不会消失？”**

而可能是：

> **“UI state 是否还需要和 application state 一一对应？”**

传统软件：

```text
button
 ↓
component
 ↓
event handler
 ↓
API
```

每一层都非常 rigid。

Solaris 可能变成：

```text
用户表达意图 / interaction
          ↓
      semantic agent
       ↙        ↘
 real tool      world model
   ↓                ↓
真实状态          个性化视觉
```

也就是说：

**未来可能消失的不是 backend，也不是结构化 state，而是现在这一整层“把结构化 state 手工编码成固定 GUI”的 frontend application logic。**

这比“视频模型生成网页”其实深得多。

你可以把它理解成：

```text
今天：

structured state
      ↓
HTML / React / CSS
      ↓
pixels


Solaris 式未来：

structured state ─────┐
user intent ──────────┼→ generative runtime → pixels
interaction history ──┘
```

**React/HTML 这层 intermediate representation 被削弱甚至消失，结构化业务状态却仍然保留。**

而 Runway 现在展示的 Solaris，其实只做出了右半边的 **generative runtime**，还没有完整展示怎么和左边的真实业务 state 拼起来。

所以你的质疑并不是一个小工程细节，反而是这篇文章从 demo 走向真正“新操作系统/新软件范式”的**核心未解问题之一**。([Runway][1])

这也解释了为什么我会把 Solaris 更准确地称作 **Interface World Model**，而不是“已经可以替代 Web runtime 的世界模型”。

[1]: https://runway.com/news/research/introducing-solaris "Runway News | Introducing Solaris"

---
