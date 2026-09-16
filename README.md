# College Baseball Play-by-Play Dataset (2016–2026)

An open play-by-play corpus for NCAA college baseball — Divisions I, II, and III,
seasons **2016–2026** — in **Retrosheet event-file format**, with every play, player,
and game tied together by stable IDs.

- **198,480 games** as Retrosheet event files (`.EVC`), one file per home team per season
- **Play-by-play text with resolved IDs** (each play's narrative + every player in it → an 8-char ID)
- **Season batting & pitching stats** (official NCAA season totals)
- **Advanced metrics** — wOBA, wRC+, OPS+, JOPS, ISO, FIP, ERA+, WPT, … park-adjusted and per-division
- **Run values** — RE24 matrix, linear weights, and park factors, per division × season
- **Game-by-game batting & pitching logs** (2024–2026; see *Coverage* below)
- One **player crosswalk** and one **team-code table** join everything together

Everything is derived from publicly available data on **stats.ncaa.org**.
*Not affiliated with or endorsed by the NCAA.*

---

## What's in here

```
CollegeBaseballDB/
├── README.md                     ← this file
├── FORMAT.md                     ← exact field-by-field spec of every file
├── CHANGELOG.md · MANIFEST.md · LICENSE
├── teams.csv                     ← team code → school, division (1,204 teams)
├── player_crosswalk.csv          ← the join hub: player_id ↔ retro_id ↔ name ↔ bio
├── events/                       ← Retrosheet event files, one tarball per season
│   └── {SEASON}.tar.gz           ← unpacks to {SEASON}{TEAM}.EVC, {TEAM}{SEASON}.ROS, TEAM{SEASON}
├── playbyplay/
│   └── pbp_ids_{SEASON}.csv.gz   ← play-by-play text + per-play player IDs (gzipped)
├── stats/
│   ├── season_batting.csv        ← per player-season, standard totals (all years)
│   ├── season_pitching.csv       ← per player-season, standard totals (all years)
│   ├── advanced_batting.csv      ← per player-season, advanced metrics (wOBA, wRC+, OPS+, …)
│   ├── advanced_pitching.csv     ← per player-season, advanced metrics (FIP, ERA+, K%, …)
│   ├── gamelog_batting.csv.gz    ← per player-game, 2024–2026 (gzipped)
│   └── gamelog_pitching.csv.gz   ← per player-game, 2024–2026 (gzipped)
└── run_values/                   ← sabermetric run values & context
    ├── re24.csv                  ← 24-state run-expectancy matrix (per division × season)
    ├── linear_weights.csv        ← run value of each event type (per division × season)
    ├── run_constants.csv         ← league constants (wOBA scale, FIP constant, lgOBP/ERA, …)
    ├── park_factors.csv          ← per-team park factors (100 = neutral), pooled 2016–2026
    └── jops_constants.csv        ← the JOPS OBP-weight (a in a·OBP+SLG) per division
```

**Compressed files:** the bulk is compressed to keep the repo light (~655 MB). `.csv.gz` files
load directly in pandas (`pd.read_csv("…​.csv.gz")`) or with `gzip`/`zcat`; each `events/{year}.tar.gz`
extracts with `tar xzf events/2018.tar.gz`. The small analysis tables (season/advanced stats,
`teams.csv`, `player_crosswalk.csv`, `run_values/`) are left uncompressed for direct use.

See **[FORMAT.md](FORMAT.md)** for the columns and record types in each file, and
**[MANIFEST.md](MANIFEST.md)** for sizes + checksums.

---

## Quickstart

```python
import pandas as pd

# 1) Top hitters by park-adjusted wRC+ (D1 2026, min 150 PA)
bat = pd.read_csv("stats/advanced_batting.csv")
d1 = bat[(bat.division == "D1") & (bat.season == 2026) & (bat.PA >= 150)]
print(d1.nlargest(10, "wRC+")[["name", "team", "PA", "OPS", "JOPS", "wRC+"]])

# 2) One player's career line (find their id in the crosswalk, then filter)
xw = pd.read_csv("player_crosswalk.csv")
pid = xw.loc[xw.name == "Herron, Jimmy", "retro_id"].iloc[0]     # -> herrj001
print(bat[bat.retro_id == pid][["season", "team", "OPS", "wRC+"]])

# 3) Season pitching leaders by FIP (2026, min 60 IP)
pit = pd.read_csv("stats/advanced_pitching.csv")
print(pit[(pit.season == 2026) & (pit.IP.astype(float) >= 60)]
      .nsmallest(10, "FIP")[["name", "IP", "ERA", "FIP", "WPT"]])

# 4) Every play a player was involved in (2026) — pandas reads .csv.gz directly
pbp = pd.read_csv("playbyplay/pbp_ids_2026.csv.gz")
print(pbp[pbp.player_ids.str.contains(pid, na=False)][["game_id", "narrative"]].head())

# 5) Read one team's Retrosheet event file from its season tarball
import tarfile
with tarfile.open("events/2018.tar.gz") as t:
    print(t.extractfile("2018/2018DUKE.EVC").read().decode()[:400])
```

Join keys: `retro_id` links plays ↔ rosters ↔ stats; `game_id` links plays ↔ games ↔ gamelogs;
team `code` links to `teams.csv`. See below.

---

## The IDs (how everything joins)

| ID | What it is | Where it lives |
|---|---|---|
| `retro_id` | 8-char Retrosheet-style player id (e.g. `herrj001`) | event files, rosters, pbp, stats |
| `player_id` | clean sequential player id, 1–168006 | crosswalk, stats |
| `canonical_id` | internal NCAA-derived player key | crosswalk, stats |
| `game_id` | `{HOMECODE}{YYYYMMDD}{N}` (e.g. `DUKE201802160`) | event files, pbp, gamelogs |
| team `code` | 3–4 char team code (e.g. `DUKE`) | file names, teams.csv, gamelogs |

**One player = one `retro_id` = one `player_id`** (the mapping is one-to-one). The
`player_crosswalk.csv` translates between all three player keys and carries name / bats /
throws / seasons.

### Worked example — Jimmy Herron

```
events/2018.tar.gz→2018DUKE.EVC   play,1,0,herrj001,??,,43/G           ← grounds out 4-3 (event code)
playbyplay/pbp_ids_2018.csv.gz    DUKE201802160,…,"Herron, J. grounded out to 2b.",herrj001,…   ← same play: text + IDs
player_crosswalk.csv         herrj001,29030,5196744,"Herron, Jimmy",…,2016,2018,1
stats/season_batting.csv     herrj001,29030,…,2018,2017-18,Duke,…,250,61,76,18,…   ← his 2018 line
```

Join `pbp` / event files → a player via `retro_id`; join a player → season/career stats via
`retro_id` (or `player_id`); join any play → its game via `game_id`.

---

## Advanced stats & sabermetrics

Every rate/index stat is computed **per (division, season) run environment** — D1, D2, D3 and
each year get their own league constants, weights, and park factors, never mixed. Index stats
(wRC+, OPS+, ERA+, FIP-) are **park-adjusted**. Nothing here needs a modeling "big decision"
(no WAR, no replacement level) — just formulas fit to this data. Details/columns in
[FORMAT.md](FORMAT.md); the constants used are in `run_values/`.

**Batting** (`stats/advanced_batting.csv`): AVG, OBP, SLG, OPS, **JOPS**, ISO, BABIP, BB%, K%,
BB/K, HR%, XBH%, SB%, SecA, RC, **wOBA**, wRAA, wRC, **wRC+**, **OPS+**, wSB.
- **JOPS** = `a·OBP + SLG`, where `a` is the OBP weight that best predicts runs in that division
  — found by regressing team runs on OBP & SLG: **D1 2.48, D2 2.75, D3 2.52** (higher than MLB's
  ~1.8; OBP matters more in college). JOPS correlates with run scoring better than OPS.

**Pitching** (`stats/advanced_pitching.csv`): ERA, **FIP**, FIP-, ERA-, **ERA+**, WHIP, **TBIP**,
**WPT**, K/9, BB/9, HR/9, K%, BB%, K-BB%, BABIP, LOB%, opponent slash (BAA/oppOBP/oppSLG/oppOPS),
GO/FO, Pit/PA.
- **TBIP** = `(BB + total bases allowed) / IP` — bases allowed per inning, counting a walk as one
  base (so TBIP ≥ WHIP; the gap is the extra bases from extra-base hits).
- **WPT** = `WHIP + TBIP` — the pitching analog of OPS (**lower is better**).

**Run values** (`run_values/`, all per division × season):
- `re24.csv` — the 24-state run-expectancy matrix (plus an `ALL` cumulative row per division).
- `linear_weights.csv` — the run value of each event (single, walk, HR, out, SB, …).
- `run_constants.csv` — league constants (wOBA scale, FIP constant, lgOBP/lgSLG/lgERA, …).
- `park_factors.csv` — per-team park factors (100 = neutral), pooled 2016–2026. Validated on the
  known altitude parks: **Air Force (118)** and **New Mexico (117)** top the hitter's-park list.
- `jops_constants.csv` — the JOPS OBP-weight per division.

---

## Coverage & honest limitations

The scrape reached the box-score/play-by-play page for ~99% of games, but **what those pages
contain varies by year** — this shapes what's available:

| Data | Years | Notes |
|---|---|---|
| Play-by-play (event files, pbp) | **2016–2026** | full coverage |
| Season batting / pitching totals | **2016–2026** (some back to 2011) | official, from player career pages |
| **Per-game** batting/pitching logs | **2024–2026 only** | stats.ncaa.org serves per-game stat *numbers* only from 2024; 2016–2023 pages list participants only |
| Fielding stats | **none** | not captured at season level; per-game fielding exists 2024+ but isn't included here |
| Fielding **position** played | 2016–2026 | in the event files (`start`/`sub`) and rosters |

Other caveats:
- **Play-by-play player linkage ≈ 98%** of plate appearances resolve to a full player record.
  The rest are non-NCAA opponents (no roster exists) or games whose box was never captured.
- **Handedness** (`bats`/`throws`) is only populated for players active 2022–2026 (~27% overall).
- **Event fields are narrative-derived.** Pitch sequences pass through StatCrew coding;
  hit-location zones are not emitted; fielding sequences reflect what the narrative states.
  Some games' narratives omit pitch counts (`count` shows `??`) or run clauses together
  (an occasional stolen base / advance is missed). An unparseable play is a no-op, never a
  fabricated event.
- **Advanced-stat inputs.** Extra-base-hit-allowed components (`HR-A`, `2B-A`, `3B-A`) are ~34%
  blank in season pitching data; those blanks are **filled from the play-by-play** (flagged by
  `season_pitching.csv`'s `xbh_source`), so `TBIP`/`WPT`/`FIP`/`oppSLG` are complete — PBP fills
  are estimates (~0.6 HR mean error), so non-`official` rows read marginally low. A small number
  of season rows are corrupt
  (e.g. H > AB from a career-table misparse). **Park factors** are single team-level run factors
  (no self-park correction, no separate OBP/HR/handedness splits), pooled multi-year because a
  single ~28-game college home slate is too small; ~95% of qualified players get a real factor,
  the rest use neutral 100.

---

## License & provenance

**Data: [CC0 1.0 Universal](LICENSE) (public domain).** Use it for anything — research,
commercial, redistribution — no permission or attribution required (a credit is always
appreciated).

Derived from publicly available box scores and play-by-play on **stats.ncaa.org**.
Not affiliated with, licensed by, or endorsed by the NCAA.
