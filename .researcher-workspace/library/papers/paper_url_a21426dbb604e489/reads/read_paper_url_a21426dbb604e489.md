---
title: "Runway News | Introducing Solaris"
authors: []
paper_id: "paper_url_a21426dbb604e489"
source_kind: "url"
source_id: "url:https://runway.com/news/research/introducing-solaris"
source_url: "https://runway.com/news/research/introducing-solaris"
pdf_url: ""
read_id: "read_paper_url_a21426dbb604e489"
kind: library-read
doc_type: "other"
tags: []
---

我先读取完整提示与文档原文，再按要求只产出 Library 阅读笔记正文。# Runway News | Introducing Solaris

> 传统软件必须把视觉设计译成代码再跑 → Runway 提出 Interface World Model（Solaris）逐帧联合渲染与交互 → 去掉中间表示，让界面像活环境一样随意图连续响应。

## Essence

1. **问题** — 现有 OS/应用把“屏幕上画什么”和“交互后发生什么”固定为预先实现的程序；设计稿虽可近乎成品，仍须译成代码等中间表示，既损失视觉保真，又把可能交互空间压成“发货前冻结的有损压缩”。知识系统（搜索/助手）会答但静态；实时系统（JS/游戏引擎/世界模型）会动但不懂任务与产品意图。
2. **做法** — Solaris 作为首个 Interface World Model：基于 Gen-4.5 改造，把点击/拖拽等用户输入当作与文本/图像同类的下一帧条件；自回归逐帧生成 + 少步蒸馏 + 在自身输出上再训，达到可交互延迟；LLM 负责推理（改当前场景 vs 切场景、活态行为、引导 prompt），世界模型负责渲染与即时响应。
3. **证据** — 文中明确交互延迟阈值约 **0.5s**；目标为 **整会话连贯 + 720p 画质**；相对“截图→多模态 LLM 重建界面”，SSIM/DINOv3 显示复杂度上升时信息持续丢失；相对 Claude Opus 5 编码界面，250 人×30 例约 7500 对判：跟指令 61% vs 24%、更自然 71% vs 21% 偏好 Solaris。
4. **边界** — 文中自承：稳定可读文字、信任/幻觉锚定、长会话连贯、无障碍与栈集成仍未解决；公开形态为合作伙伴早期访问，非可复现系统规格。

## Key takeaways

- **负载决策**：界面不再经 code/UI 框架中间表示，而由单一世界模型**联合**生成每一帧与每一次响应对用户输入的结果。
- **架构拆分**：LLM = 推理/状态机意图；世界模型 = 实时渲染与物理感交互；用户动作只作为**已发生**条件进入下一帧（因果、非预知未来）。
- **工程三关**：速度（~0.5s 交互感）、整会话连贯（文字/布局/物体身份）、成本（相对标准视频扩散“数量级更便宜”的厂商主张）。
- **能力主张三件套**：entirely visual（图即应用）、alive（持续演化非等点击）、open-ended（行为由模型能力决定，非预编程工作流）。
- **对比结论可移植**：把界面译成语言/代码再重建会系统性丢信息；“跟指令”与“场景内自然”是可分开测的偏好轴，后者更能拉开生成式世界模型与脚本化 UI 的差距。

## Decisions / claims

- **命名与定位**〔开篇〕：Solaris 是 Interface World Models 家族的第一个模型；核心问题是“OS 在使用中生成 app/网站会怎样”。
- **相对传统 OS**〔开篇〕：传统 OS 规定渲染与动作后果，应用固定直至更新；Solaris **直接渲染该层**，实时、逐帧合成界面。
- **中间表示有损**〔开篇 / Evaluating · The Cost of Translation〕：设计→代码的翻译限制可响应行为，并牺牲视觉保真；“software ships as a lossy compression of the space of possible interactions”。
- **联合生成**〔开篇〕：单一世界模型生成每一帧与对输入的每一响应，消除转换步 → “there's no loss, and the entire frame becomes the interface”。
- **对 agent 训练**〔开篇〕：文本模型在固定编码界面上训练易过拟合布局；Solaris 让 agent 面对持续变化、甚至从未存在过的布局。
- **三能力**〔What's New〕：entirely visual；alive（含自然语言改场景）；open-ended（同场景可支持未预定义行为）。
- **为何现在才有**〔Why Hasn't…〕：知识系统与实时系统长期分裂；Interface World Model 必须**同时**理解意图并连续渲染交互世界。
- **实时化路径**〔How Solaris Works〕：(1) 自回归逐帧 (2) 多步去噪蒸馏为少步 (3) 在快模型自身输出上再训以稳住长交互画质；基座为 Gen-4.5，并延续 GWM-1 路线。
- **交互即条件**〔Learning interaction〕：只见已发生交互，学习动作→视觉结果，无需显式编程点击/拖拽语义；场景内用文本 prompt 规定交互含义。
- **评测主张**〔Evaluating〕：多模态 LLM（含 Claude Fable 5）从单截图重建 30 类界面时，复杂度↑则保真↓；Solaris vs Claude Opus 5 编码结果的用户研究偏好如上。
- **产品愿景**〔New Kinds…〕：应用不再是交互基本单位；店面/教程可按意图与上下文实时生成并保品牌可识别性。

## Constraints & assumptions

- **延迟硬约束**：交互感约在 **半秒** 延迟处崩坏；标准视频扩散“秒到分钟”级不可用。
- **质量目标**：实时交互、整会话连贯、**720p** 视觉质量为构建焦点。
- **因果假设**：训练/推理只条件于已发生交互，从不看未来动作。
- **成本主张依赖厂商工程**：相对标准视频扩散“数量级更便宜”——无独立复现数据。
- **锚定策略**〔What It Can't Do Yet〕：当前靠起始帧（真实产品图/参考）接地；会话中更丰富的已验证上下文仍是研究重点。
- **适用强度自述**：最强于环境动效、点选拖拽、场景切换；文字密集 UI、长会话、无障碍非当前强项。
- **发布形态**：与关键伙伴推进公开发布；文末为 early access 表单——非开源模型卡或基准可复现包。
- **评测范围假设**：重建基准 30 界面；用户研究 250 人、30 交互例——均为公司自报，基线与协议细节未完整公开。

## Open questions

- 实时生成下**稳定、可读文字**如何与连续交互并存（文中提出混合：可接受短暂停顿时用图像模型渲文字页，视频模型管连续交互——仍属开放）。
- **信任**：教学/商业场景中“看起来对但错”的危害如何系统约束；会话中途注入已验证产品数据/文档的机制未落地描述。
- **长会话**视觉与语义连贯的可度量上限与失败模式未给定量结果。
- **无障碍与集成**：与 screen reader / accessibility API 及既有软件栈如何共存，仅列为挑战。
- Interface World Model 相对游戏引擎脚本世界、以及相对“截图+LLM 写代码”路线的**外部**可复现基准仍缺。
- Agent 训练收益（跨布局泛化）为方向性主张，本文未给任务成功率等数字。

## Relations

- extends Runway Gen-4.5 / GWM-1 路线 [med]：文中写明 Solaris 由 Gen-4.5 改造以理解交互并实时响应，并“follows the path we opened with GWM-1”。
- competes-with 截图→多模态 LLM→代码/重建 的界面自动化范式 [med]：重建保真与用户偏好实验直接对打该路径（含 Claude 系列），主张去掉语言/代码中间表示。
- orthogonal 经典实时 UI（JavaScript/CSS、游戏引擎）[low]：文中将其归为“会实时响应但不懂任务/产品意图”的一极；Solaris 声称合并知识与实时，但是否在延迟/确定性上替代引擎未实证。
- builds-on 交互式世界模型 / 视频世界模型文献脉络 [low]：产品叙事把 Interface World Model 定义为“既是确定性软件界面又是视觉世界生成器”，与通用 world model 工作同族但目标从内容生成转向 OS 级界面层。

## Takeaway

- **可抄**：交互作下一帧条件 + LLM 推理与世界模型渲染分离；评测拆成“跟指令”与“场景内自然”两轴，并正视 ~0.5s 延迟墙。
- **宜疑**：数量级成本优势、agent 训练泛化、以及“无中间表示故无损失”在商业/文字 UI 上的外推——多为厂商主张或小规模自报研究。
- **记忆钩**：Solaris = 把 OS 的界面层做成实时 Interface World Model：帧即应用，点拖即条件，代码中间层被拿掉。
