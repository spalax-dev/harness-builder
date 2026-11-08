# Harness Builder

A reusable agent skill for adding **task-scoped verification** to a repository:
light checks for docs/chores, targeted tests for fixes, and the full gate only
when a feature or refactor needs it. It ships the harness plus the agent
contract (`AGENTS.md`, `.opencode` roles, `progress/`, `docs/harness/`).

## What it does

- Derives the level from the task type:
  - `feat` / `refactor` / breaking → **full** (all gates + reviewer at close)
  - `fix` / `perf` / `test` → **medium** (affected type + targeted specs)
  - `chore` / `docs` / `style` / `ci` / `build` → **light** (changed
    format/lint + structure/docs)
- Adds `--for-task`, `--explain`, `--fast`, `--changed` and `--dev-build`.
- Runs the spec next to a changed file; falls back to the package suite when
  there is none.
- Keeps full functionally identical when gates are parallelized.
- Ships the contract: `AGENTS.md`, `.opencode/agent/*`, `progress/`,
  `docs/harness/*`, and a tracked-vs-ignored git policy.

## Quick start

```bash
cp -R harness-builder ~/.agents/skills/harness-builder
```

Then ask the agent:

> Use the `harness-builder` skill to add a verification harness to this repo.
> Detect the stack, scaffold the agent contract, and fill the gate commands.

Minimum checks:

```bash
node --test scripts/harness-lib.spec.mjs
node scripts/harness.mjs --explain 1.a
node scripts/harness.mjs --fast
node scripts/harness.mjs --for-task 1.a
```

**Good fit:** a project with at least three meaningful gates and a full run
that costs more than a few seconds. **Not a fit:** a three-file script with one
test command; if the full run is already under ~10 s and stable, skip this.

## Stacks

Angular, Node/TypeScript (React, Vue, Svelte, Next, Nuxt), Python, plus Rust/Go
via the stack table in `SKILL.md`. Project-specific commands live in
`references/stacks/`; the core is stack-agnostic.

## Git

Track source, tests, docs, `AGENTS.md`, `.opencode/agent/*`, harness scripts and
one lockfile. Ignore dependencies, build output, caches, secrets and live
session state (`activities.json`, `progress/current.md`). The durable history
(`progress/history.md`) is tracked. Full policy in `assets/GIT_TRACKING.md`.

## Learn more

- `SKILL.md` — full workflow, gate selection and anti-patterns.
- `references/lessons.md` — the real lessons behind the design.
- `references/stacks/` — command tables per stack.
