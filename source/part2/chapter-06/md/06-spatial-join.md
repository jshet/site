---
jupyter:
  jupytext:
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.3'
      jupytext_version: 1.15.2
  kernelspec:
    display_name: Python 3 (ipykernel)
    language: python
    name: python3
---

# Spatial join

Spatial join is yet another classic GIS problem. Getting attributes from one layer and transferring them into another layer based on their spatial relationship is something you most likely need to do on a regular basis. In the previous section (Chapter 6.6), we learned how to perform spatial queries, such as investigating if a Point is located within a Polygon. We can use this same logic to conduct a spatial join between two layers based on their spatial relationship and transfer the information stored in one layer into the other. We could, for example, join the attributes of a polygon layer into a point layer where each point would get the attributes of a polygon that `intersects` with the point. **Figure 6.xx** illustrates this logic by showing how we can combine information from two layers (Land use and Buildings) into the point layer that represents the locations of four properties. Each layer has their own attribute information shown on the right. The table at the bottom of the figure shows the result after both of these layers have been combined with the properties layer. The logic for merging the information between layers is based on how the individual points in the Properties layer intersect with different land use areas (commercial, residential, industrial, or natural) and the building footprints. Spatial join is always conducted between two layers at a time. Hence, in practice, to be able to make a spatial join between three layers, we first need to make a spatial join between Properties and Land use and store this into an intermediate result. After the first join, you need to make another spatial join with this intermediate result and the third layer, namely the Buildings dataset, which in our case will be the final result shown at the bottom. As a result, we now have enriched our data with many additional attributes from the other layers that are stored as separate columns in the table. In a similar manner, you could also continue joining data (attributes) from other layers as long as you need.  

![_**Figure 6.XX**. Spatial join allows you to combine attribute information from multiple layers based on spatial relationship._](../img/spatial-join-basic-idea.png)

_**Figure 6.XX**. Spatial join allows you to combine attribute information from multiple layers based on spatial relationship._


<!-- #region -->
![_**Figure 6.XX**. Different approaches to join two data layers with each other based on spatial relationships._](../img/spatial-join-alternatives.png)

_**Figure 6.XX**. Different approaches to join two data layers with each other based on spatial relationships._


Luckily, [spatial join is already implemented in Geopandas](http://geopandas.org/mergingdata.html#spatial-joins), thus we do not need to create our own function for doing it. There are three possible types of join that can be applied in spatial join that are determined with ``op`` -parameter in the ``gpd.sjoin()`` -function:

-  ``"intersects"``
-  ``"within"``
-  ``"contains"``

Sounds familiar? Yep, all of those spatial relationships were discussed in the [Point in Polygon lesson](point-in-polygon.ipynb), thus you should know how they work. 

Furthermore, pay attention to the different options for the type of join via the `how` parameter; "left", "right" and "inner". You can read more about these options in the [geopandas sjoin documentation](http://geopandas.org/mergingdata.html#sjoin-arguments) and pandas guide for [merge, join and concatenate](https://pandas.pydata.org/pandas-docs/stable/user_guide/merging.html).

Let's perform a spatial join between these two layers:
- **Addresses:** the geocoded address-point (we created this Shapefile in the geocoding tutorial)
- **Population grid:** 250m x 250m grid polygon layer that contains population information from the Helsinki Region.
    - The population grid a dataset is produced by the **Helsinki Region Environmental
Services Authority (HSY)** (see [this page](https://www.hsy.fi/fi/asiantuntijalle/avoindata/Sivut/AvoinData.aspx?dataID=7) to access data from different years).
    - You can download the data from [from this link](https://www.hsy.fi/sites/AvoinData/AvoinData/SYT/Tietoyhteistyoyksikko/Shape%20(Esri)/V%C3%A4est%C3%B6tietoruudukko/Vaestotietoruudukko_2018_SHP.zip) in the  [Helsinki Region Infroshare
(HRI) open data portal](https://hri.fi/en_gb/).

<!-- #endregion -->

- Here, we will access the data directly from the HSY wfs:

<!-- #endregion -->

```python
import geopandas as gpd
from pyproj import CRS
import requests
import geojson

# Specify the url for web feature service
url = "https://kartta.hsy.fi/geoserver/wfs"

# Specify parameters (read data in json format).
# Available feature types in this particular data source: http://geo.stat.fi/geoserver/vaestoruutu/wfs?service=wfs&version=2.0.0&request=describeFeatureType
params = dict(
    service="WFS",
    version="2.0.0",
    request="GetFeature",
    typeName="asuminen_ja_maankaytto:Vaestotietoruudukko_2018",
    outputFormat="json",
)

# Fetch data from WFS using requests
r = requests.get(url, params=params)

# Create GeoDataFrame from geojson
pop = gpd.GeoDataFrame.from_features(geojson.loads(r.content))
```

Check the result: 

```python
pop.head()
```

Okey so we have multiple columns in the dataset but the most important
one here is the column `asukkaita` ("population" in Finnish) that
tells the amount of inhabitants living under that polygon.

-  Let's change the name of that column into `pop18` so that it is
   more intuitive. As you might remember, we can easily rename (Geo)DataFrame column names using the ``rename()`` function where we pass a dictionary of new column names like this: ``columns={'oldname': 'newname'}``.

```python
# Change the name of a column
pop = pop.rename(columns={"asukkaita": "pop18"})

# Check the column names
pop.columns
```

Let's also get rid of all unnecessary columns by selecting only columns that we need i.e. ``pop18`` and ``geometry``

```python
# Subset columns
pop = pop[["pop18", "geometry"]]
```

```python
pop.head()
```

Now we have cleaned the data and have only those columns that we need
for our analysis.


## Join the layers

Now we are ready to perform the spatial join between the two layers that
we have. The aim here is to get information about **how many people live
in a polygon that contains an individual address-point** . Thus, we want
to join attributes from the population layer we just modified into the
addresses point layer ``addresses.shp`` that we created trough gecoding in the previous section.

-  Read the addresses layer into memory:

```python
# Addresses filpath
addr_fp = "data/Helsinki/addresses.shp"

# Read data
addresses = gpd.read_file(addr_fp)
```

```python
# Check the head of the file
addresses.head()
```

In order to do a spatial join, the layers need to be in the same projection

- Check the crs of input layers:

```python
addresses.crs
```

```python
pop.crs
```

If the crs information is missing from the population grid, we can **define the coordinate reference system** as **ETRS GK-25 (EPSG:3879)** because we know what it is based on the [population grid metadata](https://hri.fi/data/dataset/vaestotietoruudukko). 

```python
# Define crs
pop.crs = CRS.from_epsg(3879).to_wkt()
```

```python
pop.crs
```

```python
# Are the layers in the same projection?
addresses.crs == pop.crs
```

Let's re-project addresses to the projection of the population layer:

```python
addresses = addresses.to_crs(pop.crs)
```

-  Let's make sure that the coordinate reference system of the layers
are identical

```python
# Check the crs of address points
print(addresses.crs)

# Check the crs of population layer
print(pop.crs)

# Do they match now?
addresses.crs == pop.crs
```

Now they should be identical. Thus, we can be sure that when doing spatial
queries between layers the locations match and we get the right results
e.g. from the spatial join that we are conducting here.

-  Let's now join the attributes from ``pop`` GeoDataFrame into
   ``addresses`` GeoDataFrame by using ``gpd.sjoin()`` -function:

```python
# Make a spatial join
join = gpd.sjoin(addresses, pop, how="inner", predicate="within")
```

```python
join.head()
```

Awesome! Now we have performed a successful spatial join where we got
two new columns into our ``join`` GeoDataFrame, i.e. ``index_right``
that tells the index of the matching polygon in the population grid and
``pop18`` which is the population in the cell where the address-point is
located.

- Let's still check how many rows of data we have now:

```python
len(join)
```

Did we lose some data here? 

- Check how many addresses we had originally:

```python
len(addresses)
```

If we plot the layers on top of each other, we can observe that some of the points are located outside the populated grid squares (increase figure size if you can't see this properly!)

```python
import matplotlib.pyplot as plt

# Create a figure with one subplot
fig, ax = plt.subplots(figsize=(15, 8))

# Plot population grid
pop.plot(ax=ax)

# Plot points
addresses.plot(ax=ax, color="red", markersize=5)
```

_**Figure 6.34**. ADD PROPER FIGURE CAPTION!._

Let's also visualize the joined output:


Plot the points and use the ``pop18`` column to indicate the color.
   ``cmap`` -parameter tells to use a sequential colormap for the
   values, ``markersize`` adjusts the size of a point, ``scheme`` parameter can be used to adjust the classification method based on [pysal](http://pysal.readthedocs.io/en/latest/library/esda/mapclassify.html), and ``legend`` tells that we want to have a legend:


```python
# Create a figure with one subplot
fig, ax = plt.subplots(figsize=(10, 6))

# Plot the points with population info
join.plot(
    ax=ax, column="pop18", cmap="Reds", markersize=15, scheme="quantiles", legend=True
)

# Add title
plt.title("Amount of inhabitants living close the the point")

# Remove white space around the figure
plt.tight_layout()
```

_**Figure 6.35**. ADD PROPER FIGURE CAPTION!._

In a similar way, we can plot the original population grid and check the overall population distribution in Helsinki:

```python
# Create a figure with one subplot
fig, ax = plt.subplots(figsize=(10, 6))

# Plot the grid with population info
pop.plot(ax=ax, column="pop18", cmap="Reds", scheme="quantiles", legend=True)

# Add title
plt.title("Population 2018 in 250 x 250 m grid squares")

# Remove white space around the figure
plt.tight_layout()
```

_**Figure 6.36**. ADD PROPER FIGURE CAPTION!._

Finally, let's save the result point layer into a file:

```python
# Output path
outfp = "data/Helsinki/addresses_population.shp"

# Save to disk
join.to_file(outfp)
```

## Spatial join nearest

ADD Materials