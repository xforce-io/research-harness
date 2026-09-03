---
title: "A Programming Paradigm for Spatiotemporal Composability"
authors: []
paper_id: "paper_url_cf29e7f27de188a0"
source_kind: "url"
source_id: "url:https://github.com/cordiverse/paper"
source_url: "https://github.com/cordiverse/paper"
pdf_url: ""
read_id: "read_paper_url_cf29e7f27de188a0"
kind: library-read
doc_type: "other"
tags: []
---

# A Programming Paradigm for Spatiotemporal Composability

> 现代软件（插件系统、自进化 Agent 框架）需要运行时动态组装，但仅靠进程/容器级别的粗粒度重启来实现 -> 将经典 effect/coeffect 提升为运行时可逆 effect 与响应式 coeffect，统一为单一上下文类型 -> 实现组件级细粒度的安全加载/卸载，无需重启丢弃状态。

## Essence

1. **问题** — 动态组合（运行时加载/卸载/重配组件）缺乏细粒度形式基础。进程级重启丢失全部本地状态（缓存、连接、中间计算），容器级编排无法表达地址空间内的组件依赖。VSCode 扩展宿主无法卸载单个扩展代码；top 100 扩展中 87 个含可执行代码，移除需重启整个宿主。
2. **做法** — 将 effect system 和 coeffect system 从编译期静态分析提升为运行时机制：每个 context 变换携带显式逆函数（revertible effect），运行时跟踪逆函数以在组件移除时恢复环境；组件声明依赖规格，每次 context 变更被分类为 activating/deactivating/neutral 并驱动组件启停（reactive coeffect）。两者统一为单一递归 context 类型 `Γ∞ = μΓ. Γ × (Γ→Γ) × Σ`。
3. **证据** — 核心类型定义：effect function `𝔈Γ ≔ Γ → Γ × (Γ→Γ)`，即每次副作用返回修改后的 context **和**一个逆函数；coeffect operation 的 `set` 直接是 `𝔈Σ∗` 上的 effect function，"coeffect operations are effects, and effects are revertible" [§3.2.1]。独立性定理（Theorem 40）保证不同 key 上的操作自动独立，使多组件交错 effect 可在意序下恢复。
4. **边界** — 不处理跨进程/跨服务的组合（仅限单地址空间内组件级）；可逆性是"观测等价"而非物理状态精确还原（`free` 不恢复堆布局，生成式名称不回退）；非交换操作（如有序中间件链）的恢复顺序必须由 coeffect 显式排序，不能自动处理。

## Key takeaways

- **可逆 effect 的核心设计**：effect 不再是 `Γ→Γ`，而是 `Γ→Γ×(Γ→Γ)`——每个副作用在发生点返回自己的逆函数，运行时累积逆函数链。恢复时按逆序应用，或（满足独立性时）任意顺序应用。
- **coeffect 即 effect**：依赖注册（`set`）本身是一个可逆 effect，自动获得跟踪和恢复。这是"时空可组合性"统一的数学基础——依赖管理不需要独立机制。
- **观测等价 ≃ 替代精确相等**：恢复保证不要求物理状态完全一致，只要求"通过 coeffect 操作不可区分"。这允许堆布局重排、名称重生成等无法物理还原的副作用在观测层面被正确回退。
- **独立性由 key 分离自动获得**：不同 dependency key 上的操作天然交换（Theorem 40），无需额外证明。同 key 操作需声明为 commutative 才能跨组件无序恢复。
- **隔离 realm（isolation realm）实现运行时 ad-hoc 多态**：同一逻辑 key 在不同 context 下解析到不同值，用于多租户、测试沙箱、组件隔离——类似依赖注入但更细粒度，且可动态调整。
- **粗粒度变通方案的成本被量化**：重启重建状态需秒到分钟级 [§1.2.3 引用 ref 15]；维护可用性需要冗余副本补偿单组件不可恢复。

## Decisions / claims

- **时间可组合性 = 可逆 effect**：每个 context 变换必须携带逆函数，运行时通过 accumulator `φ` 跟踪。卸载组件 = 应用其 accumulator。[§3.1, §3.1.1 Definition 2]
- **空间可组合性 = 响应式 coeffect**：组件声明依赖集 `d ⊆ K`，系统在每次 context 变更时检查 satisfaction predicate `σ ⊧ d` 并分类为 activating/deactivating/neutral。[§3.2.2 Definition 25-26]
- **统一 context 为递归类型**：`Γ∞ = μΓ. Γ × (Γ→Γ) × Σ`，将 effect accumulator 和 coeffect 表合并为自相似结构。`effect` 映射 `𝔈Γ∞ → 𝔈Γ∞`，消除了 `∂`-tower 的多层嵌套。[§3.3.1 Definition 32]
- **观测等价是恢复的语义基础**：`≃` 由各 key 的 `≃k` 组装，两状态 related 当且仅当 coeffect 投影 bind 相同 key 且 value related。恢复保证从 `=` 弱化为 `≃`（Lemma 38）。[§3.3.2 Definition 33, 37]
- **coeffect interception 使用 monoid merge**：每个 key 的 metadata `ℳk` 配备 monoid `(ℳk, ⊕k, ϵk)`，context-carried metadata 与 component-declared metadata 合并，right-biased 使外层 context 可约束组件行为。[§3.2.2 Definition 30-31]
- **calculus 的 metatheory 覆盖四个转换**：withdrawal（组件移除）、iteration（循环）、asynchrony（异步）、failure（故障），从单组件扩展到交错组件系统。[§4.3, §4.4]
- **Cordis 实现**：核心库提供 effect tracking 和 coeffect resolution；声明式组件加载器支持配置调和与热模块替换。Koishi 作为 case study。[§5]
- **VSCode top 100 扩展数据**：87/100 含可执行代码（移除需重启），仅 7/100 声明 `extensionDependencies`。[§1.2.1, 数据截至 2026-06-09]

## Constraints & assumptions

- **可逆性是观测等价，非物理还原**：堆布局、生成式名称等无法物理回退的副作用，只有在"无 key 绑定它们"时才被 `≃` 遗忘。若地址作为 outcome 被相等比较，则分配操作不可交换，key 非 commutative。[§3.3.2]
- **同 key 操作的交换性需显式满足**：Theorem 40 仅保证不同 key 的独立性。同 key 上的两个操作若不交换（如有序中间件链），则恢复顺序必须由 coeffect 声明显式管理，不能自动处理。[§3.3.2 Definition 39, Theorem 42]
- **set 的前置条件**：`set(k, v)` 要求 `k ∉ dom(σ)`（不能重复提供）；`get(k)` 要求 `k ∈ dom(σ)`（不能访问缺失依赖）。违反前置条件产生错误且不产生转换。[§3.2.1 Definition 23]
- **通知不保证拆解顺序**：`notify` 能检测依赖丢失并触发 deactivation，但不能单独保证提供者 A 的恢复推迟到依赖者 B 完成拆解之后。这需要 Section 4.3.1 的 withdrawal 机制。[§3.2.2]
- **单地址空间假设**：整个框架在单进程内操作组件级组合；跨进程/跨服务组合由 OS/容器编排器处理，属于"粗粒度变通方案"。[§1.2.3, §6.1-6.2]
- **effect function 的逆函数由调用方在应用点提供**：`𝔈Γ∗` 要求 `g(δ) = γ` 在应用状态成立，但不要求 `g ∘ f = idΓ` 全局成立（除非需要 `𝔈∂Γ∗`）。[§3.1.2 Definition 8, Theorem 15]

## Open questions

- **跨语言/跨 OS 协同设计**：文档在 §6.4 和 §6.7 提出语言无关性目标，但未给出非 JS/TS 语言的绑定实现或类型系统集成方案。
- **mutual dependency 和组件粒度**：§6.5 讨论了互依赖和粒度问题但未给出形式化处理；calculus（Section 4）的 metatheory 是否覆盖循环依赖未明确。
- **依赖类型化与版本控制**：§6.6 提出但未形式化；当前 `𝒱k` 类型族不携带版本信息。
- **access control 与沙箱的实现路径**：§6.3 讨论了 interception metadata 可用于权限约束，但未给出具体权限模型或安全证明。
- **大尺度系统性能**：accumulator `φ` 是函数复合，长生命周期组件链的 `φ` 深度增长是否导致恢复性能退化未讨论。

## Relations

- **builds-on** effect system (Lucassen & Gifford; Moggi; Plotkin & Power) 和 coeffect system (Petricek et al.; Gaboardi et al.) [high]: 文档 §2 和 §7.1 明确将经典 effect/coeffect 理论列为理论基础，核心贡献是"lifting"这些静态分析概念为运行时机制。
- **extends** algebraic effect handlers (Plotkin & Pretnar) [med]: effect handler 的 continuation 语义是 revertible effect 的概念前身，但本文将 handler 从编译期解释器变为运行时逆函数跟踪器，且 handler 不要求可逆性。
- **orthogonal** to transactional memory / software transactional memory (STM) [med]: 两者都关注状态回退，但 STM 面向并发冲突的乐观重试，本文面向组件生命周期的确定性卸载；STM 的 rollback 是 all-or-nothing，本文支持单组件选择性 revert。
- **competes-with** RaII / bracket patterns [med]: §1.1 明确指出静态设置下 temporal composability 归约为 RAII 和 bracket，但本文解决的是 RAII 无法处理的"非词法作用域、长生命周期、运行时到达/离去"场景。
- **extends** inversion-of-control (IoC) containers [high]: §3.2.1 明确声明将 IoC 形式化为 coeffect context，并将 key-value 依赖表扩展为带类型族、等价关系、操作集的依赖类型。

## Takeaway

- **可复用的核心洞察**：将"副作用可逆"从编程模式（try/finally, bracket）提升为类型级保证——effect function 的返回类型包含逆函数，运行时自动跟踪。任何需要运行时动态组装的系统都可借鉴此设计。
- **需谨慎对待的前提**：可逆性建立在观测等价 `≃` 上，`≃` 的选择是设计决策——选得太细（如精确地址相等）会使分配操作不可交换；选得太粗可能遗漏语义差异。文档未给出 `≃` 选择的系统化方法。
- **一句话记忆钩**：coeffect operations are effects, and effects are revertible——依赖注册本身是可逆副作用，这是时空可组合性统一的数学支点。
