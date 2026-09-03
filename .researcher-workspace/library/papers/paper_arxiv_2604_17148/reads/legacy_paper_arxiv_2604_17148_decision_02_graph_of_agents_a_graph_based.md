---
title: "Graph-of-Agents: A Graph-based Framework for Multi-Agent LLM Collaboration"
paper_id: "paper_arxiv_2604_17148"
source_kind: "arxiv"
source_id: "arxiv:2604.17148"
source_url: "https://arxiv.org/abs/2604.17148"
kind: legacy-topic-read
topic: "decision"
source_note: "notes/active/02_graph_of_agents_a_graph_based.md"
---

# Legacy topic read

This Library read artifact was backfilled from an existing topic note. It preserves historical reading state; it was not produced by the current Library deep-read runner.

## Source topic note

---
zone: active
pin: false
score: 0
dwell: 0
paper: "Graph-of-Agents: A Graph-based Framework for Multi-Agent LLM Collaboration"
arxiv: "2604.17148"
authors: ["Sukwon Yun", "Jie Peng", "Pingzhi Li", "Wendong Fan", "Jie Chen", "James Zou", "Guohao Li", "Tianlong Chen"]
year: 2026
venue: "ICLR 2026"
note_number: 2
---

## Claims

- GoA using only 3 agents outperforms MoA and all other multi-agent baselines using 6 agents on most benchmarks: GoAMax achieves MMLU 79.18 vs. MoA 75.71, MMLU-Pro 54.78 vs. MoA 53.33, MedMCQA 60.04 vs. MoA 54.94; GoAMean achieves GPQA 40.54 vs. MoA 32.83 and MATH 73.12 vs. MoA 65.80 [2: §4.2, Table 1].
- GoA reduces inference cost by ~58%: 11 LLM calls vs. MoA's 19, 19k tokens vs. 56k, 100s latency vs. 240s, while improving accuracy on MMLU-Pro (54.78 vs. 53.33) [2: §4.2, Table 2].
- Relevance-directed message passing is the decisive mechanism: reversing the flow direction causes the largest performance drop (–2.60 MMLU-Pro, –5.05 GPQA), larger than removing either directional pass individually [2: §4.4, Table 5].
- MoA is a degenerate special case of GoA: GoA reduces to MoA when k=N, the adjacency matrix is fully connected with uniform weights, and mean pooling is applied [2: §3.3, Proposition 1].
- Node sampling via model cards prevents irrelevant agents from injecting noise: in the anatomy case study, MoA's inclusion of math and code agents produces wrong answers while GoA's graph excludes them and reaches the correct answer [2: §4.3, Figure 3].
- Edge scoring with relevance weighting (Aij normalized by neighbor scores) consistently outperforms uniform edge weights (Aij = 1): +1.87 MMLU-Pro, +2.64 GPQA [2: §4.4, Table 5].

## Assumptions

- Hugging Face model card READMEs are sufficient to characterize each agent's domain and task specialization for selection purposes; the meta-LLM reads these summaries without ground-truth capability evaluation [2: §3.2.1].
- Peer-ranking — each agent scores the responses of all other agents — produces calibrated relevance estimates; the paper excludes self-scoring to reduce self-bias but does not verify that cross-agent scoring tracks actual correctness [2: §3.2.2].
- A static pool of 6 domain-specialized 7–8B LLMs is sufficiently diverse to benefit from graph-structured collaboration; the agent pool composition is fixed and not optimized per query [2: §4.1].
- Benchmark tasks (MMLU, MATH, HumanEval, MedMCQA, GPQA) are representative of the multi-domain decision problems GoA targets; no agentic or tool-use tasks are included [2: §4.1].
- The threshold τ=0.05 for edge pruning generalizes across task types; this value is fixed across all benchmarks without per-domain tuning rationale [2: §3.2.2].

## Method

GoA frames multi-agent collaboration as a directed graph G=(V,E) over selected agent nodes with relevance-weighted edges. Four stages:

**❶ Node Sampling.** A meta-LLM (Qwen2.5-7B-Instruct in experiments) reads model cards summarizing each agent's domain, task specialization, and size, then selects top-k agents most aligned with the query. This forms the active node set Vs ⊆ V with |Vs| = k.

**❷ Edge Sampling.** Each selected agent generates an initial response. Each agent then scores every other agent's response (normalized to sum 1.0); scores are summed per agent to produce a relevance score Sj. Agents below threshold τ=0.05 are pruned. The weighted adjacency matrix Aji = Si / Σ_{k∈Nj} Sk encodes directed influence from source (high Sj) to target (low Sj) nodes.

**❸ Message Passing (bidirectional).**
- *Source-to-Target*: Higher-ranked (source) agents propagate their responses to lower-ranked (target) agents, which refine their answers incorporating weighted neighbor context.
- *Target-to-Source*: Source agents then update using the refined target responses, incorporating neighborhood consensus.

**❹ Graph Pooling.** Two variants: GoAMax selects the refined response of the most-connected source node (highest incoming edge count); GoAMean passes all refined responses through the meta-LLM for weighted synthesis. Both avoid the O(LNd) token stacking cost of MoA.

## Eval

**Benchmarks:** MMLU, MMLU-Pro, GPQA (multi-domain); MATH, HumanEval, MedMCQA (domain-specific). MMLU uses 50 stratified samples/category (57 categories); MMLU-Pro uses 150/category (14 categories).

**Agent pool:** 6 LLMs at 7–8B parameters: Qwen2.5-7B-Instruct (General), Qwen2.5-Coder-7B-Instruct (Code), Mathstral-7B-v0.1 (Math), Bio-Medical-Llama3-8B (Biomedical), finance-Llama3-8B (Finance), Saul-7B-Instruct-v1 (Legal).

**Baselines:** Single-agent (each of the 6 specialist LLMs); multi-agent with 6 agents: Debate, Self-Consistency, Refine, ReConcile, MoA, Self-MoA.

**Metrics:** Zero-shot CoT accuracy. Efficiency: LLM call count, total token usage (input+output), wall-clock latency.

**Key results:**
- GoAMax: MMLU 79.18 (+1.04 vs. Self-MoA), MMLU-Pro 54.78 (+0.59 vs. Self-MoA), MedMCQA 60.04 (+4.48 vs. Self-MoA). GoAMean: GPQA 40.54 (+6.70 vs. Self-MoA), MATH 73.12 (+4.92 vs. Self-MoA).
- Scaling to gpt-4o pool (Table 3): GoAMax (3 agents) beats DyLAN (8 agents) on GPQA (55.05 vs. 58.89 — DyLAN wins here), but outperforms on HumanEval (93.29 vs. 92.07) and matches on MedMCQA.
- Fixed-node relevance-passing ablation (Table 4): GoAMax 85.98 on HumanEval vs. MoA 85.37, Debate 71.95, Reconcile 80.61.

## Weaknesses

- **N² edge sampling cost is not analyzed at scale.** Edge sampling requires each of the k selected agents to score all k−1 others, giving O(k²) scoring calls. For k=3, this is tractable. The paper claims GoA "scales to large agent pools," but no experiment uses more than 6 agents in the pool and the O(k²) scaling behavior is not measured. The scalability claim is theoretical, not empirical.
- **Model card quality is uncontrolled and potentially misleading.** The meta-LLM reads Hugging Face READMEs to select agents. These documents are author-written marketing material and vary widely in quality and accuracy. No ablation compares model-card-based selection against a task-aware oracle or random selection; the selection mechanism's contribution to performance is conflated with the message-passing mechanism.
- **Benchmarks are closed-form QA; no agentic evaluation.** All six benchmarks have ground-truth answers and measure single-turn reasoning. GoA's claimed applicability to "multi-agent LLM ecosystems" and "collective intelligence" is not demonstrated on tool-use, planning, or iterative decision tasks — the setting most relevant to decision agent deployment.
- **Peer-ranking calibration is not validated.** Agents score each other's responses without access to ground truth. The paper assumes these scores track actual response quality, but in error-correlated failure modes (all agents generate the same wrong answer), high-relevance nodes will reinforce the error. No analysis of failure cases or error correlation is provided.
- **Statistical rigor is absent.** Tables 1 and 3 report single-point accuracy estimates with no confidence intervals or significance tests. Given the stratified sampling (50–150 examples per category), variance across category splits is uncharacterized. It is unclear whether the 1–2pp improvements over Self-MoA are reliable.
- **DyLAN comparison is incomplete.** On the gpt-4o scaling experiment (Table 3), DyLAN with 8 agents outperforms GoAMax with 3 agents on GPQA (58.89 vs. 55.05). The paper highlights GoAMax (6 agents) achieving 56.57, closer but still below DyLAN. This partial reversal is not discussed in the main text.

## Relations

- **competes-with 01_co_evolving_llm_decision_and_skill [low]:** COS-PLAY [1] addresses long-horizon task execution via two trained agents (decision + skill bank) with GRPO; GoA addresses test-time multi-domain QA via prompt-only graph collaboration. Both frame multi-agent collaboration as the path to improved outcomes, but operate in completely different regimes (trained vs. zero-shot, agentic vs. QA). The "workers team" framing in the thesis applies to both, but their architectural implications for decision agent design are largely orthogonal.
- **builds-on Graph-of-Thought reasoning [med]:** GoA explicitly cites Graph-of-Thought (Besta et al., 2024; Yao et al., 2024) as a precursor that applies graph traversal to single-model reasoning chains; GoA extends the graph paradigm from intra-model chain structuring to inter-agent communication with distinct specialized agents [2: §2].
- **competes-with MoA (Wang et al., 2024) [high]:** GoA directly targets MoA's three failure modes (agent selection, communication, integration) and proves MoA is a special case of GoA (Proposition 1). The benchmark comparison is explicit head-to-head and GoA consistently outperforms MoA [2: §1, §3.3, §4.2].
- **competes-with DyLAN (Liu et al., 2024) [med]:** Both use dynamic agent activation and bidirectional peer-rating (DyLAN's Agent Importance Score is structurally analogous to GoA's relevance scoring). GoA differs in that it operates purely via prompts without requiring a predefined topology or optimization; however, on the gpt-4o GPQA benchmark DyLAN (8 agents) outperforms GoAMax (3 agents), suggesting the predefined topology in DyLAN may encode useful domain priors [2: §2, Table 3].

### Relation to thesis

GoA is a test-time inference framework for multi-domain QA — not an agentic task-execution architecture. It addresses RQ1 (multi-agent workers team) only at the inference layer: selecting which agents handle a query and how they communicate, but not how agents execute multi-step decisions or acquire skills over time. The graph-based agent selection mechanism (node sampling via model cards) is directly applicable to the thesis's "decision agent as team coordinator" framing: a meta-LLM selecting relevant specialist sub-agents per query is a lightweight instantiation of the openclaw → decision agent dispatch chain described in the thesis. No contradiction to thesis detected; GoA provides evidence that selective, structured agent routing outperforms full-pool broadcasting (relevant to RQ2's openclaw-as-scheduler framing).
