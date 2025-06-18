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

 ##Schema
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

