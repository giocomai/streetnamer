
<!-- README.md is generated from README.Rmd. Please edit that file -->

# streetnamer

<!-- badges: start -->

[![Lifecycle:
experimental](https://img.shields.io/badge/lifecycle-experimental-orange.svg)](https://lifecycle.r-lib.org/articles/stages.html#experimental)
<!-- badges: end -->

The goal of `streetnamer` is to facilitate the matching of street name
to their Wikidata identifiers.

This is a pre-release version. Some elements of the interface work as
expected, but the package it’s unusable in its present state. You have
been warned.

## Installation

You can install the development version of `streetnamer` with:

``` r
remotes::install_github("giocomai/latlon2map") # required dependency not on CRAN
remotes::install_github("EDJNet/tidywikidatar") # available on CRAN, but some features used by `streetnamer` may be added to the development version
remotes::install_github("giocomai/streetnamer")
```

This package relies heavily on
[`tidywikidatar`](https://edjnet.github.io/tidywikidatar).

Since all three packages (`streetnamer`, `latlon2map`, and
`tidywikidatar`) are being developed concurrently, leaving to each a
separate group of tasks, at this stage updates impacting the app may
occur to any of them. Hence, if anything is not working as expected, you
are invited to update those packages before reporting.

## How does it work?

At this stage, not much really works.

In order to get a preview of how the interface looks like, you can try
running the following code chunks.

Keep in mind that OpenStreetMap data for the whole country are
downloaded when you first select a city, so be prepared to wait many
minutes. Municipality-level data are cached and retrieved efficiently
afterwards.

``` r
library("streetnamer")
library("latlon2map")
library("tidywikidatar")
options(timeout = 60000) # big timeout, as big downloads needed 

ll_set_folder(path = fs::path(fs::path_home_r(),
                              "R",
                              "ll_data"))
#> /home/g/R/ll_data
sn_set_data_folder(fs::path(fs::path_home_r(),
                            "R",
                            "sn_data"))

# tidywikidatar cache
tw_set_cache_folder(path = fs::path(fs::path_home_r(),
                            "R",
                            "tw_data"))

## or in a temporary folder for testing

# tw_set_cache_folder(path = fs::path(tempdir(),
#                                     stringi::stri_rand_strings(n = 1, length = 24)))
#             

tw_create_cache_folder(ask = FALSE)

## if using rstudio, I'd suggest you set open this in your default browser
## rather than in rstudio's enabling the following option

# options(shiny.launch.browser = .rs.invokeShinyWindowExternal)

# sn_run_app()
```

## Function naming conventions

`streetnamer` has two main types of functions:

-   a set of functions used to facilitate processing, that can
    conventionally be used from the command line, or internally by the
    Shiny app: they all start with `sn_` followed by a verb,
    e.g. `sn_get_lau_street_names()`
-   a set of functions that are in effect Shiny modules (see below).
    They typically start with `mod_sn_` and are currently not exported
    (as is customary for non-exported functions, they can be used with
    the triple `:`, e.g. `streetnamer:::mod_sn_street_info_app`) .

## Shiny modules

In order to facilitate development, as well as to allow integration of
component parts of this app in spin-off projects, key components of the
Shiny app have been developed as modules and can be tested
independently.

### Module that shows info about Wikidata

``` r
streetnamer:::mod_sn_street_info_app(street_name = "Belvedere San Francesco",
                                     gisco_id = "IT_022205")
```

### Module for showing

### Module for exporting data

## What happens in the background

The selectors on the top and the left allow to pick a municipality, and
then a street.

When you click on a street name, a set of options to add data on a given
street name appears. This is the choices that appear with this module:

``` r
streetnamer:::mod_sn_street_info_app(street_name = "Belvedere San Francesco",
                                     gisco_id = "IT_022205")
```

All the choices made in this interface are transformed into a data
frame, that is written into a database:

``` r
sn_write_street_name_wikidata_id(
  gisco_id = "IT_022205",
  country = "IT",
  street_name = "Belvedere San Francesco",
  person = TRUE,
  wikidata_id = "Q676555",
  gender = "male",
  category = "religion",
  tag = "",
  checked = TRUE,
  session = "testing",
  append = TRUE,
  overwrite = FALSE,
  disconnect_db = TRUE
)
#> NULL
#> # A tibble: 1 × 13
#>   gisco_id  stree…¹ country wikid…² person gender categ…³ checked ignore dedic…⁴
#>   <chr>     <chr>   <chr>   <chr>    <int> <chr>  <chr>     <int>  <int>   <int>
#> 1 IT_022205 Belved… IT      Q676555      1 male   religi…       1     NA      NA
#> # … with 3 more variables: tag <chr>, session <chr>, time <dbl>, and
#> #   abbreviated variable names ¹​street_name, ²​wikidata_id, ³​category,
#> #   ⁴​dedicated_to_n
#> # ℹ Use `colnames()` to see all variable names


street_info_df <- sn_get_street_name_wikidata_id(
  gisco_id = "IT_022205",
  street_name = "Belvedere San Francesco",
  country = "IT"
)
#> Warning: <Pool> uses an old dbplyr interface
#> ℹ Please install a newer version of the package or contact the maintainer
#> This warning is displayed once every 8 hours.

street_info_df %>% 
  dplyr::distinct(gisco_id, .keep_all = TRUE) %>% 
  tidyr::pivot_longer(cols = dplyr::everything(),
                      names_to = "type",
                      values_to = "value",
                      values_drop_na = FALSE,
                      values_transform = as.character) %>% 
  print(n = 100)
#> # A tibble: 13 × 2
#>    type           value                    
#>    <chr>          <chr>                    
#>  1 gisco_id       "IT_022205"              
#>  2 street_name    "Belvedere San Francesco"
#>  3 country        "IT"                     
#>  4 wikidata_id    "Q676555"                
#>  5 person         "1"                      
#>  6 gender         "male"                   
#>  7 category       "religion"               
#>  8 checked        "1"                      
#>  9 ignore          <NA>                    
#> 10 dedicated_to_n  <NA>                    
#> 11 tag            ""                       
#> 12 session        "testing"                
#> 13 time           "1658240047.54168"
```

Each time the “confirm” button is clicked, a new row is added to the
database. Hence, when you process the data you need to decided which
criteria to use for keeping data, e.g. the most recent row, or the most
confirmed.

This set of data support a number of special cases, and different
degrees of information that can be shared:

Done: - data is confirmed at the country or city level - we expect data
to be valid if confirmed at country levels, but especially with common
surnames (or e.g. common names of saints, where one city has places
dedicated to a locally born but globally less famous saint) it may be
useful to check data at the city level - when checking if a street is
tagged, this can be effectively done by filter for either the `gisco_id`
column or the `country` column - it is possible to ignore a given street
name - in OpenStreetMap is relatively common to have some streets that
do not have a proper street name, mostly because they are improperly
tagged (e.g. just a number, or a hyphen), or because they have
descriptive names that are not actually street names (e.g. “access ramp
to hospital”). These should simply be ignored and not added in the count
of total streets. - this is expressed via the `ignore` column, with
expected values either 1 (TRUE) or 0 (FALSE) - make it possible to
confirm that a street name is not named after a human, without adding
anything else - this is useful because in some use cases the main point
of interest is humans, and requiring to add a Wikidata identifier would
needlessly prolong the checking times - this is expressed via the
`person` column, with expected values either 1 (TRUE) or 0 (FALSE) -
make it possible to claim that a street is named after more than one
person/individual - this is achieved by having a column with how many
entities the street is dedicated to, `dedicated_to_n`. When reading the
data, if `dedicated_to_n` is more than 1, then more than one row with
data is expected to be found. Is is the responsibility of those who read
the data do deal with potential inconsistencies

To do:

## Deduplication

-   add Wikidata identifier of the actual street - this can be useful,
    as a number of properties are associated to it, possibly including
    different values for “named after” with qualifiers when street names
    changed
    -   this is achieved with a separate column, `wikidata_street_id`.
        This should always be considered in combination with a given
        municipality.

## Caching

Rather than adopting a separate caching infrastrucutre, `streetnamer`
relies on the caching infrastructure of `tidywikidatar`. In brief, it
generates separate tables with non-conflicting names in the same
database used by `tidywikidatar` (be it a local SQLite or another
odbc-compliant servers such as SQL)

## Deployed shiny app

Given that Shiny Server limits access to environment variables, for the
deployed app a connection must be directly passed to `sn_run_app()`, and
cannot be simply be set before startup (which works fine when running
the app locally).

## Data sources

-   OpenStreetMap data (© OpenStreetMap contributors) as kindly made
    available by [Geofabrik](http://download.geofabrik.de/)

## Desired features

It should be possible deal with the following circumstances:

-   streets that are on OSM
-   streets that are available on other lists, but not on OSM
-   streets with wikidata id or without
-   streets that are a person or not a person
-   different streets that are the same street (deduplication)
-   not a street / irrelevant
-   single street has more wikidata id (e.g. dedicated to two
    individuals)
-   add a tag for each street (maybe, free tag from a controlled
    vocabulary, e.g. to mark streets related to some issue that would
    not appear from relevant Wikidata identifier)

## Checks for things that should work

## On naming things

OpenStreetMap groups all sorts of roads, streets, squares, and paths
under the confusing label of “highway”. Within this package, the generic
word used in function and documentation will be “streets”, as the
package is expected to be used chiefly in reference to urban centres.

## Contributing

Suggestions and contributions are welcome; they can be discussed via
GitHub issues.

## Copyright and credits

This package has been created by [Giorgio
Comai](https://giorgiocomai.eu), data analyst and researcher at
[OBCT/CCI](https://balcanicaucaso.org/), within the scope of
[EDJNet](https://europeandatajournalism.eu/), the European Data
Journalism Network.

It is distributed under the MIT license.
