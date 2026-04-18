# Docs Trace — replacement-multi-pos-all-spec-2026-04-18

## Summary

This run produced a design doc for the `multi_pos = "all"` mode of `replacement_level()`.
No user-facing help files, vignettes, or man pages were modified (those are deferred to the
implementation run). The changes are developer-facing design artifacts.

## Files Modified / Created

| File | Action | Notes |
|------|--------|-------|
| `specs/spec-replacement-multi-pos-all.md` | Created | Design doc — primary deliverable |
| `plans/error-messages.md` | Modified | Added `rotostats_error_multi_pos_all_unsupported` row |
| `ARCHITECTURE.md` | Modified | Run header update; "all" mode design notes; new Key Design Decisions section |

## Doc Generation Commands

None required. No roxygen2 changes were made.

## Architecture Diagram

`ARCHITECTURE.md` was updated in the target repo and copied to the run directory.
The update is minimal: run header, one bullet point under "Known Limitations", and a new
"Key Design Decisions (replacement-multi-pos-all-spec-2026-04-18)" section.

## Deferred Items

- Man page for `replacement_level()` will need updating when the `"all"` mode is
  implemented: `@param multi_pos`, `@return` (shape change for `replacement_stats`,
  `position_assignments` list format), and a new `@examples` block showing the long-frame
  output.
- Man pages for `par()`, `zar()`, `dollar_values()` will need a note that `multi_pos = "all"`
  input raises `rotostats_error_multi_pos_all_unsupported`.
- `NEWS.md` entry for user-visible change (new `"all"` mode) is a builder/scriber
  responsibility in the implementation run.

## Notes

- `devtools::document()` is not required for this run.
- `devtools::check()` is not required for this run (no R code changed).
