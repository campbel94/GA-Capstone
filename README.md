# General Assembly DSI Capstone: Plant-Based Planning

##  Background

The rise of plant-based meat alternatives in recent years is one of, if not the fastest growing trends in the food industry. As consumer demand for vegan alternatives increases, plant-based food companies, as well as the stores and restaurants that sell their products, have scaled up their supply accordingly. 

From April to May of 2020 alone (the early stages of the COVID-19 pandemic) [plant-based meat substitutes saw a 35 percent increase in restaurant sales and a 53 percent increase in uncooked grocery sales.](https://www.nytimes.com/2020/05/22/dining/plant-based-meats-coronavirus.html?searchResultPosition=2) The growing popularity of plant-based alternatives is not only great news for the health of our planet and animal welfare, it provides exciting new fast food options for consumers who begrudgingly abandoned Big Macs and McChicken sandwiches in the name of sustainability. 

One restaurant chain providing exactly that, and a personal favorite of mine, is [Project Pollo](https://www.projectpollo.com/). Founded as a lone food truck in downtown San Antonio, Project Pollo locations spread across Texas by serving classically comforting fast food that is 100% vegan.

> "*Project Pollo was built around one simple mission, challenging chicken by putting people over profits and offering pollo with a purpose.*
>
>*We sought out to redefine consumer behavior by offering convenience, quality, and accessibility. Inevitably the future of mass consumption is plant based; we want to lead that front, to put one on every corner and have affordable options in every community that surrounds us.*"<br><br>
>        - [*Project Pollo Mission Statement*](https://www.projectpollo.com/about)

##  Business Case / Task at Hand

For this capstone project we'll be imagining that Project Pollo is planning to open a new Austin, TX location and has enlisted the help of a data scientist to identify neighborhoods that could be a good fit. While there are many factors that contribute to the final decision such as zoning restrictions, lot availability, cost of rent, etc. understanding the culinary makeup of the city is a key starting point. That’s where I come in.<br>

**The task at hand** is to identify what (if any) relationships exist between categories of restaurants and the neighborhoods they’re located in.

## Data Sources
The three categories of data used in this project are listed below:
- [City of Austin Open Data Portal](https://data.austintexas.gov/Building-and-Development/Neighborhoods/a7ap-j2yt)
    - Geometric MultiPolygon datatypes for each neighborhood in Austin provided by the Housing and Planning Department. The dataset also includes neighborhood name and total area.
    - Data is accessible via SODA API and is exportable in CSV, GeoJSON or Shapefile format.
- [Oatly](https://us.oatly.com/pages/oatfinder), [Beyond Meat](https://www.beyondmeat.com/where-to-find) and [Impossible Foods](https://impossiblefoods.com/locations) Store Locators
    - Interactive maps used to locate nearby restaurants that carry plant-based meat and dairy alternatives. 
- [Foursquare Developers API](https://developer.foursquare.com/)
    - Foursquare's `Explore` API endpoint provides the name, location and category of every restaurant within a user-defined radius of any set of geographic coordinates.


## Python Libraries
The imported libraries listed below were essential to the project's workflow:
###### Data Collection
The JSON and Requests libraries were used to leverage the Socrata Open Data API, Google Maps Geocoding API and Foursquare's Developers API.
- `import json`
- `import requests`
- `from sodapy import Socrata`
- `import gmaps`
###### Data Cleaning & Vizualizing
Pandas, Matplotlib and Numpy allowed for the standard data cleaning, feature engineering and visualization procedures. Geopandas and Shapley were leveraged to calculate geographic distances and create neighborhood heatmaps.
- `import pandas as pd`
- `import numpy as np`
- `import matplotlib.pyplot as plt`
- `import geopandas as gpd`
- `from shapely.geometry import Polygon, Point`
###### Modeling
Silhouette scores helped gauge the appropraite number of total clusters while Kmeans was our model of choice.
- `from sklearn.metrics import silhouette_score`
- `from sklearn.cluster import KMeans`

## Data Dictionary
###### Neighborhood Data
| Column  | Datatype  | Description  |
|----------|----------|--------------|
| the_geom | MultiPolygon (dictionary) | - A MultiPolygon list of geo coordinates outlining each neighborhood in Austin, TX |
| stripped_geom | float (list of lists) | - Same list of geo coordinates as 'the_geom' column but with the 'type' and 'coordinates' keys removed<br>- Used for calculating centroids and anything involving the Shapely package |
| neighname | string  |   - Neighborhood name  |
| shapely_center_coords | Shapely geometry point |  - Neighborhood center points derived from calling Shapely's Polygon().centroid function<br>Ordered in (lng, lat) |
| calculated_center_coords | float (list) |  - Manually calculated neighborhood center points<br>- Ordered in (lat, lng)<br>- Latitude calculation: (lat<sub>max</sub> +  lat<sub>min</sub>) / 2<br>- Longitude calculation: (lng<sub>max</sub> +  lng<sub>min</sub>) / 2)
| gmaps_center_coords | float (list) |   - Neighborhood center points derived from Google Map's [Geocoding API](https://developers.google.com/maps/documentation/geocoding/overview)<br>- Ordered in (lat, lng)  |

###### Restaurant Data
| Column  | Datatype  | Description  |
|----------|----------|--------------|
| venue | string  |   - Restaurant Name  |
| address | string  |   - Street address  |
| latitude | float  |   - Latitude provided by Forsquare API |
| longitude | float |   - Longitude provided by Forsquare API  |
| neighborhood | string |   - Neighborhood the venue is located in<br>- Manually calculated  |
| category |string |  - Restaurant category provided by Forsquare  |
| cluster_label | string |  - The specific cluster each row was assigned to by the Kmeans clustering model |

## Workflow
The sections below provide an overview of the data sources; as well as the collection and cleaning, exploratory analysis and modeling process.
### Data Collection & Cleaning
The neighborhood data was created first so that it could be leveraged to assign restaurant venues their appropriate neighborhood in the subsequent dataframes. 
###### Neighborhood Data (Austin Open Data Portal)
- Call `request.get()` (Socrata) to retrieve `the_geom` and `neighname` of each neighborhood in Austin.
- Identify central coordinates of each neighborhood. These coordinates will later serve as the foci (`ll` parameter) of which Foursquare will return the surrounding restaurants.
- Engineer new `stripped_geom` column by removing dictionary keys from `the_geom`. This allows for easier manual calculations of neighborhood center coordinates.
- Produce and evaluate neighborhood center coordinates using three different methods: **Shapely’s `Polygon.centroid()`**, **Manual Calculation** and **Google Map’s Geocoding API**. These will produce our `shapely_center_coords`, `calculated_center_coords` and `gmaps_center_coords` columns.
    - Calculate and analyze distances between the center coordinates for each neighborhood to understand how much they vary and the distrobution of the variences.
    - Mark the coordinates on a Gmaps figure for an additional visual reference.
###### Restaurant Data (Foursquare)
- Instantiate a `restaurant_df` pandas dataframe.
- Write a for loop that iterates through the list of neighborhood center coordinates and collects each surrounding `venue` and corresponding `address`, `latitude`, `longitude`, `neighborhood` and `category` from Foursquare’s ‘Explore’ API endpoint. Data from each iteration (neighborhood center coordinates) is appended to `restaurant_df`.
- After cleaning the dataframe, the `Polygon.contains(Point)` method is used to assign venues their appropriate neighborhood. For restaurants located between neighborhoods or that can’t be placed using the Shapely method, they are assigned to whichever neighborhood has the nearest center coordinates.
##### Restaurant  Data (Impossible, Beyond & Oatly Store Locators)
In addition to the venues available through Foursquare, I wanted to add additional categories called `Serves Impossible Products`, `Serves Beyond Products` and `Serves Oatly Products` to emphasize venues and neighborhoods already selling plant-based meat and dairy alternatives even if they weren't classified as `Vegetarian / Vegan` by Foursquare. These additions are limited to restaurants, cafes and coffee shops (no grocery stores), and there was no deduplication of venues after being appended to `restaurant_df`.<br>
In each of the three instances below, address were gathered by navigating straight to the 'Store Locator' page of the company website and searching for Austin, TX.
###### Impossible Foods Location Data
-  Copied the `<div></div>` tags that contained the list of restaurant markers from the interactive map, and saved them as a string in a new Jupyter Notebook.
- Wrote a series of for loops to split, strip and extract restaurant names and their corresponding addresses into a new `venues_df` pandas dataframe. 
- Used each restaurant's address to retrieve geo coordinates from Google Maps' Geocoding API
- Assigned each venue a neighborhood by using the same `Polygon.contains(Point)` method as before.
###### Beyond Meat Location Data
- List of venue names and addresses was short enough to be manually assembled.
- Used each restaurant's address to retrieve geo coordinates from Google Maps' Geocoding API
- Assigned each venue a neighborhood by using the same `Polygon.contains(Point)` method as before.
###### Oatly Location Data
- List of venue names and addresses was short enough to be manually assembled.
- Used each restaurant's address to retrieve geo coordinates from Google Maps' Geocoding API
- Assigned each venue a neighborhood by using the same `Polygon.contains(Point)` method as before.
<br><br>
---
With all Foursquare, Impossible, Beyond and Oatly Data collected, cleaned and assigned a neighborhood, the four dataframes are appended into one with the following columns: `venue`, `address`, `latitude`, `longitude`, `neighborhood`, `category`. 
<!-- <br>
<img height="700" src="/assets/df.png"/>
<br>
 -->
### Exploratory Data Analysis
Before modeling our data and digging into the clusters, conducting an initial exploratory analysis will familiarize us with general trends and distributions of the data.
###### Distributions
Looking at a bar chart of **restaurant count by neighborhood** shows a fairly gradually decline from Downtown (not surprisingly) to the smaller neighborhoods on the outskirts of town. The average restaurant count per neighborhood is 15 (although this could be slightly skewed by the addition of Impossible, Beyond and Oatly data), with a max of 60, minimum of 1 and a standard deviation of 11.<br><br>
**Restaurant count by category** however is much more skewed towards a select few. Over 50% of the restaurants in the dataset belong to just eight of the 77 total categories. The average restaurant count per category is 18.6 with a max of 160, a minimum of 1 and standard deviation of 29.7.
###### Neighborhood Heatmaps
Neighborhood heatmaps of restaurant count by neighborhood give a feel for where the majority of restaurants are located. Interestingly, some of the highest count neighborhoods are far from Downtown and Central Austin, which I was not expecting.
###### Categories of Interest
Using a for loop, bar charts were created to display the neighborhoods where each of the high priority restaurant categories are found.
### Modeling
###### Preprocessing
- Simplified dataframe to just `neighborhood` and `category`
- One hot encoded the category column 
- Used groupby().sum() method to produce 96 rows (each neighborhood) and 77 columns (each category).
###### Clustering
- Produced the average silhouette score for each Kmeans model with `n_clusters` (number of clusters) 2 - 10.
### Continued Exploration
- Decided to go with three clusters due to the relatively high silhouette score and room for exploration
- Mapped the cluster labels back to the original dataframe that contains retaurant names.
- This will allow us to dig in even further.

## Key Learnings
- Objectivity and consistency of data hasa huge impact on learnings
- Visualizations and EDA provided easier to understand learnings than the Kmeans modeling did