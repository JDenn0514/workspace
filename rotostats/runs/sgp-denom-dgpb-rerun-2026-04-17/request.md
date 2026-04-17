# Request: sgp-denom-dgpb-rerun-2026-04-17

**Target repo:** JDenn0514/rotostats
**Workspace repo:** JDenn0514/workspace
**Feature branch:** feature/sgp-denom-dgpb-rerun
**Base branch:** develop
**PR base:** develop
**Workflow:** Simulation-only (Workflow 12 variant — re-run only)
**Pipeline:** Planner (sim-spec patch) -> Simulator -> Tester -> Reviewer -> Shipper
**BrainMode:** isolated (user explicitly declined /contribute)

## Summary

Re-run Q4 and Q7 of the `sgp_denominators()` Monte Carlo validation under a corrected DGP-B
that shifts within-year sigma (not mean) across the structural break. No changes to R/ source,
spec.md, or test-spec.md.

## Authoritative Scope

See `plans/sgp-cleanup.md` section "R2 — sgp-denom-dgpb-rerun" (lines 185-296) in the rotostats
repo (commit ce1fbde on develop).

## Prior Run

`/Users/jacobdennen/.claude/plugins/data/statsclaw-statsclaw/workspace/rotostats/runs/sgp-denominators-2026-04-16/`
- `simulation.md` section 4 (Q4 finding) and section 7 (Q7 finding)
- `review.md` "Concerns flagged for Tester and Reviewer" item 2
- `sim-spec.md` section 2 already contains a DGP-B redesign sketch (pre_sd=10, post_sd=25)
  appended at end of that run. Planner must evaluate whether those parameters give a clear
  detectable signal at n_years=10, R=2000, and tune if needed.

## What changes this run

### Planner
- Patch `sim-spec.md` DGP-B definition to sigma-shift at break_at=7. Pre-break sigma = 15,
  post-break sigma = 25 per the plan (Planner may tune for detectable signal; document choice).
- Add a sim-spec changelog note documenting the DGP-B correction and rationale.
- Confirm Q4 and Q7 acceptance criteria and seed strategy are unchanged.

### Simulator
- Re-run Q4 (6 scenarios: 2 DGPs x 3 weight schemes) and Q7 (3 scenarios) ONLY under corrected DGP-B.
- Q1, Q2, Q3, Q5, Q6 are NOT re-run. Reference prior-run verdicts by path.
- Master seed 2026041601 + same study/scenario/rep offsets as 2026-04-16.
- Produce a fresh `simulation.md` that references the prior run's Q1/Q2/Q3/Q5/Q6 verdicts
  by path and reports fresh Q4 and Q7 tables.

### Tester
- Precondition: empirically calibrate theta_pre vs theta_post under the new DGP-B and confirm
  the difference exceeds sampling noise. If not, BLOCK and re-route to Planner.
- Then evaluate Q4 acceptance criterion "under DGP-B, exp_decay(0.7) MSE <= flat MSE" and
  Q7 acceptance criterion "SB override improves SB MSE" cleanly.

### Reviewer
- Merge report: keep Q1/Q2/Q3/Q5/Q6 verdicts from 2026-04-16 unchanged. Replace only Q4 and Q7.
- If decay-helps-break FAILS under the corrected DGP, flag as candidate spec.md change
  (default weights) and file a follow-up request. DO NOT modify spec.md in this run.

### Shipper
- Commit run artifacts to `feature/sgp-denom-dgpb-rerun` and open PR to `develop`.
- No R/ source changes. If Scriber updated roxygen (conditional on decay-default change), include.
- Sync run workspace to JDenn0514/workspace.

## Hard constraints

- No changes to `R/` source code.
- No changes to `spec.md`, `test-spec.md`, or the function's public interface.
- Seed strategy must match 2026-04-16 (master_seed = 2026041601) so shared tables remain reproducible.

## Acceptance criteria

1. Corrected DGP-B produces a statistically detectable pre/post shift in the OLS asymptotic theta
   (documented in simulation.md with empirical calibration).
2. Q4 criterion "under DGP-B, exp_decay(0.7) MSE <= flat MSE" evaluated against the corrected DGP;
   pass or fail reported cleanly with diagnostic interpretation.
3. Q7 criterion "SB override improves SB MSE" evaluated cleanly under the corrected DGP.
4. If decay-helps-break FAILS even under corrected DGP, Reviewer flags as candidate spec.md change
   and files a follow-up request (no in-run spec.md modification).
