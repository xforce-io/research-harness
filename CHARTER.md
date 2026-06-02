# CHARTER — research-harness

> 全支柱研究的**共享锚**。每根 active 支柱会把相关切片(**共享核心 + 本支柱 excerpt**)同步进自己仓的 `.researcher/charter.md`(`AUTO-SYNCED — do not edit there`),`researcher` 管线读它以保持锚定。
>
> 漂移以 **「charter tension」** 形式 surface 交人裁决 —— **双向**:可能是支柱跑偏(纠支柱),也可能是研究发现真东西、反过来该更新本 CHARTER。锚,不是紧身衣。

---

## 0. 北极星(shared invariant)

为"**有目标地稳定执行长任务**"的 agent harness 奠定基石。每根支柱深耕一根基石,共同支撑这一总纲 —— 没有这些组件,智能体有目标的稳定长任务执行没有基石。

## 1. 支柱图与共享概念边界(shared invariant)

```
 安全治理(security, 左横轴) ─────────────────────────────►
 ┌─────────── evolution(进化底座之上的 artifacts)──────────┐
 │  prompts · 技能库 · 微调模型 · 决策策略 · 本体 pattern …    │ ◄ trace 喂信号
 │  data ┐                                                    │
 │       ├─► ontology ─► decision                             │
 │  exec ┘  (统一 schema)                                     │
 └────────────────────────────────────────────────────────────┘
 trace(轨迹/可观测/诊断, 右横轴) ────────────────────────►
```

锁定的不变式(所有支柱共享、不得各自重定义):

- **资源分两类**:`data`(供给知识)+ `execution`(供给行动力),结构对偶,**各自持有自己的 catalog/元数据**。
- **元数据不是支柱**:它横跨所有资源;支柱归属判据是资源**角色**(知识 vs 行动),**不是**"有没有元数据"。
- **`ontology` = 两类资源 catalog 共同遵循的 schema 层**(类型/概念模式);各支柱元数据是符合该 schema 的实例级 catalog。
- **`evolution` = 进化架在 data/execution/ontology/decision 之上的 artifacts**(prompts / 技能 / 微调模型 / 决策策略 / 本体 pattern),由 `trace` 信号驱动;是反馈层,**不是**改框架代码。
- **`security` / `trace` = 左右两条端到端横切轴**。
- **命名通则**:名字 = 功能,精确 scope 进各支柱自己的 thesis(故 `data`≠resource、`decision`≠agent、`evolution`≠artifact-evolution —— **永不用整体名命名部分**)。

## 2. 各支柱 excerpt(mandate / 边界 / 接口)

### `trace` —— 右横切轴
- **Mandate**:轨迹管理、可观测性、诊断。把 agent 执行轨迹变成可观测、可诊断、可 triage 的信号。
- **边界**:管"观测与诊断信号";**不管**如何用这些信号去改 artifacts(那是 `evolution`)。
- **接口**:向 `evolution` 喂信号(轨迹 → 可学习信号);横切观测所有纵向支柱。

### `decision` —— 纵向
- **Mandate**:决策智能体。给定 data/execution/ontology,决定下一步做什么、如何编排多步执行以稳定推进长任务。
- **边界**:管"决定与编排";**不管**知识供给(`data`)、行动力供给(`execution`)、schema(`ontology`)、artifact 进化(`evolution`)。
- **接口**:消费 `data`+`execution`+`ontology`;受 `trace` 观测;其策略/prompt 作为 artifact 被 `evolution` 进化。

### `data` —— 纵向·供给
- **Mandate**:数据资源管理。维护一套对数据资源的**类型化元数据 catalog + 连接/访问**,覆盖结构化、非结构化、向量与视图数据,让 agent 在长任务中可靠地**发现、选择、访问**正确的数据。供给"知识"(与 `execution` 供给"行动力"对偶)。
- **边界**:管"知识供给"(catalog/元数据 + 连接 + 视图 + 新鲜度);**不管**决策编排(`decision`)、行动力供给(`execution`)、统一 schema 本身(`ontology`)、artifact 进化(`evolution`)。元数据是本支柱自有 catalog 面,不是独立支柱。
- **接口**:catalog 元数据**遵循 `ontology` 的 schema**;向 `decision` 供给经类型化的数据上下文;受 `trace` 观测;受 `security` 治理(访问控制)。

> 其余支柱(execution / ontology / evolution / security)的 excerpt 在各自支柱启动研究时补入。
