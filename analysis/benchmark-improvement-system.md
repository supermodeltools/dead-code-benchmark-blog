# Benchmark-Driven Parser Improvement System

## The Core Loop

```
1. Run benchmark -> measure P/R/F1 per task
2. Sample FPs -> categorize root causes (fp-taxonomy.md)
3. Pick highest-impact root cause -> implement fix
4. Run benchmark again -> verify improvement, check for regressions
5. Repeat
```

## Tiered Testing

| Tier | Speed | Cost | What It Tests | When to Run |
|------|-------|------|--------------|-------------|
| **Tier 1** | Seconds | Free | Unit tests: parser extracts `export default` correctly | Every commit |
| **Tier 2** | Minutes | Free | Candidate-level regression: analysis on 5 repos, diff counts | Every PR |
| **Tier 3** | Hours | $5-50 | Full agent benchmark via mcpbr: whole pipeline E2E | Weekly / pre-release |

### Tier 1: Unit Tests

Located in `~/supermodel-public-api/tests/unit/data-plane/dead-code-analysis.test.ts`

Test individual parser behaviors:
- Does `isNodeExported()` detect `export default`?
- Does import extraction find `export type { X } from`?
- Does Phase 7.5b rescue `<Component />` references?
- Does Step 2b fixpoint correctly cascade dead imports?

Run: `cd ~/supermodel-public-api/src/data-plane && npx jest`

### Tier 2: Candidate-Level Regression (Cheap & Fast)

Test parser changes WITHOUT running the expensive agent benchmark. Just compare candidate lists.

**Process:**
1. For each of 5 reference repos (tyr, mimir, directus, jslpsolver, latitude):
   - Check out at pre-PR commit
   - Run Supermodel dead code analysis (API call only)
   - Count total candidates
   - Check if known ground truth items are in candidate list
2. Compare to baseline counts

**What to track:**
- Total candidate count (should decrease with fixes)
- TP count (should stay same or increase)
- Known FP categories eliminated

**Example baseline (pre-barrel-fix):**
```
tyr_pr258:      775 candidates, 21/22 TPs
mimir_pr3613:   ~400 candidates, 7/9 TPs
directus:       2,808 candidates, 22/23 TPs
jslpsolver:     42 candidates, 5/10 TPs
latitude:       ~1,500 candidates, 5/5 TPs
```

**After barrel-fix (Mar 6 internal):**
```
directus:       2,178 candidates (-22.4%), 22/23 TPs (preserved)
```

### Tier 3: Full Agent Benchmark

```bash
cd ~/mcpbr-eval
uv run mcpbr run --config config/supermodel-deadcode-pr.yaml --mcp-only -vv
```

Cost: ~$5-10 for MCP-only, ~$15-20 with baseline
Time: ~2-4 hours
Measures: Full pipeline (API analysis -> agent consumption -> REPORT.json -> evaluation)

## Ground Truth Registry

Formalize as a JSON file for automated regression testing:

```json
{
  "version": "2026-03-09",
  "tasks": [
    {
      "id": "tyr_pr258",
      "repo": "uncovering-world/track-your-regions",
      "pr_number": 258,
      "merge_commit": "6f480121e4641b5551e6af092283a5d161490e5f",
      "language": "typescript",
      "ground_truth_count": 22,
      "baseline_api_recall": 0.955,
      "baseline_candidate_count": 775,
      "known_misses": ["1 item missed by API -- unknown which"]
    },
    {
      "id": "directus_pr26311",
      "repo": "directus/directus",
      "pr_number": 26311,
      "merge_commit": "acf6d3d4caebe52888a9d65fdd69a85e0897ba27",
      "language": "typescript",
      "ground_truth_count": 15,
      "baseline_api_recall": 1.0,
      "baseline_candidate_count": 2808,
      "post_barrel_fix_candidate_count": 2178,
      "known_misses": []
    }
  ]
}
```

## FP Classification Pipeline

After each benchmark run:

1. **Extract FPs** from results.json (items in `found` but not in `expected`)
2. **Classify each FP:**
   - Is the symbol's file a "root file" (no importers)? -> P0
   - Is the symbol a type re-export? -> P1
   - Is the symbol an `export default`? -> P2
   - Is the symbol a React component used via JSX? -> P3
   - Is the symbol only imported by test/config files? -> P4
   - Is the symbol a Python module-level function? -> P5
   - Is the symbol a TypeScript interface? -> P6
3. **Aggregate counts** per category
4. **Compare to previous run** -- are category counts decreasing?

## Tracking Metrics Over Time

| Metric | Feb 20 | Feb 23 | Mar 9 (target) |
|--------|--------|--------|----------------|
| Avg recall (5 key tasks) | 74.6% | 54.3% | **85%+** |
| Avg precision (5 key tasks) | 1.3% | 3.5% | **5%+** |
| Total candidates (Directus) | 2,450 | 777 | **<600** |
| Resolved tasks (P>=80%, R>=80%) | 0/10 | 0/16 | **1+** |

## What "Success" Looks Like

**Short-term (Mar 9 benchmark):**
- Recall on 5 key tasks matches or exceeds internal numbers (85%+ avg)
- Candidate counts visibly reduced from Feb 20/23 runs
- At least 1-2 tasks approach "resolved" threshold

**Medium-term (next 2-4 weeks):**
- P4 fix (test imports) lands, reducing FPs by ~15-20%
- P1 fix (type re-exports) lands, reducing FPs by ~10-15%
- Precision crosses 10% on multiple tasks

**Long-term:**
- CI pipeline runs Tier 2 regression on every PR
- Parser improvements measured against ground truth registry
- Dead code tool usable in production with <50 FPs on typical repos
