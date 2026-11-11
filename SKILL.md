---
name: harness-builder
description: >-
  Build, redesign or upgrade a two-stage repository harness: PLANNING
  (foundation docs, C4/PlantUML, stack, contracts, coarse user stories in
  BACKLOG.md) then DEVELOPMENT (stories derived into activities, parallel
  batches by dependency and touches, documentation-in-code, story-scoped TDD,
  user approval instead of an agent reviewer). Includes an upgrade path for
  harnesses already installed. Use when a project needs fast quality gates, a
  documentation generator, parallel agent execution, or when every change
  currently runs the full suite. Stack-agnostic: Node/TypeScript,
  Angular/React/Vue/Svelte, Python, Rust, Go and monorepos.
---

# Harness Builder

Build a verification harness that is **stage-correct, story-driven, honest about
what it did not check, and impossible to run serially when the work allows
parallelism**.

The harness is an enforcement layer for the project's own rules. It does not
invent policy.

---

## 1. When to build a harness — and when not to

Build one when **at least three gates carry independent signal** and a full run
costs more than an edit loop:

- formatter/linter, compiler/type checker, build, tests: any three catch
  different failure classes;
- the full run is slower than ~20 s, or developers skip it;
- changes are frequent enough that running everything every time costs more than
  the harness.

Do **not** build one when:

- the project is a three-file script with one test command;
- the full run is under ~10 s and stable;
- there is no agreed definition of done — write the foundation first;
- you only need a one-off check.

A harness has a cost: maintenance, stack detection, false positives, and the
discipline of the contract. Start with the smallest gate set that catches real
regressions.

---

## 2. Philosophy: two exclusive stages

The harness enforces a one-way transition:

| Stage         | Purpose                                   | Produces                                                  |
| ------------- | ----------------------------------------- | --------------------------------------------------------- |
| `planning`    | Decide the foundation                     | overview, architecture, stack + ADRs, structure, contracts, C4, sequences, BACKLOG |
| `development` | Build the stories                         | code, tests, docstrings, generated docs, ADRs caused by real decisions |

Rules:

1. **No code, no tests in planning.** Implementation is locked, mechanically.
2. **You cannot go back.** Planning artifacts stay editable in development; the
   stage does not move backwards. There is no "re-plan" phase.
3. **The stage is a project decision, tracked in git** (`.harness/state.json`),
   not session noise.
4. **Planning is never the end state.** It exists to be approved and left.

This is not TDD for the whole repository. It is:

```
PLANNING  -> foundation, coarse stories
DEVELOPMENT:
  story selected
    -> activities derived just-in-time
      -> DOCUMENT (decision + docstrings)
        -> TEST (story-scoped, must fail)
          -> IMPLEMENT
            -> verify
              -> user approves
```

### Why no global TDD

Upfront specs fail when the user does not yet know what they want. A coarse
story plus a just-in-time decomposition is more honest than 80 activities written
before the first conversation. Inside the selected story, TDD is strict: the
tests activity precedes the implementation activity.

### Documentation as a system, not specs

- Planning docs describe the foundation and stay editable.
- In-code docstrings are the source of truth for what exists.
- A documentation generator is **mandatory**: the harness refuses to leave
  planning without one, and runs it at activity/story close.
- No standalone markdown feature specs. A decision becomes an ADR; behavior
  becomes docstrings; the generator publishes them.

---

## 3. The agent contract

The engine is only half the harness. Ship the contract with it:

| File / folder | Responsibility |
| --- | --- |
| `AGENTS.md` | Entry point: stage, map, hard rules, workflow, approvals |
| `.opencode/agent/main-orchestrator.md` | Stage work, story decomposition, batch scheduling, state ownership |
| `.opencode/agent/task-executor.md` | One activity: claim, document, test, implement, report |
| `.opencode/plugins/harness-guard.js` | Mechanical enforcement of planning lock, batch claims and `touches` |
| `opencode.json` (optional) | Shared permissions/safety denies |
| `BACKLOG.md` | User stories (user-owned, coarse) |
| `activities.json` | Activities of the selected story (local) |
| `.harness/state.json` | Stage (tracked) |
| `progress/current.md` | Live session |
| `progress/history.md` | Durable record (tracked) |
| `progress/explore/` | Research notes |
| `docs/planning/*` | Foundation documents and diagrams |
| `docs/harness/*` | How verification and the harness work |

There is **no reviewer role**. Review is the user's job, at two explicit
checkpoints:

1. **Plan approval** — after activities are derived, before execution.
2. **Story approval** — after the full gate is green, before `done`.

A cold agent must answer three questions without guessing:

1. *Where am I?* → `--stage`.
2. *What do I run now?* → `--fast`, `--schedule`, `--activity`, `--story`.
3. *When is it done?* → `verification.md` + the approval protocol.

---

## 4. Work model: stories → activities → tasks

`BACKLOG.md` is **not** replaced by activities. It is the source they come from.

```
BACKLOG.md                activities.json                 tasks
user stories  --select--> activities (storyId)  -->  tasks (per activity)
(coarse)                  (derived just-in-time)          (technical steps)
```

- 50 stories do not produce 80 activities. Only the selected story is decomposed.
- An activity is a cohesive set of interrelated tasks, usually one technical
  layer.
- Activities are flat: each carries `storyId`. There is no nested story object.
- `dependsOn` and `touches` make the plan schedulable.

### Schema

```json
{
  "activities": [
    {
      "id": "ACT-02",
      "storyId": "US-003",
      "name": "auth-contract",
      "status": "pending",
      "dependsOn": ["ACT-01"],
      "touches": ["src/auth/http/", "docs/openapi/"],
      "tasks": [
        { "id": "T-01", "description": "Endpoints and error shapes in OpenAPI", "status": "pending" },
        { "id": "T-02", "description": "Controller signatures with docstrings", "status": "pending" }
      ]
    }
  ]
}
```

- `storyId` — the only story link; must exist in `BACKLOG.md`.
- `dependsOn` — `[]` means it can run in the next batch.
- `touches` — required path prefixes; two activities in one batch must not
  overlap.
- `tasks` — at least one; the first task of every activity is its documentation
  task.
- No `type`, no `breaking`, no per-task verification level. The step decides the
  gates.

### Derivation example

Story: *"As a user I want to log in"*:

| Activity | Depends on | Touches | Tasks |
| --- | --- | --- | --- |
| data model | — | `src/auth/model/`, `db/migrations/` | docstrings, table, migration, repository interface |
| contract | model | `src/auth/http/`, `docs/openapi/` | OpenAPI shape, controller signatures + docstrings, error catalog |
| tests | contract | `src/auth/tests/` | happy path, invalid credentials, locked account, expiry |
| implementation | tests | `src/auth/service/`, `src/auth/middleware/` | logic, token issuance, session, wiring |
| integration | implementation | `db/`, `docs/` | migration run, docsgen, smoke |

The tests activity precedes implementation: that is the story-scoped TDD.

---

## 5. Gate architecture

### Planning gates

| Gate | Proves |
| --- | --- |
| `planning-structure` | `docs/planning/{adr,c4,sequences}` exist |
| `backlog` | `BACKLOG.md` parses, statuses valid, max 1 active |
| `planning-docs` | overview, architecture, stack, structure, contracts, ADR index, diagrams exist and are not placeholders |
| `puml` | every `.puml` has balanced `@startuml`/`@enduml` and content |
| `docsgen-config` | a documentation generator is configured |

No format/lint/type/test/build gate runs in planning. Development commands are
rejected with `[BLOCKED]`, not silently skipped.

### Development steps

| Step | Command | Gates |
| --- | --- | --- |
| iteration | `--fast` | changed format/lint + structure |
| iteration | default | + affected type + targeted tests + docs |
| activity | `--activity ACT-x` | iteration + docsgen + `touches` diff check |
| story | `--story US-x` | all gates + production build + contract + full tests + docsgen + approval |

`touches` is a real gate: every changed file must be inside the union of the
batch's declared prefixes (state files excluded). The plugin enforces the
individual activity's scope at edit time; the harness catches stray edits at
close. That is what makes parallel executors safe.

### Exit codes

- `0` green.
- `1` failed.
- `3` story gates green, **user approval missing**. This is a boundary, not a
  bug.

---

## 6. Scheduling and forced parallelism

`--schedule` computes batches:

```
ready(A)  = pending AND every dependsOn is done
batch     = maximal conflict-free subset of ready (touches disjoint)
```

- Batches are sequential.
- Activities inside a batch run in parallel.
- A batch of N **must** run with N executors in one message.

### How it is enforced, not suggested

| Rule | Mechanism | Strength |
| --- | --- | --- |
| N activities → N agents | plugin blocks edits until the batch is fully claimed | hard |
| Only the orchestrator writes state | agent permissions + plugin | hard |
| Executors stay inside `touches` | harness diff gate + plugin | hard |
| Dependency order | scheduler + claim rejection | hard |
| No batch 2 before batch 1 is green | harness state machine | hard |
| "Remember to parallelize" | `AGENTS.md` + executor prompt | procedural |

Layer 1 — `AGENTS.md`:

> If `--schedule` prints a batch of N activities, launch N `task-executor`
> subagents in ONE message. A batch of N executed one-by-one is a protocol
> violation.

Layer 2 — OpenCode config: the orchestrator may `task`; the executor may not.

Layer 3 — plugin `harness-guard.js`:

- planning: rejects edits outside the foundation allowlist;
- during a batch: rejects state writes and any path outside the union of claimed
  `touches`;
- if the batch is not fully claimed: rejects edits and names the missing
  activities.

Layer 4 — harness: `--claim` rejects an activity that is not ready or not in the
current batch; `--activity` rejects changed files outside `touches`.

Honest limit: an agent could still refuse to follow the schedule, but it cannot
do the work: the plugin blocks the edits.

---

## 7. Stack detection and the documentation generator

Read the manifests before writing commands:

| Signal | Stack / clue |
| --- | --- |
| `package.json` | Node/JS/TS; inspect scripts, deps, type |
| lockfiles | package manager |
| `angular.json` | Angular |
| `vite.config.*`, `next.config.*`, `svelte.config.*`, `nuxt.config.*` | React/Vue/Svelte/Next/Nuxt |
| `tsconfig.json` | TypeScript |
| `pyproject.toml`, `requirements.txt`, `uv.lock`, `poetry.lock` | Python + runner |
| `Cargo.toml` | Rust |
| `go.mod` | Go |
| `Makefile`, `justfile` | project-owned commands |
| `biome.json`, `.eslintrc*`, `ruff.toml` | formatter/linter |
| `vitest.config.*`, `jest.config.*`, `pytest.ini` | test runner |

### Gate → command

| Gate | Node/TS | Angular | Python (uv) | Rust | Go |
| --- | --- | --- | --- | --- | --- |
| format | `prettier --check .` | `prettier --check .` | `ruff format --check .` | `cargo fmt -- --check` | `gofmt -l .` |
| lint | `eslint . --max-warnings 0` | `eslint . --max-warnings 0` | `ruff check .` | `cargo clippy -- -D warnings` | `go vet ./...` |
| type | `tsc --noEmit` | per-package `tsc -p` | `mypy`/`pyright` | `cargo check` | `go build ./...` |
| build | `npm run build` | `ng build` | `uv build` | `cargo build --release` | `go build ./...` |
| test | `npm test` | `ng test --watch=false` | `uv run pytest -q` | `cargo test` | `go test ./...` |
| docsgen | `typedoc` | `compodoc` | `mkdocs build --strict` / `sphinx-build` | `cargo doc` | `go doc` / `pkgsite` |

### Documentation generator per stack

| Stack | Generator | Gate command |
| --- | --- | --- |
| TypeScript | TypeDoc | `npx typedoc` |
| JS/TS + JSDoc | JSDoc + jsdoc2md | `npm run docs:gen` |
| Angular | Compodoc | `npx compodoc -p tsconfig.json` |
| Python | MkDocs + mkdocstrings, or Sphinx | `mkdocs build --strict` / `sphinx-build -b html` |
| Rust | rustdoc | `cargo doc --no-deps` |
| Go | godoc / pkgsite | `go doc ./...` |
| Java/Kotlin | Javadoc / Dokka | `./gradlew dokkaHtml` |
| C/C++ | Doxygen | `doxygen Doxyfile` |

The generator is chosen in **planning**. `--start-dev` fails without one.

### Detection rules

1. Prefer project-owned scripts over guessed binaries.
2. One package manager per repo, detected from the lockfile.
3. Split type checks per package when packages are independent.
4. Use the test runner the project already has; never add a second.
5. Missing gate → `null` and `[SKIP]`; never invent a command that may pass
   silently.

---

## 8. Performance rules

1. **Async `spawn` + `Promise.all`**, never `spawnSync` in a loop. The canonical
   repo went from 118.65 s to a documented 70–85 s target by parallelizing
   independent gates.
2. **Parallelize independent gates**; serialize only real dependencies
   (`contract` after `build`).
3. **Filter changed files before spawning.** Two changed files are seconds; five
   hundred are the full gate.
4. **Cache directory walks** for structure and spec discovery.
5. **Ignore generated output first.** A gate that scans `dist/` is not a gate.
6. **Use the steps, not discipline.** `--fast` for editing, `--activity` for
   closing an activity, `--story` once for evidence.
7. **One docsgen run per activity close**, not per save.
8. **Measure before optimizing further.**

---

## 9. Workflow rules

### The verification ladder

| Step | Action | Cost |
| --- | --- | --- |
| 0 | Read the activity, its `touches` and the relevant planning docs | 0 s |
| 1 | `--fast` after a meaningful edit | seconds |
| 2 | `--activity` when the activity is done | targeted + docsgen |
| 3 | `--story` at close, then user approval | full + build |

Rules:

- Never run `--story` after every edit.
- Budget **≤2 story-level runs per story**: one for evidence, one after approval.
- Do not edit while the harness runs.
- Verify activities serially, after their executors return.
- One batch at a time; one story at a time.
- Document in `progress/current.md` while working.

### Approval policy

- The agent presents evidence; the user decides.
- `--story` exits `3` without approval.
- `--approve-story` requires the story to be `awaiting-approval` and `--by`.
- No agent may mark `done` on its own. There is no reviewer role.

### Documentation step

Every activity's first task is documentation:

- decision → ADR + affected planning doc/diagram;
- new/changed public surface → docstrings in code;
- no standalone markdown feature docs.

The linter enforces docblock presence. If those rules are off, turn them on
before claiming the documentation policy.

---

## 10. Using the assets

Copy, do not reconstruct:

| Asset | Copy to |
| --- | --- |
| `assets/AGENTS.md.tmpl` | `AGENTS.md` |
| `assets/AGENTS.verification.md.tmpl` | paste into an existing `AGENTS.md` |
| `assets/opencode/agent/main-orchestrator.md.tmpl` | `.opencode/agent/main-orchestrator.md` |
| `assets/opencode/agent/task-executor.md.tmpl` | `.opencode/agent/task-executor.md` |
| `assets/opencode/plugins/harness-guard.js.tmpl` | `.opencode/plugins/harness-guard.js` |
| `assets/opencode.json.tmpl` | `opencode.json` |
| `assets/BACKLOG.md.tmpl` | `BACKLOG.md` |
| `assets/docs/planning/*.tmpl` | `docs/planning/*` (overview, architecture, stack, structure, contracts) |
| `assets/docs/planning/adr/*.tmpl` | `docs/planning/adr/` (index + template) |
| `assets/docs/planning/c4/*.tmpl` | `docs/planning/c4/` (context, container, component) |
| `assets/docs/planning/sequences/login.puml.tmpl` | `docs/planning/sequences/` |
| `assets/docs/harness/*.md.tmpl` | `docs/harness/*` |
| `assets/verification.md.tmpl` | `docs/harness/verification.md` |
| `assets/progress/current.md.tmpl` | `progress/current.md` |
| `assets/progress/history.md.tmpl` | `progress/history.md` |
| `assets/build_harness.mjs.tmpl` | `scripts/build_harness.mjs` |
| `assets/harness.mjs.tmpl` | `scripts/harness.mjs` |
| `assets/harness-lib.mjs.tmpl` | `scripts/harness-lib.mjs` |
| `assets/harness-lib.spec.mjs.tmpl` | `scripts/harness-lib.spec.mjs` |
| `assets/activities.schema.json` | `activities.schema.json` |
| `assets/package.scripts.json.tmpl` | merge into `package.json` |
| `assets/GIT_TRACKING.md` | `docs/harness/git-tracking.md` |
| `assets/gitignore.tmpl` | `.gitignore` |
| `assets/ignores.txt` | `.prettierignore`, `.eslintignore`, structure walk |

### Placeholders

`{{PROJECT_NAME}}`, `{{STACK}}`, `{{PACKAGE_MANAGER}}`, `{{HARNESS_CMD}}`,
`{{TEST_CMD}}`, `{{BUILD_CMD}}`, `{{DOCSGEN_CMD}}`, `{{FORMAT_CMD}}`,
`{{LINT_CMD}}`, `{{TYPE_CMD}}`, `{{TARGET_TEST_CMD}}`, `{{SOURCE_DIR}}`,
`{{DOCS_DIR}}`, `{{REPO_MAP_ROWS}}`, `{{LAYOUT_TREE}}`, `{{QUALITY_GATES}}`,
`{{TREE}}`, `{{FILE_SET}}`, `{{TEST_LOCATION}}`, `{{LANGUAGE}}`,
`{{FORMATTER}}`, `{{LINTER}}`, `{{MAX_LINE}}`, `{{TEST_RUNNER}}`,
`{{GENERATED_DOCS}}`, `{{PLANNING_DOCS}}`, `{{CODE_ROOTS}}`, `{{TEST_ROOTS}}`,
`{{IGNORE_DIRS}}`, `{{STAGE}}`.

The docs gate checks word count, not truth. Replace every placeholder and delete
the HTML replacement comments before declaring the harness ready. The
`docs/planning/*` templates also contain content placeholders
(`{{PRIMARY_ACTOR}}`, `{{LAYER}}`, `{{CONTRACT_NAME}}`, …); a template that still
contains one is not a foundation.

### Filling `harness.mjs`

Replace the `CONFIG` block:

- `codeRoots` — changed files that count as code.
- `testRoots` — roots searched for sibling specs.
- `planning.docs` — required foundation docs and minimum words.
- `planning.requiredDirs` — planning folders.
- `structure` — required dirs, forbidden patterns, max lines.
- `specs` — sibling discovery and targeted command.
- `gates` — `format`, `lint`, `type`, `build`, `test`, `contract`, `docsgen`,
  `docblocks`.
- `docs` — harness docs checked in development.
- `selfTest` — pure logic test.

Every gate has a `// GATE:` comment. Delete what the stack does not have; set
optional gates to `null`. `docsgen` is required in practice.

---

## 11. Git tracking contract

Put the policy in `docs/harness/git-tracking.md`; start `.gitignore` from
`assets/gitignore.tmpl`.

**Track:** source, tests, `BACKLOG.md`, `.harness/state.json`,
`docs/planning/**`, `docs/harness/**`, `AGENTS.md`, `.opencode/agent/*`,
`.opencode/plugins/*`, `opencode.json`, harness scripts, manifests and one
lockfile, formatter/linter/test config, `progress/history.md`,
`progress/explore/.gitkeep`, `.env.example`.

**Do not track:** dependencies, build output, generated documentation,
framework/test caches, logs, secrets, `activities.json`,
`.harness/schedule.json`, `.harness/claims/**`, `.harness/approvals.json`,
`progress/current.md`.

Verify:

```bash
git check-ignore -v activities.json .harness/schedule.json progress/current.md
git ls-files .harness/ docs/planning/ progress/
git status --ignored --short | head -40
```

Never commit a secret to make the harness green. A credential in git is a
release blocker, not a gate.

---

## 12. Validation: prove the harness works

| Check | Target |
| --- | --- |
| `--planning` on a complete foundation | green, < 2 s |
| `--planning` on a missing diagram | red with the file name |
| `--activity` in planning | exit ≠ 0 with `[BLOCKED]` |
| `--start-dev` without `--user-approved` | exit ≠ 0 |
| `--start-dev --user-approved` | writes development state + history |
| `--schedule` with independent activities | parallel batch |
| `--schedule` with overlapping `touches` | serial batches |
| `--claim` outside the current batch | exit ≠ 0 |
| `--claim` all batch members | claims complete |
| `--activity` with a file outside `touches` | exit ≠ 0 |
| `--activity` in scope | green, claim released |
| `--story` without approval | exit `3` |
| `--approve-story` on a non-awaiting story | exit ≠ 0 |
| `--story` after approval | green |
| plugin: edit source in planning | rejected |
| plugin: edit before batch complete | rejected |
| plugin: state edit during batch | rejected |
| plugin: edit inside claimed `touches` | allowed |
| `node --test scripts/harness-lib.spec.mjs` | green, < 1 s |

Use a temporary fixture with stubbed gate commands before wiring real ones. The
shipped spec covers backlog parsing, plan validation, batching, claim checks,
touches matching and mode resolution.

---

## 13. Anti-patterns

| Anti-pattern | Why it fails |
| --- | --- |
| Writing code in planning | The foundation changes after the code exists; decisions become archaeology |
| Allowing a return to planning | It becomes rework and a moving target |
| Pre-decomposing the backlog | 50 vague stories do not justify 80 activities |
| Global TDD on vague requirements | Tests encode guesses; the user gets what they did not ask for |
| An agent reviewer | It costs a full run and still is not the user's judgment |
| Serial execution of an independent batch | Wastes the only real parallelism the harness has |
| Relying on `dependsOn` only | Two "independent" activities edit the same ADR index or migration folder |
| No `touches` | Cannot verify scope; parallel agents corrupt each other |
| Markdown feature specs | They duplicate the code and go stale; use docstrings + ADR |
| `docsgen` configured but never run | The docs drift silently; run it at activity close |
| Docblock rules disabled | The documentation policy is a lie |
| Full gate after every edit | The harness gets bypassed |
| Sequential `spawnSync` for independent gates | Sums the critical path |
| Editing while a gate runs | Reads half-written files |
| `--story` without approval | The user loses the only decision that matters |
| Tracking session state or secrets | Conflicts and credential leaks |
| Leaving planning docs as placeholders | Agents invent architecture; the docs gate becomes decoration |
| Silently skipping a gate | A false green is worse than a false red |

---

## Upgrading an existing harness

If the repository already has a harness, **do not re-scaffold it**. A live repo
has history, in-flight work and a foundation that may not exist yet. Use the
upgrade procedure instead of the build sequence:

1. **Inventory read-only** — detect the old CLI (`--for-task`, `--dev-build`),
   typed activities, the reviewer role and the missing planning docs.
2. **Branch and back up** — never migrate on top of uncommitted work, never
   rewrite git history, never delete `progress/history.md`.
3. **Replace the engine** — copy the new `harness.mjs`, `harness-lib.mjs`, spec,
   bootstrap and plugin; run the pure-logic tests before anything else.
4. **Reconstruct the foundation** — reverse-engineer `docs/planning/**` from the
   real code and the old docs; label reconstructed decisions; ask the user for
   intent. Create `BACKLOG.md` from pending work, not from old activities.
5. **Handle documentation debt** — enable the generator and docblock rules; where
   legacy code fails, create a visible baseline plus a backlog story, never a
   silent skip.
6. **Enter development** — finish planning, run `--start-dev --user-approved`,
   then verify the protocol on the first real story.

Stop for user approval at the inventory, the reconstructed foundation and the
stage transition. Full phase-by-phase procedure, rollback and a copy-paste
instruction: `references/migrating-existing-harness.md`.

## Build sequence

Follow the same order you want the project to follow:

1. **Detect** — manifests, package manager, test runner, doc generator.
2. **Contract** — `AGENTS.md`, agent roles, plugin, `opencode.json`, `BACKLOG.md`,
   `progress/`, `docs/harness/*`, git policy.
3. **Foundation** — `docs/planning/*` and PlantUML diagrams, filled with the
   project's real decisions (never placeholders).
4. **Bootstrap** — `scripts/build_harness.mjs` creates `.harness/state.json`,
   `activities.json`, `BACKLOG.md`, `progress/*`, planning folders only when
   missing.
5. **Test the engine** — copy the pure lib + spec; make it green in under 1 s.
6. **Implement the engine** — copy `harness.mjs`, fill `CONFIG`, wire the stage,
   schedule, claim, activity and story commands.
7. **Test the protocol** — fixture with stubbed gates: planning lock, batch
   scheduling, claims, touches, approval exit `3`.
8. **Test the plugin** — planning lock, incomplete batch, state lock, scope
   union.
9. **Wire** — run the real commands, add ignores, fix false positives.
10. **Accept** — run the validation matrix and record real timings.
11. **Document** — update the stack reference and pending optimizations.

## Source-of-truth rule

If the project says X, teach X. If the project says nothing, do not invent it.
Stack-specific rules go in `references/stacks/`. Universal rules stay in this
file and in `references/lessons.md`.

Read `references/lessons.md` before changing the scheduling or gate semantics.
Every lesson there has a symptom, a cause, a fix and a number.
