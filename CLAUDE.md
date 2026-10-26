# React

Monorepo for the React runtime (`react`, `react-dom`, reconciler, Fizz/Flight servers, scheduler, DevTools). Source is `packages/<pkg>/src` (ESM + Flow), imported across packages by package name (`shared/...`, `react-reconciler/src/...`). `compiler/` is a separate yarn project with its own `.claude/` (Rust port: `compiler/docs/rust-port/`, branch `rust-research`).

## Commands (yarn 1 only)
- Test: `yarn test --silent --no-watchman <pattern>`. Never run `jest` directly (it throws) or without a pattern (whole suite).
- `--silent` prints one dot per *file*. To prove a new test ran, drop `--silent`, add `-t "test name"`, and look for `1 passed`.
- Default channel is experimental + development. Also check `-r=stable`, `--prod`, and `-r=www-modern --variant=false` (www defaults to `--variant=true`). DevTools tests need `--project devtools`.
- Format + lint changed files: `yarn prettier && yarn linc`. linc diffs against local `main` and enforces the source rules (no `?.`, no `for...of`, DEV-only `console.error`, error codes), so run it rather than guessing.
- Types: `yarn flow dom-node`. Renderer names are in `scripts/shared/inlinedHostConfigs.js`; there is no `native`.
- Full check before finishing: `/verify`. Tests don't need a build; don't run `yarn build` unless asked.

## Test runtime traps (not visible in code, not caught by lint)
- Any unasserted `console.error`/`warn` fails the test; stray `console.log` fails only in CI. Assert with `assertConsoleErrorDev` (see `packages/react-dom/src/__tests__/ReactDOMComponent-test.js:198`).
- `act` throws if `Scheduler.log` entries haven't been checked with `assertLog`.
- `// @gate flag` does **not** skip: the test runs and must *fail* when the gate is false. If your change makes a gated test pass, update the gate.
- Only files directly in `__tests__/` are collected; helpers go in a subfolder.

## Pointers (read when relevant)
- Feature flags: `/feature-flags`. Error messages / `codes.json`: `/extract-errors`.
- `react-reconciler/src/ReactFiberConfig.js` (and other `*Config.js`) are stubs. The real host configs come from `src/forks/` (e.g. `react-dom-bindings/src/client/ReactFiberConfigDOM.js`); a new export goes in every renderer's config.
- Don't hand-edit `build/`, `flow-typed/`, or `ReactDOMFizzInstructionSetInlineCodeStrings.js` (regenerate with `yarn generate-inline-fizz-runtime`).
