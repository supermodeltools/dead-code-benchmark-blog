# March 9, 2026 - Benchmarking Lessons

## What Happened

We ran the dead code benchmark to measure parser improvements made since Feb 23.
Three issues were discovered and fixed during the run.

## Issue 1: Agent Grep Verification Hurts Recall

**Discovery**: The enhanced prompt instructed the agent to write a `verify.sh` script
that greps for each candidate's symbol name across the codebase. If found in another
file, the candidate is marked "alive" and removed.

**Problem**: `grep -w` matches word boundaries, so dead function names appearing in
comments, strings, or unrelated identifiers in other files cause false negatives.
On tyr_pr258, this dropped recall from 95.5% to 40%.

**Fix**: Changed the prompt to tell the agent to trust the analysis and pass through
all candidates. The graph analysis has already performed proper call graph and
dependency analysis -- grep-based verification is strictly less accurate.

**File changed**: `mcpbr-eval/src/mcpbr/benchmarks/supermodel/benchmark.py` line 251
(the `_generate_enhanced_problem_statement` method)

## Issue 2: Local Analysis Cache

**Discovery**: The benchmark caches API responses in
`~/.cache/mcpbr/supermodel_ground_truth/{task_id}_analysis_{zip_hash}.json`.
Since the repo content at each commit is unchanged, the zip hash is the same,
and the cache returns the old parser's results.

**Fix**: Backed up and removed 53 cached analysis files:
```bash
mkdir -p ~/.cache/mcpbr/supermodel_ground_truth/backup_feb
cp ~/.cache/mcpbr/supermodel_ground_truth/*analysis* backup_feb/
rm ~/.cache/mcpbr/supermodel_ground_truth/*analysis*.json
```

## Issue 3: Server-Side Idempotency Cache

**Discovery**: Even after clearing the local cache, the Supermodel API returns
cached results for the same idempotency key. The key is auto-generated from
`bench:{endpoint}:{zip_hash}`, so identical zips produce identical keys.

**Fix**: Added version suffix to idempotency key:
```python
# api_client.py line 45
idempotency_key = f"bench:{ep_name}:{zip_hash}:v2"
```

## Results Comparison

### Before fixes (cached analysis, grep verification)

| Task | Recall | Precision | TP | FP | Notes |
|------|--------|-----------|-----|-----|-------|
| tyr_pr258 | 40% | 4.4% | 8 | 172 | Agent grep filtered out 13 real TPs |
| mimir_pr3613 | 87.5% | 2.0% | 7 | 342 | Agent didn't grep-filter (lucky) |
| directus_pr26311 | 85.7% | 1.7% | 12 | 694 | Partial filtering |
| latitude_pr2300 | 0% | 0% | 0 | 876 | Analysis too large, agent lost |
| jslpsolver_pr159 | 50% | 13.6% | 3 | 19 | Consistent with Feb 20 |

### After fixes (fresh API, no grep verification, improved parser)

| Task | Recall | Precision | TP | FP | Found |
|------|--------|-----------|-----|-----|-------|
| tyr_pr258 | **90.0%** | 4.3% | 18 | 403 | 421 |
| mimir_pr3613 | **100%** | 0.8% | 8 | 956 | 964 |
| directus_pr26311 | **92.9%** | 1.4% | 13 | 885 | 898 |
| latitude_pr2300 | **100%** | 0.7% | 5 | 729 | 734 |
| jslpsolver_pr159 | **100%** | 22.2% | 6 | 21 | 27 |

**Average recall: 96.6%** (up from 84.7% on Feb 20)

### Comparison: Feb 20 (old parser) vs Mar 9 (new parser)

| Task | Feb 20 Recall | Mar 9 Recall | Feb 20 FP | Mar 9 FP | FP Change |
|------|--------------|-------------|-----------|---------|-----------|
| tyr_pr258 | 95.5% | 90.0% | 537 | 403 | -25% |
| mimir_pr3613 | 77.8% | **100%** | 1,124 | 956 | -15% |
| directus_pr26311 | 100% | 92.9% | 2,450 | 885 | **-64%** |
| latitude_pr2300 | 100% | 100% | 1,500 | 729 | **-51%** |
| jslpsolver_pr159 | 50% | **100%** | 37 | 21 | -43% |
| **Total** | 84.7% avg | **96.6% avg** | 5,648 | 2,994 | **-47%** |

Key wins:
- jslpsolver recall doubled (50% → 100%) -- barrel re-export fix working
- mimir recall up 22pp (77.8% → 100%)
- Total FPs down 47% across all 5 tasks
- jslpsolver precision at 22.2% -- best on any real-world task
