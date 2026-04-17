# docs.md — sgp-input-hardening-2026-04-17

**Scriber:** Claude Sonnet 4.6
**Date:** 2026-04-17
**Branch:** `feature/sgp-input-hardening-docs`

---

## Summary

This run regenerated `man/sgp.Rd` from the updated roxygen2 source committed in `a1a56e1`
(builder) and `fc6f980` (tester). The roxygen source had been updated before the merge of
PR #5, but `devtools::document()` had not been run. This scriber pass runs it and commits
the regenerated Rd file.

---

## Files Modified / Created

| File | Action | Description |
|------|--------|-------------|
| `man/sgp.Rd` | modified (regenerated) | Rebuilt from updated roxygen in `R/sgp.R` via `devtools::document()`. Now reflects pool_baseline validation behavior, split warning classes, and 6-item validation order. |
| `ARCHITECTURE.md` | modified | Updated run metadata header and diagram highlights to reflect sgp-input-hardening run. |

---

## Changes per File

### `man/sgp.Rd`

Three roxygen sections now appear correctly in the generated Rd:

**1. `\item{pool_baseline}` argument description (lines 46–51)**

Before (stale, pre-run):
```
\item{pool_baseline}{Character scalar.  Determines how pool constants are
constructed for the blended-pool rate-stat formulas.  Currently only
\code{"projection_pool"} (the default) is implemented.}
```

After (regenerated):
```
\item{pool_baseline}{Character scalar. Determines how pool constants are
constructed for the blended-pool rate-stat formulas. Currently only
\code{"projection_pool"} (the default) is implemented; any other value
aborts with \code{rotostats_error_invalid_pool_baseline}. The other
documented values (\code{"per_player"}, \code{"universal_constants"}) are
not yet implemented.}
```

**2. `\subsection{Validation order}` (lines 135–155)**

Before (stale, 5-item list starting with `rate_conversion`):
```
\enumerate{
\item \code{rate_conversion} must be one of the five recognized values ...
\item When \code{rate_conversion = "blended_pool"} ...
...
```

After (regenerated, 6-item list, `pool_baseline` now item 1):
```
\enumerate{
\item \code{pool_baseline} must be \code{"projection_pool"} (else
\code{rotostats_error_invalid_pool_baseline}). This check fires before
any other validation, regardless of \code{rate_conversion}.
\item \code{rate_conversion} must be one of the five recognized values ...
...
```

**3. `\section{Warnings}` (lines 172–195)**

Before (stale, single class in `\enumerate{}` block):
```
\code{sgp()} emits \code{rotostats_warning_missing_category_column} (via
\code{\link[cli:cli_abort]{cli::cli_warn()}}) in two situations:
\enumerate{
\item A scored category column is entirely absent ...
\item A player has 0 or \code{NA} projected IP ...
}
```

After (regenerated, two `\subsection{}` blocks, one per class):
```
\code{sgp()} may emit one or both of the following warnings via
\code{\link[cli]{cli_warn}}:

\subsection{rotostats_warning_missing_category_column}{
  A scored category column is entirely absent from \code{projections} ...
}

\subsection{rotostats_warning_zero_playing_time}{
  A player has 0 or \code{NA} projected IP (when ERA or WHIP is scored) or
  projected AB (when AVG is scored) ...
}

To suppress only one class, use
\code{withCallingHandlers(rotostats_warning_missing_category_column = ...)}
or \code{withCallingHandlers(rotostats_warning_zero_playing_time = ...)}
independently.
```

---

## devtools::check() Result

Run after `devtools::document()` with `document = FALSE, args = c('--no-manual')`:

```
0 errors | 0 warnings | 5 notes
```

NOTEs identical to tester baseline (pre-existing infrastructure notes):
- `.playwright-mcp` hidden directory
- `unable to verify current time`
- Non-standard top-level files (`ARCHITECTURE.md`, `HANDOFF.md`, `api_files`, `specs`)
- `NEWS.md`: No news entries found (pre-existing; note is expected to resolve once NEWS.md format is recognized)
- Escaped LaTeX specials in `replacement_from_prices.Rd` and `replacement_level.Rd` (pre-existing)

No new NOTEs introduced.

---

## Deferred Items

- None. All documentation for this run is complete.

---

## ARCHITECTURE.md

`ARCHITECTURE.md` was produced and is located at:
- Target repo root: `/Users/jacobdennen/rotostats/ARCHITECTURE.md`
- Run directory: `/Users/jacobdennen/.claude/plugins/data/statsclaw-statsclaw/workspace/rotostats/runs/sgp-input-hardening-2026-04-17/ARCHITECTURE.md`

Changes from previous run: updated header metadata (run name, branch, commit hash); updated
Module Reference Table to mark `R/sgp.R — sgp()` as changed; updated Mermaid diagram
highlight styles to reflect this run's changes (sgp-related nodes highlighted, replacement_level
nodes no longer highlighted).

---

## Notes for Reviewer

- Only `man/sgp.Rd` was modified by `devtools::document()` — no other Rd files were touched.
- `NAMESPACE` was NOT modified (no new exports or S3 registrations).
- The `NEWS.md` breaking-changes entry (written by builder) references `rotostats_warning_zero_playing_time` — this matches the regenerated Rd content.
