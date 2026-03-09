# Dead Code Benchmark Blog

This repo contains everything for our dead code detection benchmark: raw data from previous runs, the blog post draft, analysis of false positive patterns, and instructions for running new benchmarks.

## Repository Structure

```
dead-code-benchmark-blog/
  README.md                           # This file (planning, methodology, run instructions)
  blog/
    2026-03-09-dead-code-graphs-ai-agents.md   # Blog post draft
  runs/
    feb-06-initial/                   # First antiwork/helper benchmark
    feb-17-18-full-suite/             # 16-run benchmark (blogpost-deadcode.zip)
    feb-20-with-synthetic/            # 10-task MCP vs baseline (fri-feb-20.zip)
    feb-23-post-fixes/                # 16-task MCP-only post-fixes (sun-feb-23.zip)
    mar-09-post-parser-improvements/  # NEW: Run after barrel re-export + 7 new phases
  analysis/
    fp-taxonomy.md                    # False positive root cause taxonomy
    parser-improvement-plan.md        # Prioritized improvement roadmap
    benchmark-improvement-system.md   # Continuous improvement loop design
```

---

## Previous Benchmark Results Summary

### Timeline

| Date | Run | Tasks | Key Finding |
|------|-----|-------|-------------|
| Feb 6 | Initial | antiwork/helper + synthetic | **91.7% recall, 100% precision** on real codebase (resolved!) |
| Feb 9 | Replication | Same as above | Identical results -- confirmed reproducibility |
| Feb 17-18 | Full suite | 16 runs (synthetic + 12 real PRs) | MCP wins 9/16, 0 resolved on large repos |
| Feb 20 | MCP vs baseline | 10 real PRs | **Baseline scored 0% on all 10 tasks.** MCP: 75-100% recall on 6/10 |
| Feb 23 | Post-fixes (MCP only) | 16 PRs | Directus 80% recall, some improvement |
| **Mar 9** | **Post parser improvements** | **TBD** | **Run this now -- see instructions below** |

### Best MCP Recall Per Real-World Task (across all runs)

| Task | Best Recall | Best Precision | Run |
|------|------------|----------------|-----|
| directus_pr26311 | **100%** | 0.6% | Feb 20 |
| podman_pr16084 | **100%** | 0.3% | Feb 20 |
| latitude_pr2300 | **100%** | 0.3% | Feb 20 |
| tyr_pr258 | **95.5%** | 3.8% | Feb 20 |
| antiwork/helper | **91.7%** | **100%** | Feb 6 (RESOLVED) |
| mimir_pr3613 | **77.8%** | 0.6% | Feb 20 |
| otel_js_pr5444 | **75.0%** | 1.3% | Feb 20 |
| maskbook_pr12361 | **63.6%** | 0.2% | Feb 20 |

### Parser Improvements Since Last Benchmark (Feb 20-23)

These changes landed but have NOT been measured in a full benchmark yet:

1. **Barrel re-export filtering** (Mar 6, #444/#469)
   - Internal testing: recall 59.0% -> 85.2% across 5 repos
   - tyr: 36.4% -> 86.4%, mimir: 77.8% -> 88.9%, directus: 80% -> 93.3%, latitude: 80% -> 100%
   - Propagates `isReExport` flag, excludes re-export-only consumers from import counts
   - Step 2b fixpoint loop: cascading dead import detection

2. **7 new pipeline phases** (Feb 26-Mar 6, #469)
   - Phase 2b: Vue SFC import extraction
   - Phase 2c: Dynamic `import()`/`require()` filesystem loading detection
   - Phase 4c: Library package detection (types/, module/, packages/)
   - Phase 4d: Cross-package import resolution for monorepos
   - Phase 6b: Class-to-method reachability propagation
   - Phase 6d: Exported object literal property reachability
   - Directus: 2,808 candidates -> 2,178 (-22.4%), preserved 22/23 known TPs

3. **Class rescue via new/instanceof/extends** (Mar 6, #497)
   - Non-exported classes used via `new`, `instanceof`, `extends` no longer flagged dead
   - Example: `BlobNotFoundError` in job-worker.service.ts

4. **Same-file rescue patterns** (broadened, #443)
   - Callback references: `items.map(processItem)`
   - Object literal methods: `{ handler: myFunction }`
   - JSX component references: `<TopLinks />`
   - Module-scope calls: `const X = getZipLimits()`
   - Default parameters: `function foo(cb = myHelper) {}`

---

## Running a New Benchmark

### Prerequisites

1. **Docker Desktop** running
2. **Environment variables** set:
   ```bash
   export ANTHROPIC_API_KEY="sk-ant-..."
   export SUPERMODEL_API_KEY="smsk_live_..."
   ```
3. **mcpbr-eval** repo set up (see below)

### Setup (one-time)

```bash
# The benchmark repo
cd ~/mcpbr-eval
git checkout feat/supermodel-benchmark

# Install dependencies
uv pip install -e ".[dev]"

# Verify it works
uv run pytest -m "not integration"
```

### Run the Benchmark

**Option A: Run all 17 tasks (MCP-only, ~$5-10, ~2 hours)**

This is the fastest way to get "after" numbers for the blog post. No baseline needed since we already have baseline data showing 0% across the board.

```bash
cd ~/mcpbr-eval
uv run mcpbr run --config config/supermodel-deadcode-pr.yaml --mcp-only -vv
```

**Option B: Run specific high-signal tasks (recommended first)**

Start with the 5 tasks that had internal recall improvements:

```bash
cd ~/mcpbr-eval

# These are the tasks with known recall improvements from parser fixes
for task in tyr_pr258 jslpsolver_pr159 mimir_pr3613 directus_pr26311 latitude_pr2300; do
  echo "=== Running $task ==="
  uv run mcpbr run --config config/supermodel-deadcode-pr.yaml --mcp-only -t $task -vv --no-incremental
done
```

**Option C: Run with baseline comparison (~$15-20, ~4 hours)**

Full A/B comparison. Only needed if we want fresh baseline numbers.

```bash
cd ~/mcpbr-eval
uv run mcpbr run --config config/supermodel-deadcode-pr.yaml -vv
```

### After the Run

Results will be in the latest `.mcpbr_run_*` directory:

```bash
# Find the latest run
ls -td ~/mcpbr-eval/.mcpbr_run_* | head -1

# Copy results to this repo
LATEST=$(ls -td ~/mcpbr-eval/.mcpbr_run_* | head -1)
cp -r "$LATEST" ~/dead-code-benchmark-blog/runs/mar-09-post-parser-improvements/

# Quick summary
cat "$LATEST/results.json" | python3 -c "
import json, sys
d = json.load(sys.stdin)
print(f\"MCP: {d['summary']['mcp']['resolved']}/{d['summary']['mcp']['total']} resolved\")
print(f\"Cost: \${d['summary']['mcp']['total_cost']:.2f}\")
for t in d['tasks']:
    m = t['mcp']
    print(f\"  {t['instance_id']}: P={m.get('precision',0):.1%} R={m.get('recall',0):.1%} F1={m.get('f1_score',0):.1%} ({m.get('true_positives',0)} TP, {m.get('false_positives',0)} FP)\")
"
```

### Configuration Reference

The benchmark config is at `~/mcpbr-eval/config/supermodel-deadcode-pr.yaml`. Key settings:

| Setting | Value | Notes |
|---------|-------|-------|
| `model` | `claude-sonnet-4-20250514` | Claude Sonnet 4 |
| `agent_harness` | `claude-code` | Uses Claude Code as agent |
| `max_iterations` | 30 | Agent turn limit |
| `timeout_seconds` | 1200 | 20 min per task |
| `supermodel_api_base` | staging.api.supermodeltools.com | Uses staging API |
| `resolved_threshold` | 0.8 | P>=80% AND R>=80% to "resolve" |

### Task Quality Notes

From the config file comments and benchmark experience:

| Task | Quality | Notes |
|------|---------|-------|
| tyr_pr258 | Best | 95.5% API recall (21/22 GT), clean TypeScript monorepo |
| directus_pr26311 | Good | 15 GT items, multi-package structure |
| mimir_pr3613 | Good | 9 GT items, Statistics Norway portal |
| latitude_pr2300 | Good | 5 GT items, monorepo with packages |
| jslpsolver_pr159 | Good | 10 GT items, smaller repo (baseline previously outperformed) |
| podman_pr16084 | OK | Only 2 GT items (small sample) |
| otel_js_pr5444 | OK | 4 GT items, non-determinism observed |
| maskbook_pr12361 | OK | 22 GT items but 6000+ candidates (analysis dump risk) |
| gemini_cli_pr18681 | Poor | 33% API recall, test-import gap |
| prisma_pr28485 | Poor | 0% recall (GT items only referenced by tests) |
| n8n_pr23572 | Invalid | Feature removal, not dead code (GT items are alive) |
| typescript_pr56817 | Blocked | Too large for staging API without cached_analysis |

---

## False Positive Taxonomy

### 7 Categories Ranked by Impact

| Priority | Root Cause | % of FPs | Status (Mar 9) |
|----------|-----------|----------|----------------|
| **P0** | Import resolution: too many root files (145 in Tyr, should be ~3) | ~51% | **Partially fixed** (barrel re-exports done, framework wiring still missing) |
| **P1** | `export type { X } from` re-exports not tracked | ~15% | Not fixed |
| **P2** | `export default X` not detected as export | ~5% | Not fixed |
| **P3** | JSX usage not recognized as calls (React components) | ~20% in React repos | **Partially fixed** (same-file rescue catches `<Foo />`, cross-file misses) |
| **P4** | Test/script/config file imports not scanned | 11/28 scream test FPs | Not fixed |
| **P5** | Python `isExported` broken (only checks `__all__`) | Disables Python pipeline | Not fixed |
| **P6** | Type/interface structural typing references | ~21% in TS repos | Hard to fix |

### Detail: P0 - Import Resolution

The dominant false positive source. When import resolution fails to trace how a file is consumed (e.g., Express `app.use(router)`, dynamic `require()`, framework wiring), the file becomes a "root file" with no importers. All its exports then get flagged as "exported but file never imported."

In the Tyr benchmark, there were **145 root files** (should be ~3-5). This single issue produced ~396 of 775 candidates (51%).

**What's fixed:** Barrel re-exports now correctly traced. Step 2b fixpoint loop handles cascading dead imports.

**What's still broken:** Framework-level wiring (Express router mounting, NestJS module registration, dynamic requires).

### Detail: P4 - Test/Script Imports

Symbols imported only from test files, build scripts, or config files (e.g., `vitest.config.ts`, `rollup.config.mjs`) appear to have "no importers" because the scanner doesn't cover those file types. This was 11 of 28 FPs in the jslpsolver scream test.

**Highest-ROI unfixed item.** Scanning test file imports would eliminate a huge chunk of FPs with minimal regression risk.

---

## Benchmark-Driven Parser Improvement System

### The Loop

```
1. Run benchmark -> measure P/R/F1 per task
2. Sample FPs -> categorize root causes (using taxonomy above)
3. Pick highest-impact root cause -> implement fix
4. Run benchmark again -> verify improvement, check for regressions
5. Repeat
```

### Tiered Testing

| Tier | Speed | Cost | What It Tests | When to Run |
|------|-------|------|--------------|-------------|
| **Tier 1** | Seconds | Free | Unit tests: does parser extract `export default`? | Every commit |
| **Tier 2** | Minutes | Free | Candidate-level regression: run analysis on 5 repos, diff candidate counts | Every PR |
| **Tier 3** | Hours | $5-50 | Full agent benchmark via mcpbr: measures whole pipeline end-to-end | Weekly / pre-release |

### Tier 2: Candidate-Level Regression (Cheap & Fast)

Before running an expensive Tier 3 benchmark, verify parser changes with a candidate-level check:

```bash
# 1. Check out the benchmark repo at pre-PR commit
# 2. Run Supermodel dead code analysis (just the API, no agent)
# 3. Count candidates and check if known TPs are still detected
# 4. Compare candidate count to baseline

# Example for Directus:
# Before fix: 2,808 candidates, 22/23 TPs detected
# After fix:  2,178 candidates (-22.4%), 22/23 TPs still detected
# = Good: fewer FPs, no TP regression
```

### Ground Truth Registry

Formalize the 12 PRs as a regression test suite:

```json
{
  "tasks": [
    {
      "id": "tyr_pr258",
      "repo": "uncovering-world/track-your-regions",
      "merge_commit": "6f480121...",
      "ground_truth_count": 22,
      "best_api_recall": 0.955,
      "best_agent_recall": 0.955,
      "known_misses": ["one item missed by API"]
    }
  ]
}
```

### FP Classification Pipeline

After each benchmark run, automatically categorize FPs:

1. For each FP, check: imported by test file? Framework entry point? Type re-export? Generated directory?
2. Tag each FP with root cause category
3. Track category volumes over time
4. Build a precision/recall dashboard per root cause

---

## Key Files in Related Repos

| What | Where |
|------|-------|
| **mcpbr-eval** (benchmark runner) | `~/mcpbr-eval` (branch: `feat/supermodel-benchmark`) |
| **Benchmark config** | `~/mcpbr-eval/config/supermodel-deadcode-pr.yaml` |
| **Supermodel benchmark code** | `~/mcpbr-eval/src/mcpbr/benchmarks/supermodel/` |
| **Dead code endpoint impl** | `~/mcpbr-eval/src/mcpbr/benchmarks/supermodel/endpoints/dead_code.py` |
| **Parser (tree-sitter)** | `~/supermodel-public-api/src/data-plane/src/parsers/tree-sitter.ts` |
| **Dead code pipeline** | `~/supermodel-public-api/src/data-plane/src/services/job-worker.service.ts:1344-3181` |
| **Blog post draft** | `~/jonathanpopham.github.io/_drafts/2026-03-09-dead-code-graphs-ai-agents.md` |
| **Previous run data (Downloads)** | `~/Downloads/blogpost-deadcode/`, `fri-feb-20-*`, `sun-feb-23-*` |
