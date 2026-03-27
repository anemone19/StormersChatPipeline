# StormersChat — Data Download Pipeline Context
## Session Summary: `download_all_data.R` — Oval Insights API Full Data Pipeline
**Project:** UCT Statistical Science StormersChat — Text-to-SQL system for Stormers Rugby coaching staff
**API:** Oval Insights (RugbyViz) — `https://api.rugbyviz.com/rugby/v1` — HTTP Basic Auth
**Season:** 2025/26 (`SEASON_ID = 202501`)
**Competitions:** URC (`comp_id = 1068`), European Rugby Champions Cup (`comp_id = 1008`)
**Stormers team ID:** `3994`
**Output directory:** `/data/` (relative to project root)

---

## Constants & Key IDs

| Constant | Value | Source |
|---|---|---|
| `BASE_URL` | `https://api.rugbyviz.com/rugby/v1` | `.env` / default |
| `URC_ID` | `1068` | `.env` / default |
| `CHAMPS_ID` | `1008` | `.env` / default |
| `SEASON_ID` | `202501` | `.env` / default |
| `STORMERS_ID` | `3994` | `.env` / default |

---

## Helper Infrastructure

- **`oval_get(endpoint, quiet)`** — Authenticated GET with Basic Auth, 30s timeout, 3 retries with exponential backoff. Returns parsed JSON or NULL.
- **`safe_int(x)`** — Returns `NA_integer_` if NULL/empty, else `as.integer(x[[1]])`
- **`safe_chr(x)`** — Returns `NA_character_` if NULL/empty, else `as.character(x[[1]])`
- **`safe_dbl(x)`** — Returns `NA_real_` if NULL/empty, else `as.double(x[[1]])`
- **`safe_lgl(x)`** — Returns `NA_integer_` if NULL/empty, else `as.integer(as.logical(x[[1]]))`

---

## Downloaded Dataframes (Section A)

### 1. `matches` → `data/matches.csv`
**API endpoint:** `GET /match/search?compId={id}&seasonId={id}`
**Description:** All fixtures and results for URC + Champions Cup 2025/26.
**Key columns:**
- `match_id` (PK), `date_time`, `season_id`, `season_name`, `competition_id`, `competition`
- `round`, `match_status`, `finalised` (0/1)
- `home_team_id`, `home_team`, `home_team_short`, `home_team_abbr`
- `home_score_ht`, `home_score_ft`
- `away_team_id`, `away_team`, `away_team_short`, `away_team_abbr`
- `away_score_ht`, `away_score_ft`
- `winner_team_id`, `venue`, `venue_id`, `venue_country`, `venue_timezone`, `venue_floodlit`, `neutral_venue`, `attendance`

**Convenience vectors derived from `matches`:**
- `finalised_ids` — `match_id` values where `finalised == 1`
- `stormers_match_ids` — finalised match IDs involving DHL Stormers (`team_id == 3994`)

---

### 2. `player_match_stats` → `data/player_match_stats.csv`
**API endpoint:** `GET /matchstats/{match_id}` (once per finalised match)
**Description:** Per-player performance statistics for every finalised match. ~6,992 rows.
**Key columns:**
- `match_id` (FK → matches), `player_id` (FK → players), `player_name`
- `team_id`, `team_name`, `is_home_team`, `position_id`, `position_name`, `shirt_number`
- **Scoring/time:** `minutes_played`, `minutes_played_first_half`, `minutes_played_second_half`, `tries_scored`, `points_scored`, `conversions_made`, `penalties_made`, `drop_goals_made`
- **Discipline:** `penalties_conceded`, `yellow_cards`, `red_cards`
- **Attack:** `carries`, `metres_made`, `passes`, `defenders_beaten`, `crossed_gainline`, `failed_gainline`, `turnovers_conceded`, `post_contact_metres`, `try_assists`, `break_assists`
- **Defence:** `tackles_made`, `tackles_missed`, `total_tackles_attempted`, `tackle_success_pct`, `turnovers_won`, `jackals`, `dominant_tackles`
- **Lineout:** `lineout_throws_won`, `lineout_throws_lost`, `lineout_won_clean`, `lineout_won_other`, `throw_front`, `throw_middle`, `throw_back`
- **Kicking:** `kicks_in_play`, `kick_metres`, `territorial_kicks`
- **Rucks:** `ruck_entries_attacking`, `ruck_entries_defensive`, `ruck_entries_total`, `ruck_nuisance`
- **Scrum:** `scrum_involvement_total`, `scrum_won_total`, `scrum_won_penalty`, `scrum_won_outright`
- **Ball Reception:** `defensive_catches`, `defensive_catch_success`, `restart_catches`, `restart_catch_success`

**Note:** API omits zero-value fields; all stats use `safe_int`/`safe_dbl` to handle missing keys gracefully.

---

### 3. `match_team_stats` → `data/match_team_stats.csv`
**API endpoint:** `GET /matchstats/{match_id}` (same call as player stats, different extraction)
**Description:** Per-team aggregated stats for every finalised match. ~304 rows (2 teams × ~152 matches).
**Key columns:**
- `match_id` (FK → matches), `team_id`, `team_name`, `is_home_team`
- **Attack:** `carries`, `metres_made`, `passes`, `defenders_beaten`, `clean_breaks`, `crossed_gainline`, `failed_gainline`, `offloads`, `post_contact_metres`, `try_assists`
- **Scoring:** `tries_scored`, `points_scored`, `conversions_made`, `penalties_made`
- **Defence:** `tackles_made`, `tackles_missed`, `total_tackles_attempted`, `tackle_success_pct`, `turnovers_won`, `dominant_tackles`
- **Lineout:** `lineout_throws_won`, `lineout_throws_lost`, `lineout_success_pct` (computed or API)
- **Scrum:** `scrum_won`, `scrum_lost` (computed), `scrum_success_pct`
- **Kicking:** `kicks_in_play`, `kick_metres`
- **Discipline:** `penalties_conceded`
- **Possession & territory:** `rucks_won`, `rucks_lost`, `mauls_won`, `mauls_lost`, `maul_success_pct`, `visits_to_22`, `points_from_22_visits`, `possession_pct`, `possession_pct_fh`, `possession_pct_sh`, `territory_pct`, `territory_pct_fh`, `territory_pct_sh`
- **Restarts:** `total_restarts`, `restarts_retained`, `restarts_50m`, `restarts_goal_line`, `restart_success_pct`
- **Ball Reception:** `attacking_catches`, `attacking_catch_success`, `defensive_catches`, `defensive_catch_success`, `restart_catches`, `restart_catch_success`, `aerial_contests`, `aerial_contests_won`, `aerial_contest_pct`

---

### 4. `players` → `data/players.csv`
**Source:** Extracted as a side product of the match stats loop
**Description:** Deduplicated player registry built from all players seen in match stats. ~1,336 rows.
**Key columns:**
- `player_id` (PK), `name`, `known_name`, `first_name`, `last_name`

---

### 5. `standings` → `data/standings.csv`
**API endpoint:** `GET /standing/search?compId={id}&seasonId={id}`
**Description:** Current league table for URC and Champions Cup. ~40 rows.
**Key columns:**
- `competition_id`, `competition`, `season_id`
- `position`, `team_id` (FK → teams), `team_name`, `team_short_name`
- `played`, `won`, `drawn`, `lost`, `byes`
- `points_for`, `points_against`, `point_diff`, `tries_for`, `tries_against`
- `bonus_pts_win`, `bonus_pts_try`, `bonus_pts_loss`, `league_points`
- `points_percentage`, `points_scored_pct`

---

### 6. `round_standings` → `data/round_standings.csv`
**API endpoint:** `GET /roundstanding/search?compId={id}&seasonId={id}&round={n}`
**Description:** League table snapshot after every completed round. ~304 rows.
**Key columns:** Identical structure to `standings` plus `round` column.
- `competition_id`, `competition`, `season_id`, `round`
- All same team/points/tries columns as `standings`
**Deduplication:** `distinct(competition_id, round, team_id)`

---

### 7. `season_stats` → `data/season_stats.csv`
**API endpoint:** `GET /seasonstats/search?compId={id}&seasonId={id}&coverage=performance`
**Description:** Season-aggregate team statistics per competition. ~40 rows.
**Key columns:**
- `competition_id`, `competition`, `season_id`, `team_id`, `team_name`
- **Attack (17 cols):** `total_carries`, `total_metres_made`, `total_passes`, `total_clean_breaks`, `total_defenders_beaten`, `total_try_assists`, `total_offloads`, `total_turnovers_conceded`, `total_post_contact_metres`, `total_break_assists`, `total_dominant_carries`, `total_carry_distance`, `total_metres_in_contact`, `total_crossed_gainline`, `total_failed_gainline`, `total_handling_errors`, `total_errors`
- **Scoring (8 cols):** `total_tries`, `total_points`, `total_conversions`, `total_penalties_made`, `total_missed_conversions`, `total_missed_penalties`, `total_penalty_tries`, `goal_success_pct`
- **Defence (4 cols):** `total_tackles_made`, `total_tackles_missed`, `tackle_success_pct`, `total_turnovers_won`
- **Lineout:** `lineout_throws_won`, `lineout_success_pct`
- **Kicking:** `total_kicks_in_play`, `total_kick_metres`
- **Scrum:** `scrums_won`, `scrum_success_pct`
- **Discipline:** `total_penalties_con`, `total_yellow_cards`, `total_red_cards`
- **Possession (8 cols):** `rucks_won`, `rucks_lost`, `mauls_won`, `mauls_lost`, `maul_success_pct`, `visits_to_22`, `points_from_22_visits`, `possession_pct`, `territory_pct`
- **Restarts:** `total_restarts`, `restarts_retained`, `restart_success_pct`
- **Ball Reception (7 cols):** `attacking_catches`, `attacking_catch_success`, `defensive_catches`, `restart_catches`, `aerial_contests`, `aerial_contests_won`, `aerial_contest_pct`
- **Rucks (5 cols):** `ruck_entries_attacking`, `ruck_entries_defensive`, `ruck_entries_total`, `ruck_nuisance`, `ruck_nuisance_jackal`

---

### 8. `squad` → `data/squad.csv`
**API endpoint:** `GET /team/{team_id}` (once per unique team, ~40 teams)
**Description:** Full squad rosters for all teams across both competitions. ~1,740 rows.
**Key columns:**
- `competition_id`, `competition`, `team_id`, `team_name`
- `player_id` (FK → players), `player_name`, `known_name`, `first_name`, `last_name`
- `position`, `gender`, `date_of_birth`, `birthplace`, `country_of_birth`, `national_team`
- `height_cm`, `weight_kg`

**Implementation note:** Team IDs derived from `matches` (no dedicated teams endpoint); squad found at `squad_raw$teams[[1]]$squad`.

---

### 9. `match_events` → `data/match_events.csv`
**API endpoint:** `GET /matchevents/{match_id}` (once per finalised match)
**Description:** Full event-by-event match timeline for all finalised matches. ~358,692 rows. 27 unique event types.
**Key columns:**
- `match_id` (FK → matches), `event_id`, `event_type`, `event_type_id`
- `time`, `minute`, `second`, `period`
- `team_id`, `team_name` (present for most event types; NA for structural events like "period")
- `player_id`, `player_name` (present when a specific player is involved; NA otherwise)
- `score_home`, `score_away`, `start_x`, `start_y`, `end_x`, `end_y`, `direction`
- `properties` — pipe-separated `name=value` string capturing all action-detail flags and metrics (e.g., `carry metres=3 | crossed gainline= | tackled=`)
- `is_deleted`, `timestamp`, `last_modified`

**Event types (27):** period, sequence, restart, ball reception, possession, carry, tackle, ruck, ruck entry, pass, turnover won, turnover, scrum, penalty conceded, kick, lineout throw, lineout contest, attacking quality, missed tackle, goal kick, advantage, maul, try, ref review, substitution, card, sinbin end

**Properties note:** 320 unique property names across all events. Properties are boolean flags (empty value = true) or numeric metrics. Team/player data exists as top-level fields for applicable event types, NOT inside properties.

---

### 10. `match_quarters` → `data/match_quarters.csv`
**API endpoint:** `GET /matchquarters/{match_id}?quarter={q1|q2|q3|q4}` (4 calls per match)
**Description:** Team stats broken down by quarter for all finalised matches.
**Key columns:**
- `match_id` (FK → matches), `quarter` (q1/q2/q3/q4)
- `home_team_id`, `home_team_name`, `away_team_id`, `away_team_name`
- `stat_key`, `stat_label`, `home_value`, `away_value`, `home_share_pct`, `away_share_pct`

**Implementation note:** Stats accessed as `raw$statsQ1`, `raw$statsQ2`, etc. (capital Q).

---

### 11. `head_to_head` → `data/head_to_head.csv`
**API endpoint:** `GET /headtohead/{match_id}` (Stormers matches only)
**Description:** Historical head-to-head summary between the two teams playing in each Stormers match.
**Key columns:**
- `match_id` (FK → matches)
- `home_team_id`, `home_team_name`, `away_team_id`, `away_team_name`
- `total_matches`, `home_wins`, `away_wins`, `draws`
- `home_win_pct`, `away_win_pct`, `draw_pct`
- `biggest_win_margin`, `wins_at_venue`, `neutral_wins`
- `total_points_scored`, `total_tries_scored`
- `last_5_matches` (serialised string), `form` (serialised string)

---

### 12. `venues` → `data/venues.csv`
**API endpoint:** `GET /venue/search?compId={id}&seasonId={id}`
**Description:** Venue metadata for all stadiums used in both competitions. 41 rows after deduplication.
**Key columns:**
- `venue_id` (PK), `venue_name`, `competition_id`, `competition`
- `location`, `country_id`, `country`, `timezone`
- `capacity`, `roof`, `surface`, `year_built`
- `latitude`, `longitude`

**Deduplication:** `distinct(venue_id, .keep_all = TRUE)`

---

### 13. `officials` → `data/officials.csv`
**API endpoint:** `GET /official/search?compId={id}&seasonId={id}`
**Description:** Match officials (referees, TMOs, assistant referees) across both competitions. 187 rows.
**Key columns:**
- `official_id` (PK), `first_name`, `last_name`, `known_name`, `initials`
- `gender`, `dob`, `country`, `competition_id`, `competition`

**Deduplication:** `distinct(competition_id, official_id, .keep_all = TRUE)`

---

### 14. `player_top10` → `data/player_top10.csv`
**API endpoint:** `GET /playertop10stats?compId={id}&seasonId={id}&stat={stat_string}`
**Description:** Top-10 player leaderboards for 18 statistical categories, both total and per-80-minutes. Two competitions × 18 stats × 2 metric types × up to 10 players.
**Key columns:**
- `competition_id`, `competition`, `stat_name`, `stat_api_key`
- `metric_type` (`total` or `per80`)
- `rank`, `player_id`, `player_name`, `team_id`, `team_name`
- `appearances`, `minutes_played`, `stat_value`

**Stats covered (18):** tries, points, carries, metres_made, metres_per_carry, defenders_beaten, try_assists, tackles_made, tackle_success, missed_tackles, turnovers_won, jackals, lineout_throws, lineout_success, kick_metres, post_contact_metres, offloads, initial_breaks

---

### 15. `team_top_stats` → `data/team_top_stats.csv`
**API endpoint:** `GET /teamtopstats?compId={id}&seasonId={id}&stat={stat_string}`
**Description:** Team leaderboards for the same 18 statistical categories, both total and per-match. Two competitions × 18 stats × 2 metric types × up to all teams.
**Key columns:**
- `competition_id`, `competition`, `stat_name`, `stat_api_key`
- `metric_type` (`total` or `perMatch`) — note: `perMatch` not `per80` as in player endpoint
- `rank`, `team_id`, `team_name`, `stat_value`

---

## Derived Dataframes (Section B)

### `stormers_results`
**Source:** Filtered and mutated from `matches`
**Description:** Stormers-centric view of all their finalised matches.
**Key columns:** `match_id`, `date`, `competition`, `round`, `stormers_home`, `opponent`, `stormers_score`, `opp_score`, `result` (W/L/D), `venue`, `attendance`

### `player_season_totals`
**Source:** Aggregated from `player_match_stats`
**Description:** Season cumulative totals per player across all matches played.
**Key columns:** `player_id`, `player_name`, `team_name`, `matches_played`, `total_minutes`, `tries`, `points`, `carries`, `metres_made`, `passes`, `defenders_beaten`, `try_assists`, `tackles_made`, `tackles_missed`, `turnovers_won`, `turnovers_conceded`, `penalties_conceded`, `yellow_cards`, `lineout_throws_won`, `kicks_in_play`, `kick_metres`, `ruck_entries_total`, `metres_per_carry` (computed), `tackle_success_pct` (computed)
**Sorted:** descending by `tries`

### `stormers_player_season`
**Source:** Filtered from `player_season_totals`
**Description:** Season totals for DHL Stormers players only.
**Filter:** `grepl("Stormers", team_name, ignore.case = TRUE)`

---

## Transformations Applied (Section B)

### B1 — Match result enrichment (applied to `matches`)
- Added `date` — `as.Date(substr(date_time, 1, 10))`
- Added `score_diff` — `home_score_ft - away_score_ft`
- Added `home_win` — integer flag (1 = home win)
- Added `home_result` — character W/L/D

### B2 — Player season aggregation
- Grouped `player_match_stats` by `player_id`, `player_name`, `team_name`
- Summed 20 counting stats with `na.rm = TRUE`
- Computed `metres_per_carry = metres_made / max(carries, 1)` (rounded to 2dp)
- Computed `tackle_success_pct = tackles_made * 100 / max(tackles_made + tackles_missed, 1)` (rounded to 1dp)

### B3 — Stormers filter
- Subset of `player_season_totals` where team name contains "Stormers"

---

## Database / File Relationships

```
matches  ──────────────────────────────┐
  match_id (PK)                        │ FK in all match-level tables
  home_team_id / away_team_id ─────────┼──→ standings.team_id
  venue_id ────────────────────────────┼──→ venues.venue_id
  competition_id ──────────────────────┘
         │
         ├──→ player_match_stats.match_id
         │         └── player_id ──→ players.player_id
         │                       └── player_id ──→ squad.player_id
         ├──→ match_team_stats.match_id
         ├──→ match_events.match_id
         ├──→ match_quarters.match_id
         └──→ head_to_head.match_id (Stormers only)

standings        → team_id, competition_id
round_standings  → team_id, competition_id, round
season_stats     → team_id, competition_id

player_top10     → player_id, team_id, competition_id
team_top_stats   → team_id, competition_id

squad            → player_id (links to players), team_id, competition_id
officials        → official_id, competition_id
venues           → venue_id (links to matches.venue_id)
```

---

## API Field Notes (confirmed from debug runs)

| Table | Field note |
|---|---|
| `matches` | `venue.country` can be flat string OR `{id, name}` object — handled defensively |
| `match_events` | 27 event types; `team`/`player` are top-level fields present only for applicable types (not in `properties`). Properties are 320 named flags/metrics |
| `venues` | No `city` field — location is `location`. `roof`, `surface`, `yearBuilt` confirmed present |
| `officials` | No `name`, `role`, `nationality`, `matchesCount` — uses `firstName`, `lastName`, `knownName`, `initials`, `dob`, `country` |
| `head_to_head` | No `totalMeetings` — field is `totalMatches`. No avg scores or streaks |
| `team_top_stats` | Uses `perMatch` not `per80`. No `appearances` field in team entries |
| `season_stats` | API omits zero-value fields — all extraction uses `safe_*` helpers |
