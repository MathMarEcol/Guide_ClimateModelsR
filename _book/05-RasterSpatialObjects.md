# Raster objects and Spatial object

## Introduction

The aim here is to provide a worked example of how intersect rasters with spatial polygon objects using the _sf R_ and _exactextractr R_ packages.

This example used two objects.

* __spatial polygon object__: located at `data/PlanningUnits/PUs_WeddellSea_100km2.shp`. This object contains hexagonal polygons of equal area. It has two columns: integer unique identifiers ("id"), geometry information ("geometry")  
* __raster object__: located at `data/VoCC/tos/voccMag_tos_GFDL-CM4_ssp585_2015-2100.grd_2050-2100_.tif`. This object is a raster _.tif_ object of climate-velocity estimates in the southern ocean.

The objective here is to assign to eash hexagon in the _spatial polygon object_ a climate velocity value

## Data import

First, load the required packages and the data.


```r
# load packages
  library(raster)
  library(sf)
  library(dplyr)
  library(exactextractr)
  library(stringr)
  library(nngeo)
```

## Function to replace NAs with nearest neighbor

This functions helps to replace NAs with nearest neighbor interpolation method.


```r
# Function to replace NAs with nearest neighbor. Function wrtitten by Jason Everett
  fCheckNAs <- function(df, vari) {
    if (sum(is.na(pull(df, !!sym(vari))))>0){ # Check if there are NAs
      gp <- df %>%
        mutate(isna = is.finite(!!sym(vari))) %>%
        group_by(isna) %>%
        group_split()
      
      out_na <- gp[[1]] # DF with NAs
      out_finite <- gp[[2]] # DF without NAs
      
      d <- st_nn(out_na, out_finite) %>% # Get nearest neighbour
        unlist()
      out_na <- out_na %>%
        mutate(!!sym(vari) := pull(out_finite, !!sym(vari))[d])
      df <- rbind(out_finite, out_na)
    }
    return(df)
  }
```

## Raster by spatial polygon object

The aim here is to provide an example of how integrate raster in an sf polygon spatial object


```r
# Reading the spatial polygon object
  pu_region <- st_read("data/PlanningUnits/PUs_WeddellSea_100km2.shp")
  st_crs(pu_region) <- "+proj=laea +lat_0=-90 +lon_0=0 +x_0=0 +y_0=0 +datum=WGS84 +units=m +no_defs"
  
# Reading the raster object
  vocc_file <- raster::raster("data/VoCC/tos/voccMag_tos_GFDL-CM4_ssp585_2015-2100.grd_2050-2100_.tif")
  crs(vocc_file) <- CRS("+proj=longlat +datum=WGS84 +ellps=WGS84 +towgs84=0,0,0")
  weight_rs_vocc <- raster::area(vocc_file)
  vocc_file <- projectRaster(vocc_file, crs = CRS("+proj=laea +lat_0=-90 +lon_0=0 +x_0=0 +y_0=0 +datum=WGS84 +units=m +no_defs"), 
                             method = "ngb", over = FALSE)
  weight_vocc <- projectRaster(weight_rs_vocc, crs = CRS("+proj=laea +lat_0=-90 +lon_0=0 +x_0=0 +y_0=0 +datum=WGS84 +units=m +no_defs"), 
                               method = "ngb", over = FALSE)
  names(vocc_file) <- "layer"
  
# Getting the value by planning unit
  vocc_bypu <- exact_extract(vocc_file, pu_region, "weighted_mean", weights = weight_vocc)
  vocc_shp <- pu_region %>%
    dplyr::mutate(vocc = vocc_bypu) %>%
    dplyr::relocate(cellID, vocc)
# Replace NAs with nearest neighbor
  vocc_sfInt <- fCheckNAs(df = vocc_shp, vari = names(vocc_shp)[2]) %>%
  dplyr::select(-isna)
```

## Spatial polygon object by region of interest

The aim here is to provide a simple script to assign a Longhurst province identifier per planning unit


```r
# Reading the spatial polygon object
  pu_region <- st_read("data/PlanningUnits/PUs_WeddellSea_100km2.shp") %>% 
    st_transform(crs = CRS("+proj=laea +lat_0=-90 +lon_0=0 +x_0=0 +y_0=0 +datum=WGS84 +units=m +no_defs"))
# Reading Longhurst Provinces Shapefile
  bioprovince <- st_read("data/Boundaries/LonghurstProvinces/Longhurst_world_v4_2010.shp") %>% 
    st_transform(crs = CRS("+proj=laea +lat_0=-90 +lon_0=0 +x_0=0 +y_0=0 +datum=WGS84 +units=m +no_defs")) %>% 
    st_make_valid()
# Get the Longhurst Provinces per Planning unit
  nr <- st_nearest_feature(pu_region, bioprovince)
  pu_region <- pu_region %>% 
    dplyr::mutate(province = paste(as.vector(bioprovince$ProvCode[nr]), prov_name, sep = "_")) %>% 
    dplyr::arrange(layer)
```



