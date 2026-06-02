# research-harness

> Agent harness 研究矩阵的**超级仓**。把"有目标地稳定执行长任务"所需的基石组件,按 **7 根支柱**拆成平级的独立 [`researcher`](https://github.com/xforce-io/researcher) topic 子仓;本仓只持**全景索引 + 总纲(CHARTER)+ 集成地图 + 控制面板**,通过 git submodule 缝合。

## 为什么是超级仓 + submodule

每根支柱是一个**独立标准 researcher topic 仓**(自己的 `main` / PR / seen-set),保持"窄 thesis、深耕"的研究纪律;超级仓提供全景视角与统一编排,而不把多条研究流塞进一个 git 历史。详见各支柱仓的 `report.md`。

## 支柱矩阵

| # | 支柱 | path | 子仓 | 类型 |
|---|---|---|---|---|
| 1 | 数据资源(结构化+非结构化+视图+自有 catalog) | `data` | `research-harness-data` | 纵向·供给 |
| 2 | 执行资源(MCP/技能/函数/接口+沙盒+自有注册表) | `execution` | `research-harness-execution` | 纵向·供给 |
| 3 | 本体(统一 schema 层) | `ontology` | `research-harness-ontology` | 纵向·schema |
| 4 | 决策智能体 | `decision` | `research-harness-decision` | 纵向 |
| 5 | 自进化(进化底座之上的 artifacts) | `evolution` | `research-harness-evolution` | 纵向·反馈 |
| 6 | 安全治理 | `security` | `research-harness-security` | 横切(左) |
| 7 | trace(轨迹/可观测/诊断) | `trace` | `research-harness-trace` | 横切(右) |

> 已挂载:`trace`、`decision`。其余支柱**开始研究时再懒创建**。

## 锚定与防漂移

[`CHARTER.md`](./CHARTER.md) 是所有支柱的共享锚(总纲 + 共享概念边界 + 各支柱 excerpt)。每根 active 支柱跑研究前,其切片被同步进子仓的 `.researcher/charter.md`,`researcher` 管线读它保持锚定;概念漂移以「charter tension」surface 交人裁决(双向)。

## 控制面板与运行

[`researcher.workspace.yml`](./researcher.workspace.yml) 声明各支柱与 `active` 状态。在本仓根执行:

```sh
researcher run        # workspace 模式:逐个推进 active 支柱、dormant 不碰,各子仓开各自 PR
```

## 克隆

```sh
git clone --recurse-submodules https://github.com/xforce-io/research-harness.git
# 已克隆则:
git submodule update --init --recursive
```

## 集成地图

[`docs/integration-map.md`](./docs/integration-map.md) —— 工程型集成地图:最小基石组件集 + 支柱间接口/协同(手工维护,不跑 researcher)。
