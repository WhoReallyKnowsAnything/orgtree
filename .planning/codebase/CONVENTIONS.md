# Coding Conventions

**Analysis Date:** 2026-09-17

## Naming Patterns

**Files:**
- TS/TSX: lowercase, hyphen-free camel/kebab mix by domain — main-process modules are single words or hyphenated (`engine.ts`, `build-channel.ts`, `installer-upgrade.ts`), renderer components/tests are lowercase-no-separator (`agentstray.test.tsx`, `docketname_probe.py`)
- Python test/probe files: `snake_case` with `test_` prefix (`test_account_registry.py`) or `_probe` suffix for renderer DOM probes (`docket_layout_probe.py`)
- Test files always end `.test.mjs`, `.test.tsx`, or `test_*.py` — this suffix is load-bearing for the default test globs (see TESTING.md)

**Functions:**
- `camelCase` in TS (`attachWithRetry`, `requestManagedShutdown`, `refreshTrayEngineMenu`)
- `snake_case` in Python (`create_account`, `registry_path`)

**Variables:**
- `camelCase` in TS, `snake_case` in Python. Constants are `SCREAMING_SNAKE_CASE` (`DISRUPTIVE_ENV`, `DEV_APP_ID`, `ATTACH_RETRY_BUDGET_MS`)

**Types:**
- `PascalCase` interfaces/classes (`EngineOptions`, `RuntimeStats`, `Engine`). No `I` prefix on interfaces.

## Code Style

**Formatting:**
- No `.eslintrc*` or `.prettierrc*` file exists anywhere in the repo — style is not tool-enforced. Match the surrounding file's spacing/quoting by hand.
- Single quotes preferred in TS (`'Engine already started'`); double quotes common in Python docstrings/strings.
- `tsconfig.json` (`~/Documents/Projects/orgtree/tsconfig.json`) sets `strict: true`, target `ES2022`, module `ESNext`/`Bundler`, `jsx: react-jsx`. Treat strict-mode compliance (no implicit `any`, exhaustive null checks) as mandatory even though no linter enforces it.

**Linting:**
- None configured. `npm run typecheck` (`tsc --noEmit`) is the only automated gate on TS code quality.

**Build/Dev:**
- esbuild bundles both app code and TS test files identically — same bundler builds tests as builds the app (`apps/desktop/renderer/tests/run.mjs`). Any new renderer test that requires special transform support needs no config, since it goes through the standard vite/esbuild pipeline.

## Import Organization

**Order:**
- Node builtins first (`node:fs`, `node:path`, `node:os`), then third-party (`esbuild`), then local relative imports (`../tools/dev-build.mjs`)
- No enforced separation/blank-line convention beyond what's already in each file — follow the file you're editing.

**Path Aliases:**
- None. All imports are relative or `node:` prefixed. `packages/contracts` is imported by relative path from `apps/desktop/main/engine.ts` (e.g. `../../../packages/contracts/index`).

## Error Handling

**Patterns:**
- Fail loud with descriptive `throw new Error('...')` messages that name the exact invariant broken, not generic messages — e.g. `'descriptor trust rejected: ' + trust.detail`, `'identity root mismatch'`, `'Python runtime missing. Configure ORGTREE_V2_PYTHON for dev.'` (`apps/desktop/main/engine.ts:131-186`)
- Precondition guards at the top of async methods (`if (this.child) throw new Error('Engine already started')`) rather than deep nested handling.
- Python side uses `unittest.SkipTest(...)` with an explanation string when a check cannot run on the current platform, rather than silently passing (`tests/test_service_host.py:78-117`).
- Disruptive/dangerous test probes use an explicit opt-in gate (`tests/disruptive/gate.mjs`) rather than try/catch suppression — a gated-out probe must report as **skipped**, never as a false pass, because a silent skip that looks like "ran fine" is treated as a correctness bug in the harness itself.

## Comments

**When to Comment:**
- Module-level "why" comments are common and expected at the top of non-trivial files, explaining the design rationale and pitfalls (see `apps/desktop/renderer/tests/harness.ts`, `apps/desktop/renderer/tests/run.mjs`, `tests/disruptive/gate.mjs`) — these read like short design docs, not restatements of the code.
- Inline warnings use `⚠` markers to flag load-bearing ordering/behavior that's easy to break (e.g. "IMPORT ORDER IS LOAD-BEARING" in `harness.ts`).
- Docstrings on Python test classes describe the design doc backing the tests and what invariant each check verifies, including *why a negative case exists* (`tests/test_account_registry.py`).

## Function Design

**Size:** Main-process methods run long when they own a state machine (`Engine.attach`, `Engine.start` in `engine.ts` span 20-70 lines with named `cc=` cyclomatic complexity annotations up to `cc=28`) — this is accepted for engine lifecycle code, not a target for cleanup.

**Parameters:** Options objects with defaults are the norm for async methods with tunable timeouts (`stopForQuit(deadlineMs = QUIT_STOP_BUDGET_MS)`, `attachWithRetry(options, deadlineMs = ATTACH_RETRY_BUDGET_MS, intervalMs = 1000)`).

**Return Values:** Union string literals for state results instead of booleans/enums where there are 3+ outcomes (`Promise<'stopped' | 'forced' | 'unverified'>`, `Promise<'attached' | 'spawned' | 'failed'>`) — prefer this pattern over booleans when adding new tri-state (or more) outcomes.

## Module Design

**Exports:** Named exports throughout (`export class Engine`, `export function mailhubStats`), no default exports observed in main-process code.

**Barrel Files:** None found — no `index.ts` re-export barrels; consumers import directly from the source file.

---

*Convention analysis: 2026-09-17*
