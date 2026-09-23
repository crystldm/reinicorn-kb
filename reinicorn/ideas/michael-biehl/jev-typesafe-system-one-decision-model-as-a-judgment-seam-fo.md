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

## Spike results (2026-09-22, openjev on a GTX 1070)

Ran the suggested first step with `TheoLeeCJ/openjev` (Qwen3.5-4B Q4_K_M,
llama.cpp backend built with CUDA for sm_61, all layers on the GPU). Per
PR: state = the compressed diff (context lines dropped, 30-40 changed lines
per file, ~25k chars), one `Choice` per spec Design subsection over
{implemented, deviates, omitted, unrelated}, shared-state mode so the diff
is prefilled once.

| PR | Spec | Sections | Match vs hand labels | Wall |
|---|---|---|---|---|
| #27 | enforce-the-review-lane | 6 | 4/6 | 20 s |
| #30 | unified-kb-doc-frontmatter | 7 | 3/5 labeled | 25 s |
| #70 | process-as-config (stage 2) | 10 | 3/8 labeled | 32 s |

Findings, most important first:

1. **Evidence truncation caused the worst miss, not the model.** The one
   real rename drift in #70 (`kb/required-sections` shipped as
   `kb/plan-structure`) appears 10 times in the raw diff and zero times in
   the compressed state, so "Shipped defaults" scored `unrelated` 0.93.
   Any real design needs the diff-to-state step to be claim-aware (grep the
   diff for every identifier the spec section names) rather than a blind
   per-file cap.
2. **Scope is not the model's question.** The known cut in #27 (spec §6,
   never shipped) scored `unrelated` 0.86 instead of `omitted`, which is
   correct by the option text: from the diff alone "not delivered" and
   "not this PR's job" are indistinguishable. Whether a section was in
   scope comes from the plan's Tasks in code; the model should only be
   asked `Noul` "does this diff deliver this section".
3. **Coarse 4-way Choice over a whole section is too blunt.** Stage
   boundaries, philosophy sections ("Behaviors, not types") and worked
   examples all confused the label. Better: extract concrete claims per
   section (a lint name, a field, a command, a default) and ask one `Noul`
   per claim; then the drift report is "claims not found in diff".
4. **Where it was right, it was usefully right.** #30's extra frontmatter
   fields (`review_pr`, `approved_by`, `review_cancelled`, absent from the
   spec) came back `deviates` as the top option; the spec sections that
   did land scored `implemented` at 0.67-0.88; and it correctly said #70
   never touched AGENTS.md (`unrelated` 0.96).
5. **Cost is a non-issue on a GPU.** 6-10 questions over a 5-8k token diff
   cost 11-30 s of compute on a 2017 GPU, ~0.5 s per question after the
   shared prefill. On CPU the same prefill was ~20 tok/s (minutes per PR)
   and, on this machine, tripped the CPU thermal limit twice.

Practical notes for a rerun: openjev's shared mode rejects a dict-shaped
state (the closing `"}` merges with the following `,` in the tokenizer;
a string state works); prebuilt CUDA wheels of llama-cpp-python (0.3.19)
predate the Qwen3.5 architecture, so 0.3.35 must be built from source
(conda-forge `cuda-nvcc=12.8` via micromamba, no root, ~18 min at -j2).

Verdict: the primitive shape works and the speed is there; the accuracy
gap is in question design and evidence preparation, which are code, not
model, problems. Next experiment: claim extraction + per-claim `Noul`,
with the diff filtered by the claim's identifiers.

## Spike 2 (2026-09-22): claim extraction + per-claim yes/no

Same model, same three PRs, but the unit of judgment is a **claim**, not a
section: every sentence, bullet or table row in a Design section that
carries a backticked identifier becomes one yes/no question ("does the
diff deliver this claim as written?"), and its evidence is the slice of the
raw diff that mentions the claim's identifiers, not the whole compressed
diff. 129 claims across the three PRs; direct mode on the GPU, 36-85 s per
PR, ~1 s per claim including prefill.

Two retrievers were tried. v1 grepped every backticked token; v2 turns
`name(...)` into `def name(`, scopes the search to a file when the claim
names one, and ranks evidence rarest-identifier-first so the line cap
cannot fill up with test noise.

| | coarse 4-way (spike 1) | claims v1 | claims v2 |
|---|---|---|---|
| section label agreement (18 labels) | 9/18 | 3/18 | 7/18 |
| claim-level agreement (22 hand-verified claims) | n/a | 11/22 | 16/22 |

Findings:

1. **Retrieval dominates, again.** Every v1→v2 gain was evidence, not
   judgment: #30's `read`/`write`/`validate` went from 0.11-0.58 to
   0.75-0.99 once `def write(` outranked the 84 lines that merely contain
   the word "write". The judge was fine; it had never been shown the
   function.
2. **Per-claim answers beat any aggregation of them.** On #70's "Stages"
   section the model said Relations yes (0.92) and Loader, Gates, Defaults
   no (0.19-0.30), which is exactly what stage-2 PR #70 is. Averaging those
   into a section label produced "deviates", which is wrong. The useful
   output is the list itself: *these claims have no evidence in this PR*.
   Mapping claims to section verdicts scored worse than the blunt
   4-way question, so do not build that mapping.
3. **Four claim shapes a diff cannot answer.** (a) Claims about what
   *stays* unchanged ("stays severity: warning") need the post-merge file,
   not the diff. (b) Claims about *absence* ("the only place a type key may
   appear is doc_types.py") read as "not delivered" even when the removals
   are right there. (c) Claims that name a file the code put the helper
   somewhere else (#27's resolver lives in `spec_refs.py`, the spec said
   `draft_refs.py`/`pre_push.py`) go no-hit under file scoping, which is
   arguably the correct "as written" answer but not what a reviewer wants.
   (d) Name/signature drift (`_ensure_plan_spec_approved(root)` vs
   `ensure_plan_spec_approved(root, branches)`) is waved through at 0.93;
   a 4B model does not hold "as written" that tightly.
4. **No-hit is the cheap signal.** 14 of 129 claims matched nothing in
   the diff. For in-scope sections that list is the spec-drift report
   with zero model calls (#27 §6's next-step hints, #30's migration
   script, #70's `EXCLUDED_FILENAMES`). The model only earns its keep on
   claims that *do* have evidence and need reading.
5. **Mechanical claim extraction is half noise.** Rationale sentences
   ("Containment. git ls-files only ever emits…", "Two changes to
   linter/rules/draft_refs.py.") get scored like requirements and drag
   the numbers. Claim extraction is the step that wants a frontier model
   (or the spec author, at spec-writing time: a `Claims` block), not a
   regex.

Verdict after two spikes: for spec drift the pipeline that fits is
**claims extracted once at review time → identifier retrieval against the
PR (post-state files for "unchanged" claims) → one yes/no per claim with
evidence → report the no-evidence and low-probability claims verbatim**.
Section labels are the wrong output. The typed-decision model is the
cheapest part and the least of the accuracy problem; the retrieval and
claim-extraction code around it is where the work is, and that code is the
same whether the backend is Jev or a local 4B.

## Spike 3 (2026-09-23): retro substance gate

This was the best-fit job from the follow-up brainstorm. The input is short
and self-contained, the answer is a genuine binary, and no retrieval is
needed. Setup: 83 sections from the 18 real retros, all labelled pass (9 of
them are deliberate "None open." / "None. (No spec)" lines). Against them
are 60 synthetic filler sections, written by a separate agent: 12 each of
template leftovers, platitudes, activity logs with no evaluation, vague
reflection, and hollow dismissals. The model scored each section alone
(branch + heading + body), with P(pass) read from logits. Qwen3.5-4B Q4 ran
on the GPU at about 0.4 s per row, and Tdie peaked at 61 °C.

| Scorer | AUC | Best accuracy |
|---|---|---|
| Body length in characters (no model) | **0.91** | — |
| "Substantive or filler?", rich option descriptions | 0.80 | 0.78 |
| Same, options in the other order | 0.85 | 0.81 |
| Same, terse descriptions | 0.74 | 0.75 |
| "Names a concrete thing?" | 0.67 | 0.69 |
| "Explains why, not just what?" | 0.86 | 0.78 |
| specific × why | 0.85 | 0.80 |

On the hard slice (real sections vs platitude/vague/activity-log), length
scores 0.97 AUC and the best model question scores 0.87.

1. **No prompt beat character count.** The synthetic filler is short, so
   length is flattered here. Even so, the model's failures were not about
   length: platitudes ("Tests passed and the PR merged smoothly.") passed
   at P 0.90–1.00 under every holistic question.
2. **Strong yes-bias on holistic judgments.** "- Nothing." and
   "- (none yet)" passed at P=1.00 under the rich prompt. Swapping the
   option order moved AUC by 0.05, so position bias is real too.
3. **Narrow questions answer narrowly.** "Names a concrete thing?" is
   exactly what activity logs satisfy (1.00). "Explains why?" is the one
   question that separates activity logs (0.03), but it also fails 18 of
   74 real sections. Real "What Went Well" bullets are often evidence
   lists, which is legitimate but reads as "no why".
4. **The explicit-none vs hollow-none split depends on the heading.**
   "None." is fine under Action Items and filler under Lessons Learned.
   That is a table lookup on the heading plus a length check, not a
   judgment.

Verdict: a 4B logit readout is not a substance gate. Leftovers and hollow
dismissals are a regex plus a per-heading minimum length. Fluent generic
text needs a stronger reader or a structural rule (for example, every
Lessons bullet must reference a named artifact). Untested caveat: a
length-matched filler set (long, fluent, generic) would remove the length
advantage. The platitude results suggest the model would do no better
there.

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
