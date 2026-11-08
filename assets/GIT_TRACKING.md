# Git tracking contract

> What belongs in git and what must stay local. This is part of the harness
> contract: a clean repository is a gate, not a preference.

Read this before creating `.gitignore`. Use `assets/gitignore.tmpl` as the
starting point and `assets/ignores.txt` for formatter/linter/structure walks.

## Track

| Path / kind                                                                                                   | Why                                                           |
| ------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| Source code (`src/`, `app/`, `lib/`, `packages/`, etc.)                                                       | The product                                                   |
| Tests and fixtures                                                                                            | The contract; they must be reviewable                         |
| `AGENTS.md`                                                                                                   | The agent entry point                                         |
| `.opencode/agent/*.md`                                                                                        | Orchestrator, executor and reviewer roles                     |
| `opencode.json` (if used)                                                                                     | Shared safety/permission config                               |
| `scripts/harness.mjs`, `scripts/harness-lib.mjs`, `scripts/harness-lib.spec.mjs`, `scripts/build_harness.mjs` | The executable harness                                        |
| `docs/` (including `docs/harness/*`)                                                                          | Architecture, conventions, testing, verification, checkpoints |
| Package manifests and **one** lockfile                                                                        | Reproducible installs                                         |
| Formatter/linter/type/test config                                                                             | The gates depend on them                                      |
| `.gitignore`, `.prettierignore`, `.eslintignore`                                                              | The ignore policy                                             |
| `progress/history.md`                                                                                         | Durable session record                                        |
| `progress/explore/.gitkeep`                                                                                   | Keeps the research folder in the tree                         |
| `.env.example` (no secrets)                                                                                   | Documents required configuration                              |

## Do not track (local only)

| Path / kind                                                                                | Why                                                                                                                                                     |
| ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `activities.json`                                                                          | Session state; canonical repo ignores it to avoid merge conflicts. Track it only if the team wants shared activity state, and accept the conflict cost. |
| `progress/current.md`                                                                      | Live session state; the durable summary lives in `progress/history.md`.                                                                                 |
| `node_modules/`, `.pnpm-store/`, `.venv/`, `vendor/`                                       | Dependencies are reproducible from the lockfile                                                                                                         |
| `dist/`, `build/`, `out/`, `target/`, `bin/`, `obj/`                                       | Generated build output                                                                                                                                  |
| `.next/`, `.nuxt/`, `.svelte-kit/`, `.angular/`, `.turbo/`, `.nx/`, `.cache/`, `coverage/` | Framework/test caches                                                                                                                                   |
| `*.log`, `*.tmp`, `.DS_Store`                                                              | Local noise                                                                                                                                             |
| `.env`, `.env.*` (except `.env.example`), keys, credentials, service-account JSON          | Secrets must never enter git                                                                                                                            |
| Project-local state (`.tuneles/`, `.data/`, uploads)                                       | Runtime state; add the project's own path here                                                                                                          |
| `progress/explore/*` research notes                                                        | Optional; commit only if the team wants a durable research log. Keep `.gitkeep`.                                                                        |

## The canonical decision

The canonical `tuneles` repo deliberately ignores:

```gitignore
# Session state (local only)
/activities.json
/progress/current.md
```

and tracks:

```text
progress/history.md
progress/explore/.gitkeep
```

Why: two agents or two branches editing the live session file create conflicts
that have nothing to do with the product. The durable record is
`progress/history.md`; the live state is local. If a team wants shared activity
state, that is a conscious trade-off: track `activities.json`, keep the file
small, and expect conflicts.

## Verify the policy

```bash
git status --short
git status --ignored --short | head -40
git check-ignore -v activities.json progress/current.md
git ls-files progress/
```

Expected: `activities.json` and `progress/current.md` are ignored; `progress/history.md`
and `progress/explore/.gitkeep` are tracked; no secrets appear in `git status`.

## Rules

1. Never commit a secret, token, private key or credential file.
2. Never commit generated output that the build can recreate.
3. Commit one lockfile; do not commit multiple package-manager locks.
4. Keep `progress/history.md` append-only and reviewed.
5. If you change the tracking policy, update this file and `.gitignore` in the
   same change.
