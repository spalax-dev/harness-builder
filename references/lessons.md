# Lessons behind the two-stage harness

Source: the canonical `tuneles` harness (async gates, targeted tests), plus the
redesign that moved review to the user, split planning from development and made
parallel batches mandatory.

Each lesson is **symptom → cause → fix → evidence**. Numbers are measured on the
canonical repo; treat them as order-of-magnitude evidence, not promises.

---

## 1. An agent reviewer costs a full run and still is not the user

- **Symptom:** every full activity paid for an independent reviewer run
  (**91–104 s**) that re-ran the suite and produced a verdict the user could not
  fully trust.
- **Cause:** review was delegated to an agent because it was "the process".
- **Fix:** delete the reviewer role. The agent prepares evidence (diff, tests,
  ADRs, generated docs) and the **user** decides. The harness enforces the
  boundary with exit code `3` and a recorded approval.
- **Evidence:** the canonical reviewer run was 91–104 s per activity; the user
  checkpoint costs a conversation, not a build.
- **Source:** activity #58 history; `docs/harness/verification.md`.

---

## 2. Global TDD fails when the requirement is vague

- **Symptom:** docs and tests written before the first conversation encoded
  guesses; the delivered behavior was not what the user wanted, and the tests
  defended the wrong contract.
- **Cause:** "docs → test → implement" was applied to the whole repository,
  including requirements that were not understood yet.
- **Fix:** keep coarse user stories in `BACKLOG.md`; derive activities only for
  the selected story; apply TDD inside that story (tests activity before the
  implementation activity).
- **Evidence:** the backlog is intentionally sparse; `open questions` is where
  ambiguity lives and is resolved when the story starts.
- **Source:** `docs/harness/development.md`, `docs/harness/testing.md`.

---

## 3. Do not pre-decompose the backlog

- **Symptom:** 50 user stories became 80 activities before any work started. Most
  activities were wrong, and the plan became a second backlog nobody trusted.
- **Cause:** decomposition was treated as a planning deliverable.
- **Fix:** `activities.json` holds only the selected story's activities. Its
  activities carry `storyId`; there is no story object and no cross-story plan.
- **Evidence:** the scheduler and the validator only look at one `storyId` at a
  time.
- **Source:** `harness-lib.mjs` `storyActivities`, `planBatches`.

---

## 4. Two exclusive stages prevent rework

- **Symptom:** every implementation discovery reopened the architecture; planning
  and building interleaved until the foundation described nothing real.
- **Cause:** no boundary between deciding and building.
- **Fix:** `planning → development` is one-way. Planning artifacts stay editable
  during development (a decision can change), but there is no `--plan` command
  and no re-planning phase.
- **Evidence:** the state file is tracked in git; `--start-dev` requires
  `--user-approved`; development commands are `[BLOCKED]` in planning.
- **Source:** `.harness/state.json`, `scripts/harness.mjs`.

---

## 5. Parallelism is the only big win left

- **Symptom:** independent activities (data model, UI shell, observability) were
  executed one after another because "one task at a time" was the rule.
- **Cause:** the old harness parallelized gates but serialized work.
- **Fix:** schedule activities from `dependsOn`; a batch of independent
  activities runs with N executors in one message. Verification stays serial.
- **Evidence:** gate parallelization took the canonical full run from **118.65 s**
  to a **70–85 s** target; work parallelization removes the sum of independent
  activity times, which is larger.
- **Source:** `planBatches`, `--schedule`.

---

## 6. `dependsOn` alone is not enough: you need `touches`

- **Symptom:** two "independent" activities both edited the ADR index and the
  migrations folder; the parallel merge was undebuggable.
- **Cause:** technical independence does not imply write independence.
- **Fix:** every activity declares `touches` (path prefixes). A batch is a
  maximal set of ready activities with disjoint `touches`. Missing `touches` runs
  alone.
- **Evidence:** `touchesOverlap` treats prefixes hierarchically; the activity
  gate re-checks the real diff.
- **Source:** `harness-lib.mjs`, `--activity`.

---

## 7. Asking for parallelism is not enforcing it

- **Symptom:** "launch independent agents in parallel" was ignored; work stayed
  serial.
- **Cause:** a prompt rule is a suggestion.
- **Fix:** four layers: `AGENTS.md` contract, agent permissions (`task` denied to
  executors), the `harness-guard` plugin (edits blocked until the batch is fully
  claimed), and the harness (claims and schedule validation).
- **Evidence:** the plugin rejects edits with "batch incomplete: ACT-04 not
  claimed" until all members claim.
- **Source:** `.opencode/plugins/harness-guard.js`, `--claim`.

---

## 8. The documentation generator is not optional

- **Symptom:** in-code docstrings existed, but nobody could read them; markdown
  feature docs drifted from the code.
- **Cause:** documentation had no build step.
- **Fix:** the harness requires a generator before leaving planning and runs it
  at activity/story close. Markdown feature docs are forbidden; docstrings are
  the source.
- **Evidence:** `--start-dev` fails when `CONFIG.gates.docsgen` is null; the
  `docsgen` gate runs in activity and story steps.
- **Source:** `runDocsgenConfig`, `runDocsgen`.

---

## 9. `DOCUMENT` means decisions and docstrings, not comments

- **Symptom:** the documentation step produced docstrings but no record of why
  the architecture changed.
- **Cause:** "docs" was read as "comments in code".
- **Fix:** every activity's first task documents both: a decision that changes the
  foundation gets an ADR plus the affected planning doc/diagram; new public
  surface gets a docstring.
- **Evidence:** ADR index rules, C4/sequence updates and the `docsgen` gate all
  live in the same step.
- **Source:** `docs/harness/conventions.md`, `docs/harness/development.md`.

---

## 10. The stage record belongs in git

- **Symptom:** a fresh clone did not know whether the project was in planning or
  development; an agent tried to re-plan a built project.
- **Cause:** all harness state was treated as local session noise.
- **Fix:** `.harness/state.json` is tracked (it is a project decision);
  `activities.json`, the schedule, claims, approvals and `progress/current.md`
  stay local.
- **Evidence:** `git check-ignore` distinguishes them; the plugin reads the same
  file.
- **Source:** `assets/GIT_TRACKING.md`, `assets/gitignore.tmpl`.

---

## 11. Async `spawn` + `Promise.all` beats sequential `spawnSync`

- **Symptom:** the canonical full harness took **118.65 s**; app tests alone
  **51.30 s**, ESLint **16.28 s**, Prettier **5.71 s**, build **7.38 s**. The
  critical path was the sum of independent gates.
- **Cause:** `spawnSync` in a top-to-bottom script.
- **Fix:** async `spawn` + `Promise.all`; serialize only real dependencies
  (contract after build).
- **Evidence:** documented target **70–85 s**; app tests alone are ~51 s, so full
  cannot go below that without splitting the suite.
- **Source:** canonical `scripts/harness.mjs`; `docs/harness/verification.md`.

---

## 12. Targeted tests are the real iteration win

- **Symptom:** a one-line component change ran the whole app suite
  (**45–51 s**) plus the CLI suite (**20 s**).
- **Cause:** the harness only knew "run all tests".
- **Fix:** derive sibling specs from changed files; fall back to the package
  suite and warn when a changed source file has no spec.
- **Evidence:** a leaf spec ran **5.5–8.3 s** vs **45–51 s**; 14 targeted CLI
  specs **2.2 s** vs **19.8 s**; `--changed` with 42 files was **13.99 s**.
- **Source:** `testEntry`, `CONFIG.specs`.

---

## 13. `--explain` must be zero-cost

- **Symptom:** `--explain` printed gate output before the level, contradicting
  "run no gate".
- **Cause:** the explain branch lived after the early gate blocks.
- **Fix:** resolve state and the mode quietly, print, exit. The integration test
  asserts no gate output.
- **Evidence:** **< 0.2 s** after the fix.
- **Source:** `main`, `--explain`.

---

## 14. Ignore generated and vendor output before adding gates

- **Symptom:** formatters, linters and structure walks scanned `dist/`,
  `coverage/`, `.angular/`, `node_modules/`; runs were slow and noisy.
- **Cause:** gates did not know what the project does not own.
- **Fix:** define the ignore list first (`.prettierignore`, `.eslintignore`,
  `.gitignore`, structure walk). Then add gates.
- **Evidence:** the canonical ESLint config ignores `dist/**`, `coverage/**`,
  `.angular/**`, `node_modules/**`, `docs/design/stitch/**`, `.opencode/**`.
- **Source:** `eslint.config.mjs`, `assets/ignores.txt`.

---

## 15. Build before artifact/contract checks

- **Symptom:** a token contract passed against a stale `dist/` or failed before
  the build.
- **Cause:** the artifact check read files the build produces.
- **Fix:** contract after build; if the build is skipped, skip the contract and
  say so.
- **Evidence:** the build is ~7.4 s; a false pass costs a broken deploy.
- **Source:** `runDevGates`, `docs/harness/verification.md`.

---

## 16. The engine is not the contract

- **Symptom:** a working `harness.mjs` existed, but a cold agent invented its own
  workflow.
- **Cause:** no entry point, roles, tracker or durable docs.
- **Fix:** ship `AGENTS.md`, the orchestrator/executor roles, the plugin, the
  progress files and `docs/harness/*` with the engine.
- **Evidence:** a cold agent must answer: where am I (`--stage`), what do I run
  (`--fast`/`--activity`/`--story`), when is it done (approval protocol).
- **Source:** `AGENTS.md`, `.opencode/agent/*`, `progress/*`.

---

## 17. A missing gate must say `[SKIP]`, never pass silently

- **Symptom:** a misconfigured gate returned success because the command was
  empty.
- **Cause:** absence of configuration was treated as success.
- **Fix:** `null` gates print `[SKIP]`; required gates (docsgen) fail when
  unconfigured; the summary distinguishes "green" from "not checked".
- **Evidence:** a false green is worse than a false red.
- **Source:** `executeGate`, `runDocsgenConfig`.

---

## 18. Measure the real tree, not the plan

- **Symptom:** the original plan predicted 30–50 s; the baseline was **118.65 s**.
- **Cause:** estimates ignored the app suite (51 s) and the CLI suite (20 s).
- **Fix:** measure each gate before setting budgets. The honest full target is
  "under the project's current full time, with the same gates".
- **Evidence:** baseline **118.65 s**; final full **98.26–104.35 s**; target
  **70–85 s**.
- **Source:** activity #58 baseline and acceptance logs.
