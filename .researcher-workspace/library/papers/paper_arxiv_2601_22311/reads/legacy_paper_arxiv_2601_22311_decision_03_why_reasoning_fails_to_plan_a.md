---
title: "Why Reasoning Fails to Plan: A Planning-Centric Analysis of Long-Horizon Decision Making in LLM Agents"
paper_id: "paper_arxiv_2601_22311"
source_kind: "arxiv"
source_id: "arxiv:2601.22311"
source_url: "https://arxiv.org/abs/2601.22311"
kind: legacy-topic-read
topic: "decision"
source_note: "notes/active/03_why_reasoning_fails_to_plan_a.md"
---

# Legacy topic read

This Library read artifact was backfilled from an existing topic note. It preserves historical reading state; it was not produced by the current Library deep-read runner.

## Source topic note

---
zone: active
pin: false
score: 0
dwell: 0
paper: "Why Reasoning Fails to Plan: A Planning-Centric Analysis of Long-Horizon Decision Making in LLM Agents"
arxiv: "2601.22311"
authors: ["Zehong Wang", "Fang Wu", "Hongru Wang", "Xiangru Tang", "Bolian Li", "Zhenfei Yin", "Yijun Ma", "Yiyang Li", "Weixiang Sun", "Xiusi Chen", "Yanfang Ye"]
year: 2026
note_number: 3
---

## Claims

- LLM step-wise reasoning (CoT, ReAct, Reflexion) is formally equivalent to a greedy policy that selects actions by maximizing a local surrogate score; it is structurally insufficient for long-horizon planning [3: §2, §3.2].
- Reasoning-based policies systematically induce early myopic commitments: at the first decision step, greedy and beam strategies select locally optimal but globally suboptimal "trap" actions at rates of 55.6% and 71.9% respectively, vs. 23.6% for lookahead [3: §3.1, Table 3].
- Early deviations under step-wise policies are effectively irreversible: single-step recovery probability after a first error is 5.4% vs. 22.4% for lookahead and 29.7% for FLARE [3: §3.1, Table 3].
- Beam search postpones but cannot prevent long-horizon failure: Proposition 3.2 proves that for any fixed beam width B, there exist environments where the optimal trajectory is irrevocably pruned at depth 1 regardless of B [3: §3.2].
- Even one-step lookahead strictly dominates step-wise decision making in worst-case performance: Proposition 3.3 proves there exist environments where all step-wise policies attain zero return while k=1 lookahead achieves the optimum [3: §3.2].
- FLARE (Future-aware LookAhead with Reward Estimation) consistently outperforms single-step, beam search, and shallow lookahead across all benchmarks, frameworks, and LLM backbones; average Hits@1 on CWQ: 71.8 (FLARE) vs. 66.5 (lookahead) vs. 63.3 (beam) vs. 59.8 (single) [3: §5.2, Table 1].
- LLaMA-8B with FLARE frequently matches or exceeds GPT-4o with standard step-wise reasoning, demonstrating that planning capability cannot be recovered by scaling model parameters [3: §5.2].
- FLARE's improvement changes the dominant failure mode from myopic early commitment to exploration and termination limits — pointing to a qualitatively different class of residual failures that reasoning improvements alone cannot address [3: §5.3].

## Assumptions

- Deterministic environments with explicit state representations and known transition dynamics are a sufficient diagnostic sandbox to isolate agent-side decision failures from environment-side uncertainty [3: §2, §3]; this controlled setting is not representative of real-world partially observable or stochastic environments [3: Limitations].
- A trajectory-level evaluative signal r̂ is available at planning time for both lookahead simulation and backward value propagation; this assumption is frequently violated in open-ended agentic tasks where reward is sparse, delayed, or entirely absent during execution [3: §4.3].
- KGQA (knowledge graph traversal) is a structurally representative proxy for long-horizon decision making; the oracle-structure setting guarantees a solution path exists in every subgraph, which removes the exploration problem that plagues real deployment [3: §3.1, §5.1].
- LLM-as-evaluator trajectory scoring (relative preference ranking over candidates) produces reliable enough estimates to guide value propagation; the paper does not measure evaluator calibration or characterize reward-hacking failure modes [3: §4.2].

## Method

**FLARE** instantiates three minimal planning requirements—explicit lookahead, backward value propagation, and limited commitment—via MCTS under a receding-horizon scheme:

**1. Explicit lookahead (tree expansion + action pruning).**
At each state s_t, FLARE builds a search tree using UCB-style selection:
`a_t = argmax_a [ Q(s_t, a) + c * sqrt(log N(s_t) / (N(s_t,a)+1)) ]`
where Q aggregates returns from simulated trajectories through (s_t, a). Tree expansion is restricted to a bounded candidate set `A_k(s)` (|A_k(s)| ≤ k) proposed by an LLM, limiting branching cost without biasing action values.

**2. Trajectory-level evaluation with trajectory memory.**
Each simulated trajectory is scored using an LLM that compares multiple candidates and assigns relative preference scores (not step-wise rewards). To amortize cost, FLARE maintains a bounded trajectory memory M; for a new trajectory τ, the most similar cached trajectory τ̃* is reused when sim(τ, τ̃*) ≥ δ, otherwise R(τ) is computed fresh.

**3. Backward value propagation.**
For each simulated trajectory of fixed depth H=3, cumulative return R(τ) is propagated backward to update Q(s_t, a_t) for all visited state-action pairs. Final action selection uses: `a* = argmax_a Q(s, a)`.

**4. Receding-horizon commitment.**
FLARE commits only to the next action and replans after each transition, preventing brittle long-term commitment under noisy evaluation signals. This is the key design choice that distinguishes FLARE from both CoT (no lookahead) and offline RL (planning baked into weights, no online revision).

## Eval

**Environments:**
- Primary (diagnostic): CWQ, WebQSP, GrailQA — KGQA benchmarks with oracle-structured subgraphs; deterministic state transitions; explicit evaluation signals. Satisfies all controlled-setting assumptions.
- Secondary (robustness check): ALFWorld (via AgentGym) — tool-use environment with discrete text actions; does not satisfy controlled-state assumptions; used only to test cross-domain generalization.

**Agent frameworks:** Think-on-Graph (ToG), Plan-on-Graph (PoG) for KGQA; ReAct, Reflexion for ALFWorld.

**LLM backbones:** LLaMA-8B, LLaMA-70B, GPT-4o mini, GPT-4o.

**Planning strategy baselines:** Single step (greedy CoT), Beam Search, Lookahead (shallow rollout without backward propagation), FLARE.

**Prior method baselines (Table 2):** IO Prompt, CoT, SC, ToT, RAP, LATS, KB-BINDER, ToG 2.0, RoG, ToG, PoG, GCR, DP, RwT, ProgRAG, DoG, PathMind.

**Metrics:** Hits@1 for KGQA; success rate + first-error step position for ALFWorld.

**Key results:**
- KGQA (Table 1): FLARE achieves best average Hits@1 across all three datasets, all four LLM backbones, and both frameworks. CWQ averages: FLARE 71.8, lookahead 66.5, beam 63.3, single 59.8.
- Prior MCTS/KGQA methods (Table 2): FLARE (PoG) achieves CWQ 78.8, WebQSP 93.9, GrailQA 92.0 — competitive with or surpassing specialized methods without task-specific training.
- Mechanism-level (Table 3): FLARE reduces trap@1 to 17.8% (vs. 55.6% single step), raises first-error step to 3.2 (vs. 1.6), and achieves 29.7% recovery rate (vs. 5.4%).
- ALFWorld: FLARE achieves 78% success (ReAct backbone, first-error step 4.6) vs. 61% for single step (2.4) and 72% for lookahead (3.9).
- Efficiency (Figure 3): FLARE shows sustained scaling gains with token budget; beam search and lookahead saturate well below FLARE accuracy.

## Weaknesses

- **The diagnostic setting is too controlled to generalize.** The entire empirical and theoretical analysis assumes deterministic transitions, explicit state representations, and evaluative signals available at planning time. The paper's own limitations section acknowledges this, but FLARE's design inherits these assumptions; whether backward value propagation remains useful when r̂ is noisy or delayed is undemonstrated.
- **KGQA is a narrow proxy for decision agent planning.** Graph traversal has a fixed branching structure and a definitive oracle solution path. Real decision agents face dynamic action spaces, partially observable state, and reward that appears only at task completion. The "long-horizon" claim applies to multi-hop reasoning depth (up to 8+ hops), not to temporal horizon in the sense of multi-day tool-use workflows.
- **ALFWorld is the only cross-domain eval, and it is a toy environment.** ALFWorld consists of household tasks with discrete text-based actions (pick/put/open/etc.) in a simulator. It does not test the open-ended tool use, API calls, or collaborative task decomposition that characterize production decision agents. The generalization claim is correspondingly weak.
- **No multi-agent setting evaluated.** FLARE is a single-agent planning mechanism. All experiments involve one agent planning against an environment. The thesis's core question (multi-agent workers team) is entirely orthogonal to FLARE's contribution; this paper provides no evidence about how planning mechanisms interact when multiple agents coordinate.
- **LLM-as-evaluator calibration is not measured.** Trajectory scoring via LLM preference comparisons is the core reward signal for FLARE's value propagation. The paper does not report evaluator accuracy, inter-annotator agreement, or sensitivity to evaluator choice. If the LLM evaluator systematically misranks trajectories, backward propagation could steer the tree toward suboptimal paths — a failure mode not characterized.
- **Absolute computational cost is not reported.** Figure 3 shows performance vs. token budget as a relative comparison; no wall-clock time or absolute token count at comparable accuracy targets is provided for FLARE vs. single-step. For practitioners, knowing whether FLARE's gains come at 2× or 20× inference cost is decision-critical.

## Relations

- **orthogonal to 01_co_evolving_llm_decision_and_skill [med]:** COS-PLAY [1] addresses multi-agent co-evolution and skill acquisition for game-domain long-horizon tasks; FLARE addresses single-agent planning via lookahead and value propagation. Both target "long-horizon decision making," but at different levels — COS-PLAY improves *what* actions the agent knows to take (skill bank), FLARE improves *how* the agent selects among known actions (planning mechanism). A combined system would need both. The two papers address orthogonal gaps, so no contradiction; but together they suggest that decision agents require both planning mechanisms (FLARE's contribution) and reusable skill abstractions (COS-PLAY's contribution) [1: §6, 3: §7].
- **orthogonal to 02_graph_of_agents_a_graph_based [low]:** GoA [2] addresses test-time multi-agent routing for single-turn QA; FLARE addresses single-agent multi-step planning. Neither paper acknowledges or targets the other's problem. Both are relevant to the decision agent thesis but operate at completely different system layers (inter-agent communication vs. intra-agent action selection). No competitive relationship.
- **builds-on MCTS tradition [high]:** FLARE explicitly builds on MCTS (Silver et al., 2018) and cites RAP (Hao et al., 2023), LATS (Zhou et al., 2024), and ReAct+MCTS as prior integrations of tree search with LLMs; FLARE's conceptual contribution is the theoretical diagnosis (Propositions 3.1–3.3) distinguishing reasoning from planning, not the MCTS algorithm itself [3: §4, §6].

### Relation to thesis

The thesis argues decision agent is an independently deployable multi-agent workers team. FLARE directly challenges the adequacy of the "step-wise reasoning" paradigm underlying most current decision agents, providing a theoretical and empirical proof that reasoning alone cannot substitute for planning in long-horizon tasks. This supports a thesis extension: the internal planning mechanism of individual decision agent workers must be future-aware, not greedy-local — otherwise the workers team will systematically fail on tasks requiring more than a few sequential decisions. This strengthens RQ1 (how decision agents complete end-to-end tasks) by specifying that action selection within each agent must incorporate lookahead. No contradiction to the multi-agent framing, but FLARE does not directly address RQ2 (openclaw × decision agent) or RQ3 (deployment independence). FLARE's evaluation is single-agent; the multi-agent coordination layer remains an open question even after this paper.
