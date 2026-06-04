# Speaker Transcript — Inductive Deductive Synthesis (IDS)

**Target:** ~8 min talk + Q&A · 12 slides
**How to use:** Each slide has a **time budget**, a **word-for-word script** (≈ speaking pace 140 wpm), and a **cue** for when to advance. Total scripted time ≈ 8:05, leaving buffer. Two slides are optional asides — the **Rocq/Coq** slide (skippable comic relief) and the **Q&A** slide (a backstop for the discussion); cut either if you're running long.

---

## Slide 1 — Title  ·  [0:00–0:25]  ·  ~25 sec

> "Hi everyone. I'm going to talk about **Inductive Deductive Synthesis**, or IDS — a method for getting AI agents to generate systems that are *formally verified*, not just tested. This is work out of Berkeley, Google, and UC Santa Cruz. The one-sentence version: we make a coding agent write the code **and** its mathematical proof of correctness at the same time, so the result is guaranteed correct on every input — and we do it for real distributed systems."

**Cue:** advance as you say "every input."

---

## Slide 2 — The Problem  ·  [0:25–1:25]  ·  ~60 sec

> "Here's the gap we're closing. Coding agents are *great* at a lot: they generate code, run tests, and refine their own output. But they only ever cover the cases they happen to try. What they **can't** do is give you a guarantee that holds across *every* possible behavior.
>
> Distributed systems are the textbook example. A property like consistency between reads and writes has to hold under *every* interleaving of messages, failures, and concurrent updates. That's a combinatorial space no test suite can ever cover — and when a bug hides in there, it surfaces at scale, often catastrophically: a store that loses data, a filesystem that drops a write, a key that leaks."

**Cue:** advance after "drops a write, a key that leaks."

---

## Slide 3 — The Evidence  ·  [1:25–2:25]  ·  ~60 sec

> "So the natural question is: can't we just hand a coding agent a proof assistant and ask it to prove its own code? We tried. The answer is **no**.
>
> A verified system has three parts — a *specification* of what correct means, an *implementation*, and a machine-checked *proof* that the two agree. The catch is that writing these proofs by hand takes **months to years**. Chapar's key-value-store proof took nine to twelve months of expert effort.
>
> And when we give state-of-the-art agents — Codex on GPT-5.4 and Claude Code on Opus 4.6 — a proof assistant and ask them to do it, they solve only **two out of seven** specs. The failure is *structural*: they treat verification as a downstream check. Write all the code, then try to prove it all at once. That means writing the entire proof in one shot, and getting zero correctness feedback until the very end. So the agent just stalls."

**Cue:** advance on "the agent just stalls."

---

## Slide 4 — The Key Idea  ·  [2:25–3:20]  ·  ~55 sec

> "Our key idea is simple: don't write the code first and prove it later. Synthesize the code **and** the proof **jointly and incrementally** — every implementation step paired with a proof step.
>
> This buys two things. First, each little joint step is *much* easier to prove than one giant proof at the end. Second — and this is the important one — the proof assistant can check **partial** proofs. So a dead-end design gets ruled out *before* you waste effort on it, and a failed step becomes a counterexample that improves the next attempt.
>
> Think of it like chain-of-thought reasoning — except every intermediate step is formally encoded and *verified*, not just plausible."

**Cue:** advance on "not just plausible." *(The figure on the left shows code and proof advancing together — gesture to it.)*

---

## Slide 5 — Background: the Oracle  ·  [3:20–4:20]  ·  ~60 sec

> "Why does partial checking even work? Because the proof assistant — here Rocq, formerly Coq — is a **perfect oracle**. It accepts a proof *if and only if* it's correct. No false positives, no false negatives. Compare that to tests, which only see the inputs you tried; static analysis, which over-approximates; or an LLM judge, which just guesses.
>
> The trick is on the left. Unproven goals are written as **deferred holes** — `Admitted` — and the file *still type-checks*. So the agent always knows whether it's on track or in a dead end, without having finished the proof.
>
> This counter is the toy version. Scale it up and the axioms become read-your-writes consistency, one number becomes a per-client vector clock, and a one-line tactic becomes an inductive simulation with dozens of lemmas."

**Cue:** advance on "dozens of lemmas."

---

## Slide 5b — Aside: Rocq or Coq?  ·  [~10 sec, optional]

> "Quick aside, since I keep saying *Rocq, formerly Coq* — yes, that's real. The Coq proof assistant was officially renamed **Rocq** as of version 9.0. Part of it is honoring *Inria Rocquencourt*, where it started — and part of it, per the dev team, is avoiding the unfortunate slang. Same tool, new name. Okay, back to the system."

**Cue:** *(Optional comic-relief slide — it's just the screenshot. Skip entirely if running long.)* Advance on "back to the system."

---

## Slide 6 — The System  ·  [4:20–5:15]  ·  ~55 sec

> "We realize this as a multi-agent system. Two roles matter.
>
> The **Deductive Synthesis Agent** — DSA, on the left — recursively breaks a component and its spec into sub-parts using placeholders, and the type-checker grades *every* step, complete or partial.
>
> When it gets stuck, the **Inductive Synthesis Agent** steps in with two modes: a **proposer** that suggests helper lemmas for a tactical stall, and a **reloader** that throws out the design and respawns the DSA with a fresh one when it hits a true dead end.
>
> A coordinator runs these in parallel, **benchmarks** candidates eagerly — even before the proof closes — and **audits** finished proofs to make sure they're not vacuous."

**Cue:** advance on "not vacuous."

---

## Slide 7 — Q1: Hard Specifications  ·  [5:15–6:05]  ·  ~50 sec

> "Now the results. First question: does it handle hard specs? **Seven out of seven**, versus two out of seven for both baselines under the same prompt and budget.
>
> Look at Chapar causal consistency, top row — the baselines fail, IDS closes it in ten hours for a hundred and forty-eight dollars. That proof took human experts nine to twelve months. So that's roughly **two-hundred times faster**, with no human in the loop.
>
> The recurring trick is in the data: IDS *backtracks* and changes the storage layout — one entry per key, say — so the proof splits into small, easy cases. The baselines keep the default layout, never backtrack, and stall on the same hard cases."

**Cue:** advance on "stall on the same hard cases."

---

## Slide 8 — Q2: Performance  ·  [6:05–6:55]  ·  ~50 sec

> "Second question: are these verified implementations actually *fast*? Yes — they match or beat **every** hand-written expert reference. Up to **three times** the throughput on Chapar's vector-clock store, one-point-four times on causal consistency and monotonic reads. On two specs the expert reference just times out.
>
> And here's the nice part: that speed wasn't bolted on. The *proof obligation itself* pushed the agent toward bounded data representations, and then the benchmark feedback steered it to the fastest among the candidates that actually verify. Correctness and performance came out of the same loop."

**Cue:** advance on "the same loop."

---

## Slide 9 — Q3: Ablations & Generality  ·  [6:55–7:35]  ·  ~40 sec

> "Third: what's actually doing the work? Two things matter most. Remove the **joint** code-and-proof discovery and the hard specs collapse to zero. And the single most important component is the **rich Coq feedback** — if you replace the detailed error trace with a bare accept-or-reject, performance drops thirty-five percent.
>
> Finally, this isn't Coq-specific or distributed-systems-specific. The agent alone sets a new state of the art across four verification languages — Dafny, Lean, Verus, Coq — and on code-and-proof benchmarks like VERINA it goes from a prior best of thirty-eight to a hundred and seventy-six."

**Cue:** advance on the VERINA number. *(Gesture to the bar chart on the left.)*

---

## Slide 12 — Conclusion  ·  [7:35–7:55]  ·  ~20 sec

> "So, to wrap up: verified software used to be a *person-year* problem. IDS turns it into a *compute* problem — joint, incremental code-and-proof under a partial-proof oracle. Seven out of seven, three times faster, two-hundred times cheaper in human time, and it generalizes to any domain with a machine-checkable oracle. The next bottleneck is writing the *specification* itself. Happy to take questions."

**Cue:** stop. Open for Q&A.

---

## Slide 11 — Anticipated Q&A (on-screen backstop)

> *(This is now a slide — the six common questions are on screen as a grid. Don't read it top to bottom; let the audience ask, then point to the relevant tile and answer from the notes below. If no questions come, you can volunteer one: "A question I usually get is whether this just shifts the burden to writing the spec — and the honest answer is yes…")*

**Cue:** advance to the conclusion when discussion winds down (or skip straight here if you prefer to close on the summary).

The six tiles, with fuller answers:

- **"Doesn't this just shift the burden to writing the spec?"**
  Yes — and the paper says so explicitly. Writing the formal spec is now the bottleneck. They point to two future directions: adversarial spec synthesis (an agent interrogating a human in natural language) and spec extraction from existing systems.

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

These are the ones a skeptical reviewer asks. The move on each: **acknowledge the limit honestly, then redirect to what the data does support.**

- **"Only 3 runs per spec — isn't suite CC at 2/3 basically one bad run from failure? How robust is this?"**
  Fair — N=3 is small, and suite CC is the honest weak point (the paper reports it as 2/3, doesn't hide it). The "≤7% relative std" in the paper is about *timing and cost*, not pass/fail. What I'd lean on: the *gap* is large and consistent — baselines are 0/3 on five specs, IDS is 3/3 on six. That separation is hard to explain by run-to-run noise. Tightening the run count is exactly the kind of follow-up I'd want.

- **"Six of the seven specs were written by your own team. Aren't you grading your own homework?"**
  Honest answer: yes, six are co-developed with an expert and released as the IDS suite; only Chapar is fully external. Two mitigations worth stating: (1) Chapar is the cleanest data point and IDS closes a proof that took experts 9–12 months; (2) the baselines run on the *same* specs under the *same* prompt and budget, so any spec-style bias would help them too. But I'd agree the external-spec evidence is one data point, and broadening it is future work.

- **"You say 200× faster than experts — but that's the proof, given the spec. Writing the spec is the hard part, which you admit. Isn't that misleading?"**
  Good catch, and the paper makes the same admission — the spec is now the bottleneck. The honest framing is "**200× on the proof, given the spec**," and I should say it that way. The claim isn't end-to-end automation of verified systems; it's automating the months-to-years *proof* effort once a spec exists.

- **"The whole win seems to be 'backtrack and change the data representation.' Is IDS general, or did it just learn one trick that happens to fit KV stores?"**
  That's the sharpest version of the generality question. The recurring win here *is* representation change (per-key, per-(key,client)). The counter-evidence for generality is the DSA-as-prover result: it sets SOTA across Dafny, Lean, Verus, and Coq on problems with no such trick. So the *prover* generality is shown; the *distributed-synthesis* generality beyond KV stores is conjectured, not demonstrated.

- **"'Reference times out' on MW/RYW+MW is a strong claim — was that a fair, optimized reference or a naive one?"**
  These are the expert-written references released with each spec. I'd point them to App. perf-curves for the implementation detail rather than overclaim from the stage. The mechanism is concrete: the reference's key→value closure chain grows per put and every get walks it, so it degrades with workload while IDS's bounded representation doesn't.

- **"Isn't this just prompt + a good model? How do I know it's the architecture?"**
  This one the data answers cleanly: swapping the base model under the same prompt (Codex→Claude, no IDS architecture) gains ≤6% on the cross-language benchmarks, while the IDS architecture adds +10% to +51%. And the −VF ablation shows the *structured* Coq feedback (not just accept/reject) is worth 35 points on VERINA. It's the method.

- **"How is the proof trusted end-to-end? Could the agent weaken the spec or stub something?"**
  The audit step (④) is exactly this guard: no remaining `Admitted`/axioms, the spec is immutable by construction, the real interface must be implemented, and the implementation must be non-trivial. They caught a `put-guard` returning `false` unconditionally — the audit rejects that class of vacuous proof.

---

## Timing summary

| Slide | Topic | Budget | Running |
|---|---|---|---|
| 1 | Title | 0:25 | 0:25 |
| 2 | Problem | 1:00 | 1:25 |
| 3 | Evidence (2/7) | 1:00 | 2:25 |
| 4 | Key idea | 0:55 | 3:20 |
| 5 | The oracle | 1:00 | 4:20 |
| 5b | Rocq/Coq aside *(optional)* | 0:10 | 4:30 |
| 6 | Architecture | 0:55 | 5:25 |
| 7 | Q1 — 7/7 | 0:50 | 6:15 |
| 8 | Q2 — perf | 0:50 | 7:05 |
| 9 | Q3 — ablations | 0:40 | 7:45 |
| 11 | Q&A backstop *(during discussion)* | — | — |
| 12 | Conclusion | 0:20 | 8:05 |

**If you're running long:** drop the **Rocq/Coq aside** (5b) and compress slides 5 and 9 first (the most cuttable). The Q&A slide (11) only costs time if questions come. **If short:** expand the Chapar backtracking story on slide 7 — it's the most memorable concrete result.
