# Tech Debt Dashboard

Tech debt is catalogued as it is encountered — during feature work, code review, or refactoring. It is tracked here so it can be prioritized, scheduled, and paid down incrementally rather than ignored until it becomes a crisis.

For the project's philosophy on tech debt, see [Core Beliefs: Principle 6](../specs/core-beliefs.md) — "Tech debt is a high-interest loan."

---

## Severity Classification

| Severity | Definition | Response |
|----------|------------|----------|
| **critical** | Blocks active work. Causes failures, data loss, or makes a domain unworkable. | Fix immediately. Add to `cleanup-queue.md` at top priority. |
| **high** | Causes frequent issues. Slows development, produces recurring bugs, or confuses agents regularly. | Schedule for next sprint or iteration. Add to `cleanup-queue.md`. |
| **medium** | Inconvenient. Creates friction, requires workarounds, or makes code harder to understand, but does not block work. | Address opportunistically or during scheduled cleanup. |
| **low** | Cosmetic. Naming inconsistencies, minor style issues, or small improvements that would be nice but are not urgent. | Fix when touching the affected code for other reasons. |

---

## Summary


| Severity | Count |
|----------|-------|
| Critical | 1 |
| High | 3 |
| Medium | 12 |
| Low | 12 |
| **Total** | **28** |

---

## Breakdowns

Debt items are organized two ways for easy navigation:

- **Individual debt docs** (`./*.md`) — one doc per debt item, created via `rcorn debt create "<title>"`. The affected domain is recorded in each doc's `**Domain:**` field.
- **[By Category](./by-category/)** — Debt grouped by the type of debt (e.g., missing tests, stale docs, architectural violations, deprecated dependencies). Use this view when planning targeted cleanup efforts.

---

## Cataloging New Debt

When you encounter or introduce tech debt:

1. **Identify the domain and category.** Which business domain does this debt live in? What kind of debt is it (missing tests, architectural violation, deprecated dependency, etc.)?

2. **Create a debt doc.** Run `rcorn debt create "<title>"` and fill in the template (optionally also add to the matching by-category file). Each entry should include:
   - **Description** — What the debt is, concretely.
   - **Severity** — One of: critical, high, medium, low.
   - **Impact** — How this debt affects development (blocks work, slows agents, causes bugs, etc.).
   - **Remediation** — What needs to be done to fix it, with enough detail that an agent could attempt it.
   - **Date Added** — When the debt was catalogued.

3. **Update this dashboard.** Increment the count in the Summary table above.

4. **Add to the cleanup queue if actionable.** If the debt is critical or high severity, add it to [`cleanup-queue.md`](./cleanup-queue.md) so it gets scheduled for remediation.

---

## Periodic Cleanup

Periodic kb reviews go through the tech debt catalog:

- **Completed items** are removed from `cleanup-queue.md` and marked as resolved in the debt docs and by-category files.
- **Stale items** (debt that has been resolved by other work but not updated here) are flagged for cleanup.
- **Summary counts** are recalculated to match actual entries.

During a scheduled cleanup pass, pick from [`cleanup-queue.md`](./cleanup-queue.md), starting with the highest-priority items. See that file for details on how the queue is managed.
