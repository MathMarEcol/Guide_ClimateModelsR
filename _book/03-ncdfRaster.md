# netCDF files in R: Raster, Spatial objects

## Introduction

The aim of this tutorial is to provide a worked example (i.e., a function) of how to transform a regridded **netCDF** into a Raster object using R.

## Data import

Load the required packages.


```r
# load packages
library(raster)
library(ncdf4)
library(ncdf4.helpers)
library(PCICt)
```

## Function to transform netCDF files into Raster objects

You can read netCDF using `raster:::stack` or the `raster:::terra` functions from the `raster` and `terra` packages. However, this function allows more control over the outputs.


```r
# This code was written by Isaac Brito-Morales (i.britomorales@uq.edu.au)
# Please do not distribute this code without permission.
# NO GUARANTEES THAT CODE IS CORRECT
# Caveat Emptor!
 
  ncdf_2D_rs <- function(nc, from, to, v = "tos", x = "lon", y = "lat") {
  # Extract data from the netCDF file  
   nc <- nc_open(nc)
   dat <- ncvar_get(nc, v) # x, y, year 
   dat[] <- dat
   X <- dim(dat)[1]
   Y <- dim(dat)[2]
   tt <- nc.get.time.series(nc, v = "time", time.dim.name = "time")
   tt <- as.POSIXct(tt)
   tt <- as.Date(tt)
   nc_close(nc)
   rs <- raster(nrow = Y, ncol = X) # Make a raster with the right dims
  # Fix orientation of original data
    drs <- data.frame(coordinates(rs))
  # Create Rasters Stack
    rs_list <- list() # empty list to allocate results
    st <- stack()
    for (i in 1:length(tt)) {
      dt1 <- rasterFromXYZ(cbind(drs, as.vector(dat[,, i])))
      dt1[]<- ifelse(dt1[] <= -2, NA, dt1[]) # for some models that have weird temperatures
      dt1[]<- ifelse(dt1[] >= 40, NA, dt1[]) # for some models that have weird temperatures
      st <- addLayer(st, flip(dt1, 2))
      print(paste0(i, " of ", length(tt)))
    }
    names(st) <- seq(as.Date(paste(from, "1", "1", sep = "/")), 
                     as.Date(paste(to, "12", "1", sep = "/")), by = "month")
    crs(st) <- "+proj=longlat +datum=WGS84 +ellps=WGS84 +towgs84=0,0,0"
    return(st)
    }
```







