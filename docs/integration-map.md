# 集成地图 — agent harness 的基石与支柱协同

> **工程型**活文档（手工维护，**不跑 researcher**）。回答 [`CHARTER.md`](../CHARTER.md) 北极星派生的集成层问题。

## 1. 最小基石组件集

| 层 | 支柱 | 它解决的「没有它就没基石」的问题 | 状态 |
|---|---|---|---|
| 供给 | `data` | 长任务中稳定取到正确上下文 | active |
| 推理 | `decision` | 多步执行朝目标收敛 | active |
| 闭环 | `experience` | 交互流成为可学习介质（捕获→更新→评价） | active |
| 供给 | `execution` | 受管行动力（MCP/技能/沙盒） | 懒创建 |
| schema | `ontology` | 两类 catalog 的 schema | 懒创建 |
| 横切 | `security` | 权限 / 数据 / 行动治理 | 懒创建 |

更新步（改权重或 harness）在 `experience` 闭环内，不另立 `evolution` 支柱。

## 2. 支柱间接口

```
data.catalog ─┐
              ├─(conform to)─► ontology.schema ─►(typed context)─► decision
execution.registry ─┘                                               │
                                                                     ▼
        experience ─(捕获/分诊)─► 更新(权重|harness) ─(评价验收)─┐
                                                                 │
                    （回灌策略/harness 到 decision 等底座）◄──────┘
        security ─(贯穿治理)─► data访问 · execution行动 · decision授权 · experience脱敏
```

关键接口契约：
- **data/execution → ontology**：两类 catalog 可被 schema 类型化。
- **ontology → decision**：decision 消费类型化上下文。
- **experience 闭环**：轨迹须携带足以分诊与 hindsight 的结构；更新必须声明改权重还是改 harness；评价必须能对照采样基线。
- **security ⟂ 全栈**：横切策略层。

## 3. 待解问题

- 哪些基石是真·必需，哪些可省？
- `data` 与 `execution` 的 catalog 是否共享同一 ontology 子集？
- experience 的流 schema 与 decision 的编排 trace 是否同源？
- security 是策略注入还是独立 gatekeeper？
