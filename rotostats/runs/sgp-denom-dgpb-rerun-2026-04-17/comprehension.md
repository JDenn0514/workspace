# Comprehension Record: sgp-denom-dgpb-rerun-2026-04-17

**Planner:** claude-opus-4-7
**Date:** 2026-04-17
**Run type:** Simulation-only sim-spec patch (no spec.md / test-spec.md change)
**HOLD rounds used:** 0
**Verdict:** FULLY UNDERSTOOD

---

## 0a — Input Materials Inventoried

| Material | Type | Relevance |
|---|---|---|
| `request.md` (this run) | Leader artifact | Primary scope + hard constraints |
| `impact.md` (this run) | Leader artifact | Confirms no target-repo source changes |
| `plans/sgp-cleanup.md` §"R2 — sgp-denom-dgpb-rerun" (rotostats repo) | Plan doc | Authoritative scope for R2 |
| `runs/sgp-denominators-2026-04-16/simulation.md` §4, §7 | Prior run artifact | Q4/Q7 prior findings (DGP-B mean-shift failed to test criterion) |
| `runs/sgp-denominators-2026-04-16/review.md` (Concerns) | Prior run artifact | Concern 2: DGP-B design flaw identified + remediation direction |
| `runs/sgp-denominators-2026-04-16/sim-spec.md` | Prior run artifact | Already contains a seeded DGP-B σ-shift sketch (pre_sd=10, post_sd=25) |
| `runs/sgp-denominators-2026-04-16/spec.md` | Prior run artifact (reference-only) | Not modified this run |
| `runs/sgp-denominators-2026-04-16/test-spec.md` | Prior run artifact (reference-only) | Not modified this run |

No uploaded PDFs/papers for this request. All inputs are internal artifacts.

---

## 0b — Internalized Content

### Core requirement (restated in my own words)

The Q4 (Time-Decay Variance Trade-off) and Q7 (Per-Category Override Sanity Check)
simulation studies evaluate whether exponentially decaying weights and per-category
overrides reduce MSE under a structural-break regime. Both studies use DGP-B (for Q4)
or a two-category DGP (for Q7) with a within-sample break at year index 7 of 10. The
previous run's DGP-B shifted the **mean** of team totals across the break while holding
σ fixed. This produces zero change in the OLS denominator because the OLS slope of
`rank ~ total` is mean-invariant: sorting by total gives the same rank structure whether
totals are centered at 100 or 500 — only the within-year spread matters. Therefore the
"decay helps under break" criterion and the "SB override improves SB MSE" criterion
were untestable, not refuted. This run corrects DGP-B to shift within-year σ across the
break — producing a genuine denominator regime change — and re-runs only Q4 and Q7.

### Key formulas (restated and verified)

**OLS estimand for DGP-A / DGP-B (single regime)** — from sim-spec §2:

```
E[β̂_OLS] ≈ E[R_n] / ((n−1) × σ)
θ_OLS     ≈ (n−1) × σ / E[R_n]
```

For n_teams = 12:
- `E[R_12] = 3.2587` (from `expected_range_normal(12)`)
- θ_OLS(σ=10) ≈ 11 × 10 / 3.2587 ≈ **33.75**
- θ_OLS(σ=15) ≈ 11 × 15 / 3.2587 ≈ **50.62**
- θ_OLS(σ=25) ≈ 11 × 25 / 3.2587 ≈ **84.39**

**Jensen-gap caveat:** The analytical formula gives the asymptotic estimand; at finite
n_years there is an additional upward bias (Q2). For Q4's MSE comparisons this cancels
across weight schemes (all use the same OLS estimator), so the decay-vs-flat ordering is
not confounded.

**DGP-B post-break target:** Estimator's target under DGP-B is the post-break σ regime
(the current regime). θ_B = θ_OLS(post_sd). A flat-weight estimator averages across
both regimes; decay weighting downweights pre-break.

### Why this works (intuition)

Under OLS on `rank ~ total` within a year, ranks are deterministic given totals (sort
order). The slope depends on how spread the totals are: tightly clustered totals
(small σ) give a steeper rank-vs-total line → large β̂ → small denominator. Widely
spread totals (large σ) give a shallower line → small β̂ → large denominator. Shifting
the league **mean** translates totals uniformly, preserving spread and thus the slope —
this is why a mean shift leaves θ unchanged. Shifting **σ** expands or contracts the
spread, changing the slope and therefore θ.

For n=12, σ doubling (10 → 25, factor 2.5) produces θ scaling by 2.5×. The MSE
of a flat-weight estimator across a pre/post break is bias-dominated: it targets a
weighted average of θ_pre and θ_post (roughly 60/40 by year count) while the estimand
is θ_post. The decay-weight estimator approximates θ_post more closely, reducing bias.

---

## 0c — Self-Test Answers

**Q1. Restate the core requirement in one paragraph without looking at the source.**
Re-run Q4 and Q7 of the sgp_denominators Monte Carlo study with DGP-B corrected to
shift within-year σ at break_at=7 (not the mean). Produce a fresh simulation.md
referencing prior Q1/Q2/Q3/Q5/Q6 verdicts by path. Tester must first confirm the
corrected DGP-B produces a measurable pre/post θ shift via empirical calibration
before evaluating the Q4/Q7 acceptance criteria. No changes to R/ source, spec.md, or
test-spec.md. Seed strategy matches 2026-04-16 for reproducibility.

**Q2. Write out every formula from memory and explain each term.**
Done above under "Key formulas." θ_OLS ≈ (n-1)σ / E[R_n]. E[R_n] is the expected
range of n i.i.d. standard normals. Formulas verified against sim-spec §2 and match
exactly.

**Q3. Any symbols/terms I cannot precisely define?** None. All symbols are inherited
from the prior run's sim-spec which uses consistent notation.

**Q4. Any steps where I would need to make a judgment call not supported by the source?**
Yes — one: **choice of exact σ values for DGP-B**. The plan text says
"e.g., pre-break σ = 15, post-break σ = 25 (planner picks exact parameters to produce a
clear, statistically detectable signal at n_years = 10, R = 2000)". The prior run's
Planner-patch (end of 2026-04-16 run, appended to sim-spec.md §2) already seeded
pre_sd = 10, post_sd = 25 as the corrective target. I evaluate both candidates:

| Option | pre/post σ | Ratio | θ_pre | θ_post | θ gap | Verdict |
|---|---|---|---|---|---|---|
| A (plan-text example) | 15 / 25 | 1.67× | 50.62 | 84.39 | 33.77 | Moderate signal |
| B (seeded in sim-spec) | 10 / 25 | 2.5×  | 33.75 | 84.39 | 50.64 | Strong signal |

**Decision: Option B (pre_sd = 10, post_sd = 25).** Rationale:
(i) the plan text says "e.g." — leaves parameter choice to Planner;
(ii) Option B produces a 50-unit θ gap at n=12 which dominates per-rep sampling noise
at R=2000 (empirical σ of β̂ per replicate is O(1) for these settings, so
sampling SE of mean θ̂ across 2000 reps is O(10^-1) — easily detectable);
(iii) it matches the prior Planner-patch already committed to the sim-spec seed of
this run, preserving continuity;
(iv) it produces a clearer diagnostic in the downstream Q4 MSE table (if decay
still doesn't help with a 2.5× shift, the decay-default recommendation is much more
confidently invalidated). Explicit assumption declared: choosing stronger signal
over plan-text example.

**Q5. "Why does this work?"** Explained under "Why this works (intuition)" above.

**Q6. Implicit assumptions the source relies on?**
- Normality of totals: DGP-A and DGP-B use `rnorm()` — θ_OLS formula assumes Gaussian.
  The prior run's Q6 heavy-tailed scenarios already tested robustness; not re-tested here.
- Independence across years: the DGP generates each year's totals independently; there
  is no autocorrelation in the structural-break design. This is the intended simplification.
- Master seed reproducibility: `set.seed(2026041601)` + `sample.int()` pattern depends on
  R's Mersenne-Twister default; sim-spec §3 locks this in.

---

## 0d — Verdict

**FULLY UNDERSTOOD.**

No HOLDs required. The only judgment call (σ parameter choice) is explicitly delegated
to Planner by the plan text, and I have documented my decision (Option B: pre=10,
post=25) with rationale above.

---

## Pipeline-isolation check

This run touches only `sim-spec.md`. `spec.md` and `test-spec.md` are seeded from the
prior run for reference only and will not be modified (hard constraint from request.md).
The sim-spec patch does NOT alter the estimator interface, scenario grid, metrics,
acceptance criteria thresholds, or seed strategy — only the DGP-B definition (already
sketched in the seeded sim-spec) and an added changelog note.

---

## Handoff plan

**To Simulator:** Use sim-spec.md (patched). DGP-B in §2 uses pre_sd=10, post_sd=25.
Q4 §7 and Q7 §10 are unchanged in structure; simulator re-runs those 2 studies only.
Q1/Q2/Q3/Q5/Q6 are referenced by path to 2026-04-16 CSVs; do NOT re-run them.
Seed: master 2026041601 + study offsets 4 and 7 as before.

**To Tester:** Before evaluating Q4/Q7 acceptance, perform empirical θ calibration
under the new DGP-B:
1. Run DGP-B once with n_years = 2000 (large), pre_sd=10, compute θ̂_pre.
2. Run DGP-B once with n_years = 2000 (large), post_sd=25, compute θ̂_post.
3. Confirm |θ̂_post − θ̂_pre| / SE_estimate > 5 (a conservative threshold — analytical
   gap is 50.64; at n_years=2000 the SE is << 1, so z-score will be very large if DGP
   is correct). If the gap is not detectable, BLOCK and re-route to Planner.

**To Reviewer:** Keep Q1/Q2/Q3/Q5/Q6 verdicts from 2026-04-16/review.md unchanged.
Replace only §§ on Q4 and Q7 with fresh evaluations from this run's audit.md.
If Q4 decay-helps-break criterion FAILS even with corrected DGP, flag for a
candidate spec.md default-weights change (as follow-up request — do NOT modify
spec.md in this run).
