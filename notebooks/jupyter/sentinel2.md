# Sentinel-2 data in Python

**OpenGeoHub Summer School 2023**

- Lorena Abad
- 2023-09-01

## Accessing data via STAC API

Libraries needed for this exercise are imported below:


```python
import geogif # render gifs from raster images
import geopandas as gpd # handle geospatial data frames
from IPython.display import Image # visualize URLs
import pandas as pd # data wrangling
import pystac_client # connecting to the STAC API
from rasterio.enums import Resampling # perform resampling operations
import rioxarray # handle spatio-temporal arrays
import shapely # create vector objects
import stackstac # build an on-demand STAC data cube
```

### Querying data with `pystac-client`

[STAC](https://stacspec.org/en) stands for SpatioTemporal Asset Catalog and it is "a common language to describe geospatial information, so it can more easily be worked with, indexed, and discovered". 

[`pystac-client`](https://pystac-client.readthedocs.io/en/stable/quickstart.html#python) allows the querying of a STAC API using Python.

There are several APIs available to query data, you can browse them all in the [STAC catalog index](https://stacindex.org/catalogs). Some of these APIs will require authentication to access the data. We will use the [Earth Search](https://www.element84.com/earth-search/) catalog for this notebook, which allows querying data on Amazon Web Services (AWS). The data we will fetch does not require authentication.


```python
# STAC API URL 
api_url = 'https://earth-search.aws.element84.com/v1'
```

To start fetching data, we will open the client. We can see the collections available for this API:


```python
client = pystac_client.Client.open(api_url)
for collection in client.get_collections():
    print(collection)
```

Let's focus on Sentinel-2 data level 2a. [Here are the different levels that Sentinel-2 has](https://sentinels.copernicus.eu/web/sentinel/user-guides/sentinel-2-msi/processing-levels). Level 2a provides atmospherically corrected data representing surface reflectance.


```python
# collection ID
collection = 'sentinel-2-l2a'
```

Let's define now the spatial and temporal extent for our query. We will query all scenes intersecting the point coordinates and the time range given. 


```python
# coordinates
lon = 16.9
lat = 52.4
# date range
datetime = '2022-05-01/2022-10-01'
point = shapely.Point(lon, lat)
```

And we pass these arguments to our search:


```python
search = client.search(
    collections=[collection],
    intersects=point,
    datetime=datetime,
    # query=["eo:cloud_cover<10"],
)
```


```python
items = search.item_collection()
len(items)
```

We can view our query as a Geopandas data frame for easier readability:


```python
df = gpd.GeoDataFrame.from_features(items.to_dict(), crs="epsg:4326")
df
```

This proves useful for example when we want to visualize the cloud cover of our whole collection:


```python
df["datetime"] = pd.to_datetime(df["datetime"])

ts = df.set_index("datetime").sort_index()["eo:cloud_cover"]
ts.plot(title="eo:cloud-cover")
```

Let's explore the properties of one item. But first let's look for an item index with low cloud cover and low nodata.


```python
df_filt = df.loc[(df['eo:cloud_cover'] <= 2) & (df['s2:nodata_pixel_percentage'] <= 10)]
df_filt
```


```python
item = items[df_filt.index[0]]
item.geometry
```


```python
item.datetime
```


```python
item.properties
```

We can also take a look at the assets for the item. That is which bands are available. 


```python
item.assets.keys()
```

And we can also preview how this scene looks like:


```python
thumbnail = item.assets["thumbnail"].href
Image(url = thumbnail)
```

Let's take a look at one single band:


```python
asset = item.assets["red"]
```


```python
asset.extra_fields
```

And read it with [`rioxarray`](https://corteva.github.io/rioxarray/stable/).


```python
red = rioxarray.open_rasterio(item.assets["red"].href, decode_coords="all")
```

We can now plot the data, in this case a subset to speed up the process. That is achieved with the `isel()` function.


```python
red.isel(x=slice(2000, 4000), y=slice(8000, 10500)).plot(robust=True)
```

What about an RGB representation?


```python
rgb = rioxarray.open_rasterio(item.assets["visual"].href)
```


```python
rgb.isel(x=slice(2000, 4000), y=slice(8000, 10500)).plot.imshow()
```

### Creating a STAC data cube

To work with the STAC items as a data cube we can use the [`stackstac`](https://stackstac.readthedocs.io/en/latest/) package. 

To limit our data cube size we will create it only focused on the Poznan bounding box.


```python
footprint = gpd.read_file("../../data/poznan.geojson")
footprint.total_bounds
```


```python
cube = stackstac.stack(
    items,
    resolution=100,
    bounds_latlon=footprint.total_bounds,
    resampling=Resampling.bilinear
)
cube
```

We can further wrangle this cube by selecting only RGB bands and creating monthly composites. We can achieve this with `xarray` resample. This are all the [time range formats](https://docs.xarray.dev/en/latest/generated/xarray.cftime_range.html) supported. 


```python
rgb = cube.sel(band=["red", "green", "blue"])
monthly = rgb.resample(time="MS").median("time", keep_attrs=True)
```


```python
monthly
```

We will use the `compute()` function from [`dask`](https://docs.dask.org/en/stable/) to read our object in-memory. 


```python
monthly = monthly.compute()
```

This will make plotting tasks faster.


```python
monthly.plot.imshow(
    col="time",
    col_wrap=3,
    rgb="band",
    robust=True,
    size=4,
    add_labels=False,
)
```

Let's take a look now at a smaller area to visualize crops around Poznan.


```python
crops = gpd.read_file("../../data/crops.geojson")
cube = stackstac.stack(
    items,
    resolution=10,
    bounds_latlon=crops.total_bounds,
    resampling=Resampling.bilinear
)
rgb = cube.sel(band=["red", "green", "blue"])
rgb
```

For a quick view, we can generate a GIF of the Sentinel-2 scenes we have available.


```python
gif_crops = geogif.dgif(rgb).compute()
```


```python
gif_crops
```

We can work on derived calculation, for example, we can compute the NDVI per scene


```python
nir, red = cube.sel(band="nir"), cube.sel(band="red")
ndvi = (nir - red) / (nir + red)
```


```python
ndvi
```

Let's create a composite with the maximum NDVI value for the whole collection over time.


```python
ndvi_comp = ndvi.max("time")
```


```python
ndvi_comp
```


```python
ndvi_comp = ndvi_comp.compute()
```


```python
ndvi_comp.plot(vmin=0, vmax=0.8, cmap="YlGn")
```

And finally, let's compute the NDVI anomaly, i.e., how much does each pixel from the composite deviates from the mean of the whole collection.


```python
anomaly = ndvi_comp - ndvi.mean()
```


```python
anomaly = anomaly.compute()
```


```python
anomaly.plot(cmap="PiYG")
```

### Downloading the data

There might be at some point the need to download the scenes that you just queried. To do so you can use the `os` and `urllib` modules that are part of the python standard library as the code snippet below shows. This will download all of the items from your search, so make sure you apply enough filtering so that you don't download data that you don't need.

``` python
download_path = "path/to/dir"

for item in items:
    download_path = os.path.join(item.collection_id, item.id)
    if not os.path.exists(download_path):
        os.makedirs(download_path, exist_ok=True)
    for name, asset in item.assets.items():
        urllib.request.urlretrieve(asset.href, 
                                   os.path.join(download_path, 
                                   os.path.basename(asset.href)))
```

## Exercises

1. Adjust your STAC query accordingly and create a new data cube **grouped by season**. Think about data sizes and play with an AOI of your choice.
2. Compute a time series of NDVI values for one crop parcel of your choice. *Hint:* you can easily create a geojson polygon with https://geojson.io/. Take the temporal grouping of your choice, but what would make sense to compare such vegetation values?

## More resources

This material was based on the Carpentries introduction to [Geospatial RAster and Vector data in Python](https://carpentries-incubator.github.io/geospatial-python/) and the [EGU23 short course on the same topic by Francesco Nattino and Ou Ku](https://github.com/esciencecenter-digital-skills/2023-04-25-ds-geospatial-python-EGU/tree/main).
