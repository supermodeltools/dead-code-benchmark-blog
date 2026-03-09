# Parser Improvement Plan

## Prioritized Fixes

### P0: Import Resolution (Partially Fixed)

**Impact:** ~51% of all FPs
**Status:** Barrel re-exports fixed (Mar 6). Framework wiring still missing.

**What's done:**
- Barrel re-export filtering with `isReExport` flag propagation
- Step 2b fixpoint loop for cascading dead imports
- Cross-package import resolution for monorepos (Phase 4d)
- Dynamic `import()`/`require()` detection (Phase 2c)

**Still needed:**
- Express/Koa/Hapi router mounting (`app.use('/api', router)`)
- NestJS module registration patterns
- Framework convention scanning (Next.js `app/`, `pages/` directories)
- Detect entry points from `package.json` `main`/`bin`/`exports` fields

### P1: Type Re-exports (`export type { X } from`)

**Impact:** ~15% of FPs, 9/28 scream test FPs
**Status:** Not fixed

**Implementation:** Extend import scanner to recognize `export type { X } from './module'` as creating a dependency edge. The `type` keyword before the export braces is the distinguishing syntax.

**Files to change:**
- `src/data-plane/src/parsers/tree-sitter.ts` -- extract type re-exports
- `src/data-plane/src/services/codegraph.service.ts` -- resolve type import edges

### P2: `export default X` Detection

**Impact:** ~5% of FPs, 2/28 scream test FPs
**Status:** Not fixed

**Implementation:** Detect `export default <identifier>` and `export default function/class` patterns. Map the default export to the underlying symbol for call graph resolution.

**Files to change:**
- `src/data-plane/src/parsers/tree-sitter.ts` -- `isNodeExported()` function

### P3: Cross-File JSX Usage

**Impact:** ~20% in React repos (155 FPs in Tyr alone)
**Status:** Partially fixed (same-file only)

**Implementation:** When parsing JSX files, treat `<ComponentName` as a CALLS edge to `ComponentName`. Requires resolving the import to know which file/symbol is being called.

**Complexity:** Medium-high. JSX component resolution requires:
1. Parse JSX elements as potential call sites
2. Resolve component name to import source
3. Create CALLS edge from JSX usage to imported component

### P4: Test/Script/Config Import Scanning

**Impact:** 11/28 scream test FPs. **Highest-ROI unfixed item.**
**Status:** Not fixed

**Implementation:** Include test files (`*.test.ts`, `*.spec.ts`, `__tests__/`), build scripts (`rollup.config.*`, `vite.config.*`, `webpack.config.*`), and config files (`jest.config.*`, `vitest.config.*`) in the import scan.

Mark symbols imported by these files as "alive" (or at minimum, lower-confidence dead code candidates).

**Risk:** Low. Test files importing a symbol is strong evidence it's alive.

**Files to change:**
- `src/data-plane/src/services/job-worker.service.ts` -- Phase 2 import extraction (include test/config files)
- Possibly: add a `confidence` field that's lower for symbols only imported by tests

### P5: Python `isExported` Fix

**Impact:** Disables entire Python dead code pipeline
**Status:** Not fixed

**Implementation:** In `isNodeExported()` for Python:
- If `__all__` exists, use it (current behavior)
- If no `__all__`, mark module-level symbols as exported UNLESS they start with `_` (Python convention for private)

**Files to change:**
- `src/data-plane/src/parsers/tree-sitter.ts:1396-1438` -- `isNodeExported()` Python branch

### P6: Type/Interface Structural Typing

**Impact:** ~21% in TS repos
**Status:** Hard to fix

**Why it's hard:** TypeScript's structural type system means an interface can be satisfied without explicit reference. `interface Foo { bar: string }` can be satisfied by any object with a `bar: string` property. No import or reference needed.

**Partial mitigations:**
- Track `import type` statements as references
- Track type annotations in function signatures
- Track `as Type` assertions
- Accept this will always have some FPs and rank these candidates lower

## Recommended Fix Order

1. **P4** (test/script imports) -- highest ROI, lowest risk, easiest implementation
2. **P1** (type re-exports) -- 15% of FPs, straightforward parser change
3. **P2** (export default) -- small fix, universally beneficial
4. **P5** (Python isExported) -- unlocks entire Python pipeline
5. **P3** (cross-file JSX) -- high impact but medium-high complexity
6. **P0 remainder** (framework wiring) -- complex, framework-specific heuristics needed
7. **P6** (structural typing) -- diminishing returns, accept some FPs
