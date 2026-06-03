# Full Summary — *Inductive Deductive Synthesis: Enabling AI to Generate Formally Verified Systems*

> **Review-grade summary.** Section-by-section, with the key claims, mechanisms, numbers, and a critical-reading section at the end.

- **arXiv:** 2605.23109 · built on the NeurIPS 2026 style file (preprint; the source does not state acceptance)
- **System name in the paper:** **IDS** (written `\sys` in the LaTeX; the technique is *Inductive Deductive Synthesis*)
- **Authors:** Shubham Agarwal\*, Alexander Krentsel\*, Shu Liu\*, Mert Cemri\*, Audrey Cheng, Rui Meng, Tomas Pfister, Chun-Liang Li, Sylvia Ratnasamy, Aditya Parameswaran, Matei Zaharia, Ion Stoica (UC Berkeley / Google); Mohsen Lesani (UC Santa Cruz). \*equal contribution.
- **Code:** https://github.com/skydiscover-ai/skydiscover
- **Backend used:** Rocq/Coq. **Primary LLM backend:** Codex (GPT-5.4).

---

## 1. One-paragraph abstract (paraphrased)

AI agents are good at generating, testing, and refining code, but fall short when a task needs *formal guarantees of full coverage* that testing can't provide. Distributed systems are the canonical case: consistency between reads and writes must hold under every interleaving of events. Mechanized verification can guarantee this but historically takes months to years of expert effort. Even SOTA coding agents (Codex/GPT-5.4, Claude Code/Opus 4.6) given a proof assistant solve only **2/7** distributed key-value-store specs. The paper introduces **Inductive Deductive Synthesis (IDS)**, which *jointly and incrementally* synthesizes implementation and proof and *learns from failed attempts*. As an agentic system it achieves **7/7** in ~**6.8 h** and **$106/spec** on average — ~**200×** faster than expert effort and **17%** cheaper than SOTA agents — and with performance feedback in the same loop yields implementations up to **3×** faster than published verified systems.

---

## 2. Problem & motivation (§1)

**The capability gap.** Coding agents excel at writing code and self-refining via tests, but tests only cover the cases tried. They cannot certify correctness across *every* behavior. Distributed systems make this acute: properties (e.g. consistency) must hold over a combinatorially large space of message/failure/concurrency interleavings. Failures are severe — lost data, dropped writes, leaked keys.

**Why not just give agents a proof assistant?** The paper's experiments say this *doesn't work*. Codex and Claude Code fail on **5 of 7** consistency specs. The diagnosed failure mode is **structural**, not a model-capacity issue:
- Agents follow the human pattern — treat verification as a *downstream check* on already-written code.
- This (a) forces writing the whole proof at once (Chapar's KV-store proof took experts **9–12 months**), and (b) defers *all* correctness feedback to the end, removing guiding signal. The agent stalls.

**The proposed fix — IDS.** Synthesize implementation *and* proof **jointly and incrementally**; every implementation decision (add a data structure, change control flow) is paired with a proof update (add constraints, split lemmas). Two benefits:
1. **Each joint step is simpler to prove** → lower proof-generation complexity.
2. **Partial proofs are checkable** → rule out a dead-end *before* wasting work; a negative result is a counterexample improving the next design.

This is framed as *"chain-of-thought, but the intermediate states are formally encoded and verified."* Performance flows through the same loop: completed candidates are benchmarked, steering search toward efficient implementations.

**Stated contributions:**
1. **IDS** — first general technique for jointly synthesizing code + machine-checked proof under a *partial-proof oracle*, with failure and performance feedback in one loop.
2. **IDS multi-agent system** — first agentic system to autonomously generate *verified* distributed systems (7/7 vs SOTA 2/7; up to 3× throughput).
3. **IDS suite** — six new Coq specifications for KV-store consistency models, released as an open benchmark.

---

## 3. Related work (§ Related)

Verified code generation has three components — (1) spec, (2) implementation, (3) proof. The paper positions IDS against four threads:

| Thread | Representative work | How IDS differs |
|---|---|---|
| **Specification generation** | nl2postcond, AutoSpec, Clover | IDS *consumes* specs as input, doesn't produce them |
| **Program generation w/ evaluator** | SWE-agent, Reflexion, MetaGPT/ChatDev/AutoGen, FunSearch, AlphaEvolve, AlphaDev | Same evaluator-guided-search scheme, but the evaluator is a **proof assistant** — guarantees on *all* inputs vs sampled tests/scores |
| **Automated proof generation** | AutoVerus, Laurel (SMT); CoqGym, DeepSeek-Prover, Goedel, LeanDojo, COPRA, etc. (tactic) | Prior work fixes impl + spec, asks only for the proof; IDS jointly generates impl + proof and *also optimizes performance* |
| **Verified synthesis** | Manna–Waldinger, Synquid, Sketch/CEGIS; VERINA, AlphaVerus; hand-built IronFleet, Verdi, FSCQ, Anvil, Chapar, Iris/Grove/Perennial | Classical methods confined to small functions; hand-built systems cost months–years. IDS shows verified synthesis *at systems scale, in hours* |

---

## 4. Background & the oracle (§2)

**The three pieces of a verified system:** *specification* (precise correctness statement), *implementation* (running code), *proof* (machine-checkable argument they agree on every input). Historically humans wrote all three; IDS takes a human-written spec and auto-produces impl + proof.

**Worked toy example — a counter** (Fig. *counter-listing*): state machine with `init`, `inc`, `read`. Spec states two axioms: `read init = 0` and `read (inc s) = S (read s)`. Shown in three columns:
- **Spec** (a Coq `Module Type`).
- **Partial synthesis** — `inc`'s body and the `read_inc` theorem are `Admitted` (placeholders); Coq *still accepts the file* because the chosen representation (a list whose length is the count) is consistent.
- **Complete synthesis** — holes filled (`inc := tt :: s`, proof closed with `Qed`).

**The key property — type-checker as a perfect oracle:**
- Accepts a declaration/proof **iff** it satisfies its type/proposition; rejects with a precise diagnostic otherwise.
- **No false positives, no false negatives** — unlike tests (only inputs tried), static analysis (over-approximate), or LLM-as-judge (guesses).
- Crucially, the verdict extends to **partial** code: unproven obligations become *deferred holes* (`Admitted` lemmas / stub bodies) and the file still type-checks. (App. C walks an `all_less_than` example.)
- IDS uses this for **deductive synthesis**: at each step extend the partial impl+proof, ask Coq if it still type-checks; *yes* keeps the design, *no* rules it out. If progress stalls (measured by remaining empty partial proofs), IDS reverts to an earlier state.

**Scaling counter → distributed store:** axioms grow into conditions over executions with messages/clients/replicas/failures (e.g. `read_inc` becomes **read-your-writes**); a single `nat` becomes a per-client **vector** tracking increments; the proof grows into an **inductive simulation argument** linking implementation and spec states with dozens of helper lemmas.

---

## 5. The agentic architecture (§3, Fig. OVERVIEW_final)

IDS is realized as a multi-agent system. **Synergy:** *deductive synthesis builds code+proof under a given strategy; inductive synthesis discovers better strategies from failures.*

**① DSA — Deductive Synthesis Agent.** A generic LLM coding agent under strict constraints (well-typed before proving; spec immutable; no unproven assumptions in final output; bounded state). It recursively decomposes a component and its spec into sub-components via placeholders, builds proofs assuming sub-parts correct, down to trivial elements. At each search-tree node it can:
- *Define* a partial implementation,
- *Close* an open proof branch,
- *Decompose* a complex lemma into helper lemmas.
After each step → **② Coq type-checker** grades it. No errors → state saved (building a tree of well-typed partial impls/proofs). Errors → *repair* (try a different approach) or *revert* to an earlier node.

**⑤ ISA — Inductive Synthesis Agent.** Stateless LLM that reasons over a strategy summary (not the full proof state) and proposes new approaches across two horizons:
- **Ⓐ Proposer (tactical):** strategy is promising but the DSA is stuck on a specific proof → suggests helper lemmas / decompositions.
- **Ⓑ Reloader (strategic):** strategy is a dead-end or slow → respawns the DSA with an entirely new high-level design (e.g. vector clock instead of dependency list).

**Coordinator.** Launches/orchestrates a *parallel pool* of DSAs, maintains state. DSAs can report local errors but can't detect strategic dead-ends, so the coordinator monitors progress, records failed strategies, and invokes the ISA before respawning. It also:
- **③ Benchmarks each candidate eagerly** — extracts impl to executable code and runs the harness *even before the proof closes*; feeds measurements to the ISA so future strategies favor efficient implementations.
- **④ Audits** closed proofs (no remaining assumptions, expected interface implemented, non-trivial implementation, correct theorem stated).
- Returns the most efficient verified solution found within the time budget.

---

## 6. Evaluation (§5)

**Specs (7 total):** Chapar's published causal-consistency (CC) + six new **IDS suite** specs co-developed with an expert: read-your-writes (RYW), monotonic writes (MW), monotonic reads (MR), their composition (RYW+MW), suite CC, and **LCC** (labeled CC — clients subscribe to topic labels; causality enforced only on those). Three difficulty tiers: *simple* (RYW, MW) → *many proof cases* (MR, RYW+MW) → *explicit dependency sets* (CC, LCC).

**Harness:** verified Coq → OCaml via built-in extraction; run on a **5-VM Google Cloud cluster**; 4 parallel workers × 1,000 random ops; sweep put rates; scaling sweep at put=50% over N ∈ {1k, 2k, 5k, 20k}. Each cell = median of 3 runs; relative std ≤ 7%. Failed run = exceeds wall-clock or dollar budget.

**Baselines:** Codex (GPT-5.4) and Claude Code (Opus 4.6) under the *same prompt, model, verifier, budget* — isolating the architecture. Performance references: hand-written expert implementations (Chapar's vector-clock and list-based; per-spec expert references, except LCC which has none).

### Q1 — Synthesis correctness (§5.2)

IDS returns verified implementations on **all 7** specs (≥2/3 runs) — a **3.5×** pass rate over **2/7** each for the baselines. Even on the two specs both baselines pass (RYW, MW), IDS is **1.6× faster** and **17% cheaper**.

| Specification | Codex | Claude Code | IDS | IDS h · $ |
|---|---|---|---|---|
| Chapar CC | 0/3 | 0/3 | **3/3** | 10 · 148 |
| RYW | 3/3 | 3/3 | **3/3** | 2 · 52 |
| MR | 0/3 | 0/3 | **3/3** | 7 · 102 |
| MW | 2/3 | 3/3 | **3/3** | 3 · 58 |
| RYW+MW | 0/3 | 1/3 | **3/3** | 5 · 93 |
| suite CC | 0/3 | 0/3 | **2/3** | 11 · 155 |
| LCC | 0/3 | 0/3 | **3/3** | 10 · 136 |
| **Total** | **2/7** · 67h · $880 | **2/7** · 56h · $885 | **7/7** · **48h** · **$744** | |

- **Chapar CC** closed in 10 h / $148 ≈ **200× faster** than the cited 9–12 months. Baselines fail even within 1.2× IDS's wall-clock and cost.
- **The winning mechanism (recurs across specs):** IDS *backtracks and changes the data representation* so the proof splits into small cases — e.g. one entry per key (Chapar CC), one per (key, client) pair (MR), one case per message (CC variants). Baselines keep the spec's default layout, never backtrack, stall.
- **LLM-only baselines** (impl only, best-of-N=100): spec-given Codex passes both an adversarial trace check and a refinement proof on **1/4** properties; "vibe coding" (natural-language description only) on **0/4**. Even with the formal spec and 100 candidates, the LLM alone can't produce guaranteed-correct implementations.

### Q2 — Runtime performance (§5.3, Fig. eval_perf)

IDS matches or beats **every** reference:
- **3×** throughput on Chapar's vector-clock reference,
- **1.4×** on suite CC and monotonic reads,
- comparable (within 20%) on RYW,
- **152k / 139k ops/s** on MW / RYW+MW where the reference **times out at every put rate**.

**Mechanism:** IDS's data-store representations are **bounded**; references' grow with workload. The representations *emerged from the joint code+proof loop* (the proof obligation drove the agent to them); performance feedback then steers toward the fastest *verifying* candidate. Detail: suite CC reference uses a key→value chain that grows per put (every get walks it); IDS uses a balanced-tree map (lookup ≤ tree depth) → up to 1.4× throughput, 1.7× lower p99 at low put rate; gap narrows at high put rate as broadcast dominates. On ops-per-worker scaling, the reference's throughput collapses >½ from 1k→20k while IDS loses ~¼.

### Q3 — Ablations & generalization (§5.4)

**Component ablations** (3 runs/spec; VERINA = 189 Lean tasks):

| | full | −J | −P | −R | −A | −VF |
|---|---|---|---|---|---|---|
| Chapar CC | 3/3 | 0/3 | 0/3 | 0/3 | 1/3 | 0/3 |
| RYW | 3/3 | 2/3 | 2/3 | 3/3 | 3/3 | 1/3 |
| MW | 3/3 | 1/3 | 2/3 | 2/3 | 2/3 | 0/3 |
| RYW+MW | 3/3 | 0/3 | 1/3 | 2/3 | 1/3 | 0/3 |
| MR | 3/3 | 1/3 | 0/3 | 1/3 | 0/3 | 0/3 |
| suite CC | 2/3 | 0/3 | 0/3 | 0/3 | 0/3 | 0/3 |
| LCC | 3/3 | 0/3 | 0/3 | 0/3 | 0/3 | 0/3 |
| **VERINA** | **176** | 160 | 150 | 167 | 176 | 110 |

- **−J (no joint discovery):** hard specs → 0/3; VERINA −8%. A frozen implementation limits the proof phase.
- **−VF (Coq feedback → bare accept/reject):** *most consequential* — every spec ≤1/3, VERINA −35%. The structured diagnostic (goal, hypotheses, tactic backtrace) carries the signal, not just the verdict.
- **−P / −R:** each drops the four hardest specs; VERINA −14% / −5%.
- **−A (no audit):** ships vacuous proofs — e.g. a `put-guard` returning `false` unconditionally, trivially satisfying the theorem.
- **−PF (no performance feedback):** full IDS is on average **1.42× faster** across the six specs with references.

**DSA as a general prover** (Table bench-results — first 4 are proof-only, so IDS reduces to its DSA):

| Method | DafnyBench | miniCodeProps | Verus-Bench | CoqStoq | VERINA | AlgoVeri | CloverBench |
|---|---|---|---|---|---|---|---|
| Prior SOTA | 52/100 | 49/100 | 137/150 | 28/100 | 38/189 | 31/77 | — |
| Codex | 78 | 80 | 148 | 46 | 136 | 51 | 62/62 |
| Claude Code | 76 | 86 | 148 | 51 | 149 | 56 | 62/62 |
| **IDS** | **88** | **100** | 149 | **97** | **176** | **65** | 62/62 |

- Saturates miniCodeProps (100/100) and near-saturates Verus-Bench (149/150); beats prior SOTA by **36–69%** on DafnyBench and CoqStoq.
- The IDS architecture adds **+10%** (DafnyBench), **+20%** (miniCodeProps), **+51%** (CoqStoq) over Codex under the same prompt — whereas merely swapping the base model (Codex→Claude) gains ≤6%. **It's the method, not the model.**
- Code-and-proof: **176/189** VERINA (prior 38), **65/77** AlgoVeri (2× prior 31), **62/62** CloverBench.

---

## 7. Discussion, limitations, conclusion (§ Conclusion)

**Language/domain-agnostic.** IDS needs a backend with (i) partial-proof checking, (ii) a programmatic interface an agent can drive, (iii) executable extraction for performance feedback. Lean and Verus qualify. Coq chosen for ecosystem maturity; KV stores chosen because Chapar provided a complete formal spec to extend.

**Limitations (three):**
1. Requires a formal Coq **specification as input** — writing complete specs is itself heavy expert work.
2. The 7 specs cover correctness-critical behavior but **not the full operational envelope** (scale-out, reconfiguration, fault recovery, logging/observability).
3. Empirical evaluation is **distributed KV stores only**; other domains (OS protocols, crypto) are future work.

**The specification bottleneck (the big open problem).** Any guarantee is only as strong as its spec. Two proposed directions: (a) *adversarial spec synthesis* — an agent interrogates a human in natural language until a formal spec emerges; (b) *spec extraction from existing systems* — recover correctness properties of a deployed system while leaving the design space open, so IDS can synthesize faster alternatives satisfying the same properties.

**Conclusion.** Verified-systems construction shifts from a **human-labor bottleneck** to a **compute-driven** approach — generalizes to any domain with a machine-checkable correctness oracle (OS kernels, compilers, crypto protocols, hardware). "Vibe coding → verified coding."

---

## 8. Critical reading notes (for your review)

**Strengths**
- Clean, falsifiable central claim with a strong controlled baseline: *same model, prompt, verifier, budget* — so the 2/7 → 7/7 jump genuinely isolates the architecture rather than model strength.
- The −VF ablation (accept/reject vs structured diagnostic: −35%) is a genuinely informative result — it pinpoints *what* signal matters, not just *that* the system works.
- Performance emerging from the proof obligation (bounded representations) is an elegant, non-obvious finding, and the "method not model" ablation (≤6% from model swap vs +51% from architecture) is convincing.
- Releases six new Coq specs as a benchmark — real artifact contribution.

**Things to probe / potential weaknesses**
- **Statistical power:** 3 runs/spec. suite CC at 2/3 is one run from failure; with N=3, single-run variance is large despite the "≤7% relative std" claim (which is about timing/cost, not pass/fail).
- **Spec authorship overlap:** six of seven specs are authored *by the team* ("co-developed with an expert"). Difficulty calibration and any implicit fit between spec style and the agent's decomposition patterns are hard to rule out. Chapar (external) is the cleanest data point.
- **Cost framing:** "200× faster than experts" compares wall-clock hours to human months but excludes the human effort to write the *spec*, which the paper itself flags as the remaining bottleneck. The honest framing is "200× on the proof, given the spec."
- **Generality is asserted, not shown:** language/domain-agnosticism is conjectured; all empirics are Coq + KV stores. The cross-language prover results (Dafny/Lean/Verus) support the *prover* generality but not the *distributed-synthesis* generality.
- **Reference baselines for performance:** "reference times out" (MW, RYW+MW) is a strong claim — worth checking in the appendix whether the reference is a fair/optimized implementation or a naive one.
- **Reproducibility:** runs depend on Codex/GPT-5.4 and Claude/Opus 4.6 behavior + a cloud cluster; exact reproduction is model- and infra-dependent.

**Questions I'd ask the authors**
1. How sensitive are results to the number of runs — what's the pass-rate distribution at N=10?
2. How much expert time went into each of the six new specs, and does spec style affect IDS's success?
3. Does the reloader's design-space search transfer to a domain *without* a clean "change the data representation" trick (the recurring win here)?
4. What's the failure mode on suite CC's 1/3 miss — strategic dead-end the ISA never escaped, or budget exhaustion mid-proof?
