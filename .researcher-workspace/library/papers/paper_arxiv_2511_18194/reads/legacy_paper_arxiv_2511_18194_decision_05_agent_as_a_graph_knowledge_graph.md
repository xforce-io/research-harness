---
title: "Agent-as-a-Graph: Knowledge Graph-Based Tool and Agent Retrieval for LLM Multi-Agent Systems"
paper_id: "paper_arxiv_2511_18194"
source_kind: "arxiv"
source_id: "arxiv:2511.18194"
source_url: "https://arxiv.org/abs/2511.18194"
kind: legacy-topic-read
topic: "decision"
source_note: "notes/active/05_agent_as_a_graph_knowledge_graph.md"
---

# Legacy topic read

This Library read artifact was backfilled from an existing topic note. It preserves historical reading state; it was not produced by the current Library deep-read runner.

## Source topic note

---
zone: active
pin: false
score: 0
dwell: 0
paper: "Agent-as-a-Graph: Knowledge Graph-Based Tool and Agent Retrieval for LLM Multi-Agent Systems"
arxiv: "2511.18194"
authors: ["Faheem Nizar", "Elias Lumer", "Anmol Gulati", "Pradeep Honaganahalli Basavaraju", "Vamse Kumar Subbiah"]
year: 2025
venue: "arXiv preprint (PwC Commercial Technology and Innovation Office)"
note_number: 5
---

## Claims

- Representing tools and their parent agents as co-equal nodes in a knowledge graph and traversing tool→agent edges at retrieval time outperforms agent-only retrieval (single-description matching) and tool-only retrieval (no bundle context) for LLM multi-agent routing [5: §1, §3].
- On LiveMCPBench (70 MCP servers, 527 tools, 95 questions), Agent-as-a-Graph achieves Recall@5 = 0.85 and nDCG@5 = 0.47 with optimal type-specific weighting, vs. prior SOTA ScaleMCP at 0.74/0.40 — 14.9% / 14.6% relative improvement on Recall@5 / nDCG@5 [5: §1, §4.2, Table 1].
- Type-specific weighted RRF (separate weights αA for agent nodes and αT for tool nodes within a unified candidate list) outperforms standard weighted RRF by 2.41% on the same benchmark; optimal ratio is modest agent emphasis at αA:αT = 1.5:1 [5: §1, §4.4, Table 3].
- Performance gains are architecture-agnostic: across 8 embedding models (Vertex AI, Gemini, Amazon Titan v1/v2, OpenAI ada-002 / 3-small / 3-large, All-MiniLM-L6-v2), Agent-as-a-Graph improves Recall@5 by an average of 19.4% over MCPZero with std ≈ 0.02 [5: §4.3, Table 2].
- Both node types contribute meaningfully to retrieval: 39.13% of retrieved top-K items originate directly from agent nodes CA, and 34.44% of matched tool nodes resolve to agents through ownership-edge traversal — i.e., agent-side evidence remains essential even when tool-level retrieval is added [5: §4.2.1].
- Concatenating agent and tool descriptions in the same vector space (the prior-SOTA approach in ScaleMCP) suffers from multi-hop query limitations characteristic of RAG; separating them as distinct typed nodes within a joint KG resolves this [5: §1].
- Extreme weight ratios degrade precision: αA:αT = 3:1 drops Recall@5 to 0.76, αA:αT = 1:3 drops to 0.80 — confirming that aggressive bias toward either node type harms routing [5: §4.4.1, Figure 2].

## Assumptions

- MCP manifests and schemas provide enough structured metadata to automatically induce the agent-tool bipartite graph without manual curation; the paper reports automated induction from existing MCP catalogs but does not measure how malformed or sparse manifests degrade graph quality [5: §2.2, §3.1].
- LiveMCPBench's 95 questions, averaging 2.68 steps with 2.82 tools / 1.40 MCP agents per query, are representative of production multi-agent routing workloads — the paper draws all conclusions from this single benchmark [5: §4.1].
- Step-wise query decomposition (the primary evaluation setting) faithfully reproduces upstream agent planning behavior; the paper assumes the decomposition is given (per benchmark annotations) and does not evaluate end-to-end retrieval quality when decomposition itself is noisy [5: §3.2.1, §4.1].
- Cosine similarity over text embeddings of (name, description) is sufficient to characterize tool / agent capabilities; the paper does not consider parameter schemas, type signatures, or tool I/O contracts as retrieval signal [5: §3.1.2].
- The damping constant k = 60 in RRF transfers from document retrieval to agent-tool retrieval without re-tuning; the paper holds this fixed across all experiments [5: §3.3.2].
- Type weights (αA, αT) generalize across query distributions; the paper sweeps the ratio on the test set itself, which conflates hyperparameter selection with evaluation [5: §4.4].

## Method

Agent-as-a-Graph retrieval has three components: knowledge graph construction, retrieval over the graph, and type-specific weighted reciprocal rank fusion.

**❶ Bipartite KG Construction (§3.1).** A catalog of MCP servers (agents A) and their tools T is modeled as a bipartite graph G = (A, T, E) where edges encode ownership owner(t) = a. Each tool node carries (name, description) text fields and an explicit edge to its parent agent node; each agent node carries (name, description) summarizing aggregate capabilities. The entire catalog C = CT ∪ CA is indexed in a shared vector space; ownership edges are stored in a graph backend (Neo4j-compatible, queryable via Cypher).

**❷ Retrieval (§3.2, Algorithm 1).** Given query q (either the raw user query or a decomposed sub-step):
1. Top-N retrieval from CT and CA independently via vector similarity, producing LT and LA. The paper sets N ≫ K.
2. Merge into single candidate list L = LT ∪ LA with a global rank r(e).
3. Apply type-specific weighted RRF (see ❸) to rerank.
4. Graph traversal: walk each ranked entity to its parent agent via owner(·); collect unique agents in order until K agents are returned.

**❸ Type-Specific Weighted RRF (§3.3).** Standard RRF computes s(e) = Σ_m 1/(k + r_m(e)). The paper modifies this to a piecewise scoring function:

```
s_new(e) = αT / (k + r(e))   if e ∈ CT
         = αA / (k + r(e))   if e ∈ CA
```

with k = 60 fixed. The two αs are explicit knobs that bias ranking toward tool specificity (αT) or agent coverage (αA) without learned re-ranking. The unified base rank r(e) is computed once over the merged list, decoupling tool/agent calibration while preserving the calibration-insensitive properties of RRF.

**Query paradigms.** Direct querying (raw user query → top-K agents) and step-wise querying (decomposed sub-tasks, each retrieved independently). Step-wise is the primary setting in the evaluation.

## Eval

**Benchmark:** LiveMCPBench (Mo et al., 2025) — 70 MCP servers, 527 tools, 95 real-world questions with step-by-step breakdowns and ground-truth agent-tool mappings. Average 2.68 steps, 2.82 tools, 1.40 agents per question.

**Baselines:** BM25 (lexical), Q.Retrieval (single-vector concatenated description), MCPZero (active tool discovery, Fei et al. 2025), ScaleMCP (prior SOTA dynamic auto-sync MCP retrieval, Lumer et al. 2025c), and standard RRF.

**Embedding models (cross-architecture experiment):** Vertex AI text-embedding-005, Gemini-embedding-001, Amazon Titan v1/v2, OpenAI ada-002 / 3-small / 3-large, All-MiniLM-L6-v2.

**Metrics:** Recall@K, mAP@K, nDCG@K for K ∈ {1, 3, 5}.

**Protocol:** Step-wise querying. Top-N initial retrieval, type-weighted reranking, graph traversal to top-K unique agents.

**Key results:**
- Baseline comparison (Table 1, OpenAI ada-002): Agent-as-a-Graph Recall@5 = 0.85, nDCG@5 = 0.47, mAP@5 = 0.35 vs. ScaleMCP 0.74 / 0.40 / 0.29, MCPZero 0.70 / 0.41 / 0.31, BM25 0.20 / 0.14 / 0.12.
- Cross-architecture (Table 2): Average Recall@5 0.85 (ours) vs. 0.70 (MCPZero), std ≈ 0.02. Largest gain on Amazon Titan v2 (+28.8% relative). All-MiniLM-L6-v2 still gains +19.4%.
- Type-weighted ranking (Table 3, Figure 2): Optimal αA:αT = 1.5:1 at Recall@5 = 0.85 / nDCG@5 = 0.47. Unweighted (1:1) = 0.83 / 0.46. Standard wRRF at 1:1 = 0.79 / 0.44. Extreme bias 3:1 drops to 0.76; 1:3 drops to 0.80.
- Provenance breakdown: 39.13% of retrieved top-K items come directly from agent nodes; 34.44% of tool nodes resolve to agents through graph traversal.

## Weaknesses

- **Hyperparameter selection conflated with test evaluation.** The optimal αA:αT = 1.5:1 weighting is selected by sweeping on LiveMCPBench and then reporting the same benchmark's score. There is no held-out validation split or cross-domain test. The 2.41% wRRF improvement claim is therefore an upper bound; out-of-distribution generalization of the type weights is undemonstrated [5: §4.4].
- **Single-benchmark evidence base.** All reported numbers come from LiveMCPBench. The paper claims general applicability to multi-agent systems with "thousands of tools," but LiveMCPBench has 527 tools across 70 servers — modest by enterprise scale (e.g., Decision Agent's target of ~1k–10k tools). Scaling behavior beyond ~500 tools is not measured.
- **Graph traversal cost is not analyzed.** Algorithm 1's traversal step iterates the merged candidate list and resolves each tool to its parent. For large N (top-N retrieval cutoff) and dense agent-tool edges, this could dominate latency. The paper provides no wall-clock latency comparison vs. baselines, despite claiming "transparent, non-learned scoring" as a deployment advantage.
- **Tool description quality is uncontrolled.** Both retrieval signal (tool/agent text) and the ablation use only natural-language (name, description) fields. Tool descriptions in production MCP catalogs vary widely in quality and consistency; the paper does not study how description quality affects retrieval, nor does it use structural signal (parameter schemas, type information) that could disambiguate similar tools.
- **No comparison against learned re-rankers.** The paper positions itself against expensive LLM re-ranking but does not include any cross-encoder or LLM-as-judge re-ranker as a strong baseline. The implicit claim that wRRF is "good enough" relative to learned re-ranking is asserted but not measured.
- **Step-wise decomposition is provided by the benchmark.** All step-wise evaluation uses LiveMCPBench's annotated decomposition. In production, an upstream agent must generate the decomposition itself, which introduces error. The retrieval-decomposition coupling is not studied.
- **The 14.9% headline improvement is over ScaleMCP at K=5; at K=1 the gap is smaller (Recall@1: 0.53 vs. 0.49 ≈ 8%).** The paper highlights the K=5 result throughout but agent routing in production typically commits to K=1 or K=2 (not all K=5 candidates can be invoked simultaneously). The marketing emphasis on K=5 may overstate impact for top-1 routing.
- **Reproducibility gap on graph backend.** The paper mentions Neo4j and Cypher compatibility but provides no graph schema, ingestion pipeline, or open-sourced implementation. The claim of "automatic induction from MCP manifests" is not accompanied by a released artifact.

## Relations

- **builds-on prior tool/agent retrievers (ScaleMCP, MCPZero, Toolshed) [high]:** All from the same research group (Lumer et al. team, PwC). Agent-as-a-Graph extends ScaleMCP's concatenated-description retrieval with explicit type separation and graph traversal, citing the multi-hop query limitation as motivation [5: §1, §2.1]. The same authors' Tool-to-Agent Retrieval (Lumer et al. 2025d) is the most direct precursor — Agent-as-a-Graph adds the typed wRRF layer on top.
- **builds-on Reciprocal Rank Fusion (Cormack et al. 2009) [high]:** Standard wRRF is the explicit point of departure; the paper's contribution is the piecewise type-conditioned form (Equation 3). The shared denominator k + r(e) is preserved, ensuring within-type ordering matches RRF [5: §3.3].
- **competes-with 02_graph_of_agents [med]:** Both apply graph structure to multi-agent LLM systems but at different layers. Graph-of-Agents (GoA) builds a graph at *inference time* over selected reasoning agents to coordinate output via message passing; Agent-as-a-Graph builds a graph at *retrieval time* over the agent-tool catalog to select which agents to invoke. They could compose: Agent-as-a-Graph selects K agents, GoA-style message passing then coordinates them. Neither paper cites the other (concurrent work).
- **competes-with 01_co_evolving_llm_decision_and_skill [low]:** COS-PLAY frames skill management as a separate trained agent that learns reusable skill abstractions over time; Agent-as-a-Graph treats tool/agent catalog as static and focuses on retrieval over it. The two approaches address different sub-problems: skill *acquisition* vs. tool *selection*. They are largely orthogonal but compete for the same architectural role of "how does the system know which capabilities to invoke" — COS-PLAY learns the answer, Agent-as-a-Graph retrieves it.
- **orthogonal to 03_why_reasoning_fails_to_plan_a [high]:** Planning vs. retrieval are distinct problems. Agent-as-a-Graph assumes a query (or decomposed sub-step) is already given and finds agents to handle it; the planning critique in 03 concerns whether step-wise reasoning generates correct decomposition in the first place. Both could co-exist in the same architecture (planner produces sub-steps; Agent-as-a-Graph retrieves agents per sub-step).
- **competes-with 04_rethinking_the_value_of_multi_agent [med]:** Paper 04 questions whether multi-agent decomposition adds value over a single strong LLM; Agent-as-a-Graph operates within the multi-agent paradigm and assumes its value, focusing on improving the routing layer. The two papers represent opposing stances on multi-agent architectures: 04 is skeptical, Agent-as-a-Graph is constructive.

### Relation to thesis

Direct hit on the ContextLoader / tool-management research gap. The thesis explicitly flags "工具检索精确召回 (向量 vs 语义结构化对比)" as a high-priority topic, and Agent-as-a-Graph is exactly such a comparison: typed graph + wRRF vs. concatenated single-vector retrieval (ScaleMCP) vs. tool-only retrieval (MCPZero). Three takeaways for the Decision Agent stack:

1. **BKN's "logic/action separation" framing has empirical support.** Agent-as-a-Graph's core finding — that separating tool-type and agent-type as distinct typed nodes in a unified retrieval space outperforms concatenation — is structurally analogous to BKN's separation of logic from action. The 14.9% Recall@5 lift is direct evidence for the value of typed semantic decoupling, beyond just engineering convenience.

2. **Modest agent emphasis (αA:αT = 1.5:1) is a tunable knob worth replicating.** ContextLoader's tool selection mechanism could expose similar type-specific weights to bias routing toward parent-skill / parent-domain coverage when query intent is ambiguous, while preserving fine-grained tool retrieval when intent is specific.

3. **Scale gap remains open.** LiveMCPBench at 527 tools is well below the thesis's enterprise-scale target (thousands of tools). The architecture-agnostic robustness across 8 embedding models is encouraging for portability, but the paper offers no evidence at the scale where ContextLoader's claimed 90%+ context savings actually matter. This is a concrete benchmark gap the Decision Agent team could fill.

No contradiction to thesis. Strong supporting evidence for the typed-decoupling architectural choice; flags the missing piece (scale validation beyond ~500 tools).
