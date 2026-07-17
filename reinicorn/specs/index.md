# Specs

## What Is a Spec?

A spec is the **implementation contract** for a piece of work — the artifact produced during brainstorming and the reference an implementation is checked against. It captures the problem, the goals, the chosen design, and what is explicitly out of scope, so future contributors (human or agent) understand *what* we are building and *why we chose this approach*.

This matches the meaning of "spec" in our spec-driven-development skills: brainstorming produces a spec, `writing-plans` turns it into tasks, and the spec-compliance review checks the implementation against it.

Write a spec when:

- Introducing a **new domain or subsystem**
- Making an **architectural decision** that affects multiple components
- Performing a **significant refactor** that changes how existing code is organized
- Choosing between multiple valid approaches where the rationale matters

Specs are not meeting notes, not product requirements (see `../prds/`), and not tutorials. They answer: "What are we building, why, what did we decide, and what were the alternatives?"

Create one with `reins spec create "title"` — never hand-write files in this directory.
