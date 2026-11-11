# Upgrading a harness that is already installed

> Procedure for an agent that finds a **v1 harness** (task-scoped levels, typed
> tasks, an agent reviewer) or any earlier `harness-builder` output, and must move
> it to the two-stage model **without losing history or rewriting git**.

Read this whole file before touching a file. Migration is not a re-scaffold.

---

## 0. What "already installed" means

Detect the current state read-only:

```bash
git status --short
ls .harness/state.json BACKLOG.md activities.json 2>/dev/null
ls .opencode/agent/ .opencode/plugins/ 2>/dev/null
grep -n -- "--for-task\|--dev-build\|--skip-" scripts/harness.mjs 2>/dev/null
grep -n -- '"type"\|"verification"\|"breaking"' activities.json 2>/dev/null
grep -n -- "light\|medium\|full" docs/harness/verification.md 2>/dev/null
```

| Marker | Meaning |
| --- | --- |
| `.harness/state.json` exists | already on the two-stage model; stop and verify |
| `implementation-reviewer.md` exists | v1 with an agent reviewer |
| `--for-task`, `--dev-build` in the engine | v1 CLI |
| `"type"` / `"verification"` / `"breaking"` in `activities.json` | v1 schema |
| `docs/harness/verification.md` describes `light/medium/full` | v1 policy |
| no `BACKLOG.md` | pre-story model |
| no `docs/planning/` | pre-foundation model |

If the repo is already on the two-stage model, only run the acceptance checks in
§9 and do not migrate a second time.

---

## 1. Safety rules

1. **Ask before starting.** Show the inventory and the plan.
2. **Branch first**: `git switch -c chore/harness-v2` (or the project's naming).
3. **Commit or stash the current work.** Never migrate on top of uncommitted
   changes you do not own.
4. **Back up local state** outside the repo:
   `cp activities.json progress/current.md /tmp/harness-backup-$(date +%s)/`.
5. **Never rewrite git history.** No `reset --hard`, no force push, no
   `rm -rf`.
6. **Never delete `progress/history.md`.** It is the durable record. Append a
   migration entry; do not edit old entries.
7. **Stop for user approval at three points**: inventory, reconstructed
   foundation, and the `--start-dev` transition.

---

## 2. What is kept, merged, replaced, removed

| Old artifact | Action |
| --- | --- |
| `progress/history.md` | **Keep**; append the migration entry |
| `progress/current.md` | **Replace** with the new template; archive the old content in `/tmp` |
| `activities.json` | **Archive** (local). Do not import typed tasks. Rebuild per story |
| `scripts/harness.mjs`, `harness-lib.mjs`, `harness-lib.spec.mjs`, `build_harness.mjs` | **Replace** with the new assets |
| `.opencode/agent/implementation-reviewer.md` | **Delete**; review belongs to the user |
| `.opencode/agent/{main-orchestrator,task-executor}.md` | **Replace** with the new roles |
| `.opencode/plugins/harness-guard.js` | **Add** |
| `AGENTS.md` | **Replace** with the new contract (merge project-specific map rows) |
| `opencode.json` | **Merge** the agent permissions; keep project denies |
| `docs/harness/verification.md` | **Replace**; levels are gone |
| `docs/harness/development.md`, `testing.md` | **Replace**; workflow and TDD scope changed |
| `docs/harness/architecture.md` | **Split**: product architecture → `docs/planning/architecture.md`; harness architecture stays |
| `docs/harness/structure.md`, `conventions.md`, `CHECKPOINTS.md` | **Merge** the project facts, replace the policy |
| `docs/planning/**` | **Create** from the real system (see §4) |
| `BACKLOG.md` | **Create** from open work, not from old activities (see §4) |
| `docs/harness/git-tracking.md`, `.gitignore` | **Update** to the new policy |
| `package.json` scripts | **Merge** the new harness scripts; drop `harness:task`, `--for-task` aliases |

Do not keep a compatibility layer. Two workflows in one repo is how the old one
survives.

---

## 3. Engine installation

Replace the engine, add the guard and the new state files. `assets/` below is
the **harness-builder skill's** `assets/` directory, not the target repo's:

```bash
# from the harness-builder skill assets
cp assets/harness.mjs.tmpl            scripts/harness.mjs
cp assets/harness-lib.mjs.tmpl        scripts/harness-lib.mjs
cp assets/harness-lib.spec.mjs.tmpl   scripts/harness-lib.spec.mjs
cp assets/build_harness.mjs.tmpl      scripts/build_harness.mjs
cp assets/opencode/plugins/harness-guard.js.tmpl .opencode/plugins/harness-guard.js
node scripts/build_harness.mjs "<project>" "<description>"
```

`build_harness.mjs` creates `.harness/state.json` with `stage: "planning"`,
`activities.json`, `BACKLOG.md`, `progress/*` and the planning folders **only if
missing**. It never overwrites an existing file, so run it after the backup and
inspect what it reports.

Then fill `CONFIG` from the detected stack (see `SKILL.md` §10). Do not leave
the example npm commands if the project is not npm.

Run the pure logic tests before going further:

```bash
node --test scripts/harness-lib.spec.mjs
```

They must be green in under a second. If they fail, the engine installation is
wrong; stop.

---

## 4. Reconstruct the foundation (existing codebase)

An existing project has code but no foundation. You must **reverse-engineer**
the planning docs from the real system, not invent them.

Order:

1. **Inventory the code**: entry points, modules, data stores, external calls,
   tests, existing docs.
2. **`docs/planning/overview.md`** — what the product is, who uses it, what is
   out of scope. Ask the user for what the code cannot tell you.
3. **`docs/planning/architecture.md`** — context, containers, layers, data.
   Extract the product parts from the old `docs/harness/architecture.md`;
   do not duplicate the harness section.
4. **`docs/planning/c4/*.puml`** — context, container, component as they exist.
5. **`docs/planning/sequences/*.puml`** — at least the main flow and the most
   fragile flow.
6. **`docs/planning/stack.md`** — what is actually used (read the manifests and
   lockfiles), plus reverse ADRs for decisions already baked into the code.
7. **`docs/planning/structure.md`** — the tree as it is.
8. **`docs/planning/contracts.md`** — APIs, events, schemas, error shapes; use
   the OpenAPI/IDL/proto files if they exist.
9. **`docs/planning/adr/`** — mark reconstructed decisions as
   `status: accepted` with a note: *"reconstructed from the code on DATE"*. Add
   `NNNN-adopt-two-stage-harness.md` for this migration (template:
   `assets/docs/planning/adr/adopt-two-stage-harness.md.tmpl`).

Rules:

- If the code and the old docs disagree, **the code wins** for facts and the user
  wins for intent. Ask.
- Do not "fix" the architecture during reconstruction. Capture it, then create
  stories.
- Every `{{...}}` placeholder must be replaced or turned into an `open question`.
- Reconstructed diagrams are better than none, but they are not evidence of
  correctness; label the reconstruction date.

### `BACKLOG.md`

Create coarse stories from:

- open issues / TODOs / known defects;
- the old `activities.json` **only as context**, not as stories or tasks;
- the user's actual pending wants.

Do not copy old typed tasks. Do not turn every old activity into a story. If a
large in-flight activity exists, ask the user whether to finish it under the old
harness first or re-plan it as a story.

---

## 5. Documentation generator and legacy docstring debt

The new model requires a generator. Existing code rarely has complete docblocks.

1. Choose the generator for the stack (TypeDoc, Compodoc, MkDocs/Sphinx,
   rustdoc, godoc, Dokka, Doxygen). Wire `CONFIG.gates.docsgen`.
2. Enable the docblock linter rules.
3. If the legacy code fails the rule massively, **do not disable it**. Create a
   baseline:
   - generate the current violations into `docs/harness/docblock-baseline.txt`;
   - configure the docblock check to fail only on **new or changed** symbols
     (changed-file scope, or an ignore list generated from the baseline);
   - add a backlog story to burn the baseline down.
4. Document the baseline in `docs/harness/conventions.md` and in the migration
   ADR. A baseline without an owner is just a disabled rule.

The same policy applies to any gate the legacy code fails (lint, type, format):
baseline + visible debt + a story. Never a silent `--skip`.

---

## 6. Documentation merge

- Move product content from `docs/harness/architecture.md` and
  `docs/harness/structure.md` into `docs/planning/`.
- Keep `docs/harness/` about the harness itself.
- Replace `verification.md` completely: levels and `--for-task` no longer exist.
- Rewrite `development.md`: stages, story loop, documentation step.
- Rewrite `testing.md`: story-scoped TDD.
- Update `CHECKPOINTS.md` to C1–C6 (stage, registry, protocol, docs, verification,
  closure).
- Update `conventions.md`: docstrings are the documentation source; ADRs for
  decisions.
- Update `git-tracking.md` and `.gitignore`:
  track `BACKLOG.md`, `.harness/state.json`, `docs/planning/**`,
  `docs/harness/**`, agents, plugin; ignore `activities.json`, the schedule,
  claims, approvals, `current.md` and generated docs.

---

## 7. Stage strategy for an existing codebase

The new state starts in `planning` even though code exists. That is deliberate:
the foundation must be written before the harness lets you build.

1. Complete §§3–6 while still in planning. Those paths are allowed.
2. Run `{{HARNESS_CMD}} --planning` until green.
3. Show the user the reconstructed foundation and the migration ADR.
4. Only with an explicit yes:

```bash
{{HARNESS_CMD}} --start-dev --user-approved --by <name>
```

5. Run `node scripts/harness.mjs --stage` and confirm `development`.

Do not hand-edit `.harness/state.json` to skip planning. If the foundation cannot
be reconstructed, that is a finding: record it in the ADR and ask the user.

---

## 8. In-flight work

- **Old activity `in_progress`**: finish it with the old harness before the
  migration, or stop it and re-plan it as a story. Do not convert it.
- **Old `done` activities**: history. They are not migrated; they stay in
  `progress/history.md` and in the archived `activities.json`.
- **Old reviewer tasks**: delete them with the role. Review is now the user.
- **Uncommitted code changes**: commit or stash them before the migration; the
  plugin and the gates assume a clean baseline.

Append to `progress/history.md`:

```md
## Harness migration — two-stage model

- at: <timestamp>
- from: v1 (task-scoped levels + reviewer)
- engine: scripts/harness.mjs replaced; plugin added
- foundation: reconstructed from the code on <date>
- ADR: docs/planning/adr/NNNN-adopt-two-stage-harness.md
- approved by: <name>
```

---

## 9. Acceptance after migration

Run in order, in the real repo:

- [ ] `node --test scripts/harness-lib.spec.mjs` — green, < 1 s.
- [ ] `{{HARNESS_CMD}} --stage` — prints `planning`.
- [ ] `{{HARNESS_CMD}} --planning` — green with the reconstructed foundation.
- [ ] `{{HARNESS_CMD}} --activity ACT-01` — exits `[BLOCKED]` (still planning).
- [ ] `{{HARNESS_CMD}} --start-dev` without approval — fails.
- [ ] `{{HARNESS_CMD}} --start-dev --user-approved --by <name>` — succeeds.
- [ ] `{{HARNESS_CMD}} --schedule` on the first story — real batches.
- [ ] `{{HARNESS_CMD}} --claim` on an out-of-batch activity — fails.
- [ ] Plugin: editing a source file while in planning (test on a branch) —
      rejected. Verify by calling the hook in a fixture, not by breaking the repo.
- [ ] Plugin: editing state during a batch — rejected.
- [ ] `{{HARNESS_CMD}} --story US-xxx` without approval — exit `3`.
- [ ] `{{HARNESS_CMD}} --approve-story US-xxx --by <name>` then `--story` — green.
- [ ] `git check-ignore -v activities.json progress/current.md` — ignored.
- [ ] `git ls-files docs/planning/` — tracked.
- [ ] Old reviewer role and `--for-task` references — gone from the repo.

If any box fails, fix it before declaring the migration done. A half-migrated
harness is worse than the old one.

---

## 10. Rollback

- The migration lives on its own branch and one or more commits. Go back with
  `git switch <old-branch>`; never `reset --hard`.
- The plugin is a single file: removing `.opencode/plugins/harness-guard.js`
  disables the mechanical guard, but the new engine still expects the new state.
  Do not do this as a "fix"; revert the branch.
- If the planning reconstruction is rejected by the user, keep the branch and ask
  for the missing decisions. Do not enter development with a placeholder
  foundation.

---

## 11. Copy-paste instruction for the user

```text
Use the harness-builder skill to UPGRADE the harness that is already installed
in this repository to the two-stage model.

Read references/migrating-existing-harness.md first and follow it phase by
phase. Do not re-scaffold, do not delete progress/history.md, do not rewrite git
history, and do not disable gates to make the migration pass.

Stop and ask me at three points:
1. after the inventory (what exists, what will be replaced);
2. after reconstructing docs/planning/** and the migration ADR;
3. before running --start-dev.

If an old activity is in progress, ask me whether to finish it first or re-plan
it as a story. Legacy docstring/lint debt must become a baseline plus a backlog
story, never a silent skip.
```
