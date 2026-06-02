# GOAL — agent harness 研究矩阵重构

> 本文件是这次重构的**总纲与验收基准**。跨两个 issue:
> - 结构侧:[xforce-io/agent-harness-research#11](https://github.com/xforce-io/agent-harness-research/issues/11)
> - 工具侧:[xforce-io/researcher#9](https://github.com/xforce-io/researcher/issues/9)
>
> 状态图例:⬜ 未开始 · 🔄 进行中 · ✅ 完成

---

## 1. 终态愿景(为什么)

把单话题研究仓升级为**多支柱研究矩阵**:一个能"有目标地稳定执行长任务"的 agent harness 需要多根基石支柱协同。要既保持每根支柱**窄 thesis、深耕**的研究纪律,又提供**全景视角**,并用 **CHARTER 锚定防止支柱漂移**。

**终态架构**:`research-harness` submodule 超级仓,挂 7 根平级支柱子仓;每个子仓是独立标准 researcher topic 仓(自己的 main/PR/seen-set);超级仓持 `CHARTER.md` 总纲 + 集成地图 + `researcher.workspace.yml` 控制面板;`researcher` 工具新增 workspace 感知的 run,在超级仓根一次推进所有 active 支柱、自动同步 charter、surface 漂移张力。

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

### 7 支柱矩阵

| # | 支柱 | path | repo | 首批状态 |
|---|---|---|---|---|
| 1 | 数据资源(结构化+非结构化+视图+自有 catalog) | `data` | `research-harness-data` | 懒创建 |
| 2 | 执行资源(MCP/技能/函数/接口+沙盒+自有注册表) | `execution` | `research-harness-execution` | 懒创建 |
| 3 | 本体(统一 schema 层) | `ontology` | `research-harness-ontology` | 懒创建 |
| 4 | 决策智能体 | `decision` | `research-harness-decision` | **首批挂载** |
| 5 | 自进化(进化底座之上的 artifacts) | `evolution` | `research-harness-evolution` | 懒创建 |
| 6 | 安全治理(左横轴) | `security` | `research-harness-security` | 懒创建 |
| 7 | trace(轨迹/可观测/诊断,右横轴) | `trace` | `research-harness-trace` | **首批挂载(本仓改名而来)** |

### 概念边界(已锁定的不变式)
- 资源分两类:`data`(供给知识)+ `execution`(供给行动力),对偶,**各持自有 catalog/元数据**。
- 元数据**非独立支柱**(横跨所有资源);判据是资源**角色**(知识 vs 行动)。
- `ontology` = 两套 catalog 共同遵循的 **schema 层**。
- `evolution` = 进化架在 data/execution/ontology/decision **之上的 artifacts**,由 trace 信号驱动。
- `security` / `trace` = 左右两条端到端**横切轴**。
- 命名通则:**名字=功能,精确 scope 进 thesis**(故 data≠resource、decision≠agent、evolution≠artifact-evolution —— 不用整体名命名部分)。

---

## 2. 工作分解

### 阶段 0 — 基础改名与建仓(线 A · 难回退 · 需逐条确认)
- ⬜ `agent-harness-research` → 改名 `research-harness-trace`(纯 GitHub 侧,本地工作树不受影响)
- ⬜ `research-decision-agent` → 改名 `research-harness-decision`
  - ⚠️ 注意:该仓当前工作树脏、停在 `researcher/19_*` 在途分支(上次 tick 未收尾)。改名不影响它,但那次 tick 需另行收尾。
- ⬜ 新建空壳超级仓 `research-harness`(PUBLIC)
- ⬜ 本地卫生(可回退):更新两改名仓 remote URL;本地目录名对齐

**验收**:三仓在 GitHub 各就各位、目标名生效;两改名仓本地仍可正常 push/pull(重定向 or 已更新 URL);`research-harness` 空仓可克隆。

### 阶段 1 — 超级仓骨架(线 A · 无代码)
- ⬜ `CHARTER.md` —— 共享 invariants 核心 + 每支柱 excerpt(首批写 trace/decision 两节)
- ⬜ `README.md` —— 全景图 + 各支柱链接 + 矩阵表
- ⬜ `docs/integration-map.md` —— 工程型集成地图(最小基石 + 支柱间接口/协同;手工维护,不跑 researcher)
- ⬜ `researcher.workspace.yml` —— 控制面板(首批:trace=active,decision=active 或 dormant 由首轮验证需要定)
- ⬜ 挂 submodule:`trace`、`decision`
- ⬜ 各支柱 `.researcher/` 现状确认(trace 已有泛化后的 thesis/project.yaml;decision 有自己的)

**验收**:`git clone --recurse-submodules research-harness` 后能看到 trace/decision 两个子仓内容齐全;`CHARTER.md` 结构正确(共享核心 + 两节 excerpt);workspace.yml 可被解析。

### 阶段 2 — `researcher` 能力增强(线 B · 有代码 · researcher#9)
按你的规范在 `researcher` 仓走 `feat/9-workspace-run` 分支 + `docs/design/9-workspace-run.md`。
- ⬜ `src/workspace/manifest.ts` —— manifest parser + schema 校验
- ⬜ run 探测分支(`.researcher/` → 单话题;`researcher.workspace.yml` → workspace;都无 → 报错)
- ⬜ `src/workspace/orchestrator.ts` —— 串行循环 + 错误隔离 + run 前 charter sync + 汇总表
- ⬜ `src/workspace/charter.ts` —— 从 `CHARTER.md` 抽共享核心 + 指定支柱节(切片)
- ⬜ core 注入:`src/pipeline/discover_triage.ts`、`synthesize.ts` prompt 加 `charter` 字段(缺失兼容旧仓)
- ⬜ 漂移 surface:synthesize 扩展"against thesis"→"也 against charter",标「charter tension」,soft 不 block
- ⬜ 测试 + 设计文档
- ⬜ PR → review → merge

**验收**:`researcher run` 在普通 topic 仓行为不变(回归);在超级仓根进入 workspace 编排;有 charter 的仓其 prompt 含 charter;单元测试覆盖 manifest 解析/探测/切片/错误隔离。

### 阶段 3 — 新架构下跑一轮研究,结果符合预期(端到端验收)
在 `research-harness` 超级仓根执行 `researcher run`:
- ⬜ workspace 模式被正确识别,只推进 active 支柱、dormant 完全不碰
- ⬜ 每个 active 支柱跑前,`CHARTER.md` 切片被同步进该子仓 `.researcher/charter.md`(头标 AUTO-SYNCED)
- ⬜ 每个 active 支柱在**自己子仓**开出研究 PR(或优雅报告"无新论文")
- ⬜ 各子仓 PR 互不干扰(独立 main/分支)
- ⬜ 若有与 charter 的张力,在 report 中作为「charter tension」surface 出来
- ⬜ 末尾汇总表准确反映每个支柱的结果 + PR 链接

**最终验收(结果符合预期)**:
1. 子仓产出的 note/report **紧扣本支柱 thesis**,未跑偏到别的支柱;
2. 产出**被 charter 锚定**(概念用法与共享定义一致);
3. 任何概念漂移都被 surface、未被静默吞掉;
4. 整个流程**一条命令、串行、可读汇总**,无需逐个 cd 进子模块;
5. dormant 支柱零改动。

---

## 3. 顺序与依赖

```
阶段0(改名建仓) ──► 阶段1(超级仓骨架) ──┐
                                          ├──► 阶段3(端到端验收)
阶段2(researcher 能力增强,可独立开发)──┘
```

- 阶段 2 可与 0/1 **并行开发**,但**验收(阶段3)需 0/1 产物就位**。
- 建议执行序:阶段 0 → 阶段 1 → 阶段 2 → 阶段 3。

## 4. 不做(YAGNI)
- 不一次性创建全部 7 个空仓(懒加,用到再建)。
- 集成地图是工程文档,**不跑 researcher**。
- 不在超级仓提交研究产物 / 不 bump submodule 指针 / 不开超级仓 PR(指针同步留未来显式步骤)。
- 漂移仅 surface,不自动 reconcile、不做版本号校验。

## 5. 风险 / 注意
- **二次改名**(本仓上轮刚从 `research-agent-triage` 改名):GitHub 自动重定向旧名;`researcher` README 内对旧名的链接需同步更新。
- `research-harness-decision` 改名时正处一次未收尾的 tick(在途 `researcher/19_*` 分支)—— 改名安全,但该 tick 需另行清理。
- submodule 指针同步的心智负担(子 PR 合并后 `git submodule update --remote`)。

## 6. 关联
- 结构侧 issue:agent-harness-research#11
- 工具侧 issue:researcher#9
- 本 goal.md 将来迁入 `research-harness` 超级仓根(或 `docs/`)作为活文档。
