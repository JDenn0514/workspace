# Request: sgp-input-hardening-2026-04-17

## Target repo

- Name: rotostats
- URL: https://github.com/JDenn0514/rotostats
- Checkout: /Users/jacobdennen/rotostats
- Default branch: develop
- Feature branch (this run): feature/sgp-input-hardening
- PR base: develop

## Workspace repo

- Path: /Users/jacobdennen/.claude/plugins/data/statsclaw-statsclaw/workspace/rotostats
- Remote: JDenn0514/workspace
- Status: PASS (push access confirmed 2026-04-17)

## Scope

Harden input validation in `sgp()` — no algorithmic change.

### Changes required

1. **`pool_baseline` validation.** Add a top-of-function check: `pool_baseline`
   must equal `"projection_pool"`. Anything else aborts with the new error class
   `rotostats_error_invalid_pool_baseline` and a message naming the valid values.
   Add the class to `plans/error-messages.md`.

2. **Warning class split.** Replace the current
   `rotostats_warning_missing_category_column` (which conflates two distinct
   conditions) with two classes:
   - `rotostats_warning_missing_category_column` — category absent from
     `projections`.
   - `rotostats_warning_zero_playing_time` — row has `IP = 0` or `AB = 0` on a
     scored rate category.

   Update `plans/error-messages.md`. Update call sites in `R/sgp.R`.
   Add a `NEWS.md` entry noting the minor breaking change for callers using
   `withCallingHandlers()` on the old class.

## Acceptance criteria

- `sgp(..., pool_baseline = "projection_pool")` behaves as before.
- Any other `pool_baseline` value aborts with
  `rotostats_error_invalid_pool_baseline` naming the valid values.
- `sgp()` on a projections frame missing a scored category fires
  `rotostats_warning_missing_category_column` AND NOT the zero-playing-time class.
- `sgp()` on a projections frame with `IP = 0` on a scored rate category fires
  `rotostats_warning_zero_playing_time` AND NOT the missing-column class.
- R CMD check: 0 ERRORs, 0 WARNINGs, NOTEs unchanged from main.
- All existing sgp tests still pass.

## Hard constraints

- Errors/warnings via `cli_abort()` / `cli_warn()` with `class=` from
  `plans/error-messages.md` — never bare `stop()` or `warning()`.
- No refactors of code unrelated to the two changes above.
- Vectorized operations only; no row-wise loops.

## Workflow

- Code-only (Workflow 2: code + ship). No simulation — changes are input
  validation, not statistical.
- Pipeline: Planner → Builder → Tester → Scriber → Reviewer → Shipper.
- Brain mode: isolated. No distiller.

## References

- specs/spec-sgp.md
- plans/error-messages.md
- plans/sgp-cleanup.md §"R1 — sgp-input-hardening"

## Uploaded files

None.

## Brain mode

Isolated. Distiller is skipped.
