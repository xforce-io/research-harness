# CHARTER — research-harness

> 全支柱研究的**共享锚**。每根 active 支柱会把相关切片（**共享核心 + 本支柱 excerpt**）同步进自己仓的 `.researcher/charter.md`（`AUTO-SYNCED — do not edit there`），`researcher` 管线读它以保持锚定。
>
> 漂移以 **「charter tension」** 形式 surface 交人裁决 —— **双向**：可能是支柱跑偏（纠支柱），也可能是研究发现真东西、反过来该更新本 CHARTER。锚，不是紧身衣。

---

## 0. 北极星（shared invariant）

为"**有目标地稳定执行长任务**"的 agent harness 奠定基石。每根支柱深耕一根基石，共同支撑这一总纲。

## 1. 支柱图与共享概念边界（shared invariant）

```
 安全治理(security, 左横轴, 懒创建) ──────────────────────────►
 ┌─────────────────────────────────────────────────────────────┐
 │  data ┐                                                     │
 │       ├─► ontology ─► decision                              │
 │  exec ┘                                                     │
 └─────────────────────────────────────────────────────────────┘
 experience(捕获 → 更新 → 评价, 右横轴) ────────────────────►
```

锁定的不变式（所有支柱共享、不得各自重定义）：

- **资源分两类**：`data`（供给知识）+ `execution`（供给行动力），结构对偶，**各自持有自己的 catalog/元数据**。
- **元数据不是支柱**：它横跨所有资源；支柱归属判据是资源**角色**（知识 vs 行动），**不是**"有没有元数据"。
- **`ontology` = 两类资源 catalog 共同遵循的 schema 层**；懒创建。
- **`experience` = 右轴闭环**：把执行变成可学习介质（捕获流 → 更新权重或 harness → 评价）。**不是**另立 `evolution` 仓；原 evolution 的更新步收进本闭环。
- **`security` = 左横切轴**；懒创建。
- **命名通则**：名字 = 功能，精确 scope 进各支柱 thesis（永不用整体名命名部分；故不用 `agent-harness` 当支柱名）。

**当前 active 支柱恰好 3 个：** `experience` · `decision` · `data`。`execution` / `ontology` / `security` 用到再立。

## 2. 各支柱 excerpt（mandate / 边界 / 接口）

### `experience` —— 右横切轴（由 `trace` 就地升级）

- **Mandate**：生产交互流的捕获、分诊、更新与评价。介质是 agent 自身经验，不是更多人类语料或新 trainer。
- **边界**：管闭环三节（流 / 更新 / 评价）。**不管**知识供给（`data`）、行动力供给（`execution`）、决策编排（`decision`）、world model、inference serving。通用 agentic RL recipe 视为上半场，排除。
- **接口**：横切观测所有纵向支柱；向 decision 回馈经评价验收的策略/harness artifact。
- **GitHub 身份**：同一仓库对象，现名 `research-harness-experience`（原 `research-harness-trace`，star 保留）。

### `decision` —— 纵向

- **Mandate**：决策智能体。给定 data/execution/ontology，决定下一步做什么、如何编排多步执行以稳定推进长任务。
- **边界**：管"决定与编排"；**不管**知识供给、行动力供给、schema、experience 闭环本身。
- **接口**：消费 `data`+`execution`+`ontology`；受 `experience` 观测；其策略可作为更新对象进入 experience。

### `data` —— 纵向·供给

- **Mandate**：数据资源管理。带类型 metadata catalog + 连接/访问，覆盖结构化、非结构化、向量与视图。
- **边界**：管知识供给；不管决策编排、行动力供给、统一 schema 本身、experience 闭环。
- **接口**：catalog 遵循 `ontology`；向 `decision` 供给类型化上下文；受 `experience` 观测；受 `security` 治理。

> `execution` / `ontology` / `security` 的 excerpt 在各自支柱启动研究时补入。
