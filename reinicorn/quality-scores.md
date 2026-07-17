# Quality Scores

Quality scores provide an at-a-glance view of each domain's health. They are maintained by developers and agents during periodic kb reviews.

---

## Grading Scale

| Grade | Definition |
|-------|------------|
| **A** | Excellent. High test coverage, up-to-date docs, minimal tech debt. The domain is well-maintained and easy for agents to work in. |
| **B** | Good. Adequate test coverage with minor gaps, docs are mostly current, manageable tech debt. No urgent issues. |
| **C** | Acceptable. Some test coverage gaps, docs may be stale in places, tech debt is accumulating but not blocking. Needs attention soon. |
| **D** | Poor. Significant test coverage gaps, docs are outdated or missing, tech debt is causing friction. Prioritize improvement. |
| **F** | Critical. Minimal or no tests, docs are absent or misleading, tech debt is actively blocking work or causing bugs. Requires immediate remediation. |

---

## Domain Scores

| Domain | Test Coverage | Doc Coverage | Tech Debt Level | Overall Grade |
|--------|--------------|--------------|-----------------|---------------|
| CLI & Commands | B | B | low | **B** |
| Init Workflow | A | A | minimal | **A** |
| Kb Management | B | B | low | **B** |
| Publish/Sync | B | B | minimal | **B** |
| Skills | B | A | minimal | **A** |
| Linting Framework | B | A | minimal | **A** |
| Git Hooks | B | B | low | **B** |
| Update Path | B | A | minimal | **B** |

**Last updated:** 2026-07-17

### Notes

- **Init Workflow** (A): 40+ tests covering all init paths, gh integration, multi-repo, slug override. Full design docs.
- **Skills** (A): Unified skill set with upstream tracking, attribution, copy tests. Well-documented.
- **Linting Framework** (A): 5 lint rules, framework docs, lint-rule template, CI integration.
- **CLI & Commands** (B): Good coverage but some commands (status, feedback) have lighter tests.
- **Publish/Sync** (B): Conflict handling, retry, diverged history tests. Background worker tested.

### Column Definitions

- **Test Coverage**: Reflects the breadth and quality of automated tests. Consider unit, integration, and end-to-end coverage.
- **Doc Coverage**: How well the domain's architecture, interfaces, and decisions are documented in `kb/`.
- **Tech Debt Level**: The severity and volume of known debt items in `kb/reinicorn/tech-debt/`. Values: `minimal`, `low`, `medium`, `high`, `critical`.
- **Overall Grade**: A holistic assessment considering all three dimensions plus subjective factors like code clarity and agent navigability.

---

## When to Update Scores

Update quality scores when:

- **Test coverage changes significantly** — A new test suite is added, coverage drops due to new untested code, or a testing gap is filled.
- **Documentation is added or becomes stale** — Architecture docs are updated, new domain docs are created, or existing docs fall out of date.
- **Tech debt is added or paid down** — New debt items appear in `kb/reinicorn/tech-debt/`, or existing items are resolved.
- **A domain undergoes major refactoring** — Large-scale changes warrant a fresh assessment.
- **During scheduled reviews** — Periodic (e.g., weekly or sprint-end) quality reviews.

A periodic kb review maintains these scores by checking for staleness signals, cross-referencing tech debt counts, and flagging scores that may no longer reflect reality.

---

## Tech Debt Details

For detailed tech debt information — individual items, severity classifications, and remediation plans — see [`tech-debt/`](./tech-debt/index.md).

The cleanup queue at [`tech-debt/cleanup-queue.md`](./tech-debt/cleanup-queue.md) lists prioritized debt items ready for remediation.
