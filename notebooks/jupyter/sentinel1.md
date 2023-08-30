# Sentinel-1 data in Python

**OpenGeoHub Summer School 2023**

- Lorena Abad
- 2023-08-31


```python

```

## Querying S1-SLC data

Sentinel-1 data comes at different levels and provides different products. For applications such as measuring deformation due to tectonic or volcanic activity, quantifying ground subsidence or to generate digital elevation models (DEM), [interferometric SAR (InSAR)](https://en.wikipedia.org/wiki/Interferometric_synthetic-aperture_radar) techniques can be used. 

To apply such workflows with Sentinel data, we can use [Sentinel-1 Level 1 Single Look Complex](https://sentinels.copernicus.eu/web/sentinel/technical-guides/sentinel-1-sar/products-algorithms/level-1/single-look-complex/interferometric-wide-swath) products.

![](https://sentinels.copernicus.eu/documents/247904/1824983/Sentinel-1-core-fig-1.jpg)

SM mode is designed to support ERS (European Remote Sensing) and Envisat missions; IW mode is the default mode over land; EW mode is designed for maritime, ice, and polar zone observation services where wide coverage and short revisit times are demanded; and WV mode is the default mode over the open ocean.

So far, very few cloud computing capabilities are available to compute such complex workflows, therefore, there is still a need to download data. Depending on the application, we will need to download data with certain characteristics. 

<img src="https://www.mdpi.com/remotesensing/remotesensing-09-00638/article_deploy/html/images/remotesensing-09-00638-g002.png" alt="InSAR principles" style="width: 500px;"/>

> Figure from: Xiong S, Muller J-P, Li G. The Application of ALOS/PALSAR InSAR to Measure Subsurface Penetration Depths in Deserts. Remote Sensing. 2017; 9(6):638. https://doi.org/10.3390/rs9060638

[Note on terminology](https://earthenable.wordpress.com/2020/08/11/new-insar-terminology-coming-in-vogue-master-slave-to-reference-secondary/).

For DEM generation, for example, we would require a pair of Sentinel-1 scenes acquired closely in time and that have a perpendicular baseline between 150 and 300 m. Usually, computing the perpendicular baseline between two images requires the download of the image pairs. 

To avoid downloading several unnecessary Sentinel-1 scenes, we can make use of the [Alaska Satellite Facility (ASF)](https://search.asf.alaska.edu/) geographic and baseline tools to query the data we need via their API.

Libraries needed for this exercise are imported below:


```python
import asf_search as asf
import geopandas as gpd
import matplotlib.pyplot as plt
import numpy as np
import pandas as pd
```

### Define extent

We will define an aoi and a start and end date for our queries.


```python
aoi = gpd.read_file("../../data/poznan.geojson")
aoi.explore()
```


```python
footprint = aoi.to_wkt()
date_start = "2022/05/01"
date_end = "2022/10/01"
```

### Geographical search

Now we can use the [`asf_search` Python module](https://docs.asf.alaska.edu/asf_search/basics/) to perform our geographical search. We specify here the platform and the processing level (SLC) that we are looking for, and we limit the results for this exercise to 10 scenes.


```python
products = asf.geo_search(platform=[asf.PLATFORM.SENTINEL1],
                          intersectsWith=footprint.geometry[0],
                          processingLevel=[asf.PRODUCT_TYPE.SLC],
                          # beamSwath='IW',
                          start=date_start,
                          end=date_end,
                          maxResults=10)
```

We can then add the results of the query to a pandas dataframe for easier inspection:


```python
products_df = pd.DataFrame([p.properties for p in products])
products_df
```

Ascending vs. Descending:
<img src="https://site.tre-altamira.com/wp-content/uploads/ascending-and-descending-orbits.png" alt="Schema pass" style="width: 250px;"/> - <img src="https://pbs.twimg.com/media/FamfGWGWIAAhMLv?format=jpg&name=900x900" alt="Ascending pass" style="width: 250px;"/> - <img src="https://pbs.twimg.com/media/FamfGsMWYAIVV5v?format=jpg&name=900x900" alt="Descending pass" style="width: 250px;"/>"/>

### Baseline search

Now that we have scenes that intersect with our defined extent, we can do a baseline search that will allow us to fetch all the S1 scenes that pair with the first S1 result from our geographical query. The baseline search returns a set of products with precomputed perpendicular baselines, so that we can focus our download on the data that we need. 


```python
stack = products[0].stack()
```


```python
print(f'{len(stack)} products found in stack')
```

We can take a look at the data again as a pandas data frame and we will see that the last two columns correspond to the temporal and perpendicular baseline. 


```python
stack_df = pd.DataFrame([p.properties for p in stack])
stack_df

# sort_values()
```

To have an idea of how spread our data is, we can plot the temporal and the perpendicular baselines against each other.


```python
stack_df.plot.scatter(x="temporalBaseline", y="perpendicularBaseline")
```

Ideally, we will filter those values where `temporalBaseline <= 30` and `150 <= perpendicularBaseline <= 300` for instance to get image pairs suitable for DEM generation. So we can filter our data frame for those values. We look for absolute values since the order of the images is not relevant.


```python
stack_df[(abs(stack_df['temporalBaseline']) <= 30) &
         (abs(stack_df['perpendicularBaseline']) >= 150) &
         (abs(stack_df['perpendicularBaseline']) <= 300)]
```

We only get one image fitting the characteristics we require. Let's look at its properties:


```python
stack[416].properties
```

Let's also remember this is paired with the original product we calcualted the baselines for.


```python
products[0].properties
```

### Downloading the data

Finally, with the ASF API we can download our data to further analyse it with, e.g. SNAP. To do so we can make use of the url property.


```python
urls = [
    products[0].properties['url'],
    stack[416].properties['url']
]
```

Once that is set we can use the `download_urls()` function as speccified below to get our data in a desired directory. To download the data we will need [EarthData credentials](https://urs.earthdata.nasa.gov/). This [notebook from the ASF](https://github.com/asfadmin/Discovery-asf_search/blob/master/examples/5-Download.ipynb) describes the authentication process. 

```python
asf.download_urls(urls=urls, path='data/s1', session=user_pass_session, processes=5)
```

## Exploring S1-RTC data


```python
import matplotlib.pyplot as plt
import numpy as np
import os
import pandas as pd
import rioxarray as rio
import xarray as xr
```

Now let's take a look at a bit more processed data that we can directly work with. Still in Level-1 you will see the [Ground Range Detected (GRD) product](https://sentinels.copernicus.eu/web/sentinel/technical-guides/sentinel-1-sar/products-algorithms/level-1-algorithms/ground-range-detected) in the figure above. This is S1 data that has been further processed (it has been detected, multi-looked and projected to ground range). The SLC products we queried before preserve phase information and are processed at the natural pixel spacing whereas GRD products contain the detected amplitude and are multi-looked to reduce the impact of speckle.

An extra processing step is to perform [Radiometric Terrain Correction](https://planetarycomputer.microsoft.com/dataset/sentinel-1-rtc), and some data providers like Microsoft Planetary Computer make this dataset available worldwide. Feel free to explore the Planetary Computer access options to work on larger datasets if you are interested. 

In the spirit to avoid the need for you to get credentials for this particular workshop, we will use a [Sentinel-1 RTC dataset for the Contiguous United States (CONUS)](https://registry.opendata.aws/sentinel-1-rtc-indigo/) which is freely accessible. 

We will access this data using the Amazon Web Services (AWS) CLI directly (with the `awscli` package). Let's explore the available data:


```python
!aws s3 ls s3://sentinel-s1-rtc-indigo/ --no-sign-request
```

To download a scene we can directly request the data as:


```python
# Run only if you want to have 110MB of random data on your laptop!
!aws s3 cp s3://sentinel-s1-rtc-indigo/tiles/RTC/1/IW/12/R/UV/2021/S1B_20210121_12RUV_DSC/Gamma0_VV.tif S1B_20210121_12RUV_DSC/Gamma0_VV.tif --no-sign-request
```

The available bands have the prefix `Gamma0`. This is the result of the RTC algorithm. Read more about [the backscatter types here](https://hyp3-docs.asf.alaska.edu/guides/rtc_product_guide/#radiometry).

We will also see that the data has a suffix, either VV or VH, this is the polarization. That refers to the way data is collected. 

<img src="https://hyp3-docs.asf.alaska.edu/images/polarizations_ASF_dashed.png" alt="Polarization" style="width: 350px;"/> <img src="https://pbs.twimg.com/media/FanwpCPXwAA5rnP?format=webp&name=900x900" alt="HH-VV" style="width: 400px;"/>

[Read more about it here](https://sentinels.copernicus.eu/web/sentinel/user-guides/sentinel-1-sar/product-overview/polarimetry), [here](https://learningzone.rspsoc.org.uk/index.php/Learning-Materials/Radar-Imaging/Image-Interpretation-Polarisation) and [here](https://hyp3-docs.asf.alaska.edu/guides/introduction_to_sar/).

Let's start exploring the data. For this we will use `rioxarray`. We will set an environment key to establish no sign request for AWS. And we will also be leveraging the tight integration between xarray and dask to lazily read in data via the `chunks` parameter. 

We will get both the `vv` and `vh` data.


```python
os.environ['AWS_NO_SIGN_REQUEST'] = 'YES'
```


```python
url_vv = 's3://sentinel-s1-rtc-indigo/tiles/RTC/1/IW/12/R/UV/2021/S1B_20210121_12RUV_DSC/Gamma0_VV.tif'
url_vh = 's3://sentinel-s1-rtc-indigo/tiles/RTC/1/IW/12/R/UV/2021/S1B_20210121_12RUV_DSC/Gamma0_VV.tif'
s1_vv = rio.open_rasterio(url_vv, chunks=True)
s1_vh = rio.open_rasterio(url_vh, chunks=True)
```


```python
s1_vv
```

Let's visualize a slice of the data:


```python
s1_vv_ss = s1_vv.isel(x=slice(1000, 1500), y=slice(1000, 1500)).compute()
```


```python
s1_vv_ss.plot(cmap=plt.cm.Greys_r)
```

To better visualize the data, we can apply a power to dB scale. This transformation applies a logarithmic scale to the data for easier visualisation, but it is not recommended to use this for any computations, since the data gets distorted. 


```python
def power_to_db(input_arr):
    return (10*np.log10(np.abs(input_arr)))
```


```python
power_to_db(s1_vv_ss).plot(cmap=plt.cm.Greys_r)
```

## Exercise:

- Try to combine the `s1_vv` and `s1_vh` objects together, compute a new band with the result of `VH/VV` and use these three layers to generate a false color RGB composite. 

## More resources:

Working with Sentinel-1 SLC data can also be done with Python. There are a couple of packages available for this (`snappy` and [`snapista`](https://snap-contrib.github.io/snapista/)), but ESA is currently working on a follow up of the `snappy` package called `esa-snappy` which will be compatible with the upcoming SNAP-10. Since the [developers claim its worth the wait](https://forum.step.esa.int/t/snappy-and-snap-10-release/39606), I would at this point direct you to the webpage where they seem to be documenting basic usage of the tool. So for that [feel free to check this site more or less at the end of August](https://senbox.atlassian.net/wiki/spaces/SNAP/pages/2499051521/Configure+Python+to+use+the+new+SNAP-Python+esa+snappy+interface+SNAP+version+10).

A lot of the examples for this notebook, mainly for the RTC processing were adapted from Emma Marshall's excellent tutorial on [Sentinel-1 RTC data workflows with xarray](https://e-marshall.github.io/sentinel1_rtc/intro.html).
