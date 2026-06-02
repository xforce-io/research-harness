# 集成地图 — agent harness 的基石与支柱协同

> **工程型**活文档(手工维护,**不跑 researcher**)。回答 [`CHARTER.md`](../CHARTER.md) 北极星派生的集成层问题:
> *长任务稳定执行的最小基石组件集是什么?支柱之间的接口/协同模式是什么?哪些必需、哪些可选?*
>
> 与各支柱仓的关系:支柱仓深耕"树木"(单支柱文献与机制),本图缝合"森林"(支柱如何咬合)。各支柱的研究产出若与本图的接口假设冲突,应作为「charter tension」回流更新本图。

## 1. 最小基石组件集(草案,待支柱研究反哺)

一个能"有目标地稳定执行长任务"的 harness,假设以下为**必需**基石:

| 层 | 支柱 | 它解决的"没有它就没基石"的问题 |
|---|---|---|
| 供给 | `data` | 没有受管的知识供给(元数据+连接+视图+非结构化),agent 无法在长任务中稳定取到正确上下文 |
| 供给 | `execution` | 没有受管的行动力供给(MCP/技能/函数/接口注册 + 沙盒),agent 无法安全、可复现地行动 |
| schema | `ontology` | 没有统一 schema,data 与 execution 的 catalog 各说各话,跨步骤语义漂移 |
| 推理 | `decision` | 没有决策/编排核心,多步执行无法朝目标稳定收敛 |
| 反馈 | `evolution` | 没有 artifact 进化,系统不能从执行中学习,长期不改进 |
| 横切 | `trace` | 没有轨迹可观测/诊断,失败不可定位、信号无法回流 evolution |
| 横切 | `security` | 没有端到端安全治理,长任务中的权限/数据/行动失控 |

## 2. 支柱间接口(草案)

```
data.catalog ─┐
              ├─(conform to)─► ontology.schema ─►(typed context)─► decision
execution.registry ─┘                                               │
                                                                     ▼
        trace ─(观测)─► 所有纵向支柱 ─(轨迹/信号)─► evolution ─(改 artifacts)─┐
                                                                              │
                          (回灌 prompts/技能/模型/策略/pattern 到各底座)◄──────┘
        security ─(贯穿治理)─► data访问 · execution行动 · decision授权 · trace脱敏
```

关键接口契约(待各支柱研究确认/反驳):
- **data/execution → ontology**:两类 catalog 的实例必须可被 ontology schema 类型化。
- **ontology → decision**:decision 消费的是经 schema 类型化的上下文,而非裸数据/裸工具。
- **trace → evolution**:轨迹信号需携带足以 hindsight-relabel 的结构(失败模式、friction、可学习性)。
- **evolution → 底座**:进化产物(artifacts)回灌不得破坏 ontology schema 约束。
- **security ⟂ 全栈**:作为横切策略层施加于每个接口,而非单点。

## 3. 待解问题(集成层 open questions)

- 哪些基石是真·必需,哪些在特定场景可省?(最小集的边界)
- `data` 与 `execution` 的 catalog 是否应共享同一 ontology 子集,还是各自子 schema?
- `trace → evolution` 的信号 schema 与 `decision` 的编排 trace 是否同源?
- `security` 横切是策略注入还是独立 gatekeeper 服务?

> 本节随各支柱研究推进持续更新;每次实质更新应在 commit message 注明触发它的支柱发现。
