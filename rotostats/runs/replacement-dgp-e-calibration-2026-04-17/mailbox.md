# Mailbox: replacement-dgp-e-calibration-2026-04-17

---

## 2026-04-18 — Planner handoff

**Type:** HANDOFF

**Comprehension verdict:** FULLY UNDERSTOOD. No HOLD rounds needed.

**Option chosen:** Option 3 (fixed complement SP pool). Rationale: eliminates all
between-replication pool variance from the rank_diff metric. The rank_diff becomes a
deterministic constant per fixed pool, guaranteeing pct_within_2 = 1.0 if the
pre-loop sanity check passes. Option 1 (further below pool mean) may help but
does not eliminate the variance source. Option 2 (lower pool sigma) changes the pool
distribution character and may be insufficient.

---

### For simulator

**Read:** `runs/replacement-dgp-e-calibration-2026-04-17/sim-spec.md`

**Task:** Patch `inst/simulations/dgp/dgp_e.R` as specified in §2.E (PATCHED). Then
run Study E only (R=500, seed strategy unchanged) and write `simulation.md` to this
run directory.

**Key changes to `dgp_e.R`:**

1. Add module-level constants at the top of the file (outside any function):
   ```r
   .FIXED_POOL_SEED <- 25260416L   # = 20260416L + 5000000L
   .N_FIXED_POOL    <- 95L
   set.seed(.FIXED_POOL_SEED)
   .FIXED_IP_SP    <- pmin(pmax(round(stats::rnorm(.N_FIXED_POOL, 170, 15)), 120L), 230L)
   .FIXED_ERA_SP   <- pmin(pmax(stats::rnorm(.N_FIXED_POOL, 3.80, 0.45), 2.50), 6.00)
   .FIXED_WHIP_SP  <- pmin(pmax(stats::rnorm(.N_FIXED_POOL, 1.22, 0.10), 0.90), 1.80)
   .FIXED_K9_SP    <- pmin(pmax(stats::rnorm(.N_FIXED_POOL, 8.5, 1.2), 4.0), 14.0)
   .FIXED_W_SP     <- stats::rpois(.N_FIXED_POOL, lambda = .FIXED_IP_SP / 9 * 0.44)
   .FIXED_K_SP     <- round(.FIXED_IP_SP * .FIXED_K9_SP / 9)
   ```

2. In the `dgp_e(seed, n_teams)` function body: replace the complement SP random draws
   (the `IP_SP <- rnorm(...)`, `ERA_SP <- rnorm(...)`, etc. block) with subsetting from
   the fixed pool:
   ```r
   n_complement_sp <- n_teams * sp_slots + 2L
   IP_SP   <- .FIXED_IP_SP[seq_len(n_complement_sp)]
   ERA_SP  <- .FIXED_ERA_SP[seq_len(n_complement_sp)]
   WHIP_SP <- .FIXED_WHIP_SP[seq_len(n_complement_sp)]
   W_SP    <- .FIXED_W_SP[seq_len(n_complement_sp)]
   K_SP    <- .FIXED_K_SP[seq_len(n_complement_sp)]
   # Remove K9_SP reference since K_SP is now precomputed
   ```

3. Move the `set.seed(seed)` call to AFTER the complement SP block (before RP draws),
   since the complement SP no longer use the per-replication seed:
   ```r
   set.seed(seed)  # now drives only RP and hitter draws
   # RP draws follow...
   ```

4. Before the R=500 replication loop, compute and log the deterministic sanity check
   (per sim-spec.md §2.E "Deterministic boundary-rank sanity check"). STOP and emit
   BLOCK if expected_rank_diff_analytical > 2.

5. After Study E runs, include the regression guard (per sim-spec.md §6.E "Regression
   guard") in the simulation.md output.

**Harness changes:** The harness `replacement-mc.R` requires NO structural changes.
The `run_study_e()` function continues to call `dgp_e(seed_r, n_teams)` identically.

**Output:** Write `simulation.md` (fresh Study E section only; reference prior run's
A/B/C/D verdicts by path) and `simulation_results.rds` to this run directory.

**Key constraint:** Do NOT modify `R/replacement.R` or any `R/*.R` file. Simulator
write surface is `inst/simulations/dgp/dgp_e.R` only (with optional minor harness
tweak to support study-E-only re-run if desired).

---

### For tester

**Read:** `runs/replacement-dgp-e-calibration-2026-04-17/test-spec.md`

**Task:**

1. Run `devtools::test()` — confirm FAIL=0. If any previously-passing test fails, BLOCK.
2. Run the determinism check (§3.2 of test-spec.md) — verify fixed-pool invariant holds.
3. Read `simulation.md` from this run directory; evaluate Study E acceptance criteria
   (§3.3 of test-spec.md): median_abs_rank_diff ≤ 2 AND pct_within_2 ≥ 0.90.
4. Run the regression guard (§3.4 of test-spec.md) — log old DGP-E rank_diff for r=1.
5. Run `devtools::check()` — confirm 0 ERRORs, 0 WARNINGs.
6. Write `audit.md` with results.

**If Study E still fails:** Issue BLOCK; flag as threshold-revision candidate if three
planner rounds exhausted. Do NOT silently relax thresholds.

---

### For reviewer

**Read:** both this run's `sim-spec.md`, `test-spec.md`, `simulation.md`, `audit.md`;
plus parent run's `review.md`, `simulation.md`, `audit.md`.

**Task:** Produce `review.md` for this run with:
- §§Studies A/B/C/D: inherited verdicts from parent `replacement-2026-04-16/review.md`
- §Study E: fresh evaluation of this run's DGP-E patch
- §Pipeline isolation: confirm `R/*.R` was not touched; only `dgp_e.R` changed
- Final verdict: PASS or PASS_WITH_FOLLOWUP or BLOCK

---

*Planner out.*
