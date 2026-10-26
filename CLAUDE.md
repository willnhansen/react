# React

Monorepo for the React runtime (`react`, `react-dom`, reconciler, Fizz/Flight servers, scheduler, DevTools). `compiler/` is a separate yarn project with its own `.claude/` (Rust port: `compiler/docs/rust-port/`, branch `rust-research`).

## Architecture
- `packages/<pkg>/index*.js`: public entry points, forked per release channel (`.experimental`, `.stable`, `.fb`). `packages/<pkg>/src/`: the implementation (ESM + Flow).
- `react-reconciler/src/ReactFiberConfig.js` and the other `*Config.js` files are throwing stubs. The real host configs come from `src/forks/` (e.g. `react-dom-bindings/src/client/ReactFiberConfigDOM.js`).
- `scripts/`: the jest/rollup/flow/eslint tooling behind every `yarn` command.

## Commands
- Test: `yarn test --silent --no-watchman <pattern>`. Never run `jest` directly (it throws) or without a pattern (whole suite).
- `--silent` prints one dot per *file*. To prove a new test ran, drop `--silent`, add `-t "test name"`, and look for `1 passed`.
- Default channel is experimental + development. Also check `-r=stable`, `--prod`, and `-r=www-modern --variant=false` (www defaults to `--variant=true`). DevTools tests need `--project devtools`.
- Format + lint changed files: `yarn prettier && yarn linc` (diffs against local `main`).
- Types: `yarn flow dom-node`. Renderer names are in `scripts/shared/inlinedHostConfigs.js`; there is no `native`.
- Full check before finishing: `/verify`.
- Build: tests don't need one, so only build when asked: `yarn build react/index,react-dom/index --type=NODE_DEV` (output in `build/`).

## Conventions
- Packages import each other by package name (`shared/...`, `react-reconciler/src/...`).
- Source style rules (no `?.`, no `for...of`, `console.error` only inside `if (__DEV__)`, error codes) are enforced by `yarn linc`, so run it rather than guessing. For feature flags use `/feature-flags`; for new error messages / `codes.json` use `/extract-errors`.
- Name tests `*-test.js` and put them directly in `__tests__/` (helpers go in a subfolder). Use `*-test.internal.js` only if the test mutates `ReactFeatureFlags`.
- Test idiom from `internal-test-utils`: render via `ReactDOMClient.createRoot` or `ReactNoop` inside `await act(...)`. Record ordering and effects with `Scheduler.log('x')`, then check them with `assertLog([...])`.
- Any unasserted `console.error`/`warn` fails the test. A stray `console.log` fails only in CI (reproduce with `CI=true yarn test ...`). Assert with `assertConsoleErrorDev` (see `packages/react-dom/src/__tests__/ReactDOMComponent-test.js:198`).
- `act` throws if `Scheduler.log` entries haven't been checked with `assertLog`.
- `// @gate flag` does **not** skip: the test runs and must *fail* when the gate is false. If your change makes a gated test pass, update the gate.

## Guardrails
- yarn 1 only (no npm/pnpm/npx).
- Don't hand-edit `build/`, `flow-typed/`, or `ReactDOMFizzInstructionSetInlineCodeStrings.js` (regenerate with `yarn generate-inline-fizz-runtime`).
- `packages/<pkg>/npm/` holds hand-written ES5 CJS shims, not source: edit `src/`. `fixtures/` are manual demo apps, not tests.
