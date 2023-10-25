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
7. We can also see a few instaces where the distance travelled was less than 0.1km (100m) but the trips have faires as high as 180 USD. This doesnt seem likely so we will delete these records.
```sql
SELECT *
FROM combined_data
where Distance < 0.01;
```
- Here we can see many entries that dont logically make sense.
1. It is unlikely that someone would hire an uber for a trip that would take around 1 minutes to walk. 
2. The fares for these rides range from reasonable prices ~5 USD to very high prices $180 USD.
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

- **Analyzing Trip Duration**:
  ```sql
  -- anaylsis code here.

## Results/Findings: 

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


