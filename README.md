# Amazon Movies and TV Shows Data Analysis using SQL

## Overview
This project presents an advanced SQL analysis of the Amazon content dataset using PostgreSQL. It explores global trends, genre dynamics, and business insights by leveraging complex SQL queries, window functions, and data transformation techniques. The analysis aims to deliver actionable intelligence for content strategy and decision-making.

## Objective
 -To extract, analyze, and visualize key patterns and trends in Amazon's global catalogue of movies and TV shows.
 -To demonstrate proficiency in advanced SQL concepts such as CTEs, window functions, and string manipulation.
 -To provide business-relevant insights, including content growth rates, genre popularity, and international distribution, supporting     data-driven recommendations.

## DataSet
 The data used for this project is soucred from kaggle dataset:
 -**Dataset Link:** [Movies dataset](https://www.kaggle.com/datasets/utkarshx27/movies-dataset)

 ## Schema
 ```sql

 DROP TABLE IF EXISTS Amazon;
CREATE TABLE Amazon
(
      show_id VARCHAR(6),	
	  type	VARCHAR(10),
	  title VARCHAR(150),
	  director VARCHAR(208),
	  casts	VARCHAR(1000),
	  country VARCHAR(150),
	  date_added VARCHAR(50),	
	  release_year INT,
	  rating VARCHAR(10),
	  duration	VARCHAR(15),
	  listed_in	VARCHAR(100),
	  description VARCHAR(250)
 );
 ```
## Q1. Find movies that have a rare rating (appearing in less than 5% of all movies) but are produced in the top 3 countries by movie count. List the movie title, country, and rating.

 ``` sql
 WITH movie_counts AS (
    SELECT COUNT(*) AS total_movies FROM Amazon WHERE type = 'Movie'
),
rare_ratings AS (
    SELECT rating
    FROM Amazon
    WHERE type = 'Movie'
    GROUP BY rating
    HAVING COUNT(*) < (SELECT total_movies * 0.05 FROM movie_counts)
),
top_countries AS (
    SELECT country
    FROM Amazon
    WHERE type = 'Movie'
    GROUP BY country
    ORDER BY COUNT(*) DESC
    LIMIT 3
)
SELECT title, country, rating
FROM Amazon
WHERE type = 'Movie'
  AND rating IN (SELECT rating FROM rare_ratings)
  AND country IN (SELECT country FROM top_countries);
 ```
## Q2. For each year, calculate the percentage growth in the number of new titles added to Netflix compared to the previous year.

 ```sql
 WITH yearly_additions AS (
    SELECT
        EXTRACT(YEAR FROM TO_DATE(date_added, 'Month DD, YYYY'))::INT AS year_added,
        COUNT(*) AS titles_added
    FROM Amazon
    WHERE date_added IS NOT NULL
    GROUP BY year_added
    ORDER BY year_added
)
SELECT
    year_added,
    titles_added,
    LAG(titles_added) OVER (ORDER BY year_added) AS previous_year,
    ROUND(
        (titles_added - LAG(titles_added) OVER (ORDER BY year_added))::NUMERIC /
        NULLIF(LAG(titles_added) OVER (ORDER BY year_added), 0) * 100, 2
    ) AS pct_growth
FROM yearly_additions
ORDER BY year_added;
 ```
## Q3. Find the longest consecutive streak of months where at least 100 new titles were added.
```sql
WITH monthly_additions AS (
    SELECT
        TO_CHAR(TO_DATE(date_added, 'Month DD, YYYY'), 'YYYY-MM') AS month_added,
        COUNT(*) AS titles_added
    FROM Amazon
    WHERE date_added IS NOT NULL
    GROUP BY month_added
),
flagged_months AS (
    SELECT month_added,
           titles_added,
           CASE WHEN titles_added >= 100 THEN 1 ELSE 0 END AS is_streak
    FROM monthly_additions
),
streaks AS (
    SELECT month_added,
           is_streak,
           SUM(CASE WHEN is_streak = 0 THEN 1 ELSE 0 END) OVER (ORDER BY month_added) AS streak_group
    FROM flagged_months
)
SELECT streak_group,
       COUNT(*) AS streak_length,
       MIN(month_added) AS streak_start,
       MAX(month_added) AS streak_end
FROM streaks
WHERE is_streak = 1
GROUP BY streak_group
ORDER BY streak_length DESC
LIMIT 1;
```





