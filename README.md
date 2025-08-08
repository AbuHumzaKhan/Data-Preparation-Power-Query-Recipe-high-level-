# Data-Preparation-Power-Query-Recipe-high-level-


Project: IPL 2008–2024 — end-to-end analysis & dashboards (Excel + Power BI + ChatGPT-assisted)

Author: You (visualized and prepared using Excel, Power BI and ChatGPT)

🎯 Project Summary

This project is a polished, interactive visualization suite for Indian Premier League (IPL) data spanning 2008–2024. It contains multiple dashboard pages, each focused on a different level of insight: tournament overview, team performance, player performance, and advanced analytics. The visuals were created with Power BI (desktop) for interactivity and Excel for data exploration and quick summaries. ChatGPT helped with analysis ideas, measure logic, and documentation.

Use cases:

Scout player strengths and weaknesses (batting vs bowling).

Compare teams across seasons and venues.

Find high-impact matches and venues.

Produce resume-grade visuals for interviews and portfolio.

<img width="1053" height="655" alt="Screenshot 2025-08-08 224157" src="https://github.com/user-attachments/assets/aabf0033-57c0-4027-8f5d-7f1e0f4343e0" />

<img width="1050" height="659" alt="Screenshot 2025-08-08 223827" src="https://github.com/user-attachments/assets/b45c63d7-5d89-4c9b-9cc9-4af2f7abe8ea" />

<img width="930" height="625" alt="Screenshot 2025-08-08 223432" src="https://github.com/user-attachments/assets/11d1835e-3252-4201-8c36-4acf986f9615" />

<img width="941" height="633" alt="Screenshot 2025-08-08 223404" src="https://github.com/user-attachments/assets/f263c6dc-7ec3-4b3a-b151-68909b0d5eb6" />


🔎 Dataset Overview

Primary tables used

matches — match-level metadata (match_id, date, season, venue, team1, team2, winner, toss, result type).

deliveries — ball-by-ball events (match_id, inning, over, ball, batsman, bowler, runs_bat, extras, wicket_type, boundary_flag).

players — player lookup (player_id, player_name, role, country).

Key preprocessing performed

Combined season-level CSVs into a single matches and deliveries table.

Normalised team and player names (consistent naming across years).

Converted dates and numeric types, removed mismatched rows and fixed errors flagged in Power Query.

Created derived fields: TotalRuns, BoundaryRuns (4s+6s), IsBoundary, IsWicket, BatsmanStrikeRate, BowlingEconomy.

Missing & inconsistent data — common issues handled:

Different spellings of players/teams → mapping table or fuzzy merge.

Missing venue details → filled when available from matches table by join.

Errors in deliveries (bad numeric values) → removed or coerced using Power Query Try/Otherwise logic.

⚙️ I only have matches.csv and deliveries.csv — how to build the model

Good — having both match-level and ball-by-ball data is everything you need. Below are practical, copy-paste steps to create the core dimension tables (Team, Player, Season) and the measures/tables you need in Power BI using Power Query + DAX.

1) Power Query: minimal ETL to create Teams & Players tables

A. Load files

Home → Get Data → Text/CSV and load matches.csv and deliveries.csv.

B. Create Teams table (unique canonical team names)

From matches query: select team1, team2, venue (optional).

Unpivot / Append team1 and team2 into one column (use Transform → Unpivot Columns or create a new query with Table.Union).

Remove duplicates and trim text: Home → Remove Rows → Remove Duplicates; Transform → Format → Trim.

Add TeamID (Index Column starting at 1) and set Load = To model as Teams.

C. Create Players table (unique batters & bowlers)

From deliveries query: select batsman, bowler, fielder (if present).

Unpivot/append these columns into one player_name column.

Remove duplicates, trim, add PlayerID index.

(Optional) Add PrimaryRole by computing counts: Group By player_name → count of times appears as batsman vs bowler; pick role with higher count.

D. Create Matches table (clean canonical keys)

In matches query: trim team names and map using Teams table (Merge Queries → left join) to add Team1ID, Team2ID.

Convert date to Date type, season to text/number.

E. Enrich deliveries

Trim player/team names to canonical form and Merge with Players and Teams to add IDs.

Add calculated columns:

is_boundary = if batsman_runs in {4,6} then 1 else 0

boundary_runs = if boundary then batsman_runs else 0

is_wicket = if wicket_player_id not null then 1 else 0 (or use wicket_kind)

Create total_runs = batsman_runs + extra_runs (if not already present).

Power Query tip: create one heavy deliveries_clean query, then create small aggregated summary queries (player-season, team-season) using Group By and turn Enable Load off for intermediate steps.

2) Data Model relationships (recommended)

matches[match_id] (1) -> deliveries[match_id] (*)

Teams[TeamID] (1) -> matches[Team1ID] (*) and matches[Team2ID] (*)

Players[PlayerID] (1) -> deliveries[batsman_id] (*)

Players[PlayerID] (1) -> deliveries[bowler_id] (*)

Use single-direction relationships from dimension → fact. Hide the heavy deliveries table from Report view and expose summary tables.

3) Key DAX measures to create now (copy/paste)

Core counters

Total Runs = SUM(deliveries_clean[total_runs])

Total Matches = DISTINCTCOUNT(matches[match_id])

Total Boundaries = SUM(deliveries_clean[boundary_runs])

Total Sixes = CALCULATE(COUNTROWS(deliveries_clean), deliveries_clean[batsman_runs] = 6)

Total Fours = CALCULATE(COUNTROWS(deliveries_clean), deliveries_clean[batsman_runs] = 4)

Total Wickets = SUM(deliveries_clean[is_wicket])

Aggregates by player (examples)

Player Runs = CALCULATE(SUM(deliveries_clean[batsman_runs]), ALLEXCEPT(Players, Players[PlayerID]))

Player Balls = CALCULATE(COUNTROWS(deliveries_clean), deliveries_clean[batsman_id] = SELECTEDVALUE(Players[PlayerID]))

Player Strike Rate = DIVIDE([Player Runs], [Player Balls]) * 100

Player 4s = CALCULATE(COUNTROWS(deliveries_clean), deliveries_clean[batsman_id] = SELECTEDVALUE(Players[PlayerID]) , deliveries_clean[batsman_runs] = 4)

Player 6s = CALCULATE(COUNTROWS(deliveries_clean), deliveries_clean[batsman_id] = SELECTEDVALUE(Players[PlayerID]) , deliveries_clean[batsman_runs] = 6)


Bowling Wickets = CALCULATE(SUM(deliveries_clean[is_wicket]), deliveries_clean[bowler_id] = SELECTEDVALUE(Players[PlayerID]))

Bowling Balls = CALCULATE(COUNTROWS(deliveries_clean), deliveries_clean[bowler_id] = SELECTEDVALUE(Players[PlayerID]))

Bowling Overs = DIVIDE([Bowling Balls], 6)

Bowling Economy = DIVIDE(CALCULATE(SUM(deliveries_clean[total_runs_conceded]), deliveries_clean[bowler_id] = SELECTEDVALUE(Players[PlayerID])), [Bowling Overs])

Aggregates by team (examples)

Team Total Runs = CALCULATE([Total Runs], FILTER(ALL(matches), matches[Team1ID] = SELECTEDVALUE(Teams[TeamID]) || matches[Team2ID] = SELECTEDVALUE(Teams[TeamID])))

Team Wins = CALCULATE(COUNTROWS(matches), matches[winner] = SELECTEDVALUE(Teams[TeamName]))

Note: Team Wins may require a normalized winner_id in matches (map winner name to TeamID).

4) Aggregation tables (performance)

Create pre-aggregated tables in Power Query to speed visuals:

player_season_summary (player_id, season, runs, balls, 4s, 6s, wickets, overs, economy)

team_season_summary (team_id, season, runs_scored, runs_conceded, wins, losses)

Build these with Group By on the cleaned deliveries_clean and matches tables and load them to the model. Use these tables for charts and tables instead of raw deliveries.

Dashboard Pages & What They Show

1) Tournament Summary Overview

Large title banners show stadium, top team, and season context. Core elements:

KPI tiles: Total Runs, Total Wickets, Total Sixes, Total Matches.

Trend line: total matches per season (to detect format changes).

Match table: clickable list of matches filtered by venue / team / season.

Best for: High-level storytelling & venue analysis.

2) Team Performance Analysis

Focus: team aggregates across seasons.

Stacked bar/time-series: total runs by team & season.

Bar charts: Total Wins / Losses / Draws and rank by Wins.

Table: team-level metrics — matches, wins, losses, win %, total wickets.

Best for: board-level summaries and season-by-season comparisons.

3) Player Performance Insight

Split view for Bowler and Batsman:

Top KPI banners with player name & role.

Bar charts: Total Wickets (by bowler), Total Runs (by batter).

Tables with sortable stats: average, strike rate, totals, 50s/100s.

Best for: scouting players and preparing player cards for presentations.

4) Advanced Analytics

Deeper metrics and distributions:

Bar/line combo: Economy and Bowling Average by bowler.

Trend: Strike Rate vs Average Runs by batter.

Scatterplots: Strike Rate vs Average; Economy vs Bowling Average (identify outliers / all-rounders).

Best for: advanced hiring / analytics interviews, model-features selection.

Key Metrics & Example Measures

Below are the most important measures — ready to paste into Power BI as DAX. Adjust column/table names to match your model.

Basic totals

Total Runs = SUM(deliveries[runs_total])
Total Sixes = CALCULATE(COUNTROWS(deliveries), deliveries[batsman_runs] = 6)
Total Fours = CALCULATE(COUNTROWS(deliveries), deliveries[batsman_runs] = 4)
Total Wickets = CALCULATE(COUNTROWS(deliveries), NOT(ISBLANK(deliveries[wicket_player_id])))
Total Extras = SUM(deliveries[extra_runs])
Unique Players = DISTINCTCOUNT(players[player_id])

Runs by Boundaries (4s + 6s)

Runs by Boundaries =
SUMX(
    FILTER(deliveries, deliveries[batsman_runs] = 4 || deliveries[batsman_runs] = 6),
    deliveries[batsman_runs]
)

Player-specific aggregates (example for batter)

Batter Runs = CALCULATE(SUM(deliveries[batsman_runs]), deliveries[batsman = SELECTEDVALUE(players[player_name])))
Batter Strike Rate =
DIVIDE(
    [Batter Runs],
    CALCULATE(COUNTROWS(deliveries), deliveries[batsman = SELECTEDVALUE(players[player_name])])
) * 100

Bowling economy (by bowler)

Bowling Economy =
DIVIDE(
    CALCULATE(SUM(deliveries[total_runs_conceded]), deliveries[bowler] = SELECTEDVALUE(players[player_name])),
    CALCULATE(SUMX(deliveries, 1/6), deliveries[bowler] = SELECTEDVALUE(players[player_name]))
)

(Note: adjust denominator to overs calculation — typically TotalBalls/6)

Tip: Create TotalBalls measure (COUNT of legal deliveries) then use TotalOvers = TotalBalls / 6 for accurate economy.

Power Query Recipe (high level) section in your README with a clear, step-by-step Power Query workflow:

Strategy & mode recommendations

Combine & load instructions

Standard cleanups to run on every query

Canonicalization & mapping guidance (teams & players)

Exact steps to create deliveries_clean with calculated columns and error handling

How to build aggregated summary tables (player_season_summary, team_season_summary) for fast visuals

Final model hygiene and practical Power Query tips

Open the canvas document to view the updated section. If you want, I can now:

paste ready-to-use Power Query M snippets for parts (e.g., unpivot teams, create Players table, boundary flag), or

generate the exact Group By steps (and M) to produce player_season_summary.

