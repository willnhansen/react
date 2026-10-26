# React

Monorepo for the React runtime (`react`, `react-dom`, the reconciler, Fizz/Flight servers, scheduler, DevTools). Everything outside `compiler/` is covered here; `compiler/` is a separate yarn project with its own `.claude/` (Rust port: plans in `compiler/docs/rust-port/`, code in `compiler/crates/`, branch `rust-research`).

## Layout
- `packages/<pkg>/src/` — source (ESM + Flow). `packages/<pkg>/index*.js` — entry points, forked per channel (`index.experimental.js`, `index.stable.js`, `index.fb.js`): a new public export goes in each relevant variant.
- `packages/<pkg>/npm/` — hand-written **ES5 CommonJS** shims that pick `cjs/*.development.js` vs `.production.js`. No modern syntax here.
- `react-reconciler/src/ReactFiberConfig.js` (and `react-server/src/ReactFizzConfig.js`, etc.) are throwing stubs. At build/test time they resolve to `src/forks/<Name>.<renderer>.js`, which usually re-exports the real host config (e.g. `react-dom-bindings/src/client/ReactFiberConfigDOM.js`). Edit the real host config, and add new exports to **every** renderer's config (DOM, Noop, Fabric, ART, TestHost) or a `ReactFiberConfigWithNo*.js` shim. Renderers are listed in `scripts/shared/inlinedHostConfigs.js`.
- Cross-package imports use package names (`shared/ReactFeatureFlags`, `react-reconciler/src/...`), never `../../other-pkg`.
- `scripts/` — jest, rollup, flow, eslint-rules, error-codes. `fixtures/` — manual test apps, not tests.

## Commands (yarn 1 only; Node per `.nvmrc`)
- Test one file/pattern: `yarn test --silent --no-watchman ReactHooks-test`. Never run `jest` directly (it throws), and never run without a pattern (whole suite). `--silent` prints one dot per *file*, so to prove a new test ran (not skipped by `@gate`), drop `--silent` and add `-t "test name"`; look for `1 passed`.
- Default channel is **experimental + development**. CI also runs: `yarn test -r=stable`, `yarn test --prod`, `yarn test -r=www-modern --variant=false` (www/xplat default to `--variant=true`, so test both). DevTools tests only run with `yarn test --project devtools`.
- Format/lint changed files: `yarn prettier` then `yarn linc` (diffs against local `main`, so keep a `main` branch).
- Types: `yarn flow dom-node` (~1 min first run). Valid renderers: `dom-node`, `dom-browser`, `fabric`, `test`, `art`, `noop`, ... (there is no `native`).
- Builds aren't needed for tests. `yarn build react/index,react-dom/index --type=NODE_DEV` only when asked, or before `yarn extract-errors` (it reads `build/`).
- Before finishing: `/verify` (prettier → linc → flow → test source + www).

## Source conventions (lint-enforced; most are relaxed for tests/scripts/devtools)
- Files start with the Meta MIT license header and `@flow`.
- No optional chaining (`?.`) and no `for...of` in `packages/*/src`.
- `console.error`/`console.warn` must sit directly inside `if (__DEV__) {`, and the first argument must be a string literal. There is no `warning()` module.
- Errors: `throw new Error('literal ' + x)`, never `invariant`. Every production error message must exist in `scripts/error-codes/codes.json` (`react-internal/prod-error-codes`). That file is append-only: never edit or renumber codes. Run `/extract-errors` for new messages.
- Runtime code gates on named flags from `shared/ReactFeatureFlags`, not on `__EXPERIMENTAL__`/`__PROFILE__` directly. A new flag goes in **every** file in `packages/shared/forks/ReactFeatureFlags.*.js` (see the `feature-flags` skill).

## Test conventions
- Tests live directly in `__tests__/` (anything there is collected, so helpers go in a subfolder). `*-test.js` is the default. Use `*-test.internal.js` only if the test mutates `ReactFeatureFlags` or imports internals, since internal tests are skipped against builds.
- Pattern: `let React, ReactDOMClient, act, assertLog;` at top; in `beforeEach`, `jest.resetModules()` then `require(...)`. Helpers come from `internal-test-utils` (`act`, `waitForAll`, `waitFor`, `assertLog`, `assertConsoleErrorDev`). `Scheduler` is the mock: log with `Scheduler.log('x')`, then `assertLog(['x'])`.
- **Any unasserted `console.error`/`warn` fails the test.** Stray `console.log` fails only in CI. Assert with `assertConsoleErrorDev(['msg\n    in Comp (at **)'])`. `toErrorDev`/`toWarnDev`, `ReactTestUtils.act`, and global `spyOn` are gone (use `spyOnDev`/`spyOnProd`).
- `act` throws if the Scheduler log has unasserted entries. Fake timers are global.
- `// @gate flagName` does **not** skip: the test still runs and must *fail* when the gate is false. If your change makes a gated test pass, update the gate. Use inline `gate(flags => ...)` when both paths should run.
- New tests use `ReactDOMClient.createRoot` or `ReactNoop`. `ReactDOM.render` exists only in legacy tests gated `// @gate !disableLegacyMode`.

## Don't hand-edit
- `build/`, `flow-typed/`, devtools `dist/`.
- `ReactDOMFizzInstructionSetInlineCodeStrings.js`: edit the sibling `ReactDOMFizzInline*.js` instead, then run `yarn generate-inline-fizz-runtime` (CI diffs it).
- Versions: `ReactVersions.js` and `packages/shared/ReactVersion.js` must change in lockstep; don't bump versions unless asked.
