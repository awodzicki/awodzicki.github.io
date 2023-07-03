## Analyzing Bike Share Behavior Using SQL & Tableau
*This project was completed as the final case study in the Google Data Analytics Professional Certificate provided through Coursera. Real data was analyzed from a fictional company,* Cyclistic, *with a focus on identifying the differences between types of riders using the bike sharing service.*

Key Findings:
1. The average ride duration for casual riders is almost twice as long as members.
2. Casual Riders completed more rides during summer months, but the number of rides completed in these months did not exceed the rides completed by members.
3. Member riders use the bike sharing service more consistently the cansual riders across all months and weekdays.

[View the Tableau dashboard here](https://public.tableau.com/views/CyclisticBikeShare_16879150650930/CyclisticBikeShare?:language=en-US&:display_count=n&:origin=viz_share_link)

---

### 1. Scenario

  This report aims to provide insights into the use of Cyclistic bike rentals between annual members and casual riders. The goal is to inform strategic marketing decisions aimed at converting casual riders into annual members. 
	Cyclistic is a bike-share program in Chicago that features more than 5,800 bicycles and 600 docking stations. Cyclistic offers flexibility of its pricing plans including single-ride passes, full-day passes, and annual memberships. Customers who purchase single-ride or full-day passes are referred to as casual riders. Customers who purchase annual memberships are Cyclistic members. Customers who purchase single-ride or full-day passes are referred to as casual riders. Cyclistic’s finance analysts have concluded that annual members are much more profitable than casual riders. The director of marketing believes the company’s future success depends on maximizing the number of annual memberships. 

 **1.1 Business Task**
   The marking team is tasks with identifying the patterns between annual members and casual riders. The insights derived from this analysis will serve as the foundation for a targeted marketing strategy aimed at converting casual riders into annual members. 

**1.2	Data Sources**
  The primary data source for this analysis is [Cyclistic’s historical trip data](https://divvy-tripdata.s3.amazonaws.com/index.html). This dataset provides historical bike trip data and has been made publicly available by Motivate International Inc. The data is organized in CSV files with data available in years, fiscal quarters, and months depending on the year. I will be using the 12 months of data from May 2022 to April 2023 available in 12 CSV files.

### 2. Preparation for data analysis
  The data is organized in rows and columns. Each row represents a record of one bike rental. Each record contains the following fields:
+	Name, Type, Description
+	ride_id, Character varying, The distinct identifier for each ride completed
+	rideable_type, Character varying, The type of bike used
+	started_at, DATETIME, The date and time stamp when the ride began
+	ended_at, DATETIME, The date and time stamp when the ride ended
+	start_station_name, Character Varying, The name of the start station
+	start_station_id, Character Varying, The start station identifier
+	end_station_name, character Varying, The name of the end station
+	end_station_id, Character Varying, The end station identifier
+	start_lat, Numeric, the latitude for the start location of the bike rental
+	start_lng, Numeric, the longitude for the start location of the bike rental
+	end_lat, Numeric, the longitude for the end location of the bike rental
+	end_lng, Numeric, the longitude for the end location of the bike rental
+	member_casual, Character Varying, The type of rider completing the rental: member or casual

**2.1 Uploading to PostrgreSQL**
	The CSV files for 12 months of data with were too large to combine and manipulate within excel, so I used PostgreSQL to import and combine the files into one table. I ensured that there was consistency between the column titles of each file before importing. Combing the 12 CSV files into a single dataset streamlined the analysis process.

**2.2 Preview data**
	I previewed the data by running the query below to gain a better understanding of they formatting of the data and identify any missing data.

```SQL
SELECT *
FROM public.tripdata
LIMIT 100
```
**2.3 Null Values**
  There were several NULL values within the records for the start_station_name, start_station_id, end_station_name, and end_station_id columns. I wanted to determine what percentage of the data contained NULL values to determine if the null values needed to be updated or could be removed. 

I ran a SQL query to count the total number of rows:
```SQL
SELECT COUNT(*)
FROM public.tripdata
```
I ran another query to identify any row that contained a NULL value for the 4 columns:
```SQL
SELECT COUNT(*)
	FROM public.tripdata
	WHERE start_station_name IS NULL
	OR start_station_name IS NULL
	OR end_station_name IS NULL
	OR end_station_id IS NULL;
```
Using the results from these queries, I calculated the percentage of data with NULL values.
1324946 Null Values / 5859061 Total Rows = 0.226
	
About 23% of the data was incomplete. I decided to move forward with removing any row with a NULL value since majority of the data was complete.

**2.4 Start and End Times**
  When completing a bike rental, the date and time stamp for the start time should ocurr before the end time. Any data with an end time before the start time would be inaccurate. I ran a query to identify if any started_at times were after the ended_at times. This resulted in 70 rows. I removed this data, as the ride end time cannot be before ride start time.
```SQL
SELECT COUNT(*)
FROM public.tripdata
WHERE started_at > ended_at ;
```
**2.5 Ride Lengths**
  The dataset included columns for start and end times. I also wanted to include a column for ride length to determine if any rides had unusually short or long ride durations. I created a temporary table with a column for trip_duration and previewed the data.
```SQL
WITH tripdata_withduration as(
	SELECT ride_id,
	rideable_type,
	started_at, 
	ended_at,
	ended_at - started_at AS trip_duration,
	member_casual
	FROM public.tripdata)
SELECT*
FROM tripdata_withduration
ORDER BY trip_duration ASC;
LIMIT 250
```

This query returned several trip_durations as 00:00:00 or only a few seconds. I also sorted by descending trip_duration and found the longest trip duration was 22 days. There were also trip lengths of 7, 6, and 3 days. 
	I decided to only analyze trip durations that were less than a day in length and more than 2 minutes. I created a new table titled tripdata_cleaned with a column for trip_duration, calculating the difference between the end time and start time. I only kept data with trip durations that were greater or equal to 2 minutes and less than 1 day.


### 3. Data Analysis
**3.1 Total rides completed**
Which rider type is completing the most rides/rentals? Members are completing more rides, accounting for 60.5% of all rides in the year.
```SQL
SELECT 
	member_casual AS Rider_type,
	COUNT(*) AS number_of_riders,
	ROUND(
		COUNT(*) * 100.0 / (SELECT COUNT(*) from tripdata_cleaned),
		2
		) AS percentage_of_trips
FROM tripdata_cleaned
WHERE trip_duration < '1 day' AND trip_duration >= '00:00:02'
GROUP BY member_casual
ORDER BY member_casual;
```

**Time Spent Riding**
  Which riders are spending more time riding? Casual riders spent more time riding during the past year, riding 128302.95 more hours than member riders.
```SQL
SELECT
	member_casual AS Rider_type,
	SUM(trip_duration) AS duration_riding,
	CAST(SUM(EXTRACT(epoch from trip_duration) / 60) as DECIMAL(16,2)) AS duration_in_minutes,
	CAST(SUM(EXTRACT(epoch from trip_duration) / 60/60 ) as DECIMAL(16,2)) AS duration_in_hours
FROM tripdata_cleaned
WHERE trip_duration < '1 day' AND trip_duration >= '00:00:02'
GROUP BY member_casual
ORDER BY member_casual;
```

  After identifying that casual riders spent more time riding, I wanted to know what is the average ride length for each rider type? On average, casual riders spend 23 minutes riding and members spend 12 minutes riding. Casual riders spend almost twice as much time riding than members during their rental!
```SQL
SELECT
	member_casual AS rider_type,
	AVG(trip_duration) AS avg_tripduration,
	CAST( AVG(EXTRACT(epoch from trip_duration) / 60) as DECIMAL(12,2)) AS Minutes_avg_tripduration
FROM tripdata_cleaned
WHERE trip_duration < '1 day' AND trip_duration >= '00:00:02'
GROUP BY member_casual
```

**3.3 Seasonal Trends**
	* *What are the most popular months for bike rentals?* *
How many rides are completed each month for each rider type?
July is the most popular month for bike rentals, accounting for 642,522 total rides by casual riders and members. June and August were also popular months. The number of bike rentals in these months correlates to the warmer weather in Chicago, ideal for bike rentals.
```SQL
SELECT
  DATE_PART('month', started_at) AS month,
  COUNT(*) AS total_rides,
  COUNT(CASE WHEN member_casual = 'casual' THEN 1 end) AS casual_rides,
  CAST(COUNT(CASE WHEN member_casual = 'casual' THEN 1 end) * 100.0 / COUNT(*)
	   as DECIMAL(4,2))
	  AS percent_casual_rides,
  COUNT(CASE WHEN member_casual = 'member' THEN 1 end) AS member_rides,
  CAST(COUNT(CASE WHEN member_casual = 'member' THEN 1 end) * 100.0 / COUNT(*)
	   as DECIMAL(4,2))
	  AS percent_member_rides
FROM tripdata_cleaned
GROUP BY DATE_PART('month', started_at)
ORDER BY total_rides DESC
```

  * *Does the time of year impact how much time casual riders and members spend riding?* *
  The average trip length for member riders stays fairly consistent throughout the year, ranging between 9 and 13 minutes. Casual riders spend more time riding during the summer months, with average trip durations longer than 22 minutes in May, June, July, and August. Casual riders spend more time riding in summer months when compared to members.
```SQL
SELECT
  DATE_PART('month', started_at) AS month,
  AVG(CASE WHEN member_casual = 'casual' THEN trip_duration end) AS Casual_avg_ridelength,
  AVG(CASE WHEN member_casual = 'member' THEN trip_duration end) AS Member_avg_ridelength
 FROM tripdata_cleaned
GROUP BY
  DATE_PART('month', started_at)
```

**3.4 Rides by Weekday**
  * *What are the most popular weekdays for bike rides among casual riders and members?* *
  The most bike rides occur on Saturday and the number of rides completed by casual riders exceeds rides completed by members. The number of rides completed by members is more consistent through the week, where the number of casual riders increases on Saturday, Sunday, and Friday. This may be due to casual riders spending time working during the weekdays and renting bikes during their free time on the weekends. Whereas members may be using the bike rentals to commute to and from work during the weekdays.
```SQL
SELECT
  EXTRACT(dow from started_at) AS Weekday,
  COUNT(*) AS total_rides,
  COUNT(CASE WHEN member_casual = 'casual' THEN 1 end) AS casual_rides,
  CAST(COUNT(CASE WHEN member_casual = 'casual' THEN 1 end) * 100.0 / COUNT(*)
	   as DECIMAL(4,2))
	  AS percent_casual_rides,
  COUNT(CASE WHEN member_casual = 'member' THEN 1 end) AS member_rides,
  CAST(COUNT(CASE WHEN member_casual = 'member' THEN 1 end) * 100.0 / COUNT(*)
	   as DECIMAL(4,2))
	  AS percent_member_rides
FROM tripdata_cleaned
GROUP BY EXTRACT(dow from started_at)
ORDER BY EXTRACT(dow from started_at) ASC
```

**3.5 Most Utilized Start Stations**
  * *What start stations are most popular among casual riders and members?* *

```SQL
SELECT 
    DISTINCT start_station_name,
    COUNT(*) AS total_rides,
    SUM(CASE WHEN member_casual = 'member' THEN 1 END) AS member_rides,
    SUM(CASE WHEN member_casual = 'casual' THEN 1 END) AS casual_rides
FROM tripdata_cleaned
WHERE trip_duration < '1 day' AND trip_duration >= '00:00:02' 
GROUP BY start_station_name
ORDER BY total_rides DESC
LIMIT 20;
```

**3.6 Most Utilized End Stations**
  * *What end stations are most popular among casual riders and members?* *




### 4. Conclusions

### 5. Business Reccomendations

### 6. Considerations for further analysis

---

[Back to Portfolio](awodzicki.github.io)

