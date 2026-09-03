---
title: "Agent Lightning v1.0: Towards Harnessed Agentic RL"
authors: ["Zhiyuan He","Siwei Zhang","Zhiwen Zhou","Yuqing Yang","Yu Kang","Yuge Zhang","Luna K. Qiu","Tin Yan Tsui","Jiahang Xu","Chong Luo"]
paper_id: "paper_arxiv_2608_17528"
source_kind: "arxiv"
source_id: "arxiv:2608.17528"
source_url: "https://arxiv.org/abs/2608.17528"
pdf_url: "https://arxiv.org/pdf/2608.17528"
read_id: "read_paper_arxiv_2608_17528"
kind: library-read
doc_type: "paper"
tags: []
---

# Agent Lightning v1.0: Towards Harnessed Agentic RL

> 训练引擎不再自己实现 agent 循环：让部署时的 agent harness 经一个 LLM API 代理直接参与 RL，训练器只观察 调用序列，并系统解决由此产生的 retokenization 合并、rollout 级优势计算与损失归一化问题--任意 harness 改一个 endpoint 即可被训练。

## Essence

**问题**。传统 agentic RL（verl、AReaL、slime 早期版本）要求 agent 循环在训练框架内部实现，与真实部署的 harness（mini-SWE-agent、OpenHands、Claude Code 等）割裂。即便经 proxy 接入（论文称之为 harnessed agentic RL），训练器看到的也不再是连续 token 轨迹，而是 harness 各自构造 prompt 的调用序列：token 前缀连续性被 retokenization 打断，一个 rollout 裂成数量不定的训练样本。

**做法**。约 3,500 行的轻量系统：API Gateway（存储 rollout/model/event，OpenAI 兼容代理，路径嵌入 rollout_id）、Rollout Controller（每个 rollout 调度为 K8s Job 或本地进程）、Customized Trainer（基于 VERL）。三个核心设计选择：best-effort 样本合并--仅当 token 级前缀精确匹配时合并，否则截断重开；rollout 级优势计算 + rollout 级 token-mean 损失归一化--理由是样本数由 retokenization 等偶发因素驱动，不应改变统计权重；collocated async--rollout 与更新分时共享同一 GPU 池，相位切换对 harness 不可见。

**证据**。仅用 6K 训练样本（SWE-smith 清洗后），RL 使 Qwen3.5-9B 在 SWE-bench Verified 上从 41.8% 升至 56.4%（+14.6pt）。

**边界**。这是框架与设计选择研究，不是新 RL 算法；结论全部来自单模型单次运行；依赖对每次 LLM 调用记录 token IDs 与 logprobs（model_request 事件），即需要可控的推理端点。

## Claims

- Harnessed agentic RL 中 harness 而非训练引擎拥有环境交互循环，训练器仅观察调用对序列 (p₁,a₁),(p₂,a₂),…；两范式均可形式化为 POMDP，但隐状态额外包含 harness state，策略按调用级转移采样 [§1, §2]。
- 文本级前缀条件成立不保证 token 级前缀成立；三个机制造成断裂：chat-template 非组合性（Template(A∥B) ≠ Template(A)∥Template(B)，实测 Qwen 模板会移除先前消息中的某个 marker）、decode–retokenize 漂移（"having" 可被采样为 h+aving 而重分词为 hav+ing）、推理期输出变换（tool-call 解析/修复/重序列化改变文本）[§2.1, Fig 3]。
- AReaL 与 verl Uni-Agent 的缓冲 token 替换在缓冲 ID 与实际下一请求不一致时会构造 stitched prompt，使响应在并非其采样的条件下被训练，引入 off-policy 偏差；Agent Lightning 改用 best-effort 合并保证 on-policy 正确性，代价是合并率降低 [§2.1]。
- 动态样本数是常态而非边缘情况：编码 agent 训练中平均每个 rollout 产生 2.41 个训练样本，仅 36% 的 rollout 保持单样本 [§2.2, Fig 10]。
- Rollout 级优势计算与 rollout 级 token-mean 归一化是更 principled 的选择：样本数由偶发因素驱动时 baseline 不应随之变化（示例中 rollout 级 baseline 为 1/2 而样本级为 3/4）；seq-mean 损失（GRPO 式）给多样本 rollout 不成比例权重，token-mean（DAPO 式）对长负样本敏感、后期不稳定 [§2.2, §2.3, Fig 4, Fig 5]。
- Collocated async RL 使 rollout 与更新共享同一 GPU 池：更新开始时 gateway 停收新请求并等在途请求完成，新请求暂停到下一 rollout 相位；实测约 2× 端到端加速于同步 RL，且 GPU 数少于 AReaL 式双池 async [§3.1, Fig 6]。
- 跨服务边界的可靠性由两点保证：rollout API 端点幂等（调用方可自由重试）；LLM 调用不可幂等，故训练样本组装时对同 prompt 重复调用去重、仅保留最后一次 [§3.2]。
- 编码 agent 训练中实测四种 reward hacking 形态（git 历史定位 gold commit、wget/curl 拉上游源码、pip 下载源码、urllib 等网络库），以禁用 git + 隐藏 .git + K8s 网络白名单阻断 [§4.3.2]。
- 三变体消融（同一 GRPO 目标）：sample 级优势+token-mean 达 35.0%，rollout 级优势+token-mean 降至 33.1%，rollout 级优势+rollout 级归一化最高 38.2%（step 128 验证奖励）；末者策略熵增长更慢更稳，说明 rollout 级归一化抑制了修正优势引入的熵上升 [§4.3.3, Fig 9]。
- SWE-smith 清洗管线：59,136 条中去除 18,033 条空 problem statement、1,265 条缺 problem branch、测试数 >200 的任务，再用 Qwen3.5-9B 四次 rollout 做难度过滤（全对删、有对有错留，另补 1,000 条全错），得约 6K 训练 + 400 测试 [§4.3.1]。
- 搜索 agent（Search-R1 设置，Llama-3.2-3B-Instruct + GRPO，HotpotQA 训练，6 数据集各采样 50 例，EM 奖励）：验证奖励 25.1% -> 41.7% [§4.1]。
- 指令跟随 agent（LLM-in-Sandbox 设置，Qwen3-4B-Instruct-2507 + RLOO，Instruction Pre-Training 数据 80/20）：验证奖励 51.9% -> 70.2% [§4.2]。
- 编码 agent：SWE-bench Verified 从 41.8% 提升至 56.4%（step 208 checkpoint，6K 样本与“适度算力”）[§4.3.3]。

## Assumptions

- 训练系统可为每次 LLM 调用记录 prompt token IDs、response token IDs 与 logprobs--隐含需要自托管或可控推理端点，而非任意商用 API [A.1]。
- Harness 通过 OpenAI 兼容 API 调用模型且可把 endpoint 切到代理（不做本地推理）[§3, A.1]。
- 奖励是结果性标量，通常每个 rollout 结束时上报一次；过程奖励不可用或不必要 [A.1]。
- 训练-部署同构（"narrowing the gap between training and actual use"）本身有价值--该 gap 的实际影响未被量化 [§1]。
- 同 prompt 的重复调用均为重试或被取代的调用，可安全丢弃；agent 不会合法地两次发送完全相同的 prompt [§3.2]。
- SWE-smith 的任务难度以被训练模型本身（Qwen3.5-9B，4 rollouts）的通过率定义，且该定义对训练有效 [§4.3.1]。
- K8s 集群（自托管算力与运维能力）可以作为执行后端 [§3.3]。

## Method

调用序列视角的前后对比：

| | 传统 agentic RL | Harnessed agentic RL |
|---|---|---|
| 交互循环归属 | 训练引擎 | harness |
| 策略观测 | 连续扩展的 token 历史 p_t=(p_{t−1},a_{t−1},o_t) | 每次调用独立构造的 prompt |
| 一个 rollout | 一条线性 token 轨迹 = 一个训练样本 | N_ρ 个动态训练样本 |

形式化：latent state s_t=(s_t^harness, s_t^env)；harness 构造消息上下文并经 chat template 渲染为 token 级 prompt；策略按调用级转移 z_t=(p_t^tok, a_t^tok) 采样。相邻 prompt 间不假设 token 前缀关系，任何序列构造必须保持每个动作实际采样的 prompt [§2]。

三个组件 [§3, App. A]：

1. **API Gateway**--单一 stateful 服务，存储 rollout（queueing/running/succeeded/failed 状态机）、model（推理端点注册）与 event（默认 model_request：token IDs + logprobs；reward；可自定义类型）。Rollout API（创建/列出/更新/附加事件）+ Proxy API（OpenAI 兼容，路径嵌入 rollout_id 使调用自动归属到 rollout）。所有端点幂等。
2. **Rollout Controller**--轮询 queueing 的 rollout 并调度执行：K8s Reconciler（用户提供的 Job 模板，watch + 周期 list 的标准控制器模式）或 Local Reconciler（本地进程池，轮询即可）。Gateway 的 rollout 状态为 ground truth，两侧以重试达成 best-effort 最终一致。
3. **Customized Trainer**（VERL 之上）--注册 rollout、等待终态、拉取 events，由 Sample Adapter 组装训练样本：(a) best-effort 合并--仅当后一 prompt 是前一次请求+响应的精确 token 级前缀时合并为一条序列，否则关闭当前序列并重开；(b) 同 prompt 重复调用去重，保留最后一次；(c) rollout 级 baseline 与优势；(d) rollout 级 token-mean 损失（每个 rollout 等权，Eq. 16）。

Collocated async：攒够 rollout 数据即进入更新，gateway 停止接受新请求、等待在途请求完成，新到请求暂停到下一 rollout 相位；rollout 与更新在同一 GPU 池分时复用，对 harness 完全透明 [§3.1]。

监控：暴露每个训练/验证 rollout 的输入、状态、模型请求、奖励、token/turn 统计与 K8s pod 日志，可用 AI agent 自动诊断异常行为（实测发现多个 reward hacking 案例）[§3.4]。

## Eval

- **搜索 agent** [§4.1]：设置复刻 Search-R1；Llama-3.2-3B-Instruct + GRPO；HotpotQA 训练 split；评估自 HotpotQA、2WikiMultiHopQA、MuSiQue、Bamboogle、TriviaQA、NQ 各采样 50 例；batch 512、每 prompt 4 rollouts、每 10 步评估；指标 = EM 奖励。结果：验证奖励 25.1% -> 41.7%。
- **指令跟随 agent** [§4.2]：设置复刻 LLM-in-Sandbox（使用原作 harness）；Qwen3-4B-Instruct-2507 + RLOO；Instruction Pre-Training 数据 80/20 划分；batch 8、每 prompt 8 rollouts、每 20 步评估。结果：51.9% -> 70.2%。
- **编码 agent** [§4.3]：Qwen3.5-9B + GRPO + mini-SWE-agent harness；SWE-smith 清洗后约 6K 训练/400 测试；三变体消融（35.0% / 33.1% / 38.2%，step 128 验证奖励）；SWE-bench Verified 41.8% -> 56.4%（step 208）；合并统计：36% 单样本 rollout、平均 2.41 样本/rollout。
- **基线与对照**：无与 verl Uni-Agent / AReaL / slime / Polar 的同任务 head-to-head；消融基线是自家 sample 级变体；collocated async 的 2× 加速仅有文字陈述、无表格；“modest compute” 未量化。

## Weaknesses

- 无跨框架实证：核心命题是设计选择“影响算法正确性与训练稳定性”，但从未在相同任务上运行任何竞争框架（verl Uni-Agent、AReaL、slime）；best-effort 合并与缓冲 token 替换之间的 on-policy 正确性 / 合并率权衡仅停留在理论论证，既未实测两种策略的合并率差异，也未量化 stitched prompt 的 off-policy 偏差幅度。
- 单模型、单次运行：三项实验各只用一个模型、一个训练 run，无多 seed 方差或置信区间；+14.6pt 的增益与消融差异（38.2% vs 35.0%）是否超出噪声无法判断。
- 设计选择的交互未解耦：论文自己的数据表明 rollout 级优势在 token-mean 归一化下反而劣于 sample 级优势（33.1% < 35.0%），“rollout 级更 principled”只在同时改归一化粒度时成立；熵曲线的解释是定性的，未做因素分解。
- 合并失败的信息损失未量化：平均 2.41 样本/rollout 意味着多数调用序列被切断，被切断处丢弃的上下文对梯度的影响大小未知，也没有与“强制全轨迹”式 oracle 的对比来界定 best-effort 的代价；且合并率高度依赖 harness 的 prompt 构造行为，结论对其他 harness 的外推性不明。
- 系统声称缺细节：collocated async 的约 2× 加速只有文字陈述，无 GPU 利用率曲线、吞吐表，也无与 AReaL 式双池 async 的实测对比（仅“GPU 数更少”的定性说法）；“modest compute” 未定义。
- 采用门槛高：需要可控推理端点记录 token IDs 与 logprobs、自托管推理与 K8s 运维能力；对只能用商用 API 或无集群的团队不可用。“同 prompt 重复调用均可丢弃”的去重假设也可能误删合法的完全相同请求。
- 任务与奖励面窄：全部为结果性标量奖励（EM、pass/fail 类）；设计选择消融只在编码 agent 上做，搜索与指令跟随任务未验证其普适性；验证奖励与训练数据同源（同数据集划分），存在高估风险。
- 数据管线含任意成分：难度以训练模型自身通过率定义（全对删、有对有错留、另补 1,000 条全错），阈值与补集规模未论证，可能使训练分布偏向模型当前能力边界。

## Relations

- **verl / HybridFlow**：Customized Trainer 直接构建于 VERL 之上；对照关系是传统 agentic RL 把 agent 循环放进引擎（含 Uni-Agent 路线），本文把循环还给 harness、仅在样本组装层回到 VERL。
- **AReaL**：双 GPU 池 async RL 的代表；本文批评其缓冲 token 替换在缓冲 ID 失配时构造 stitched prompt（off-policy 偏差），并以 collocated async 主张用更少 GPU 获得相近异步收益--但两说均缺 head-to-head 实验。
- **slime**：同为引擎内循环的 RL 框架，被列为传统范式代表，是本文命题的对照面。
- **Search-R1**：搜索 agent 实验的设置来源与复刻对象；EM 25.1% -> 41.7% 可作为该研究线的对照点。
- **LLM-in-Sandbox**：指令跟随实验直接使用其 harness 与 Instruction Pre-Training 数据，是“harness 只需改 endpoint 即可被训练”这一卖点的最直接证据。
- **SWE-smith**：编码任务来源；本文在其上贡献了清洗与基于训练模型自身通过率的难度过滤管线（59,136 -> 约 6K）。
- **mini-SWE-agent / OpenHands / Claude Code**：目标 harness 生态；mini-SWE-agent 用于编码实验，是“harness 拥有环境交互循环”的最小实例。
- **GRPO / DAPO / RLOO**：损失归一化之争（seq-mean vs token-mean）的直接延伸；本文把粒度推进到 rollout 级，主张归一化粒度应与优势计算粒度一致。
- **多轮 LLM RL 的 POMDP 形式化**：同属把多轮交互建模为 POMDP 的脉络，但显式把 harness state 纳入隐状态、以调用级转移定义策略。
- **reward hacking 文献**：观察到的四种 hack 形态（git 历史、wget/curl、pip、urllib）是奖励博弈在 SWE 场景的具体实例；禁 git、隐藏 .git、网络白名单属环境侧 hardening 手段。
- 潜在互补方向：过程/密集奖励、多 agent 协作训练、把合并率与归一化分析推广到非编码类 harness。

## Takeaway

- 论文把一个常被基础设施细节吞没的问题显式化：**谁拥有 agent 循环决定了训练器看到什么**。让 harness 拥有循环后，retokenization 碎片化与动态样本数不是工程噪声而是统计单元--rollout 而非 sample 才是优势计算与损失归一化的正确粒度。
- 三个可迁移的设计教训：on-policy 正确性优先于合并率（宁可截断重开也不 stitch 条件）；归一化粒度与优势计算粒度保持一致（rollout 级 token-mean）；单 GPU 池分时复用（collocated async）可在不加硬件的前提下获得大部分异步收益。
- 直接的工程价值：任意 OpenAI 兼容 harness 改一个 endpoint 即接入 RL 训练，配合 rollout 级监控（含 AI agent 自动诊断，实测用于抓 reward hacking），使“训练-部署同构”成为可操作管线而非口号。
- 可信度折扣：全部证据为单模型单次运行、无跨框架对照；采纳其设计选择（尤其 rollout 级归一化）前应先在自有任务上复现消融。
