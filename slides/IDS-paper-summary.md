# Inductive Deductive Synthesis (IDS)
### Enabling AI to Generate Formally Verified Systems

**arXiv:** 2605.23109 · built on the NeurIPS 2026 style (preprint)
**Authors:** Shubham Agarwal\*, Alexander Krentsel\*, Shu Liu\*, Mert Cemri\*, Audrey Cheng, Rui Meng, Tomas Pfister, Chun-Liang Li, Sylvia Ratnasamy, Aditya Parameswaran, Matei Zaharia, Ion Stoica (UC Berkeley / Google), Mohsen Lesani (UC Santa Cruz). *(\*equal contribution)*
**Code:** https://github.com/skydiscover-ai/skydiscover

---

## TL;DR

AI coding agents are great at writing and testing code, but they cannot **guarantee** correctness over *every* possible behavior — which is exactly what distributed systems need (consistency must hold under every interleaving of messages, failures, and concurrent updates). Formal verification gives those guarantees but has historically cost **months to years** of expert effort per system.

**IDS** synthesizes the **implementation and its machine-checked proof jointly and incrementally**, using a proof assistant (Rocq/Coq) as a *perfect oracle that grades even partial work*. Realized as a multi-agent LLM system, IDS solves **7/7** distributed key-value-store consistency specs (SOTA agents solve only 2/7), in **~6.8 hours and ~$106 per spec**, roughly **200× faster** than expert effort, and produces implementations up to **3× faster** than published verified references.

---

## The Problem

- Agents treat verification as a **downstream check**: write all the code, then try to prove it all at once.
- This (a) requires writing the whole proof in one shot — exceptionally hard even for humans (Chapar's KV-store proof took 9–12 months), and (b) defers *all* correctness feedback to the very end, so the agent gets no guiding signal and stalls.
- **Evidence:** Given a proof assistant and the same prompt/budget, Codex (GPT-5.4) and Claude Code (Opus 4.6) each succeed on only **2 of 7** key-value-store consistency specifications.

---

## Key Idea

Synthesize **code + proof together, step by step**, learning from failures:

1. **Each joint step is simpler to prove** — decreasing proof-generation complexity.
2. **Partial proofs are checkable** — a dead-end implementation path is ruled out *before* further work is wasted; a failed step becomes a counterexample that improves the next design.

> Analogous to chain-of-thought reasoning — except the intermediate states are **formally encoded and verified**. Performance flows through the same loop: completed candidates are benchmarked, steering the search toward efficient implementations.

---

## Background: the Oracle

A verified system has three pieces — **specification** (precise correctness statement), **implementation** (the code), and **proof** (machine-checked argument that the impl matches the spec on every input).

- Rocq/Coq's type-checker **accepts iff correct** — **no false positives, no false negatives** (unlike tests, static analysis, or LLM-as-judge).
- The verdict extends to **partial** code: unproven goals are stated as **deferred holes** (`Admitted` lemmas / stub bodies) and the file *still type-checks*.
- This lets an agent tell "on track" from "dead end" without committing to a full proof.

The paper illustrates with a **counter** (spec → partial synthesis → complete synthesis), then scales the same loop to distributed stores: axioms become read-your-writes, one `nat` becomes a per-client vector clock, simple tactics become an inductive simulation argument with dozens of helper lemmas.

---

## The System (multi-agent architecture)

- **DSA — Deductive Synthesis Agent** ①: recursively decomposes a component and its spec into sub-components via placeholders, constructs proofs assuming sub-parts are correct; Coq's type-checker ② grades every complete and partial step. Actions: *define* partial impls, *close* proof branches, *decompose* lemmas; on error, *repair* or *revert* in the search tree.
- **ISA — Inductive Synthesis Agent** ⑤: learns from failed strategies. Two roles:
  - **Proposer** (A, *tactical*): suggests helper lemmas / decompositions when stuck on a specific proof.
  - **Reloader** (B, *strategic*): respawns the DSA with an entirely new high-level design on a dead-end.
- **Coordinator:** orchestrates a parallel pool of DSAs, detects stagnation, **eagerly benchmarks** candidates ③ (even before the proof closes) and feeds performance back to the ISA, and **audits** ④ closed proofs for non-vacuity (no remaining assumptions, correct interface, non-trivial).

---

## Results

### Q1 — Hard specifications (synthesis correctness)

IDS succeeds on **all 7** specs (≥ 2/3 runs); Codex and Claude Code each reach only **2/7** under the same prompt and budget.

| Specification | Codex | Claude Code | IDS | IDS time · cost |
|---|---|---|---|---|
| Chapar Causal Consistency | 0/3 | 0/3 | **3/3** | 10 h · $148 |
| Read-Your-Writes (RYW) | 3/3 | 3/3 | **3/3** | 2 h · $52 |
| Monotonic Reads (MR) | 0/3 | 0/3 | **3/3** | 7 h · $102 |
| Monotonic Writes (MW) | 2/3 | 3/3 | **3/3** | 3 h · $58 |
| RYW + MW | 0/3 | 1/3 | **3/3** | 5 h · $93 |
| Suite Causal Consistency | 0/3 | 0/3 | **2/3** | 11 h · $155 |
| Labeled Causal Consistency (LCC) | 0/3 | 0/3 | **3/3** | 10 h · $136 |
| **Total** | **2/7** | **2/7** | **7/7** | **~6.8 h · $106 avg** |

- Chapar's causal-consistency proof closed in **10 h / $148** ≈ **200× faster** than the cited 9–12 months of expert effort.
- The recurring winning move: **backtrack and change the data representation** (e.g., one entry per key, or per (key, client) pair) so the proof splits into small, manageable cases. Baselines keep the spec's default layout, never backtrack, and stall.
- **LLM-only baselines** (best-of-N=100, implementation only): from the formal spec, Codex passes on **1/4** properties; "vibe coding" from a natural-language description, **0/4**.

### Q2 — Runtime performance

Synthesized implementations match or beat **every** expert reference: **3×** throughput on Chapar's vector-clock reference, **1.4×** on suite CC and monotonic reads, comparable (within 20%) on RYW; on MW and RYW+MW the reference *times out* at every put rate.

> Mechanism: IDS's data-store representations are **bounded**, while references' grow with workload. These emerged from the joint code+proof loop, and benchmark feedback steers the agent to the fastest *verifying* candidate.

### Q3 — Ablations & generalization

- **Joint discovery (−J):** removing it drops the hard specs to 0/3 and VERINA by −8%.
- **Coq feedback (−VF):** replacing the structured diagnostic (goal, hypotheses, tactic backtrace) with mere accept/reject is the single most consequential loss — VERINA −35%; every spec ≤ 1/3.
- **Proposer / Reloader (−P / −R):** each drops pass rate on the four hardest specs (VERINA −14% / −5%).
- **Audit (−A):** without it, agents ship vacuous proofs (e.g., a `put-guard` returning `false` unconditionally).
- **Performance feedback (−PF):** full IDS is on average **1.42× faster**.
- **DSA as a general prover** (sets new SOTA across 4 languages): saturates miniCodeProps (100/100) and Verus-Bench (149/150), +36–69% over prior SOTA on DafnyBench and CoqStoq. Code-and-proof benchmarks: **176/189** VERINA (prior 38), **65/77** AlgoVeri (prior 31), **62/62** CloverBench. The IDS architecture adds +10% to +51% over the same model without it.

---

## Contributions

1. **IDS** — the first general technique for jointly synthesizing code and machine-checked proof under a *partial-proof oracle*, with failure and performance feedback in the same loop.
2. **IDS multi-agent system** — first agentic system to autonomously generate *verified* distributed systems (7/7 vs SOTA 2/7, up to 3× throughput).
3. **IDS suite** — six new Coq specifications for KV-store consistency models (RYW, MW, MR, RYW+MW, suite CC, LCC), released as an open benchmark.

---

## Limitations & Outlook

- IDS still **requires a formal Coq specification as input** — and **writing the specification is now the bottleneck** (a guarantee is only as good as its spec).
- The 7 specs cover correctness-critical behavior, not the full operational envelope (reconfiguration, fault recovery, observability), and evaluation is limited to distributed KV stores.
- IDS is **language- and domain-agnostic** in principle — any backend with (i) partial-proof checking, (ii) a programmatic interface, and (iii) executable extraction (Lean, Verus qualify). Future directions: **adversarial spec synthesis** and **spec extraction from existing systems**.

> **Bottom line:** verified software generation is shifting from a *person-year* bottleneck to a *compute-driven* problem — generalizable to any domain with a machine-checkable correctness oracle (OS kernels, compilers, crypto protocols, hardware).
