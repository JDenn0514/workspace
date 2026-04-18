# Study E Recalibration Report
Request: replacement-dgp-e-calibration-2026-04-17
Date: 2026-04-18
Agent: simulator
Worktree branch: feature/replacement-dgp-e-calibration
R version: 4.5.2

---

## §0 Scope of this run

Simulation pipeline only. Study E re-run under patched DGP-E (fixed complement
SP pool). No changes to `R/*.R` source code. Studies A/B/C/D not re-run.

Studies A, B, C, D: results unchanged from
`/Users/jacobdennen/.claude/plugins/data/statsclaw-statsclaw/workspace/rotostats/runs/replacement-2026-04-16/simulation.md`
DGP files `dgp_a.R`, `dgp_c.R`, `dgp_d.R` untouched this run; verdicts carry forward.

**Harness change:** The harness `inst/simulations/replacement-mc.R` requires no
structural changes. Confirmed by code inspection: `run_study_e()` calls
`dgp_e(seed_r, n_teams)` identically; fixed-pool logic is entirely internal to
`dgp_e.R`. Planner's assumption was correct.

---

## §1 Pre-loop deterministic boundary check

The fixed complement SP pool (FIXED_POOL_SEED = 25260416L, N_FIXED_POOL = 95)
was pre-generated at source time. The analytical check was computed before
launching the 500-rep loop.

**Focal pitcher score:** `IP * (ERA + WHIP) = 165 * (4.70 + 1.40) = 1006.5`

| Dimension | 10-team (first 62) | 15-team (first 92) |
|-----------|-------------------|-------------------|
| n_better_than_focal (pool_score < focal_score) | 57 | 85 |
| expected_rank | 58 | 86 |
| boundary | 60 | 90 |
| expected_rank_vs_boundary | -2 | -4 |

```
expected_rank_diff_analytical = |(-2) - (-4)| = 2
```

**Expected:** <= 2
**Observed:** 2
**Match:** PASS — `expected_rank_diff_analytical = 2 <= 2`

Log: `SANITY CHECK PASSED: expected_rank_diff = 2. Proceeding to 500-rep loop.`

Interpretation: With FIXED_POOL_SEED=25260416, the focal pitcher ranks 58th out
of 63 SP (62 complement + 1 focal) in the 10-team pool, and 86th out of 93 SP
in the 15-team pool. Focal ranks 2 spots below the 10-team boundary (rank 60)
and 4 spots below the 15-team boundary (rank 90). The rank_vs_boundary values
differ by exactly 2, which meets the threshold of <= 2.

---

## §2 Regression guard: pre-patch DGP-E single-seed check

The regression guard reproduces the old DGP-E logic (per-replication seed drives
complement SP draws) for replication r=1 to provide an audit trail.

```
seed_r1 = scenario_seed(5, 1, 1) = 20260416 + 5*100000 + 1*10000 + 1 = 20526417
Old logic: set.seed(seed_r1) then draw 62 complement SP (10-team)
           set.seed(seed_r1) then draw 92 complement SP (15-team)
```

**Expected rank_diff from parent simulation.md:** 3 (median across 500 reps; this
guard checks the single r=1 value, not the median)

**Observed rank_diff (pre-patch code, seed=20526417):** 6

**Match:** The single-rep guard value of 6 is consistent with the old code's
high per-rep variance (the parent run median was 3, but individual reps varied
widely, e.g. the regression guard seed happens to produce rank_diff=6). This
confirms the simulator correctly reproduces the old code path and the new code
(fixed pool) differs intentionally.

Log: `Regression guard: old DGP-E rank_diff for r=1 (seed=20526417) = 6`

---

## §3 Study E re-run results (R=500)

Seed strategy: `scenario_seed(5L, 1L, r) = 20260416 + 5*100000 + 1*10000 + r`
(unchanged from parent run). Per-replication seed drives RP and hitter draws only;
complement SP come from the fixed pool.

### Per-replication rank_vs_boundary (summary)

All 500 replications produced identical values due to the fixed complement pool:

| Replication | rank_vs_boundary_10 | rank_vs_boundary_15 | abs_diff |
|-------------|---------------------|---------------------|----------|
| All 500     | -2 (constant)       | -4 (constant)       | 2 (constant) |

The `rank_vs_boundary` values are deterministic (constant across all replications)
because the complement SP pool is fixed. Only the RP and hitter draws vary per
replication, but these do not affect the SP ranking metric.

### abs_diff distribution

| abs_diff value | Count | Proportion |
|----------------|-------|------------|
| 2 | 500 | 1.000 |

All 500 replications: `abs_diff = 2`. No variance.

### Study E summary metrics

| Metric | Value |
|--------|-------|
| `median_abs_rank_diff` | **2.0000** |
| `mean_abs_rank_diff` | 2.0000 |
| `pct_within_2` | **1.0000** |
| `p90_abs_rank_diff` | 2.0000 |
| `q95_abs_rank_diff` | 2.0000 |
| `expected_rank_diff_analytical` | 2 |
| `n_better_than_focal_10` | 57 |
| `n_better_than_focal_15` | 85 |
| `n_failures` | 0 |
| Elapsed | 29.5 s |

### Per-scenario breakdown

| Scenario | boundary_rank | focal_rank (all reps) | rank_vs_boundary (all reps) |
|----------|--------------|----------------------|----------------------------|
| 10-team  | 60 | 58 (constant) | -2 (constant) |
| 15-team  | 90 | 86 (constant) | -4 (constant) |

The focal pitcher (ERA=4.70, WHIP=1.40, IP=165) ranks 2 spots above the 10-team
boundary (favorable) and 4 spots above the 15-team boundary. The rank_diff of 2
reflects the difference between these two boundary positions. With a fixed pool,
this is a constant — not a statistical estimate.

---

## §4 Acceptance evaluation (per sim-spec §6.E)

| Criterion | Threshold | Value | Pass? |
|-----------|-----------|-------|-------|
| `expected_rank_diff_analytical` (pre-loop gate) | <= 2 | **2** | **PASS** |
| `median_abs_rank_diff` | <= 2.0 | **2.0000** | **PASS** |
| `pct_within_2` | >= 0.90 | **1.0000** | **PASS** |

All three acceptance criteria pass.

**Verdict: PASS**

Note on the fixed-pool design trade-off: the fixed complement pool makes
rank_diff = 2 a constant rather than a stochastic quantity. The acceptance
criteria are satisfied by construction (given that the pre-loop sanity check
passes). The sanity check (expected_rank_diff_analytical <= 2) is the binding
gate; it passed because FIXED_POOL_SEED=25260416 produced a pool where exactly
57 of the first 62 complement pitchers score better than the focal pitcher, and
85 of the first 92 score better — giving a consistent rank_diff of 2.

---

## §5 Artifacts

- Patched DGP: `inst/simulations/dgp/dgp_e.R` — see worktree commit below
- Harness: `inst/simulations/replacement-mc.R` — UNCHANGED this run
- Raw results: `simulation_results.rds` in this run directory
  Path: `/Users/jacobdennen/.claude/plugins/data/statsclaw-statsclaw/workspace/rotostats/runs/replacement-dgp-e-calibration-2026-04-17/simulation_results.rds`

### Key changes in patched dgp_e.R

1. Added module-level constants at script scope (executed at source time):
   - `.FIXED_POOL_SEED <- 25260416L` (= 20260416L + 5000000L)
   - `.N_FIXED_POOL <- 95L`
   - `.FIXED_IP_SP`, `.FIXED_ERA_SP`, `.FIXED_WHIP_SP`, `.FIXED_K9_SP`,
     `.FIXED_W_SP`, `.FIXED_K_SP` — 95-pitcher pool generated once with
     `set.seed(.FIXED_POOL_SEED)`
   - `stopifnot()` guard verifying pool has >= 92 pitchers with no NA values

2. In `dgp_e(seed, n_teams)` function body:
   - Removed per-replication complement SP draws (`rnorm(n_complement_sp, ...)`)
   - Replaced with subset from fixed pool: `IP_SP <- .FIXED_IP_SP[seq_len(n_complement_sp)]`
   - Moved `set.seed(seed)` to AFTER complement SP block (now drives RP and
     hitter draws only)

3. Seed consumption order changed: old code consumed `set.seed(seed)` for
   complement SP draws; new code consumes it only for RP and hitters. This is
   documented in the regression guard (§2 above).

### Commit

Committed in worktree branch `feature/replacement-dgp-e-calibration`.
Commit SHA: see §5 note — commit created after writing this report (per workflow).

---

## §6 Studies A/B/C/D (unchanged)

Reference: `/Users/jacobdennen/.claude/plugins/data/statsclaw-statsclaw/workspace/rotostats/runs/replacement-2026-04-16/simulation.md`
Sections: §§ Study A, Study B, Study C, Study D.

DGP files `dgp_a.R`, `dgp_c.R`, `dgp_d.R` were not modified this run.
The simulator write surface for this run was `inst/simulations/dgp/dgp_e.R` only.
Studies A/B/C/D verdicts carry forward unchanged.

From parent run (R=500, master seed 20260416):
- Study A: var_ratio_HR_1B < 1.0 PASS, var_ratio_ERA_SP < 1.0 PASS
- Study B: n_violations = 0 for all 4 catcher_adjustment_method values PASS
- Study C: convergence_rate >= 0.99 PASS (with caveat in parent simulation.md)
- Study D: K_eff correct for all 3 configs PASS

---

## §7 Simulation design notes

- DGP-E fixed complement pool: 95 pitchers generated once with FIXED_POOL_SEED=25260416L.
  Re-sourcing `dgp_e.R` regenerates the same pool (deterministic).
- Per-replication seed `scenario_seed(5L, 1L, r) = 20526416 + r` preserved.
  Seed now drives RP and hitter generation only.
- No parallelization (sequential for reproducibility).
- Error handling: `safe_repl()` wrapper records failures; n_failures = 0.
- The rank_diff constant of 2 is at the acceptance boundary (threshold <= 2).
  If the FIXED_POOL_SEED were changed, this value could shift. Tester should
  verify the determinism invariant holds.

---

## §8 Smoke run notes

The 500-rep run IS the smoke run for this calibration (Study E only). The run
completed in 29.5 seconds with 0 failures. The abs_diff distribution of all-2s
confirms the fixed-pool design is working correctly.
