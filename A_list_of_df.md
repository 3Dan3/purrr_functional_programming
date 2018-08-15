purrr functions
================
DanielH
August 13, 2018

-   [`modify_depth()`](#modify_depth)
-   [`every()`, `some()`](#every-some)
-   [`keep()`, `discard()`](#keep-discard)
-   [`modify_if()`, `modify_at()`](#modify_if-modify_at)
-   [`head_while()`, `tail_while()`](#head_while-tail_while)
-   [`pluck()`](#pluck)
-   [`negate()`](#negate)

This is an exploration of the R Package `purrr` and its functions. The package is a complete and consistent functional programming toolkit for R.

Here we consider two cases:

-   a list of dataframes
-   a list of lists

Data for the first case can be found here: <https://d396qusza40orc.cloudfront.net/rprog%2Fdata%2Fspecdata.zip>

------------------------------------------------------------------------

`modify_depth()`
----------------

> `modify_depth()` only modifies elements at a given level of a nested data structure.

``` r
## extract date column for each element, for example element 88

# case one
set.seed(0841)
specdata_list %>%
  .[88] %>%           # here we select list 88 containing a df
  modify_depth(1, select, Date) %>%
  modify_depth(1, sample_n, 3)
```

    ## [[1]]
    ## # A tibble: 3 x 1
    ##   Date      
    ##   <date>    
    ## 1 2008-03-29
    ## 2 2004-07-19
    ## 3 2006-04-15

``` r
# case two
set.seed(0841)
specdata_list %>%
  .[[88]] %>%  # here we select df 88
  select(Date) %>%
  sample_n(3)
```

    ## # A tibble: 3 x 1
    ##   Date      
    ##   <date>    
    ## 1 2008-03-29
    ## 2 2004-07-19
    ## 3 2006-04-15

``` r
# extract ID col, print first 3 rows, convert to vector, remove names
set.seed(0841)
specdata_list %>%
  sample(3) %>%         # 3 random elements of the list
  modify_depth(1, select, ID) %>%              # select ID in each element
  modify_depth(1, head, 3) %>%               # first 3 rows for each element
  modify_depth(1, as_vector) %>%         # convert the dfs to vectors
  modify_depth(1, unname) %>%             # to unnamed vectors
  flatten() %>%        # remove one level
  as_vector()        # make it a single vector
```

    ## [1] 202 202 202  26  26  26 108 108 108

``` r
#--------------- compute number of rows for each element of the list, dfs
# custom function
sel_col1 <- function(x) {
  x %>%
    dim() %>% 
    .[1]
}

# map function
specdata_list[203:206] %>% 
  modify_depth(1, sel_col1)
```

    ## [[1]]
    ## [1] 3287
    ## 
    ## [[2]]
    ## [1] 1096
    ## 
    ## [[3]]
    ## [1] 3652
    ## 
    ## [[4]]
    ## [1] 730

``` r
# equivalently
specdata_list[203:206] %>%
  modify_depth(1, nrow)
```

    ## [[1]]
    ## [1] 3287
    ## 
    ## [[2]]
    ## [1] 1096
    ## 
    ## [[3]]
    ## [1] 3652
    ## 
    ## [[4]]
    ## [1] 730

`every()`, `some()`
-------------------

> Do every or some elements of a list satisfy a predicate?

``` r
# check whether ALL element names always the same & matching the vector
specdata_list %>%
  modify_depth(1, names) %>%   # extract df names
  every(~.x %in% c("Date", "sulfate", "nitrate", "ID"))
```

    ## [1] TRUE

``` r
# check whether ALL element names are partially matching the vector
specdata_list %>%
  modify_depth(1, names) %>%   # extract df names
  flatten_chr() %>% 
  every(~.x %in% c("Date", "sulfate", "whatever", "ID"))
```

    ## [1] FALSE

``` r
# check whether some element names are matching the vector
specdata_list %>%
  modify_depth(1, names) %>%   # extract df names
  flatten_chr() %>% 
  some(~.x %in% c("Date", "sulfate", "bullo", "ID"))
```

    ## [1] TRUE

``` r
# check whether some ID cols contain the value: '14'
specdata_list %>%
  modify_depth(1, select, ID) %>%      # list of 331 dfs
  map(as_vector) %>%       # list 331 vectors
  map(unname) %>%        # 331 unnamed vectors
  flatten_int() %>%      # single integer vector, length 332
  some(~. == 14)
```

    ## [1] TRUE

``` r
# check whether some ID cols contain the value: '18565'
specdata_list %>%
  modify_depth(1, select, ID) %>%     # list of 331 dfs
  map(as_vector) %>%      # list 331 vectors
  map(unname) %>%      # 331 unnamed vectors
  flatten_int() %>%     # single integer vector, length 332
  some(~. == 18565)
```

    ## [1] FALSE

`keep()`, `discard()`
---------------------

> Keep or discard elements using a predicate function

``` r
## check how many cases we have for which ID == 34
# method 1
specdata_list %>%
  modify_depth(1, select, ID) %>%     # return list of 332 'ID' tibbles
  modify_depth(1, as_vector) %>%     # extract list of 332 'ID' vectors
  modify_depth(1, unname) %>%        # remove names from each vector
  flatten_int() %>% 
  keep(~. == 34) %>%
  length()
```

    ## [1] 1096

``` r
# method 2 
specdata_list %>%
  modify_depth(1, pull, ID) %>%       # return list of 332 'ID' vectors
  flatten_int() %>%             # return a single vector, length 332
  keep(~. == 34) %>% 
  length()
```

    ## [1] 1096

``` r
# method three
specdata_list %>%
  modify_depth(1, pull, 4) %>%
  flatten_int() %>%        # return a single vector, length 332
  keep(~. == 34) %>% 
  length()
```

    ## [1] 1096

``` r
# verify previous three 
specdata_list %>%
  .[[34]] %>%        # extract 34th dataframe
    dim() %>%         # rows and cols
  .[1]               # extract rows
```

    ## [1] 1096

``` r
## Same cases as above but using discard()
# method 1
specdata_list %>%
  modify_depth(1, select, ID) %>%       # return list of 332 'ID' tibbles
  modify_depth(1, as_vector) %>%       # extract list of 332 'ID' vectors
  modify_depth(1, unname) %>%         # remove names from each vector
  flatten_int() %>% 
  discard(~. != 34) %>%
  length()
```

    ## [1] 1096

``` r
# method 2 
specdata_list %>%
  modify_depth(1, pull, ID) %>%         # return list of 332 'ID' vectors
  flatten_int() %>%                 # return a single vector, length 332
  discard(~. != 34) %>% 
  length()
```

    ## [1] 1096

`modify_if()`, `modify_at()`
----------------------------

> `modify_if()` only modifies the elements of x that satisfy a predicate and leaves the others unchanged.

> `modify_at()` only modifies elements given by names or positions.

``` r
# modify the numeric col 'ID'
specdata_list %>%
  modify_depth(1, na.omit) %>%
  modify_depth(1, modify_if, is.numeric, ~.x + 1) %>%
  .[[5]] %>%
  sample_n(3)
```

    ## # A tibble: 3 x 4
    ##   Date       sulfate nitrate    ID
    ##   <date>       <dbl>   <dbl> <dbl>
    ## 1 2006-04-23    4.18    1.46     6
    ## 2 2009-10-22    3.07    1.30     6
    ## 3 2010-03-21    3.15    1.43     6

``` r
# modify col 'ID'
specdata_list %>%
  modify_depth(1, na.omit) %>%
  modify_depth(1, modify_at, "ID", ~sqrt(.x) + 1) %>%
  .[[5]] %>%
  sample_n(3)
```

    ## # A tibble: 3 x 4
    ##   Date       sulfate nitrate    ID
    ##   <date>       <dbl>   <dbl> <dbl>
    ## 1 2006-04-05    2.75   0.848  3.24
    ## 2 2006-12-01    1.3    0.824  3.24
    ## 3 2004-07-26    7.81   0.373  3.24

``` r
# modify col 'sulfate'
specdata_list %>%
  modify_depth(1, na.omit) %>%
  modify_depth(1, modify_at, "sulfate", ~sqrt(.x) + 10) %>%
  .[[5]] %>%
  sample_n(3)
```

    ## # A tibble: 3 x 4
    ##   Date       sulfate nitrate    ID
    ##   <date>       <dbl>   <dbl> <int>
    ## 1 2003-04-09    12.0   1.38      5
    ## 2 2009-07-24    12.1   0.332     5
    ## 3 2004-07-26    12.8   0.373     5

``` r
# modify col 'nitrate'
specdata_list %>%
  modify_depth(1, na.omit) %>%
  modify_depth(1, modify_at, "nitrate", ~sqrt(.x) + 10) %>%
  .[[265]] %>%
  sample_n(3)
```

    ## # A tibble: 3 x 4
    ##   Date       sulfate nitrate    ID
    ##   <date>       <dbl>   <dbl> <int>
    ## 1 2007-02-23    2.02    11.6   265
    ## 2 2005-06-27    7.83    10.9   265
    ## 3 2003-04-15    4.68    11.0   265

``` r
# modify col 'nitrate' and 'sulfate'
specdata_list %>%
  modify_depth(1, na.omit) %>%
  modify_depth(1, modify_at, c("nitrate", "sulfate"), ~sqrt(.x) + 100) %>%
  .[[265]] %>%
  sample_n(3)
```

    ## # A tibble: 3 x 4
    ##   Date       sulfate nitrate    ID
    ##   <date>       <dbl>   <dbl> <int>
    ## 1 2008-11-26    101.    101.   265
    ## 2 2003-04-21    102.    101.   265
    ## 3 2004-09-30    103.    101.   265

``` r
# modify at col 'Date'
specdata_list %>%
  modify_depth(1, na.omit) %>%
  modify_depth(1, modify_at, "Date", month, 
               label = TRUE, abbr = FALSE) %>%
  .[[113]] %>%
  sample_n(3)
```

    ## # A tibble: 3 x 4
    ##   Date   sulfate nitrate    ID
    ##   <ord>    <dbl>   <dbl> <int>
    ## 1 June      3.98   0.288   113
    ## 2 July      2.74   0.279   113
    ## 3 August    5.82   0.169   113

``` r
# modify if col type Date
specdata_list %>%
  modify_depth(1, na.omit) %>%
  modify_depth(1, modify_if, is.Date, year) %>%
  .[[113]] %>%
  sample_n(3)
```

    ## # A tibble: 3 x 4
    ##    Date sulfate nitrate    ID
    ##   <dbl>   <dbl>   <dbl> <int>
    ## 1  2009    1.76   0.402   113
    ## 2  2005    4.85   0.935   113
    ## 3  2006    3.95   0.503   113

`head_while()`, `tail_while()`
------------------------------

We can use this construct to figure out the length of each element (df) of the list. For example:

``` r
# using head_while
specdata_list %>%
  modify_depth(1, pull, ID) %>%
  flatten_int() %>%
  head_while(~. == 1) %>% 
  length()
```

    ## [1] 1461

``` r
# verify
specdata_list %>%
  .[[1]] %>%
  dim() %>%
  .[1]
```

    ## [1] 1461

``` r
# using tail_while
specdata_list %>%
  modify_depth(1, pull, ID) %>%
  flatten_int() %>%
  tail_while(~. >= 332) %>% 
  length()
```

    ## [1] 731

``` r
# verify
specdata_list %>%
  .[[332]] %>%
  dim() %>%
  .[1]  
```

    ## [1] 731

`pluck()`
---------

``` r
# create a named list
list_names <- seq(1:332)

specdata_list_named <-
  specdata_list %>%
  set_names(list_names)


specdata_list_named[222:224] %>%
  pluck("224") %>%
  sample_n(3)
```

    ## # A tibble: 3 x 4
    ##   Date       sulfate nitrate    ID
    ##   <date>       <dbl>   <dbl> <int>
    ## 1 2003-01-14      NA      NA   224
    ## 2 2003-05-04      NA      NA   224
    ## 3 2002-08-31      NA      NA   224

``` r
# here pull and pluck are equivalent
sub_list1 <-
  specdata_list[22:24] %>%
  modify_depth(1, pluck, 1)

sub_list2 <-
  specdata_list[22:24] %>%
  modify_depth(1, pull, 1)
# check
sub_list1 %>% 
  identical(sub_list2)
```

    ## [1] TRUE

``` r
specdata_list[223:225] %>%
  modify_depth(1, names) %>%
  modify_depth(1, str_to_upper)
```

    ## [[1]]
    ## [1] "DATE"    "SULFATE" "NITRATE" "ID"     
    ## 
    ## [[2]]
    ## [1] "DATE"    "SULFATE" "NITRATE" "ID"     
    ## 
    ## [[3]]
    ## [1] "DATE"    "SULFATE" "NITRATE" "ID"

``` r
# using attr_getter()
dfs <- 
  list(iris, mtcars)

dfs %>%
  str(max.level = 1)
```

    ## List of 2
    ##  $ :'data.frame':    150 obs. of  5 variables:
    ##  $ :'data.frame':    32 obs. of  11 variables:

``` r
dfs %>% 
  pluck(2, attr_getter("row.names")) %>%
  sample(5)
```

    ## [1] "Pontiac Firebird" "Merc 450SLC"      "Dodge Challenger"
    ## [4] "Datsun 710"       "Merc 280C"

``` r
dfs %>%
  pluck(1, attr_getter("row.names")) %>%
  sample(5)
```

    ## [1]  76  59 137 132 149

### more on col names matching

`negate()`
----------

Here we want to match our character vector `c("Date", "sulfate", "nitrate", "ID")` x 20 to a character vector like this: `c("Date", "sulfate", "ID")` We have a total of 80 elements of a list/vector, and `nitrate` (that is 1/4 of total cases) doesn't match the predicate `~.x %in% c("Date", "sulfate", "ID")`

Using keep/discard, we remove all the `nitrate` cols

``` r
rep(c("Date", "sulfate", "nitrate", "ID"), 20) %>% 
  #as.list() %>% 
  keep(negate( ~.x %in% c("Date", "sulfate", "ID")))  
```

    ##  [1] "nitrate" "nitrate" "nitrate" "nitrate" "nitrate" "nitrate" "nitrate"
    ##  [8] "nitrate" "nitrate" "nitrate" "nitrate" "nitrate" "nitrate" "nitrate"
    ## [15] "nitrate" "nitrate" "nitrate" "nitrate" "nitrate" "nitrate"

``` r
rep(c("Date", "sulfate", "nitrate", "ID"), 20) %>% 
  #as.list() %>% 
  discard(~.x %in% c("Date", "sulfate", "ID")) 
```

    ##  [1] "nitrate" "nitrate" "nitrate" "nitrate" "nitrate" "nitrate" "nitrate"
    ##  [8] "nitrate" "nitrate" "nitrate" "nitrate" "nitrate" "nitrate" "nitrate"
    ## [15] "nitrate" "nitrate" "nitrate" "nitrate" "nitrate" "nitrate"
