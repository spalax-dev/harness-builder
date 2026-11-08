---
name: harness-builder
description: >-
  Build or redesign a repository verification harness with task-scoped levels
  (light/medium/full), --for-task/--explain/--fast/--changed modes, targeted
  changed-file tests, selective reviewer policy, and copy-paste templates.
  Use when a project needs fast quality gates, CI/pre-commit verification, or
  when every change currently runs the full suite. Stack-agnostic: detects
  Node/TypeScript, Angular/React/Vue/Svelte, Python, Rust, Go and monorepos,
  and adapts the gate commands instead of assuming one framework.
---

# Harness Builder

Build a verification harness that is **fast for small tasks, strict for risky
ones, and honest about what it did not check**. The harness is an enforcement
layer for the project's own rules; it does not invent policy.

Use the canonical `tuneles` harness as the reference implementation, but do not
copy its project-specific rules (Angular structure, Tailwind tokens, CLI layout)
unless the target project already has equivalent rules.

A harness is not just the script. It is the **agent contract**: an entry point
(`AGENTS.md`), roles (`.opencode/agent/*`), a session tracker (`progress/`),
durable docs (`docs/harness/*`) and a tracked-vs-ignored policy. The engine is
maybe half the work; the contract is what makes it usable by another agent.

---

## 1. When to build a harness — and when not to

Build one when **at least three gates carry independent signal** and a full run
costs more than a coffee-break edit loop. A useful heuristic:

- The project has a formatter/linter, a compiler or type checker, a build, and
  a test suite; any three of those catches a different class of failure.
- The full run is slower than ~20 s, or developers repeatedly skip it.
- Changes are frequent enough that running everything every time wastes more
  than the harness costs to build.

Do **not** build one when:

- The project is a three-file script with one test command. A single `npm test`
  is already the harness.
- The full run is under ~10 s and stable. Optimize later, when it hurts.
- There is no agreed definition of done. A harness cannot enforce a policy that
  does not exist; write the docs first.
- You only need a one-off check. Put it in a script, not in a gate.

A harness has a cost: config maintenance, stack detection, and false positives.
Start with the smallest gate set that catches real regressions.

---

## 2. Philosophy: `docs → test → implement`, applied to every gate

A gate is only legitimate when it verifies a documented expectation.

| Phase         | Gate family                      | What it validates                                       |
| ------------- | -------------------------------- | ------------------------------------------------------- |
| **docs**      | `docs`, `structure`              | The project's own rules exist and are not placeholders  |
| **test**      | `format`, `lint`, `type`, `test` | The code is well-formed, typed and behaviorally covered |
| **implement** | `build`, `contract`              | The artifact exists and matches its public contract     |

Do not add a gate because “it is standard”. Add it because a failure would
catch a bug the other gates miss. Remove it when it only duplicates another
gate or when its output is ignored.

### Gate selection table

| Gate        | Keep when                                              | Drop when                                             |
| ----------- | ------------------------------------------------------ | ----------------------------------------------------- |
| `format`    | The team wants deterministic diffs                     | No formatter, or formatting is intentionally free     |
| `lint`      | Rules catch real defects or forbidden patterns         | The linter is a style duplicate of the formatter      |
| `type`      | The language has a checker and public contracts        | The project has no types or the checker is unreliable |
| `build`     | The shipped artifact can break independently of tests  | The app is interpreted and has no build step          |
| `test`      | There is behavior worth protecting                     | There are no tests yet (write them first)             |
| `contract`  | The artifact has a public shape (CSS, exports, schema) | Nothing consumes the artifact                         |
| `structure` | The tree/naming is a documented contract               | The layout is intentionally free-form                 |
| `docs`      | Docs are the source of truth and can go stale          | There are no maintained docs                          |

A gate with no owner and no failure story is noise. Delete it.

---

## 2.5 The agent contract (not just the engine)

A harness script is only the execution layer. Another agent cannot use it
consistently without an entry point, roles, a tracker and durable docs.

Ship these together:

| File / folder                                | Responsibility                                                                                            |
| -------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| `AGENTS.md`                                  | Entry point: repo map, stack, hard rules, verification levels, activity schema, delegation, session close |
| `.opencode/agent/main-orchestrator.md`       | Planning, decomposition, delegation, session close                                                        |
| `.opencode/agent/task-executor.md`           | One-task implementation, test-first                                                                       |
| `.opencode/agent/implementation-reviewer.md` | Full-level validation at activity close                                                                   |
| `opencode.json` (optional)                   | Shared permissions/safety denies                                                                          |
| `progress/current.md`                        | Live session: activity, task, plan, notes, blockers                                                       |
| `progress/history.md`                        | Durable append-only session record                                                                        |
| `progress/explore/`                          | Sub-agent research notes (kept with `.gitkeep`)                                                           |
| `docs/harness/architecture.md`               | Layout, layers, state, gates                                                                              |
| `docs/harness/conventions.md`                | Format/lint split, naming, imports, errors                                                                |
| `docs/harness/structure.md`                  | Where every file lives and how to add one                                                                 |
| `docs/harness/development.md`                | `docs → test → implement` workflow                                                                        |
| `docs/harness/testing.md`                    | Mutation mindset, test-first, assertions, mocking                                                         |
| `docs/harness/verification.md`               | Levels, gate matrix, docs/tests per type                                                                  |
| `docs/harness/CHECKPOINTS.md`                | Final state evaluation                                                                                    |

The contract is what lets a cold agent answer three questions without guessing:

1. **What do I run now?** → `--for-task`, `--fast`, full at close.
2. **Where does this file live?** → `structure.md` + `architecture.md`.
3. **When is it done?** → `verification.md` + `CHECKPOINTS.md`.

If you ship the engine without this layer, the next agent will invent its own
workflow and the gates will drift.

---

## 3. Gate architecture

### Gate types

- **docs** — required docs exist and meet a minimum content threshold.
- **structure** — required dirs/files, forbidden patterns, max file size,
  naming rules. Cheap, always on.
- **lint** — static rules and forbidden patterns.
- **type** — compiler/type checker; can be split per package.
- **build** — production artifact.
- **test** — full suite or targeted specs.
- **contract** — artifact assertions that only make sense **after** a build.

### Levels

| Level    | Intent                                       | Typical gates                                               |
| -------- | -------------------------------------------- | ----------------------------------------------------------- |
| `light`  | docs/config/chore loop                       | changed format/lint + structure/docs + harness self-test    |
| `medium` | behavior change without public contract risk | + affected type check + targeted specs                      |
| `full`   | feature/refactor/breaking, activity close    | all gates, production build, contract after build, reviewer |
| `none`   | explicit escape hatch                        | every gate skipped; document why it was used                |

### Modes

| Mode          | One-line behaviour                                                    |
| ------------- | --------------------------------------------------------------------- |
| default       | full, every configured gate (final gate)                              |
| `--for-task`  | resolve the task type to a level and run only that level              |
| `--explain`   | print level + gate set, run no gate                                   |
| `--fast`      | changed-file iteration: no type/build/test/contract                   |
| `--changed`   | auto light/medium from changed files; targeted tests                  |
| `--dev-build` | development build for faster artifact checks; not a full substitute   |
| `--skip-*`    | escape individual gates; print a warning when a “full” run is reduced |

The modes are not cosmetic. They are the difference between a 5 s edit loop and
a 2 min loop.

---

## 4. Stack detection

Read the manifests before writing any command. Do not hardcode a stack.

| Signal file                                                             | Stack / tooling clue                                      |
| ----------------------------------------------------------------------- | --------------------------------------------------------- |
| `package.json`                                                          | Node/JS/TS; inspect `scripts`, `dependencies`, `type`     |
| `pnpm-lock.yaml` / `package-lock.json` / `yarn.lock` / `bun.lockb`      | package manager                                           |
| `angular.json`                                                          | Angular; use `ng` targets                                 |
| `vite.config.*` / `next.config.*` / `svelte.config.*` / `nuxt.config.*` | React/Vue/Svelte/Next/Nuxt + Vite                         |
| `tsconfig.json`                                                         | TypeScript; check `noEmit`, project references            |
| `pyproject.toml` / `requirements.txt` / `poetry.lock` / `uv.lock`       | Python + tool runner                                      |
| `Cargo.toml`                                                            | Rust; `cargo fmt/clippy/test/build`                       |
| `go.mod`                                                                | Go; `gofmt`/`go vet`/`go test`/`go build`                 |
| `Makefile` / `justfile`                                                 | Project-owned commands; prefer them over guessed binaries |
| `biome.json` / `.eslintrc*` / `ruff.toml`                               | Formatter/linter choice                                   |
| `vitest.config.*` / `jest.config.*` / `pytest.ini`                      | Test runner                                               |

### Stack → gate command table

| Gate        | Node/TS (pnpm example)                        | Angular (this repo)                                     | Python (uv example)                       | Rust                          | Go                  |
| ----------- | --------------------------------------------- | ------------------------------------------------------- | ----------------------------------------- | ----------------------------- | ------------------- |
| format      | `pnpm exec prettier --check .`                | `pnpm exec prettier --check .`                          | `uv run ruff format --check .`            | `cargo fmt -- --check`        | `gofmt -l .`        |
| lint        | `pnpm exec eslint . --max-warnings 0`         | `pnpm exec eslint . --max-warnings 0`                   | `uv run ruff check .`                     | `cargo clippy -- -D warnings` | `go vet ./...`      |
| type        | `pnpm exec tsc --noEmit`                      | app+spec+CLI `tsc -p ...`                               | `uv run mypy .` / `pyright`               | `cargo check`                 | `go build ./...`    |
| build       | `pnpm run build`                              | `pnpm run build` (`ng build`)                           | `uv build`                                | `cargo build --release`       | `go build ./...`    |
| test full   | `pnpm run test:ci`                            | `pnpm run test:ci` + `test:tunelesctl`                  | `uv run pytest -q`                        | `cargo test`                  | `go test ./...`     |
| test target | `pnpm exec vitest run <spec>`                 | `ng test --watch=false --include=<spec>` + `vitest run` | `uv run pytest <spec>`                    | `cargo test <name>`           | `go test ./pkg/...` |
| contract    | emitted CSS/exports/bundle check              | Tailwind `@theme` tokens in `dist` CSS                  | `python -c "import pkg"` / wheel contents | binary/feature check          | binary/API check    |
| structure   | dirs, forbidden patterns, component file sets | `src/app` tree + `tunelesctl` rules                     | `src/<pkg>` layout, no `__pycache__`      | module layout, no `unsafe`    | package layout      |

For the full command details, read:

- `references/stacks/angular.md` — derived from the canonical repo.
- `references/stacks/node.md` — Node/TS, React/Vue/Svelte, monorepos.
- `references/stacks/python.md` — uv/poetry/pip, ruff/black, mypy/pyright, pytest.

### Detection rules

1. Prefer project-owned scripts (`package.json` scripts, `Makefile`,
   `pyproject.toml` tasks) over guessed binaries.
2. Detect the package manager from the lockfile; keep one manager per repo.
3. Split full type checks per package when the project has independent
   packages; otherwise one check is enough.
4. Use the test runner the project already uses. Do not introduce a second one.
5. If a gate command is missing, set it to `null` and print `[SKIP] gate: not
configured`; never invent a command that might silently pass.

---

## 5. Performance rules

These come from the canonical repo and its measured runs.

1. **Use async `spawn` + `Promise`, never `spawnSync` in a loop.**
   Independent gates should overlap. The canonical full run was 118.65 s with
   sequential `spawnSync`; the documented parallelization target is 70–85 s
   (`docs/harness/verification.md` §5). App tests alone are ~51 s, so no
   harness can go below that without splitting the test suite.
2. **Parallelize independent gates with `Promise.all`.** Format, lint and type
   are independent. Build and tests are independent unless the tests consume
   the build artifact.
3. **Serialize only real dependencies.** A contract check that reads `dist/`
   runs after the build. A migration check runs after the schema gate. Do not
   serialize “just in case”.
4. **Cache directory walks.** Structure and spec discovery walk the tree; walk
   once per run and reuse the list.
5. **Filter changed files before spawning.** A changed-file Prettier/ESLint run
   on 2 files is seconds; on 500 files it is the full gate.
6. **Verify ignore files first.** If `node_modules/`, `dist/`, `.venv/`,
   `target/` are not ignored, every gate scans generated output.
7. **Use modes, not discipline.** The harness should make the fast path the
   easy path: `--fast` for iteration, `--for-task` for done, full at close.
8. **Measure before optimizing further.** A leaf app spec is ~5–8 s; the root
   contract spec can be ~23 s because it imports the whole app. Optimize the
   hot spec, not the whole harness.

---

## 6. Workflow rules

Extracted from `AGENTS.md` and the canonical activity history.

### The verification ladder

| Step | Action                                                   | Cost     |
| ---- | -------------------------------------------------------- | -------- |
| 0    | Read the file, its sibling spec and the task type        | 0 s      |
| 1    | `--fast` after a meaningful edit                         | 3–8 s    |
| 1.5  | Targeted spec when a failure names a file                | 3–15 s   |
| 2    | `--for-task <ref>` when the task is done                 | 5–100 s  |
| 3    | Full at activity close + reviewer when the level is full | 90–120 s |

Rules:

- **Never run full after every edit.** It is the most common way to make a
  harness hated and ignored.
- **Budget ≤2 full runs per activity.** A third needs a concrete reason.
- **Do not edit while the harness runs.** Captured output and partial files
  produce confusing failures.
- **Reviewer only at activity close**, only for full-level activities. The
  reviewer needs the whole change, not a partial one.
- **Run the task's level, not the activity's maximum**, during iteration.
- **One task at a time**; parallel tasks only inside one activity.
- **Document in the session file while working**, not after.

### Reviewer policy

- `feat`, `refactor`, `breaking: true` → full → reviewer at close.
- `fix`, `perf`, `test` → medium → no reviewer.
- `chore`, `docs`, `style`, `ci`, `build` → light → no reviewer.
- A full activity with no reviewer task gets a warning, not a hard failure.
- The reviewer task should come after implementation tasks; warn if it is first.

---

## 7. Task-type-aware levels

Level mapping:

| Type                                        | Level  | Why                                                               |
| ------------------------------------------- | ------ | ----------------------------------------------------------------- |
| `feat`                                      | full   | New behavior and public surface; the whole artifact matters       |
| `refactor`                                  | full   | Behavior must not change; tests and types are the proof           |
| `fix` + `breaking: true`                    | full   | The contract breaks; specs and docs must move with it             |
| `fix` / `perf` / `test`                     | medium | Behavior changes locally; affected type + targeted specs suffice  |
| `chore` / `docs` / `style` / `ci` / `build` | light  | No runtime behavior; changed format/lint + structure/docs suffice |
| `verification: "none"`                      | none   | Explicit escape hatch; every gate skipped, document why           |

Fallback:

```text
task.type ?? activity.type ?? 'medium'
```

Why `medium` and not `light`: a false green is worse than a false red. Do not
migrate historical activities just to add `type`; the fallback is deliberate
and silent. New tasks should type every task.

`breaking` is a boolean flag, not a type. Convention: `fix` + `breaking` →
`fix!`; `feat` + `breaking` → `feat!`.

---

## 8. Using the assets

Copy, do not reconstruct:

| Asset                                | Copy to / use as                                                                              |
| ------------------------------------ | --------------------------------------------------------------------------------------------- |
| `assets/AGENTS.md.tmpl`              | `AGENTS.md` (entry point)                                                                     |
| `assets/AGENTS.verification.md.tmpl` | paste into `AGENTS.md` when you keep your own structure                                       |
| `assets/opencode/agent/*.md.tmpl`    | `.opencode/agent/*.md` (orchestrator, executor, reviewer)                                     |
| `assets/opencode.json.tmpl`          | `opencode.json` (optional safety/permissions config)                                          |
| `assets/progress/current.md.tmpl`    | `progress/current.md`                                                                         |
| `assets/progress/history.md.tmpl`    | `progress/history.md`                                                                         |
| `assets/progress/explore/.gitkeep`   | `progress/explore/.gitkeep`                                                                   |
| `assets/build_harness.mjs.tmpl`      | `scripts/build_harness.mjs`                                                                   |
| `assets/docs/harness/*.md.tmpl`      | `docs/harness/*.md` (architecture, conventions, structure, development, testing, CHECKPOINTS) |
| `assets/verification.md.tmpl`        | `docs/harness/verification.md`                                                                |
| `assets/harness.mjs.tmpl`            | `scripts/harness.mjs`                                                                         |
| `assets/harness-lib.mjs.tmpl`        | `scripts/harness-lib.mjs`                                                                     |
| `assets/harness-lib.spec.mjs.tmpl`   | `scripts/harness-lib.spec.mjs`                                                                |
| `assets/activities.schema.json`      | `activities.schema.json` (or merge into the repo)                                             |
| `assets/package.scripts.json.tmpl`   | merge into `package.json` `scripts`                                                           |
| `assets/GIT_TRACKING.md`             | `docs/harness/git-tracking.md`                                                                |
| `assets/gitignore.tmpl`              | `.gitignore`                                                                                  |
| `assets/ignores.txt`                 | append to `.prettierignore`, `.eslintignore` and the structure walk                           |

### Filling `harness.mjs`

Replace the `CONFIG` block. The shape is intentional:

- `codeRoots` — changed files that select medium.
- `testRoots` — roots searched for sibling specs (keep tooling out unless it
  has colocated tests).
- `docs` — required docs and minimum word counts.
- `structure` — required dirs, forbidden patterns, max lines.
- `specs` — sibling-spec discovery and targeted command.
- `gates` — `format`, `lint`, `type`, `build`, `test`, `contract`.
- `selfTest` — pure-logic test command; runs in light/medium only.

Every gate has a `// GATE:` comment. Delete a gate from `CONFIG` if the stack
has no equivalent; set it to `null` if it is optional. The template's example
values are npm-based and intentionally runnable; replace them with the detected
package manager and commands.

### Placeholders

The example values are already substituted so the files parse and start. The
names to look for when replacing are:

`{{PROJECT_NAME}}`, `{{STACK}}`, `{{PACKAGE_MANAGER}}`, `{{HARNESS_CMD}}`,
`{{EXPLAIN_CMD}}`, `{{BOOTSTRAP_CMD}}`, `{{FORMAT_CMD}}`, `{{LINT_CMD}}`,
`{{TYPE_CMD}}`, `{{BUILD_CMD}}`, `{{TEST_CMD}}`, `{{TARGET_TEST_CMD}}`,
`{{CONTRACT_CMD}}`, `{{DOC_PATHS}}`, `{{CODE_ROOTS}}`, `{{TEST_ROOTS}}`,
`{{IGNORE_DIRS}}`, `{{SOURCE_DIR}}`, `{{DOCS_DIR}}`, `{{REPO_MAP_ROWS}}`,
`{{LAYOUT_TREE}}`, `{{LAYERS_TABLE}}`, `{{QUALITY_GATES}}`, `{{TREE}}`,
`{{FILE_SET}}`, `{{TEST_LOCATION}}`, `{{LANGUAGE}}`, `{{FORMATTER}}`,
`{{LINTER}}`, `{{MAX_LINE}}`, `{{TEST_RUNNER}}`.

The docs gate checks word count, not truth. Replace every `{{...}}` and delete
the HTML replacement comments before declaring the harness ready; otherwise
the docs are placeholders that happen to be long.

### Git tracking contract

A harness is also a repository contract. Put the policy in
`docs/harness/git-tracking.md` and start `.gitignore` from
`assets/gitignore.tmpl`.

**Track:** source, tests, `AGENTS.md`, `.opencode/agent/*`, `opencode.json`,
the harness scripts, `docs/` (including `docs/harness/*`), manifests and one
lockfile, formatter/linter/test config, `progress/history.md`,
`progress/explore/.gitkeep`, `.env.example`.

**Do not track:** dependencies, build output, framework/test caches, logs,
secrets/credentials/service-account JSON, and live session state.

The canonical repo deliberately ignores `/activities.json` and
`/progress/current.md` (local session state) and tracks
`progress/history.md` and `progress/explore/.gitkeep`. That avoids conflicts
between agents and branches. If the team wants shared activity state, that is
a conscious trade-off; document it.

Verify the policy:

```bash
git check-ignore -v activities.json progress/current.md
git ls-files progress/
git status --ignored --short | head -40
```

Never commit a secret to “make the harness green”. A credential in git is a
release blocker, not a gate.

---

## 9. Validation: prove the harness works

Minimum acceptance matrix after wiring the harness:

| Check                                    | Target                                                                                |
| ---------------------------------------- | ------------------------------------------------------------------------------------- |
| `--explain` on a light task              | < 1 s, no gate output                                                                 |
| `--for-task` light                       | ≤ 10 s                                                                                |
| `--for-task` medium with a changed spec  | < 15 s, targeted                                                                      |
| `--for-task` medium with no sibling spec | fallback + warning, still < full                                                      |
| `--for-task` full                        | all gates, ≤ 2 min                                                                    |
| default full                             | all gates, green                                                                      |
| `--changed` with code changes            | medium, targeted                                                                      |
| `--changed` with docs only               | light                                                                                 |
| invalid `--for-task` id                  | exit ≠ 0, lists valid refs                                                            |
| `--for-task` on a done task              | warning, runs anyway                                                                  |
| full over a light in-progress task       | warning, not failure                                                                  |
| full activity without reviewer           | warning, not failure                                                                  |
| docs grep                                | no “run full on every task”                                                           |
| `AGENTS.md` cold start                   | points to docs/harness + commands                                                     |
| `.opencode/agent/` roles                 | orchestrator + executor + reviewer                                                    |
| `progress/current.md` + `history.md`     | present; history tracked, current local                                               |
| `docs/harness/`                          | architecture, conventions, structure, development, testing, verification, CHECKPOINTS |
| Git policy                               | `activities.json`/`current.md` ignored; history tracked                               |
| Cold-agent question                      | can answer “what to run / where / when done”                                          |

Also test the pure logic with `node --test scripts/harness-lib.spec.mjs`.
It must be under 1 s and dependency-free.

Use a temporary fixture project with stubbed gate commands to test every mode
before wiring real commands. The canonical harness tests 27 pure/integration
cases; a new harness should at least cover the mapping, parse, fallback,
activity level and changed-file classification.

---

## 10. Anti-patterns

| Anti-pattern                                          | Why it fails                                                        |
| ----------------------------------------------------- | ------------------------------------------------------------------- |
| Full gate after every edit                            | 2 min × dozens of edits = the harness is bypassed                   |
| Sequential `spawnSync` for independent gates          | Sums the critical path; 118 s instead of the 70–85 s target         |
| Editing while the harness runs                        | Reads half-written files; failures point at the wrong place         |
| Retry loop on a red gate                              | Hides the root cause; fix the code or the gate                      |
| Noisy warnings on historical/done activities          | Trains the user to ignore warnings                                  |
| `--no-cache` before a cache exists                    | A flag that disables nothing; false comfort                         |
| `harness:ci` that skips tests                         | Misleading name; CI must run the tests it claims to run             |
| Full run with `--skip-*` treated as full              | It is not a complete verification; warn and do not close on it      |
| Light level for an unknown type                       | A false green is worse than a false red; default to medium          |
| Contract check before the build                       | Reads a stale artifact; run it after the build                      |
| Scanning `node_modules`/`dist`                        | Slow and noisy; add ignores before the first gate                   |
| Reviewer on every task                                | Reviewer sees partial changes; invoke once at activity close        |
| Migrating all historical tasks just to add `type`     | No value and a chance to corrupt working history; use the fallback  |
| Parallelizing by changing gate semantics              | Refactor mechanics only; keep gates, order and per-gate output      |
| Inventing project rules in the harness                | The harness enforces docs, it does not create policy                |
| Shipping the engine without `AGENTS.md`/docs/progress | The next agent invents a workflow and the gates drift               |
| Tracking live session state or secrets                | Merge conflicts and credential leaks; history is the durable record |
| Leaving `docs/harness/*` as placeholders              | Agents invent structure and the docs gate becomes a lie             |
| No `.opencode` roles                                  | No boundary between planning, implementation and review             |

---

## Build sequence

Follow the same `docs → test → implement` order you want the project to follow:

1. **Contract** — `AGENTS.md`, `.opencode/agent/*`, `progress/`, `docs/harness/*`
   and the git policy. This is the entry point a cold agent reads first.
2. **Docs** — write `verification.md`, architecture, conventions, structure,
   testing, development and checkpoints.
3. **Bootstrap** — `scripts/build_harness.mjs` creates `activities.json` and
   `progress/*` only when missing.
4. **Test** — copy the pure lib + spec; make the mapping tests pass.
5. **Implement** — copy the harness template, fill `CONFIG`, wire the modes.
6. **Smoke** — run a fixture with stubbed commands for every mode.
7. **Wire** — run the real commands; add ignores; fix false positives.
8. **Accept** — run the acceptance matrix; record real timings.
9. **Document** — update the stack reference and the pending optimizations.

## Source-of-truth rule

If the project says X, teach X. If the project says nothing, do not invent it.
If a rule is stack-specific, put it in `references/stacks/`. If it is universal,
keep it in this file and in the lessons.

Read `references/lessons.md` before changing the gate semantics. Every lesson
there has a symptom, a cause, a fix and a number from the canonical repo.
