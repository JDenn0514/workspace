# Leader Impact Assessment

Request: replacement-dgp-e-calibration-2026-04-17
Authored by: leader
Date: 2026-04-18

## Summary

Simulation-pipeline-only calibration of DGP-E so Study E of the
`replacement_level()` Monte Carlo harness meets its acceptance thresholds
(median_abs_rank_diff ≤ 2 AND pct_within_2 ≥ 0.90) at R=500 reps. No
changes to `R/`, function signatures, or user-facing docs.

## Affected Surfaces

### In target repo (write candidates for Simulator worktree only)

| File | Role | Expected change |
| --- | --- | --- |
| `inst/simulations/dgp/dgp_e.R` | DGP-E generator — focal pitcher + pool | YES — focal-pitcher parameters and/or pool noise tightened per Planner's chosen option. L1–L180 in scope. |
| `inst/simulations/replacement-mc.R` | Harness driver | MAYBE — only if study-selector / seed-offset plumbing needs a tiny tweak to support "Study E only" re-run. Keep changes minimal. |

### Frozen — MUST NOT be touched

- `R/replacement.R` (and all other `R/*.R`) — hard constraint of this request
- `specs/spec-replacement.md` — frozen (public spec)
- `tests/testthat/*.R` — unit-test surface, unchanged (Tester will re-run to confirm FAIL=0)
- `man/*.Rd`, `NAMESPACE`, `DESCRIPTION` — no interface change
- Other DGPs: `dgp_a.R`, `dgp_c.R`, `dgp_d.R` — untouched
- `NEWS.md` — no user-visible change

### In workspace repo (planner/simulator/tester/reviewer artifacts)

| File | Role | Owner |
| --- | --- | --- |
| `runs/<this>/sim-spec.md` | Successor sim-spec with patched §"Study E" DGP-E definition + changelog entry | planner |
| `runs/<this>/test-spec.md` | Successor test-spec; Study E acceptance criteria explicit; A/B/C/D referenced by path | planner |
| `runs/<this>/simulation.md` | Fresh Study E tables; A/B/C/D referenced by path to replacement-2026-04-16 | simulator |
| `runs/<this>/audit.md` | Tester report on boundary determinism + Study E acceptance | tester |
| `runs/<this>/review.md` | Unified successor: A/B/C/D inherited; Study E fresh | reviewer |
| `runs/<this>/shipper.md` | Commit + push + PR log | shipper |

## Risk Areas

1. **Pool-composition variance dominates residual error.** Prior focal at
   ERA=4.70/WHIP=1.40/IP=165 already produces deterministic focal stats —
   residual rank variance is entirely driven by between-rep variation in
   non-focal players, which perturbs the replacement-band composition.
   Planner's three levers (focal distance from pool mean, pool σ, fixed
   pool) all target this. Most expected-to-work: **fixed pool** (option 3)
   — eliminates between-rep pool variance entirely, leaving only seed-level
   RNG in the focal-rank measurement. Option 2 (lower σ) is second-best.
   Option 1 (focal further below pool) may overshoot into "obviously
   replacement" territory and lose calibration meaning.

2. **Seed reproducibility.** Seed base `20260416` + study_offset + rep must
   be preserved for Studies A/B/C/D to remain reproducible by reference.
   Study E's seed stream changes ONLY if the DGP function's internal RNG
   consumption order changes — Planner should flag any such change so
   simulator can add a deterministic test that an unchanged-DGP call with
   study-E seed produces the old rank_diff=3 baseline (regression guard).

3. **Pipeline-isolation cross — prior precedent.** The original fix
   (commit `21270fb`) crossed isolation because builder touched both
   `R/replacement.R` AND `inst/simulations/dgp/dgp_e.R`. This run is the
   clean version: simulator (only) touches DGP-E. Reviewer should confirm
   isolation holds and no `R/` files changed.

4. **Tester acceptance on R=500 may still fail.** If option 1/2/3 still
   can't push pct_within_2 ≥ 0.90, tester MUST NOT relax thresholds.
   Acceptance is BLOCK → respawn planner with revised DGP option, or if
   three planner rounds exhausted, flag as threshold-revision follow-up
   and emit a clear mailbox note for user decision.

## Required Teammates

| Teammate | Required | Worktree | Notes |
| --- | --- | --- | --- |
| planner | YES | yes | Patch sim-spec §"Study E" DGP-E + test-spec Study E acceptance; changelog entry |
| builder | **NO** | — | Frozen. `R/` untouched per hard constraint. |
| simulator | YES | yes | Patch `inst/simulations/dgp/dgp_e.R` and re-run Study E at R=500 |
| tester | YES | no | Deterministic boundary check + Study E acceptance; full `devtools::test()` for FAIL=0 regression; `R CMD check` |
| scriber | **NO** | — | Deferred to follow-up iff thresholds revised |
| distiller | **NO** | — | BrainMode=isolated |
| reviewer | YES | no | Unified successor review.md |
| shipper | YES | no | Commit, push, PR base=develop, workspace sync |

## Pipeline Isolation Plan

- Planner sees request.md, impact.md, prior run artifacts.
- Simulator sees sim-spec.md ONLY (not spec.md, not test-spec.md). Its
  worktree write surface is `inst/simulations/dgp/dgp_e.R` and optionally
  `inst/simulations/replacement-mc.R`. NOT `R/`.
- Tester sees test-spec.md ONLY. Validation surface: devtools::test,
  R CMD check, and a scripted Study-E re-run assertion.
- Reviewer sees ALL artifacts (this run + cross-references to
  replacement-2026-04-16).

## Dependency on Prior Run

This run **inherits by reference** from
`/Users/jacobdennen/.claude/plugins/data/statsclaw-statsclaw/workspace/rotostats/runs/replacement-2026-04-16/`:

- `simulation.md` §§ Study A, B, C, D (verdicts unchanged)
- `review.md` § Load-bearing acceptance criteria (verdicts unchanged)
- `sim-spec.md` §§ 1–6 (structure carried forward; §"Study E" DGP-E replaced)
- `test-spec.md` (Study E acceptance section patched; other test IDs unchanged)

Simulator MUST include explicit path references in the successor
`simulation.md` so a reviewer/reader can trace A/B/C/D verdicts without a
re-run.

## Ship Plan

- Feature branch: `feature/replacement-dgp-e-calibration` (cut by shipper
  from `develop`).
- Commit scope: `inst/simulations/dgp/dgp_e.R` (+ optional harness tweak).
- PR base: `develop`. Title: `sim(replacement): recalibrate DGP-E focal
  pitcher to pass Study E thresholds`.
- Workspace sync: copy run log + CHANGELOG + HANDOFF entries.
