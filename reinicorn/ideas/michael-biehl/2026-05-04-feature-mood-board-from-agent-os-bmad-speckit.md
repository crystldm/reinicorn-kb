# Feature mood board — Agent OS, BMAD, SpecKit

**Date:** 2026-05-04
**Author:** Michael Biehl
**Status:** open

## Description

Captured during a comparison of reins vs. Agent OS, BMAD, and SpecKit. Reins's
positioning is deliberately distinct (team harness, not synthetic-team
simulator — see `project_reins_positioning` memory), but several features in
the other three are worth borrowing. This is a research/inspiration doc, not a
commitment to implement any of these.

## The three frameworks at a glance

| | **Agent OS** | **BMAD** | **SpecKit** |
|---|---|---|---|
| **Mental model** | Standards-first single-agent | Persona-driven agile | Spec-as-executable governance |
| **Anchor doc** | `mission.md` + `standards/` | PRD + Architecture | `constitution.md` |
| **Unit of work** | Spec shaped by extracted standards | User story passed between personas | Spec → Plan → Tasks artifacts |
| **Multi-agent** | No | Yes (Analyst/PM/Architect/SM/Dev/QA…) | No (agent-agnostic) |
| **Sources** | github.com/buildermethods/agent-os | github.com/bmad-code-org/BMAD-METHOD | github.com/github/spec-kit |

## Features worth borrowing

### From SpecKit

- **Broad agent/IDE coverage out of the box.** SpecKit ships templates for 30+
  targets (Claude Code, Copilot, Gemini CLI, Cursor, Codex, Qwen, Tabnine,
  Mistral, Goose…). Reins currently optimizes for Claude Code with conventions
  that "carry natively to Cursor and Copilot" — could be much more concrete.
  Borrow: a per-agent template injection system so `reins init` produces the
  right command files for whatever agent the user runs.
- **Cross-artifact `/analyze` consistency check.** A formal step that audits
  spec/plan/tasks for drift before implementation. Reins has structural lints
  for kb shape, but no semantic check that exec-plan goals match
  progress.md tasks match the actual diff. Worth a `reins plan analyze`
  command.
- **Optional `/clarify` step.** Forces underspecified areas to be resolved
  before planning proceeds. Reins exec-plans don't have a rigor gate — a
  clarification pass could be added to `reins plan create`.
- **Constitution as first artifact.** SpecKit creates `constitution.md` before
  anything else. Reins has `golden-principles.md` but it's not foregrounded
  during init the same way. Could promote it earlier in the setup flow.
- **`specify check` command.** A pre-flight that verifies the install is
  healthy. Reins has `reins status` for kb health — could expand to verify
  hook installation, agent template freshness, extension manifest validity.
- **Extensions and presets as first-class CLI surface.** SpecKit ships
  `specify extension add/search` and `specify preset add/search`. Reins has
  `apply-variant.sh` but no discovery mechanism — adding `reins extension
  list/search/add` would make the extension surface usable instead of
  hypothetical.

### From Agent OS

- **Context-aware standards injection.** Instead of dumping the whole
  principles doc into context, only the standards relevant to the current
  spec/branch are surfaced. Reins's `golden-principles.md` is currently
  all-or-nothing. A way to tag principles by domain and inject only the
  relevant subset into agent context would scale better as the principles
  list grows.
- **Standards extraction from existing code.** Agent OS's "Discover
  Standards" stage scans the codebase for patterns and turns them into
  documented standards. Reins's `/analyze-codebase` skill does some of this
  but doesn't produce lintable rules. Worth pairing extraction with
  promotion-to-linter.
- **Profile system.** `profiles/default/global/` holds per-agent config
  layers. Reins has `extensions/stacks/` for language overlays but no
  notion of agent profiles. Could be useful as agent coverage broadens.

### From BMAD

Most of BMAD's distinctives (personas, party mode, persona-handoff workflow)
are explicit non-goals for reins per the positioning memory. But a few
mechanics are still worth borrowing:

- **`npx bmad-method install`.** Zero-friction install with `--directory
  --modules --tools --yes` flags. Reins has `pip install reins && reins
  init`, which is fine for Python users but creates a dependency. A
  platform-agnostic installer (npx, curl-pipe-bash, or Homebrew) would lower
  the bar.
- **Greenfield vs. brownfield distinction.** BMAD scale-adapts its workflow
  to project complexity and explicitly handles both new and existing
  projects. Reins has `init-greenfield.sh` and `attach-to-repo.sh` but the
  distinction isn't surfaced in the CLI. Worth promoting to top-level
  `reins init` modes.
- **`@bmad-help` skill.** A first-class help skill that explains the
  framework conversationally. Reins has README and CLI `--help` but no
  in-agent guide. A `/reins-help` skill that answers "what does this kb
  directory do" or "how do I create a plan" without leaving the agent
  session would be useful.
- **Trigger codes (CP, CA, DS, QD…).** Short two-letter aliases for common
  workflows. Lower priority but a nice ergonomic touch for power users.

## Cross-cutting themes

Three patterns show up in all three projects that reins could borrow:

1. **Top-level CLI verb for every artifact type.** SpecKit: `specify`,
   `plan`, `tasks`. BMAD: `*pm`, `*architect`, `*sm`. Agent OS: per-stage
   commands. Reins's CLI is already shaped this way (`plan`, `idea`,
   `lint`) but could fill in gaps (`reins principle add`, `reins decision
   add`, `reins debt add`).
2. **First artifact is governance.** Constitution / standards / mission all
   come before any feature work. Reins should consider whether `reins init`
   should *require* a pass through `golden-principles.md` and `DESIGN.md`
   rather than leaving them as templates.
3. **Templates ship inside the tool, not the repo.** SpecKit's `.specify/
   templates/` lives in the install, not the user's repo, so updates flow
   through tool upgrades. Reins's `kb/_template/` lives in the user's kb,
   which means template improvements don't propagate. Worth revisiting.

## Explicit non-goals (per positioning memory)

Do **not** borrow from BMAD:

- Agent personas (PM-agent, Architect-agent, QA-agent, etc.)
- Multi-agent orchestration / "party mode" / role-played debates
- Anything that replaces a human role rather than augmenting it

## Notes

_No additional notes yet._
