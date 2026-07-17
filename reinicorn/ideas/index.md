# Ideas

A space for capturing ideas, feature requests, and improvements discovered while working. Ideas are informal and low-friction — the goal is to capture the thought before it is lost, not to write a product spec.

Ideas captured here are reviewed during end-of-sprint digests, where they are either promoted to product specs, filed as tickets, or archived.

---

## Organization

Ideas are organized by author to avoid conflicts when multiple developers are working simultaneously:

```
kb/reins/ideas/
  index.md          <-- this file
  shared/           <-- team-wide ideas (not attributed to one person)
  {username}/       <-- per-developer ideas
```

- **Per-developer ideas** go in a subdirectory named after the developer (e.g., `michael/`, `sarah/`). This prevents merge conflicts when multiple people capture ideas at the same time.
- **Team-wide ideas** — ideas that emerge from group discussion or that are not attributed to a specific person — go in `shared/`.


---

## Capturing Ideas

### Using the `/capture-idea` skill

The fastest way to file an idea:

```
/capture-idea "Brief title of the idea" --details "Optional longer description"
```

The skill will:

1. Create a timestamped entry in your `{username}/` subdirectory (or `shared/` if specified).
2. Format the idea with a title, date, author, and description.
3. Update this index if needed.
4. Push the change to the kb.

### Manual capture

If you prefer to write ideas manually:

1. Create or open your subdirectory file (e.g., `kb/reins/ideas/{username}/2026-02.md` — one file per month keeps things manageable).
2. Add an entry with this format:

```markdown
### [Brief title]
**Date:** [YYYY-MM-DD]
**Author:** [your name]

[Description of the idea. What problem does it solve? Who benefits? Any rough thoughts on implementation.]
```

3. Keep entries brief. A few sentences is fine. The goal is capture, not specification.

---

## End-of-Sprint Digest


A scheduled job (or manual run of `/update-kb`) collects each developer's ideas and produces a summary for team review. The digest:

1. **Aggregates** all ideas added since the last digest across all developer subdirectories and `shared/`.
2. **Groups** ideas by theme (feature requests, DX improvements, tech debt observations, etc.).
3. **Produces a summary** that the team reviews during sprint planning or retrospective.

During review, each idea is triaged:

- **Promote** — The idea becomes a product spec in `kb/reins/product-specs/` or a ticket in the issue tracker.
- **Park** — The idea is interesting but not actionable now. It stays in the ideas directory for future review.
- **Archive** — The idea is no longer relevant. Move it to an `_archived` section in the source file.

---

## Recent Ideas


No ideas captured yet. Use `/capture-idea` to get started.
