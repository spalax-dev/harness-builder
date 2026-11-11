# Harness Builder

A reusable agent skill for adding a **two-stage verification harness** to a
repository:

- **PLANNING** — foundation docs, C4/PlantUML, stack and ADRs, contracts, and a
  coarse `BACKLOG.md` of user stories. No code, no tests. One-way.
- **DEVELOPMENT** — one story at a time, derived into activities and tasks,
  scheduled into parallel batches by dependency and write scope,
  documentation-first per activity, story-scoped TDD, and **user approval**
  instead of an agent reviewer.

It ships the engine, the agent contract (`AGENTS.md`, roles, plugin) and the
foundation templates.

## What it does

- Records the stage in `.harness/state.json` and refuses to mix stages.
- Parses `BACKLOG.md`; derives activities only for the selected story.
- Schedules batches from `dependsOn` + `touches`; independent activities run in
  parallel, conflicting ones serialize.
- Blocks source edits in planning and out-of-scope edits during a batch
  (`.opencode/plugins/harness-guard.js`).
- Requires a documentation generator and runs it at activity/story close.
- Keeps TDD scoped to the story being built.
- Makes the user approve the plan and the finished story; `--story` exits `3`
  until then.
- Ships the contract: `AGENTS.md`, `.opencode/agent/*`, plugin,
  `progress/*`, `docs/harness/*` and the git policy.

## Commands

```bash
node scripts/harness.mjs --stage
node scripts/harness.mjs --planning
node scripts/harness.mjs --start-dev --user-approved --by <name>
node scripts/harness.mjs --schedule
node scripts/harness.mjs --claim ACT-01
node scripts/harness.mjs --activity ACT-01
node scripts/harness.mjs --story US-001
node scripts/harness.mjs --approve-story US-001 --by <name>
node scripts/harness.mjs --fast
```

## Quick start

Copy the skill to `~/.agents/skills/harness-builder`, then ask the agent:

> Use the `harness-builder` skill to add a two-stage harness to this repo.
> Detect the stack and the documentation generator, scaffold the foundation and
> the agent contract, fill the gate commands, and prove the protocol with a
> fixture.

Minimum checks after scaffolding:

```bash
node --test scripts/harness-lib.spec.mjs
node scripts/harness.mjs --explain
node scripts/harness.mjs --stage
```

**Good fit:** a project with at least three meaningful gates and work that
benefits from parallel activities. **Not a fit:** a three-file script with one
test command.

## Stages at a glance

| Stage | Produces | Blocked |
| --- | --- | --- |
| `planning` | `docs/planning/**`, `BACKLOG.md`, ADRs, C4, sequences | code, tests, build, dev commands |
| `development` | code, tests, docstrings, generated docs, new ADRs | returning to planning |

## Git

Track source, tests, `BACKLOG.md`, `.harness/state.json`, `docs/planning/**`,
`docs/harness/**`, `AGENTS.md`, roles, plugin, harness scripts and one lockfile.
Ignore dependencies, build output, generated docs, caches, secrets,
`activities.json`, the schedule, claims, approvals and `progress/current.md`.
Full policy in `assets/GIT_TRACKING.md`.

## Learn more

- `SKILL.md` — methodology, gates, scheduling and anti-patterns.
- `references/migrating-existing-harness.md` — upgrade an installed v1 harness
  without losing history.
- `references/lessons.md` — the design lessons behind it.
- `references/stacks/` — command tables per stack.
