# Lessons from the canonical harness

Source: the `tuneles` repository at commit `8293629`
(`chore(harness): verify by task type with selective reviewer`), plus its
history and the activity #58 acceptance run.

Each lesson is **symptom → cause → fix → number → source**. Numbers are
measured on the canonical repo; treat them as order-of-magnitude evidence, not
as promises for another machine.

---

## 1. Async `spawn` + `Promise.all` beats sequential `spawnSync`

- **Symptom:** the full harness took **118.65 s**; the app suite alone was
  **51.30 s**, the CLI suite **19.77 s**, ESLint **16.28 s**, Prettier
  **5.71 s**, build **7.38 s**. The critical path was the sum of independent
  gates.
- **Cause:** `spawnSync` in a top-to-bottom script. Formatting, linting, type
  checking and the two test suites do not depend on each other, but they ran
  one after another.
- **Fix:** use async `spawn` plus `Promise.all` for independent gates. Keep
  only real dependencies serial (contract after build).
- **Number:** the canonical repo documents a pending parallelization target of
  **70–85 s** (from ~2 min). App tests alone are ~51 s, so full cannot go below
  that without splitting the app suite. Activity #58 measured **98.26 s** full
  and **91.40 s** for a full `--for-task` on the final tree; `harness:fast`
  was **4.76 s**.
- **Source:** `scripts/harness.mjs`; `docs/harness/verification.md` §5;
  measured baseline in activity #58.

> Important: the canonical repo still uses `spawnSync`. The skill's template
> adopts the documented pending optimization. Do not claim the canonical repo
> already runs parallel; it does not.

---

## 2. Build before artifact/contract checks

- **Symptom:** the Tailwind `@theme` check passed against a stale `dist/` CSS
  after source changes, or failed when the build had not run.
- **Cause:** the artifact check reads `dist/tuneles/**/*.css`. Before the build
  there is either no artifact or an old one.
- **Fix:** run the contract gate **after** the build. When the build is skipped,
  skip the contract gate too and say so; never validate stale output.
- **Number:** the canonical Tailwind gate checks four token groups
  (`--color-surface: #131313`, `--color-page: #282828`, radius tokens, `Work
Sans`); the production build is ~7.4 s. A false pass costs a broken deploy,
  not 7 seconds.
- **Source:** `scripts/harness.mjs` section 9; `docs/harness/verification.md`
  §2.9; activity #11.

---

## 3. Targeted tests are the medium-level win

- **Symptom:** a one-line component change ran the whole app suite
  (**45–51 s**) and sometimes the CLI suite (**20 s**) for no reason.
- **Cause:** the harness only knew “run all tests”.
- **Fix:** derive sibling specs from changed files:
  `X.ts → X.spec.ts`, `X.html/css → X.spec.ts`, plus runner-specific patterns.
  If a changed source file has no sibling, fall back to the package suite and
  warn.
- **Number:** a single leaf app spec ran in **5.5–8.3 s** vs **45–51 s** for the
  full suite; 14 targeted CLI specs ran in **2.2 s** vs **19.8 s** for the full
  CLI suite. `--changed` with 42 changed files, all three `tsc` targets and 14
  targeted CLI specs was **13.99 s**.
- **Source:** `scripts/harness.mjs` `collectAppSpecs`/`collectCliSpecs`/
  `testEntry`; activity #58 acceptance.

Caveat: `src/app/app.spec.ts` is the `styles.css` token contract and imports the
whole app; it takes **~23 s**. Do not use it as the benchmark for “targeted is
fast”; optimize or split that spec separately.

---

## 4. `--explain` must be zero-cost

- **Symptom:** the first implementation of `--explain` printed gates 1–3
  (environment, base files, schema) before the level. The reviewer flagged a
  contradiction with “run no gate” in the help, docs and acceptance criteria.
- **Cause:** the explain branch lived after the early gate blocks.
- **Fix:** resolve `activities.json` quietly, parse the ref, compute the mode,
  print the level and checks, exit. No environment/base/gate output.
- **Number:** after the fix, `--explain` takes **< 0.2 s** and prints no
  numbered gate. The integration test asserts `doesNotMatch(/── \d+\./)`.
- **Source:** `scripts/harness.mjs` early `EXPLAIN` block; `scripts/
harness-cli.spec.mjs`; reviewer M1 in activity #58.

---

## 5. Fallback order: sibling spec → package suite → `HEAD~1`

- **Symptom:** a changed source file with no sibling spec produced a green
  medium run that tested nothing relevant.
- **Cause:** “targeted” was treated as “find a spec if convenient”, not as a
  contract.
- **Fix:** if any changed source file has no sibling spec, run the package
  suite and warn. If the working tree is clean, try
  `git diff --name-only HEAD~1` before falling back — this covers “I just
  committed and now I run `--for-task`”.
- **Number:** in the canonical repo, `--changed` classified `scripts/` as code
  (medium) but `testRoots` keeps tooling out of spec discovery; the harness
  self-test covers the tooling. The dangerous case is a changed app/CLI source
  file with no spec: fallback is slower but honest.
- **Source:** `scripts/harness.mjs` `changedFilesForRun`, `collect*Specs`;
  `docs/harness/verification.md` §2.12. The `HEAD~1` path is implemented but
  was not exercised end-to-end in #58 because the tree was dirty; treat it as a
  residual risk and test it in the target repo.

---

## 6. Warn when full runs over a light task

- **Symptom:** a docs/chore task triggered a ~2 min run; the developer learned
  to ignore the harness.
- **Cause:** the default command was full, and nothing pointed at the task
  level.
- **Fix:** when the in-progress task is light and no `--for-task` was passed,
  warn with the exact `--for-task` command. It is a warning, not a failure:
  there are legitimate reasons to run full (pre-PR, release).
- **Number:** light `--for-task` was **4.76–5.78 s**; full was **98.26 s**
  (and **118.65 s** before the type-aware redesign). A 20× difference is worth
  one warning.
- **Source:** `scripts/harness.mjs` `resolveMode`; activity #58 acceptance #5.
  Exempt reviewer tasks: at activity close the reviewer task may be light, but
  the activity level is full, so the warning would recommend the wrong run.

---

## 7. Reviewer only at activity close

- **Symptom:** a reviewer invoked per task saw partial changes, repeated the
  same review, and multiplied full runs.
- **Cause:** the reviewer needs the whole activity to judge cross-cutting
  behavior, docs and tests.
- **Fix:** run the task level during iteration; invoke the reviewer once when
  the activity closes and its level is full. A full activity needs at least one
  `agent: "reviewer"` task; warn (do not fail) if it is missing.
- **Number:** the canonical activity #58 had nine tasks; one reviewer task at
  the end. The reviewer’s independent full run was **91–104 s**. Doing that per
  task would have been 9× the cost.
- **Source:** `AGENTS.md`; `docs/harness/verification.md` §1/§4;
  `.opencode/agent/implementation-reviewer.md`; `activities.json`.

---

## 8. Do not migrate historical activities

- **Symptom:** the type-aware redesign needed `task.type`, but the 25 historical
  activities (ids 32–56) had no task types.
- **Cause:** those activities already worked; a mass migration would touch
  working history for no behavioral gain.
- **Fix:** make `task.type` optional and fall back to
  `task.type ?? activity.type ?? 'medium'`. New tasks type every task; old
  tasks keep working. Warn only when both levels are missing on an active task.
- **Number:** 25 activities stayed untouched; the schema still validates all of
  them. The fallback defaults to **medium**, not light: a false green is worse
  than a false red.
- **Source:** `scripts/harness-lib.mjs`; `assets/activities.schema.json`;
  activity #58 acceptance #1; `docs/harness/verification.md` §2.3.

---

## 9. Parallelizing must not change gate semantics

- **Symptom:** the parallelization refactor risked reordering gates, changing
  output and hiding a contract check.
- **Cause:** “make it faster” is tempting to combine with “make it different”.
- **Fix:** keep the full level functionally identical — same gates, same order,
  same per-gate output. Change only the mechanics (`spawn`/`Promise.all`).
  This is a precondition for reviewing the refactor safely.
- **Number:** full stayed at the same gate set; only the §14 summary hint and
  the required full-over-light warning were additive. The canonical full run
  stayed ~98–118 s depending on load; the target after parallelization is
  **70–85 s**.
- **Source:** activity #58 approval condition; `docs/harness/verification.md`
  §5; `scripts/harness.mjs` section order.

---

## 10. Ignore generated and vendor output before adding gates

- **Symptom:** formatters, linters and structure walks scanned `dist/`,
  `coverage/`, `.angular/`, `node_modules/`; runs were slow and noisy.
- **Cause:** the gates did not know what the project does not own.
- **Fix:** define the ignore list first (`.prettierignore`, `.eslintignore`,
  `.gitignore`, structure walk ignore dirs). Then add gates.
- **Number:** the canonical ESLint config ignores `dist/**`,
  `tunelesctl/dist/**`, `coverage/**`, `.angular/**`, `node_modules/**`,
  `docs/design/stitch/**`, `.opencode/**`. The structure walk skips
  `node_modules` and `dist`. Without those, the full lint is dominated by
  generated files.
- **Source:** `eslint.config.mjs`; `scripts/harness.mjs` structure walks;
  `assets/ignores.txt`.

---

## 11. `--no-cache` only exists when a cache exists

- **Symptom:** the plan proposed a `harness:no-cache` alias before any cache
  mechanism.
- **Cause:** aliases are cheap to add; the mechanism is not.
- **Fix:** drop `--no-cache` until there is a content-hash cache. When the cache
  lands, add the flag and document the key.
- **Number:** with no changes, `harness:fast` runs only structure/docs in
  **0.21 s** (post-commit, clean tree). There is nothing for a cache to disable
  yet.
- **Source:** activity #58 decision; `docs/harness/verification.md` §5;
  `package.json` (no `harness:no-cache`).

---

## 12. Unknown types default to medium, never light

- **Symptom:** a typo or a legacy task without `type` could quietly get the
  light path and skip tests.
- **Cause:** the fallback was chosen for convenience, not safety.
- **Fix:** default to `medium`. Light requires an explicit
  `chore`/`docs`/`style`/`ci`/`build` type or `verification: "light"`.
- **Number:** the canonical fallback is `DEFAULT_LEVEL = 'medium'`; the unit
  tests cover `verificationLevel({}, {}) === 'medium'` and
  `effectiveTaskType({}, {}) === null`.
- **Source:** `scripts/harness-lib.mjs`; `scripts/harness-lib.spec.mjs`
  “fallback is task.type ?? activity.type ?? medium”.

---

## 13. Full must be the final gate, not the iteration gate

- **Symptom:** the harness was run fully after every edit; developers waited
  2 min per change and then stopped running it.
- **Cause:** the only command was full.
- **Fix:** make the ladder explicit: `--fast` (3–8 s) → targeted spec → task
  level → full at activity close. Budget ≤2 full runs per activity.
- **Number:** `--fast` **4.76 s**; a light task **5.78 s**; full **98.26 s**.
  The ratio is ~20×.
- **Source:** `AGENTS.md`; `docs/harness/verification.md` §1; activity #58
  acceptance timings.

---

## 14. Project rules belong in stack references, not in the core skill

- **Symptom:** a generic harness could accidentally enforce Angular’s 60-line
  component cap or Tailwind token names in a Python repo.
- **Cause:** the canonical repo’s rules are specific to Angular/Tailwind/CLI.
- **Fix:** keep the core skill stack-agnostic. Put the exact commands and rules
  in `references/stacks/<stack>.md`. The target project’s docs are the source of
  truth; the harness enforces them.
- **Number:** the canonical repo has a 60-code-line cap for component `.ts`
  files (`docs/harness/components.md` §4) and CLI `.ts` files
  (`docs/harness/cli.md` §3). That is an Angular/CLI rule, not a universal law.
- **Source:** `docs/harness/components.md`; `docs/harness/cli.md`;
  `references/stacks/angular.md`.

---

## 15. The engine is not the contract

- **Symptom:** a working `scripts/harness.mjs` exists, but a cold agent does not
  know how to use it: no `AGENTS.md` entry point, no roles, no session tracker,
  no durable docs. The agent asks “what do I run?”, “where does this file go?”
  and “when is it done?” and invents an answer.
- **Cause:** the harness engine is only the execution layer. The **agent
  communication contract** — entry point, roles, tracker and docs — is a
  separate deliverable.
- **Fix:** ship the contract with the engine:
  - `AGENTS.md` (repo map, stack, hard rules, levels, activities, delegation,
    session close);
  - `.opencode/agent/main-orchestrator.md`, `task-executor.md`,
    `implementation-reviewer.md`;
  - `progress/current.md`, `progress/history.md`, `progress/explore/`;
  - `docs/harness/architecture.md`, `conventions.md`, `structure.md`,
    `development.md`, `testing.md`, `verification.md`, `CHECKPOINTS.md`;
  - `opencode.json` if the team uses a shared permission config.
- **Number:** the canonical repo has **1** entry point (`AGENTS.md`), **3** agent
  roles, **7** `docs/harness` documents, and **2** live/durable progress files
  plus `explore/`.
- **Source:** `AGENTS.md`, `.opencode/agent/*`, `docs/harness/*`, `progress/*`,
  `docs/harness/CHECKPOINTS.md`.

---

## 16. Git tracking is part of the harness contract

- **Symptom:** two agents edit the live session file and create conflicts;
  generated output and caches pollute `git status`; a credential or service
  account JSON is one `git add .` away from being committed.
- **Cause:** the project has a formatter/linter ignore list but no explicit
  tracked-vs-ignored policy for the harness state.
- **Fix:** document and enforce the policy:
  - **Track:** source, tests, docs, `AGENTS.md`, `.opencode/agent/*`, harness
    scripts, manifests + one lockfile, config, `progress/history.md`,
    `progress/explore/.gitkeep`, `.env.example`.
  - **Ignore:** dependencies, build output, framework/test caches, logs,
    secrets/credentials/service-account JSON, live session state.
  - Canonical decision: `/activities.json` and `/progress/current.md` are
    local-only; `progress/history.md` and `progress/explore/.gitkeep` are
    tracked.
- **Number:** the canonical `.gitignore` has exactly that session-state block;
  `git ls-files progress/` shows `history.md` and `explore/.gitkeep`, while
  `activities.json`/`current.md` are ignored.
- **Source:** `.gitignore`, `progress/`, `AGENTS.md` session-close rules;
  `assets/GIT_TRACKING.md` and `assets/gitignore.tmpl`.

---

## 17. `docs/harness` must answer the cold-start questions

- **Symptom:** docs exist but are placeholders; the agent invents a folder
  layout, skips the test policy, or cannot tell what “done” means.
- **Cause:** the harness was built without the documentation layer that defines
  the project's quality contract.
- **Fix:** maintain the seven docs: architecture (layout/layers), conventions
  (style/naming/lint), structure (file homes/process), development
  (`docs → test → implement`), testing (mutation mindset/assertions), verification
  (levels/gates/docs-tests per type), CHECKPOINTS (final state). The docs gate
  checks existence and a minimum word count; the reviewer checks truth.
- **Number:** canonical `architecture.md` is **912 words**; verification has the
  **§2.13** matrix and five checkpoint groups (C1–C5). The docs gate fails on a
  placeholder under 50 words.
- **Source:** `docs/harness/*`, `docs/harness/CHECKPOINTS.md`,
  `docs/harness/verification.md` §2.13.

---

## 18. Measure the real tree, not the plan

- **Symptom:** the original plan predicted a full feat run at 30–50 s; the
  actual baseline was **118.65 s**.
- **Cause:** estimates ignored the app suite (51 s) and the CLI suite (20 s).
- **Fix:** measure each gate before setting budgets. The honest full target is
  “under the project’s current full time, with the same gates”; only the
  light/medium targets are aggressive.
- **Number:** baseline **118.65 s**; final full **98.26–104.35 s**; target after
  parallelization **70–85 s**; app tests alone **45–51 s**.
- **Source:** activity #58 baseline and acceptance logs;
  `docs/harness/verification.md` §5.
