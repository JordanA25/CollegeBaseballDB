# Changelog

## v1.2.0 — 2026-11-07

Data-quality release: play-by-play from 2016–2021 now reconciles with official scores as well
as 2022–2026 does.

**Fixes**
- **2018 home/away reversed (all divisions).** The 2018 scoreboard lists home and away
  backwards for ~98% of games (proof: games whose play-by-play ends after a top half — only
  possible when the home team leads — were "won" by the listed home team 1–4% of the time vs
  99.5% in other years). Fixed per game by matching each game's play-by-play runs to the
  scoreboard either way round; 342 games that were listed correctly are kept. As a result:
  - **2018 `game_id`s change** (they start with the home team's code) and 2018 games are now
    filed in the correct team's `.EVC`.
  - 2018 rows in the adjusted stats carry the player's own team (they showed the opponent's).
  - Park factors and the adjusted park effects no longer include a season of reversed
    home/road splits (Air Force PF 118 → 127).
- **Older narrative formats.** 2016–18 text separates runner clauses with a colon
  (`walked: Albaugh advanced to second`) and 2019–21 often with nothing
  (`3 RBI J. Millen scored Z. Coutts scored`). Runner advances after them were dropped; they are
  now parsed. Play-by-play runs vs the official box score, D3 (D1/D2 similar):

  | | 2016 | 2017 | 2018 | 2019 | 2020 | 2021 | 2022+ |
  |---|---|---|---|---|---|---|---|
  | exact score, before | 14% | 17% | 1% | 44% | 36% | 43% | ~87% |
  | exact score, after | 85% | 88% | 90% | 90% | 88% | 89% | ~88% |
  | within 1 run, after | 95% | 97% | 99% | 99% | 98% | 99% | ~98.6% |

  Event files, play-by-play, RE24, linear weights, run constants, advanced, adjusted and "+"
  stats were all rebuilt.
- Games whose home team had no code (e.g. "Saint Anselm" spelled out) were missing from the
  corpus and are now included (198,480 → 198,805 games in total); new codes for three
  non-NCAA opponents (BATT, LSNO, MSCH) bring `teams.csv` to 1,207 teams.
- Every linked player keeps the same `retro_id` (verified for all 146,834).

**Known gaps (unchanged)**: play-by-play exists for only ~41% of 2016 and ~71% of 2019 D3 games,
the 2023 scrape is missing ~13% of D3 games, and ~420 2026 D3 games are still listed as
scheduled — player totals from play-by-play run short of official totals in those seasons.

## v1.1.0 — 2026-11-04

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
