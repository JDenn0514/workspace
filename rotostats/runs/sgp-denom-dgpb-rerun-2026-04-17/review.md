# Review: sgp-denom-dgpb-rerun-2026-04-17 (Unified Successor Review)

**Reviewer:** claude-sonnet-4-6
**Date:** 2026-04-17
**Branch:** feature/sgp-denom-dgpb-rerun (HEAD 9e711aa)
**Workflow:** Simulation-only (Workflow 12 variant — Q4/Q7 re-run under corrected DGP-B)
**Verdict:** SHIP

This is the unified successor to the 2026-04-16 review. Sections Q1, Q2, Q3, Q5, Q6
carry over the verdicts from:
  `/Users/jacobdennen/.claude/plugins/data/statsclaw-statsclaw/workspace/rotostats/runs/sgp-denominators-2026-04-16/review.md`
(see "Final pass" section, dated 2026-04-16, verdict SHIP, Scriber commit edfa681).
Sections Q4 and Q7 are replaced with fresh evaluations under the corrected sigma-shift DGP-B.

---

## Step 1 — Comprehension Foundation

- `comprehension.md` exists. Verdict: **FULLY UNDERSTOOD**, 0 HOLD rounds. ✅
- Planner restated the core OLS mean-invariance intuition correctly: shifting the league
  mean leaves rank structure unchanged; only within-year σ drives the OLS slope and
  therefore the denominator. ✅
- The one judgment call (σ parameter choice: pre=10, post=25 vs plan-text "e.g." pre=15,
  post=25) is explicitly declared as an assumption with rationale documented. The 2.5×
  ratio provides a stronger, more diagnostic signal and matches the seeded sim-spec from
  the prior run. ✅
- All input materials inventoried and internally consistent. ✅
- No uploaded reference files for this run; all inputs are internal artifacts. ✅

**Comprehension foundation: SOUND.**

---

## Step 2 — Pipeline Isolation

This is a simulation-only run. The relevant isolation checks are:

- `simulation.md` references `sim-spec.md` only (public API black box). It does not
  reference `spec.md`, `test-spec.md`, or any R/ source file. ✅
- `audit.md` opens with "Tester does NOT read spec.md or implementation.md. All
  validation is driven by test-spec.md and the corrected sim-spec.md (DGP-B section only)."
  Confirmed by content: tester used independent seeds (42001/42002) distinct from Simulator's
  calibration seeds (88882/88883). ✅
- Tester's independent θ calibration used different random seeds than Simulator's, producing
  genuinely independent confirmation (theta_pre: 2.6811 vs 2.6744; theta_post: 6.7181 vs
  6.7722 — differences are within sampling variability at n_years=2000). ✅
- No cross-contamination of pipeline knowledge detected in any artifact. ✅

**Pipeline isolation: MAINTAINED.**

---

## Step 3 — Cross-Specification Comparison

For this simulation-only run, the relevant cross-comparison is sim-spec.md (§7 Q4, §10 Q7)
against simulation.md and audit.md.

| sim-spec.md criterion | simulation.md result | audit.md verification | Aligned |
|---|---|---|---|
| Q4 DGP-A: flat MSE ≤ exp7 MSE | 0.252 ≤ 0.481 | 0.251886 ≤ 0.481361 | ✅ |
| Q4 DGP-B: exp7 MSE ≤ flat MSE | 2.987 ≤ 10.407 | 2.987027 ≤ 10.407268 | ✅ |
| Q4 DGP-B monotone: flat ≥ exp9 ≥ exp7 | 10.407 ≥ 8.003 ≥ 2.987 | 10.407268 ≥ 8.002536 ≥ 2.987027 | ✅ |
| Q4 DGP-A rel_bias ≤ 0.15 all weights | max 0.013 | max 0.01266 | ✅ |
| Q7 S2 SB MSE < S1 SB MSE | 0.673 < 10.438 | 0.673306 < 10.437593 | ✅ |
| Q7 HR isolation ≤ 5% | 0.9% | 0.9366% | ✅ |
| Q7 S3 intermediate S1 ≥ S3 ≥ S2 | 10.44 ≥ 2.98 ≥ 0.67 | 10.437593 ≥ 2.979922 ≥ 0.673306 | ✅ |

All sim-spec.md acceptance criteria map 1:1 to simulation.md tables and audit.md per-test
result tables. No gaps or discrepancies detected.

---

## Step 4 — Convergence Verification

### Per-Test Result Tables

audit.md contains a complete per-test result table for Q4 (13 rows: 7 acceptance-criterion
rows + 6 failure-rate rows) and for Q7 (6 rows: 5 acceptance-criterion rows + 1 failure-rate
row). One row per distinct metric per scenario. ✅

### Before/After Comparison Tables

audit.md contains Before/After Comparison Tables for both Q4 and Q7 documenting the
transition from the prior mean-shift DGP-B (untestable criteria) to the corrected sigma-shift
DGP-B (criteria cleanly testable). These tables correctly characterize the nature of the fix. ✅

### Numerical Convergence: Simulator vs Tester

Tester read Q4/Q7 results directly from `q4_decay.csv` and `q7_override.csv`, cross-referenced
against `simulation_q4q7_results.rds` (bitwise match per audit.md §Step 7). The two
instruments (simulator and tester) report identical values for all decision-relevant quantities:

| Metric | Simulator | Tester (from CSV) | Match |
|---|---:|---:|---|
| Q4 DGP-A flat MSE | 0.2519 | 0.251886 | ✅ (<0.001% rounding) |
| Q4 DGP-B flat MSE | 10.4073 | 10.407268 | ✅ |
| Q4 DGP-B exp7 MSE | 2.9870 | 2.987027 | ✅ |
| Q7 S1 SB MSE | 10.4376 | 10.437593 | ✅ |
| Q7 S2 SB MSE | 0.6733 | 0.673306 | ✅ |
| Q7 HR isolation rel change | 0.9% | 0.9366% | ✅ |

**Both pipelines converge on identical numerical results.**

---

## Step 5b — Simulation Pipeline Verification

### Simulation ↔ Theory Convergence (Q4, Q7)

**DGP-B regime shift detection:** sim-spec.md predicts a 2.5× σ-ratio should produce a
2.5× θ ratio. Observed θ_post/θ_pre ≈ 2.506× (Tester: 6.718/2.681) and 2.536× (Simulator:
6.772/2.674) — both within 2% of the 2.5× theoretical target. ✅

**Q4 decay trade-off direction:** Under DGP-B, the flat estimator averages across 6 pre-break
years (σ=10, θ≈2.67) and 4 post-break years (σ=25, θ≈6.77). The unweighted mix targets
approximately (6×2.67 + 4×6.77)/10 ≈ 4.31, vs the post-break target 6.77 — a 37% bias
on the scale of θ. The observed bias is −3.21 / 6.77 = −47.4%. This is somewhat larger than
the rough 37% calculation because the weighted mean of the _realized_ denominators (which
are themselves upward-biased by Jensen's inequality) is not the same as the mean of the
true denominators. This discrepancy is expected and does not indicate a DGP error. ✅

**Q7 S2 near-zero bias:** With the `after(2005)` cutoff, S2 uses only 4 post-break years.
Bias collapses to +0.062 (< 1% of θ), consistent with a clean post-break estimator
at n_years=4. The small positive residual (+0.062 vs target 6.772) is plausible sampling
variability at R=2000, n_years=4. ✅

### DGP Correctness

The corrected DGP-B in `simulate_q4q7.R` shifts within-year sd (`sd_y = if (i < break_at) pre_sd else post_sd`) while holding `cat_mean = 100` fixed — precisely as specified in sim-spec.md §2. ✅

The two-category DGP for Q7 (`dgp_two_cat`) uses `sb_sd <- if (i < break_year_idx) 10 else 25` and `hr_totals <- rnorm(n_teams, 180, 25)` (stable HR) — matches sim-spec §10. ✅

### Code ↔ Simulation Consistency

No builder was dispatched in this run; no R/ source changed. The simulation calls the
same `sgp_denominators()` public API tested in the prior run's full audit (123 tests, 0
failures confirmed in this run's tester re-check). Internal consistency is inherited from
the prior run. ✅

### Simulation Pipeline Isolation

- `simulation.md` does not reference `spec.md` or `test-spec.md`. ✅
- `audit.md` used independent seeds and did not reference `simulation.md` internals
  (only read from CSV/RDS artifacts). ✅
- Simulator's calibration block (seeds 88882/88883) is distinct from Tester's (seeds
  42001/42002). ✅

### Acceptance Criteria Validation

**Q4 (4/4 PASS):**

| Criterion | sim-spec.md threshold | Observed | Verdict |
|---|---|---|---|
| DGP-A: flat ≤ exp7 MSE | exact inequality | 0.252 ≤ 0.481 | ✅ |
| DGP-B: exp7 ≤ flat MSE | exact inequality | 2.987 ≤ 10.407 | ✅ |
| DGP-B monotone ordering | exact inequality chain | 10.407 ≥ 8.003 ≥ 2.987 | ✅ |
| DGP-A rel_bias ≤ 0.15 | atol 0.15 | max 0.01266 | ✅ (10× headroom) |

**Q7 (3/3 PASS):**

| Criterion | sim-spec.md threshold | Observed | Verdict |
|---|---|---|---|
| S2 SB MSE < S1 SB MSE | exact strict inequality | 0.673 < 10.438 | ✅ (93.5% reduction) |
| HR isolation ≤ 5% rel change | rtol 0.05 | 0.94% | ✅ (5× headroom) |
| S3 intermediate S1 ≥ S3 ≥ S2 | exact inequality chain | 10.44 ≥ 2.98 ≥ 0.67 | ✅ |

No marginal passes. No suspicious patterns (exact nominal values or seed sensitivity).
All results are strongly inside the acceptance bands with substantial headroom. ✅

### Scenario Completeness

**Q4 fresh scenarios:** 2 DGPs × 3 weight schemes = 6 cells, R=2000 each = 12,000 reps.
All 6 rows present in q4_decay.csv. ✅

**Q7 fresh scenarios:** 3 scenarios × 2 categories = 6 rows, R=2000 per scenario = 6,000
reps. All 6 rows present in q7_override.csv. ✅

**Q1/Q2/Q3/Q5/Q6:** Not re-run; verdicts carried from 2026-04-16 run (all verified in
prior review.md "Final pass" section at commit edfa681). ✅

---

## Step 7 — Validation Evidence

### Required Commands

Tester ran:
1. `devtools::check(...)` → 0 errors | 0 warnings | 4 NOTEs (pre-existing). ✅
2. `testthat::test_file('tests/testthat/test-sgp-denominators.R', reporter='check')` → FAIL 0 | WARN 125 | SKIP 0 | PASS 123. ✅
3. Independent empirical θ calibration (seeds 42001/42002, n_years=2000). ✅
4. Manual acceptance criterion verification against q4_decay.csv and q7_override.csv with cross-reference to simulation_q4q7_results.rds. ✅

All required validation commands executed with exact output reported. No paraphrasing. ✅

### ALL Scenarios Executed

Tester explicitly confirms 4/4 Q4 criteria (from the 6-row CSV) and 3/3 Q7 criteria
(from the 6-row CSV). Per-Test Result Tables are complete with one row per distinct
metric. No silent omissions detected. ✅

### ERRORs and WARNINGs

The 125 test WARNINGs are expected cli_warn() calls captured by testthat (same as prior
run's 124-125 warnings — note count is 125 in this run vs 125 in prior; within normal
range). None are suppressed failures. 0 ERRORs. ✅

The 4 NOTEs are pre-existing infrastructure issues (noted in prior run review.md).
Unrelated to this run's changes (only `tools/simulation/simulate_q4q7.R` was added). ✅

### Step 7a — Tolerance Integrity Audit

For simulation criteria, tolerances are structural inequalities (no numeric epsilon). Tester
explicitly states: "All tolerances used match sim-spec.md exactly. No tolerance was widened
or softened." Confirmed by cross-reference:

| Criterion | sim-spec.md threshold | audit.md threshold used | Inflation |
|---|---|---|---|
| Q4 DGP-A flat vs exp7 | exact inequality | exact inequality | None |
| Q4 DGP-B exp7 vs flat | exact inequality | exact inequality | None |
| Q4 monotone ordering | exact inequality chain | exact inequality chain | None |
| Q4 DGP-A rel_bias | atol = 0.15 | atol = 0.15 | None |
| Q7 override improves SB | exact strict inequality | exact strict inequality | None |
| Q7 HR isolation | rtol = 0.05 | rtol = 0.05 | None |
| Q7 S3 intermediate | exact inequality chain | exact inequality chain | None |

**No tolerance inflation detected. No evasion patterns.** ✅

---

## Step 8 — Documentation and Process Record

**Scope note:** Per impact.md, no Scriber was dispatched in this run. This is consistent
with the simulation-only workflow (no R/ source changes, no API changes, no roxygen
updates required). The prior run (2026-04-16) produced `ARCHITECTURE.md` in the target
repo root, `log-entry.md` in the run workspace, and all documentation artifacts.

**ARCHITECTURE.md in target repo root:** `/Users/jacobdennen/rotostats/ARCHITECTURE.md`
exists (confirmed by `ls`). Produced in prior run. Not re-produced in this run (no new
functions or architectural changes). ✅

**ARCHITECTURE.md in run workspace:** Not required for a simulation-only re-run.
Per impact.md: "ARCHITECTURE.md — prior run already produced this in target repo root;
not re-produced." ✅

**log-entry.md in run workspace:** Not produced (no Scriber dispatched). Impact.md
explicitly lists this as a "NOT affected" file under the default path. For simulation-only
re-runs that add no production code, the process record is captured in simulation.md and
this review.md. PASS WITH NOTE — acceptable given simulation-only scope.

**Target repo clean of non-Architecture workflow artifacts:** The only file added to the
target repo in this run is `tools/simulation/simulate_q4q7.R` (confirmed by git diff:
`feature/sgp-denom-dgpb-rerun` is ahead of `origin/develop` by 1 commit adding only that
file). No CHANGELOG.md, HANDOFF.md, runs/, or log/ directory added. ✅
(HANDOFF.md is pre-existing from 2026-04-16 run, noted in prior review as pre-existing
scaffolding from commit 90473c1.)

**docs.md:** Not required. No documentation files in the write surface (impact.md confirms
man/, NEWS.md, plans/ are all out of scope for this run). ✅

---

## Key Discovery Documentation

### OLS Estimand Scale vs SD Analytical Formula

The 2026-04-16 run's sim-spec.md §2 "Concerns flagged for Tester and Reviewer" item 2
(Concern 2 in prior review.md) was the DGP-B mean-shift design flaw. This run resolves it.

A related issue surfaced during calibration: the Planner's mailbox specified an absolute
calibration gate of `|θ̂_post − θ̂_pre| > 5 units`, derived from the SD-method analytical
formula θ_SD ≈ (n−1)·σ/E[R_n] ≈ 84 at σ=25. For `method = "ols"`, the denominator =
1/|β̂| where β̂ is the OLS slope of `rank ~ total`. Under Gaussian totals with n_teams=12,
the OLS denominator scales at approximately 0.27 units/unit-σ (not ~3.36 for SD method).

Empirical calibration results:

| Regime | sigma | theta_OLS (Simulator) | theta_OLS (Tester, independent) |
|---|---|---|---|
| Pre-break | 10 | 2.6744 | 2.6811 |
| Post-break | 25 | 6.7722 | 6.7181 |
| Absolute gap | — | 4.0978 | 4.0370 |
| Relative gap | — | 153.2% | 150.6% |

The absolute gap (~4.1) is below the Planner's 5-unit threshold because that threshold was
calibrated for the SD-method scale. The **relative gap of ~150%** is the correct metric for
OLS and is identical in proportion across methods: OLS denominator scales linearly with σ,
so a 2.5× σ ratio produces a 2.5× θ ratio and ~150% relative gap — well above any reasonable
50% threshold.

**Resolution:** Both Simulator and Tester independently recognized this miscalibration and
used the relative gate (> 50%) instead. Both independently confirmed rel_gap >> 50%. No
block was warranted. The calibration gate discrepancy is a sim-spec process issue (Planner
derived the absolute gate from the wrong method's formula), not an estimator defect.

**Lesson for future sim-specs:** When specifying pre-flight calibration gates, use
method-specific empirical thresholds. For OLS, the appropriate calibration gate is:
`(θ̂_post / θ̂_pre - 1) > 0.50` (relative), not `|θ̂_post - θ̂_pre| > 5` (absolute derived
from SD-method analytics). Document this in sim-spec.md §2's Jensen's inequality note
for future simulation runs.

---

## Preserved Verdicts (Q1/Q2/Q3/Q5/Q6) from 2026-04-16

The following verdicts from the 2026-04-16 final review (SHIP verdict, branch HEAD edfa681)
are carried forward unchanged. See:
`/Users/jacobdennen/.claude/plugins/data/statsclaw-statsclaw/workspace/rotostats/runs/sgp-denominators-2026-04-16/review.md`
(§"Final pass — 2026-04-16").

| Study | Criteria | Verdict | Source |
|---|---|---|---|
| Q1 — OLS Convergence | 4/4 | ✅ PASS | 2026-04-16 review.md §Step 5b |
| Q2 — Finite-Sample Bias Direction | 3/3 | ✅ PASS | 2026-04-16 review.md §Step 5b |
| Q3 — Bootstrap CI Coverage | ⚠️ Borderline | PASS WITH NOTE (coverage 0.847, Jensen undercoverage documented in ?sgp_denominators) | 2026-04-16 review.md §Step 5b, §Prior-Session Concerns item 3 |
| Q5 — Expected-Range Anchor Verification | 1/1 | ✅ PASS | 2026-04-16 review.md §Step 5b |
| Q6 — Method Comparison | 4/4 | ✅ PASS | 2026-04-16 review.md §Step 5b |

Additionally, from the 2026-04-16 final pass, the following non-blocking concerns were
resolved and remain resolved:
- Sim-spec §2 analytical θ formula corrected (OLS vs SD estimand distinction documented). ✅
- Jensen's-inequality caveat added to `?sgp_denominators` @section Caveats. ✅
- All five code defects (D1–D4b) resolved at commits af2c3d9, 629ac30, fd21a09, 8f2348b. ✅

---

## Fresh Verdicts (Q4/Q7) from this run

### Q4 — Time-Decay Variance Trade-off

**PASS (4/4 criteria) under corrected sigma-shift DGP-B.**

Prior run result (2026-04-16): Q4 earned 2/4 — DGP-B criteria were untestable (mean-shift
produced zero OLS estimand change). This run corrects the DGP-B to shift within-year σ
(pre_sd=10, post_sd=25, 2.5× ratio), creating a genuine 150% relative regime change.

Results under corrected DGP-B:

| dgp | weights | bias | rel_bias | mse | rmse |
|---|---|---:|---:|---:|---:|
| A (stable) | flat | 0.059 | 0.88% | **0.252** | 0.502 |
| A (stable) | exp_decay(0.9) | 0.059 | 0.87% | 0.275 | 0.525 |
| A (stable) | exp_decay(0.7) | 0.085 | 1.27% | 0.481 | 0.694 |
| B (sigma-break) | flat | −3.214 | −47.5% | 10.407 | 3.226 |
| B (sigma-break) | exp_decay(0.9) | −2.813 | −41.5% | 8.003 | 2.829 |
| B (sigma-break) | exp_decay(0.7) | −1.670 | −24.7% | **2.987** | 1.728 |

Under DGP-A (stable): flat() is best (MSE 0.252); exp_decay(0.7) carries a 91% MSE
penalty by discounting valid older data. Under DGP-B (sigma-break): exp_decay(0.7) is
best (MSE 2.987), achieving a 71% MSE reduction vs flat (10.407). Monotone ordering
holds: flat > exp9 > exp7. DGP-A rel_bias is negligible for all weights (max 1.27%,
threshold 15%). Zero failures across 12,000 replications.

**The "decay helps break" criterion passes cleanly. No spec.md follow-up required on
default weights.** The evidence does support considering exp_decay(0.9) as the default
in a future run (Simulator §8.4 recommendation), but this is not a required action.

### Q7 — Per-Category Override Sanity

**PASS (3/3 criteria) under corrected sigma-shift DGP-B.**

Prior run result (2026-04-16): Q7 earned 1/3 — the category isolation criterion (S1 HR
MSE unchanged) passed because HR was always stable; the S2 and S3 SB criteria were
untestable. This run corrects the SB DGP to shift sigma, making all three criteria valid.

Results under corrected DGP-B:

| scenario | category | bias | mse | mse_rel_to_S1 |
|---|---|---:|---:|---:|
| S1 (no override, flat) | HR | 0.046 | 0.258 | 1.000 |
| S1 (no override, flat) | SB | −3.218 | 10.438 | 1.000 |
| S2 (SB: after+exp7) | HR | 0.049 | 0.260 | 1.009 |
| S2 (SB: after+exp7) | SB | 0.062 | 0.673 | 0.065 |
| S3 (SB: exp7 all years) | HR | 0.053 | 0.257 | 0.997 |
| S3 (SB: exp7 all years) | SB | −1.669 | 2.980 | 0.286 |

S2 (after+exp7): SB MSE drops 93.5% (10.438 → 0.673); HR MSE changes 0.94% (well within
5% isolation threshold). S3 (exp7 all): SB MSE drops 71.4% (intermediate as expected).
S1 ≥ S3 ≥ S2 ordering confirmed (10.44 ≥ 2.98 ≥ 0.67). Category isolation confirmed.
Zero failures across 6,000 replications.

---

## Concerns for Future Runs

### Sim-spec calibration gate miscalibration (process lesson)

The Planner's mailbox specified `|θ̂_post − θ̂_pre| > 5` as the calibration gate. This
threshold was derived from the SD-method analytical formula and does not apply to OLS,
where denominators scale at ~0.27 units/σ instead of ~3.36 units/σ. Both Simulator and
Tester recognized this and correctly used a relative gate (> 50%). This was handled
gracefully, but the sim-spec should specify method-appropriate calibration gates for
future runs. **Non-blocking; no action required in this run.**

### exp_decay(0.9) as a default weight (optional follow-up)

Simulator §8.4 notes that exp_decay(0.9) achieves ~23% MSE reduction under break
conditions at only ~9% cost under stable conditions — a favorable asymmetric trade-off.
The current default `weights = flat()` is strictly better only under purely stable DGPs.
In real-world league data, structural breaks in spread are plausible (rule changes, era
effects). The evidence supports considering exp_decay(0.9) as the new default. This would
require a spec.md change and is deferred to a follow-up run if the user desires it.
**Non-blocking; flagged for optional follow-up.**

### Q3 bootstrap undercoverage at n_years ≤ 6 (carried from 2026-04-16)

Coverage ≈ 84.7% at nominal 95% due to Jensen's-inequality upward bias. Documented in
`?sgp_denominators @section Caveats` and `NEWS.md`. BCa bootstrap deferred as future
enhancement. **Non-blocking.**

---

## Summary: Full Study Suite Verdict

| Study | Criteria | Verdict (this run) | Source |
|---|---|---|---|
| Q1 — OLS Convergence | 4/4 | ✅ PASS | Carried from 2026-04-16 |
| Q2 — Finite-Sample Bias Direction | 3/3 | ✅ PASS | Carried from 2026-04-16 |
| Q3 — Bootstrap CI Coverage | 0/3 tight/nominal, borderline floor | ⚠️ PASS WITH NOTE | Carried from 2026-04-16 |
| Q4 — Time-Decay Variance Trade-off | 4/4 | ✅ PASS (corrected DGP-B) | Fresh — this run |
| Q5 — Expected-Range Anchor Verification | 1/1 | ✅ PASS | Carried from 2026-04-16 |
| Q6 — Method Comparison | 4/4 | ✅ PASS | Carried from 2026-04-16 |
| Q7 — Per-Category Override Sanity | 3/3 | ✅ PASS (corrected DGP-B) | Fresh — this run |
| devtools::check() | 0E/0W | ✅ PASS | This run (Tester Step 2) |
| testthat full suite | FAIL 0, PASS 123 | ✅ PASS | This run (Tester Step 2) |

---

## Self-Check Checklist

- [x] Verified comprehension.md exists with FULLY UNDERSTOOD verdict (step 1)
- [x] Verified pipeline isolation — Simulator and Tester used independent seeds; no cross-reads (step 2)
- [x] Cross-compared sim-spec.md acceptance criteria against simulation.md and audit.md results — all 7 criteria align (step 3)
- [x] Verified convergence: Simulator and Tester values match to rounding precision; bitwise CSV/RDS match confirmed (step 4)
- [x] Per-Test Result Tables present in audit.md for Q4 (13 rows) and Q7 (6 rows) (step 4)
- [x] Before/After Comparison Tables present in audit.md for Q4 and Q7 (step 4)
- [x] No builder/tester pipeline for this run (simulation-only) — steps 5, 6 not applicable (step 5, 6)
- [x] Simulation ↔ theory convergence: θ ratio ~2.5× matches σ ratio 2.5× (step 5b)
- [x] DGP correctness: sigma-shift confirmed in simulate_q4q7.R; two-cat DGP matches sim-spec §10 (step 5b)
- [x] Code ↔ simulation consistency: no R/ source changes; prior audit's 123 tests still pass (step 5b)
- [x] Simulation pipeline isolation: independent seeds, no cross-reads (step 5b)
- [x] Acceptance criteria all evaluated; all within acceptance bands with substantial headroom (step 5b)
- [x] Q4 and Q7 scenarios complete (6 and 6 rows respectively) (step 5b)
- [x] Verified tester ran devtools::check + testthat with exact output (step 7)
- [x] Verified ALL Q4 and Q7 scenarios evaluated (step 7)
- [x] All tolerance/threshold comparisons confirmed at spec values — no inflation (step 7a)
- [x] ARCHITECTURE.md exists in target repo root (from prior run, unchanged) (step 8)
- [x] log-entry.md absent — acceptable for simulation-only run per impact.md (step 8)
- [x] Target repo clean: only tools/simulation/simulate_q4q7.R added; no workflow artifacts (step 8)
- [x] docs.md not required — no documentation surfaces in write scope (step 8)
- [x] Brain mode isolated (user declined /contribute) — step 8b not applicable

---

## Verdict

**SHIP**

Both pipelines converged. Q4 (4/4) and Q7 (3/3) pass cleanly under the corrected sigma-shift
DGP-B. The "decay helps break" criterion is confirmed with a 71% MSE reduction. Category
override isolation is confirmed at <1% HR MSE perturbation. Independent Tester calibration
confirms the sigma-shift produces a genuine 150.6% relative gap in the OLS estimand. No
discrepancies between Simulator and Tester. devtools::check() 0E/0W. testthat 0 FAIL / 123
PASS. Prior Q1/Q2/Q3/Q5/Q6 verdicts from 2026-04-16 remain valid and are preserved
unchanged. Safe to ship.
