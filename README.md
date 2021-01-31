# World_Weather_Analysis

## Purpose

The purpose of the project was to use API resources to assist in planning vacations based on desired weather conditions.

## Resources

- Data Resources: https://openweathermap.org/, https://console.cloud.google.com
- Software: Python 3.9.1, Visual Studio Code 1.51.1, jupyter-notebook 6.1.4, gmaps, Pandas, numpy, matplotlib, citipy, scipy

## Results

### Pulling Weather Data

In order to create a random dataset to work with, we created a random pair of latitudes and longitudes using the `np.random.uniform` module. This module would create random float numbers within a defined range. The amount of random numbers was also defined and in this case set equal to 1500. Next to pair the coordinates together, the lists were zipped together.

```
lats = np.random.uniform(low=-90.000, high=90.000, size=1500)
lngs = np.random.uniform(low=-180.000, high=180.000, size=1500)
lat_lngs = zip(lats, lngs)
```

Next, the `cititpy` module was used within a `for` loop to go through the coordinates list and determine the closest city and add it to an empty list.

```
cities = []
for coordinate in coordinates:
    city = citipy.nearest_city(coordinate[0], coordinate[1]).city_name

    if city not in cities:
        cities.append(city)
```

The next step was to pull the weather data from the random list of cities that had just been created. The paramaters necessary for an API request were set previous to adding them to another `for` loop that would iterate through the list of cities and pull the necessary data from the `.json` returned from the API request. A new list was also created to hold the data being pulled.

```
city_data = []
print("Beginning Data Retrieval     ")
print("-----------------------------")

record_count = 1
set_count = 1

for i, city in enumerate(cities):
    if (i % 50 == 0 and i >= 50):
        set_count +=1
        record_count = 1

    city_url = url + "&q=" + city.replace(" ","+")
    
    print(f"Processing Record {record_count} of Set {set_count} | {city}")
    record_count += 1

    try:
        city_weather = requests.get(city_url).json()

        city_lat = city_weather["coord"]["lat"]
        city_lng = city_weather["coord"]["lon"]
        city_max_temp = city_weather["main"]["temp_max"]
        city_humidity = city_weather["main"]["humidity"]
        city_clouds = city_weather["clouds"]["all"]
        city_wind = city_weather["wind"]["speed"]
        city_country = city_weather["sys"]["country"]
        city_date = datetime.utcfromtimestamp(city_weather["dt"]).strftime('%Y-%m-%d %H:%M:%S')

        city_data.append({"City": city.title(),
                          "Lat": city_lat,
                          "Lng": city_lng,
                          "Max Temp": city_max_temp,
                          "Humidity": city_humidity,
                          "Cloudiness": city_clouds,
                          "Wind Speed": city_wind,
                          "Country": city_country,
                          "Date": city_date})

    except:
        print("City not found. Skipping...")
        pass

print("-----------------------------")
print("Data Retrieval Complete      ")
print("-----------------------------")
```

With such a large dataset, creating records and sets helped viualize the processing data as it was being pulled. Using the `enumerate()` function printed out the city names as na string instead of the index associated with the city name. `city_url` was the api request which contained the base url + the API key, the parramaters seperator, and then city with + replacing spaces to ensure the request pulled the correct cities information. Parsing the data and adding it to the `city_data` list was done in a `try` loop to prevent the data already collected from being dumped when encountering an error with any of the cities, however a print function was added to create an alert when an error occured and the data would not be entered into the list. Finally, when the data pull was completed a print function was used to create an alert.

### Analysis of Weather Data via Line Regression

The data in the list was then converted into a DataFrame using pandas so it could be analyzed further. From the dataframe different fields were extracted for further analysis to be added to scatter plots for regression analysis. After importing the lineregress module from `scipy.stats`, I created a function to plot a line regression on a scatterplot with the correct inputs.

```
def plot_linear_regression(x_values, y_values, title, y_label, text_coordinates):

    (slope, intercept, r_value, p_value, std_err) = linregress(x_values, y_values)
    regress_values = x_values * slope + intercept
    line_eq = "y = " + str(round(slope,2)) + "x + " + str(round(intercept,2))

    plt.scatter(x_values,y_values)
    plt.plot(x_values,regress_values,"r")
    # Annotate the text for the line equation.
    plt.annotate(line_eq, text_coordinates, fontsize=15, color="red")
    plt.xlabel('Latitude')
    plt.ylabel(y_label)
    plt.show()
```

This new function would plot and derive the regression line with just five input variables which could be pulled from the dataframes that had been split into north and south hemispheres. An example of the function results can be seen below:

```
x_values = northern_hemi_df["Lat"]
y_values = northern_hemi_df["Max Temp"]

plot_linear_regression(x_values, y_values, 'Linear Regression on the Northern Hemisphere \n for Maximum Temperature', 'Max Temp',(10,40))
```

![](/WeatherPy_Regression_Analysis.png)

### Pulling Hotel Data based on User Input

With the weather data dataframe, I could now pull neaby hotel data from google. First, to reduce the amount data needed to be pulled, we collected the ideal vacation temperatures from the user using the code below:

```
min_temp = float(input("What is the minimum temperature you would like for your trip?"))
max_temp = float(input("What is the maximum temperature you would like for your trip?"))
```

Once this information was determined, the `.loc` module was used to create a new DataFrame within the ideal temperatures for the user. An empty series was then created within that DataFrame to hold the information that was going to be pulled in the API request. A `for` loop was used again to iterate through the rows of the DataFrame and pull the necessary information from the google API.

```
for index, row in hotel_df.iterrows():
    lat = row["Lat"]
    lng = row["Lng"]
    base_url = "https://maps.googleapis.com/maps/api/place/nearbysearch/json"
    params["location"] = f"{lat},{lng}"
    hotels = requests.get(base_url, params=params).json()
    try:
        hotel_df.loc[index, "Hotel Name"] = hotels['results'][0]['name']
    except(IndexError):
        print("Hotel not found... skipping.")
print("Finished")
```

Based on the json the google API returned, the first hotel name was parsed and added to the DataFrame. This new information could then be added to a `gmaps` figure and marker level to create a pop up map with the hotel names and information. In order to do this another dataframe needed to be created to hold the data to be displayed. The loctions of the markers also needed to be defined.

```
info_box_template = """
<dt><b>Hotel Name</b></dt><dd>{Hotel Name}</dd>
<dt><b>City</b></dt><dd>{City}</dd>
<dt><b>Country</b></dt><dd>{Country}</dd>
<dt><b>Current Weather</b></dt><dd>{Current Description} and {Max Temp} °F</dd>
"""
hotel_info = [info_box_template.format(**row) for index, row in clean_hotel_df.iterrows()]
locations = clean_hotel_df[["Lat", "Lng"]]
```

Once the data has been formatted it can be used to created a marker map layer and created into a figure:

```
fig = gmaps.figure(center=(30.0, 31.0), zoom_level=1.5)
marker_layer = gmaps.marker_layer(locations, info_box_content=hotel_info)
fig.add_layer(marker_layer)
fig
```

![](/Vacation_Search/WeatherPy_vacation_map.png)

### Creating a Vacation Itinerary

After analyzing the cities available and the weather associated, 4 were selected to plan a round trip. Using the gmaps module gmaps I was able to plot a course between the cities.

![](/Vacation_Itinerary/WeatherPy_travel_map.png)

Afterwards, through refactoring previous code, a marker layer was also able to be added that displayed the hotels for those 4 cities.

![](/Vacation_Itinerary/WeatherPy_travel_map_markers.png)

## Summary

Using API calls and python modules, we were able to pull data and create a program that would allow users to help plan trips to areas that they would find enjoyable.