
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

The last step is to add the tweetmap as a layer to a standard Leaftlet framework, and display this framework.


```python
folium.LayerControl().add_to(tweetmap) #Add layer control to toggle on/off
tweetmap.save(twitter_handle+'.html') #save HTML
tweetmap #display map
```




<div style="width:100%;"><div style="position:relative;width:100%;height:0;padding-bottom:60%;"><iframe src="data:text/html;charset=utf-8;base64,PCFET0NUWVBFIGh0bWw+CjxoZWFkPiAgICAKICAgIDxtZXRhIGh0dHAtZXF1aXY9ImNvbnRlbnQtdHlwZSIgY29udGVudD0idGV4dC9odG1sOyBjaGFyc2V0PVVURi04IiAvPgogICAgPHNjcmlwdD5MX1BSRUZFUl9DQU5WQVMgPSBmYWxzZTsgTF9OT19UT1VDSCA9IGZhbHNlOyBMX0RJU0FCTEVfM0QgPSBmYWxzZTs8L3NjcmlwdD4KICAgIDxzY3JpcHQgc3JjPSJodHRwczovL2Nkbi5qc2RlbGl2ci5uZXQvbnBtL2xlYWZsZXRAMS4yLjAvZGlzdC9sZWFmbGV0LmpzIj48L3NjcmlwdD4KICAgIDxzY3JpcHQgc3JjPSJodHRwczovL2FqYXguZ29vZ2xlYXBpcy5jb20vYWpheC9saWJzL2pxdWVyeS8xLjExLjEvanF1ZXJ5Lm1pbi5qcyI+PC9zY3JpcHQ+CiAgICA8c2NyaXB0IHNyYz0iaHR0cHM6Ly9tYXhjZG4uYm9vdHN0cmFwY2RuLmNvbS9ib290c3RyYXAvMy4yLjAvanMvYm9vdHN0cmFwLm1pbi5qcyI+PC9zY3JpcHQ+CiAgICA8c2NyaXB0IHNyYz0iaHR0cHM6Ly9jZG5qcy5jbG91ZGZsYXJlLmNvbS9hamF4L2xpYnMvTGVhZmxldC5hd2Vzb21lLW1hcmtlcnMvMi4wLjIvbGVhZmxldC5hd2Vzb21lLW1hcmtlcnMuanMiPjwvc2NyaXB0PgogICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL2Nkbi5qc2RlbGl2ci5uZXQvbnBtL2xlYWZsZXRAMS4yLjAvZGlzdC9sZWFmbGV0LmNzcyIgLz4KICAgIDxsaW5rIHJlbD0ic3R5bGVzaGVldCIgaHJlZj0iaHR0cHM6Ly9tYXhjZG4uYm9vdHN0cmFwY2RuLmNvbS9ib290c3RyYXAvMy4yLjAvY3NzL2Jvb3RzdHJhcC5taW4uY3NzIiAvPgogICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL21heGNkbi5ib290c3RyYXBjZG4uY29tL2Jvb3RzdHJhcC8zLjIuMC9jc3MvYm9vdHN0cmFwLXRoZW1lLm1pbi5jc3MiIC8+CiAgICA8bGluayByZWw9InN0eWxlc2hlZXQiIGhyZWY9Imh0dHBzOi8vbWF4Y2RuLmJvb3RzdHJhcGNkbi5jb20vZm9udC1hd2Vzb21lLzQuNi4zL2Nzcy9mb250LWF3ZXNvbWUubWluLmNzcyIgLz4KICAgIDxsaW5rIHJlbD0ic3R5bGVzaGVldCIgaHJlZj0iaHR0cHM6Ly9jZG5qcy5jbG91ZGZsYXJlLmNvbS9hamF4L2xpYnMvTGVhZmxldC5hd2Vzb21lLW1hcmtlcnMvMi4wLjIvbGVhZmxldC5hd2Vzb21lLW1hcmtlcnMuY3NzIiAvPgogICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL3Jhd2dpdC5jb20vcHl0aG9uLXZpc3VhbGl6YXRpb24vZm9saXVtL21hc3Rlci9mb2xpdW0vdGVtcGxhdGVzL2xlYWZsZXQuYXdlc29tZS5yb3RhdGUuY3NzIiAvPgogICAgPHN0eWxlPmh0bWwsIGJvZHkge3dpZHRoOiAxMDAlO2hlaWdodDogMTAwJTttYXJnaW46IDA7cGFkZGluZzogMDt9PC9zdHlsZT4KICAgIDxzdHlsZT4jbWFwIHtwb3NpdGlvbjphYnNvbHV0ZTt0b3A6MDtib3R0b206MDtyaWdodDowO2xlZnQ6MDt9PC9zdHlsZT4KICAgIAogICAgICAgICAgICA8c3R5bGU+ICNtYXBfOTc0MjQ5MGFhZDhiNDgyN2E4ODVmMDQzOTBjYjM5ZTYgewogICAgICAgICAgICAgICAgcG9zaXRpb24gOiByZWxhdGl2ZTsKICAgICAgICAgICAgICAgIHdpZHRoIDogMTAwLjAlOwogICAgICAgICAgICAgICAgaGVpZ2h0OiAxMDAuMCU7CiAgICAgICAgICAgICAgICBsZWZ0OiAwLjAlOwogICAgICAgICAgICAgICAgdG9wOiAwLjAlOwogICAgICAgICAgICAgICAgfQogICAgICAgICAgICA8L3N0eWxlPgogICAgICAgIAogICAgPHNjcmlwdCBzcmM9Imh0dHBzOi8vY2RuanMuY2xvdWRmbGFyZS5jb20vYWpheC9saWJzL2xlYWZsZXQubWFya2VyY2x1c3Rlci8xLjEuMC9sZWFmbGV0Lm1hcmtlcmNsdXN0ZXIuanMiPjwvc2NyaXB0PgogICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL2NkbmpzLmNsb3VkZmxhcmUuY29tL2FqYXgvbGlicy9sZWFmbGV0Lm1hcmtlcmNsdXN0ZXIvMS4xLjAvTWFya2VyQ2x1c3Rlci5jc3MiIC8+CiAgICA8bGluayByZWw9InN0eWxlc2hlZXQiIGhyZWY9Imh0dHBzOi8vY2RuanMuY2xvdWRmbGFyZS5jb20vYWpheC9saWJzL2xlYWZsZXQubWFya2VyY2x1c3Rlci8xLjEuMC9NYXJrZXJDbHVzdGVyLkRlZmF1bHQuY3NzIiAvPgo8L2hlYWQ+Cjxib2R5PiAgICAKICAgIAogICAgICAgICAgICA8ZGl2IGNsYXNzPSJmb2xpdW0tbWFwIiBpZD0ibWFwXzk3NDI0OTBhYWQ4YjQ4MjdhODg1ZjA0MzkwY2IzOWU2IiA+PC9kaXY+CiAgICAgICAgCjwvYm9keT4KPHNjcmlwdD4gICAgCiAgICAKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGJvdW5kcyA9IG51bGw7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgdmFyIG1hcF85NzQyNDkwYWFkOGI0ODI3YTg4NWYwNDM5MGNiMzllNiA9IEwubWFwKAogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgJ21hcF85NzQyNDkwYWFkOGI0ODI3YTg4NWYwNDM5MGNiMzllNicsCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICB7Y2VudGVyOiBbNTIuMTA4NjAyLDQuOTc2OTQ1Njc1NTFdLAogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgem9vbTogNywKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIG1heEJvdW5kczogYm91bmRzLAogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgbGF5ZXJzOiBbXSwKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIHdvcmxkQ29weUp1bXA6IGZhbHNlLAogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgY3JzOiBMLkNSUy5FUFNHMzg1NwogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHRpbGVfbGF5ZXJfNWU3YzUzZmRlM2RlNDVjOGJjNjg4ZTQwYTlhZDYxZTEgPSBMLnRpbGVMYXllcigKICAgICAgICAgICAgICAgICdodHRwczovL3tzfS50aWxlLm9wZW5zdHJlZXRtYXAub3JnL3t6fS97eH0ve3l9LnBuZycsCiAgICAgICAgICAgICAgICB7CiAgImF0dHJpYnV0aW9uIjogbnVsbCwgCiAgImRldGVjdFJldGluYSI6IGZhbHNlLCAKICAibWF4Wm9vbSI6IDE4LCAKICAibWluWm9vbSI6IDEsIAogICJub1dyYXAiOiBmYWxzZSwgCiAgInN1YmRvbWFpbnMiOiAiYWJjIgp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF85NzQyNDkwYWFkOGI0ODI3YTg4NWYwNDM5MGNiMzllNik7CiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIGZlYXR1cmVfZ3JvdXBfMGFmMWQ2MTUxMTViNGZkMzhiYTdhNmVmYzg5OGQ1M2QgPSBMLmZlYXR1cmVHcm91cCgKICAgICAgICAgICAgICAgICkuYWRkVG8obWFwXzk3NDI0OTBhYWQ4YjQ4MjdhODg1ZjA0MzkwY2IzOWU2KTsKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgbWFya2VyX2NsdXN0ZXJfYzA3ZWE2N2ZhMDY1NDg0ZDhmODMwYzA3MTM1NjIxMzAgPSBMLm1hcmtlckNsdXN0ZXJHcm91cCh7CiAgICAgICAgICAgICAgICAKICAgICAgICAgICAgfSk7CiAgICAgICAgICAgIGZlYXR1cmVfZ3JvdXBfMGFmMWQ2MTUxMTViNGZkMzhiYTdhNmVmYzg5OGQ1M2QuYWRkTGF5ZXIobWFya2VyX2NsdXN0ZXJfYzA3ZWE2N2ZhMDY1NDg0ZDhmODMwYzA3MTM1NjIxMzApOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl8xNjczZmQ4YTE1YTY0NjRkYWIzNDQ3NjBkM2Y4Y2ZmMyA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzUyLjA4MDk1MTY1LDUuMTI3NjgwMzE1NV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFya2VyX2NsdXN0ZXJfYzA3ZWE2N2ZhMDY1NDg0ZDhmODMwYzA3MTM1NjIxMzApOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfYTU5M2EyMzFhNDA3NDQ5OTk2ODMzOWY1YjUzOTJhZjEgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGlfZnJhbWVfNmJkY2YyNmExODgwNGZiM2E3NDEyNWIxYWIwNzA2MDQgPSAkKCc8aWZyYW1lIHNyYz0iZGF0YTp0ZXh0L2h0bWw7Y2hhcnNldD11dGYtODtiYXNlNjQsQ2lBZ0lDQlBiM04wWlhKb2RXbHpVR1YwWlhJOFluSStVR1YwWlhJZ1QyOXpkR1Z5YUhWcGN6eGljajVWZEhKbFkyaDAiIHdpZHRoPSIzMDAiIHN0eWxlPSJib3JkZXI6bm9uZSAhaW1wb3J0YW50OyIgaGVpZ2h0PSIxMDAiPjwvaWZyYW1lPicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfYTU5M2EyMzFhNDA3NDQ5OTk2ODMzOWY1YjUzOTJhZjEuc2V0Q29udGVudChpX2ZyYW1lXzZiZGNmMjZhMTg4MDRmYjNhNzQxMjViMWFiMDcwNjA0KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfMTY3M2ZkOGExNWE2NDY0ZGFiMzQ0NzYwZDNmOGNmZjMuYmluZFBvcHVwKHBvcHVwX2E1OTNhMjMxYTQwNzQ0OTk5NjgzMzlmNWI1MzkyYWYxKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzIxN2YxZTBkYjg4ZjRkYTM4YzI2NTllODkyNjE5NDc0ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbNTIuMDE4MTE5MzUsNC43MTExMjIxMzQ3XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXJrZXJfY2x1c3Rlcl9jMDdlYTY3ZmEwNjU0ODRkOGY4MzBjMDcxMzU2MjEzMCk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9lY2Q0MzhjNzI0OTk0MzdlODMyNDk3NTU4NThlNDAwNCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaV9mcmFtZV9mODcyY2Y5OTAyMmI0ODFjOGY5MjMxMDI5ODliNTVlYiA9ICQoJzxpZnJhbWUgc3JjPSJkYXRhOnRleHQvaHRtbDtjaGFyc2V0PXV0Zi04O2Jhc2U2NCxDaUFnSUNCelpXOXVkVzVzUEdKeVBscHZaV3R0WVdOb2FXNWxUV0Z5YTJWMGFXNW5QR0p5UGtkdmRXUmgiIHdpZHRoPSIzMDAiIHN0eWxlPSJib3JkZXI6bm9uZSAhaW1wb3J0YW50OyIgaGVpZ2h0PSIxMDAiPjwvaWZyYW1lPicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfZWNkNDM4YzcyNDk5NDM3ZTgzMjQ5NzU1ODU4ZTQwMDQuc2V0Q29udGVudChpX2ZyYW1lX2Y4NzJjZjk5MDIyYjQ4MWM4ZjkyMzEwMjk4OWI1NWViKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfMjE3ZjFlMGRiODhmNGRhMzhjMjY1OWU4OTI2MTk0NzQuYmluZFBvcHVwKHBvcHVwX2VjZDQzOGM3MjQ5OTQzN2U4MzI0OTc1NTg1OGU0MDA0KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2JkYWI3Yjk0OWM1YzQyMmQ4MzAxNWFhNWUwYjU5ZDMzID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbNTEuOTExMzQwMiw0LjExNzczMjJdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcmtlcl9jbHVzdGVyX2MwN2VhNjdmYTA2NTQ4NGQ4ZjgzMGMwNzEzNTYyMTMwKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzA4ZDg0NTcyOWRhOTRjNGJiYjA1ODc5OTcxNWExNmZkID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBpX2ZyYW1lX2MxODYyMDUxNWM2ZTQ4OWY4YjAzYjFiMDQ5NWE0YjQzID0gJCgnPGlmcmFtZSBzcmM9ImRhdGE6dGV4dC9odG1sO2NoYXJzZXQ9dXRmLTg7YmFzZTY0LENpQWdJQ0JTVTBkZlNHOXNiR0Z1WkR4aWNqNVNiM2xoYkNCVFlXWmxkSGtnUjNKdmRYQThZbkkrVDI5emRIWnZiM0p1WlN3Z1JHRnNkMlZuSURJMiIgd2lkdGg9IjMwMCIgc3R5bGU9ImJvcmRlcjpub25lICFpbXBvcnRhbnQ7IiBoZWlnaHQ9IjEwMCI+PC9pZnJhbWU+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8wOGQ4NDU3MjlkYTk0YzRiYmIwNTg3OTk3MTVhMTZmZC5zZXRDb250ZW50KGlfZnJhbWVfYzE4NjIwNTE1YzZlNDg5ZjhiMDNiMWIwNDk1YTRiNDMpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl9iZGFiN2I5NDljNWM0MjJkODMwMTVhYTVlMGI1OWQzMy5iaW5kUG9wdXAocG9wdXBfMDhkODQ1NzI5ZGE5NGM0YmJiMDU4Nzk5NzE1YTE2ZmQpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfYjQ3NzFkYjI1MTk3NDNjNmJiMDhiMTAxNzg5N2IzZDEgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFs1Mi4wNDc0ODEzLDUuMjc1NTY5MTk0NDNdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcmtlcl9jbHVzdGVyX2MwN2VhNjdmYTA2NTQ4NGQ4ZjgzMGMwNzEzNTYyMTMwKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2IzZTk0YmJhZjNlOTRmMzdhOGI2NjQ5YWYwM2E3MjhiID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBpX2ZyYW1lX2RlZjFhOWJiMzZhNTQ2ZGZiZjBiNDViNDdkYWZmZTdlID0gJCgnPGlmcmFtZSBzcmM9ImRhdGE6dGV4dC9odG1sO2NoYXJzZXQ9dXRmLTg7YmFzZTY0LENpQWdJQ0JwYm1kbGFtRnVjMlZ1T0RFOFluSStTVzVuWlNCS1lXNXpaVzQ4WW5JK1JISnBaV0psY21kbGJpMVNhV3B6Wlc1aWRYSm4iIHdpZHRoPSIzMDAiIHN0eWxlPSJib3JkZXI6bm9uZSAhaW1wb3J0YW50OyIgaGVpZ2h0PSIxMDAiPjwvaWZyYW1lPicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfYjNlOTRiYmFmM2U5NGYzN2E4YjY2NDlhZjAzYTcyOGIuc2V0Q29udGVudChpX2ZyYW1lX2RlZjFhOWJiMzZhNTQ2ZGZiZjBiNDViNDdkYWZmZTdlKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfYjQ3NzFkYjI1MTk3NDNjNmJiMDhiMTAxNzg5N2IzZDEuYmluZFBvcHVwKHBvcHVwX2IzZTk0YmJhZjNlOTRmMzdhOGI2NjQ5YWYwM2E3MjhiKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzE4YmUxZjk4NWZjMzRkZTZhYjkzMTJlZGIzYWUwZTU0ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbNTIuMTYzNzczOSw1LjM5NjU4Nzk0OTM1XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXJrZXJfY2x1c3Rlcl9jMDdlYTY3ZmEwNjU0ODRkOGY4MzBjMDcxMzU2MjEzMCk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF82NjM1N2Q1MmVkYjM0ZTY2ODZkYjBkM2JmYTlkOTQwYiA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaV9mcmFtZV9jNGU2ZTc0ZWI0NjY0NDcyOTYyMTYxNDU1YTg3NTYwMCA9ICQoJzxpZnJhbWUgc3JjPSJkYXRhOnRleHQvaHRtbDtjaGFyc2V0PXV0Zi04O2Jhc2U2NCxDaUFnSUNCc2FYTmxkSFJsYUdGb1lXZGxianhpY2o1TWFYTmxkSFJsSUVoaFoyVnVQR0p5UGtGdFpYSnpabTl2Y25Rc0lFNWxaR1Z5YkdGdVpBPT0iIHdpZHRoPSIzMDAiIHN0eWxlPSJib3JkZXI6bm9uZSAhaW1wb3J0YW50OyIgaGVpZ2h0PSIxMDAiPjwvaWZyYW1lPicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNjYzNTdkNTJlZGIzNGU2Njg2ZGIwZDNiZmE5ZDk0MGIuc2V0Q29udGVudChpX2ZyYW1lX2M0ZTZlNzRlYjQ2NjQ0NzI5NjIxNjE0NTVhODc1NjAwKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfMThiZTFmOTg1ZmMzNGRlNmFiOTMxMmVkYjNhZTBlNTQuYmluZFBvcHVwKHBvcHVwXzY2MzU3ZDUyZWRiMzRlNjY4NmRiMGQzYmZhOWQ5NDBiKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2M0ZTMwZjI3ZDJhODRjNGFiMWNjMDg3ZmFlNTk3ODZlID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbNTIuMzAwNTU4NSw0LjY3NTMyMDU1Mjk2XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXJrZXJfY2x1c3Rlcl9jMDdlYTY3ZmEwNjU0ODRkOGY4MzBjMDcxMzU2MjEzMCk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF85ZjcxNGI3Y2QzZWQ0YTQ4OGM0MTcwYTFhNjJhY2YxOSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaV9mcmFtZV9hMTAzYzcxM2NhYWU0Mjc3YTMwNmMzNWI2YTcwNTMxMiA9ICQoJzxpZnJhbWUgc3JjPSJkYXRhOnRleHQvaHRtbDtjaGFyc2V0PXV0Zi04O2Jhc2U2NCxDaUFnSUNCVGNHOTBUMjVOWldScFkzTThZbkkrVTNCdmRFOXVUV1ZrYVdOelBHSnlQa2h2YjJaa1pHOXljQT09IiB3aWR0aD0iMzAwIiBzdHlsZT0iYm9yZGVyOm5vbmUgIWltcG9ydGFudDsiIGhlaWdodD0iMTAwIj48L2lmcmFtZT4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzlmNzE0YjdjZDNlZDRhNDg4YzQxNzBhMWE2MmFjZjE5LnNldENvbnRlbnQoaV9mcmFtZV9hMTAzYzcxM2NhYWU0Mjc3YTMwNmMzNWI2YTcwNTMxMik7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyX2M0ZTMwZjI3ZDJhODRjNGFiMWNjMDg3ZmFlNTk3ODZlLmJpbmRQb3B1cChwb3B1cF85ZjcxNGI3Y2QzZWQ0YTQ4OGM0MTcwYTFhNjJhY2YxOSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl8zNjliZTcyM2Y2YjM0NWRlYjhkMzEwNWEzYTY0OTQ0OCA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzUyLjIzNzk4OTEsNS41MzQ2MDczODE2Ml0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFya2VyX2NsdXN0ZXJfYzA3ZWE2N2ZhMDY1NDg0ZDhmODMwYzA3MTM1NjIxMzApOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfNTMyNjQyOTliYjViNGZhOGE0MjJkZDUzMjExNjhhMmQgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGlfZnJhbWVfMzE5OGUwYjFhOTMyNDlhMWIyMDAwZDcxNmQwY2IyZDAgPSAkKCc8aWZyYW1lIHNyYz0iZGF0YTp0ZXh0L2h0bWw7Y2hhcnNldD11dGYtODtiYXNlNjQsQ2lBZ0lDQkxUa2RHWDBaNWMybHZQR0p5UGt0T1IwWmZSbmx6YVc4OFluSStUbVZrWlhKc1lXNWsiIHdpZHRoPSIzMDAiIHN0eWxlPSJib3JkZXI6bm9uZSAhaW1wb3J0YW50OyIgaGVpZ2h0PSIxMDAiPjwvaWZyYW1lPicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNTMyNjQyOTliYjViNGZhOGE0MjJkZDUzMjExNjhhMmQuc2V0Q29udGVudChpX2ZyYW1lXzMxOThlMGIxYTkzMjQ5YTFiMjAwMGQ3MTZkMGNiMmQwKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfMzY5YmU3MjNmNmIzNDVkZWI4ZDMxMDVhM2E2NDk0NDguYmluZFBvcHVwKHBvcHVwXzUzMjY0Mjk5YmI1YjRmYThhNDIyZGQ1MzIxMTY4YTJkKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBsYXllcl9jb250cm9sX2YzYmJlNzI4M2VjNzQ3NzA4N2Y3ODM0MTJkM2Y4ZDhiID0gewogICAgICAgICAgICAgICAgYmFzZV9sYXllcnMgOiB7ICJvcGVuc3RyZWV0bWFwIiA6IHRpbGVfbGF5ZXJfNWU3YzUzZmRlM2RlNDVjOGJjNjg4ZTQwYTlhZDYxZTEsIH0sCiAgICAgICAgICAgICAgICBvdmVybGF5cyA6IHsgInB0X2x5ciIgOiBmZWF0dXJlX2dyb3VwXzBhZjFkNjE1MTE1YjRmZDM4YmE3YTZlZmM4OThkNTNkLCB9CiAgICAgICAgICAgICAgICB9OwogICAgICAgICAgICBMLmNvbnRyb2wubGF5ZXJzKAogICAgICAgICAgICAgICAgbGF5ZXJfY29udHJvbF9mM2JiZTcyODNlYzc0NzcwODdmNzgzNDEyZDNmOGQ4Yi5iYXNlX2xheWVycywKICAgICAgICAgICAgICAgIGxheWVyX2NvbnRyb2xfZjNiYmU3MjgzZWM3NDc3MDg3Zjc4MzQxMmQzZjhkOGIub3ZlcmxheXMsCiAgICAgICAgICAgICAgICB7cG9zaXRpb246ICd0b3ByaWdodCcsCiAgICAgICAgICAgICAgICAgY29sbGFwc2VkOiB0cnVlLAogICAgICAgICAgICAgICAgIGF1dG9aSW5kZXg6IHRydWUKICAgICAgICAgICAgICAgIH0pLmFkZFRvKG1hcF85NzQyNDkwYWFkOGI0ODI3YTg4NWYwNDM5MGNiMzllNik7CiAgICAgICAgCjwvc2NyaXB0Pg==" style="position:absolute;width:100%;height:100%;left:0;top:0;border:none !important;" allowfullscreen webkitallowfullscreen mozallowfullscreen></iframe></div></div>



Done!
