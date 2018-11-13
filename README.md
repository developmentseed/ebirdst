
<!-- README.md is generated from README.Rmd. Please edit that file -->
ebirdst: Tools for Loading, Mapping, Plotting and Analysis of eBird Status and Trends Data Products
===================================================================================================

<!-- [![License: GPL v3](https://img.shields.io/badge/License-GPL%20v3-blue.svg)](http://www.gnu.org/licenses/gpl-3.0) -->
**THIS PACKAGE IS UNDER ACTIVE DEVELOPMENT. FUNCTIONALITY MAY CHANGE AT ANY TIME.**

The goal of this package is to providet tools for loading, mapping, plotting, and analysis of eBird Status and Trends data products. If you're looking for the previous version (known as stemhelper), which is no longer being maintained, please access the stemhelper\_archive branch.

Installation
------------

You can install stemhelper from GitHub with:

``` r
# install the development version from GitHub
# install.packages("devtools")
devtools::install_github("CornellLabofOrnithology/ebirdst")
```

Vignette
--------

TODO: update website, then updated this section.

For a full introduction and advanced usage, please see the package [website](https://cornelllabofornithology.github.io/stemhelper). An [introductory vignette](https://cornelllabofornithology.github.io/stemhelper/articles/stem-intro-mapping.html) is available, detailing the structure of the results and how to begin loading and mapping the results. Further, an [advanced vignette](https://cornelllabofornithology.github.io/stemhelper/articles/stem-pipd.html) details how to access additional information from the model results about predictor importance and directionality, as well as predictive performance metrics.

Quick Start
-----------

TODO: Add details on how to acquire results.

**IMPORTANT. AFTER DOWNLOADING THE RESULTS AND UNZIPPING THEM, DO NOT CHANGE THE FILE STRUCTURE.** All functionality in this package relies on the structure inherent in the delivered results. Changing the folder and file structure will cause errors with this package. See the SETUP PATHS part of the example code below.

One of the most common first tasks is to load a RasterStack of relative abundance estimates and make a map.

``` r
library(viridis)
library(raster)
library(rnaturalearth)

# SETUP PATHS
# Once you have downloaded and unzipped the results, place the resulting folder,
# with the unique species ID name (e.g., woothr-ERD2016-PROD-20170505-3f880822)
# wherever you want to keep it. Then provide the root_path variable as that
# location (without the unique species ID name) and provide the unique species 
# ID name as the species variable below. Use the paste() function to combine

# This example downloads example data directly
#species <- "amerob-ERD2016-EXAMPLE_DATA-20171122-5a9e702e"
#species_url <- paste("http://ebirddata.ornith.cornell.edu/downloads/hidden/", 
#                     species, ".zip", sep = "")

#temp <- tempfile()
#temp_dir <- tempdir()
#download.file(species_url, temp)
#unzip(temp, exdir = temp_dir)
#unlink(temp)

#sp_path <- paste(temp_dir, "/", species, sep = "")

# Temp version before I put these on the web
root_path <- "~/Box Sync/Projects/2015_stem_hwf/documentation/data-raw/"
species <- "woothr-ERD2016-SP_TEST-20180724-7ff34421"
sp_path <- paste(root_path, species, sep = "")

abunds <- raster::stack(paste0(sp_path, "/results/tifs/", species,
                               "_hr_2016_abundance_umean.tif"))
abunds <- label_raster_stack(abunds)

# to subset the data and add context, pull in some reference data from the rnaturalearth package
wh <- ne_countries(continent = c("North America"))
wh_states <- ne_states(iso_a2 = unique(wh@data$iso_a2))

# region for subsetting
wh_sub <- wh_states[wh_states$name %in% c("Minnesota", "Wisconsin", "Illinois", 
                                          "Iowa", "Indiana", "Ohio", "Michigan"), ]

# subset to a portion of the range
ne_extent <- list(type = "polygon",
                  polygon = wh_sub,
                  t.min = 0.48,
                  t.max = 0.50)
abund <- raster_st_subset(abunds, ne_extent)
rm(abunds)

# project to Mollweide for mapping and extent calc
mollweide <- CRS("+proj=moll +lon_0=-90 +x_0=0 +y_0=0 +ellps=WGS84")

# the nearest neighbord method preserves cell values across projections
abund_moll <- projectRaster(abund, crs = mollweide, method = 'ngb')

# project the reference data as well
wh_moll <- spTransform(wh, mollweide)
wh_states_moll <- spTransform(wh_states, mollweide)

# calculate extent
#sp_ext <- calc_full_extent(abund_moll, sp_path)

#xlimits <- c(sp_ext[1], sp_ext[2])
#ylimits <- c(sp_ext[3], sp_ext[4])

#xrange <- xlimits[2] - xlimits[1]
#yrange <- ylimits[2] - ylimits[1]

# these are RMarkdown specific here, but could be passed to png()
#w_cm <- 8.5
#h_cm <- w_cm*(yrange/xrange)
#knitr::opts_chunk$set(fig.width = w_cm, fig.height = h_cm)

# calculate ideal color bins for abundance values for this week
week_bins <- calc_bins(abund_moll)

# start plotting
par(mfrow = c(1, 1), mar = c(0, 0, 0, 6))

# use the extent object to set the spatial extent for the plot
plot(extent(trim(abund_moll, values = NA)), col = 'white')

# add background spatial context
plot(wh_states_moll, col = "#eeeeee", border = NA, add = TRUE)

# plot zeroes
plot(abund_moll == 0,
     maxpixels = ncell(abund_moll),
     ext = extent(trim(abund_moll, values = NA)),
     col = '#dddddd',
     xaxt = 'n',
     yaxt = 'n',
     legend = FALSE,
     add = TRUE)

# plot the raster
these_cols <- rev(viridis::plasma(length(week_bins$bins) - 2, end = 0.9))
grayInt <- colorRampPalette(c("#dddddd", these_cols[1]))
qcol <- c(grayInt(4)[2], these_cols)

plot(abund_moll,
     maxpixels = ncell(abund_moll),
     ext = extent(trim(abund_moll, values = NA)),
     breaks = week_bins$bins,
     col = qcol,
     xaxt = 'n',
     yaxt = 'n',
     legend = FALSE,
     add = TRUE)

# for legend plotting create a smaller set of bin labels and do it separately
bin_labels <- format(round(week_bins$bins, 2), nsmall = 2)
bin_labels[!(bin_labels %in% c(bin_labels[1],
                               bin_labels[round((length(bin_labels) / 2)) + 1],
                               bin_labels[length(bin_labels)]))] <- ""

bin_labels <- c("0", bin_labels)
qcol <- c('#dddddd', qcol)
ltq <- seq(from = week_bins$bins[1], to = week_bins$bins[length(week_bins$bins)],
               length.out = length(week_bins$bins))
ltq <- c(0, ltq)

plot(abund_moll ^ week_bins$power,
     col = qcol,
     legend.only = TRUE,
     breaks = ltq,
     lab.breaks = bin_labels,
     legend.shrink = 0.97,
     legend.width = 2,
     axis.args = list(cex.axis = 0.9, lwd.ticks = 0))

# add state boundaries on top
plot(wh_states_moll, add = TRUE, border = 'white', lwd = 1.5)
```

<img src="README-quick_start-1.png" style="display: block; margin: auto;" />

``` r

# clean up
#unlink(temp_dir)
```
