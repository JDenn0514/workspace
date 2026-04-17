# docs.md — replacement-name-match-audit-2026-04-17

## Summary

This run wires a previously-registered but never-emitting warning class into the rotostats replacement-level calibration functions.

## Documentation Changes

No user-facing documentation files (man pages, vignettes, README, tutorials) were modified in this run. The roxygen comments at `R/replacement.R:960-961` and `R/replacement.R:973` already document the intended warning behavior — those lines were written when the warning class was registered but before the emit sites existed. The `NEWS.md` entry was added by the builder (commit `80afe5c`) and is not duplicated here.

## What the User Experiences

When `verbose = TRUE`, `replacement_level()` now emits a `rotostats_warning_name_match_failure` warning whenever one or more player names in `league_history$prices` cannot be matched to any player in `projections` after Unicode name normalization (NFD decomposition, diacritic stripping, lowercasing, punctuation removal). The warning shows the count of unmatched names and a sample of up to three, and explicitly states that calibration output is unchanged. This warning has always been documented in `?replacement_level` and registered in `plans/error-messages.md`; it simply never fired before this run.

When `verbose = TRUE` and `prices` does not contain a `player_id` column, `replacement_from_prices()` now emits the same `rotostats_warning_name_match_failure` class whenever two or more rows in `prices` have names that normalize to the same string (for example, `"Jose Ramirez"` and `"Jos\u00e9 Ram\u00edrez"`). The warning identifies these as potential silent merge candidates and recommends supplying a `player_id` column for exact identity matching. Again, the stat-line computation is unchanged.

Both warnings are purely diagnostic. Setting `verbose = FALSE` (the default for both functions) suppresses all name-match warnings entirely. Users who set `verbose = TRUE` deliberately opt in to this level of data-quality feedback.

## ARCHITECTURE.md

`ARCHITECTURE.md` was updated in this run. The previous version covered only the `sgp_denominators()` subsystem (sgp-denominators-2026-04-16). The new version covers the full repo: all modules across the Config, Replacement, SGP Denominators, and SGP Scarcity layers. The replacement module — including both emit sites, `normalize_player_name()`, and the `.validate_league_history_inputs()` internal — is documented in the Module Structure diagram, the Function Call Graph (two sub-diagrams, one per public function), and the Data Flow diagram. Changed nodes are highlighted in blue. The previous Key Design Decisions (rank-flip for inverse categories, `as.double` dispatch) are preserved alongside the four new decisions from this run.

## Deferred Items

The "Thrown by" column in `plans/error-messages.md` line 96 lists only `replacement_level()` as the throwing function for `rotostats_warning_name_match_failure`. Now that `replacement_from_prices()` also emits this class, the registry entry should be updated to list both functions. This is a minor documentation follow-up and does not affect package behavior.

## Doc Generation

No `devtools::document()` run is required for scriber's changes — no roxygen blocks were modified in this run. The builder already ran `devtools::document()` as a hygiene step and confirmed no man/ drift.
