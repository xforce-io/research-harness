---
title: "Causal Inference with Unstructured Outcomes"
authors: ["Kevin Christian Wibisono","Yixin Wang"]
paper_id: "paper_arxiv_2608_03085"
source_kind: "arxiv"
source_id: "arxiv:2608.03085"
source_url: "https://arxiv.org/abs/2608.03085"
pdf_url: "https://arxiv.org/pdf/2608.03085"
read_id: "read_paper_arxiv_2608_03085"
kind: library-read
doc_type: "paper"
tags: []
---

先读取常驻 Skill 与完整提示，确认 Library 读卡的输出格式与论文正文位置。这是 Library 深读任务，不是 Kairo 主题推进。接下来定位产物路径并读取完整论文正文。目标路径已存在。接下来对照既有 Library 读卡格式，并提取论文全文。提示要求只返回 Markdown 正文、不写文件。正在读取输出规范、样例读卡与论文全文。继续读论文核心章节：识别、估计与实验。论文提取在 Experiment I 中途截断。正在补取 Experiment II/III 与讨论部分的关键结果。已掌握方法、实验与关键数字。按要求只返回 Library 读卡 Markdown 正文。

# Causal Inference with Unstructured Outcomes

> 文本/图像等结果无法相减做 ATE → 在有界打分函数类里用 IPW 对比最大化，学出把处理组与对照组潜在结果拉开最开的标量打分（MCF）→ 再靠高低分样本解读“处理到底改了结果的哪一面”。

## Essence

**问题**：临床笔记、开放问卷、影像这类非结构化结果没法做 \(Y(1)-Y(0)\) 的均值差；先手工 codebook / topic / 某条 embedding 轴再估效应，又容易漏掉真正被处理改变的那一面。

**做法**：先把原始对象嵌成向量 \(Y=\psi(O)\)，再在有界函数类 \(\mathcal{G}\)（常见为带 sigmoid 输出的网络）里找 \(g^\star=\arg\max_g \theta(g)\)，其中 \(\theta(g)=\mathbb{E}[g(Y(1))-g(Y(0))]\)。观测数据上用倾向得分 IPW 目标训练 \(g\)；需要时让 \(g(Y,X)\) 随协变量变化，或同时学处理侧打分 \(f(A)\) 与结果侧打分 \(g(Y)\)（用匹配负对照结果做协变量中心化）。解读靠高低分样本、近邻与 embedding 空间 nudging，而不是事先点名“正式度/模糊度”。

**证据**：GYAFC 正式度半合成实验 Scenario 1 中，学到的分数对非正式句约 0.19–0.21、对正式句约 0.86（两主题皆然），与处理诱导的正式度方向一致 [§4.1.1, Fig.2]。

**边界**：MCF 只在所选表示 \(\psi\) 与函数类 \(\mathcal{G}\) 内找“对比最大的一面”；主实验多为已知 DGP 的半合成/合成设定，不是真实医院笔记的确认性推断。

## Claims

1. 对非结构化结果，通常的 ATE \(\mathbb{E}\{Y(1)-Y(0)\}\) 没有可操作含义，因为文本/图像没有规范的减法与“平均文档”对象 [§1]。
2. 对任意有界特征打分 \(g\)，\(\theta(g)=\mathbb{E}[g(Y(1))-g(Y(0))]\) 是合法的特征特异因果对比；MCF 把“选哪个特征”并入因果查询：\(g^\star\in\arg\max_{g\in\mathcal{G}}\theta(g)\) [§2.1, Eq.1–2]。
3. 在 SUTVA、重叠与无混杂下，每个固定 \(g\) 的 \(\theta(g)\) 可由条件均值差或 IPW 公式识别；若 \(\mathcal{G}\) 内最大化唯一，则 MCF 本身也可识别 [§2.2, Prop.1–2]。
4. Oracle 类 \(0\le g\le 1\) 上，MCF 等价于标记处理提高密度的区域：\(g^\star(y)=\mathbf{1}\{\Delta(y)>0\}\)（\(\Delta=p_1-p_0\)）[§2.2, Prop.2; §2.5]。
5. IPW 目标使倾向模型一次学完后固定，从而避免“\(g\) 一变、条件均值回归目标跟着变”的嵌套优化；在正则条件下 \(\hat\beta\) 一致，并在固定-\(\beta\) IPW 对比有效等条件下渐近正态且半参数有效 [§2.4, Prop.4–5, Alg.1]。
6. 预算版 MCF 按 treatment enrichment score \(\Delta(y)/p_{\mathrm{ref}}(y)\) 排序并截到质量 \(\tau\)，用于先看“处理富集最强”的结果子集 [§2.5, Prop.6]。
7. 协变量适应 MCF \(g(Y,X)\) 最大化 \(\theta_X(g)\)，oracle 为 \(\mathbf{1}\{\Delta(y\mid x)>0\}\)，可描述子群体内不同的结果变化方向 [§2.3, Prop.3; §2.5, Prop.7]。
8. 当处理与结果皆非结构化时，最大化 \(\mathbb{E}[f(A)\{g(Y)-m_g(X)\}]\)（等价于层内 \(\mathrm{Cov}(f(A),g(Y)\mid X)\)）可同时学 MIF–MCF 对；匹配负对照结果可估计该中心化目标而无需每步重拟合 \(m_g\) [§3.1–3.2, Prop.8–9, Eq.18–19]。
9. GYAFC 正式度：同质 MCF 在两主题上均把正式句打到约 0.86、非正式句约 0.19–0.21；异质 Scenario 2 中娱乐主题偏好非正式（约 0.68 vs 0.10），家庭主题偏好正式（约 0.23 vs 0.72）[§4.1.1, Fig.2]。
10. Embedding nudging 沿 \(\hat g\) 梯度移动时，Scenario 1 平均正式度约从 0.15 升至 0.82；Scenario 2 随主题沿相反方向移动 [§4.1.1]。
11. 相对 Egami et al. 的 LDA codebook：\(K=8\) 对比最大，但入选/未入选 topic 无法直接对应正式度等风格维 [§4.1.1]。
12. ParaDetox 非毒性：Scenario 1 有毒/无毒均分约 0.05/0.92 与 0.03/0.91；Scenario 2 在高冲突语境反转（约 0.84 vs 0.06）[§4.1.2, Fig.3]。
13. 多属性文本：Scenario 1 仅 \(F=1,P=1\) 达 0.940；Scenario 2 只抬高标点维；Scenario 3 只抬高正式度维 [§4.1.3, Table 3]。
14. 细胞图像：测试集 \(\hat g<0.2\) 平均 blur 10.0，\(\hat g>0.8\) 为 34.6；nudging 在固定细胞数内容下增大模糊 [§4.2, Fig.4–5]。
15. 双非结构化：合成计划向量只恢复 \(A_1\)–\(Y_1\) 链条；新闻标题设定中 MIF–MCF 在调整主题后恢复 prompt 正式度与标题正式度的对应 [§4.3, Fig.6–7]。

## Assumptions

- 原始对象先经分析者选定的表示 \(\psi\)（句向量、风格表示、图像 style embedding 等）变为 \(Y\)；MCF 只在该表示上寻找对比 [§2.1]。
- 识别依赖经典二元处理假设：SUTVA、重叠、给定 \(X\) 后无混杂 [Assumptions 1–3, §2.2]。
- 搜索类 \(\mathcal{G}_\Theta\)（如固定架构 + sigmoid）足够逼近相关因果方向；有界到 \([0,1]\) 以固定尺度 [§2.2–2.4]。
- 半合成文本实验中，正式度/毒性标签与嵌入足以代表“处理改变的维”；风格与内容可被表示或实验设计大致分离（图像用 adversarial autoencoder 拆 content/style）[§4.1–4.2]。
- 双非结构化设定中，存在（或可匹配近似）负对照结果 \(Y^\dagger\)：与 \(Y\) 同条件于 \(X\) 的分布且给定 \(X\) 与 \(A\) 独立 [§3.2]。
- 解读阶段允许把学到的分数事后命名为“正式度/模糊度”等；论文将算法定位为探索性搜索而非对手工命名属性的确认检验 [§2.4 末]。

## Method

**与旧流程对比**

| | 旧：先 codebook / 固定摘要再估效应 | 本文：MCF |
|---|---|---|
| 特征从哪来 | 事先指定或无监督降维 | 最大化处理诱导的因果对比 |
| 标量从哪来 | 主题比例、某坐标、人工编码 | 有界打分 \(g_\beta(Y)\in[0,1]\) |
| 混杂 | 在固定特征上做标准调整 | 同一套 IPW/层内中心化，对整个 \(\mathcal{G}\) 成立 |

**单侧非结构化结果（§2）**

1. **输入**：i.i.d. \((X_i,A_i,Y_i)\)，\(A\in\{0,1\}\)，\(Y=\psi(O)\) 为嵌入后的结果。
2. **识别目标**：\(\theta(\beta)=\mathbb{E}[g(Y(1);\beta)-g(Y(0);\beta)]\)。
3. **估计**：样本分割/交叉拟合——先估 \(\hat e(X)=\widehat{\mathbb{P}}(A=1\mid X)\)，再在 hold-out 上对
   \(\hat\theta_N(\beta)=\frac1N\sum_i\Big(\frac{A_i g(Y_i;\beta)}{\hat e(X_i)}-\frac{(1-A_i)g(Y_i;\beta)}{1-\hat e(X_i)}\Big)\)
   做随机梯度上升得 \(\hat\beta\) [Alg.1]。
4. **异质**：把 \(g(Y;\beta)\) 换成 \(g(Y,X;\beta)\)。
5. **预算解读**：按 enrichment 排序取质量 \(\tau\) 的高分段，对照低分/近邻/最小对做实质解释。
6. **输出**：\(\hat g\)、高低分样本，以及可选的 nudging 轨迹。

**双侧非结构化（§3）**

1. 参数化 \(f_\gamma(A)\in(\epsilon,1-\epsilon)\)、\(g_\beta(Y)\in(0,1)\)。
2. 最大化 \(\mathbb{E}[f(A)\{g(Y)-m_g(X)\}]\)；用 \(X\)-近邻匹配权重构造 \(\hat b_\beta(X_i)=\sum_j w_{ij}g_\beta(Y_j)\)，联合训练 \((\gamma,\beta)\) [Eq.19]。
3. **输出**：成对打分，分别检视高分处理对象与高分结果对象。

## Eval

- **数据与设定**
  - 文本结果：GYAFC 正式度（半合成，主题为混杂）[§4.1.1]；ParaDetox 毒性/非毒性（半合成）[§4.1.2]；GYAFC + StyleDistance 多属性（正式度×标点）[§4.1.3]。
  - 图像结果：BBBC005v1 细胞染色图（≤40 cells），AAE 得 32 维 style，处理改目标 blur，细胞数为内容/混杂 [§4.2]。
  - 双侧：三维合成 \((A,Y)\) 向量；Qwen2.5-1.5B-Instruct 新闻标题生成 + MiniLM 嵌入，主题混杂 prompt 正式度 [§4.3]。
- **基线/参照**：Egami et al. codebook + LDA（正式度 Scenario 1）；外部 Detoxify 非毒性分数；已知 DGP 的属性标签；nudging（作者前期方法）作方向检查。
- **指标**：hold-out 上按真值标签/语境分层的平均 \(\hat g\)；nudging 后外部属性分数轨迹；图像 blur 与 \(\hat g\) 的关联；双侧设定中 \(\hat f,\hat g\) 对噪声坐标的无关性；标题实验中按 prompt 正式度分层的均分。
- **主要结果**：见 Claims 9–15；讨论称 MCF 能恢复文本/图像上的处理相关特征，并相对 topic 等基线摘要更好 [§5]。

## Weaknesses

1. **跑题例子与实验等级不对齐**：贯穿全文的临床 AI 文书场景几乎未被真实观测笔记检验；§4 全是半合成/合成或受控生成，外推到医院混杂结构缺少直接证据 [§1 vs §4]。
2. **“优于 topic model”论证偏弱**：主文对 LDA 主要是定性“对不上正式度”，缺少与 codebook 效应估计同指标的定量对照；§5 却写 “consistently improves over baseline summaries such as topic models”，强度高于 §4.1.1 所展示的内容 [§4.1.1, §5]。
3. **表示选择把关键因果问题前置**：若 \(\psi\) 混入病情/主题等不可干预内容，MCF 仍可能最大化“表示里最好分的对比”，其科学含义受 \(\psi\) 绑定；论文强调要选对表示，但实验几乎不测错误 \(\psi\) 下的失败模式 [§2.1]。
4. **确认性不足**：明确承认算法是探索性搜索，标签来自事后读样本；主结果却大量报告与真值属性的对齐分数，读者容易当成已确认“学到了正式度本身”而非“学到了与 DGP 对齐的分数” [§2.4, §4.1]。
5. **相关属性不可辨**：文中高斯例子已说明礼貌等与正式度相关的属性可进入同一线性分数；实验未系统设计“伪属性相关、真属性变化”的对抗用例来量化误读风险 [§2.5 vs §4]。
6. **IPW/匹配的实践诊断缺失**：理论依赖重叠与灵活倾向/匹配质量，正文实验几乎不报告 \(\hat e\) 校准、有效样本量、匹配距离敏感性和裁剪规则 [§2.4, §3.2, §4]。
7. **基线不完整**：同目标的 Modarressi et al.（LLM 因果主题、随机实验文本结果）等未在同一任务上对比；图像实验几乎无非 MCF 的风格分数基线 [§1.1, §4.2]。

## Relations

- competes-with Egami et al. 2022 (How to make causal inferences using texts) [high]: 同属文本结果因果；Egami 先固定 codebook 再估计，本文在观测数据上直接优化处理诱导对比，正文明确对照 [§1.1, §4.1.1]。
- builds-on Wibisono and Wang 2026 [high]: nudging 与“非结构化处理 + 标量结果”的前期框架被直接调用；本文补上结果侧特征学习与 MIF–MCF 配对 [§3, §4.1.1, §4.2]。
- extends Feder et al. 2022 / Keith et al. 2020 等 NLP 因果综述脉络 [med]: 既有工作多把文本当混杂/代理/中介，本文把非结构化对象本身当作要刻画其变化的结果 [§1.1]。
- competes-with Modarressi et al. 2025 [med]: 同为从数据中发现文本结果的因果主题/特征；对方偏随机实验 + LLM 描述，本文偏观测 IPW + 嵌入打分优化 [§1.1]。
- orthogonal Schölkopf et al. 2021 因果表示学习 [high]: 论文自划边界——表示如何识别是另一条线；MCF 假定表示已选定后再选因果特征 [§1.1]。
- extends Jeong et al. 2024 (sparse effects in high-dimensional outcomes) [med]: 都想找“被处理动到的低维结构”，但 Jeong 在固定多结果中做稀疏选择，MCF 在嵌入函数类里学可解释打分 [§1.1]。

## Takeaway

- **可借用的方法核**：把“选哪一维结果”写成 \(\arg\max_g\) 的因果对比，并用**固定倾向的 IPW**（或双侧的匹配负对照中心化）避开嵌套回归；预算版 enrichment 排序适合先审阅最尖的变化。
- **需警惕**：半合成对齐分数 ≠ 真实文书上的可解释因果发现；结论强依赖 \(\psi\) 与事后解读，且对 topic codebook 的“系统性优于”证据不足。
- **记住一件事**：非结构化结果上不要硬减两篇文档——先学一个把处理组/对照组潜在结果对比较拉到最大的有界打分，再通过高低分样本读出处理改了什么。
