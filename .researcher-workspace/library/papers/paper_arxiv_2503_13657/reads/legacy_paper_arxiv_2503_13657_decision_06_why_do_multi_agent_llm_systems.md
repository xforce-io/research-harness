---
title: "Why Do Multi-Agent LLM Systems Fail?"
paper_id: "paper_arxiv_2503_13657"
source_kind: "arxiv"
source_id: "arxiv:2503.13657"
source_url: "https://arxiv.org/abs/2503.13657"
kind: legacy-topic-read
topic: "decision"
source_note: "notes/active/06_why_do_multi_agent_llm_systems.md"
---

# Legacy topic read

This Library read artifact was backfilled from an existing topic note. It preserves historical reading state; it was not produced by the current Library deep-read runner.

## Source topic note

---
zone: active
pin: false
score: 0
dwell: 0
paper: "Why Do Multi-Agent LLM Systems Fail?"
arxiv: "2503.13657"
authors: ["Mert Cemri", "Melissa Z. Pan", "Shuyi Yang", "Lakshya A. Agrawal", "Bhavya Chopra", "Rishabh Tiwari", "Kurt Keutzer", "Aditya Parameswaran", "Dan Klein", "Kannan Ramchandran", "Matei Zaharia", "Joseph E. Gonzalez", "Ion Stoica"]
year: 2025
note_number: 6
---

## Claims

- 7 popular open-source MAS frameworks exhibit 41% to 86.7% failure rates on their respective benchmarks (e.g., AppWorld 86.7% failure on Test-C, MetaGPT 60% failure on ProgramDev) [5: §1, Figure 5].
- MAS failures cluster into 3 categories with 14 fine-grained failure modes (FMs); the category breakdown over 1642 traces is 44.2% System Design Issues (FC1), 32.3% Inter-Agent Misalignment (FC2), 23.5% Task Verification (FC3) [5: Figure 1, §4].
- The three single most prevalent failure modes are FM-1.3 Step Repetition (15.7%), FM-2.6 Reasoning-Action Mismatch (13.2%), and FM-1.5 Unaware of Termination Conditions (12.4%) [5: Figure 1].
- Three independent expert annotators reach Cohen's κ = 0.88 inter-annotator agreement using MAST on 15 traces from 5 MAS frameworks; out-of-domain validation on Magentic-One/OpenManus + GAIA/MMLU yields κ = 0.79 [5: §3.2, §3.4].
- An OpenAI o1-based LLM-as-Judge annotator with few-shot examples achieves accuracy 0.94, F1 0.80, Cohen's κ = 0.77 against expert human annotations [5: §3.3, Table 2].
- Inter-MAS failure correlations are low (0.17–0.32 between the three categories), supporting the taxonomy's category distinctiveness; inter-mode correlations max out at 0.63 (FM-2.1 vs. FM-1.4) [5: §E, Figure 6, Figure 7].
- Targeted system-design interventions guided by MAST yield +9.4% absolute success rate improvement on ChatDev (role specification fix for CPO/CEO conflict) and +15.6% on ChatDev ProgramDev (adding high-level objective verification step) — same underlying GPT-4o model [5: §1, §4 FC3, Appendix H].
- LLM-choice effect on MetaGPT: GPT-4o produces 39% fewer FC1 failures than Claude 3.7 Sonnet on programming tasks [5: §5.1, Figure 8].
- MAS-architecture effect (same GPT-4o): MetaGPT has 60–68% fewer FC1/FC2 failures than ChatDev but 1.56× more FC3 failures, attributed to ChatDev's explicit testing/review phases [5: §5.1, Figure 9].
- Systems with explicit verifier agents (MetaGPT, ChatDev) show fewer total failures than systems without, but verifier presence alone is not sufficient — most existing verifiers perform only superficial checks (e.g., compile-success) [5: §4 FC3, Insight 3].

## Assumptions

- "Failure" is defined as the MAS not achieving its intended task objective; this is a binary judgment grounded in benchmark task-completion rates rather than partial credit or graceful-degradation metrics [5: §1, Table 1 HE column].
- The Grounded Theory saturation criterion ("further data analysis does not yield new failure mode insights") is treated as having been met after 150 traces from 5 frameworks; the paper does not quantify saturation rate or false-negative risk for failure modes that exist but did not appear in the initial sample [5: §3.1].
- The LLM annotator's reported κ = 0.77 generalizes from the IAA hold-out set to the full 1642-trace MAST-Data — i.e., LLM annotation quality on unseen traces is assumed comparable to validation-set quality [5: §3.3, §3.4].
- The 14 failure modes are mutually distinguishable in principle, despite the paper acknowledging that surface symptoms can be ambiguous (e.g., missing information could be FM-2.4 withholding, FM-2.5 ignored input, or FM-1.4 history loss) [5: §4 FC2].
- Workflow-level interventions improving FM-specific counts translate to overall task-success improvement; the case studies measure both, but the paper does not address the reverse case of interventions that fix one FM but introduce another [5: §5.2, Appendix H.3].

## Method

**Data collection (§3.1).** Apply Grounded Theory to 150 traces (avg. ~15,000 lines each) from 5 MAS frameworks (HyperAgent, AppWorld, AG2, ChatDev, MetaGPT) over coding + math tasks. Six expert annotators perform open coding, constant comparative analysis, memoing, and theorize until theoretical saturation. ~20 hours/expert.

**Taxonomy validation (§3.2).** Three rounds of Inter-Annotator Agreement: 3 annotators independently label 5 random traces using preliminary MAST, then resolve disagreements (10 hours total) and refine definitions. Iterate until Cohen's κ = 0.88.

**MAST structure (§4).** 14 FMs in 3 categories, mapped to MAS execution stages (Pre-Execution, Execution, Post-Execution):
- FC1 *System Design Issues*: FM-1.1 disobey task spec, FM-1.2 disobey role spec, FM-1.3 step repetition, FM-1.4 history loss, FM-1.5 unaware of termination.
- FC2 *Inter-Agent Misalignment*: FM-2.1 conversation reset, FM-2.2 fail to ask clarification, FM-2.3 task derailment, FM-2.4 information withholding, FM-2.5 ignored input, FM-2.6 reasoning-action mismatch.
- FC3 *Task Verification*: FM-3.1 premature termination, FM-3.2 no/incomplete verification, FM-3.3 incorrect verification.

**Scalable annotation (§3.3).** OpenAI o1 LLM-as-Judge prompted with execution trace + MAST definitions + few-shot examples, validated on held-out IAA set.

**Generalization check (§3.4).** Apply finalized MAST + LLM annotator to 2 unseen MAS (Magentic-One, OpenManus) and 2 unseen benchmarks (MMLU, GAIA). Confirm κ = 0.79.

**Full dataset construction.** Annotate 1642 traces across 7 MAS × multiple LLMs (GPT-4o, GPT-4, Claude-3.7-Sonnet, GPT-4o-mini, Qwen2.5-Coder-32B, CodeLlama-7B) × benchmarks (ProgramDev, ProgramDev-v2, SWE-Bench Lite, Test-C, GSM-Plus, GAIA, OlympiadBench, GSMPlus, MMLU). Release as MAST-Data + MAST-Data-human + `pip install agentdash`.

**Case-study interventions (Appendix H).** Two ChatDev interventions on ProgramDev: (1) role-specification clarification ensuring CEO has final say (targets FM-1.2); (2) inserting a high-level task objective verification step (targets FM-3.2). Measure delta in pass rate.

## Eval

**Datasets:** ProgramDev (30 coding tasks: Tic-Tac-Toe, Chess, Sudoku, etc.), ProgramDev-v2 (100 tasks), SWE-Bench Lite (30), AppWorld Test-C (30), GSM-Plus (30), GAIA (30, 165 expansion), OlympiadBench (206), MMLU (168), GSMPlus (193). Total 1642 traces.

**MAS:** ChatDev, MetaGPT, HyperAgent, AppWorld baseline, AG2 (MathChat), Magentic-One, OpenManus.

**Models:** GPT-4o (primary), GPT-4, Claude-3.7-Sonnet, GPT-4o-mini, Qwen2.5-Coder-32B-Instruct, CodeLlama-7b-Instruct.

**Metrics:** Task completion rate (HE — human-evaluated); failure mode distribution (HA = human annotated, LA = LLM annotated); Cohen's κ for annotation agreement; case-study Δ success rate.

**Baselines:** No new MAS proposed for SOTA comparison — the paper itself is a measurement study. Annotator baselines: o1 zero-shot (κ = 0.58) vs. o1 few-shot (κ = 0.77).

**Key results:**
- 7 MAS failure rate spread: AppWorld 86.7% > MetaGPT 60% > ChatDev 66.7% > HyperAgent 74.7% > AG2 41% > Magentic-One 62%.
- MAST category split aggregate: 44.2% / 32.3% / 23.5% (FC1/FC2/FC3).
- Per-system failure profile heterogeneity (Figure 4): AppWorld dominated by FM-3.1 premature termination; OpenManus dominated by FM-1.3 step repetition; HyperAgent dominated by FM-1.3 + FM-3.3.
- Case studies: ChatDev ProgramDev role-spec fix +9.4%; verification-step addition +15.6%.

## Weaknesses

- **No causal claim, only co-occurrence.** MAST identifies which failure modes are present in failed traces but does not establish that any specific FM caused the task to fail — multiple FMs co-occur in most traces and the paper provides no counterfactual analysis (e.g., would removing FM-2.4 alone fix the run?). The 9.4% / 15.6% interventions are encouraging but represent only two data points and target different FMs in different ways.
- **LLM-as-Judge κ = 0.77 is "moderate-to-substantial" for the binary FM-presence task, but per-mode breakdown is omitted.** The paper reports overall κ but not per-FM κ. Modes with similar surface symptoms (e.g., FM-2.4 vs. FM-2.5, correlation 0.46 in Figure 7) are likely conflated by the LLM annotator at much higher rates than the aggregate suggests, undermining the fine-grained decomposition's reliability for the modes that matter most for diagnosis.
- **Theoretical-saturation claim is unverifiable.** Reaching saturation after 150 traces from 5 MAS does not guarantee coverage of failure modes specific to MAS variants the authors did not sample (e.g., enterprise-grade MAS with safety/audit gating, multi-session agents, agents with private memory). The follow-up out-of-domain validation only confirms the existing taxonomy applies, not that no new modes exist.
- **FC3 / verifier finding partially circular.** "MAS with explicit verifiers (MetaGPT, ChatDev) have fewer total failures" is consistent with the architectural definition: a verifier can catch FC1/FC2 issues before they become observed terminal failures, redirecting them into FC3. This means the FC3 vs. FC1/FC2 ratio is partly an artifact of where in the pipeline the failure surfaces, not a clean signal about which root causes are most important.
- **Intervention case studies use the same LLM-as-Judge that produced the failure annotations to verify the fix.** Both pre-intervention failure counts and post-intervention success measurement involve the same LLM-driven evaluation pipeline. If the intervention happens to address surface symptoms the LLM annotator is sensitive to, the measured improvement may overstate real-world correctness. No independent human re-evaluation of post-intervention traces is reported.
- **Step-repetition (FM-1.3) and reasoning-action mismatch (FM-2.6) prevalence may be MAS-class artifacts.** Both modes are over-represented in iterative-loop frameworks (HyperAgent, OpenManus) and may not generalize to MAS designs without retry-loops or scratchpad reasoning. The paper's aggregated 15.7% / 13.2% rates conflate architecture-specific tendencies with MAS-general failure patterns.
- **No analysis of failure interaction with task complexity.** ProgramDev tasks are deliberately "relatively straightforward" (§D); SWE-Bench Lite, AppWorld Test-C are harder. The paper does not separate failure-mode prevalence by task difficulty, so we cannot tell whether FC1 dominance is a feature of easy-task framing or a generalizable property — important for Decision Agent enterprise long-chain tasks where complexity is structurally higher.
- **Closed-source MAS (Manus, others) deliberately excluded.** Manus achieves 60% on ProgramDev but is excluded from MAST-Data due to opaque traces. Production decision-agent platforms are increasingly closed; MAST's diagnostic value for the systems most likely to matter commercially is unverified.

## Relations

- **builds-on 04_rethinking_the_value_of_multi_agent [high]:** Both papers attack the "MAS underperforms expectations" puzzle. OneFlow [4] shows for *homogeneous* MAS that single-agent simulation matches performance — implying multi-agent structure doesn't help. MAST [5] explains *why* MAS underperforms by cataloging where it breaks. The two are highly complementary: OneFlow says "the multi-agent layer adds no accuracy"; MAST says "the multi-agent layer adds 14 categories of breakage." OneFlow's simulator removes FC2 by construction (no inter-agent communication channel) and reduces FC1 (single termination point). MAST §1 explicitly cites Kapoor et al. (the same lineage as OneFlow's argument) when motivating the failure study.
- **competes-with 02_graph_of_agents_a_graph_based [med]:** GoA [2] proposes a graph-structured multi-agent routing architecture as the path to better MAS performance. MAST [5] documents that graph-routing systems suffer from FM-2.3 task derailment (7.4%) and FM-2.6 reasoning-action mismatch (13.2%) — failure modes that scale with routing complexity. MAST does not directly evaluate GoA but provides the analytical framework under which GoA's architectural complexity must be justified: each added routing edge is a new opportunity for FC2 failures. The relation is "competes" in the sense that one prescribes architectural complexity and the other documents its costs.
- **extends 03_why_reasoning_fails_to_plan [med]:** Note 3 [3] argues that step-wise reasoning is structurally insufficient for long-horizon planning. MAST FM-1.5 (Unaware of Termination Conditions, 12.4%) and FM-3.1 (Premature Termination, 6.2%) are concrete instantiations of the planning-depth limit at the system-execution layer. MAST does not formally connect to planning theory, but the empirical prevalence of termination-related failures across all 7 MAS supports note 3's claim that the bottleneck is planning mechanism, not communication topology.
- **orthogonal to 01_co_evolving_llm_decision_and_skill [med]:** COS-PLAY [1] addresses skill-decision co-evolution (a heterogeneous multi-agent training-time problem); MAST [5] addresses runtime failure diagnosis. They share the broader theme of "the multi-agent system is not just the sum of its agents" but operate in different regimes — COS-PLAY treats agents as separately optimizable artifacts; MAST treats them as instances of a fixed configuration. Both are needed for a full picture of Decision Agent reliability.
- **contradicts thesis §1 partially [med]:** The thesis posits that current Decision Agent technology stack (BKN + ContextLoader + Dolphin + ISF) "解决了上下文管理和语义对齐问题, 但 workers 之间的协同失真、长链规划退化和多智能体调度仲裁，仍缺乏学术层面的系统性验证." MAST provides exactly the systematic validation the thesis says is missing — for the *coordination* dimension. FC2 (Inter-Agent Misalignment, 32.3% of all failures) directly measures the "协同失真" the thesis identifies as a known engineering pain point. The contradiction is partial and constructive: MAST does not refute the thesis but converts an "untested concern" into a "measured prevalence" — Decision Agent's success criterion can now be benchmarked against MAST FC2 sub-rates.
