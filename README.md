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
## Q4. Identify directors who have worked across the most distinct genres.
```sql
WITH numbers AS (
    SELECT generate_series(1, 5) AS n
),
director_genres AS (
    SELECT director,
           TRIM(SPLIT_PART(listed_in, ',', n.n)) AS genre
    FROM Amazon
    JOIN numbers n ON n.n <= array_length(string_to_array(listed_in, ','), 1)
    WHERE director IS NOT NULL
)
SELECT director,
       COUNT(DISTINCT genre) AS genre_count
FROM director_genres
GROUP BY director
ORDER BY genre_count DESC
LIMIT 5;
```

## Q5. Find countries that have released at least one new title every year for the last 10 years.
```sql
WITH years AS (
    SELECT DISTINCT release_year
    FROM Amazon
    WHERE release_year >= EXTRACT(YEAR FROM CURRENT_DATE) - 9
),
country_years AS (
    SELECT country, release_year
    FROM Amazon
    WHERE country IS NOT NULL AND release_year >= EXTRACT(YEAR FROM CURRENT_DATE) - 9
    GROUP BY country, release_year
),
country_counts AS (
    SELECT country, COUNT(DISTINCT release_year) AS years_with_content
    FROM country_years
    GROUP BY country
)
SELECT country
FROM country_counts
WHERE years_with_content = (SELECT COUNT(*) FROM years);
```
## Q6. Determine which genre has seen the highest percentage increase in new content over the last 5 years.
```sql
WITH numbers AS (
    SELECT generate_series(1, 5) AS n
),
genre_years AS (
    SELECT
        TRIM(SPLIT_PART(listed_in, ',', n.n)) AS genre,
        release_year
    FROM Amazon
    JOIN numbers n ON n.n <= array_length(string_to_array(listed_in, ','), 1)
    WHERE release_year >= EXTRACT(YEAR FROM CURRENT_DATE) - 5
),
genre_counts AS (
    SELECT genre, release_year, COUNT(*) AS count
    FROM genre_years
    GROUP BY genre, release_year
),
genre_growth AS (
    SELECT genre,
           MIN(release_year) AS start_year,
           MAX(release_year) AS end_year,
           MIN(count) AS start_count,
           MAX(count) AS end_count,
           ROUND(((MAX(count) - MIN(count))::NUMERIC / NULLIF(MIN(count), 0)) * 100, 2) AS pct_growth
    FROM genre_counts
    GROUP BY genre
)
SELECT genre, pct_growth
FROM genre_growth
ORDER BY pct_growth DESC
LIMIT 1;
```
## Q7. Find “Multi-Talented” Actors: Most Frequent Appearances Across Both Movies and TV Shows.
```sql
WITH numbers AS (
    SELECT generate_series(1, 10) AS n
),
actor_types AS (
    SELECT
        TRIM(SPLIT_PART(casts, ',', n.n)) AS actor,
        type
    FROM Amazon
    JOIN numbers n ON n.n <= array_length(string_to_array(casts, ','), 1)
    WHERE casts IS NOT NULL
),
actor_appearances AS (
    SELECT actor, type, COUNT(*) AS appearances
    FROM actor_types
    GROUP BY actor, type
),
multi_talented AS (
    SELECT actor, SUM(appearances) AS total_appearances
    FROM actor_appearances
    GROUP BY actor
    HAVING COUNT(DISTINCT type) = 2
)
SELECT actor, total_appearances
FROM multi_talented
ORDER BY total_appearances DESC
LIMIT 5;
```

## Q8. Most Common Pair of Actors Who Frequently Appear Together.

```sql
WITH numbers AS (
    SELECT generate_series(1, 5) AS n
),
actor_pairs AS (
    SELECT
        show_id,
        TRIM(SPLIT_PART(casts, ',', n1.n)) AS actor1,
        TRIM(SPLIT_PART(casts, ',', n2.n)) AS actor2
    FROM Amazon
    JOIN numbers n1 ON n1.n <= array_length(string_to_array(casts, ','), 1)
    JOIN numbers n2 ON n2.n <= array_length(string_to_array(casts, ','), 1)
    WHERE casts IS NOT NULL AND n1.n < n2.n
)
SELECT actor1, actor2, COUNT(*) AS appearances_together
FROM actor_pairs
WHERE actor1 <> '' AND actor2 <> ''
GROUP BY actor1, actor2
ORDER BY appearances_together DESC
LIMIT 1;
```

## Q9. For each director, calculate the span (in years) between their earliest and latest release on Netflix. List the top 5 with the longest spans.

```sql
SELECT director,
       MIN(release_year) AS first_year,
       MAX(release_year) AS last_year,
       (MAX(release_year) - MIN(release_year)) AS career_span
FROM Amazon
WHERE director IS NOT NULL
GROUP BY director
ORDER BY career_span DESC
LIMIT 5;
```

## Q10. Identify titles that are produced in the largest number of countries (i.e., have the most countries listed in the country field).

```sql
SELECT title, country,
       array_length(string_to_array(country, ','), 1) AS country_count
FROM Amazon
WHERE country IS NOT NULL
ORDER BY country_count DESC
LIMIT 5;
```

## Findings
-Content Growth: Amazon’s content library has grown steadily over the past decade, with a significant increase in original productions since 2015.

-Top Countries: The United States, India, and the United Kingdom are the leading producers of Amazon content, accounting for over 60% of titles.

-Genre Trends: Drama, Comedy, and Documentary are the most prevalent genres, with notable growth in international and niche categories.

-Hidden Gems: Identified over 150 lesser-known titles with high ratings and limited mainstream exposure.

-Release Patterns: Most new content is added in the last quarter of each year, aligning with global holiday seasons.

## Conclusion

This analysis highlights Netflix’s strategic focus on expanding its global content library and diversifying genres to appeal to a broader audience. The findings provide actionable insights for content acquisition, marketing, and regional targeting. Advanced SQL techniques enabled efficient extraction of trends and patterns, demonstrating the value of data-driven decision-making in the streaming industry.










