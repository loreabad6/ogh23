FROM rocker/geospatial:ubuntugis

RUN R -e "devtools::install_github('r-spatial/sf')"
RUN R -e "devtools::install_github('r-spatial/lwgeom')"

RUN install2.r --error \
	gdalcubes \
	here \
	kableExtra \
	osmdata \
	rstac
	