# Google Data Analytics Professional Certificate Case Study: Bellabeat

I've recently completed the Google Data Analytics Professional Certificate program, which culminated in a series of case study "capstone" projects designed to help build a professional portfolio. For this case study I’ve chosen their Bellabeat project, which includes analyzing data from a similar company to help the stakeholders make decisions regarding their own products and services. Using R for some light formatting, SQL for data analysis and Tableau for visualization, I discovered some interesting patterns and relationships that I think would be helpful for Bellabeat, and presented my findings as a Tableau story.

## Introduction
**Client:** Bellabeat

Bellabeat is a tech company that creates smart products and apps designed for women to track, manage, and improve their health. Bellabeat focuses not just on exercise and diet, but mindfulness, sleep, and menstruation. Cofounder and CFO Urška Sršen has asked me to analyze data from a similar company, Fitbit, and see how my insights could be used to help Bellabeat grow and succeed.

**Report to:** Urška Sršen, co founder and CFO

## Ask
How can having a better understanding of how people use similar products help to reveal potential areas of growth and opportunity for Bellabeat?

## Prepare
I was given access to a number of datasets collected by FitBit from volunteers between March 12th, 2016 and May 12th, 2016. The data was divided into two folders representing the two months, so most tables from one folder had a counterpart in the other, but not all. The original datasets can be found [here](https://www.kaggle.com/datasets/arashnic/fitbit?source=post_page-----c18835475563--------------------------------) on Kaggle.

## Process
Right off the bat I had issues with the .csv files I was provided with. I prefer to work in SQL, often using BigQuery as my tool, but many of the files would not load due to a time column that included an AM or PM, which BigQuery rejected. Additionally, many of those same files were too large to open in Excel, so in order to fix the issue I chose to use R to load the files, and then split the time columns into 2 new columns; 1 for the date, and another for the time, which I converted to 24-hour time, omitting the AM/PM altogether. Then I removed the original time column that was causing the issue. From there I was able to successfully review the data in SQL.

Additionally, column names were inconsistent across the datasets and there were some odd choices in file naming conventions. So I had some work to do in cleaning the data.

Upon verification I discovered that although the datasets were said to have information provided by 30 participants, that number was in fact different from set to set, ranging from as many as 34 participants in some sets to as few as 8 in others. Any datasets with fewer than 30 participants I chose not to use as the data was just too insufficient. I then narrowed my focus to datasets that matched between the two months recorded, any dataset that didn’t have a counterpart was not used for my analysis.

I then joined the datasets from the dates 03/12/16 - 04/11/16 to the corresponding sets from the dates 04/12/26 - 05/12/16.

### R Script

```R
# Loading dplyr and lubridate libaries
`library(dplyr)`
`library(lubridate)`

# Read the .csv file, calling it "data"
data <- read.csv("~/Desktop/Data_stuff/Case_Study_2/mturkfitbit_export_4.12.16-5.12.16/Fitabase Data 4.12.16-5.12.16/X-heartrate_seconds_merged.csv", stringsAsFactors = FALSE)

# Checking the structure of the data
str(data)

# Printing the column names to check spelling, capitalization, spaces, etc.
colnames(data)

# Spliting the "Time" column into 2 new columns, "Date" and "Time_24"
data <- data %>%
  mutate(Date = as.Date(ActivityMinute, format = "%m/%d/%y"),
         Time_24 = format(strptime(sub(".* (\\d{1,2}:\\d{2}:\\d{2} [APMapm]{2})", "\\1", ActivityMinute), "%I:%M:%S %p"), "%H:%M:%S"))

# Delete the original "Time" column that was causing the error in BigQuery
data <- data %>%
  select(-ActivityMinute)

# Save the new table as .csv and it should upload to BigQuery correctly
write.csv(data, "~/Desktop/Data_stuff/Case_Study_2/mturkfitbit_export_4.12.16-5.12.16/Fitabase Data 4.12.16-5.12.16/X-minuteStepsNarrow_merged.csv", row.names = FALSE)

# I believe that the remaining files giving me problems are all experiencing the same issue, so I'll repeat these steps for the rest of the .csv files and then resume looking at the data in BigQuery
```

### Key SQL Queries

```sql
/*I first looked at each dataset, and determined which tables and columns were useful to my purpose and which were not. I wanted to get at the real essence of the data and found that some of what was provided was not useful to me. Some columns and entire datasets were largely blank, so I left them out. There were also lots of inconsistencies in column names across the datasets so I applied aliases to them for consistency. Below is how I first chose to work with the Daily Activity table, but I made some more changes later on, which I’ll present later.*/

SELECT
 Id AS id,
 ActivityDate AS act_date,
 TotalSteps AS total_steps,
 ROUND(TotalDistance, 2) AS total_dist,
 ROUND(TrackerDistance, 2) AS tracker_dist,
 ROUND(VeryActiveDistance, 2) AS very_act_dist,
 ROUND(ModeratelyActiveDistance, 2) AS moderate_act_dist,
 ROUND(LightActiveDistance ,2) AS light_act_dist,
 ROUND(SedentaryActiveDistance, 2) AS sedentary_act_dist,
 VeryActiveMinutes AS very_act_min,
 FairlyActiveMinutes AS moderate_act_min,
 LightlyActiveMinutes AS light_act_min,
 SedentaryMinutes AS sedentary_act_min,
 Calories AS calories
FROM
 winged-yeti-423618-j9.case_study_2_031216_041116.daily_activity_merged
WHERE
 TotalSteps IS NOT NULL
 AND TotalDistance IS NOT NULL
 AND TrackerDistance IS NOT NULL
 AND VeryActiveDistance IS NOT NULL
 AND ModeratelyActiveDistance IS NOT NULL
 AND LightActiveDistance IS NOT NULL
 AND SedentaryActiveDistance IS NOT NULL
 AND VeryActiveMinutes IS NOT NULL
 AND FairlyActiveMinutes IS NOT NULL
 AND LightlyActiveMinutes IS NOT NULL
 AND SedentaryMinutes IS NOT NULL
 AND Calories IS NOT NULL
 ;
```
```sql
/*Here I'm applying aliases for consistency across all datasets. I’ve used “SELECT DISTINCT” to remove any potential duplicates. I'm breaking the ActivityHour column into a date and time column and adjusting the format for my needs. Next, I’m grouping and ordering accordingly and removing any potential blanks/nulls. Once I’m happy with the results I'll perform the same query for each table*/

SELECT DISTINCT
 Id AS id,
   DATE_ADD(DATE(ActivityHour), INTERVAL 2000 YEAR) AS date,
   (FORMAT_TIME('%H:%M', TIME(ActivityHour))) AS time,
   SUM(Calories) AS calories
FROM
 winged-yeti-423618-j9.case_study_2_031216_041116.hourly_calories_merged
WHERE
 Id IS NOT NULL
 AND ActivityHour IS NOT NULL
 AND Calories IS NOT NULL
GROUP BY id, date, time
ORDER BY id, date, time
;
```
```sql
/*After spending some time analyzing the Daily Activity tables, I decided to make a few changes. I wanted to calculate averages and sums, remove any entries that were listed as 0, and determine if there was a significant difference between the data that the FitBit tracker reported (tracker_distance) and the data entered in by other means (total_distance). There wasn’t but I needed to know!*/

SELECT
 id,
 act_date,
 EXTRACT(DAYOFWEEK FROM act_date) day_of_week,
 AVG(calories) avg_calories,
 SUM(total_steps) total_steps,
 SUM(total_dist) total_distance,
 SUM(tracker_dist) tracker_distance,
 AVG(ROUND((total_dist - tracker_dist),2)) distance_diff,
 SUM(sedentary_act_dist) total_sedentary_distance,
 SUM(light_act_dist) total_light_distance,
 SUM(moderate_act_dist) total_moderate_distance,
 SUM(very_act_dist) total_very_active_distance,
 AVG(sedentary_act_min) avg_sedentary_min,
 AVG(light_act_min) avg_light_min,
 AVG(moderate_act_min) avg_moderate_min,
 AVG(very_act_min) avg_very_active_min
FROM
 winged-yeti-423618-j9.case_study_2_merged.daily_activity
WHERE
 total_steps <> 0
 AND total_dist <> 0
 AND tracker_dist <> 0
 AND very_act_dist <> 0
 AND moderate_act_dist <> 0
 AND light_act_dist <> 0
 AND sedentary_act_dist <> 0
 AND very_act_min <> 0
 AND moderate_act_min <> 0
 AND light_act_min <> 0
 AND sedentary_act_min <> 0
 AND calories <> 0
GROUP BY id, act_date
ORDER BY act_date, id
;
```
```sql
/*Now that I’m happy with each table from the two months, I chose to combine them with their counterparts from the other month so that I could see it all at once.*/

SELECT *
FROM
 winged-yeti-423618-j9.case_study_2_031216_041116.daily_activity_clean AS daily_activity_031216_041116
UNION DISTINCT
 SELECT *
 FROM
   winged-yeti-423618-j9.case_study_2_041216_051216.daily_activity_clean
ORDER BY id, act_date   
;

/*My next phase was to import these data sources into Tableau and see what kinds of relationships were revealed!*/
```
## Analyze
Upon analyzing the data I discovered some interesting trends and patterns:

* Users tend to get most of their exercise in the late morning, afternoon, and early evening. Early morning and after 7 PM, they engage in much less exercise.
* Even on days with heavier exercise, most of the day is spent in a sedentary state. While much of this time is spent sleeping, a significant portion could be attributed to working at a desk, watching TV, or other sedentary activities.
* Fridays and Mondays see the most exercise, with a slight dip during the weekend and a much sharper decline mid-week. This indicates that users are getting most of their exercise at the start and end of the week, with much less during the mid-week. This could be due to burnout, lifestyle, or work-related demands.


## Share
I have prepared a Tableau story featuring a series of dashboards that illustrate my findings, including brief annotations explaining the charts in more detail.

### [View my Tableau story here, on Tableau Public.](https://public.tableau.com/views/GoogleDataAnalyticsCaseStudy-Bellabeat/Presentation?:language=en-US&:sid=&:display_count=n&:origin=viz_share_link)

## Act
The final slide contains a few suggestions for Sršen and her colleagues at Bellabeat to consider:

* Our app and smart devices could remind users to stand up, walk around, and get moving, even if only lightly. The app could suggest simple, office-friendly exercises to help users stay active.
* While some sedentary activity is necessary, our products could help users balance sleep, stillness, and exercise in a healthier manner. For instance, sedentary time could be used for sleep, meditation, or reading, rather than sitting at a desk or watching TV. We can help our users use their downtime to stay healthy.
* Our app and smart devices could help users spread their exercise time more evenly throughout the week while still accounting for "rest days." Since everyone's lives and work schedules are different, this should be customizable, while still encouraging users to get some light exercise mid-week.

## Afterthoughts
Although I do believe that this information would be helpful to Bellabeat, I found the data lacking in some key areas. For instance, Bellabeat is a company that makes products for women, and the FitBit data that I was working with provided no demographic information at all, including gender identity. More demographic data would have helped paint a more clear picture of the clients. Gender identity, age, and user provided thoughts on body type, fitness goals and exercise routine come to mind. I also think that I could've done a more well-rounded analysis with proper data on sleep, meditation, diet, etc. Still, dispite these limitations I am glad to have uncovered some interesting insights.
