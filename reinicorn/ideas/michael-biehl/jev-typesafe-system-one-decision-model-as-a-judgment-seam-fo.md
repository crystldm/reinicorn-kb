---
type: idea
title: Jev (TypeSafe System One decision model) as a judgment seam for rcorn gates
slug: jev-typesafe-system-one-decision-model-as-a-judgment-seam-fo
lifecycle: active
status: new
created: 2026-09-22
author: Michael Biehl
origin: ai-assisted
human_validated: false
---

# Jev (TypeSafe System One decision model) as a judgment seam for rcorn gates

## Summary

Research capture (2026-09-22), not a spec. Jev is a new kind of model that
answers typed questions about a block of state instead of generating text.
Reinicorn today makes **zero** model calls: every judgment lives in the coding
agent via skills, and the CLI is deterministic. Several rcorn gates and lints
are mechanical stand-ins for a semantic judgment (mtime for staleness, path
intersection for overlap, "section non-empty" for a filled retro). Jev, or an
open-weights clone of it, is the first model shape cheap and constrained
enough to sit *inside* those gates without turning rcorn into an LLM app.

**One-line conclusion:** add one judgment seam (`state + {Choice, Score,
Noul} → answers with probabilities`) with swappable backends (TypeSafe HTTP,
a local open-weights scorer, or `none`), keep the questions and thresholds in
one reviewable config, and pilot it in shadow mode on spec-drift detection.

## What Jev is

- **Vendor:** TypeSafe AI (San Francisco, founded 2024, CEO Diogo Almeida,
  ex-OpenAI). Released 2026-09-15 in limited early access (waitlist) with a
  $40M seed led by DCVC.
- **Contract:** `POST https://api.typesafe.ai/v1/systemone`, bearer auth.
  Request = `state` (text or JSON) + `questions` map + `model`
  (`jev-latest`; pin a version in production). Three primitives, all
  evaluated in parallel and in isolation against the same state:
  - `Choice` — pick one option from `criteria: {name: description}`
    (max 255); returns `choice`, `probabilities`, `confidence`.
  - `Score` — position on an ordered rubric of 2–10 described levels;
    returns probability-weighted `score`, `probabilities`, `confidence`.
  - `Noul` — yes/no; returns a single probability in 0–1.
- **Why it matters:** no generated tokens, so no parsing, no schema errors,
  70–500 ms end-to-end, $0.042 per million input tokens and free output.
  Vendor-run benchmarks claim 40–200x faster and 40–400x cheaper than frontier
  LLMs on classification-shaped work.
- **What it is not:** it cannot write prose, do arithmetic, compare dates, or
  reason open-endedly. Typed output guarantees the *interface*, not the truth
  of the answer. State is an injection surface. Confidence is distribution
  concentration, not permission to act.
- **Ecosystem:** Python SDK `typesafe-sdk`, JS SDK, LangChain
  `langchain_typesafe`, a Claude Code plugin (`typesafe-ai/skills`, MIT,
  1.7k stars) whose SKILL.md tells agents to hunt for "prompt-and-parse steps
  that could become a structured decision" and to keep questions and
  thresholds in one place.

## Open-source substitutes

All work by reading the next-token logits for the option letters in one
forward pass of a frozen open-weights model, instead of generating JSON.

| Project | Stars | License | Notes |
|---|---|---|---|
| `TheoLeeCJ/openjev` (SemIf) | 3.6k | MIT | CPU via llama.cpp GGUF, MLX on Apple; `semif-score` CLI over JSONL rows; Qwen3.5-4B baseline. Order-sensitive: reversing options flipped 2/36 answers in one test. |
| `ikermoel/open-alternative-jev` (`so1`) | 49 | Apache-2.0 | `Decider.decide(state, questions=[Choice, yes_no])`; HF or vLLM; packed mode ~2.5x faster. Community benchmark: 73.7% vs Jev 1.13's 72.7%, better calibration. |
| `Rizzo-AI-Academy/rizzo-flow` | 311 | — | "The open, local take on Jev". Not evaluated. |
| `tphakala/jev-mcp` | 1 | — | MCP server over TypeSafe API or OpenRouter. |

Takeaway: the *contract* is the durable asset. The clones already speak the
same three primitives, so code written against the contract is not locked to
TypeSafe's waitlist.

## Where it fits reinicorn

Candidates, ranked by how badly the current mechanical stand-in hurts:

1. **Spec drift / spec-aware review** (idea `code-review-should-be-spec-aware`,
   approved spec `process-as-config` asks for spec-drift accounting).
   State = spec sections + PR diff hunks. Per section, `Choice`
   {consistent, contradicts, extends, unrelated} plus `Noul` "this hunk
   changes behavior this section specifies". Surfaced as a warning on the
   `Process gate` check and as a prompt for the retro's Spec Drift section.
2. **Retro and plan gate quality.** `closer-filled` lint checks emptiness;
   a `Noul` "this section states a concrete outcome rather than a
   placeholder" catches `N/A`-style stubs.
3. **Idea dedup at `rcorn idea create`.** `Choice` over existing idea titles
   (well under the 255 cap) with a `none` escape option; print "similar to
   <slug>?" instead of blocking.
4. **Semantic cross-branch overlap.** `check_overlap` intersects file paths.
   A `Noul` over two plans' Goal sections, "these plans change the same
   concern", would catch overlap that touches different files.
5. **Docs gardening.** `docs-freshness` uses mtime > 30 days. Fan-out
   `Noul` per doc, "the doc still describes the code listed", is the
   semantic version the docs-gardening idea asked for.
6. **PR-body process check.** `Noul` per AGENTS.md rule: cites a spec,
   states verification evidence, declares scope boundaries.
7. **Quality scores.** `Score` maps directly to the A–F rubric, but the
   grade is meant to be a human/agent judgment during review; low priority.
8. **Third bundled skill adapter** for `typesafe-ai/skills`. Trivial with
   the adapter infra, but it serves users building products with Jev, not
   rcorn itself. Nice-to-have.

## Shape of the integration

- **One seam, one file** (`reinicorn/judgment.py` or similar): the three
  primitives as frozen dataclasses, a `decide(state, questions) ->
  answers` function, backends selected by config. Validate responses at
  this boundary (golden principle 1). No SDK dependency for the HTTP
  backend; stdlib `urllib` is enough.
- **Backends:** `typesafe` (HTTP), `local` (subprocess or HTTP to an
  openjev/so1 server; never a torch dependency in the `uv tool` install),
  `none` (default; every gate degrades to today's mechanical behavior).
- **Questions and thresholds live in the kb as data**, next to
  `doc-types.yaml`, so a repo can review, tune, or delete them without
  engine changes. This is the process-as-config stance and also TypeSafe's
  own advice.
- **Gates stay warn-level.** A model answer never blocks a push or merge on
  its own; low confidence routes to "ask a human" rather than "pass".
- **Shadow mode first:** log answers alongside the mechanical result for a
  few weeks, then compare, then enforce. Pin the model version.

## Risks and open questions

- Early access only; no self-serve key yet. The local clones cover the
  spike.
- Prompt injection through PR bodies and doc text into gate decisions.
  Keep policy text out of `state`; treat answers as advisory.
- Order sensitivity and quantization loss in the open clones; the
  community benchmark used only ~100 aligned rows against Jev.
- Vendor benchmarks are self-reported; critics suspect Jev builds on
  open-weights models. Irrelevant if the seam is backend-agnostic.
- Cost of a fan-out over a large spec times a large diff is unmeasured.

## Suggested first step

Spike `so1` or `openjev` locally against the three retros written
2026-08-20/21 and their PRs: ask the spec-drift questions, compare against
what the retros actually recorded. If it catches the drift the retros
found by hand, write the seam spec.

## Sources

- https://typesafe.ai/blog/introducing-system-one-models-and-jev
- https://docs.typesafe.ai/ (api, primitives/choice, quickstart, agent-skill)
- https://en.wikipedia.org/wiki/Jev_(AI_model)
- https://techcrunch.com/2026/09/18/a-new-kind-of-ai-model-from-a-chatgpt-inventor-is-thrilling-developers/
- https://www.langchain.com/blog/building-a-harness-with-jev
- https://wavect.io/blog/jev-ai-decision-model-review/
- https://dev.classmethod.jp/en/articles/openjev-non-generative-ai-alternatives/
- https://github.com/typesafe-ai/skills
- https://github.com/TheoLeeCJ/openjev
- https://github.com/ikermoel/open-alternative-jev
