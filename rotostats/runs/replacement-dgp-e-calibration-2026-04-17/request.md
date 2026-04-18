# Request

```yaml
RequestID: replacement-dgp-e-calibration-2026-04-17
Package: rotostats
Workflow: 12 (Simulation only — variant, no scriber per user scope)
TargetRepo: JDenn0514/rotostats
TargetCheckout: /Users/jacobdennen/rotostats
WorkspaceRepo: JDenn0514/workspace
BaseBranch: develop
FeatureBranch: feature/replacement-dgp-e-calibration
BrainMode: isolated
ParentRun: replacement-2026-04-16
```

## Scope

Re-run **Study E only** of the `replacement_level()` Monte Carlo validation
under a corrected DGP-E. **NO change to `R/replacement.R`** — simulation
pipeline only.

Authoritative scope: `plans/replacement-cleanup.md` §"R5 —
replacement-dgp-e-calibration".

## Prior-run context

`~/.claude/plugins/data/statsclaw-statsclaw/workspace/rotostats/runs/replacement-2026-04-16/`

- `simulation.md` §"Study E" — current median_abs_rank_diff=3,
  pct_within_2=0.46 (R=500, seed base 20260416)
- `review.md` §"Waiver Assessment" — Study E waived at ship
- `audit.md` §"Post-Fix Re-Audit v2" — DGP-E remediation history
- Commit `21270fb` set current focal pitcher: ERA=4.70, WHIP=1.40, IP=165,
  W=8, K=130 against pool mean ERA ≈ 4.40 — improved but still gates.

## What needs to change (by teammate)

### Planner
Patch `inst/simulations/dgp/dgp_e.R` DGP design AND the successor
`sim-spec.md` §"Study E" section. Tighten so 500-rep Study E meets:

- `median_abs_rank_diff ≤ 2` AND
- `pct_within_2 ≥ 0.90`

Planner chooses from (non-exclusive):
1. Further below pool mean (reduce focal-to-pool rate gap).
2. Reduce pool stat-level noise (lower σ in DGP-E generation).
3. Use a fixed pool (no between-rep variance in non-focal players).

Document the fix in sim-spec changelog.

### Simulator
Re-run **Study E only** under the corrected DGP-E using the **same master
seed strategy**: `20260416 + study_offset + rep`. Studies A/B/C/D are NOT
re-run (they do not depend on DGP-E). Produce a new `simulation.md` that
references the prior run's A/B/C/D verdicts by path and reports fresh
Study E tables.

### Tester
1. Verify the corrected DGP-E produces a **deterministic rank shift at the
   boundary** — compute the expected rank_diff under the theoretical
   boundary; it should be tight around the target.
2. Evaluate Study E acceptance: `median_abs_rank_diff ≤ 2` AND
   `pct_within_2 ≥ 0.90` on R=500.
3. If Study E still fails, flag as a **threshold-revision candidate** for a
   follow-up request — do NOT silently relax thresholds.

### Reviewer
Produce unified successor `review.md`: keep A/B/C/D verdicts from
`replacement-2026-04-16` unchanged; replace only Study E section.

## Acceptance Criteria

1. Corrected DGP-E produces Study E: `median_abs_rank_diff ≤ 2` AND
   `pct_within_2 ≥ 0.90` on R=500 reps.
2. All other Monte Carlo criteria from `replacement-2026-04-16` (13/16
   already passing) still pass — via reference, not re-run.
3. Test suite unchanged: `devtools::test()` FAIL=0.
4. `R CMD check`: 0 ERRORs, 0 WARNINGs, NOTEs unchanged from develop.

## Hard Constraints

- **No changes to `R/` source code.** Simulation-pipeline-only run.
- **No changes** to `specs/spec-replacement.md` or function public interfaces.
- **Seed strategy must match** `replacement-2026-04-16` so unchanged tables
  remain reproducible.
- Dispatch Planner, Simulator, Tester with `isolation: "worktree"`.
- Honor `BrainMode: isolated` — no distiller, no `/contribute` prompt.

## Workflow

Simulation-only variant: **Planner (sim-spec patch) → Simulator → Tester
→ Reviewer → Shipper**. No Builder. No Scriber (deferred to follow-up if
thresholds change).

## Out of scope

- Threshold revision for Study E (handled by follow-up if Tester flags).
- Study C convergence (separate request R6).
- `R/replacement.R` changes of any kind.
