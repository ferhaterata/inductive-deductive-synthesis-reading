# Speaker Transcript — Inductive Deductive Synthesis (IDS)

**Target:** ~10 min talk · 11 slides + 1 appendix, ~1 minute each
**How to use:** Each slide has a **~1-min word-for-word script** (≈ 140 wpm) and a **cue** for when to advance. Keep the *nouns* on the slide and the *sentences* in your mouth.

**Live slide order:** 1 Title · 2 Why not just test? · 3 What is formal verification? · 4 Rocq/Coq aside · 5 The oracle · 6 The system · 7 Results 7/7 · 8 Cross-language generality · 9 Related work · 10 Anticipated questions · 11 Conclusion. **Appendix (after conclusion):** Why "Inductive Deductive"?

---

## Slide 1 — Title  ·  ~1 min

> "Hi everyone. I'm going to talk about **Inductive Deductive Synthesis** — IDS — a method for getting AI agents to generate systems that are *formally verified*, not just tested. This is work out of UC Berkeley, Google, and UC Santa Cruz.
>
> The one-sentence version: today's coding agents are great at writing and *testing* code, but testing only ever covers the cases you happen to try. For something like a distributed system, that's not enough — you want a guarantee that holds on *every* possible behavior. IDS gets you that guarantee by having the agent write the code **and** a machine-checked mathematical proof of its correctness, together. And it does it for real distributed key-value stores."

**Cue:** advance on "distributed key-value stores."

---

## Slide 2 — Why not just test?  ·  ~1 min

> "Let me motivate why we need proof at all. **Testing samples; verification proves.**
>
> A test suite checks only the inputs you *happen to try* — so a bug in any untried case slips straight through. As Dijkstra put it, testing shows the *presence* of bugs, never their absence. **Verification** is the opposite: one proof covers *every* input at once, and for a concurrent system, *every* interleaving of messages and failures. It's a guarantee of the **absence** of bugs.
>
> And that's exactly what distributed systems need. The space of behaviors — every ordering of messages, every failure, every concurrent update — is astronomically large; no test suite can cover it. IDS gives the agent a guarantee that holds on *every possible behavior*: the code **and** a machine-checked proof, together."

**Cue:** advance on "machine-checked proof, together."

---

## Slide 3 — What is formal verification?  ·  ~1 min

*(Gentle intro for anyone not steeped in formal methods.)*

> "So what *is* formal verification? A formally verified system has **three pieces**. The **specification** is a precise *mathematical* statement of what correct means. The **implementation** is the code that actually runs. And the **proof** is a machine-checkable argument that, on *every* input, the implementation matches the specification.
>
> Historically, **humans wrote all three** — and that's exactly why formal methods barely scaled over two decades: verifying a real system takes *months to years* of expert effort. Chapar's key-value-store proof took nine to twelve months.
>
> So the natural question: can we just hand a coding agent the **spec and the proof tooling** and let it produce a provably-correct system? **IDS** does precisely this — the human writes only the spec, and IDS automatically produces *both* the implementation and the proof, using LLM agents driven by the proof assistant **Rocq**."

**Cue:** advance on "the proof assistant Rocq." *(That word sets up the next slide.)*

---

## Slide 4 — Aside: Rocq or Coq?  ·  ~15–20 sec (optional)

> "Quick aside, since I just said *Rocq* — yes, that's the proof assistant you may know as **Coq**. It was officially renamed **Rocq** as of version 9.0: partly to honor *Inria Rocquencourt*, where it started, and partly — per the dev team — to avoid the unfortunate English slang. Same tool, new name. I'll say Rocq from here on."

**Cue:** advance quickly — this is comic relief, don't dwell. *(Skip entirely if running long.)*

---

## Slide 5 — The oracle  ·  ~1.5 min   *(slide locked — do not edit the slide; this is the code walkthrough)*

> "Here's *why* the partial-proof idea works: the proof assistant is a **perfect oracle** — it accepts a file *if and only if* it's correct. No false positives, no false negatives, unlike tests.
>
> The figure is the toy example — a **counter**. Let me actually read it.

**The spec (left).** It declares an abstract counter: a state type `t`, an initial state `init`, an `inc` operation that takes a state to a state, and a `read` that turns a state into a natural number. Then two **axioms** — these are the correctness requirements, in math:
>   - `read_init`: `read init = 0` — *"reading the initial state returns zero."*
>   - `read_inc`: `forall s, read (inc s) = S (read s)` — *"for every state s, reading after one increment gives the **successor** of reading before."* `S` is Peano successor, so `S n` is just `n + 1`. In English: **incrementing always bumps the count up by exactly one.**

**The partial synthesis (center).** The agent makes a design choice: represent the state as a **list of `unit`** — `unit` is the type with a single value `tt`, so the list carries no data; only its *length* matters. `init` is the empty list `nil`, and `read` is just `length`. It proves `read_init` trivially — `length nil = 0` — but it *defers* `inc`'s body and the `read_inc` theorem with `Admitted`, a placeholder for unfinished work. The key point: **Rocq still accepts this file.** The partial design is internally consistent, so the agent knows it's on track before committing.

**The complete synthesis (right).** It fills the hole: `inc s := tt :: s` — *cons* one `tt` onto the front of the list. Since the list's length *is* the count, prepending one element grows the length by exactly one — which is precisely what `read_inc` demands. The proof closes with a one-line tactic.
>
> "And note: the representation was a *free choice*. Any state satisfying those axioms works — a plain `nat` would actually be more efficient. That freedom to pick the representation is exactly the lever IDS exploits later. Now scale this up: the axioms become read-your-writes consistency, the count becomes a per-client vector clock, and the one-line tactic becomes an inductive simulation with dozens of lemmas."

**Cue:** advance on "dozens of lemmas." *(If short on time: read just the two axioms and the `tt::s` line — that's the core.)*

---

## Slide 6 — The system  ·  ~1 min

> "We realize this as a multi-agent loop. Two roles matter.
>
> The **Deductive Synthesis Agent** — the DSA — takes a strategy and recursively breaks a component and its spec into sub-parts using placeholders, building implementation and proof step by step. Rocq's type-checker grades *every* step — complete or partial — so a yes means on-track, a no rules the path out.
>
> When a DSA stalls, the **Inductive Synthesis Agent** — the ISA — steps in, in two modes: a **proposer** that suggests helper lemmas for a tactical stall, and a **reloader** that throws out the whole design and respawns a DSA with a fresh one on a true dead end. The classic example: the first design stores a replica's state as one big object and the proof gets stuck; the ISA pivots to one small table per key, and the proof splits into easy per-key cases.
>
> A **coordinator** runs DSAs in parallel, **benchmarks** candidates eagerly — even before the proof closes — and **audits** finished proofs so they're not vacuous. That audit is what stops the agent cheating its way to a trivial 'proof.'"

**Cue:** advance on "trivial proof."

---

## Slide 7 — Results: 7/7  ·  ~1 min

> "Does it work? **Seven out of seven** specs, versus *two out of seven* for both baselines — under the same prompt and the same budget. And note the column headers: Codex runs on GPT-5.4, Claude Code on Opus 4.6 — and **IDS runs on GPT-5.4 too**, the same model as Codex. So the jump from two to seven is the *architecture*, not a better model.
>
> Top row, Chapar causal consistency: both baselines fail; IDS closes it in ten hours for a hundred forty-eight dollars. That proof took human experts nine to twelve months — roughly **two hundred times faster**, no human in the loop.
>
> The recurring move is in the *data representation*: IDS **backtracks** and changes the storage layout — one entry per key, say — so the proof splits into small cases. The baselines keep the default layout, never backtrack, and stall on exactly the hard specs."

**Cue:** advance on "stall on exactly the hard specs."

*(If asked about "2/3" or "3/3": it's successful runs out of 3 attempts; a spec counts as solved at ≥2/3. Suite CC at 2/3 is the honest weak point.)*

---

## Slide 8 — Cross-language generality  ·  ~1 min

> "Right after the headline result, let me answer an obvious question: is this just a Rocq trick? No. The DSA is a **language-agnostic agent loop** — they instantiate it per language by editing the agent's brief, a `CLAUDE.md` file, to swap the verifier, the success criterion, and the tool surface, then evaluate each benchmark in its *native* tool: Lean for VERINA and miniCodeProps, Dafny for DafnyBench and CloverBench, Verus for Verus-Bench, Coq for CoqStoq. It was *not* one Rocq run.
>
> On the proof-only benchmarks — where the implementation and spec are already given — the system collapses to just the DSA prover, and it beats the prior state of the art on all four: DafnyBench 88 versus 52, miniCodeProps a perfect 100 versus 49, Verus-Bench 149 versus 137, CoqStoq 97 versus 28. On the harder **code-and-proof** benchmarks, the blue rows, it reaches sixty-two of sixty-two on CloverBench and a hundred seventy-six of one-eighty-nine on VERINA — where the prior best was just thirty-eight."

**Cue:** advance on "the prior best was thirty-eight." *(Compressible — drop to the headline VERINA number if short.)*

---

## Slide 9 — Related work  ·  ~1 min

> "Now where IDS sits in the literature. Verified code generation has three components — spec, implementation, proof — and IDS relates to four threads.
>
> *Specification generation* — the task there is intent-to-spec or code-to-spec; IDS instead *consumes* a spec. *Evaluator-guided program generation* — SWE-agent, FunSearch, AlphaEvolve — same search-against-an-evaluator scheme, but our evaluator is a *proof assistant*, so we get guarantees on *all* inputs, not sampled tests. *Automated proof generation* fixes the impl and spec and asks only for the proof; IDS generates impl and proof *jointly* and also optimizes performance. And *verified synthesis* — Synquid, CEGIS, and hand-built systems like IronFleet, Verdi, Chapar — was either confined to small functions or cost months-to-years by hand. IDS's contribution is verified synthesis at *systems scale, in hours*."

**Cue:** advance on "systems scale, in hours."

---

## Slide 10 — Anticipated questions  ·  reveal on click; use as needed

*(Tiles animate in one per click. Don't read top-to-bottom — let the audience ask and click to the relevant tile, or volunteer one if the room is quiet.)*

- **"Doesn't this just shift the burden to writing the spec?"**
  Yes — the paper says so. The spec is now the bottleneck. Future directions: adversarial spec synthesis (an agent interrogating a human in NL) and spec extraction from existing systems.
- **"Is the proof trusted, or could the agent cheat?"**
  The audit step rejects vacuous proofs — no remaining `Admitted`/axioms, the real interface implemented, non-trivial impl, correct theorem stated. They caught an agent shipping a `put-guard` that returned `false` unconditionally.
- **"Why Coq and not Lean/Verus?"**
  Ecosystem maturity, plus Chapar gave a complete formal spec to start from. IDS only needs partial-proof checking, a programmatic interface, and executable extraction — Lean and Verus both qualify (see slide 10).
- **"Cost / time per spec?"**
  ~6.8 hours and ~$106 on average; max 11 h / $155. Same model and budget as the baselines — isolates the architecture.
- **"Is the win the model or the method?"**
  The method. Swapping the base model (Codex → Claude) under the same prompt gains ≤6%; the IDS architecture adds +10% to +51% on the cross-language benchmarks.
- **"Does it work beyond key-value stores?"**
  Evaluated only on distributed KV stores; the DSA-as-general-prover results across four languages suggest it transfers, but other domains (OS protocols, crypto) are future work.

### Tougher questions (acknowledge the limit, then redirect to what the data supports)

- **"N=3 — isn't suite CC at 2/3 one bad run from failure?"** Fair, N=3 is small and suite CC is the honest weak point. But the *gap* is large and consistent: baselines 0/3 on five specs, IDS 3/3 on six — hard to explain by noise.
- **"Six of seven specs are your own. Grading your own homework?"** Yes — six are co-developed and released as the IDS suite; only Chapar is fully external. But the baselines run on the *same* specs under the *same* budget, so any spec-style bias would help them too.
- **"200× — but that's the proof, given the spec."** Right framing is "200× on the proof, given the spec." Not end-to-end automation; it's automating the months-to-years proof effort once a spec exists.
- **"Is the whole win just 'change the representation'?"** That recurring win *is* representation change — but the DSA-as-prover result (SOTA across Dafny, Lean, Verus, Coq, no such trick) shows prover generality; distributed-synthesis generality beyond KV stores is conjectured.

**Cue:** advance to the conclusion when discussion winds down.

---

## Slide 11 — Conclusion  ·  ~1 min

> "To wrap up: verified software used to be a *person-year* problem. IDS turns it into a *compute* problem — joint, incremental code-and-proof under a partial-proof oracle, with failure *and* performance feedback in the same loop.
>
> The headline: seven of seven specs, three times faster than expert references, two hundred times faster than human proof effort, six new Coq specs released. And it generalizes to any domain with a machine-checkable oracle — kernels, compilers, crypto, hardware.
>
> The honest catch — and the paper says it outright — is that the bottleneck has *moved*: writing the **specification** is now the hard part, because a guarantee is only as good as its spec. Thank you — happy to take questions."

**Cue:** stop.

---

# Appendix (after the conclusion — use only if asked)

## Appendix — Why "Inductive Deductive"?  ·  ~1 min

*(Pulled out of the main flow; jump here if someone asks about the name.)*

> "A word on the name, because it tells you how the system works. IDS fuses two classic, *opposite* synthesis strategies.
>
> **Deductive** = reasoning *downward* from rules you trust: from the spec and inference rules you mechanically derive an implementation and proof that *must* satisfy it. Truth-preserving — guaranteed correct — but only under a good strategy; pick a bad design and deductive steps grind to a halt.
>
> **Inductive** = reasoning *upward* from examples — here, generalizing from *failed attempts* to a better hypothesis about which strategy will work. It's how you discover good designs, but it guarantees nothing on its own.
>
> They're complementary: deduction guarantees correctness *under* a strategy; induction *discovers* better strategies from failure. IDS puts both in one loop — and that maps onto the two agent types, DSA and ISA. One note: 'inductive' here means learn-from-failure, *not* the mathematical induction the proofs happen to use internally."

---

## Timing summary

| Slide | Topic | Budget | Running |
|---|---|---|---|
| 1 | Title | 1:00 | 1:00 |
| 2 | Why not just test? | 1:00 | 2:00 |
| 3 | What is formal verification? | 1:00 | 3:00 |
| 4 | Rocq/Coq aside | 0:20 | 3:20 |
| 5 | The oracle *(code walkthrough)* | 1:30 | 4:50 |
| 6 | The system | 1:00 | 5:50 |
| 7 | Results — 7/7 | 1:00 | 6:50 |
| 8 | Cross-language generality | 1:00 | 7:50 |
| 9 | Related work | 1:00 | 8:50 |
| 10 | Anticipated questions | — | (Q&A) |
| 11 | Conclusion | 1:00 | ~9:50 |
| — | *Appendix — Why "Inductive Deductive"?* | reserve | — |

**If running long:** drop the Rocq/Coq aside (4), compress slides 8 and 9. **If short:** expand the Chapar backtracking story on slide 7 — the most memorable concrete result.
