# Node / TypeScript stack reference

Covers Node.js, TypeScript, React, Vue, Svelte, Next.js, Nuxt, Vite and
monorepos. Detect the package manager from the lockfile and use project-owned
scripts first.

## Detection

| Signal                                           | Meaning                                  |
| ------------------------------------------------ | ---------------------------------------- |
| `pnpm-lock.yaml`                                 | pnpm                                     |
| `package-lock.json`                              | npm                                      |
| `yarn.lock`                                      | Yarn                                     |
| `bun.lockb`                                      | Bun                                      |
| `tsconfig.json`                                  | TypeScript; check `noEmit`, `references` |
| `vite.config.*`                                  | Vite (React/Vue/Svelte)                  |
| `next.config.*`                                  | Next.js                                  |
| `nuxt.config.*`                                  | Nuxt                                     |
| `svelte.config.*`                                | SvelteKit                                |
| `vitest.config.*`                                | Vitest                                   |
| `jest.config.*`                                  | Jest                                     |
| `biome.json`                                     | Biome formatter/linter                   |
| `eslint.config.*`                                | ESLint                                   |
| `turbo.json` / `nx.json` / `pnpm-workspace.yaml` | Monorepo                                 |

## Commands by gate

| Gate        | npm                             | pnpm                                  | Notes                                                       |
| ----------- | ------------------------------- | ------------------------------------- | ----------------------------------------------------------- |
| format      | `npx prettier --check .`        | `pnpm exec prettier --check .`        | Or Biome: `npx biome check .`                               |
| lint        | `npx eslint . --max-warnings 0` | `pnpm exec eslint . --max-warnings 0` | Or `npx biome lint .`                                       |
| type        | `npx tsc --noEmit`              | `pnpm exec tsc --noEmit`              | Per package with `-p`                                       |
| build       | `npm run build`                 | `pnpm run build`                      | Vite/Next/tsup/rollup                                       |
| test full   | `npm test`                      | `pnpm test`                           | Vitest/Jest/node:test                                       |
| test target | `npx vitest run <spec>`         | `pnpm exec vitest run <spec>`         | Jest: `npx jest <spec>`; node:test: `node --test <spec>`    |
| contract    | project script                  | project script                        | `dist` exports, bundle entry, CSS tokens, package `exports` |
| structure   | config rules                    | config rules                          | dirs, forbidden patterns, max lines                         |
| docs        | required docs                   | required docs                         | README/architecture not placeholders                        |

For a pnpm project, use `pnpm exec` for binaries and `pnpm run` for scripts.
Do not mix package managers. One lockfile per repo.

## Monorepo affected-package type check

```js
// CONFIG.gates.type.affected
affected: (files) => {
  const commands = [];
  if (files.some((f) => f.startsWith("apps/web/"))) {
    commands.push([
      "pnpm",
      "exec",
      "tsc",
      "--noEmit",
      "-p",
      "apps/web/tsconfig.json",
    ]);
  }
  if (files.some((f) => f.startsWith("packages/core/"))) {
    commands.push([
      "pnpm",
      "exec",
      "tsc",
      "--noEmit",
      "-p",
      "packages/core/tsconfig.json",
    ]);
  }
  return commands.length ? commands : null;
};
```

Map package roots explicitly. Do not guess “the nearest tsconfig” unless the
project already has a convention for it.

## Targeted spec discovery

```js
// CONFIG.specs.candidates
if (/\.(?:[cm]?[jt]sx?)$/.test(file)) {
  const base = file.replace(/\.(?:[cm]?[jt]sx?)$/, "");
  return [
    ".spec.ts",
    ".spec.tsx",
    ".spec.js",
    ".spec.jsx",
    ".test.ts",
    ".test.tsx",
    ".test.js",
    ".test.jsx",
  ].map((suffix) => `${base}${suffix}`);
}
```

If a changed source file has no sibling spec, fall back to the package test
command and warn. If the tree is clean, try `git diff --name-only HEAD~1`
before falling back.

## Structure examples

- Required: `src/` or `apps/`, `packages/`.
- Forbidden: `@nestjs` module patterns, `require()` in ESM, inline SQL in
  components, `any` in public API? Only encode rules the project documents.
- Max lines: optional; use the project’s own limit, not Angular’s 60-line cap.

## Contract examples

- `package.json` `exports`/`types` resolve.
- `dist/index.js` and `dist/index.d.ts` exist.
- Bundle entry point exists and is non-empty.
- Emitted CSS contains the design tokens.
- JSON schema validates an example document.

Contract checks run after build. If build is skipped, skip contract.

## Budgets

- Light: changed format/lint + structure/docs + self-test — seconds.
- Medium: + affected type + targeted specs — usually 10–40 s.
- Full: all + build + full tests — often 60–180 s; measure before setting a
  budget. The canonical Node/TS full is ~98 s with two suites.

## Pitfalls

- Do not run full ESLint on `node_modules/`, `dist/`, `.next/`, `.turbo/`.
- Do not introduce a second test runner; use the one in the repo.
- Do not use `npx` in CI if the dependency is not installed locally; it may
  fetch a different version. Prefer `pnpm exec`/`npm exec`.
- Do not run `tsc` on a project with `noEmit: false` and expect no output; use
  `--noEmit` explicitly for the gate.
