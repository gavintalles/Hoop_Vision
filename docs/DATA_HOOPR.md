# DATA_HOOPR.md — The hoopR data layer

This is the reference for everything data-related. If you are about to call a `hoopR`
function, read the relevant section here first.

---

## 1. The hoopR function "grammar"

`hoopR` function names are systematic. Read them left to right:

**`<source>_<league>_<thing>`**

**Prefix tells you the data source** (and therefore its reliability and speed):

| Prefix     | Source                  | Use it for                                   | Caveats                                   |
|------------|-------------------------|----------------------------------------------|-------------------------------------------|
| `load_`    | hoopR-data repo (cached)| **Bulk historical seasons** (pbp, box scores)| Fast, pre-built, safe. **Default choice.**|
| `espn_nba_`| ESPN APIs               | Live games, schedules, rosters, careers      | Stable, broad coverage                    |
| `nba_`     | stats.nba.com           | Deep box scores, advanced & tracking stats   | **Rate-limited, fragile, churns**         |
| `nbagl_`   | NBA G League            | G League stats                               | Niche; not needed for MVP                 |

The rest of the name goes general → specific: `team_season_roster`,
`player_career_stats`, `event_player_box`. If you can describe what you want
("a team's roster, from ESPN"), you can usually guess the function:
`espn_` + `nba_` + `team` + `_roster` → `espn_nba_team_roster()`.

When two functions look like synonyms, **the prefix is the tiebreaker**
(`espn_nba_pbp()` vs `nba_playbyplayv3()` — same question, different source).

---

## 2. The functions this app actually uses

### Bulk loaders (the backbone — used by `scripts/refresh_cache.R`)

```r
load_nba_player_box(seasons)   # one row per player per game  <-- core fantasy table
load_nba_team_box(seasons)     # one row per team per game
load_nba_schedule(seasons)     # one row per game
load_nba_pbp(seasons)          # one row per play (large; only if needed)
```

- `seasons` is a numeric vector, e.g. `2023:most_recent_nba_season()`.
- A "season" is labeled by the year it **ends** (the 2023-24 season is `2024`).
- Wrap calls in `progressr::with_progress({ ... })` to get a progress bar.
- These read pre-built files; pulling several seasons of box scores takes seconds.

### Season helpers

```r
most_recent_nba_season()   # integer of the current/most recent NBA season
```

Always derive the current season from this helper. **Never hardcode a year.**

### ESPN getters (live / current — used for "today" views and rosters)

```r
espn_nba_scoreboard()                      # today's games + scores
espn_nba_schedule(season)                  # schedule
espn_nba_standings(season)                 # standings
espn_nba_teams()                           # team directory (ids, colors, logos)
espn_nba_team_roster(team_id, season)      # roster
espn_nba_player_box(game_id)               # player box for one game (live-capable)
espn_nba_team_box(game_id)                 # team box for one game
espn_nba_player_career_stats(athlete_id)   # career rollup (regular + playoffs)
espn_nba_player_seasons(athlete_id)        # season index for a player
espn_nba_team_schedule(team_id, season)    # one team's schedule
espn_nba_game_all(game_id)                 # list(Plays, Team, Player) for one game
```

### NBA Stats getters (deep stats — use sparingly, always cache)

```r
nba_leagueleaders(...)            # league leaders
nba_leaguedashplayerstats(...)    # full-league player stat dashboard (advanced incl.)
nba_playergamelog(player_id, season)
nba_commonplayerinfo(player_id)   # bio, position, headshot
nba_commonallplayers(season)      # master player list w/ stats.nba.com ids
nba_playbyplayv3(game_id)         # rawest play-by-play
```

> ⚠️ `nba_*` functions hit stats.nba.com, which throttles aggressively and
> occasionally requires custom headers/proxy. **Rules:** never loop them without a
> `Sys.sleep()` between calls; always write results to the cache; treat failures as
> expected and retry with backoff. For MVP, prefer `load_*` and `espn_*`.

---

## 3. Data dictionary — `load_nba_player_box()` (the fantasy core)

One row = one player's line in one game. The columns that matter for fantasy:

| Column                              | Type | Notes                                  |
|-------------------------------------|------|----------------------------------------|
| `game_id`, `game_date`, `season`    |      | game keys (ESPN id namespace)          |
| `season_type`                       | int  | 1 = pre, 2 = regular, 3 = post         |
| `athlete_id`                        | int  | **ESPN** athlete id (see §5 on ids)    |
| `athlete_display_name`              | chr  | player name                            |
| `athlete_position_abbreviation`     | chr  | PG/SG/SF/PF/C                          |
| `athlete_headshot_href`             | chr  | image URL for cards                    |
| `team_id`, `team_abbreviation`      |      | team + colors/logo also present        |
| `opponent_team_abbreviation`        | chr  | for matchup context                    |
| `home_away`                         | chr  | "home"/"away"                          |
| `minutes`                           | dbl  | **may be `NA`** for DNP                 |
| `field_goals_made` / `_attempted`   | int  | for FG% and FGM-based scoring          |
| `three_point_field_goals_made` / `_attempted` | int | 3PM is a common bonus stat   |
| `free_throws_made` / `_attempted`   | int  | for FT% and FTM scoring                |
| `offensive_rebounds`                | int  | some leagues weight OREB separately    |
| `defensive_rebounds`                | int  |                                        |
| `rebounds`                          | int  | total                                  |
| `assists`, `steals`, `blocks`       | int  | core counting stats                    |
| `turnovers`                         | int  | usually negative in scoring            |
| `fouls`                             | int  |                                        |
| `points`                            | int  | core                                   |
| `plus_minus`                        | chr  | **string like "+4"** — parse to int    |
| `starter`, `did_not_play`, `active` | lgl  | filter out `did_not_play == TRUE` for fantasy |
| `ejected`                           | lgl  |                                        |

**Cleaning rules the ingest layer must apply:**
- Filter to the games/season types you want (regular season = `season_type == 2`).
- Drop or flag `did_not_play == TRUE` rows before computing averages.
- Coerce `plus_minus` to integer (strip the `+`).
- Some team-box numeric fields arrive as **character** (e.g. `fast_break_points`,
  `points_in_paint`) — coerce explicitly; don't assume numeric.
- `minutes == NA` ≠ 0 minutes vs DNP — handle intentionally.

`load_nba_team_box()` and `load_nba_pbp()` dictionaries are documented on the hoopR
site; mirror this table for any new table you cache.

---

## 4. The cache (how the app reads data)

The app **never** calls the network at request time. Flow:

```
hoopR (network)  ─►  R/data/ingest_*.R  ─►  data-cache/*.parquet  ─►  R/data/load_*.R  ─►  modules
   (scheduled)         (transform/clean)       (on disk)              (fast read)         (UI)
```

- **Format:** parquet via `arrow` (columnar, fast, compresses well) or a single
  `duckdb` file if we want SQL-style filtering. Start with parquet per table.
- **Layout:** one file per logical table, e.g.
  `data-cache/player_box.parquet`, `data-cache/schedule.parquet`,
  `data-cache/player_index.parquet`, `data-cache/fantasy_player_game.parquet`
  (the last is player_box with scoring columns pre-attached for the default profiles).
- **Refresh:** `scripts/refresh_cache.R` pulls the current season + recent seasons,
  cleans, and rewrites the parquet files. Run nightly via cron or a GitHub Action.
- **`R/data/load_*.R`** functions just `arrow::read_parquet()` (optionally with a
  column/row filter) and return a tibble. They must be cheap and side-effect-free.
- **Staleness:** store a `data-cache/_meta.json` with the last refresh timestamp and
  the season range, and surface "data as of <date>" in the UI footer.

`data-cache/` is **gitignored**. Never commit generated data.

---

## 5. The two-ID-namespace problem (read before any join)

- `load_*` and `espn_nba_*` use **ESPN** ids (`athlete_id`, `team_id`, `game_id`).
- `nba_*` (stats.nba.com) use **different** ids (`PLAYER_ID`, `GAME_ID`, etc.).
- **These are not interchangeable.** Joining on raw id across sources produces silent
  garbage.

For MVP we live entirely in the ESPN id space (because the bulk loaders and ESPN
getters share it). **If** we later pull advanced stats from `nba_*`, build an explicit
crosswalk table (match on normalized name + team + season, verify, store as
`data-cache/player_id_crosswalk.parquet`) and join through it — never on raw id.

---

## 6. Quick reference: which function for which question

| Question                                   | Function                            |
|--------------------------------------------|-------------------------------------|
| Every player's game logs, many seasons     | `load_nba_player_box()`             |
| What's the current season number?          | `most_recent_nba_season()`          |
| Games today / live scores                  | `espn_nba_scoreboard()`             |
| A team's current roster                    | `espn_nba_team_roster()`            |
| A player's career arc                      | `espn_nba_player_career_stats()`    |
| Team directory + colors/logos              | `espn_nba_teams()`                  |
| League-wide advanced player stats          | `nba_leaguedashplayerstats()` (cache!) |
| Player bio / headshot / position           | `nba_commonplayerinfo()` (cache!)   |

---

## 7. Source docs

- Getting started: https://hoopr.sportsdataverse.org/articles/getting-started-hoopR.html
- NBA cookbook: https://hoopr.sportsdataverse.org/articles/nba-cookbook.html
- Function index: https://hoopr.sportsdataverse.org/reference/index.html
- Data repo: https://github.com/sportsdataverse/hoopR-data
