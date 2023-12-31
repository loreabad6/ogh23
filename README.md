# Tools and packages to query and process Sentinel-1 and Sentinel-2 data with R and Python

## [OpenGeoHub Summer School Poznan 2023](https://opengeohub.org/summer-school/opengeohub-summer-school-poznan-2023/)

Intro slides [here](https://loreabad6.github.io/ogh23/intro.html).

Wednesday, Aug. 31th & Sept. 1st 2023

## Part 1: Thu. 31.08. - 13h30-15h00
During this session we will take a look at Sentinel-1 data, specifically tools to query data for subsequent large processing and some basic data analysis in Python.

You can browse the material [here](https://loreabad6.github.io/ogh23/notebooks/jupyter/sentinel1.html).

## Part 2: Fri. 01.09. - 09h00-10h30
On Friday, we will focus on Sentinel-2 data and the STAC API going back and forth between R and Python environments. 

You can browse the [R material here](https://loreabad6.github.io/ogh23/notebooks/quarto/sentinel2.html) and the [Python material here](https://loreabad6.github.io/ogh23/notebooks/jupyter/sentinel2.html).

## Set-up

### Docker
Since installations of R, but mostly Python, can be complicated, I recommend using Docker for this tutorial. To use them, please download and install Docker Engine/Desktop:

- Linux: https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository
- Windows: https://docs.docker.com/docker-for-windows/install/
- Mac: https://docs.docker.com/docker-for-mac/install/

Once you have installed this, start the docker engine. 

There are two Dockerfiles for this tutorial.

#### Python environment: 

We will work on a minimal Jupyter notebook image enhanced with some geospatial flavor and the specific requirements for this lesson.

1. Go to your terminal and navigate to your desired directory. 

2. Clone this repository.
```
git clone https://github.com/loreabad6/ogh23.git
```

3. Build the Docker image:
```
cd dockerfiles/pyenv
docker build -t pyenv . 
```

4. Run the Docker container. Note that `--rm` will remove the container when you quit. 
```
cd ../..
docker run --name ogh-pyenv --rm -it -p 8888:8888 -v $PWD/:/home/jovyan pyenv 
```

5. If successful, you will see a link to the Jupyter notebook printed on the console. Copy that link in your browser and you are ready to go!

To stop the notebook environment go back to the terminal and `CTRL+C`.

#### R environment:

For the R environment we will run a RStudio cloud instance. Given that you already followed steps 1 and 2 for the Python environment:

1. Build the Docker image:
```
cd dockerfiles/renv
docker build -t renv . 
```

2. Run the Docker container
```
cd ../..
docker run --name ogh-renv --rm -e DISABLE_AUTH=TRUE -e USERID=$UID -p 8786:8787 -v $PWD/:/home/rstudio/ogh23/ renv
```

3. Go to your browser and type: http://localhost:8786/. Now you have an RStudio interface on your browser!

To stop the RStudio environment go back to the terminal and `CTRL+C`.

Since we are going back and forth with the tutorials, it would be good if you can run both containers at the same time. In simple terms this means you can execute docker run in two different command line windows and then have Jupyter Notebooks and RStudio cloud open in two tabs. Please test that this works for you before the tutorial. 

### Alternatives

If you want to work on your own local machine and do the manual installations, refer to the Dockerfiles for the specific requirements for the R and Python environments. 
