# rotostats Workspace CHANGELOG

| Date | Run | Summary | Status |
|------|-----|---------|--------|
| 2026-04-18 | [replacement-multi-pos-all-spec-2026-04-17](runs/replacement-multi-pos-all-spec-2026-04-17/2026-04-18-replacement-multi-pos-all-spec.md) | Design doc for `replacement_level(multi_pos = "all")`: tidy-frame output, generalized zero-sum invariant (1e-5), downstream reject contract, new error class `rotostats_error_multi_pos_all_unsupported`; PR #17 | PASS_WITH_NOTE |
| 2026-04-18 | [replacement-dgp-e-calibration-2026-04-17](runs/replacement-dgp-e-calibration-2026-04-17/simulation.md) | Recalibrate DGP-E with fixed complement SP pool; Study E waiver resolved (median_abs_rank_diff=2, pct_within_2=1.0); PR #16 | PASS_WITH_NOTE |
| 2026-04-18 | [sgp-denom-weights-guidance-docs-2026-04-17](runs/sgp-denom-weights-guidance-docs-2026-04-17/2026-04-18-sgp-denom-weights-guidance-docs.md) | Add "Choosing weights" guidance to `?sgp_denominators` @details; docs-only, no default change, no NEWS entry; PR #15 | PASS |
| 2026-04-18 | [sgp-denom-inverse-categories-param-2026-04-17](runs/sgp-denom-inverse-categories-param-2026-04-17/2026-04-18-sgp-denom-inverse-categories.md) | Expose `inverse_categories` arg on `sgp_denominators()`; replace hard-coded `INVERSE_CATEGORIES` constant; default-vs-explicit distinction via `missing()`; 13 new tests (T-58–T-70); PR #13 | PASS_WITH_NOTE |
| 2026-04-17 | [sgp-denom-dgpb-rerun-2026-04-17](runs/sgp-denom-dgpb-rerun-2026-04-17/simulation.md) | Re-run Q4/Q7 under corrected sigma-shift DGP-B (pre_sd=10, post_sd=25); Q4 4/4 PASS, Q7 3/3 PASS, relative gate 150.6% | PASS |
| 2026-04-17 | [replacement-name-match-audit-2026-04-17](runs/replacement-name-match-audit-2026-04-17/log-entry.md) | Wire rotostats_warning_name_match_failure at two emit sites in replacement_level() and replacement_from_prices() | PASS |
| 2026-04-17 | [sgp-input-hardening-2026-04-17](runs/sgp-input-hardening-2026-04-17/log-entry.md) | Harden sgp() input validation: pool_baseline abort + warning class split (rotostats_warning_zero_playing_time) | PASS_WITH_NOTE |
| 2026-04-17 | [replacement-2026-04-16](runs/replacement-2026-04-16/log-entry.md) | Add replacement_level(), replacement_from_prices(), default_replacement_params, rate_stat_denominators() | PASS_WITH_FOLLOWUP |
