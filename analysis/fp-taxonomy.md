# False Positive Taxonomy

## Two Failure Modes

1. **Analysis-level FPs** -- symbols the Supermodel API incorrectly flags as dead (parser/graph problem)
2. **Agent-level FPs** -- agent dumps entire candidate list without filtering ("analysis dump" pattern)

Mode 2 is larger numerically (Feb 20: 19,087 total FPs from analysis dumps). Mode 1 is the root cause -- if analysis were precise, dumping it would be fine.

## 7 Categories of Analysis-Level FPs

### P0: Import Resolution / Root File Explosion (~51% of FPs)

**The dominant source.** When import resolution fails to trace file consumption (Express `app.use(router)`, dynamic `require()`, framework wiring), files become "root files" with no importers. All exports get flagged.

- Tyr: **145 root files** (should be ~3-5), producing ~396 of 775 candidates
- Affects: route handlers, middleware, controllers, services wired via framework patterns

**Examples from Tyr:**
- `initializePassport` -- wired via Express middleware
- `configureGoogleStrategy` -- registered in passport config
- `listCurators`, `startSync` -- controller functions registered via Express router

**Status (Mar 9):** Partially fixed. Barrel re-exports now traced correctly. Framework-level wiring (Express router mounting, NestJS modules, dynamic requires) still missing.

### P1: Type Re-exports Not Tracked (~15% of FPs)

`export type { X } from './module'` -- the type-only re-export syntax is not recognized. Symbols re-exported this way appear to have no consumers.

- jslpsolver scream test: 9 of 28 FPs
- Affects every monorepo with barrel files

**Status:** Not fixed.

### P2: `export default` Not Detected (~5% of FPs)

Functions exported via `export default functionName` are not recognized as exports.

- jslpsolver scream test: 2 of 28 FPs
- Universally present in React/Next.js codebases

**Status:** Not fixed.

### P3: JSX Usage Not Recognized as Calls (~20% in React repos)

React components used as `<Footer />` don't generate CALLS edges. Cross-file JSX usage is missed.

- Tyr: 155 React components flagged as dead
- Affects: Maskbook, Logto, Latitude, Cal.com, antiwork/Helper

**Status:** Partially fixed. Phase 7.5b same-file rescue catches `<Foo />` in same file. Cross-file JSX still misses.

### P4: Test/Script/Config Imports Not Scanned (11/28 scream test FPs)

Files imported only from test files, build scripts, or config files appear to have "no importers."

- jslpsolver scream test: 11 of 28 FPs (largest single category)
- Affects: `vitest.config.ts`, `rollup.config.mjs`, `jest.config.ts`, test helpers

**Status:** Not fixed. **Highest-ROI unfixed item.**

### P5: Python `isExported` Broken (disables entire pipeline)

Python parser only checks `__all__` for exports. Most Python projects don't use `__all__`, so `isExported=false` for everything, disabling symbol-level analysis entirely.

- Python benchmark: 113 FPs out of 212 flagged
- Route handlers, Pydantic models, middleware classes all incorrectly flagged

**Fix:** Mark module-level non-underscore-prefixed symbols as exported when no `__all__` exists.

**Status:** Not fixed.

### P6: Type/Interface Structural Typing (~21% in TS repos)

TypeScript interfaces satisfied via structural typing have no explicit references. `import type { X }` may not be tracked. Generic type parameters hard to resolve.

- Tyr: 162 types/interfaces flagged (21% of all candidates)
- Grafana SUMMARY: "Both agents missed ~50% of GT, mostly interface declarations"

**Status:** Hard to fix due to structural typing semantics.

## Aggregate FP Counts (Feb 20 Benchmark)

| Task | FPs | GT | FP:GT Ratio | Likely Dominant Category |
|------|-----|-----|-------------|------------------------|
| maskbook_pr12361 | 5,880 | 22 | 267:1 | Re-exports, barrel files |
| gemini_cli_pr18681 | 3,501 | 6 | 583:1 | CLI entry points |
| logto_pr7631 | 3,059 | 8 | 382:1 | Auth framework, React |
| directus_pr26311 | 2,450 | 15 | 163:1 | Monorepo, SDK re-exports |
| latitude_pr2300 | 1,500 | 5 | 300:1 | Monorepo, component re-exports |
| mimir_pr3613 | 1,124 | 9 | 125:1 | Framework entry points |
| podman_pr16084 | 778 | 2 | 389:1 | TS interface implementations |
| tyr_pr258 | 537 | 22 | 24:1 | Route handlers, broken imports |
