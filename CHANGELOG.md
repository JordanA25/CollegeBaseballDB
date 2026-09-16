# Changelog

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
