FROM jupyter/minimal-notebook:notebook-7.0.0

# Prepare system

USER root

RUN apt -y update && apt -y upgrade && apt -y autoremove

RUN apt -y install build-essential g++ git software-properties-common

# Install geospatial system libraries
# Use the Ubuntugis PPA to get more up-to-date versions
# Also install the Python GDAL bindings

RUN add-apt-repository -y ppa:ubuntugis/ubuntugis-unstable && apt -y update

RUN apt -y install \
  gdal-bin libgdal-dev python3-gdal \
  libgeos-dev \
  libproj-dev proj-bin\
  libudunits2-dev \
  libspatialindex-dev

# Install python packages
# Note we install the packages that link to GDAL/GEOS/PROJ from source
# This is because we don't want to use their wheels that include GDAL/GEOS/PROJ
# Instead we want them to use the system installed versions of GDAL/GEOS/PROJ
# pyproj we even have to install from GitHub to fix Cython compability issues

USER jovyan

RUN python3 -m pip install --upgrade pip

RUN pip install numpy

RUN export GDAL_CONFIG=/usr/bin/gdal-config
RUN pip install fiona --no-binary fiona
RUN pip install rasterio --no-binary rasterio

RUN export GEOS_CONFIG=/usr/bin/geos-config
RUN pip install shapely --no-binary shapely

RUN export PROJ_DIR=/usr/local
RUN pip install git+https://github.com/pyproj4/pyproj.git

RUN pip install rtree
RUN pip install geopandas
RUN pip install matplotlib folium mapclassify

RUN pip install pandas xarray rioxarray geocube geogif
RUN pip install asf_search pystac-client
RUN pip install stackstac
RUN pip install --upgrade 'stackstac[viz]'

# setup for earthengine
RUN     pip install pyasn1 --upgrade
RUN     pip install --upgrade setuptools && \
        pip install google-api-python-client && \
        pip install --upgrade oauth2client && \
        pip install pyCrypto && \
        apt-get install -y libssl-dev
        
RUN     pip install earthengine-api      
RUN     pip install earthengine-api --upgrade

WORKDIR /home/jovyan
