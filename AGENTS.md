# AGENTS.md

React's monorepo: the `react`, `react-dom`, `react-reconciler`, `scheduler`, and Server Components (`react-server-dom-*`) packages, plus their build and test tooling. `compiler/` is a separate project with its own `yarn.lock` and tooling; read `compiler/CLAUDE.md` before working there (tests are `yarn snap`, not `yarn test`).

## Setup
- Node from `.nvmrc` (20.19), yarn 1.22.22 only. Don't use npm, npx, or pnpm; `.claude/settings.json` denies them.
- `yarn install --frozen-lockfile`. Postinstall generates `scripts/flow/*/.flowconfig`.

## Architecture
- `packages/<name>/src/` is source; `packages/<name>/npm/*.js` are the published shims that pick `cjs/*.development.js` or `*.production.js`. Edit `src/`, not built output.
- `packages/react-reconciler/src/` is Fiber (hooks in `ReactFiberHooks.js`, scheduling in `ReactFiberWorkLoop.js`). Host-specific code is swapped in at build time: `ReactFiberConfig.js` throws unless a renderer provides `src/forks/ReactFiberConfig.<renderer>.js`.
- `packages/shared/` holds cross-package internals. Feature flags are `shared/ReactFeatureFlags.js` plus one fork per channel in `shared/forks/ReactFeatureFlags.*.js`.
- `scripts/rollup/bundles.js` lists every bundle; `scripts/rollup/forks.js` picks which fork file each bundle gets. Renderer names (`dom-node`, `dom-browser`, `native`, `fabric`, ...) come from `scripts/shared/inlinedHostConfigs.js`.
- `fixtures/` are standalone apps for manual testing, each with its own `yarn.lock`.

## Commands (run from the repo root)
- Test: `yarn test --silent --no-watchman <pattern>`. The default channel is **experimental**, not stable. Always pass a pattern; with none, it runs the whole suite.
- Other modes: `yarn test-stable ...`, `yarn test-www ...` and again with `yarn test-www --variant=false ...` (www flags set to `__VARIANT__` are tested both ways), `yarn test --prod ...` for production behavior.
- Lint: `yarn linc` (changed files only, what CI enforces); `yarn lint` (everything, slow).
- Format: `yarn prettier` (writes changed files); `yarn prettier-check`.
- Types: `yarn flow dom-node` (~45s; use the renderer you touched); `yarn flow-ci` checks all of them.
- Build: `yarn build react/index,react-dom/client --type=NODE_DEV` builds only the listed bundles into `build/`. Only build when asked.
- Before finishing: `yarn prettier && yarn linc && yarn flow dom-node && yarn test --silent --no-watchman <pattern>`.

## Conventions
- Tests live in `__tests__/` next to the code as `Foo-test.js`. Use `Foo-test.internal.js` when the test imports internals or reads feature flags; those are skipped when tests run against built bundles.
- Test idiom: declare `let React; let act; ...` at the top, then `jest.resetModules()` and `require(...)` everything inside `beforeEach`. Helpers come from `internal-test-utils`: `act`, `waitForAll`, `assertLog` (paired with `Scheduler.log`), `assertConsoleErrorDev`.
- Flag-dependent tests use a `// @gate enableFoo` comment above `it(...)`, or `gate(flags => flags.enableFoo)` inside the test when both branches should run.
- Source files start with the Meta MIT license header and `@flow`. Dev-only checks and warnings go inside `if (__DEV__) { ... }`.
- Shipped code can't use `for...of` or `.substring()`/`.substr()` (use `.slice()`), and `console.error`/`console.warn` messages must be string literals with `%s` args. These are custom ESLint rules.
- A new feature flag goes in `ReactFeatureFlags.js` **and every** `shared/forks/ReactFeatureFlags.*.js`, or Flow fails.

## Guardrails
- Never run `jest` directly. It only matches `scripts/jest/dont-run-jest-directly.js`, which throws.
- `yarn build` overwrites `packages/shared/ReactVersion.js` with a canary version and never restores it. Run `git checkout packages/shared/ReactVersion.js` after any build and don't commit that file. The build also needs Java (Closure Compiler).
- `scripts/error-codes/codes.json` is append-only. Never edit or renumber codes; new production error messages get codes via `yarn extract-errors` after a build.
- Don't hand-edit generated or vendored files: `build/`, `flow-typed/`, `scripts/flow/*/.flowconfig`, devtools `dist/` and `build/` dirs, `packages/react-art/npm/lib`, or the compiler's `*.expect.md` snapshots (regenerate with `yarn snap -u` in `compiler/`).
