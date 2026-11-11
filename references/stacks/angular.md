# Angular stack reference

Derived from the canonical `tuneles` repository (Angular 21, standalone
components, zoneless, strict templates, Tailwind 4 CSS-first, Vitest/jsdom,
pnpm). Use this table when the target project is Angular; otherwise use
`node.md` or `python.md`.

## Commands by gate

| Gate        | Command                                                                             |
| ----------- | ----------------------------------------------------------------------------------- |
| format      | `pnpm exec prettier --check .`                                                      |
| lint        | `pnpm exec eslint . --max-warnings 0`                                               |
| type (app)  | `pnpm exec tsc --noEmit -p tsconfig.app.json`                                       |
| type (spec) | `pnpm exec tsc --noEmit -p tsconfig.spec.json`                                      |
| type (CLI)  | `pnpm exec tsc --noEmit -p tunelesctl/tsconfig.json` (only if a CLI package exists) |
| build       | `pnpm run build` (`ng build`, production)                                           |
| test full   | `pnpm run test:ci` (`ng test --watch=false`) + `pnpm run test:tunelesctl`           |
| test target | `pnpm exec ng test --watch=false --include=<spec>` + `pnpm exec vitest run <spec>`  |
| contract    | inspect `dist/tuneles/**/*.css` for `@theme` tokens, fonts and radii                |
| docsgen     | `pnpm exec compodoc -p tsconfig.app.json -d docs/api`                                |
| structure   | `src/app` tree rules + CLI package rules                                            |
| docs        | `docs/planning/*` and `docs/harness/*` are not placeholders                          |

## Structure rules (canonical)

- `src/app/` top level: `views/`, `shared/`, `core/`, `models/` plus the root
  app files. No other top-level folder without a docs update.
- `views/<vista>/`: `<vista>.ts/.html/.spec.ts` and optional `.css`; no
  `selector` (entered by route).
- `shared/<componente>/`: exactly four files
  (`<componente>.ts/.html/.css/.spec.ts`), selector `app-<componente>`.
- `core/`: flat `*.service.ts` + `*.service.spec.ts`, no subfolders.
- `models/`: flat pure `*.ts`, no specs.
- No `@NgModule`, no inline `template:`/`styles:`.
- ≤60 code lines per component `.ts` (blank and comment-only lines excluded).
- The CLI package (`tunelesctl/`) has its own rules: one kebab-case command
  folder with a colocated spec, flat `engine/`/`storage/`/`core/` with
  colocated specs, `models/` without specs, pure engine (no I/O imports), no
  cross-imports with `src/`, ≤60 code lines per `.ts`.

These are project rules, not universal Angular rules. Copy them only if the
target project already documents them.

## Contract example (Tailwind 4)

The canonical contract gate reads the emitted CSS and asserts:

- `--color-surface: #131313`
- `--color-page: #282828`
- `--radius-sm: 8px` and `--radius-full: 9999px`
- the `Work Sans` default font

Run it **after** `ng build`. If the build is skipped, skip the contract check
and say so; never read a stale `dist/`.

## Steps in this stack

- `--fast`: changed Prettier/ESLint + `src/app`/CLI structure.
- Iteration: + affected `tsc` + targeted app/CLI specs.
- `--activity`: + Compodoc + `touches` check.
- `--story`: all gates + `ng build` + both suites + Compodoc + user approval.
- No reviewer role: the user approves the story plan and the finished story.

Docblocks are enforced by ESLint (`eslint-plugin-jsdoc` for exported symbols);
Compodoc publishes them. Generated docs are ignored by git.

## Measured budget (canonical repo)

| Run                                               |           Time |
| ------------------------------------------------- | -------------: |
| `harness:fast` (no changes)                       |         0.21 s |
| `harness:fast` (42 changed files)                 |         4.76 s |
| iteration with 42 files, 3 tsc, 14 CLI specs      |        13.99 s |
| full `--story`                                    |        91.40 s |
| full default                                      | 98.26–104.35 s |

Targeted app leaf spec: ~5.5–8.3 s. The `src/app/app.spec.ts` token contract is
~23 s because it imports the whole app; split it before using it as a
medium-level benchmark.

## Pitfalls

- Do not run `ng test` without `--watch=false` in CI.
- `--include` accepts multiple specs; repeat the flag per spec.
- The CLI test suite is `vitest run tunelesctl/`; the app suite is `ng test`.
  They are separate runners; run them in parallel on full.
- Do not lint generated `dist/`, `coverage/`, `.angular/`.
- A `styles.css` change has no sibling spec; map it to the token contract spec
  or accept the package fallback.
