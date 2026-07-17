# Cleanup Queue

This is the prioritized list of tech debt items ready for remediation. Items here have been triaged, have clear remediation steps, and can be picked up by anyone on the team.

Periodic kb review and scheduled cleanup passes pull from this queue when there is capacity. Anyone can add items here; prioritization is a collaborative process.

---

## Queue

| Priority | ID | Description | Estimated Effort | Status | Added |
|----------|------|-------------|-----------------|--------|-------|
| P1 | SEC-06 | Validate kb URL before `git submodule add` (re-verify — largely closed) | XS | queued | 2026-02-17 |
| P1 | SEC-08 | External lint scripts executed without validation | M | queued | 2026-02-17 |
| P1 | ERR-01 | `SystemExit` raised deep in stack — use domain exceptions | M | queued | 2026-02-17 |
| P1 | OUT-01 | Many raw `print()` calls bypass console module | L | queued | 2026-02-17 |

### Status Values

| Status | Meaning |
|--------|---------|
| **queued** | Ready for pickup. No one is working on it yet. |
| **in-progress** | Someone (human or agent) is actively working on this item. |
| **completed** | The debt has been resolved. Will be removed during the next cleanup pass. |
| **wont-fix** | Triaged and deliberately deferred or accepted. Include a brief reason. |

Note: "someone" below means a developer on the team (optionally using an AI coding assistant), not an autonomous background process — Reinicorn is a harness for human teams.

---

## Adding Items to the Queue

When adding a new item:

1. **Check the tech debt catalog first.** The item should already be documented as a debt doc (`tech-debt/*.md`) or in `by-category/`. If not, add it there first (see [index.md](./index.md) for instructions).

2. **Assess priority.** Priority is determined by severity and impact:
   - **P0** — Critical severity. Fix immediately; it is blocking work.
   - **P1** — High severity. Schedule for the current or next sprint.
   - **P2** — Medium severity with clear remediation. Good candidate for opportunistic cleanup.
   - **P3** — Low severity or effort-heavy. Pick up when convenient or during dedicated cleanup sprints.

3. **Estimate effort.** Use t-shirt sizes: `XS` (< 1 hour), `S` (1-4 hours), `M` (half day to full day), `L` (2-3 days), `XL` (week+). Prefer `XS` and `S` items for opportunistic cleanup unless directed otherwise.

4. **Add a row to the table.** Set status to `queued` and include the date added.

5. **Keep it sorted.** Items should be roughly ordered by priority (P0 at top, P3 at bottom). Within the same priority, prefer smaller estimated effort first.

---

## How Items Are Picked Up

During a periodic kb review or scheduled cleanup pass, work through the queue as follows:

1. **Scan for `queued` items**, starting from the top of the table.
2. **Prefer small items.** Pick `XS` or `S` items first — these can be completed in a single pass with high confidence.
3. **Claim the item.** Change status to `in-progress` before starting work. This prevents duplicated effort.
4. **Do the work.** Follow the remediation steps documented in the tech debt catalog entry.
5. **Verify.** Run tests and linters to confirm the fix does not introduce regressions.
6. **Mark complete.** Change status to `completed` and update the tech debt catalog entry.
7. **Update quality scores.** If the fix materially improves a domain's health, update [`quality-scores.md`](../quality-scores.md).

---

## Periodic Cleanup

During scheduled maintenance passes:

- Remove rows with status `completed` (the historical record lives in the tech debt catalog and git history).
- Review `wont-fix` items to confirm they are still deliberately deferred.
- Re-prioritize remaining items based on current project needs.
- Recalculate summary counts in [`index.md`](./index.md).
