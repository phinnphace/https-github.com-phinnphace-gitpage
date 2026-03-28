 ##  Read the crosswalk and filter to your tract
 tract is 1400000US27053101600, but the crosswalk uses the bare 11-digit tract GEOID, so:
 tr2020ge == "27053101600"
> library(readr)
> library(dplyr)
> library(janitor)

> xwalk_path <-nhgis_blk2010_tr2020_27/nhgis_blk2010_tr2020_27.csv
> xwalk <- read_csv(xwalk_path, show_col_types = FALSE) |>
+     clean_names() |>
+     mutate(
+         blk2010ge = as.character(blk2010ge),
+         tr2020ge  = as.character(tr2020ge)
+     )
>                                                                             
> tract_id <- "27053101600"
> 
> xwalk_t <- xwalk |>
+     filter(tr2020ge == tract_id)
> 
> nrow(xwalk_t)
[1] 54

## Check ID's

> xwalk_t |> summarise(
+     atoms = n(),
+     n_blocks = n_distinct(blk2010ge),
+     weight_sum = sum(weight, na.rm = TRUE),
+     parea_sum = sum(parea, na.rm = TRUE)
+ )
# A tibble: 1 × 4
  atoms n_blocks weight_sum parea_sum
  <int>    <int>      <dbl>     <dbl>
1    54       54         54        54
> 
# (Do not assume weight_sum == 1. This is not HUD-style normalization.)
library(tidycensus)

## 2 Pull 2010 block total population( Matches crosswalk source)
crosswalk is 2010 blocks → 2020 tracts, so the clean aligned source is Decennial 2010 SF1 block population.

blk_pop_2010 <- get_decennial(
+     geography = "block",
+     variables = "P001001",   # Total population, 2010 SF1
+     year = 2010,
+     sumfile = "sf1",
+     state = "MN",
+     county = "Hennepin",
+     geometry = FALSE
+ ) |>
+     transmute(
+         blk2010ge = as.character(GEOID),
+         pop2010   = as.numeric(value)
+     )

census_api_key()

> library(tidycensus)
> 
> blk_pop_2010 <- get_decennial(
+     geography = "block",
+     variables = "P001001",   # Total population, 2010 SF1
+     year = 2010,
+     sumfile = "sf1",
+     state = "MN",
+     county = "Hennepin",
+     geometry = FALSE
+ ) |>
+     transmute(
+         blk2010ge = as.character(GEOID),
+         pop2010   = as.numeric(value)
+     )
Getting data from the 2010 decennial Census
Using FIPS code '27' for state 'MN'
Using FIPS code '053' for 'Hennepin County'
Using Census Summary File 1
  ## 3 Join + apply NHGIS weight + sum to the tract 

tract_est <- xwalk_t |>
+     left_join(blk_pop_2010, by = "blk2010ge") |>
+     mutate(pop_alloc = pop2010 * weight) |>
+     summarise(
+         pop2010_est_in_tr2020 = sum(pop_alloc, na.rm = TRUE),
+         atoms = n(),
+         blocks_matched = n_distinct(blk2010ge[!is.na(pop2010)]),
+         blocks_total = n_distinct(blk2010ge),
+         pct_blocks_matched = 100 * blocks_matched / blocks_total
+     )
> 
> tract_est
# A tibble: 1 × 5
  pop2010_est_in_tr2020 atoms blocks_matched blocks_total pct_blocks_matched
                  <dbl> <int>          <int>        <int>              <dbl>
1                  2437    54             54           54                100

## Write up: When we follow NHGIS/IPUMS best-practice interpolation exactly, the resulting estimates are internally stable. 
   However, the transformation still produces a non-zero Δx relative to alternate boundary representations, 
   and this change is undocumented in downstream use.”
