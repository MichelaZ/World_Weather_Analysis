# World Weather Analysis
## Purpose:
The client asked for a way for users to input their desired weather in max temperatures to identify potential travel destinations and nearby hotels.  For the beta testing I used Max Temps of 75-95 and created a map of the potential destinations. Then I chose 4 cities and created a travel route. Clicking on the pins for the destinations show the current weather at that location.
## Methods:
#### Deliverable 1:
1. I imported all my dependencies and API keys.
```
#add dependencies
import random
import numpy as np
import pandas as pd
#matplotlib
%matplotlib inline
import matplotlib.pyplot as plt

import timeit 
import time
from datetime import datetime
from scipy.stats import linregress
from citipy import citipy

# Import the requests library.
import requests
# Import the API key.
from config import weather_api_key

#gmaps dependencies
import gmaps
# Configure gmaps to use your Google API key.
from config import g_key
gmaps.configure(api_key=g_key)
```
2. I used the random function to create latitudes and longitudes. I then used them to find random cities nearby.
```
#Create a new set of 2,000 random latitudes and longitudes.
#create latitude
lats = np.random.uniform(low=-90.000, high=90.000, size=2000)
#create longitude
lngs = np.random.uniform(low=-180.000, high=180.000,size=2000)
lat_lngs = zip(lats, lngs)

# Add the latitudes and longitudes to a list.
coordinates = list(lat_lngs)

#list of cities
cities = []
#identify nearest city
for coordinate in coordinates:
    city = (citipy.nearest_city(coordinate[0], coordinate[1]).city_name)
    # If the city is unique, then we will add it to the cities list.
    if city not in cities:
        cities.append(city)
# Print the city count to confirm sufficient count.
len(cities)
```
3. Then I did an API call to get the weather data for thoses cities and added the data to a dataframe.
```
# Starting URL for Weather Map API Call.
url = "http://api.openweathermap.org/data/2.5/weather?units=Imperial&APPID=" + weather_api_key

# Create an empty list to hold the weather data.
city_data = []
# Print the beginning of the logging.
print("Beginning Data Retrieval     ")
print("-----------------------------")

# Create counters.
record_count = 1
set_count = 1
# Loop through all the cities in the list.
for i, city in enumerate(cities):

    # Group cities in sets of 50 for logging purposes.
    if (i % 50 == 0 and i >= 50):
        set_count += 1
        record_count = 1
        time.sleep(60)

    # Create endpoint URL with each city.
    city_url = url + "&q=" + city.replace(" ","+")

    # Log the URL, record, and set numbers and the city.
    print(f"Processing Record {record_count} of Set {set_count} | {city}")
    # Add 1 to the record count.
    record_count += 1
# Run an API request for each of the cities.
    try:
        # Parse the JSON and retrieve data.
        city_weather = requests.get(city_url).json()
        # Parse out the needed data.
        city_lat = city_weather["coord"]["lat"]
        city_lng = city_weather["coord"]["lon"]
        city_description = city_weather["weather"][0]["description"] 
        city_max_temp = city_weather["main"]["temp_max"]
        city_humidity = city_weather["main"]["humidity"]
        city_clouds = city_weather["clouds"]["all"]
        city_wind = city_weather["wind"]["speed"]
        city_country = city_weather["sys"]["country"]
      
        # Convert the date to ISO standard.
        city_date = datetime.utcfromtimestamp(city_weather["dt"]).strftime('%Y-%m-%d %H:%M:%S')
        # Append the city information into city_data list.
        city_data.append({"City": city.title(),
                          "Country": city_country,
                          "Lat": city_lat,
                          "Lng": city_lng,
                          "Max Temp": city_max_temp,
                          "Humidity": city_humidity,
                          "Cloudiness": city_clouds,
                          "Wind Speed": city_wind,
                          "Current Description": city_description, 
                          "Date": city_date})

# If an error is experienced, skip the city.
    except:
        print("City not found. Skipping...")
        pass
    
# Indicate that Data Loading is complete.
print("-----------------------------")
print("Data Retrieval Complete      ")
print("-----------------------------")

# Convert the array of dictionaries to a Pandas DataFrame.
city_data_df = pd.DataFrame(city_data)
```
4. Then I saved the dataframe to WeatherPy_Database.csv.
```
# Create the output file (CSV).
output_data_file = "Weather_Database/WeatherPy_Database.csv"
# Export the City_Data into a CSV.
city_data_df.to_csv(output_data_file, index_label="City_ID")
```
#### Deliverable 2: 

1.I imported all my dependencies and API keys.
```
# Dependencies and Setup
import pandas as pd
import numpy as np
import requests
import gmaps
#matplotlib
%matplotlib inline
import matplotlib.pyplot as plt

# Import API key
from config import g_key

# Configure gmaps API key
gmaps.configure(api_key=g_key)
```
2. I imported my data from the WeatherPy_database.csv file from deliverable 1.
```
city_data_df = pd.read_csv("Weather_Database\WeatherPy_database.csv")
city_data_df.head()
```
3. I created a prompt to ask the customer to input a max temperature minimum and maximum. I entered 75 for the min and 95 for the max to test the rest of the code. I then created a new dataframe to filter the data by the customer input.
```
min_temp = float(input("What is the minimum temperature you would like for your trip? "))
max_temp = float(input("What is the maximum temperature you would like for your trip? "))

preferred_cities_df = city_data_df.loc[(city_data_df["Max Temp"] <= max_temp) & \
                                       (city_data_df["Max Temp"] >= min_temp)]
```
5. I checked for and created a new dataframe where I dropped any null values.
```
print(len(preferred_cities_df))
preferred_cities_df.count()

clean_df = preferred_cities_df.dropna(axis="index", how="any")
```
6. I created a dataframe to store hotel names along with city, country, max temp, and coordinates. Then I added a hotel name column to the dataframe.
```
hotel_df = clean_df[["City", "Country", "Max Temp", "Current Description", "Lat", "Lng"]].copy()

hotel_df["Hotel Name"] = ""
```
7. Then I created a for loop to find hotels within 5000 meters of my cities and add them to my dataframe.
```
params = {
    "radius": 5000,
    "type": "lodging",
    "key": g_key}

for index, row in hotel_df.iterrows():
    lat = row["Lat"]
    lng = row["Lng"]
    params["location"] = f"{lat},{lng}"

    base_url = "https://maps.googleapis.com/maps/api/place/nearbysearch/json"   
    hotels = requests.get(base_url, params=params).json()
    
    try:
        hotel_df.loc[index, "Hotel Name"] = hotels["results"][0]["name"]
    except (IndexError):
        print("Hotel not found... skipping.")
```
8. I checked for nulls and created a new dataframe where the nulls were dropped. Then I saved the data frame to the WeatherPyVacation.csv; this file is also saved as city_data.csv, because I found it easier to keep it separate from the other output file.
```
print(len(hotel_df))
hotel_df.count()

clean_hotel_df = hotel_df.dropna(axis="index", how="any")
print(len(clean_hotel_df))
clean_hotel_df.count()

output_data_file = "Weather_Database/WeatherPyVacation.csv"
clean_hotel_df.to_csv(output_data_file, index_label="City_ID")
```
9. I set up the template for the infobox and then I added a map with a marker layer and infoboxes for my cities.
```
info_box_template = """
<dl>
<dt>Hotel Name</dt><dd>{Hotel Name}</dd>
<dt>City</dt><dd>{City}</dd>
<dt>Country</dt><dd>{Country}</dd>
<dt>Current Weather</dt><dd>{Current Description} and <dd>{Max Temp} °F</dd>
</dl>
"""

hotel_info = [info_box_template.format(**row) for index, row in clean_hotel_df.iterrows()]

locations = clean_hotel_df[["Lat", "Lng"]]
fig = gmaps.figure(center=(30.0, 31.0), zoom_level=1.5)
marker_layer = gmaps.marker_layer(locations, info_box_content=hotel_info)
fig.add_layer(marker_layer)
fig
```
![Destinations](https://github.com/MichelaZ/World_Weather_Analysis/blob/main/Vacation_Search/Vacation_Itinerary/desinations.png)

#### Deliverable 3:
1. I imported my dependencies, API keys and imported the data.
```
import pandas as pd
import requests
import gmaps
import numpy as np
from config import g_key
%matplotlib inline
import matplotlib.pyplot as plt
gmaps.configure(api_key=g_key)

import csv
import os
data_to_load = os.path.join("..","Weather_Database", "City_Data.csv")
vacation_df = pd.read_csv(data_to_load)
```
2. I created the infobox template again and created the map of all the potential destinations again tpo check that my code was still working.
```
info_box_template = """
<dl>
<dt>Hotel Name</dt><dd>{Hotel Name}</dd>
<dt>City</dt><dd>{City}</dd>
<dt>Country</dt><dd>{Country}</dd>
<dt>Current Weather</dt><dd>{Current Description} and <dd>{Max Temp} °F</dd>
</dl>

hotel_info = [info_box_template.format(**row) for index, row in vacation_df.iterrows()]
locations = vacation_df[["Lat", "Lng"]]

fig = gmaps.figure(center=(30.0, 31.0), zoom_level=1.5)
marker_layer = gmaps.marker_layer(locations, info_box_content=hotel_info)
fig.add_layer(marker_layer)
fig
"""
```
3. Then I picked four cities and created a route between them. I set each city to a variable. Then I used the  loc function to created dataframes for each city.
```
start_city = "Nhulunbuy"
end_city = "Nhulunbuy"
city_1 = "Port Hedland"
city_2 = "Roebourne"
city_3 = "Karratha"

vacation_start = vacation_df.loc[vacation_df["City"] == start_city]
vacation_end = vacation_df.loc[vacation_df["City"] == end_city]
vacation_stop1 = vacation_df.loc[vacation_df["City"] == city_1]
vacation_stop2 = vacation_df.loc[vacation_df["City"] == city_2] 
vacation_stop3 = vacation_df.loc[vacation_df["City"] == city_3] 
```
4. Then I grabed the latitude and longidure pairs for each city as a tuple, so they could be used as values for the waypoints on my maps.
```
start = vacation_start[["Lat", "Lng"]].to_numpy()[0]
end = vacation_end[["Lat", "Lng"]].to_numpy()[0]
stop1 = vacation_stop1[["Lat", "Lng"]].to_numpy()[0]
stop2 = vacation_stop2[["Lat", "Lng"]].to_numpy()[0] 
stop3 = vacation_stop3[["Lat", "Lng"]].to_numpy()[0]
```
5. Then I made a map with a directions layer to show the route between the 4 cities.
```
fig = gmaps.figure(center=(30.0, 31.0), zoom_level=1.5)
route = gmaps.directions_layer(start, end, waypoints=[stop1,stop2,stop3], travel_mode="DRIVING" or "BICYCLING", show_markers=True,
    stroke_color='black', stroke_weight=4.0, stroke_opacity=.2)
fig.add_layer (route)
fig
```
![Directions](https://github.com/MichelaZ/World_Weather_Analysis/blob/main/Vacation_Search/Vacation_Itinerary/WeatherPy_travel_map.png)

6. I used the concat function to combine the 4 city dataframes for my layer map to add makers and infoboxes.
```
itinerary_df = pd.concat([vacation_start, vacation_stop1, vacation_stop2,vacation_stop3 ],ignore_index=True)
itinerary_df

info_box_template = """
<dl>
<dt>Hotel Name</dt><dd>{Hotel Name}</dd>
<dt>City</dt><dd>{City}</dd>
<dt>Country</dt><dd>{Country}</dd>
<dt>Current Weather</dt><dd>{Current Description} and <dd>{Max Temp} °F</dd>
</dl>
"""
hotel_info = [info_box_template.format(**row) for index, row in itinerary_df.iterrows()]

locations = itinerary_df[["Lat", "Lng"]]

fig = gmaps.figure()
markers = gmaps.marker_layer(locations, info_box_content=hotel_info)
fig.add_layer(markers)
fig 
```
![Itinerary map with markers](https://github.com/MichelaZ/World_Weather_Analysis/blob/main/Vacation_Search/Vacation_Itinerary/WeatherPy_travel_map_markers.png)
7. Then for fun I combined the two maps. I had to turn off the markers for my infobox to work and you can see I played around with the color and opacity of the directions line, but the map works.
```
fig = gmaps.figure(center=(30.0, 31.0), zoom_level=1.5)
markers = gmaps.marker_layer(locations, info_box_content=hotel_info)
route = gmaps.directions_layer(start, end, waypoints=[stop1,stop2,stop3], travel_mode="DRIVING" or "BICYCLING",
   show_markers=False)
#stroke_opacity=.2, stroke_weight=4.0,stroke_color='black'
fig.add_layer(markers)
fig.add_layer (route)
fig 
```
![Map with destination pins, infobox and directions](https://github.com/MichelaZ/World_Weather_Analysis/blob/main/Vacation_Search/Vacation_Itinerary/direct_pin_infobox.png)

## Conclusions:
I maybe should have picked different cities. Some of my cities were very close and one was pretty far away so I can't really show all of my info boxes in an image and they are difficult to click on. I do think the premise of the assignment is a little flawed: People tend to plan trip itineraries way in advance. If I'm going on vacation I try to plan it for time when I expect weather to be generally pleasent based on the past or on the forecast, not on current temperatures.  I also think I care more about the percipitation than the temperature, because depending on where you are going that can fluctuate a lot throughout the day. I understand that weather APIs are expensive so I'm assuming that is part of out limiting factor. If I were desiging this for a real client it would be nice for a feature that allows customers to pick the dates of their travel and have the app provide the forcast for those days.

