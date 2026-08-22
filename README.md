# research-harness

> Agent harness 研究矩阵的**超级仓**。把「有目标地稳定执行长任务」所需的基石拆成平级的独立 [`researcher`](https://github.com/xforce-io/researcher) topic 子仓；本仓只持**全景索引 + 总纲（CHARTER）+ 集成地图 + 控制面板**，通过 git submodule 缝合。

## 支柱矩阵

当前 **active 恰好 3 个**。其余 CHARTER 支柱懒创建。

| # | 支柱 | path | 子仓 | 类型 |
|---|---|---|---|---|
| 1 | 数据资源 | `data` | `research-harness-data` | 纵向·供给 |
| 2 | 决策智能体 | `decision` | `research-harness-decision` | 纵向 |
| 3 | experience 闭环（捕获→更新→评价） | `experience` | `research-harness-experience` | 横切（右） |

> `experience` 由 `research-harness-trace` **就地升级**（同一 GitHub 仓库对象，star 保留）。旧 URL 仍跳转到该仓。
>
> 懒创建：`execution` · `ontology` · `security`。不单独立 `evolution` 仓（更新步已收进 experience）。

## 锚定与防漂移

[`CHARTER.md`](./CHARTER.md) 是所有支柱的共享锚。每根 active 支柱跑研究前，其切片被同步进子仓的 `.researcher/charter.md`；概念漂移以「charter tension」surface 交人裁决（双向）。

## 控制面板与运行

[`researcher.workspace.yml`](./researcher.workspace.yml) 声明各支柱与 `active` 状态。在本仓根执行：

```sh
researcher run        # workspace 模式: 逐个推进 active 支柱、dormant 不碰
```

## 克隆

```sh
git clone --recurse-submodules https://github.com/xforce-io/research-harness.git
git submodule update --init --recursive
```

## 集成地图

[`docs/integration-map.md`](./docs/integration-map.md) —— 工程型集成地图（手工维护，不跑 researcher）。
