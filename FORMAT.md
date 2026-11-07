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

`code,school,division` — all 1,207 teams. `division` ∈ {D-I, D-II, D-III, non-NCAA}.
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

## `stats/adjusted_batting.csv` / `adjusted_pitching.csv` — opponent-adjusted stats

Every counting event and rate stat, **adjusted for the quality of the opponents faced and the
parks played in**, from an ordinary-least-squares regression on every plate appearance
(2016–2026, play-by-play). **One regression per division** — D1, D2 and D3 are never mixed.

For each PA outcome *e* (single, double, triple, HR, walk, IBB, HBP, strikeout, ROE, fielder's
choice, SF, SH, interference, ground out, air out, other out, and GDP):

```
y_e = μ + h·home + β[batter-season] + γ[pitcher-season] + δ[park] + ε     (y_e = 1 if the PA ended in e)
```

- **Batter / pitcher effects are per player-season**, centred to 0 within each season, so every
  season keeps its own run environment. **Park effects are one per venue, pooled 2016–2026**
  (a single ~25-game home slate is far too noisy); neutral-site games share one level. `home`
  is the batting-team-at-home indicator.
- **Adjusted count = (season league rate + player effect) × PA** (pitchers: × batters faced) —
  the line the player would have posted against an average opponent in an average park.
  Outcomes are mutually exclusive, so adjusted counts sum exactly to PA, and **every rate stat
  is computed from the adjusted counts** — each row is an internally consistent box score.
- **RE24** (run value = RE_after − RE_before + runs on the play) is fit the same way, on the
  ~98% of PAs whose after-state is known.
- Unlinked players (non-NCAA opponents, the ~2% of unresolved names) are pooled one level per
  team-season so their PAs still inform their opponents; they aren't written out.

**Batting** — `retro_id,player_id,canonical_id,name,season,division,team,PA,AB,H,1B,2B,3B,HR,BB,`
`IBB,HBP,K,SF,SH,ROE,GDP,TB,AVG,OBP,SLG,OPS,JOPS,ISO,BABIP,K%,BB%,HR%,wOBA,wRAA,wRC+,RE24,`
`raw_OPS,raw_wOBA,raw_wRC+,raw_RE24,opp_wOBA`

**Pitching** — `retro_id,player_id,canonical_id,name,season,division,team,BF,IP,AB,H,1B,2B,3B,`
`HR,BB,IBB,HBP,K,GO,AO,GDP,TB,BAA,oppOBP,oppSLG,oppOPS,oppJOPS,BABIP,K%,BB%,K-BB%,HR%,wOBA,WHIP,TBIP,WPT,FIP,`
`RE24,raw_wOBA,raw_FIP,raw_RE24,opp_wOBA`

| column | meaning |
|---|---|
| PA / BF, IP | actual plate appearances / batters faced and innings (play-by-play; IP in thirds notation, 44.2 = 44⅔) — the only unadjusted volume columns |
| `BB` | **unintentional** walks (IBB is its own column); BB% = (BB+IBB)/PA |
| counts (1B … GDP, GO, AO) | adjusted, 1 decimal, fractional. **Floored at 0**: OLS on a 0/1 outcome (a linear probability model) can push a rare event a player never had below zero (0 HR allowed vs HR-prone opponents → −1.9 HR, making TB < H); truncating moves a row by a fraction of an event, so counts sum to PA to within ~1 |
| AVG … wRC+ | computed from the adjusted counts, same formulas as the advanced files; wOBA weights/scale from that division-season's `linear_weights.csv`. **wRC+ is not re-park-adjusted** — the park is already in the model |
| `RE24` | batters: adjusted RE24 runs (vs avg opponent & park); pitchers: adjusted runs **saved** (positive = good) |
| `FIP` | from adjusted HR/BB/IBB/HBP/K over actual IP, constant set so league FIP = league ERA |
| `WHIP` / `TBIP` / `WPT` | (H+BB+IBB)/IP, (TB+BB+IBB)/IP and their sum, from adjusted counts over actual IP — same definitions as `advanced_pitching.csv` |
| `oppJOPS` | a·oppOBP + oppSLG with the division's JOPS weight |
| `raw_*` | the same stat **unadjusted**, from the same play-by-play PAs, for comparison |
| `opp_wOBA` | strength of schedule: PA-weighted mean adjusted wOBA of the opponents faced — for a batter, the wOBA his pitchers allow (**lower = tougher**); for a pitcher, the wOBA of the batters he faced (**higher = tougher**) |
| GO / AO | ground outs / air outs (fly, line, pop) |

Small samples: an OLS effect for a player with a handful of PAs is mostly noise — filter on PA/BF.
These come from the play-by-play, so totals differ slightly from the official season files.

## "+" stats (all four advanced/adjusted files)

Every rate stat also has a **"+" index** column appended at the end of each row:
**100 = league average for that division-season, and higher is always better for the player**
(the same convention as OPS+, wRC+ and ERA+).

- Stats where more is better use `100 · x / lg` — e.g. AVG+, OBP+, SLG+, JOPS+, wOBA+, a
  hitter's BB%+, a pitcher's K%+.
- Stats where less is better use the ERA+ form `100 · lg / x` — e.g. WHIP+, TBIP+, WPT+, FIP+,
  BAA+, oppJOPS+, a pitcher's BB%+, a **hitter's K%+** (so 120 = strikes out less than average).
- League values are each stat's volume-weighted average over that division-season (AB, PA, BF or
  IP as the weight), so they equal the league ratio; composites are built from parts
  (lgJOPS = a·lgOBP + lgSLG, lgWPT = lgWHIP + lgTBIP).
- Ratio-form indexes (JOPS+, wOBA+, …) have a narrower spread than OPS+, whose formula
  `100·(OBP/lgOBP + SLG/lgSLG − 1)` adds two ratios. In the ERA+ form a very small x makes the
  index very large (a pitcher with a 1% walk rate has BB%+ ≈ 900); x = 0 is left blank.

| file | + columns |
|---|---|
| advanced_batting | AVG+ OBP+ SLG+ ISO+ BABIP+ BB%+ K%+ HR%+ XBH%+ wOBA+ JOPS+ (OPS+, wRC+ already existed) |
| advanced_pitching | WHIP+ TBIP+ WPT+ FIP+ K/9+ BB/9+ HR/9+ H/9+ K%+ BB%+ K-BB%+ BABIP+ LOB%+ BAA+ oppOBP+ oppSLG+ oppOPS+ (ERA+ already existed) |
| adjusted_batting | AVG+ OBP+ SLG+ OPS+ JOPS+ ISO+ BABIP+ K%+ BB%+ HR%+ wOBA+ (wRC+ already existed) |
| adjusted_pitching | BAA+ oppOBP+ oppSLG+ oppOPS+ oppJOPS+ BABIP+ K%+ BB%+ K-BB%+ HR%+ wOBA+ WHIP+ TBIP+ WPT+ FIP+ |

In the **advanced** files the new + stats are league-adjusted but **not park-adjusted** (the
existing OPS+, wRC+, ERA+ and FIP- are; `FIP+ = 10000 / FIP-` so it matches them). In the
**adjusted** files every + stat is opponent- and park-neutral, because the counts underneath are.

## `run_values/adjusted_parks.csv` — park effects from the PA regression

`division,park,games,PA,wOBA,wOBA_index,AVG,OBP,SLG,HR%,K%,BB%` — the δ[park] effects from
the regression above, one row per venue per division (`park` = home team code, or
`NEUTRAL`), shown as the line an average batter vs. an average pitcher would post there.
`wOBA_index` 100 = division-average park, >100 = hitter's park. Unlike `park_factors.csv`
(team runs home vs. road) this controls for exactly who batted and pitched in each game, and
has per-component splits (HR%, K%, BB%). The two agree closely (r ≈ 0.8 in D1).

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
