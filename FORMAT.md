# File formats

Every file is UTF-8. CSVs have a header row. Event/roster/TEAM files follow Retrosheet
conventions (CRLF line endings, comma-separated records).

**Compression:** the bulk files are compressed. `events/{season}.tar.gz` unpacks to the
per-season `.EVC` / `.ROS` / `TEAM` files described below; `playbyplay/pbp_ids_{season}.csv`
and `stats/gamelog_*.csv` ship gzipped as `.csv.gz`. Paths below name the *uncompressed*
files inside those archives. `.csv.gz` loads directly in pandas.

---

## `events/{season}/{season}{TEAM}.EVC` — Retrosheet event file

One file per home team per season, containing that team's home games in chronological
order. A game is a block of records starting with `id`:

```
id,DUKE201802160
version,2
info,visteam,VAND
info,hometeam,DUKE
info,date,2018/02/16
info,number,0
info,usedh,true
info,pitches,pitches
start,laska001,"Adam Laskey",0,0,1
start,herrj001,"Jimmy Herron",0,1,7
...
play,1,0,herrj001,??,,43/G
play,1,0,conig001,??,,W
play,1,1,demap001,??,,K
sub,gillj001,"Jackson Gillis",1,0,1
...
data,er,laska001,3
```

**Record types**

| Record | Fields |
|---|---|
| `id` | `game_id` — `{HOMECODE}{YYYYMMDD}{N}`, `N` = 0-based game number that day (doubleheaders → 0,1) |
| `info` | `key,value` — `visteam`, `hometeam` (team codes), `date` (YYYY/MM/DD), `number`, `usedh`, `pitches` |
| `start` / `sub` | `id,"Name",side,batting_order,field_pos` — side `0`=visitor `1`=home; order `0`=DH pitcher; field_pos 1–9 (`10`=DH, `11`=PH, `12`=PR) |
| `play` | `inning,side,batter_id,count,pitches,event` — see event notation below |
| `data` | `er,pitcher_id,earned_runs` |

**Play `event` notation** (Retrosheet-style, narrative-derived):
`BASIC[/MODIFIER…][.ADVANCES]`. Examples: `S8` single to CF, `D7` double to LF,
`HR`, `W` walk, `IW` intentional walk, `HP` hit by pitch, `K` strikeout,
`43/G` groundout 2nd-to-1st, `63/G` short-to-first, `543/GDP` around-the-horn double play,
`E6` reached on SS error, `SB2` steal of 2nd, `CS2(26)` caught stealing 2nd C-to-SS.
`count` is `balls`+`strikes` or `??` when the game's narrative has no pitch counts.

## `events/{season}/{TEAM}{season}.ROS` — roster

`id,Last,First,bats,throws,team_code,position` — one row per player who appeared for the team
that season. `bats`/`throws` ∈ {R,L,B,?}; `?` where unknown.

```
browa015,Brown,Aaron,?,?,DUKE,P
```

## `events/{season}/TEAM{season}` — team map

`code,team_name` — the team codes referenced by that season's files.

---

## `teams.csv` — master team codes

`code,school,division` — all 1,204 teams. `division` ∈ {D-I, D-II, D-III, non-NCAA}.
(`non-NCAA` = NAIA/JUCO/other opponents that appear in games but aren't NCAA programs.)

## `player_crosswalk.csv` — player join hub

`retro_id,player_id,canonical_id,name,bats,throws,first_season,last_season,linked`

One row per `retro_id` used anywhere in the corpus. `linked=1` → tied to a real player
record (has `player_id`, name, bio); `linked=0` → a narrative-only name never matched to a
box (no player record). `name` is `Last, First`.

---

## `playbyplay/pbp_ids_{season}.csv` — play-by-play with player IDs

`game_id,contest_id,seq,inning,half,batting_side,narrative,batter_id,player_ids,unmatched_names`

One row per play (non-summary): the original play-by-play text plus the players in it resolved
to IDs. The play's Retrosheet event code lives in the `.EVC` file (join on `game_id`).

```
DUKE201802160,85205,1,1,top,away,"Herron, J. grounded out to 2b.",herrj001,herrj001,
```

- `narrative` — the original stats.ncaa.org play text (verbatim, may contain HTML entities)
- `batter_id` — the batter/subject's `retro_id` (blank for non-batter events)
- `player_ids` — `;`-joined `retro_id`s of **every** player involved in the play (batter + baserunners + substitutions)
- `unmatched_names` — `;`-joined surnames the matcher could not tie to a player (quality/transparency)
- `contest_id` — the source stats.ncaa.org game id (provenance only)

---

## `stats/season_batting.csv` / `season_pitching.csv`

One row per player-season (all years). Identity columns then stat columns:

`retro_id,player_id,canonical_id,name,season,year_label,team,` …

- **batting**: `G,AB,R,H,2B,3B,HR,RBI,TB,BB,IBB,HBP,K,SB,CS,SF,SH,DP,Picked,RBI2out,BA,OBPct,SlgPct`
- **pitching**: `G,App,GS,CG,SHO,SV,W,L,ERA,IP,H,R,ER,BB,IBB,SO,KL,HR-A,2B-A,3B-A,BF,P-OAB,HB,WP,Bk,SHA,SFA,Inh Run,Inh Run Score,Pitches,GO,FO,xbh_source`

These are official NCAA season totals (from player career pages). A player-season appears in
whichever file matches their role; a two-way player shows only their primary side.

The pitching hit-type-allowed components (`HR-A`, `2B-A`, `3B-A`) are blank for ~34% of
pitcher-seasons on stats.ncaa.org. Blanks are **filled from the play-by-play** (the narrative
states each hit's type); official values are never overwritten. **`xbh_source`** flags how each
row's three components were obtained: `official` (all three from NCAA), `pbp` (all three
PBP-filled), `partial` (mix), `none` (blank, no PBP available). PBP fills are accurate to ~0.6
HR on average (73% exact, slight undercount) — good enough that `TBIP`/`FIP`/`oppSLG` in
`advanced_pitching.csv` use the filled components, but treat non-`official` rows as estimates.

## `stats/gamelog_batting.csv` / `gamelog_pitching.csv`

One row per player per game (**2024–2026 only** — see README coverage). Identity/context then stats:

`retro_id,player_id,canonical_id,name,season,division,game_id,contest_id,date,side,team,team_code,opponent,opp_code,jersey,position,` …

- `game_id` joins to the event files; `side` ∈ {home,away}; `date` = YYYY-MM-DD
- batting stats: `AB,R,H,2B,3B,HR,RBI,TB,BB,IBB,HBP,K,KL,SB,CS,SF,SH,OPP DP,Picked`
- pitching stats: `IP,H,R,ER,BB,IBB,SO,KL,HR-A,2B-A,3B-A,BF,HB,WP,Bk,SHA,SFA,Inh Run,Inh Run Score,TUER,pickoffs`

Rows with a blank `retro_id`/`player_id` (~4%) are box lines whose player couldn't be tied to a
career record; the stat line and name are still present.

---

## `stats/advanced_batting.csv` — advanced batting metrics

One row per player-season. Identity/context columns then metrics:

`retro_id,player_id,canonical_id,name,season,division,team,PA,AB,AVG,OBP,SLG,OPS,JOPS,ISO,BABIP,`
`BB%,K%,BB/K,HR%,XBH,XBH%,TB,SB%,SecA,RC,wOBA,wRAA,wRC,wRC+,OPS+,wSB`

| stat | meaning |
|---|---|
| PA | plate appearances = AB+BB+HBP+SF+SH |
| JOPS | a·OBP + SLG, where a is the OBP weight that best predicts runs in that division (D1 2.48, D2 2.75, D3 2.52; see run_values/jops_constants.csv) |
| ISO | isolated power = SLG − AVG |
| BABIP | (H−HR)/(AB−K−HR+SF) |
| BB% / K% / HR% | per plate appearance |
| XBH / XBH% | extra-base hits (2B+3B+HR) |
| SB% | SB/(SB+CS) |
| SecA | secondary average = (BB+(TB−H)+(SB−CS))/AB |
| RC | Bill James basic runs created = (H+BB)·TB/(AB+BB) |
| wOBA | weighted on-base average, on the division-season OBP scale |
| wRAA | weighted runs above average (park-neutral) |
| wRC | weighted runs created |
| wRC+ | wRC per PA vs league = 100, **park-adjusted** (see park_factors) |
| OPS+ | 100·(OBP/lgOBP + SLG/lgSLG − 1), **park-adjusted** |
| wSB | weighted stolen-base runs above average |

**All league-relative stats use each (division, season)'s own run environment** — the
weights and league averages in `run_values/`. Blank env-stats mean the player-season's
division couldn't be determined (~0.6%).

## `stats/advanced_pitching.csv` — advanced pitching metrics

`retro_id,player_id,canonical_id,name,season,division,team,IP,BF,ERA,FIP,FIP-,ERA-,ERA+,`
`WHIP,TBIP,WPT,K/9,BB/9,HR/9,H/9,K/BB,K%,BB%,K-BB%,BABIP,LOB%,BAA,oppOBP,oppSLG,oppOPS,GO/FO,GB%out,Pit/PA`

| stat | meaning |
|---|---|
| IP | innings pitched, **decimalized** (44.2 thirds → 44.67) |
| WHIP | walks + hits per inning = (BB+H)/IP |
| TBIP | bases allowed per inning, **counting a walk as 1 base** = (BB + TB_allowed)/IP; ≥ WHIP always (the gap is the extra bases from extra-base hits) |
| WPT | WHIP + TBIP, the unweighted sum (**lower is better**) — the pitching analog of OPS |
| FIP | (13·HR + 3·(BB+HBP) − 2·K)/IP + FIP-constant (per division-season) |
| FIP- / ERA- | 100 = league average; lower is better; **park-adjusted** |
| ERA+ | 100·lgERA/ERA; higher is better; **park-adjusted** |
| K% / BB% / K-BB% | per batter faced |
| BABIP | (H−HR)/(oppAB−K−HR+SF) |
| LOB% | strand rate = (H+BB+HBP−R)/(H+BB+HBP−1.4·HR) |
| BAA / oppOBP / oppSLG / oppOPS | opponent slash line |
| GO/FO, GB%out | ground-out/fly-out ratio and share (of outs, not all balls in play) |
| Pit/PA | pitches per batter faced |

## `run_values/run_constants.csv` — league constants

`division,season,PA,lgAVG,lgOBP,lgSLG,lgR/PA,wOBA_scale,lgwOBA,w1B,w2B,w3B,wHR,wBB,wHBP,`
`runSB,runCS,lgwSB,IP,lgERA,FIP_constant` — one row per division-season. These are the exact
constants used to compute the advanced stats above (so wOBA/FIP are reproducible).

## `run_values/jops_constants.csv` — JOPS weight

`division,a,n_team_seasons,r_JOPS,r_OPS` — the constant **a** in `JOPS = a·OBP + SLG`, found
per division by regressing team-season runs-per-PA on team OBP and SLG (a = OBP coef / SLG
coef) over all team-seasons. `r_JOPS`/`r_OPS` are the correlations of each metric with R/PA
(JOPS > OPS in every division). College a ≈ 2.5 — higher than MLB's ~1.8, i.e. OBP is worth
even more relative to slugging in college.

> **Note on TBIP inputs:** the extra-base components (`HR-A`, `2B-A`, `3B-A`) are ~34% blank on
> stats.ncaa.org; those blanks are filled from the play-by-play (see `season_pitching.csv`'s
> `xbh_source`), so `TBIP`/`WPT`/`FIP`/`oppSLG` are complete for almost all pitcher-seasons.
> PBP-filled components are estimates (~0.6 HR mean error, slight undercount), so those stats
> read marginally low for non-`official` rows.

## `run_values/park_factors.csv` — park factors

`team_code,school,division,stadium,seasons,home_G,road_G,home_RPG,road_RPG,PF_raw,PF`

One row per team, pooled over 2016–2026. `PF` (and `PF_raw` before regression) are on the
100 scale — **100 = neutral, >100 = hitter's park**. Computed from total runs (both teams)
per game in the team's home vs. road games; neutral-site games (any game carrying a
`location`) are excluded; normalized so the league averages 100 and regressed toward 100 by
sample size. To park-adjust a full-season line use the **half-weighted** factor `(PF+100)/200`
(a player is home only ~half the time) — which is exactly how the `wRC+`/`OPS+`/`ERA+`/`FIP-`
columns in the advanced files are already adjusted. Only teams with ≥50 home and ≥50 road
non-neutral games are included (963 teams); players on teams without a factor use 100.

## `run_values/re24.csv` — run-expectancy matrix

`division,season,bases,outs,run_expectancy,n` — one row per base-out state per
division-season. `bases` is a 3-char occupancy string (`___`, `1__`, `_2_`, `__3`, `12_`,
`1_3`, `_23`, `123`); `outs` ∈ {0,1,2}. `run_expectancy` = mean runs scored from that state to
the end of the half-inning (over completed 3-out innings); `n` = plate appearances observed.

## `run_values/linear_weights.csv` — event run values

`division,season,event,run_value,run_value_above_out,n` — the run value of each event type,
computed from the RE24 matrix as `RE(after) − RE(before) + runs_on_play`, averaged over
occurrences. `run_value_above_out` re-centers on the batted-out value (the usual linear-weights
convention, e.g. for wOBA). Both are empirical, per division-season. Note the run environment
rises markedly in 2022+ (a real college-baseball scoring shift), which lifts all above-out values.

Each file also carries a **`season = ALL`** row per division: the cumulative (2016–2026)
value, volume-weighted by PA/occurrence count — the exact pooled RE24 matrix, and the
count-weighted mean of the yearly linear weights.

## Stat abbreviations

`TB` total bases · `KL` strikeouts looking · `HR-A` HR allowed · `2B-A/3B-A` extra-base hits
allowed · `BF` batters faced · `P-OAB` opponent at-bats · `HB` hit batters · `Bk` balks ·
`SHA/SFA` sac hits/flies allowed · `TUER` team unearned runs · `OPP DP` hit into DP ·
`Picked` picked off · `RBI2out` RBI with 2 outs · `App` appearances · `GS` games started.
