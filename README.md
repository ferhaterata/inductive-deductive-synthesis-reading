# Inductive Deductive Synthesis — Paper Reading

A reading/presentation of **Inductive Deductive Synthesis (IDS)** — a technique for
jointly synthesizing an implementation *and* its machine-checked proof, realized as a
multi-agent LLM system that autonomously generates **formally verified distributed systems**.

> This repo holds my talk materials (slides, transcript, summaries) for a paper-reading
> session. It is **not** the original work. The paper is by Agarwal et al.
> (UC Berkeley / Google / UC Santa Cruz); the authors' code lives at
> [skydiscover-ai/skydiscover](https://github.com/skydiscover-ai/skydiscover).

## The paper in one line

AI agents can write and test code but can't *guarantee* correctness over every behavior.
IDS uses a proof assistant (Rocq/Coq) as a perfect oracle that grades even partial work,
synthesizing code + proof incrementally — solving **7/7** distributed KV-store consistency
specs (SOTA agents solve 2/7), and producing implementations up to **3× faster** than
published verified references.

## Contents

- [`slides/ids-talk-8min.html`](slides/ids-talk-8min.html) — the 8-minute talk (open in a browser)
- [`slides/IDS-talk-transcript.md`](slides/IDS-talk-transcript.md) — speaker transcript
- [`slides/IDS-paper-summary.md`](slides/IDS-paper-summary.md) — condensed summary
- [`slides/IDS-paper-full-summary.md`](slides/IDS-paper-full-summary.md) — full summary
- [`slides/assets/`](slides/assets/) — figures used in the talk
- [`paper/`](paper/) — the paper's LaTeX source (arXiv e-print, `main.tex` + sections/specs/tables/figures), included for reference only — copyright remains with the authors

## Viewing the talk

```sh
open slides/ids-talk-8min.html   # macOS
```

## Reference

Agarwal, Krentsel, Liu, Cemri, et al. *Inductive Deductive Synthesis: Enabling AI to
Generate Formally Verified Systems.* arXiv:2605.23109.
