# Getting Started with Climate Data Operators (CDO)

**CMIP6** models come in netCDF file format and they are usually really messy to work in R. For example, the resolution of the **CMIP6** NOAA model (SSP1-2.6) is 0.25°. Let say that you want to download that model for the **thetao** variable (sea temperature with depth). The size of that model is ~80gb. It will be really hard just to read that file in R. 

The intention with this information and scripts is to provide a basic understanding of how you can use **CDO** to speed-up your **netCDF** file data manipulation. [More info go directly to the Max Planck Institute CDO website](https://code.mpimet.mpg.de/projects/cdo/)


## Installation Process
### **MacOS**

Follow the instruction and downloaded **MacPorts**. **MacPorts** is an open-source community initiative to design an easy-to-use system for compiling, installing, and upgrading the command-line on the Mac operating system. 

[MacPorts website](https://www.macports.org/index.php)
[MacPorts download](https://www.macports.org/install.php)

After the installation (if you have admin rights) open the terminal and type:

  `port install cdo`

If you don't have admin rights, open the terminal and type:

  `sudo port install cdo` and write your password

### **Windows 10**

In the current windows 10 version(s) Microsoft includes an Ubuntu 16.04 LTS embedded Linux. This environment offers a clean integration with the windows file systems and and the opportunity to install CDO via the native package manager of Ubuntu.

Install the Ubuntu app from the Microsoft Store application. Then open the Ubuntu terminal and type:

  `sudo apt-get install cdo` and write your password

## **Linux**

For Linux go to: [Linux](https://code.mpimet.mpg.de/projects/cdo/wiki/Linux_Platform)

## Ncview: a netCDF visual browser

Ncview is quick visual browser that allows you to explore **netCDF** files very easily: `ncview`. `ncview` is an easy to use netCDF file viewer for **linux** and **OS X**. It can read any netCDF file.

To install **ncview**, open the terminal and type:

* __OS X__: `port install ncview`
* __Linux__: `sudo apt-get install ncview`

## Working with CDO

To work with **CDO** and **ncview** we need to use the terminal command line: the **Ubuntu app** in Windows and the **Terminal** on OS X.


```r
# Establish the primary directory (OS or Linux)
## cd ~/data/ClimateModels/tos/ssp585/

# Establish the primary directory in Windows. It should be located at "/mnt/c/"
## cd ~/data/ClimateModels/tos/ssp585/

# View the model with ncview
## ncview tos_Omon_GFDL-CM4_ssp585_r1i1p1f1_gr_201501-203412.nc
```

<img src="images/ncview01.png" width="100%" />

ncview example


```r
# Check the details by typing in the terminal
## cdo -sinfov tos_Omon_GFDL-CM4_ssp585_r1i1p1f1_gr_201501-203412.nc
```

<img src="images/ncview02.png" width="100%" />

Model details

### Regrid process

To regrid a **netCDF** file using **CDO** we need to use the argument `remapbil`, which is the stands for bilinear interpolation (there are other methods but this is the most conservative approach). CDO Syntax works like this:


```r
# Type in the terminal
## cdo remapbil,r360x180 tos_Omon_GFDL-CM4_ssp585_r1i1p1f1_gr_201501-203412.nc test.nc
```

The previous line will create an uniform file at 1deg of spatial resolution. It works fine for one or two **netCDF** files. However, for multiple **netCDF** files/models (which is most cases of CMIP6 models) the best way to auto the process is to write an *R* script that calls CDO through the **system()** function.

This function will help to search for ANY **netCDF** file in a particular directory and do the "regridding process".

* __Key CDO function__: `remapbil`



```r

# This code was written by Isaac Brito-Morales (i.britomorales@uq.edu.au)
# Please do not distribute this code without permission.
# NO GUARANTEES THAT CODE IS CORRECT
# Caveat Emptor!

# Function's arguments
# ipath: directory where the netCDF files are located
# opath: directory to allocate the new regrided netCDF files
# resolution = resolution for the regrid process

regrid <- function(ipath, opath, resolution) {

####################################################################################
####### Defining the main packages
####################################################################################
  # List of pacakges that we will be used
    list.of.packages <- c("doParallel", "parallel", "stringr", "data.table")
  # If is not installed, install the pacakge
    new.packages <- list.of.packages[!(list.of.packages %in% installed.packages()[,"Package"])]
    if(length(new.packages)) install.packages(new.packages)
  # Load packages
    lapply(list.of.packages, require, character.only = TRUE)
  
####################################################################################
####### Getting the path and directories for the files
####################################################################################
  # Establish the find bash command
    line1 <- paste(noquote("find"), noquote(ipath), "-type", "f", "-name", 
                   noquote("*.nc"), "-exec", "ls", "-l", "{}")
    line2 <- paste0("\\", ";")
    line3 <- paste(line1, line2)
  # Getting a list of directories for every netCDF file
    dir_files <- system(line3, intern = TRUE)
    dir_nc <- strsplit(x = dir_files, split = " ")
    nc_list <- lapply(dir_nc, function(x){f1 <- tail(x, n = 1)})
  # Cleaning the directories to get a final vector of directories
    final_nc <- lapply(nc_list, function(x) {
      c1 <- str_split(unlist(x), pattern = "//")
      c2 <- paste(c1[[1]][1], c1[[1]][2], sep = "/")})
    files.nc <- unlist(final_nc)
 
####################################################################################
####### Starting the regrid process
#################################################################################### 
  # Resolution
    if(resolution == "1") {
      grd <- "r360x180"
    } else if(resolution == "0.5") {
      grd <- "r720x360"
    } else if(resolution == "0.25") {
      grd <- "r1440x720"
    }
    
  # Parallel looop
    UseCores <- 3 # we can change this number
    cl <- makeCluster(UseCores)  
    registerDoParallel(cl)
    foreach(j = 1:length(files.nc), .packages = c("stringr")) %dopar% {
      # Trying to auto the name for every model
        var_obj <- system(paste("cdo -showname", files.nc[j]), intern = TRUE)
        var_all <- str_replace_all(string = var_obj, pattern = " ", replacement = "_")
        var <- tail(unlist(strsplit(var_all, split = "_")), n = 1) 
      # Running CDO regrid
        system(paste(paste("cdo -remapbil,", grd, ",", sep = ""), 
                     paste("-selname",var, sep = ","), files.nc[j], 
                     paste0(opath, basename(files.nc[j])), sep = (" "))) # -P 2
    }
    stopCluster(cl)
}

```

Run the regrid R function


```r
regrid(ipath = "/data/ClimateModels/",
       opath = "/data/ClimateModelsRegrid/",
       resolution = "0.5")
```



### Models with different ocean depths

Some climate models are build across different depths in the ocean. This function will help to search for ANY **netCDF** file in a particular directory, split the file by different ocean depth layer (e.g., surface, epipelagic, mesopelagic, bathypelagic), and merge those files by estimating a vertical average condition.

* __CDO functions__: `showname` `sellevel` `selname`
* __Key CDO function__: `vertmean`


```r
# This code was written by Isaac Brito-Morales (i.britomorales@uq.edu.au)
# Please do not distribute this code without permission.
# NO GUARANTEES THAT CODE IS CORRECT
# Caveat Emptor!

# Arguments
# ipath: directory where the netCDF files are located
# opath1: directory to allocate the split files
# opath2: directory to allocate the vertical average files

olayer <- function(ipath, opath1, opath2) {

####################################################################################
####### Defining the main packages (tryining to auto this)
####################################################################################
  # List of pacakges that we will be used
    list.of.packages <- c("doParallel", "parallel", "stringr", "data.table")
  # If is not installed, install the pacakge
   new.packages <- list.of.packages[!(list.of.packages %in% installed.packages()[,"Package"])]
   if(length(new.packages)) install.packages(new.packages)
  # Load packages
    lapply(list.of.packages, require, character.only = TRUE)
  
####################################################################################
####### Getting the path and directories for the files
####################################################################################
  # Establish the find bash command
    line1 <- paste(noquote("find"), noquote(ipath), "-type", "f", "-name", 
                   noquote("*.nc"), "-exec", "ls", "-l", "{}")
    line2 <- paste0("\\", ";")
    line3 <- paste(line1, line2)
  # Getting a list of directories for every netCDF file
    dir_files <- system(line3, intern = TRUE)
    dir_nc <- strsplit(x = dir_files, split = " ")
    nc_list <- lapply(dir_nc, function(x){f1 <- tail(x, n = 1)})
  # Cleaning the directories to get a final vector of directories
    final_nc <- lapply(nc_list, function(x) {
      c1 <- str_split(unlist(x), pattern = "//")
      c2 <- paste(c1[[1]][1], c1[[1]][2], sep = "/")})
    files.nc <- unlist(final_nc)
 
####################################################################################
####### Filtering by layers and generating new netCDF files with outputs
#################################################################################### 
  # Parallel looop
    cl <- makeCluster(3)
    registerDoParallel(cl)
    foreach(j = 1:length(files.nc), .packages = c("stringr")) %dopar% {
      # Trying to auto the name for every model
        var_obj <- system(paste("cdo -showname", files.nc[j]), intern = TRUE)
        var_all <- str_replace_all(string = var_obj, pattern = " ", replacement = "_")
        var <- tail(unlist(strsplit(var_all, split = "_")), n = 1) 
      # Defining depths
        levels <- as.vector(system(paste("cdo showlevel", files.nc[j]), intern = TRUE))
        lev <- unlist(strsplit(levels, split = " "))
        depths <- unique(lev[lev != ""])
      # Some can come in cm
        if(depths[1] >= 50) { 
          sf <- depths[as.numeric(depths) <= 500]
          ep <- depths[as.numeric(depths) >= 0 & as.numeric(depths) <= 20000]
          mp <- depths[as.numeric(depths) > 20000 & as.numeric(depths) <= 100000]
          bap <- depths[as.numeric(depths) > 100000]
        } else {
          sf <- depths[as.numeric(depths) <= 5]
          ep <- depths[as.numeric(depths) >= 0 & as.numeric(depths) <= 200]
          mp <- depths[as.numeric(depths) > 200 & as.numeric(depths) <= 1000]
          bap <- depths[as.numeric(depths) > 1000]
        }
      # Running CDO
        # Surface
          system(paste(paste("cdo -L -sellevel,", 
                             paste0(sf, collapse = ","), ",", sep = ""), 
                       paste("-selname,", var, sep = ""), files.nc[j], 
                       paste0(opath1, "01-sf_", basename(files.nc[j]))))
        # Epipelagic
          system(paste(paste("cdo -L -sellevel,", 
                             paste0(ep, collapse = ","), ",", sep = ""), 
                       paste("-selname,", var, sep = ""), files.nc[j], 
                       paste0(opath1, "02-ep_", basename(files.nc[j]))))
        # Mesopelagic
          system(paste(paste("cdo -L -sellevel,", 
                             paste0(mp, collapse = ","), ",", sep = ""), 
                       paste("-selname,", var, sep = ""), files.nc[j], 
                       paste0(opath1, "03-mp_", basename(files.nc[j]))))
        # Bathypelagic
          system(paste(paste("cdo -L -sellevel,", 
                             paste0(bap, collapse = ","), ",", sep = ""), 
                       paste("-selname,", var, sep = ""), files.nc[j], 
                       paste0(opath1, "04-bap_", basename(files.nc[j]))))
    }
    stopCluster(cl)
    
####################################################################################
####### Getting the path and directories for the "split by depth" files
####################################################################################
  # Establish the find bash command
    line1.1 <- paste(noquote("find"), noquote(opath1), "-type", "f", "-name", 
                     noquote("*.nc"), "-exec", "ls", "-l", "{}")
    line2.1 <- paste0("\\", ";")
    line3.1 <- paste(line1.1, line2.1)
  # Getting a list of directories for every netCDF file
    dir_files.2 <- system(line3.1, intern = TRUE)
    dir_nc.2 <- strsplit(x = dir_files.2, split = " ")
    nc_list.2 <- lapply(dir_nc.2, function(x){f1 <- tail(x, n = 1)})
  # Cleaning the directories to get a final vector of directories
    final_nc.2 <- lapply(nc_list.2, function(x) {
      c1 <- str_split(unlist(x), pattern = "//")
      c2 <- paste(c1[[1]][1], c1[[1]][2], sep = "/")})
    files.nc.2 <- unlist(final_nc.2)
    
####################################################################################
####### Filtering by layers and generating the "weighted-average depth layer" 
#################################################################################### 
  # Parallel looop
    cl <- makeCluster(3)
    registerDoParallel(cl)
    foreach(i = 1:length(files.nc.2), .packages = c("stringr")) %dopar% {
      # Running CDO
        system(paste(paste("cdo -L vertmean", sep = ""), files.nc.2[i], 
                     paste0(opath2, basename(files.nc.2[i])), sep = (" ")))
    }
    stopCluster(cl)
}
```


Running the ocean depth layer R function will:


```r
olayer(ipath = "/data/ClimateModelsRegrid/",
       opath1 = "/data/ClimateModelsRegridLayer/", 
       opath2 = "/data/ClimateModelsRegridLayerMean/")
```


### Merge several netcdf files with CDO

This function will help to merge several ocean depth layers (from the same model) into a single file.

* __Key CDO function__: `mergetime`


```r
merge_files <- function(ipath, opath1) {

####################################################################################
####### Defining the main packages (tryining to auto this)
####################################################################################
  # List of pacakges that we will be used
    list.of.packages <- c("doParallel", "parallel", "stringr", "data.table")
  # If is not installed, install the pacakge
    new.packages <- list.of.packages[!(list.of.packages %in% installed.packages()[,"Package"])]
    if(length(new.packages)) install.packages(new.packages)
  # Load packages
    lapply(list.of.packages, require, character.only = TRUE)
  
####################################################################################
####### Getting the path and directories for the files
####################################################################################
  # Establish the find bash command
    line1 <- paste(noquote("find"), noquote(ipath), "-type", "f", "-name", 
                   noquote("*.nc"), "-exec", "ls", "-l", "{}")
    line2 <- paste0("\\", ";")
    line3 <- paste(line1, line2)
  # Getting a list of directories for every netCDF file
    dir_files <- system(line3, intern = TRUE)
    dir_nc <- strsplit(x = dir_files, split = " ")
    nc_list <- lapply(dir_nc, function(x){f1 <- tail(x, n = 1)})
  # Cleaning the directories to get a final vector of directories
    final_nc <- lapply(nc_list, function(x) {
      c1 <- str_split(unlist(x), pattern = "//")
      c2 <- paste(c1[[1]][1], c1[[1]][2], sep = "/")})
    files.nc <- unlist(final_nc)
 
####################################################################################
####### Filtering by layers and generating new netCDF files with outputs
#################################################################################### 
  # Filtering (not dplyr!) by ocean layers
    sf <- files.nc[str_detect(string = basename(files.nc), pattern = "01-sf*") == TRUE]
    ep <- files.nc[str_detect(string = basename(files.nc), pattern = "02-ep*") == TRUE]
    mp <- files.nc[str_detect(string = basename(files.nc), pattern = "03-mp*") == TRUE]
    bap <- files.nc[str_detect(string = basename(files.nc), pattern = "04-bap*") == TRUE]
  # Defining how many models are per ocean layer 
    model_list_sf <- lapply(sf, function(x) 
      {d1 <- unlist(strsplit(x = basename(x), split = "_"))[4]})
    model_list_ep <- lapply(ep, function(x) 
      {d1 <- unlist(strsplit(x = basename(x), split = "_"))[4]})
    model_list_mp <- lapply(bap, function(x) 
      {d1 <- unlist(strsplit(x = basename(x), split = "_"))[4]})
    model_list_bap <- lapply(bap, function(x) 
      {d1 <- unlist(strsplit(x = basename(x), split = "_"))[4]})
    models <- unique(unlist(c(model_list_sf, model_list_ep, model_list_mp, model_list_bap)))
  # Parallel looop
    cl <- makeCluster(3)
    registerDoParallel(cl)
    foreach(i = 1:length(models), .packages = c("stringr")) %dopar% {
      f1 <- ep[str_detect(string = basename(sf), pattern = models[i]) == TRUE]
      system(paste(paste("cdo -L mergetime", paste0(f1, collapse = " "), sep = " "), 
                   paste0(opath1, paste(unlist(strsplit(basename(f1[1]), "_"))[c(1:7)], 
                                        collapse = "_"), ".nc"), sep = (" ")))
      f2 <- ep[str_detect(string = basename(ep), pattern = models[i]) == TRUE]
      system(paste(paste("cdo -L mergetime", paste0(f2, collapse = " "), sep = " "), 
                   paste0(opath1, paste(unlist(strsplit(basename(f2[1]), "_"))[c(1:7)], 
                                        collapse = "_"), ".nc"), sep = (" ")))
      f3 <- mp[str_detect(string = basename(mp), pattern = models[i]) == TRUE]
      system(paste(paste("cdo -L mergetime", paste0(f3, collapse = " "), sep = " "), 
                   paste0(opath1, paste(unlist(strsplit(basename(f3[1]), "_"))[c(1:7)], 
                                        collapse = "_"), ".nc"), sep = (" ")))
      f4 <- bap[str_detect(string = basename(bap), pattern = models[i]) == TRUE]
      system(paste(paste("cdo -L mergetime", paste0(f4, collapse = " "), sep = " "), 
                   paste0(opath1, paste(unlist(strsplit(basename(f4[1]), "_"))[c(1:7)], 
                                        collapse = "_"), ".nc"), sep = (" ")))
    }
    stopCluster(cl)
}
```

Running the merge function


```r
merge_files(ipath = "/Users/bri273/Desktop/CDO/models_regrid_vertmean/",
            opath1 = "/Users/bri273/Desktop/CDO/models_regrid_zmerge/")
```


The functions above were build based on OS. For Linux please check:

* __regrid__: [regrid R function](https://github.com/IsaakBM/CDO-climate-data-operators-/blob/master/scripts/r_scripts/01_ClimateModels_regrid02_HPC.R)
* __ocean layers__: [ocean layer R function](https://github.com/IsaakBM/CDO-climate-data-operators-/blob/master/scripts/r_scripts/02_ClimateModels_olayers02_HPC.R)
* __merge__: calculates [merge R function](https://github.com/IsaakBM/CDO-climate-data-operators-/blob/master/scripts/r_scripts/03_ClimateModels_MergeFiles02_HPC.R)

### Some useful CDO functions

CDO is more than just regridding. Some interesting useful functions are:

* __yearmean__: calculates the **annual mean** of a monthly data input **netCDF** file
* __yearmin__: calculates the **annual min** of a monthly data input **netCDF** file
* __yearmax__: calculates the **annual max** of a monthly data input **netCDF** file
* __ensmean__: calculates the **ensemble mean** of several **netCDF** files. If the input files are different models, this function will estimate a mean of all those models



