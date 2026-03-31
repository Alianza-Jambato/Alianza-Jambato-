---
title: "Species Distribution Model — Atelopus ignescens (Jambato toad)"
author: "Doménica Cevallos & Andrés Mármol-Guijarro"
date: "2026-03-31"
output: html_document
editor_options: 
  markdown: 
    wrap: 72
---

# Species Distribution Model — *Atelopus ignescens* (Jambato toad)

**Authors:** Domenica G. Cevallos & Andrés Mármol-Guijarro \
**Project:** Alianza Jambato — Single-species SDM for two time periods
(1985–1990 and 2018–2021)

------------------------------------------------------------------------

## Overview

This project models the habitat suitability of *Atelopus ignescens*
across mainland Ecuador using a Generalised Linear Model (GLM)
implemented in `biomod2`. The workflow runs two parallel SDM variants
for the 2018–2021 period: one using all environmental predictors
(climate + land use + DEM) and one using climate variables only. Outputs
are binarised suitability maps that are subsequently formatted for
visualisation in QGIS.

The project uses **relative paths throughout** via the `here` package.
All notebooks must be opened from within the project root so that
`here()` resolves correctly. The expected folder structure is:

```         
Project/
└── Atelopus niche modelling/
    └── single_SDM/
        ├── README.md
        ├── input_data/
        │   ├── Ecuador_DEM/
        │   ├── Chelsa_monthly_climate/
        │   └── other_files/
        ├── outputs/
        └── scripts/
            ├── Atelopus.ignescens/
            └── gadm_cache/
```

------------------------------------------------------------------------

## Notebook execution order

The notebooks must be run in the sequence below. Each stage depends on
outputs from the previous one.

| \#  | Notebook                            | Stage                       |
|-----|-------------------------------------|-----------------------------|
| 01  | `01_Chelsa_env_processing.rmd`      | Climate data preprocessing  |
| 02  | `02_Land_cover_processing.rmd`      | Land cover preprocessing    |
| 03  | `03_Land_use_mean_values.rmd`       | Land use period averaging   |
| 04  | `04_occurrence_data.rmd`            | Occurrence data preparation |
| 05a | `05a_allvar_ai_1821.rmd`            | SDM — all variables         |
| 05b | `05b_clim_ai_1821.rmd`              | SDM — climate only          |
| 06  | `06_Raster_processing_for_QGIS.Rmd` | QGIS raster formatting      |

Stages 05a and 05b are independent of each other and can be run in
either order, but both must complete before Stage 06.

------------------------------------------------------------------------

### Stage 1 — Climate data preprocessing

**Notebook:** `01_Chelsa_env_processing.rmd`

Reads monthly CHELSA climate rasters (`pr`, `tas`, `tasmax`, `tasmin`)
from the `Chelsa_monthly_climate/` folder at the project root, computes
period means for 1985–1990 and 2018–2021, clips them to mainland
Ecuador, and writes the results to `input_data/`.

**Requires:** - Raw CHELSA monthly `.tif` files in
`input_data/Chelsa_monthly_climate/` — update `dir_in` in the notebook
to point to this folder instead of the legacy external drive path

**Downloading the CHELSA files:**\
A `wget`-ready URL list is provided in `envidatS3paths.txt`. Run the
following from the project root to download all files into the correct
folder:

``` bash
wget -i envidatS3paths.txt -P Project/Atelopus\ niche\ modelling/single_SDM/input_data/Chelsa_monthly_climate/
```

> **Coverage note:** `envidatS3paths.txt` currently contains `tasmax`
> and `tasmin` for both periods (1985–1990 and 2018–2021) and `tas` for
> 2018–2021 only. `pr` and `tas` for 1985–1990 are not included and must
> be downloaded separately from [CHELSA
> v2.1](https://chelsa-climate.org) using the same URL pattern:
> `https://os.unil.cloud.switch.ch/chelsa02/chelsa/global/monthly/{variable}/{year}/CHELSA_{variable}_{month}_{year}_V.2.1.tif`.

**Produces** (written to
`Project/Atelopus niche modelling/single_SDM/input_data/`):

```         
mean_pr_1985-1990.tif       mean_pr_2018-2021.tif
mean_tas_1985-1990.tif      mean_tas_2018-2021.tif
mean_tasmax_1985-1990.tif   mean_tasmax_2018-2021.tif
mean_tasmin_1985-1990.tif   mean_tasmin_2018-2021.tif
```

------------------------------------------------------------------------

### Stage 2 — Land cover preprocessing

**Notebook:** `02_Land_cover_processing.rmd`

Reads annual MapBiomas Ecuador land cover rasters, reclassifies them
from the original granular categories into seven simplified classes
(`nat_forest`, `non_nat_forest`, `agriculture`, `antropic`, `water`,
`glacier`, `not_observed`), and upscales them to match the 30 arc-second
resolution of the CHELSA climate layers.

**Requires:** - MapBiomas annual `.tif` files accessible at the path
defined in `root_dir` (currently
`/Volumes/AM SSD/alianza_jambato/mapas mapbiomas`) - A raster template
(`precip_ai`) at 30 arc-second resolution — this is derived from any one
CHELSA file and must be available in the session

**Produces** (written alongside the source files on the external drive):

```         
landcover_30arc_YY_YY.tif   (one file per year covered)
```

> **Note:** Unlike other preprocessing notebooks,
> `Land_cover_processing` writes its per-year outputs back to the
> external drive, not into `input_data/`. The following stage
> (`Land_use_mean_values`) reads from there and deposits the final
> averaged files into `input_data/`.

------------------------------------------------------------------------

### Stage 3 — Land use period averaging

**Notebook:** `03_Land_use_mean_values.rmd`

Reads the per-year land cover files produced in Stage 2, groups them
into the two study periods (1985–1990 as "early", 2018–2021 as "late"),
computes a pixel-wise mean for each land cover class across all years in
each period, and writes the result to `input_data/`. The `not_observed`
band is excluded from averaging.

**Requires:** - Per-year `landcover_30arc_YY_YY.tif` files produced by
`02_Land_cover_processing.rmd`

**Produces** (written to
`Project/Atelopus niche modelling/single_SDM/input_data/`):

```         
land_use_Ecuador_1985-1990.tif
land_use_Ecuador_2018_2021.tif
```

> **Connection to SDM notebooks:** After this stage, `input_data/`
> contains all period-mean rasters (climate + land use) that the SDM
> notebooks load via `list.files(..., pattern = ".*2018-2021\\.tif$")`.
> Both the climate files from Stage 1 and the land use file from this
> stage match that pattern and are loaded together.

------------------------------------------------------------------------

### Stage 4 — Occurrence data preparation

**Notebook:** `04_occurrence_data.rmd`

Pulls *Atelopus ignescens* occurrence records from GBIF via the `rgbif`
API, merges them with field observations collected by Alianza Jambato
(2021–2022), cleans coordinates using `CoordinateCleaner` (removing
centroids, duplicates, and outliers), filters out iNaturalist records
with 30 km positional uncertainty, clips occurrences to the seven
provinces with confirmed presence, and constructs the `ai_occur`
dataframe used by the SDM notebooks.

**Requires:** - Internet connection for the GBIF API call -
`Project/Atelopus niche modelling/single_SDM/2018 - 2023/occ_2018_2023.csv`
(Alianza Jambato field records) - GADM boundary data (downloaded
automatically into `scripts/gadm_cache/` via `geodata::gadm`)

**Produces:** - `ai_occur` — in-memory dataframe (`species`, `lon`,
`lat`, `occur`) consumed directly by the SDM notebooks -
`Project/Atelopus niche modelling/single_SDM/input_data/ai_occur_clean.csv`
— persisted version for reloading across sessions

> **Session continuity:** If the SDM notebooks are run in a fresh R
> session, reload the occurrence data using the block at the end of
> `04_occurrence_data.rmd` that reads from `ai_occur_clean.csv` rather
> than re-querying GBIF.

> **Optional thinning:** A spatial thinning step using `spThin` is
> present but commented out. To activate it, uncomment the relevant
> block and switch the `dfaig_cl` / `dfaig_cl2` assignment lines as
> annotated in the notebook.

------------------------------------------------------------------------

### Stage 5a — SDM: all variables

**Notebook:** `05a_allvar_ai_1821.rmd`

Runs the full SDM pipeline using all available predictors: climate
variables (`pr`, `tas`, `tasmax`, `tasmin`), land use classes
(`nat_forest`, `non_nat_forest`, `agriculture`, `antropic`, `water`),
and DEM. Applies VIF stepwise variable selection (threshold = 10) before
modelling. Fits a GLM via `biomod2` with 2 pseudo-absence replicates
(500 points each, random strategy), projects the model, applies five
binary thresholds (950, 900, 800, 600, 500 on a 0–1000 scale), retains
only the full-data all-runs model layers, clips to the provinces of
confirmed presence, and writes the output.

**Requires** (all from previous stages): - Period-mean rasters in
`input_data/` matching `.*2018-2021\.tif$` - Raw DEM at
`input_data/Ecuador_DEM/10s090w_20101117_gmted_mea300.tif` - `ai_occur`
object in session (from `04_occurrence_data.rmd`) - `ec_mainland` and
`ai_prov` spatial objects in session (created inside the notebook via
`geodata::gadm`)

**Produces** (written to
`Project/Atelopus niche modelling/single_SDM/outputs/`):

```         
aibin_vif_allvar_1821.tif   (5-band raster, one band per threshold)
```

> **Known issue:** The seeds block (`seedformat`, `seedmodel`,
> `seedproj`) is defined twice — at the top of the notebook and again
> after the binarisation step. The second definition is redundant and
> can be removed without affecting results.

------------------------------------------------------------------------

### Stage 5b — SDM: climate variables only

**Notebook:** `05b_clim_ai_1821.rmd`

Identical pipeline to `05a_allvar_ai_1821.rmd` but restricts predictors
to climate variables and DEM only, dropping all land use layers at the
preprocessing step. This produces a climate-only suitability model for
comparison against the all-variables run.

**Requires:** Same as Stage 5a.

**Produces** (written to
`Project/Atelopus niche modelling/single_SDM/outputs/`):

```         
aibin_vif_clim_1821.tif     (5-band raster, one band per threshold)
```

> **Connection to Stage 6:** `06_Raster_processing_for_QGIS.Rmd` reads
> both `aibin_vif_clim_1821.tif` and `aibin_vif_allvar_1821.tif`
> directly by path — no pattern matching involved.

> **Path case sensitivity:** `05b_clim_ai_1821` writes to
> `here("project", ...)` (lowercase p) while
> `06_Raster_processing_for_QGIS` reads from `here("Project", ...)`
> (uppercase P). On macOS this resolves silently; on Linux
> (case-sensitive filesystem) this will break. Standardise the
> capitalisation if moving the project to a Linux environment.

------------------------------------------------------------------------

### Stage 6 — QGIS raster formatting

**Notebook:** `06_Raster_processing_for_QGIS.Rmd`

Processes both SDM outputs in a single run. A shared
`process_for_qgis()` function encapsulates the cascading `ifel()` logic,
which is applied first to the climate-only output and then to the
all-variables output. Each source file's five bands (thresholds 950,
900, 800, 600, 500) are collapsed so that each pixel appears only in the
highest threshold band at which it is classified as present, producing
five non-overlapping presence zones for QGIS styling.

**Requires:** - `aibin_vif_clim_1821.tif` produced by
`05b_clim_ai_1821.rmd` - `aibin_vif_allvar_1821.tif` produced by
`05a_allvar_ai_1821.rmd`

**Produces:**

```         
aibin_vif_clim_1821_qgis.tif    (5-band raster, QGIS-ready, NAflag = -32768)
aibin_vif_allvar_1821_qgis.tif  (5-band raster, QGIS-ready, NAflag = -32768)
```

> **Partial runs:** If only one of the two SDM notebooks has been run,
> the function will stop with an informative error on the missing file
> and leave the successfully processed output intact.

------------------------------------------------------------------------

## Session information

```         
R version 4.4.3 (2025-02-28)
Platform: aarch64-apple-darwin20
Running under: macOS 26.3.1

Matrix products: default
BLAS:   /System/Library/Frameworks/Accelerate.framework/Versions/A/Frameworks/vecLib.framework/Versions/A/libBLAS.dylib 
LAPACK: /Library/Frameworks/R.framework/Versions/4.4-arm64/Resources/lib/libRlapack.dylib;  LAPACK version 3.12.0

locale:
[1] en_US.UTF-8/en_US.UTF-8/en_US.UTF-8/C/en_US.UTF-8/en_US.UTF-8

time zone: Europe/Berlin
tzcode source: internal

attached base packages:
[1] grid      splines   stats     graphics  grDevices utils     datasets  methods   base     

other attached packages:
 [1] stringr_1.6.0           spThin_0.2.0            knitr_1.51              fields_17.1          
 [5] RColorBrewer_1.1-3      viridisLite_0.4.3       spam_2.11-3             countrycode_1.7.0      
 [9] CoordinateCleaner_3.0.1 geodata_0.6-6           lubridate_1.9.5         rgbif_3.8.4            
[13] here_1.0.2              dplyr_1.2.0             ggtext_0.1.2            tidyterra_1.0.0        
[17] ggplot2_4.0.2           xgboost_3.2.0.1         randomForest_4.7-1.2    maxnet_0.1.4           
[21] earth_5.3.5             plotmo_3.7.0            plotrix_3.8-14          Formula_1.2-5          
[25] gbm_2.2.3               mgcv_1.9-4              nlme_3.1-168            gam_1.22-7             
[29] foreach_1.5.2           mda_0.5-5               class_7.3-23            cito_1.1               
[33] rpart_4.1.24            nnet_7.3-20             biomod2_4.3-4-5         sf_1.1-0               
[37] raster_3.6-32           sp_2.2-1                usdm_2.1-7              terra_1.9-1            

loaded via a namespace (and not attached):
 [1] DBI_1.3.0              pROC_1.19.0.1          rlang_1.1.7            magrittr_2.0.4        
 [5] otel_0.2.0             e1071_1.7-17           compiler_4.4.3         maps_3.4.3            
 [9] callr_3.7.6            vctrs_0.7.1            reshape2_1.4.5         pkgconfig_2.0.3       
[13] fastmap_1.2.0          backports_1.5.0        rmarkdown_2.30         ps_1.9.1              
[17] torch_0.16.3           purrr_1.2.1            bit_4.6.0              xfun_0.56             
[21] jsonlite_2.0.0         PresenceAbsence_1.1.11 reshape_0.8.10         R6_2.6.1              
[25] stringi_1.8.7          Rcpp_1.1.1             iterators_1.0.14       Matrix_1.7-4          
[29] timechange_0.4.0       tidyselect_1.2.1       rnaturalearth_1.2.0    rstudioapi_0.18.0     
[33] abind_1.4-8            yaml_2.3.12            codetools_0.2-20       processx_3.8.6        
[37] lattice_0.22-9         tibble_3.3.1           plyr_1.8.9             withr_3.0.2           
[41] S7_0.2.1               geosphere_1.6-5        evaluate_1.0.5         survival_3.8-6        
[45] units_1.0-0            proxy_0.4-29           xml2_1.5.2             pillar_1.11.1         
[49] whisker_0.4.1          KernSmooth_2.23-26     checkmate_2.3.4        generics_0.1.4        
[53] rprojroot_2.1.1        scales_1.4.0           coro_1.1.0             glue_1.8.0            
[57] lazyeval_0.2.2         tools_4.4.3            data.table_1.18.2.1    dotCall64_1.2         
[61] tidyr_1.3.2            cli_3.6.5              rappdirs_0.3.4         gtable_0.3.6          
[65] oai_0.4.0              digest_0.6.39          classInt_0.4-11        farver_2.1.2          
[69] htmltools_0.5.9        lifecycle_1.0.5        httr_1.4.8             gridtext_0.1.6        
[73] bit64_4.6.0-1
```

## R package dependencies

| Package             | Purpose                                              |
|----------------------|--------------------------------------------------|
| `terra`             | Raster and vector spatial operations                 |
| `raster`            | Legacy compatibility with `biomod2`                  |
| `sf`                | Simple features vector data                          |
| `biomod2`           | SDM framework (GLM, ensemble, projection)            |
| `usdm`              | VIF calculation and stepwise variable selection      |
| `ggplot2`           | Plotting                                             |
| `tidyterra`         | Plotting `terra` objects with `ggplot2`              |
| `ggtext`            | Markdown formatting in `ggplot2` labels              |
| `dplyr`             | Data manipulation                                    |
| `here`              | Relative path management                             |
| `rgbif`             | GBIF occurrence API                                  |
| `lubridate`         | Date formatting                                      |
| `geodata`           | GADM administrative boundary download                |
| `CoordinateCleaner` | Occurrence record cleaning                           |
| `countrycode`       | ISO country code conversion                          |
| `spThin`            | Spatial thinning (optional, currently commented out) |
| `stringr`           | String operations (used in `Land_use_mean_values`)   |

------------------------------------------------------------------------
