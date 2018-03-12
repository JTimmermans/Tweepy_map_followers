
# Locations of twitter followers

The following script retrieves the location of twitter followers, along with their name using the python 'Tweepy' module. When all the followers for a certain account are retrieved, the follower's location are geocoded. Finally all the geocoded addresses are plotted on a map.

## Retrieving twitter location data using Tweepy

Import the required modules and set your working directory


```python
import pandas as pd
import tweepy
import csv
import os
import matplotlib as mpl
import time

```


```python
os.chdir('C:\Users\Joris\Google Drive\Gima\Python practice\Tweepy\data\\')
```

Set the verification variables and authenticate your developper account on twitter. For more information visit: https://developer.twitter.com/en/docs/basics/authentication/overview/authentication-and-authorization


```python
consumer_key =#use your own keys
consumer_secret =#use your own keys
access_token =#use your own keys
access_token_secret =#use your own keys

auth = tweepy.OAuthHandler(consumer_key, consumer_secret)
auth.set_access_token(access_token, access_token_secret)
```

The function get_user_followers_location creates a .csv file that contains the users names, screen names and location from the twitter description from all the followers of a twitter users. It takes one argument: a twitter username.


```python
def get_user_followers_locations(username):
    #set the twitter handle for further use
    twitter_handle = username

    api = tweepy.API(auth, wait_on_rate_limit = True, wait_on_rate_limit_notify = False) '''If rate limit of the Twitter API 
    is exceeded, the script waits, and continues later. '''

    o_file = open(twitter_handle+'_followers_python.csv', 'wb')
    f = csv.writer(o_file, delimiter =',')
    f.writerow(["screenname", "name", "location"])

    users = tweepy.Cursor(api.followers, screen_name=twitter_handle, count = 200).items()

    for u in users:
        screenname = u.screen_name.encode('utf-8').strip()
        name = u.name.encode('utf-8').strip()
        location = u.location.encode('utf-8').strip()

        f.writerow([screenname, name, location])
    o_file.close()
    
    #Return the twitter username for later use
    return twitter_handle
```


```python
twitter_handle = get_user_followers_locations('JimPaul95')
```

## Retrieving coordinate pairs for each follower

Read the created data and print a preview of the first five rows.


```python
pd.options.mode.chained_assignment = None  # default='warn'
df = pd.read_csv(twitter_handle+'_followers_python.csv', sep = ',')
print(df.head(5))
```

            screenname                  name               location
    0  OosterhuisPeter      Peter Oosterhuis                Utrecht
    1          seonunl  ZoekmachineMarketing                  Gouda
    2      RSG_Holland    Royal Safety Group  Oostvoorne, Dalweg 26
    3    Wouter_dehaas        Wouter de Haas                    NaN
    4     ingejansen81           Inge Jansen  Driebergen-Rijsenburg
    

Create a dataframe containing all the users that have a location.


```python
df1 = df[(df.location.notnull())]
print(df1.head(5))
```

            screenname                  name               location
    0  OosterhuisPeter      Peter Oosterhuis                Utrecht
    1          seonunl  ZoekmachineMarketing                  Gouda
    2      RSG_Holland    Royal Safety Group  Oostvoorne, Dalweg 26
    4     ingejansen81           Inge Jansen  Driebergen-Rijsenburg
    8   lisettehahagen         Lisette Hagen  Amersfoort, Nederland
    

To obtain latitude and longitude values for each location a geocoder is used. First the appropiate module is imported and a geocoder is created. Nomatim is the geocoder of openstreetmap based on a fair use policy, which can be found at: https://wiki.openstreetmap.org/wiki/Nominatim_usage_policy . More information about the geocoder plugin can be found at: https://geopy.readthedocs.io/en/1.10.0/ .


```python
from geopy.geocoders import Nominatim
geolocator = Nominatim()
```

Two functions to obtain latitude and longitude values were defined. Then, those the latitude and longitude were applied to their respective columns. Finally the first ten rows are printed to confirm the results.


```python
def get_lat(input_location):
    location = geolocator.geocode(input_location)
    #if location is not None and location.latitude is not None:
    lat = (location.latitude)
        #return lat
    return lat

def get_lon(input_location):
    #time.sleep(0.5)
    location = geolocator.geocode(input_location)
    #if location is not None and location.longitude is not None:
    lon = (location.longitude)
        #return lon
    return lon

df1['latitude'] = df1['location'].apply(get_lat)
df1['longitude'] = df1['location'].apply(get_lon)
print(df1.head(5))
```

            screenname                  name               location   latitude  \
    0  OosterhuisPeter      Peter Oosterhuis                Utrecht  52.080952   
    1          seonunl  ZoekmachineMarketing                  Gouda  52.018119   
    2      RSG_Holland    Royal Safety Group  Oostvoorne, Dalweg 26  51.911340   
    4     ingejansen81           Inge Jansen  Driebergen-Rijsenburg  52.047481   
    8   lisettehahagen         Lisette Hagen  Amersfoort, Nederland  52.163774   
    
       longitude                              geometry  
    0   5.127680  POINT (5.12768031549829 52.08095165)  
    1   4.711122   POINT (4.7111221346978 52.01811935)  
    2   4.117732          POINT (4.1177322 51.9113402)  
    4   5.275569   POINT (5.27556919443128 52.0474813)  
    8   5.396588    POINT (5.3965879493517 52.1637739)  
    

## Plotting follower data on a map using cartopy and matplotlib

The last step is to print each followers location as a point on a map. This will be done in the next section. With help of:
http://andrewgaidus.com/leaflet_webmaps_python/.


```python
#Import the necessary Python moduless
import pandas as pd
import geopandas as gpd
import numpy as np
from geopandas.tools import sjoin
import folium
from folium.plugins import MarkerCluster
from folium import IFrame
import shapely
from shapely.geometry import Point
import unicodedata
import pysal as ps
```

The coordinates are concenated as a new geometry column, and written to the ESRI shapefile (.shp) format.


```python

def create_geometry(dataset):
    dataset['geometry'] = dataset.apply(lambda x: Point((float(x.longitude), float(x.latitude))), axis=1)
    df2 = gpd.GeoDataFrame(dataset, geometry='geometry') 
    df2.crs= "+proj=longlat +ellps=WGS84 +datum=WGS84 +no_defs"
    df2.to_file(twitter_handle+'_Geometries.shp', driver='ESRI Shapefile')
    return df2


df2 = create_geometry(df1)
```

Create a new map object using the Folium mode, set the extent to cover all the points.


```python
#Create SF basemap specifying map center, zoom level, and using the default OpenStreetMap tiles
#creating a map that’s centered to our sample
tweetmap = folium.Map(location=[df2['latitude'].mean(), df2['longitude'].mean()], zoom_start=7)
#tweetmap = folium.Map([52.092876, 5.104480], zoom_start = 12)

```

Define a function to add all points to the Folium map, and call this function for the tweetmap. In the markers the the column values for screenname, name, and locations are displayed.


```python
def add_point_clusters(mapobj, gdf, popup_field_list):
    #Create empty lists to contain the point coordinates and the point pop-up information
    coords, popups = [], [] 
    #Loop through each record in the GeoDataFrame
    for i, row in gdf.iterrows():
        #Append lat and long coordinates to "coords" list
        coords.append([row.geometry.y, row.geometry.x])
        #Create a string of HTML code used in the IFrame popup
        #Join together the fields in "popup_field_list" with a linebreak between them
        label = '<br>'.join([row[field] for field in popup_field_list])
        #Append an IFrame that uses the HTML string to the "popups" list 
        popups.append(IFrame(label, width = 300, height = 100))
        
    #Create a Folium feature group for this layer, since we will be displaying multiple layers
    pt_lyr = folium.FeatureGroup(name = 'pt_lyr')
    
    #Add the clustered points of crime locations and popups to this layer
    pt_lyr.add_child(MarkerCluster(locations = coords, popups = popups))
    
    #Add this point layer to the map object
    mapobj.add_child(pt_lyr)
    return mapobj

tweetmap = add_point_clusters(tweetmap, df2, ['screenname', 'name', 'location'])
```

The last step is to add the tweetmap as a layer to a standard Leaftlet framework, and display this framework. A preview can be seen at: https://jtimmermans.github.io/Tweepy_map_followers/


```python
folium.LayerControl().add_to(tweetmap) #Add layer control to toggle on/off
tweetmap.save(twitter_handle+'.html') #save HTML
tweetmap #display map
```




The map is not displayed here, since it is a markdown file.



Done!
