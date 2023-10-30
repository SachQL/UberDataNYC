# Uber Trip Data Analysis

# Table of Contents:
- [Project Overview](#project-overview)
- [Data Cleaning/Preparation](#data-cleaningpreparation)
- [Data Analysis](#data-analysis)
- [Results/Findings](#resultsfindings)
- [Recommendations](#recommendations)
- [Limitations](#limitations)

## Project Overview
A deep dive into the Uber trip data to uncover patterns in trip durations, fares, and locations. The goal is to optimize Uber's operations, anticipate user demand, and improve the overall rider experience in New York City.

## Data Cleaning/Preparation

For This project, i used MySQL to run a local database on my machine and a SQL editior called PopSQL.
Before diving into the analysis, it was crucial to ensure the data was clean and suitable for exploration.

Some steps taken:

1. Create a suitable table for my data in the database.
  ```sql
  CREATE TABLE UberTrips (
    TripID INT AUTO_INCREMENT PRIMARY KEY,
    TripDate VARCHAR(25), 
    PickupLatitude DECIMAL(10, 8),
    PickupLongitude DECIMAL(11, 8),
    DropOffLatitude DECIMAL(10, 8),
    DropOffLongitude DECIMAL(11, 8),
    FarePaid DECIMAL(8, 2),
    NumberOfPassengers INT
  );
  ```
2. Import data using SQLqrokbench and check all the data has been added correctly.
  - There should be 20,001 entries if all entries were imported correctly.
    
    
      ```sql
      SELECT COUNT(*) FROM ubertrips
      ;
      ```
     - We can see some errors where it looks like the longitude and lattitude were not recored properly, having a value of `0.000000` (380 entries)       
        ```sql
        SELECT COUNT(*) 
        FROM ubertrips 
        WHERE PickupLatitude LIKE '0%'
          OR PickupLongitude LIKE '0%'
          OR DropOffLatitude LIKE '0%'
          OR DropOffLongitude LIKE '0%'
        ;
        ```
      - For simplicity we will simpy drop these records.
        ```sql
        DELETE  
        FROM ubertrips
        WHERE PickupLatitude LIKE '0%'
          OR PickupLongitude LIKE '0%'
          OR DropOffLatitude LIKE '0%'
          OR DropOffLongitude LIKE '0%'
          ;
        ```
3. Data enchancement. We can use googles distance matirx API to get extra information about our trips. using the pick up and drop off longitiude and latitiude, we can obtiain trip distances and durations for each trip in our database.
    - To help mitigate any mistakes i will create a seperate table so we dont ruin our original dataset.
    - I have decided to use `VARCHAR` since the duration and distance have units associated with them such as 'min' or 'km'
    
    ```sql
    CREATE TABLE ubertrips_distances (
        TripID INT PRIMARY KEY,
        Distance VARCHAR(55),
        Duration VARCHAR(55)
    );
    ```

4. I would like to have a table with all the data present so i will simply join the tables.
```sql
CREATE TABLE combined_data AS
SELECT 
    a.TripID,
    a.TripDate,
    a.PickupLatitude,
    a.PickupLongitude,
    a.DropoffLatitude,
    a.DropoffLongitude,
    a.FarePaid,
    a.NumberOfPassengers,
    b.Distance,
    b.Duration
FROM 
    ubertrips a
INNER JOIN 
    ubertrips_distances b
ON 
    a.TripID = b.TripID;
```

5. Our `TripDate` is currently stored as a string, so we will need to modify this column so it is in the correct format.
    - Using the STR_TO_DATE fuction we can extract the date from our `TripDate`
      ```sql
      UPDATE combined_data 
      SET TripDate = STR_TO_DATE(TripDate, '%Y-%m-%d %H:%i:%s UTC');
      ```
    - We can now update our `TripDate` so that it is in the correct format.
      ```sql
      ALTER TABLE combined_data
      MODIFY TripDate DATETIME;
      ```
6. likewise, we need to remove the units from our distance column and convert this into a Distance data type.
    - Update distances in meters to kilometers
      ```sql
      UPDATE combined_data
      SET Distance = CAST(REPLACE(Distance, ' m', '') AS DECIMAL(10,2)) / 1000
      WHERE Distance LIKE '% m';
      ```
    - Update to remove ' km' from the Distance column
      ```sql
      UPDATE combined_data
      SET Distance = REPLACE(Distance, ' km', '');
      ```
    - Update distance column to decimal.
      ```sql
      ALTER TABLE combined_data
      MODIFY Distance DECIMAL(10,2);
      ```
7. We can also see a few instaces where the distance travelled was less than 0.1 km (100 m) but the trips have faires as high as 180 USD.

```sql
SELECT *
FROM combined_data
where Distance < 0.01;
```
- It is unlikely that someone would hire an uber for a trip that would take around 1 minutes to walk. 
- The fares for these rides range from reasonable prices ~5 USD to very high prices $180 USD.
- This suggests that there is some error in the recording of longitude and latitude data.
- We will remove those records from our analysis
```sql
SELECT COUNT(*)
FROM combined_data
where Distance < 0.01;

-- 239 records found.

DELETE FROM combined_data
WHERE Distance < 0.01;

--239 records deleted
```
## Data Analysis

**Analysing Peak Hours**: what times a have the highest volume of rides?
    - peak hours are in the evening 5pm-11pm with the quietest hours being 4am-5am.
```sql
  SELECT 
    HOUR(TripDate) AS HourOfDay,
    COUNT(TripID) AS NumberOfRides
FROM 
    combined_data
GROUP BY 
    HOUR(TripDate)
ORDER BY 
    NumberOfRides DESC;
```
**Weekday vs Weekends Peak Hour Analysis**: Do we see differences in the peak hours on weekends comapred to weekdays?

- To understand this we will look at the percentage of rides by hour for both weekends and weekdays.
- We can see that there is a higher percentage of rides on weekdays between 6am and 10am, likely corresponding to people comuting to work.
- We can also see a slight difference in weekdays between 5pm and 8pm, probably for a simular reason.
- On weekends we see a higher percentage of rides in the early morning between 12am and 4am.
      
```sql
-- We will first define a common table expressions (CTE) to count the total number of rides for weekends and weekdays.

WITH DayTypeTotals AS (
    SELECT 
        CASE 
            WHEN DAYOFWEEK(TripDate) BETWEEN 2 AND 6 THEN 'Weekday'
            ELSE 'Weekend'
        END AS day_type,
        COUNT(*) AS total_rides
    FROM 
        combined_data
    GROUP BY 
        day_type
)

-- We need to also create a subquery to count the number of rides by hour for both weekdays and weekends.

SELECT 
    RidesByHour.day_type,
    RidesByHour.hour,
    RidesByHour.number_of_rides,
    (RidesByHour.number_of_rides * 100.0 / DayTypeTotals.total_rides) AS percentage_of_rides -- Calculate the percentage of rides by hour.
FROM 
    (SELECT 
        CASE 
            WHEN DAYOFWEEK(TripDate) BETWEEN 2 AND 6 THEN 'Weekday'
            ELSE 'Weekend'
        END AS day_type,
        HOUR(TripDate) AS hour,
        COUNT(*) AS number_of_rides
    FROM 
        combined_data 
    GROUP BY 
        day_type, hour) AS RidesByHour

JOIN 
    DayTypeTotals
ON 
    RidesByHour.day_type = DayTypeTotals.day_type

ORDER BY 
    RidesByHour.day_type, RidesByHour.hour;
```
**Airport Peak Hour Analysis**: What are the busiest days for airport pickups and dropsoffs?
- JKF airport longitude and latitude coordinates = 40.644537, -73.783260
- Note that 0.01 in longitude and lattitude = 0.69 miles
- Sundays had the highest number of airport rides (72)
- Wednesdays has the least (48)
```sql
SELECT 
    DAYOFWEEK(TripDate) AS day_of_week,
    CASE 
        WHEN DAYOFWEEK(TripDate) = 1 THEN 'Sunday'
        WHEN DAYOFWEEK(TripDate) = 2 THEN 'Monday'
        WHEN DAYOFWEEK(TripDate) = 3 THEN 'Tuesday'
        WHEN DAYOFWEEK(TripDate) = 4 THEN 'Wednesday'
        WHEN DAYOFWEEK(TripDate) = 5 THEN 'Thursday'
        WHEN DAYOFWEEK(TripDate) = 6 THEN 'Friday'
        ELSE 'Saturday'
    END AS weekday_name,
    COUNT(*) AS airport_rides_count
FROM 
    combined_data
WHERE 
    (ABS(PickupLatitude - 40.644537) < 0.01 AND ABS(PickupLongitude - -73.783260) < 0.01)
    OR
    (ABS(DropoffLatitude - 40.644537) < 0.01 AND ABS(DropoffLongitude - -73.783260) < 0.01)
GROUP BY 
    day_of_week, weekday_name
ORDER BY 
    day_of_week;
```
**Passenger Count Analysis**: Do rides with more passengers have higher fares?
- Average fares appear to be independant of passenger count.
```sql
SELECT NumberOfPassengers, AVG(FarePaid) AS AvgFare
FROM combined_data
GROUP BY NumberOfPassengers;
```

## Results/Findings: 

**Airport Peak Hour Analysis**: What are the busiest days for airport pickups and dropsoffs?
- Sundays had the highest number of airport rides (72)
- Wednesdays has the least (48)

**Analysing Peak Hours**: what times a have the highest and lowest volume of rides?
- peak hours are in the evening 5pm-11pm with the quietest hours being 4am-5am.

**Weekday vs Weekends Peak Hour Analysis**: Do we see differences in the peak hours on weekends comapred to weekdays?
- We can see that there is a higher percentage of rides on weekdays between 6am and 10am, likely corresponding to people comuting to work.
- We can also see a slight difference in weekdays between 5pm and 8pm, probably for a simular reason.
- On weekends we see a higher percentage of rides in the early morning between 12am and 4am.

**Passenger Count Analysis**: Do rides with more passengers have higher fares?
- Average fares appear to be independant of passenger count.

## Recommendations:

## Limitations:

## Other

Google distance matrix API code.
```python
import requests
import pymysql
import time

# Establish a connection to your MySQL database
connection = pymysql.connect(host='THE_HOST',
                             user='YOUR_USER',
                             password='YOUR_PASSWORD',
                             db='THE_DATABASE')

# Google Maps API endpoint
endpoint = "https://maps.googleapis.com/maps/api/distancematrix/json?"

# Your Google Maps API Key
api_key = "Your_API_KEY_HERE"

with connection.cursor() as cursor:
    # Fetch entries between start_id and end_id from the database
    cursor.execute("SELECT TripID, PickupLongitude, PickupLatitude, DropoffLongitude, DropoffLatitude FROM ubertrips)
    trips = cursor.fetchall()

    for trip in trips:
        trip_id = trip[0]
        origin = f"{trip[2]},{trip[1]}" 
        destination = f"{trip[4]},{trip[3]}" 
        
        nav_request = f"origins={origin}&destinations={destination}&key={api_key}"
        request = endpoint + nav_request

        result = requests.get(request).json()

        if 'rows' in result and len(result['rows']) > 0 and 'elements' in result['rows'][0] and len(result['rows'][0]['elements']) > 0:
            element = result['rows'][0]['elements'][0]
            if element['status'] == "OK":
                distance = element['distance']['text']
                duration = element['duration']['text']
                insert_query = """INSERT INTO ubertrips_distances (TripID, Distance, Duration) 
                                  VALUES (%s, %s, %s)"""
                cursor.execute(insert_query, (trip_id, distance, duration))
            else:
                print(f"Error for TripID: {trip_id}. Status: {element['status']}")
        else:
            print(f"Unexpected response format for TripID: {trip_id}. Response: {result}")

        # Introduce a delay to avoid hitting the rate limit, a 0.1-second delay:
        time.sleep(0.1)

    # Commit the changes
    connection.commit()

connection.close()
```


