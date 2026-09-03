---
title: "Rethinking the Value of Multi-Agent Workflow: A Strong Single Agent Baseline"
paper_id: "paper_arxiv_2601_12307"
source_kind: "arxiv"
source_id: "arxiv:2601.12307"
source_url: "https://arxiv.org/abs/2601.12307"
kind: legacy-topic-read
topic: "decision"
source_note: "notes/active/04_rethinking_the_value_of_multi_agent.md"
---

# Legacy topic read

This Library read artifact was backfilled from an existing topic note. It preserves historical reading state; it was not produced by the current Library deep-read runner.

## Source topic note

---
zone: active
pin: false
score: 0
dwell: 0
paper: "Rethinking the Value of Multi-Agent Workflow: A Strong Single Agent Baseline"
arxiv: "2601.12307"
authors: ["Jiawei Xu", "Arief Koesdwiady", "Sisong Bei", "Yan Han", "Baixiang Huang", "Dakuo Wang", "Yutong Chen", "Zheshen Wang", "Peihao Wang", "Pan Li", "Ying Ding"]
year: 2026
note_number: 4
---

## Claims

- In homogeneous multi-agent workflows (all agents share the same base LLM, differing only in prompts and tools), a single LLM can simulate the full workflow via multi-turn conversations and reach identical or slightly higher performance than the multi-agent setup [4: §3.1, §4.2.1].
- Proposition 1 formally proves that under deterministic tool side-effects, routing conditioned only on visible history, and shared randomness, the transcript distribution of a single-LLM simulator equals that of the original multi-agent system [4: §3.1].
- Single-agent KV cache reuse reduces prefill cost from Σ L_t (full prefix per step, multi-agent) to Σ ΔL_t (incremental growth only), making single-agent execution asymptotically cheaper whenever agent contexts overlap [4: §3.1].
- OneFlow (single-agent execution) achieves matching or superior performance to AFlow across 6 public benchmarks at dramatically lower cost: e.g., HotpotQA F1 73.5 vs. 72.1 for AFlow at $0.278 vs. $1.438 per dataset [4: §4.2.1, Table 1–2].
- Heterogeneous workflow performance (mixing GPT-4o-mini + Claude 3.5 Haiku) is bounded by the best homogeneous workflow using the stronger base model: DROP F1 85.5 (heterogeneous) vs. 87.5 (OneFlow with Claude 3.5 Haiku alone) [4: §4.2.3, Table 3].
- Single-agent KV cache benefit is confirmed on open-weight Qwen-3 8B with vLLM: AFlow single-agent achieves 90.5% pass@1 on HumanEval vs. 86.8% for multi-agent stateless calls, with stable throughput despite increased input token counts [4: §4.2.4, Table 4].
- OneFlow's dual meta-LLM (Designer + Critic) search discovers more cost-efficient workflows than AFlow while maintaining comparable accuracy, reducing average inference cost by up to 10× on simple tasks [4: §4.2.2, Table 2].

## Assumptions

- The homogeneous assumption (|B(W)| = 1) is the critical precondition for all theoretical and empirical claims; experiments on heterogeneous settings are explicitly labeled as "pilot" and not fully optimized [4: §2, §4.2.3].
- Tool side-effects are deterministic given inputs — a condition violated in real-world agentic tasks where tools query live APIs, read mutable databases, or interact with external services [4: §3.1, Proposition 1 condition (i)].
- Routing policy depends only on visible history; no hidden agent state exists between turns. This excludes workflows where agents maintain private scratchpads or memory not visible in the shared conversation [4: §3.1, Proposition 1 condition (ii)].
- Temperature=0 (deterministic decoding) is used across all experiments; the formal equivalence result does not cover stochastic sampling, which is required for diverse generation in production agents [4: §4.1].
- KV cache sharing with closed-weight API models (GPT-4o-mini) is simulated via conversation state cost estimation, not directly measured; actual API providers may not expose or guarantee cache sharing across sequential calls [4: §4.1, KV Cache and Cost Estimation].
- Benchmarks represent a sufficient distribution of "complex tasks": the most open-ended is TravelPlanner (single-session planning), leaving multi-day, multi-session enterprise workflows untested [4: §4.1].

## Method

**Problem framing.** Given a task T, find workflow W* = argmax [α·P(W,T) − β·C(W,T)] over agent graphs G=(N,E). The key finding is that for homogeneous W (same base LLM b), single-agent simulation is behavior-preserving and KV-efficient.

**Single-agent simulator (§3.2).** At each step t: (1) set system message to current agent's prompt p_{i_t}, appended as a user message to the shared chat history h_{t-1}; (2) query the same base model b to obtain m_t; (3) execute tool u ∈ τ_{i_t} if needed, append result r_t; (4) update h_t and advance routing per edge conditions in E. The single conversation accumulates all agent turns; KV cache is reused across all steps.

**OneFlow search (§3.3).** MCTS-based workflow search with two stages:
1. *Search:* Initialize with IO workflow (single agent, direct I/O). Run 20 MCTS iterations: selection via mixed-probability over top-3+IO candidates; expansion via dual meta-LLMs (Creative Designer proposes performance-maximizing modifications, Critical Reviewer trims for cost-efficiency using top-candidate performance and cost data); evaluation on 20% validation split; backpropagation of results and failure cases to parent node.
2. *Execution:* Apply the §3.2 single-LLM simulator to the discovered workflow.

**Heterogeneous pilot (§4.2.3).** AFlow used to auto-design workflows with two executor LLMs (GPT-4o-mini + Claude 3.5 Haiku); Claude-4.0-Sonnet serves as designer. 20 optimization iterations. Used only for bounding analysis, not as a full comparative experiment.

## Eval

**Benchmarks:** HumanEval, MBPP (code, pass@1); GSM8K, MATH (math, solve rate %); HotpotQA, DROP (QA, F1); Shopping-MMLU (domain accuracy, 10 hardest tasks); TravelPlanner (planning, task success %).

**Baselines:** Manual (IO, CoT, CoT-SC×5, MultiPersona); auto-designed multi-agent (AFlow); proposed (OneFlow); single-agent execution of AFlow and OneFlow workflows; heterogeneous AFlow (GPT-4o-mini + Claude 3.5 Haiku). Additional executor: Claude 3.5 Haiku; open-weight: Qwen-3 8B via vLLM.

**Metrics:** Task accuracy/F1/pass@1 + inference cost (USD per dataset). Three independent trials, mean ± std. Qwen-3 8B: latency (s), throughput (samples/s), token counts.

**Key results:**
- OneFlow (single-agent) matches or beats AFlow on all 6 public benchmarks with GPT-4o-mini.
- OneFlow single-agent cost vs. AFlow: HumanEval $0.020 vs. $0.198 (~10×); HotpotQA $0.278 vs. $1.438 (~5×).
- TravelPlanner (Figure 3): OneFlow single-agent is Pareto-dominant (higher success, lower cost) over multi-agent AFlow.
- Heterogeneous (DROP F1 85.5) bounded by Claude 3.5 Haiku homogeneous (87.5, OneFlow); exceeds all GPT-4o-mini methods (best: 83.1 AFlow).

## Weaknesses

- **Theoretical guarantee requires temperature=0.** All experiments set temperature to 0 to satisfy Proposition 1's shared randomness condition. Production decision agents commonly use non-zero temperature for exploration or diverse generation; the formal equivalence breaks in that regime, and no empirical result covers it.
- **KV cache savings are simulated, not measured, for closed-weight APIs.** The efficiency claim for GPT-4o-mini is derived from theoretical token counting using the final conversation state, not from measured API latency or billed tokens. Closed-weight API providers (OpenAI, Anthropic) may or may not cache multi-turn context; the paper's efficiency numbers cannot be replicated by API users without confirmed cache behavior.
- **Heterogeneous pilot is deliberately under-optimized.** The paper uses automatically discovered heterogeneous workflows and explicitly notes they "are not perfectly optimized." The upper-bounded-by-homogeneous result does not rule out that a well-engineered heterogeneous workflow (e.g., routing simple steps to small models and complex steps to frontier models) would outperform. The bounding claim is valid only for AFlow-discovered heterogeneous configurations.
- **No evaluation of multi-session or stateful workflows.** All benchmarks complete within a single inference session (TravelPlanner being the most complex, with one planning call per query). Enterprise decision agent workflows involving persistent state across sessions, approval gates, or human-in-the-loop steps are absent. The "end-to-end enterprise task" regime from the thesis is entirely untested.
- **Shopping-MMLU benchmark is selection-biased.** The 10 tasks are selected by finding where Claude 3.5 Sonnet scores <80% — this intentionally biases toward tasks where a Sonnet-class model fails, which may not represent the distribution of real enterprise task difficulty, and introduces evaluation-set contamination risk for future Sonnet-family models.
- **Parallel agent execution is not addressed.** The simulator executes agents sequentially. Workflows where agents run in parallel (e.g., fan-out-fan-in patterns where multiple workers process data simultaneously) cannot be collapsed to single-agent sequential execution without a fundamental change to latency properties. This architectural variant is not discussed.

## Relations

- **contradicts thesis §1 (multi-agent coordination is necessary) [med]:** The thesis asserts the core challenge is "使多智能体 workers team 能够可靠完成端到端的企业业务任务," treating multi-agent structure as architecturally necessary. This paper shows that for homogeneous configurations — where all workers share the same base LLM — multi-agent structure provides no accuracy benefit and increases cost. This is a partial falsification of the multi-agent-first premise. The contradiction is partial because it holds only for homogeneous workflows; the thesis's Decision Agent uses diverse specialized workers (BKN + ContextLoader + Dolphin + ISF), which may constitute a heterogeneous regime beyond the paper's scope.
- **competes-with 02_graph_of_agents_a_graph_based [med]:** GoA [2] argues for graph-structured multi-agent routing as the performance path; OneFlow shows that for homogeneous settings, sophisticated inter-agent routing structure is unnecessary — a single agent with the right sequential workflow matches it. Both papers use similar benchmarks (MMLU-Pro, etc.) but reach opposite architectural conclusions. GoA uses heterogeneous 7–8B specialists (a heterogeneous regime); OneFlow's same-LLM finding does not refute GoA's specific setting, but frames it as an efficiency unjustifiable architecture choice when agents share the same base model [4: §3.1; 2: §3.3].
- **extends 03_why_reasoning_fails_to_plan [low]:** Note 3 [3] proves step-wise reasoning is structurally insufficient for long-horizon planning. OneFlow shows that even complex multi-agent task decomposition workflows can be collapsed to single-agent sequential reasoning without performance loss in the tested domains. Together they suggest that the performance ceiling is determined by the planning mechanism (FLARE's domain), not the multi-agent communication topology — but this is my synthesis; neither paper states it.
- **orthogonal to 01_co_evolving_llm_decision_and_skill [med]:** COS-PLAY [1] uses two genuinely different trained agents (decision agent + skill bank agent with separately trained LoRA adapters) — a heterogeneous multi-agent system outside the scope of OneFlow's homogeneous equivalence result. OneFlow explicitly states single-LLM simulation "cannot capture heterogeneous workflows due to lack of KV cache sharing across different LLMs" [4: §6], which is precisely the COS-PLAY regime. No contradiction; COS-PLAY operates where multi-agent structure remains necessary by OneFlow's own criterion.
