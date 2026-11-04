# Changelog

## Unreleased

**License change**
- The dataset is now licensed under **CC BY 4.0** (attribution required) instead of CC0; see
  `LICENSE` for the citation. v1.0.0 as already published remains under CC0.

**New: opponent-adjusted stats**
- `stats/adjusted_batting.csv` / `stats/adjusted_pitching.csv` — every counting event and rate
  stat adjusted for opponent quality and park, from an OLS regression on every plate appearance
  (batter-season + pitcher-season + park fixed effects + home term), one regression per division.
  Includes `raw_` comparison columns and an `opp_wOBA` strength-of-schedule column.
- `run_values/adjusted_parks.csv` — per-venue park effects from that regression.
- **"+" stats** — a 100-scale index (higher = better) for every rate stat in the advanced and
  adjusted files: JOPS+, TBIP+, WPT+, WHIP+, FIP+, AVG+, OBP+, K%+, BB%+, wOBA+, …

**Fixes**
- Play-by-play parser: from 2018 on, stats.ncaa.org renders the `;` clause separator as a
  literal `3a` (`"…singled (1-2 SBS) 3a Botti advanced to second"`) in ~15% of plays. Runner
  advances after it were being dropped; they are now parsed. **All event files were rebuilt**
  (e.g. v1 `HR` → `HR.1-H`, `S` → `S.2-H`), as were `re24.csv`, `linear_weights.csv`,
  `run_constants.csv` and the wOBA-based advanced stats. Run expectancy for 2018+ rises
  substantially (D1 2024, bases empty/0 out: 0.62 → 0.77).
- `pbp_ids_{season}.csv.gz`: the `narrative` column now shows `;` instead of `3a` and has HTML
  entities decoded (`&amp;` → `&`).
- Every linked player keeps the exact v1 `retro_id` (verified for all 146,830). Ids of
  narrative-only names (unlinked, `linked = 0`, no stats attached) may be renumbered.
- ~54,800 blank names in `player_crosswalk.csv` (mostly unlinked narrative-only ids, plus ~2,200
  linked players missing from the source's player table) were filled from the rosters, and the
  `name` column of the season/advanced stats files backfilled from it (~4,400 rows).
- `advanced_batting.csv`: total bases are now computed from hit types; where the source left
  TB blank (mostly ≤2016) SLG/OPS/ISO/JOPS/OPS+ previously read as if TB = 0
  (e.g. D1 2016 lgSLG was 0.037).

## v1.0.0 — 2026-09-15

First public release. NCAA college baseball (Divisions I–III), seasons 2016–2026, derived
entirely from stats.ncaa.org and released under CC0.

**Play-by-play & rosters**
- 10,914 Retrosheet-format `.EVC` event files (198,480 games), one per home team per season.
- 11,007 `.ROS` roster files and per-season `TEAM{year}` code maps.
- `playbyplay/pbp_ids_{year}.csv` — every play's original narrative plus the players in it
  resolved to stable 8-char IDs (~98% of plate appearances linked to a full player record).
- `player_crosswalk.csv` — one row per player id (retro_id ↔ player_id ↔ canonical_id ↔ bio);
  `teams.csv` — team code → school, division.

**Season & game stats**
- `season_batting.csv` / `season_pitching.csv` — official NCAA season totals, all years.
- `advanced_batting.csv` / `advanced_pitching.csv` — wOBA, wRC+, OPS+, JOPS, ISO, BABIP, FIP,
  ERA+, WHIP, TBIP, WPT, and more — park-adjusted and computed per (division, season).
- `gamelog_batting.csv` / `gamelog_pitching.csv` — per-game lines (2024–2026, where the source
  provides them).

**Run values**
- `run_values/` — RE24 matrix, linear weights, league constants, park factors, and the JOPS
  OBP-weight, each per division × season.

**Data-quality notes for this release**
- Pitcher hit-type-allowed components (`HR-A`/`2B-A`/`3B-A`) that stats.ncaa.org left blank
  (~34% of pitcher-seasons) are filled from the play-by-play and flagged by
  `season_pitching.csv`'s `xbh_source` column; official values are never overwritten.
- 9,550 logically-impossible batting rows (career-table misparses: H>AB, etc.) were dropped.
- Fielding stats and per-game logs before 2024 are not available at the source; no WAR.

See [README.md](README.md) and [FORMAT.md](FORMAT.md) for full documentation, and
[MANIFEST.md](MANIFEST.md) for row counts and checksums.
