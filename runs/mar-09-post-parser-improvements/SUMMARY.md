# March 9, 2026 - Benchmark Runs

## Three Phases

### Phase 1: Cached analysis + grep verification prompt (stale)
Runs: 132846, 133715, 133859, 134145, 134619

Used cached analysis from Feb (old parser). Agent prompt included grep verification step.
These results show the effect of agent grep filtering on old analysis.

| Task | P | R | F1 | TP | FP | FN | Notes |
|------|---|---|-----|-----|-----|-----|-------|
| tyr_pr258 | 4.4% | 40.0% | 8.0% | 8 | 172 | 12 | Agent grep killed 13 real TPs |
| mimir_pr3613 | 2.0% | 87.5% | 3.9% | 7 | 342 | 1 | Agent didn't grep-filter (lucky) |
| directus_pr26311 | 1.7% | 85.7% | 3.3% | 12 | 694 | 2 | Partial filtering |
| latitude_pr2300 | 0.0% | 0.0% | 0.0% | 0 | 876 | 5 | Analysis too large, agent lost |
| jslpsolver_pr159 | 13.6% | 50.0% | 21.4% | 3 | 19 | 3 | Consistent with Feb 20 |

### Phase 2: Cache-busted but still cached locally (partial fix)
Runs: 135109, 135638, 141405

Changed idempotency key to `:v2` but local cache still served old results.

| Task | P | R | F1 | TP | FP | Notes |
|------|---|---|-----|-----|-----|-------|
| mimir_pr3613 | 0.6% | 87.5% | 1.2% | 7 | 1124 | Same old analysis, no grep filter |
| directus_pr26311 | 0.6% | 100.0% | 1.1% | 14 | 2451 | Same old analysis, no grep filter |
| latitude_pr2300 | 0.7% | 100.0% | 1.4% | 5 | 729 | Same old analysis, no grep filter |
| tyr_pr258 | ERROR | - | - | - | - | Clone failed (network) |

### Phase 3: Fresh API calls with improved parser (final)
Runs: 161346, 161447, 161504, 161735, 161838, 161919, 162133, 162248

Local cache cleared. Idempotency key `:v2`. No grep verification in prompt.
Fresh analysis from staging API with improved parser (barrel re-exports, 7 new phases).

| Task | P | R | F1 | TP | FP | FN | Found | Cost |
|------|---|---|-----|-----|-----|-----|-------|------|
| tyr_pr258 | 4.3% | 90.0% | 8.2% | 18 | 403 | 2 | 421 | $0.19 |
| mimir_pr3613 | 0.8% | 100.0% | 1.6% | 8 | 956 | 0 | 964 | $0.17 |
| directus_pr26311 | 1.4% | 92.9% | 2.9% | 13 | 885 | 1 | 898 | $0.15 |
| latitude_pr2300 | 0.7% | 100.0% | 1.4% | 5 | 729 | 0 | 734 | $0.15 |
| jslpsolver_pr159 | 22.2% | 100.0% | 36.4% | 6 | 21 | 0 | 27 | $0.11 |

**Averages: 96.6% recall, 5.9% precision, 10.1% F1**
**Total cost: $0.77 for 5 tasks**

## Comparison: Feb 20 (old parser) vs Mar 9 Phase 3 (new parser)

| Task | Feb R | Mar R | Feb P | Mar P | Feb F1 | Mar F1 | Feb FP | Mar FP | FP Change |
|------|-------|-------|-------|-------|--------|--------|--------|--------|-----------|
| tyr | 95.5% | 90.0% | 3.8% | 4.3% | 7.2% | 8.2% | 537 | 403 | -25% |
| mimir | 77.8% | 100% | 0.6% | 0.8% | 1.2% | 1.6% | 1,124 | 956 | -15% |
| directus | 100% | 92.9% | 0.6% | 1.4% | 1.2% | 2.9% | 2,450 | 885 | -64% |
| latitude | 100% | 100% | 0.3% | 0.7% | 0.7% | 1.4% | 1,500 | 729 | -51% |
| jslpsolver | 50% | 100% | 11.9% | 22.2% | 19.2% | 36.4% | 37 | 21 | -43% |
| **Avg** | **84.7%** | **96.6%** | **3.4%** | **5.9%** | **5.9%** | **10.1%** | **5,648** | **2,994** | **-47%** |

Precision improved on every task. F1 improved on every task. Recall improved on 2, held on 1, dipped slightly on 2.
