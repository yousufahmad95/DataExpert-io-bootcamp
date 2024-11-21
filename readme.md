

### Day 1: Dimensional data modeling


Lab notes/queries:
```sql
-- Select COUNT(*) from player_seasons  -- 12869

Create a new type
CREATE TYPE season_stats AS (
	season INTEGER,
	gp INTEGER,
	pts REAL,
	reb REAL,
	ast REAL
)

CREATE TABLE players (
	player_name TEXT,
	height TEXT,
	college TEXT,
	country TEXT,
	draft_year TEXT,
	draft_round TEXT,
	draft_number TEXT,
	season_stats season_stats[],
	current_season INTEGER,
	PRIMARY KEY(player_name, current_season)
)

-- Get minimum year in the 'season' column
SELECT MIN(season) FROM player_seasons;

-- Start generating cumulative table
INSERT INTO players
WITH yesterday AS (
	SELECT * FROM players WHERE current_season = 2000
),
today AS (
	SELECT * FROM player_seasons WHERE season = 2001
)

-- SELECT * from today t FULL OUTER JOIN yesterday y ON t.player_name = y.player_name

SELECT 
	COALESCE(t.player_name, y.player_name) AS player_name,
	COALESCE(t.height, y.height) AS height,
	COALESCE(t.college, y.college) AS college,
	COALESCE(t.country, y.country) AS country,
	COALESCE(t.draft_year, y.draft_year) AS draft_year,
	COALESCE(t.draft_round, y.draft_round) AS draft_round,
	COALESCE(t.draft_number, y.draft_number) AS draft_number,
	CASE WHEN y.season_stats IS NULL 
	THEN ARRAY[ROW(t.season, t.gp, t.pts, t.reb, t.ast)::season_stats]
	WHEN t.season IS NOT NULL THEN y.season_stats || ARRAY[ROW(t.season, t.gp, t.pts, t.reb, t.ast)::season_stats]
	ELSE y.season_stats
	END as season_stats,
	COALESCE(t.season, y.current_season + 1) as current_season
	-- CASE WHEN t.season IS NOT NULL THEN t.season
	-- ELSE y.current_season + 1
	from today t FULL OUTER JOIN yesterday y ON t.player_name = y.player_name;
-- END

-- Unnest season_stats values to multiple rows
SELECT player_name, UNNEST(season_stats) as season_stats from players where current_season = 2001 AND player_name = 'Michael Jordan';

-- Unnest 'season_stats' values to seperate rows and columns
WITH unnested AS 
(
 SELECT player_name,
 		UNNEST(season_stats)::season_stats AS season_stats
		FROM players
		WHERE current_season = 2001
		AND player_name = 'Michael Jordan'
)

SELECT player_name, 
	   -- (season_stats::season_stats).pts
	   (season_stats::season_stats).*
FROM unnested	   
-- END

-- Part 2
-- START BY DELETING 'players' table and add 2 extra columns to get score details per player

-- DROP TABLE players

CREATE TYPE scoring_class AS ENUM('star', 'good', 'average', 'bad');

CREATE TABLE players (
	player_name TEXT,
	height TEXT,
	college TEXT,
	country TEXT,
	draft_year TEXT,
	draft_round TEXT,
	draft_number TEXT,
	season_stats season_stats[],
	scoring_class scoring_class,
	years_since_last_season INTEGER,
	current_season INTEGER,
	PRIMARY KEY(player_name, current_season)
)

INSERT INTO players
WITH yesterday AS (
	SELECT * FROM players WHERE current_season = 2000
),
today AS (
	SELECT * FROM player_seasons WHERE season = 2001
)

-- SELECT * from today t FULL OUTER JOIN yesterday y ON t.player_name = y.player_name

SELECT 
	COALESCE(t.player_name, y.player_name) AS player_name,
	COALESCE(t.height, y.height) AS height,
	COALESCE(t.college, y.college) AS college,
	COALESCE(t.country, y.country) AS country,
	COALESCE(t.draft_year, y.draft_year) AS draft_year,
	COALESCE(t.draft_round, y.draft_round) AS draft_round,
	COALESCE(t.draft_number, y.draft_number) AS draft_number,
	CASE WHEN y.season_stats IS NULL 
	THEN ARRAY[ROW(t.season, t.gp, t.pts, t.reb, t.ast)::season_stats]
	WHEN t.season IS NOT NULL THEN y.season_stats || ARRAY[ROW(t.season, t.gp, t.pts, t.reb, t.ast)::season_stats]
	ELSE y.season_stats
	END as season_stats,
	CASE WHEN t.season IS NOT NULL THEN 
		CASE WHEN t.pts > 20 THEN 'star'
			 WHEN t.pts > 15 THEN 'good'
			 WHEN t.pts > 10 THEN 'average'
			 ELSE 'bad'
		END::scoring_class
		ELSE y.scoring_class
 	END as scoring_class,
	CASE WHEN t.season IS NOT NULL THEN 0
	ELSE y.years_since_last_season + 1
	END as years_since_last_season,
	COALESCE(t.season, y.current_season + 1) as current_season
	-- CASE WHEN t.season IS NOT NULL THEN t.season
	-- ELSE y.current_season + 1
	from today t FULL OUTER JOIN yesterday y ON t.player_name = y.player_name


SELECT * FROM players WHERE current_season = 2001;

-- CHECK the players that improved the most from first season to current season
SELECT player_name,
		(season_stats[CARDINALITY(season_stats)]::season_stats).pts/
		CASE WHEN (season_stats[1]::season_stats).pts = 0 THEN 1 ELSE (season_stats[1]::season_stats).pts END as improvement
		-- IF((season_stats[1]::season_stats).pts = 0, 1, (season_stats[1]::season_stats).pts) -- didnot work
FROM players
WHERE current_season = 2001		
ORDER BY 2 DESC
```