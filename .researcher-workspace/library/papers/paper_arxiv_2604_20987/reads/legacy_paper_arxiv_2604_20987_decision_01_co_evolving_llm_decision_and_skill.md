---
title: "Co-Evolving LLM Decision and Skill Bank Agents for Long-Horizon Tasks"
paper_id: "paper_arxiv_2604_20987"
source_kind: "arxiv"
source_id: "arxiv:2604.20987"
source_url: "https://arxiv.org/abs/2604.20987"
kind: legacy-topic-read
topic: "decision"
source_note: "notes/active/01_co_evolving_llm_decision_and_skill.md"
---

# Legacy topic read

This Library read artifact was backfilled from an existing topic note. It preserves historical reading state; it was not produced by the current Library deep-read runner.

## Source topic note

---
zone: active
pin: false
score: 0
dwell: 0
paper: "Co-Evolving LLM Decision and Skill Bank Agents for Long-Horizon Tasks"
arxiv: "2604.20987"
authors: ["Xiyang Wu", "Zongxia Li", "Guangyao Shi", "Alexander Duffy", "Tyler Marques", "Matthew Lyle Olson", "Tianyi Zhou", "Dinesh Manocha"]
year: 2026
note_number: 1
---

## Claims

- Co-evolving a decision agent and a skill bank agent jointly outperforms training either component alone; ablations show that mismatched skill banks (policy and bank optimized on different state distributions) actively degrade performance [1: §5.2].
- COS-PLAY with an 8B base model (Qwen3-8B) achieves 25.1% average reward improvement over GPT-5.4 on four single-player game benchmarks (2048, Tetris, Candy Crush, Super Mario Bros.) [1: §5.1].
- Structured, reusable skill abstractions—not RL alone—are responsible for the gains; GRPO without a skill bank (GRPO W/O SKILL) scores lower on average than the full COS-PLAY pipeline [1: §5.2, Table 1].
- Skill reusability is demonstrable: across six games, each skill is instantiated on average 12–49 times, and contract refinement versions average 2–6 per skill, indicating ongoing adaptation rather than memorization [1: §5.3, Table 2].
- Co-evolution preserves general reasoning: COS-PLAY drops only 0.8% on MMLU-Pro and 1.8% on Math-500 relative to the Qwen3-8B base, showing that game-domain adaptation does not catastrophically degrade language reasoning [1: §5.4, Table 7].
- Splitting LoRA adapters by function (two for decision agent, three for skill bank agent) outperforms merging them into a single adapter per agent; merged-LoRA variant scores 502.5 ± 22.4 vs. 648.8 ± 38.8 on Candy Crush [1: §E, Table 6].
- On multi-player social reasoning games (Avalon, Diplomacy), COS-PLAY with an 8B model is competitive with Gemini-3.1-Pro and GPT-OSS-120B, and outperforms Gemini-3.1-Pro by 8.8% on Diplomacy mean supply centers [1: §5.1, Table 1].

## Assumptions

- Game environments with text-based observations are a sufficient proxy for studying long-horizon agentic decision-making; the framework is not validated in non-game, open-ended agentic environments [1: §3, §6 Limitation].
- Cold-start initialization using GPT-5.4-generated trajectories (60 per game) plus SFT is a practical and representative starting point; the quality of these seed trajectories directly shapes early skill bank content but is not ablated [1: §5 Cold Start Initialization].
- Compact text-based state summaries faithfully represent the environment's decision-relevant state; errors in summarization are assumed not to compound critically [1: §6 Limitation — the paper itself flags this as a potential issue].
- Opponents in multi-player games (GPT-5-mini for training, GPT-5.4 for evaluation) are stable and representative proxies for human play; exact numerical reproducibility is acknowledged to vary across API versions [1: §A].
- Skills are transferable within a game domain but not across games without re-running the co-evolution loop; cross-domain skill transfer is left to future work [1: §6].

## Method

**Framework overview.** COS-PLAY is a two-agent co-evolution loop (Figure 1):

1. **Decision Agent (A_D):** At each timestep, maintains an intention state z_t and an active skill s̃_t. Pipeline: (a) retrieve skill candidates from the bank via RAG, (b) select one conditioned on current observation and intention, (c) update intention, (d) execute a primitive action. Three functional LoRA adapters: skill retrieval (πθ_skill), intention update (πθ_int), and action taking (πθ_act). Skill switching is triggered when the intention shifts sharply.

2. **Skill Bank Agent (A_S):** Processes collected rollouts through a four-stage pipeline:
   - **Boundary Proposal:** Score each timestep for transition signals (predicate flips, intention-tag changes, reward spikes, surprisal peaks). High-score timesteps become candidate cut points; nearby candidates are merged.
   - **Infer Segmentation:** Select the subset of candidates that best partitions the trajectory into skill segments. Each segment is matched against existing bank skills by overlap of added/deleted predicates with learned effect contracts; unmatched segments are labeled "new skill."
   - **Contract Learning:** Aggregate added/deleted predicates across all instances of a skill; retain only consensus effects; optionally enrich via LLM summarization; verify pass rate threshold before committing.
   - **Skill Bank Maintenance:** Five operations — refine (update contracts with new evidence), materialize (promote buffered new-skill candidates), merge (collapse near-duplicate skills), split (flag over-broad skills), retire (remove disused skills).

**Training.** Both agents optimized with GRPO using separate LoRA adapters. Decision agent rewards: per-step environment reward + skill-following shaping (predicate satisfaction bonuses) + switching cost penalty; skill-retrieval adapter gets a delayed reward at skill-switch time (contract completion, efficiency, abort penalty). Skill bank agent rewards: segmentation adapter rewards fraction of episode reward covered by existing skills + decode consistency; contract adapter rewards F1 of predicted vs. observed predicates + holdout pass rate; curator adapter rewards quality-alignment + exploration + evidence-based reasoning.

**Cold start.** 60 GPT-5.4-generated trajectories per game → SFT on Qwen3-8B → shared initialization for both agents. Co-evolution then runs for 7–25 iterations depending on game.

## Eval

**Environments:** 2048, Candy Crush, Tetris, Super Mario Bros. (single-player); Avalon, Diplomacy (multi-player social reasoning).

**Baselines:** GPT-5.4, Gemini-3.1-Pro, Claude-4.6-Sonnet, GPT-OSS-120B (frontier LLMs); Qwen3-8B base; SFT W/O SKILL; SFT + 1ST SKILL (no co-evolution); SFT + FINAL SKILL (no co-evolution); GRPO W/O SKILL; GRPO + 1ST SKILL.

**Metrics:** Native game reward for single-player; team win rate for Avalon; mean supply centers for Diplomacy.

**Sample sizes:** 16 rollouts for single-player; 10 rollouts per player for multi-player; 95% confidence intervals reported.

**Key results (Table 1):**
- Single-player avg reward: COS-PLAY 924.4 vs. GPT-5.4 717.4 (+25.1%), vs. Qwen3-8B base 379.6.
- Avalon win rate: COS-PLAY 39% vs. Gemini-3.1-Pro 42% (−3pp), GPT-5.4 65%.
- Diplomacy mean SC: COS-PLAY 2.96 vs. Gemini-3.1-Pro 2.72 (+8.8%), GPT-5.4 4.70.
- Candy Crush LoRA ablation: COS-PLAY 648.8 vs. merged-LoRA 502.5.

**Skill reusability (Table 2):** Diplomacy bank: 64 skills, 5 categories, avg 6.52 clauses/skill, 814 total sub-episodes, avg 12.7 instances/skill, Gini 0.524, avg 2.4 refinement versions.

## Weaknesses

- **Evaluation horizon is very short.** Single-player episodes are capped at 50–200 steps; Diplomacy at 20 phases. True long-horizon agentic tasks (multi-day workflows, open-ended tool use) are qualitatively different and not tested. The "long-horizon" claim in the title is relative to prior game-agent work, not to real-world agentic deployments.
- **Opponent quality confounds multi-player results.** Avalon and Diplomacy use GPT-5-mini as training opponents and GPT-5.4 as evaluation opponents. COS-PLAY's skill bank may be tuned to exploit specific GPT-5.x failure modes rather than generalizing to arbitrary opponents. No ablation over opponent diversity is provided.
- **Cold-start quality is not ablated.** The 60 GPT-5.4 seed trajectories per game are the foundation for SFT and initial skill bank construction. The sensitivity of co-evolution outcomes to seed quality, quantity, or teacher model choice is never measured. This is a significant gap for practitioners who cannot access GPT-5.4.
- **Skill bank is text-predicate-based.** Boundary proposal and contract learning rely on predicate flips extracted from text state summaries. In environments without clean predicate structure (e.g., visual games, free-form tool-use agents), the pipeline's applicability is unclear and undemonstrated.
- **No cross-game transfer.** Each game runs a separate co-evolution loop from scratch. The paper claims reusable skills, but reuse is measured only within a single game domain. Whether skills transfer across games or to non-game agentic tasks is untested.
- **Statistical power is limited.** 16 rollouts for single-player and 10 per player for multi-player produce wide 95% confidence intervals (e.g., 2048: ±192 on a mean of 1589). Several ablation comparisons (e.g., SFT+FINAL SKILL vs. GRPO W/O SKILL) overlap in confidence interval, yet the paper draws conclusions from point estimates.

## Relations

This is the first note in the corpus; no prior notes exist to relate to. The following relations are flagged for update once subsequent papers are added:

- **orthogonal to** future notes on single-agent LLM planners [low]: COS-PLAY explicitly requires two coupled agents (decision + skill bank); single-agent approaches that internalize skill management would be architecturally incompatible with co-evolution and should be filed as orthogonal or competing.
- **builds-on** the Voyager paradigm (Wang et al., 2023) [med]: COS-PLAY cites Voyager as a predecessor for open-ended LLM agents with external skill libraries; COS-PLAY extends it by making the skill bank itself a trained agent rather than a prompted GPT pipeline.
- **competes-with** SkillRL (Xia et al., 2026) and SCALAR (Zabounidis et al., 2026) [med]: Both also combine skill learning with RL, but neither couples the skill bank into a separate trained agent with its own GRPO loop; co-evolution is the distinguishing structural claim.

### Relation to thesis

The thesis frames decision agent as a multi-agent workers team completing tasks end-to-end (RQ1). COS-PLAY instantiates exactly this: a two-agent system (decision agent + skill bank agent) that closes the loop between execution and skill learning. The thesis also asks whether decision agents can operate without long-term assistant context (RQ3); COS-PLAY's game episodes are stateless across episodes (no persistent memory beyond the skill bank), which is partial evidence that closed-loop skill learning can substitute for continuous context — but the game domain limits generalizability of this finding.

No contradiction to thesis detected. Potential extension: COS-PLAY's skill bank agent is itself a specialized worker; this supports the thesis's "multi-agent workers team" framing and could inform how openclaw might dispatch specialized sub-agents for skill curation.
