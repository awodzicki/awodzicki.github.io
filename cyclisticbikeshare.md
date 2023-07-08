## Analyzing Bike Rental Behavior Using SQL & Tableau
*This project was completed as the final case study in the Google Data Analytics Professional Certificate provided through Coursera. Real data was analyzed from a fictional company,* Cyclistic, *with a focus on identifying how casual riders and annual members use the bike sharing service.*

Key Findings:
1. Casual riders rent bikes for longer durations than member riders, with an average ride duration of 23 minutes.
2. Casual riders complete more trips on weekends and in summer months.
3. Member riders complete more rides during summer months and more rides during weekdays.

[View the Tableau dashboard here](https://public.tableau.com/views/CyclisticBikeShare_16879150650930/AnalyzingBikeSharingData?:language=en-US&:display_count=n&:origin=viz_share_link)

---

### 1. Scenario

  This report aims to provide insights into the use of Cyclistic bike rentals between annual members and casual riders. The goal is to inform strategic marketing decisions aimed at converting casual riders into annual members. 
	Cyclistic is a bike-share program in Chicago that features more than 5,800 bicycles and 600 docking stations. Cyclistic offers flexibility of its pricing plans including single-ride passes, full-day passes, and annual memberships. Customers who purchase single-ride or full-day passes are referred to as casual riders. Customers who purchase annual memberships are Cyclistic members. Customers who purchase single-ride or full-day passes are referred to as casual riders. Cyclistic’s finance analysts have concluded that annual members are much more profitable than casual riders. The director of marketing believes the company’s future success depends on maximizing the number of annual memberships. 

 **1.1 Business Task**
   The marking team is tasks with identifying the patterns between annual members and casual riders. The insights derived from this analysis will serve as the foundation for a targeted marketing strategy aimed at converting casual riders into annual members. 

**1.2	Data Sources**
  The primary data source for this analysis is [Cyclistic’s historical trip data](https://divvy-tripdata.s3.amazonaws.com/index.html). This dataset provides historical bike trip data and has been made publicly available by Motivate International Inc. The data is organized in CSV files with data available in years, fiscal quarters, and months depending on the year. 12 months of data was analyzed from May 2022 to April 2023, which was available in 12 CSV files.

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
	The CSV files for 12 months of data with were too large to combine and manipulate within excel, so I used PostgreSQL to import and combine the files into one table. I ensured that there was consistency between the column titles of each file before importing and combing the 12 CSV files into a single table to query all data at once.

**2.2 Preview data**
	I previewed the data with the query below to gain a better understanding of the formatting and identify any missing data.

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
	
About 23% of the data was incomplete. I decided to move forward with removing any row with a NULL value since majority of the data was complete. Any row that contained a NULL value for start_station_name, start_station_id, end_station_name, or end_station_id was removed.

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
	I decided to only analyze trip durations that were less than a day in length and 2 minutes or longer. I created a new table titled tripdata_cleaned with a column for trip_duration, calculating the difference between the end time and start time. I only added data to this table with trip durations that were greater or equal to 2 minutes and less than 1 day.

### 3. Data Analysis
**3.1 Total rides completed**
*Which rider type is completing the most rides overall?* 
Members are completing more rides, accounting for 60.5% of all rides in the year.
<img src="images/cyclist/3.1 Total Rides.png"/>

**Time Spent Riding**
  *Which riders are spending more time riding?*
  Casual riders spent more time riding during the past year, riding 128302.95 more hours than member riders.
<img src="images/cyclist/3.2 Total Riding Time.png"/>

  After identifying that casual riders spent more time riding, I wanted to explore the average ride length for each rider type. 
<img src="images/cyclist/3.2 Average Trip Duration.png"/>
On average, casual riders spend 23 minutes riding and members spend 12 minutes riding. Casual riders spend almost twice as much time riding than members during their rental!

**3.3 Seasonal Trends**
  *What are the most popular months for bike rentals? How many rides are completed each month for each rider type?*
July is the most popular month for bike rentals, accounting for 642,522 total rides by casual riders and members. June and August were also popular months. The number of bike rentals in these months correlates to the warmer weather in Chicago, ideal for bike rentals.
<img src="images/cyclist/3.3 Monthly Number of Rides.png"/>

*Does the time of year impact how much time casual riders and members spend riding?*
  The average trip length for member riders stays fairly consistent throughout the year, ranging between 9 and 13 minutes. Casual riders spend more time riding during the summer months, with average trip durations longer than 22 minutes in May, June, July, and August. Casual riders spend more time riding in summer months when compared to members.
<img src="images/cyclist/3.3 Monthly Trip Duration.png"/>

**3.4 Rides by Weekday**
  *What are the most popular weekdays for bike rides among casual riders and members?*
  Overall, the most bike rides occur on Saturday and this is the only day where the number of rides completed by casual riders exceeds rides completed by members. The number of rides completed by members is more consistent through the week, where the number of casual riders increases on Saturday, Sunday, and Friday. This may be due to casual riders spending time working during the weekdays and renting bikes during their free time on the weekends. Whereas members may be using the bike rentals to commute to and from work during the weekdays.
<img src="images/cyclist/3.4 Weekday Rides.png"/>

**3.5 Most Utilized Start Stations**
  *What start stations are most popular among casual riders and members?*
<img src="images/cyclist/3.5 Top Start Stations.png"/>

**3.6 Most Utilized End Stations**
*What end stations are most popular among casual riders and members?*
<img src="images/cyclist/3.6 Top End Stations.png"/>

### 4. Conclusions
- On average, casual riders spend more time riding than member riders. 
- Casual riders complete the most rides on Saturdays. This is also the day with the longest average trip duration.
- Member riders complete the most rides during the week, while the number of casual riders increases on Fridays, Saturdays, and Sundays.
- The summer months of June, July, and August have more rides compelted by both casual riders and members when compared to other months.
- Streeter Dr & Grand Avenue is the most popular station for starting and ending rides. Over 3,000 more rides are started and completed at this station compared to the second most popular station. Additionally, the list of top 20 start stations and end stations were very similar with only slight differences in order when comaring the number of rides.


### 5. Business Reccomendations
When casual riders are renting a bike on Firday, Saturday or Sunday, additional information and/or advertisements about membership benefits should be presented during the initial rental sign up. Since more casual riders are renting bikes on these days, the advertisements are more likely to be seen by casual riders.

Additionally, advertisements and promotions should appear most in the summe rmonths, when casual riders are riding more frequently.

Based on the average ride time of 12 minutes by member riders, a promotional discount could be offered to casual riders who complete rides that are longer than 15 minutes. This encourages riders who are completing longer trips to convert to memberships.

Advertisements for memberships should be focused on the top 20 start stations, which include Streeter Dr & Grand Ave, DuSable Lake Shore Dr & Monroe St, Michigan Ave & Oak St, DuSable Lake Shore Dr & North Blvd, Wells St & Concord Ln stations. These stations are where advertisements would be seen by the most casual riders. QR codes could also be added to these stations to offer easy acces to additional membership information.


### 6. Considerations for further analysis
Additional data about user demographics and trip purposes could be useful to understand the motivation behind casual riders and members using the bike share service. These additional data points could assits with more targeted ads to specific age groups and provide deeper insights into user motivations and preferences. Moreover, additiona data included with the daily weather when bikes are rented could also provide insights into the number of rides completed that day and seasonal trends related to cooler and warmer months.

---

[Back to Portfolio](awodzicki.github.io)

