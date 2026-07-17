# Golden Principles

These are **mechanical, enforceable rules** that keep the codebase legible for both humans and agents. They are stack-agnostic and apply universally.

Golden principles differ from design docs and core beliefs in specificity: each principle is concrete enough to be checked by a linter or a code review bot. If you cannot imagine writing a lint rule for it, it is probably a belief or a convention, not a golden principle.

---

## The Principles

### 1. Validate data at boundaries, not internally

Parse and validate at system edges: API inputs, external data sources, file reads, environment variables, user input. Once data has crossed a boundary and been validated, internal code trusts typed interfaces. Do not scatter validation logic throughout business logic layers.

**Why:** Boundary validation gives you one place to audit, one place to fix, and clean internal code that operates on known-good data. Agents can locate all validation by searching boundary modules rather than hunting through every function.

### 2. Prefer shared utility packages over hand-rolled helpers

When you need a helper function (retry logic, string formatting, date math, etc.), check `utils/` first. If one does not exist, add it there. Do not create private helpers that duplicate logic already available in a shared package.

**Why:** Centralized utilities mean one implementation, one set of tests, one place to fix bugs. When agents see the same pattern solved differently in five places, they cannot tell which version is canonical.

### 3. No YOLO-style data probing

Never write code that guesses at data shapes, reaches into untyped objects with bracket notation, or builds logic on assumptions about external API responses. Always validate shapes at boundaries (see Principle 1) or use typed SDKs and schemas.

**Why:** Guessed shapes break silently. Agents that replicate guessed-shape patterns compound the fragility across the codebase.

### 4. Agent-readable error messages

Every error message — whether from a linter, test, or runtime — must include three things:

1. **What went wrong** — A clear description of the failure.
2. **Where** — File path, line number, function name, or request context.
3. **How to fix it** — A concrete remediation step or pointer to documentation.

Bad: `Error: validation failed`
Good: `Validation error in src/billing/service/create-invoice.ts:42 — "amount" must be a positive number. See kb/reins/golden-principles.md#1 for boundary validation rules.`

**Why:** Agents parse error messages to determine next steps. Vague errors cause agents to guess, retry blindly, or ask the developer for help unnecessarily.

### 5. File size limits


Files over **[THRESHOLD]** lines should be split into smaller, focused modules. When a file approaches the limit, refactor proactively rather than waiting for it to become unwieldy.

**Why:** Large files are hard for agents to reason about. Agent context windows are finite, and a 1,000-line file forces the agent to load significant content just to find the relevant section. Smaller files are easier to name, easier to navigate, and produce more targeted diffs.

### 6. Structured logging

Use structured (JSON) logging for all log output. Do not concatenate strings into log messages. Every log entry should be a structured object with consistent fields (timestamp, level, message, context).

Bad: `console.log("User " + userId + " failed to pay invoice " + invoiceId)`
Good: `logger.error({ event: "payment_failed", userId, invoiceId, reason: err.message })`

**Why:** Structured logs are queryable, filterable, and parseable by agents and tooling. Unstructured string logs require human eyeballs to interpret and are invisible to automated analysis.

### 7. Tests for every new behavior

No exceptions. Every new feature, bug fix, or behavior change must include tests that verify the expected behavior. If you change how something works, update the tests that cover it.

Tests are not optional documentation — they are how agents verify their own work. An agent that writes code without tests cannot confirm the code works.

**Why:** Agents run tests to validate their output. Missing tests mean missing feedback loops, which lead to silently broken code.

### 8. Documentation updated with every architectural change

If you change how something works — a new domain, a modified layer boundary, a new provider, a changed dependency rule — update the relevant docs in `kb/`. Architecture docs that do not match the code are worse than no docs, because they actively mislead agents.

**Why:** Agents read kb docs to build mental models. Stale docs cause agents to produce code that conflicts with current architecture.

### 9. One concern per file

Each file should have a single, clear responsibility. If you find yourself describing a file's purpose with "and" (e.g., "this file handles validation and formatting and error mapping"), split it. Name files after their single concern so agents can navigate by filename.

**Why:** Agents discover code by reading filenames and directory structures. A file named `utils.ts` that contains validation, formatting, and error mapping is undiscoverable. A directory with `validate.ts`, `format.ts`, and `errors.ts` is self-documenting.

---

## Promoting Conventions into Principles

Not every convention belongs here. Golden principles are reserved for rules that are:

1. **Mechanical** — Can be expressed as a linter rule or structural test.
2. **Universal** — Applies across all domains and layers, not just one area.
3. **High-impact** — Violations cause real problems (bugs, agent confusion, maintainability issues).

### When to promote

When you see the **same review comment appear 3 or more times**, that is a signal the convention should be encoded as a golden principle. The process:

1. **Identify the pattern.** What comment keeps recurring? What rule would prevent it?
2. **Draft the principle.** Write it in the format above: name, description, bad/good examples, and a "Why" explanation.
3. **Get developer approval.** Add it to this file with a PR. Principles affect the entire codebase, so they need human sign-off.
4. **Encode it in a linter.** Once approved, write a lint rule or structural test that enforces the principle automatically. Add it to the project's lint configuration.
5. **Update AGENTS.md** if the principle affects how agents should navigate or produce code.

### Encoding principles into linters

The goal is to make every golden principle enforceable by automation. For each principle:

- Write a lint rule (ESLint, Clippy, Ruff, etc.) or a structural test that fails when the principle is violated.
- Include the principle number in the error message so developers and agents can look up the rationale.
- Add the lint rule to CI so violations block merges.
- If a principle cannot be fully automated, add it to the PR review checklist in `.claude/skills/review-pr.md`.


---

## Project-Specific Principles

### P1. No dynamically generated Python source strings

Never build Python code as a string (via f-string, concatenation, or template) and execute it with `python -c`, `exec()`, or `eval()`. Background or deferred work must be implemented as a dedicated module and spawned with typed command-line arguments.

**Bad:**
```python
script = f"""
import os, time
while True:
    os.kill({ppid}, 0)
    time.sleep(0.5)
"""
subprocess.Popen(["python3", "-c", script], ...)
```

**Good:**
```python
# _my_worker.py — a dedicated module with a typed main() and __main__ block
subprocess.Popen(
    [sys.executable, "-m", "mypackage._my_worker", str(ppid), str(root)],
    ...
)
```

**Why:** Injecting runtime values into source strings is an injection vulnerability (paths with quotes, newlines, or special characters break the script silently), produces untestable code (the string is opaque to linters and type checkers), and makes the logic impossible to import or unit-test. A dedicated worker module has a real signature, can be tested in isolation, and is navigable by agents.

**Enforcement:** `ruff` — flag `subprocess` calls whose first argument list contains `"-c"` paired with an f-string argument. Also flag any `exec(f"...")` or `eval(f"...")` patterns.

### P2. All paths and metadata come from the doc-type registry

Never hard-code kb directory names (`"specs"`, `"exec-plans"`, `"tech-debt"`, etc.) outside of `src/reins/doc_types.py`. All code that needs a doc-type path, filename pattern, protection flag, or required sections must import from `REGISTRY`.

**Bad:**
```python
plan_dir = kb / repo / "exec-plans" / "active"
```

**Good:**
```python
from reins.doc_types import REGISTRY
plan_dir = kb / repo / REGISTRY["plan"].dir_path / "active"
```

**Why:** Hard-coded paths scattered across 8+ files made it impossible to change directory names or add new doc types without hunting down every reference. The registry is the single source of truth — if it's not in the registry, it doesn't exist.

**Enforcement:** `grep -rn '"specs"\|"exec-plans"\|"prds"\|"tech-debt"\|"ideas"' src/reins/ --include='*.py'` should only match `doc_types.py`.

### P3. No deprecated aliases in pre-release code

Do not add backward-compatibility wrappers, deprecated aliases, or hidden fallback commands. If something is being replaced, delete the old code entirely. Deprecation shims are for published APIs with external consumers — reins is pre-release with no external consumers.

**Bad:**
```python
sub.add_parser("attach", help="(deprecated: use 'init')")
# ... deprecation wrapper that delegates to cmd_init
```

**Good:**
```python
# Delete attach.py entirely. The command no longer exists.
```

**Why:** Deprecation wrappers add dead code, confuse agents, and create two paths to the same behavior. In pre-release code, the cost of breaking changes is zero — just make the change.

12. **Axi output surface: stdout is for agents**
   - Data, errors (`error: ...`), and `next:` suggestions go to stdout; stderr
     carries only progress/debug; exit codes carry status (0 incl. no-ops,
     1 error, 2 usage). Enforced by tests/test_output_conventions.py.
   - Prevents: agents misreading progress noise as data, and follow-up
     round trips after ambiguous output.

13. **Doc-type registry is the source of truth**
   - Per-type paths, globs, hints, and behavior derive from doc_types.REGISTRY
     fields — never branch on doc-type string literals (`doc_type == "spec"`).
     Branch→directory conversion is owned by kb.branch_dir_name /
     kb.branch_doc_path. Enforced by tests/test_source_of_truth.py and the
     TID251 banned-api rule in pyproject.toml (prefer ruff-native rules;
     hand-rolled AST checks are the fallback for what ruff can't express).
   - Prevents: parallel copies of type knowledge drifting apart when a doc
     type is added or a path pattern changes.
