---
title: "5. Getting Spatial with sftrack"
output:
  pdf_document: default
html_document: default
vignette: |
  %\VignetteIndexEntry{5. Getting Spatial with sftrack}
   %\VignetteEncoding{UTF-8}
   %\VignetteEngine{knitr::rmarkdown}
---
```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
devtools::load_all("/home/matt/r_programs/sftrack")
#library(sftrack)

```

```{r}
# Make tracks from raw data
data <- read.csv(system.file('extdata/raccoon_data.csv', package='sftrack'))
data$month <- as.POSIXlt(data$acquisition_time)$mon+1

data$time <- as.POSIXct(data$acquisition_time, tz='EST')
coords = c('longitude','latitude')
burst = list(id = data$sensor_code, month = as.POSIXlt(data$acquisition_time)$mon+1)
time = 'time'
error = 'fix'
crs = '+init=epsg:4326'
my_sftrack <- as_sftrack(data = data, coords = coords, burst = burst, time = time, error = error, crs = crs)
my_sftraj <- as_sftraj(data = data, coords = coords, burst = burst, time = time, error = error, crs = crs)
```
## Geometry column  

As stated earlier, the geometry column is built using `sf`, so functions exactly as it would in `sf`. You can modify it and redefine it using the `sf` tools. More specifically the geometry column of an sf_track object is an `sfc` column. The main difference between a standard `sf` object created using `st_as_sf` is that we automatically allow empty geometries, where as this option is turned off by default in `st_as_sf()`.  
```{r}
my_sftrack$geometry
```

An `sftrack` object is simply an `sfc` of `sfc_POINTS`, this contrasts with an `sftraj` object which is a mixture of a `POINT` and `LINESTRING`. This is because a trajectory can have a start point and an NA end point, a line segment, or an NA and an end point. This allows no-loss conversion back and forth between `sftrack` and an `sftraj`, and because linestrings can not have a NULL point in them.

```{r}
my_sftraj$geometry
```

This does mean that not all `sf` functions will handle an `sftraj` object like it would an `sftrack` if there are NAs in the data set. To help with working with more complex sftraj objects, there is a growing suite of `sftraj` specific functions:

##### coord_traj 
This function returns a data.frame (x,y,z) of the point at t1 of each sftraj geometry. It works nearly identically to `sf::st_coordinates()`.

```{r}
coord_traj(my_sftraj$geometry)[1:10,]
```

##### pts_traj 
If youd like to retain the geometries but still pull out t1 point you can use `pts_traj()`. This functions returns a list of the beginning point of each sftraj geometry, or an sfc column when using the argument `sfc = TRUE`.

```{r}
pts_traj(my_sftraj$geometry)[1:5]

pts_traj(my_sftraj$geometry, sfc= TRUE)[1:5]
```

##### is_linestring 
May help if you'd like to quickly filter an `sftraj` object to just contain pure linestrings. `is_linestring()` returns TRUE or FALSE if the geometry is a linestring. This does not recalculate anything, it just filters out steps that contained NAs in either phase. Its nearly identical to st_is(x,'LINESTRING'), but may be more intuitive for users.

```{r}
is_linestring(my_sftraj$geometry)[1:10]
new_sftraj <- my_sftraj[is_linestring(my_sftraj$geometry),]
head(new_sftraj)
```

#### Calculating step metrics of an sftraj
For use in movement models, you may need to calculate the dx, dy, length, and turn angles of each step. You can do that in `sftrack` using `step_metrics()`. It should be noted it will accept an `sftrack` object, however, it first converts the geometries internally to `sftraj` geometries and then calculates step metrics. As with other `sf` objects, the return is assumed to be in the units of the `crs` when not specified. Absolute angle is measured in radians.

```{r}
step_metrics(my_sftraj[1:10,])
```

## Working with sf

As sftrack object is an sf object, all of the sf functions apply to it.

```{r}
st_length(my_sftraj)[1:10]

df1 <- data.frame(
  id = c(1,1,1,1),
  time = as.POSIXct('2020-01-01 12:00:00', tz = 'UTC') + 60*60*(1:4),
  x = c(1,3,3,2),
  y = c(1,1,3,4)
)

road <- st_linestring(rbind(
  c(1,2),
  c(5,2),
  c(5,0)
)
)

animal1 <- as_sftraj(df1)
plot(animal1)

plot(road, add = T, col = 'red')

# Does the animal cross the road?

any(st_intersects(animal1, road, sparse = F))

# When?
animal1$time[st_intersects(animal1, road, sparse = F)]

# How often does the animal stay near the road?
st_is_within_distance(animal1, road, 1)

# How close is the animal from the road?

st_distance(animal1, road)
```

The only thing to remember, is that a sftraj is a `GEOMETRY` column, and occasionally a function may not work with it. In those cases is_linestring would be appropriate to filter out points with no t2.

## Plotting

##### Base plotting 
Currently there are some basic plotting methods. Base plotting is built ontop of the sf functionality, while simply controlling how to group and color the points. 


```{r}
plot(my_sftraj)
```

And changing the active burst will change the plot view

```{r}
active_burst(my_sftraj$burst) <- 'id'
active_burst(my_sftraj$burst)
plot(my_sftraj)
```

Because it feeds plot an sf object, the same arguments for `plot.sf` are all available. 

```{r}
plot(my_sftrack, axes = TRUE, cex=5)
```


##### ggplot
This is a work in progress, but theres a rudimentary geom_sftrack function. As of now you have to input `data` into the geom_sftrack function and not into `ggplot()`.  Again ggplot assumes active_burst is the grouping variable. Plots vary slightly based on if they're track of traj

```{r}
library(ggplot2)
ggplot() + geom_sftrack(data = my_sftraj)
```


