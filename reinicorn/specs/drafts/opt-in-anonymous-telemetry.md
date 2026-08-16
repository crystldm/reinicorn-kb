---
type: spec
title: Opt-in anonymous telemetry
slug: opt-in-anonymous-telemetry
lifecycle: active
status: draft
created: 2026-08-16
author: Michael Biehl
origin: ai-assisted
human_validated: false
---

# Opt-in anonymous telemetry

## Problem

Reinicorn's guardrails (skills, hooks, doc workflows, golden principles) are
tuned by intuition and community lore. We have no empirical signal about which
commands and workflows are actually used, where they break, or which guardrails
agents override — let alone how any of that varies by agent harness or model.

Prior art (researched 2026-08-16) shows the pieces exist but nobody has closed
the loop in the open:

- **Copilot Arena** (arXiv 2502.09328) proved in-the-wild usage signal has
  near-zero correlation with static benchmarks — real usage is the only
  ground truth for tool behavior.
- **VS Code/Copilot harness team** (code.visualstudio.com blog, 2026-05/07)
  documented telemetry → A/B → shipped per-model prompt changes. Closed
  source, single vendor.
- **aider, Cline, Goose, Roo Code** all run opt-in anonymous PostHog telemetry
  in agent CLIs, but none has published learnings from it.

General product tuning needs almost no scale ("nobody runs `debt create`",
"the session-start hook errors on 15% of installs" are actionable at N≈30
users). Per-model guardrail tuning needs volume we don't have yet — but if
events carry `harness` and `model` dimensions from day one, cross-model
analysis becomes a later query, not a later system.

## Design Goals

1. **Strict opt-in.** Nothing is ever sent without an explicit yes. Users of a
   guardrail tool are privacy-sensitive by selection; copy the aider consent
   pattern, not the Homebrew/Next.js opt-out pattern.
2. **Content never leaves the machine.** Hard "never collected" list: doc
   content, prompts, code, file paths, repo names, branch names, keys, PII.
3. **Inspectable.** The full event schema is published in docs with literal
   example JSON (Next.js pattern), and a debug mode prints events instead of
   sending them (GitHub CLI `GH_TELEMETRY=log` pattern).
4. **Cross-model-ready.** Every event carries harness + model dimensions so
   the per-model tuning story is a free upgrade once volume exists.
5. **Actionable at small N.** v1 questions must be answerable with tens of
   opted-in installs: command funnels, error rates, guardrail overrides.
6. **Findings published.** Aggregated learnings and the changes they drive are
   published openly — the part with zero OSS precedent.

## Design

### Consent

- Default: fully off. No event, no network call, no ID file.
- `rcorn telemetry enable|disable|status` manages state; `enable` prints the
  schema summary and the docs URL before confirming.
- One-time interactive prompt is allowed only at `rcorn init` (never
  mid-command), and declining is permanent — never re-prompt.
- `DO_NOT_TRACK=1` is an unconditional override, even if previously enabled.
- CI environments: off regardless of config (Nx pattern) — detect via `CI`.
- Identity: random UUID4 persisted at `~/.rcorn/telemetry-id`. Deleting the
  file permanently resets identity; no server-side way to reconnect.

### Event schema (schema-first, versioned)

Events are declared in one registry module (mirroring `doc_types.REGISTRY`
style) and validated client-side before send; anything not on the field
allowlist is dropped at the client. Schema carries a `schema_version`.

Common dimensions on every event:

- `schema_version`, `rcorn_version`, `os` (linux/darwin/windows only)
- `harness` (claude-code / codex / copilot / cursor / other / unknown —
  detected from env, never free-text)
- `model` (model id as reported by the harness env if available, else unknown)
- `anonymous_id` (the UUID4)

v1 event types (deliberately coarse):

1. `session_start` — emitted by the session-start hook: kb present, plan
   present, mode flags. One event per session; the hook is the single
   instrumentation point, not per-command patching.
2. `command` — command name + subcommand + exit class (ok / user-error /
   internal-error). No arguments, no values.
3. `doc_lifecycle` — doc type + verb (create/show/complete/archive). Enables
   funnels like plan create → complete without touching content.
4. `guardrail_event` — guardrail kind (skill / hook / principle / lint) +
   guardrail id + outcome (fired / followed / overridden / errored) +
   workflow phase (session-start / pre-commit / mid-task / finish). Phase
   matters: field data (arXiv 2601.10253) shows workflow-boundary
   interventions get ~52% engagement vs 62% dismissal mid-task.
5. `error` — component + error class (no messages, no paths).

### Transport

- PostHog (the de facto OSS-agent-CLI choice), no person profiles, events
  only; fire-and-forget with a short timeout — telemetry failure must never
  slow or break a command.
- `RCORN_TELEMETRY_DEBUG=1` prints the exact JSON to stderr instead of
  sending.
- Endpoint and project key overridable so privacy-conscious users can point
  at their own PostHog instance (aider pattern).

### Analysis discipline

- Cap per-installation weight in any aggregate (one anonymous client must not
  move a conclusion — the Chatbot Arena vote-rigging lesson, arXiv
  2501.17858).
- Before generalizing, diff the opted-in sample's harness/model mix against
  known ecosystem distributions (the arXiv 2604.05100 audit pattern) —
  opted-in users are not representative.
- Treat per-model findings as hypotheses to confirm with a fixed benchmark
  harness (aider leaderboard pattern), not conclusions.
- Publish a periodic "what the telemetry taught us" doc plus the retention
  window (12 months, then purge).

### Docs

New docs page: what is collected (with literal example events), the never-
collected list, how to enable/disable/inspect, retention, and the privacy
contact. Linked from README and from the `telemetry enable` output.

## Non-Goals

- **Per-model A/B power in v1.** The install base can't support it; v1 buys
  directional signal only. No live prompt experiments.
- **No content collection ever** — not even opt-in flags for it (unlike
  Claude Code's layered `OTEL_LOG_*` redaction opts). Reinicorn events are
  metadata-only by construction.
- **No arena / pairwise preference UI.** Different mechanism, different spec.
- **No OpenTelemetry conformance.** The OTel `gen_ai.*` conventions are still
  "Development" maturity and aimed at user-side observability; revisit if
  they stabilize.
- **Qualitative feedback flow** — covered by the retro→`rcorn feedback` CTA
  idea (see ideas/every-finished-branch-runs-a-retro…), not this spec.
