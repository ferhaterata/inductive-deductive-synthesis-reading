# Speaker Transcript — Inductive Deductive Synthesis (IDS)

**Target:** ~9 min talk · 9 slides, ~1 minute each · + 2 appendix slides (Q&A, Rocq/Coq) held in reserve
**How to use:** Each slide has a **~1-min word-for-word script** (≈ 140 wpm ≈ 140 words) and a **cue** for when to advance. Keep the *nouns* on the slide and the *sentences* in your mouth.

---

## Slide 1 — Title  ·  ~1 min

> "Hi everyone. I'm going to talk about **Inductive Deductive Synthesis** — IDS — a method for getting AI agents to generate systems that are *formally verified*, not just tested. This is work out of UC Berkeley, Google, and UC Santa Cruz.
>
> The one-sentence version: today's coding agents are great at writing and *testing* code, but testing only ever covers the cases you happen to try. For something like a distributed system, that's not enough — you want a guarantee that holds on *every* possible behavior. IDS gets you that guarantee by having the agent write the code **and** a machine-checked mathematical proof of its correctness, together. And it does it for real distributed key-value stores."

**Cue:** advance on "distributed key-value stores."

---

## Slide 2 — What is formal verification?  ·  ~1 min

*(This is the gentle intro for anyone not steeped in formal methods.)*

> "Let me start with what formal verification even is. A formally verified system has **three pieces**. The **specification** is a precise *mathematical* statement of what correct means. The **implementation** is the code that actually runs. And the **proof** is a machine-checkable argument that, on *every* input, the implementation matches the specification.
>
> Historically, **humans wrote all three** — and that's exactly why formal methods barely scaled over the last two decades: verifying a real system takes *months to years* of expert effort. Chapar's key-value-store proof took nine to twelve months.
>
> So the natural question: can we just hand a coding agent the **spec and the proof tooling** and let it produce a provably-correct system? **IDS** does precisely this — the human writes only the spec, and IDS automatically produces *both* the implementation and the proof, using LLM agents driven by the proof assistant **Rocq**."

**Cue:** advance on "the proof assistant Rocq." *(Gesture across the three pieces, then the then/now boxes.)*

---

## Slide 3 — The key insight  ·  ~1 min

> "Here's why naively handing an agent a proof assistant *doesn't* work, and what IDS does instead.
>
> The naive approach treats verification as a downstream check: write *all* the code, then try to prove it all at once. That means one giant proof and zero correctness feedback until the very end — so the agent just stalls. In fact, state-of-the-art agents given a proof assistant solve only **two of seven** of our specs.
>
> IDS's insight is to synthesize the code **and** the proof *jointly and incrementally* — every implementation step paired with a proof step. This buys two things: each little step is far easier to prove than one monster proof, and the proof assistant can check **partial** proofs, so a dead-end design is ruled out *before* you waste effort on it. Think chain-of-thought reasoning — except every intermediate step is formally *verified*, not just plausible. And a failed step becomes a counterexample that improves the next attempt."

**Cue:** advance on "improves the next attempt." *(Point to the joint code+proof figure.)*

---

## Slide 4 — The oracle  ·  ~1 min   *(slide locked — do not edit)*

> "Why does checking *partial* proofs even work? Because the proof assistant — Rocq, formerly Coq — is a **perfect oracle**: it accepts a file *if and only if* it's correct. No false positives, no false negatives. Compare that to tests, which only see inputs you tried.
>
> The figure is the toy example — a counter. The **spec**, on the left, states two axioms about `init`, `inc`, and `read`: reading the initial state returns zero, and reading after an increment returns one more than before. In the **partial** synthesis, in the center, `inc`'s body and the `read_inc` theorem are deferred with `Admitted` — a placeholder for unfinished work — but Rocq *still accepts the file*, because the chosen representation, a list whose length encodes the count, is consistent with the work so far. The **complete** synthesis on the right fills in those holes. The representation is a free choice — any state satisfying the axioms works, though a plain `nat` would be more efficient.
>
> Scale this up and the axioms become read-your-writes consistency, that one number becomes a per-client vector clock, and the one-line tactic becomes an inductive simulation with dozens of lemmas."

**Cue:** advance on "dozens of lemmas."

---

## Slide 5 — Why "Inductive Deductive"?  ·  ~1 min

> "A quick word on the name, because it tells you exactly how the system works. IDS fuses two classic, *opposite* synthesis strategies.
>
> **Deductive** means reasoning *downward* from rules you already trust: you start from the spec and inference rules and mechanically derive an implementation and proof that *must* satisfy it. It's truth-preserving — guaranteed correct — but only if you're following a good strategy; pick a bad design and deductive steps just grind to a halt with no idea how to recover.
>
> **Inductive** is the opposite: reasoning *upward* from examples — here, generalizing from *failed attempts* to form a better hypothesis about which strategy will work. It's how you discover good strategies in the first place, but generalization guarantees nothing on its own.
>
> So they're complementary. Deduction guarantees correctness *under* a strategy; induction *discovers* better strategies from what failed. IDS's contribution is putting both in one feedback loop — and that maps directly onto two agent types, which is the next slide. One note: 'inductive' here means learning-from-failure, *not* the mathematical induction the proofs happen to use internally."

**Cue:** advance on "the next slide" (or after the parenthetical if the audience looks technical).

---

## Slide 6 — The system  ·  ~1 min

> "We realize this as a multi-agent system. Two roles matter.
>
> The **Deductive Synthesis Agent**, on the left, recursively breaks a component and its spec into sub-parts using placeholders, and the type-checker grades *every* step — complete or partial.
>
> When it gets stuck, the **Inductive Synthesis Agent** steps in with two modes: a **proposer** that suggests helper lemmas for a tactical stall, and a **reloader** that throws out the design and respawns a fresh one when it hits a true dead end.
>
> A **coordinator** runs these in parallel, **benchmarks** candidates eagerly — even before the proof closes — and **audits** finished proofs to make sure they're not vacuous. That audit is what stops the agent from cheating its way to a trivial 'proof.'"

**Cue:** advance on "trivial proof."

---

## Slide 7 — Results: 7/7  ·  ~1 min

> "So, does it work? **Seven out of seven** specs, versus *two out of seven* for both baselines — under the same prompt and the same budget.
>
> Look at the top row, Chapar causal consistency: both baselines fail, IDS closes it in ten hours for a hundred and forty-eight dollars. That proof took human experts nine to twelve months — so that's roughly **two hundred times faster**, with no human in the loop and no fine-tuning.
>
> The recurring trick is in the *data representation*: IDS **backtracks** and changes the storage layout — say, one entry per key — so the proof splits into small, easy cases. The baselines keep the default layout, never backtrack, and stall on exactly the hard specs."

**Cue:** advance on "stall on exactly the hard specs."

---

## Slide 8 — Performance & what drives it  ·  ~1 min

> "Two more questions. First: are these verified implementations actually *fast*? Yes — they match or beat **every** hand-written expert reference. Up to **three times** the throughput on Chapar's vector-clock store, one-point-four times on causal consistency and monotonic reads; on monotonic writes the expert reference just *times out*. And that speed wasn't bolted on — the *proof obligation itself* pushed the agent toward bounded data representations, and benchmark feedback then steered it to the fastest candidate that actually verifies.
>
> Second: what's *driving* the result? Two things matter most. Remove the **joint** code-and-proof discovery and the hard specs collapse to zero. And the single biggest lever is **rich Rocq feedback** — replace the detailed error trace with a bare accept-or-reject and you lose thirty-five percent."

**Cue:** advance on "thirty-five percent."

---

## Slide 9 — Conclusion  ·  ~1 min

> "To wrap up: verified software used to be a *person-year* problem. IDS turns it into a *compute* problem — joint, incremental code-and-proof under a partial-proof oracle, with failure *and* performance feedback in the same loop.
>
> The headline numbers: seven out of seven specs, three times faster than expert references, two hundred times faster than human proof effort, and six new Coq specifications released as a benchmark. And it generalizes to any domain with a machine-checkable oracle — kernels, compilers, crypto, hardware.
>
> The honest catch — and the paper says this outright — is that the bottleneck has *moved*: writing the **specification** is now the hard part, because a guarantee is only as good as its spec. Happy to take questions."

**Cue:** stop. Open for Q&A. *(Jump to the appendix slides if a question lands on one.)*

---

# Appendix slides (held in reserve — not part of the 8-minute flow)

## Appendix A — Anticipated Questions  *(slide 10)*

*(A backstop grid. Don't read it; let the audience ask, then point to the relevant tile. If no questions come, volunteer: "A question I usually get is whether this just shifts the burden to writing the spec — and the honest answer is yes…")*

- **"Doesn't this just shift the burden to writing the spec?"**
  Yes — and the paper says so explicitly. Writing the formal spec is now the bottleneck. Two future directions: adversarial spec synthesis (an agent interrogating a human in natural language) and spec extraction from existing systems.

- **"Is the proof trusted, or could the agent cheat?"**
  The audit step rejects vacuous proofs — no remaining `Admitted`/axioms, the real interface must be implemented, the implementation must be non-trivial, and the correct theorem must be stated. They caught an agent shipping a `put-guard` that returned `false` unconditionally.

- **"Why Coq and not Lean/Verus?"**
  Maturity of the ecosystem, plus Chapar gave them a complete formal spec to start from. IDS only needs a backend with partial-proof checking, a programmatic interface, and executable extraction — Lean and Verus both qualify.

- **"How much does a run cost / how long?"**
  ~6.8 hours and ~$106 per spec on average; max 11 hours / $155. Same model and budget as the baselines, so the comparison isolates the architecture.

- **"Is the win the model or the method?"**
  The method. Swapping the base model (Codex → Claude) under the same prompt gains at most 6%; the IDS architecture adds +10% to +51% on the cross-language benchmarks.

- **"Does it work beyond key-value stores?"**
  Empirically evaluated only on distributed KV stores; the DSA-as-general-prover results across four languages suggest the technique transfers, but other domains (OS protocols, crypto) are future work.

### Tougher / critical questions (be ready, stay calm)

The move on each: **acknowledge the limit honestly, then redirect to what the data does support.**

- **"Only 3 runs per spec — isn't suite CC at 2/3 basically one bad run from failure?"**
  Fair — N=3 is small, and suite CC is the honest weak point. But the *gap* is large and consistent: baselines are 0/3 on five specs, IDS is 3/3 on six. That separation is hard to explain by run-to-run noise.

- **"Six of the seven specs were written by your own team. Aren't you grading your own homework?"**
  Yes — six are co-developed and released as the IDS suite; only Chapar is fully external. Mitigations: Chapar is the cleanest data point (a 9–12-month proof), and the baselines run on the *same* specs under the *same* budget, so any spec-style bias would help them too.

- **"200× faster than experts — but that's the proof, given the spec. Isn't that misleading?"**
  The honest framing is "**200× on the proof, given the spec**." The claim isn't end-to-end automation; it's automating the months-to-years *proof* effort once a spec exists.

- **"Is the whole win just 'change the data representation'?"**
  That recurring win *is* representation change. The counter-evidence for generality is the DSA-as-prover result — SOTA across Dafny, Lean, Verus, and Coq, problems with no such trick. Prover generality is shown; distributed-synthesis generality beyond KV stores is conjectured.

## Appendix B — Rocq or Coq?  *(slide 11)*

> "Quick aside, since I keep saying *Rocq, formerly Coq* — yes, that's real. The Coq proof assistant was officially renamed **Rocq** as of version 9.0. Part of it honors *Inria Rocquencourt*, where it started — and part of it, per the dev team, is avoiding the unfortunate slang. Same tool, new name."

*(Screenshot-only slide. Pure comic relief; use only if asked.)*

---

## Timing summary

| Slide | Topic | Budget | Running |
|---|---|---|---|
| 1 | Title | 1:00 | 1:00 |
| 2 | What is formal verification (then vs now) | 1:00 | 2:00 |
| 3 | Key insight — joint & incremental | 1:00 | 3:00 |
| 4 | The oracle *(locked)* | 1:00 | 4:00 |
| 5 | Why "Inductive Deductive"? | 1:00 | 5:00 |
| 6 | The system | 1:00 | 6:00 |
| 7 | Results — 7/7 | 1:00 | 7:00 |
| 8 | Performance & what drives it | 1:00 | 8:00 |
| 9 | Conclusion | 1:00 | 9:00 |
| — | *Appendix A — Q&A grid* | reserve | — |
| — | *Appendix B — Rocq/Coq* | reserve | — |

**If running long:** slides 5 and 8 are the most compressible (5 sets up 6, so you can merge their delivery). **If short:** expand the Chapar backtracking story on slide 7 — it's the most memorable concrete result.
