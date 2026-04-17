# Handoff Notes

**Run:** replacement-name-match-audit-2026-04-17
**Date:** 2026-04-17
**Slug:** 2026-04-17-replacement-name-match-audit
**Branch:** feature/replacement-name-match-audit @ 1d113f5 → PR #6 against develop
**Verdict:** PASS

---

1. **`rotostats_warning_name_match_failure` is now live at both documented sites.** If the `normalize_player_name()` helper is ever updated (e.g., to handle different Unicode normalization forms), both emit sites will automatically pick up the change — no modification to `replacement.R` is needed.

2. **This run closes item R1 from `plans/replacement-cleanup.md`.** No follow-up items from this run itself.

3. **The "Thrown by" column in `plans/error-messages.md` line 96 only lists `replacement_level()`.** Now that `replacement_from_prices()` also emits this class at Site 2, the registry entry should be updated to list both functions. This is a minor documentation follow-up and does not affect package behavior.

4. **The NEWS.md `## Improvements` entry does not resolve the pre-existing R CMD check NOTE** (`Problems with news in 'NEWS.md': No news entries found`). That NOTE is a pre-existing format issue unrelated to this change and is among the 4 baseline NOTEs on develop.

5. **No NAMESPACE, man/, DESCRIPTION, or function signature changes.** The PR requires no breaking-change annotations.

6. **One builder respawn cycle occurred** (non-ASCII em-dash in `cli_warn()` string literals). Fix: `\u2014` escape at lines 1065 and 1285 in commit `272cb97`. Rendered warning message is byte-identical at runtime.

---

**PR:** https://github.com/JDenn0514/rotostats/pull/6
**Commits:** `80afe5c` → `d9b9fe8` → `272cb97` → `1d113f5`
