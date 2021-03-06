NPS case
================
DanielH
August 23, 2018

-   [load data and set names](#load-data-and-set-names)
-   [fit linear model](#fit-linear-model)
-   [check assumptions](#check-assumptions)
-   [transform variable](#transform-variable)

------------------------------------------------------------------------

This case is based on National parks service data.

Here we’ll look at annual total recreation, tent camping, and backcountry camping visits from the years 2007-2016.

Our data is stored in three distinct datasets, each foEach year’s data is stored as a separate CSV file

load data and set names
-----------------------

``` r
# load data
load("purrr_data.Rdata")

# rename list
nps_list <- datalist

# remove datalist
rm(datalist)

# names vector
list_nms <- c("total recreation data", "tent camping data", "backcountry camping data")

# name elements of the list
nps_list <-
  nps_list %>%
  set_names(list_nms)
```

fit linear model
----------------

Here we want to fit a linear model to each element of the `nps_list`. First we create a function `park_model`

``` r
# define function
park_model <- 
  function(df) {
  lm(value ~ year * region, data = df)
  }


# map the function to each element in the list
nps_list_nested <-
  nps_list %>%
  map(park_model)
```

check assumptions
-----------------

Here we wanto to check whetehr our data satisifies the conditions for using a linear model. First we look at the residuals, then we look at the response variable distribution

``` r
# ---------------------- we now want to plot residuals(y) vs fitted.values(x)

# define plot function
nps_res_plot <-
  function(df) {
    df %>%
      ggplot(aes(residuals, fitted.values)) +
      geom_point()
   }

    
# plot residuals vs fitted values    
nps_list_nested %>%
  walk(~ print(ggplot(data = ., 
                      aes(x = .$residuals,
                          y = .$fitted.values)) +
                 geom_point() +
                 labs(x = "residuals", 
                      y = "fitted values") +
                 coord_flip()))
```

![](NPS_case_files/figure-markdown_github/unnamed-chunk-4-1.png)![](NPS_case_files/figure-markdown_github/unnamed-chunk-4-2.png)![](NPS_case_files/figure-markdown_github/unnamed-chunk-4-3.png)

``` r
# plot variable 'value' distributions
nps_list_nested %>%
  walk(~ print(ggplot(data = ., aes(x = value)) +
                 geom_histogram(color = "white",
                                fill = "cadetblue")))
```

![](NPS_case_files/figure-markdown_github/unnamed-chunk-4-4.png)![](NPS_case_files/figure-markdown_github/unnamed-chunk-4-5.png)![](NPS_case_files/figure-markdown_github/unnamed-chunk-4-6.png)

transform variable
------------------

As we can see the response variable is not normal and not even nearly normal, therefore we try a log transformation.

We then take a look at the R squared values

``` r
# log transformation of the variable 'value'
nps_list_nested <-
  nps_list %>%
  map(~mutate_at(.x, "value", log)) %>%
  set_names(list_nms) %>%   # name elements
  modify_depth(1, park_model)


# check transormed variable
nps_list_nested %>%
  walk(~ print(ggplot(data = ., aes(x = value)) +
                 geom_histogram(color = "white",
                                fill = "cadetblue")))
```

![](NPS_case_files/figure-markdown_github/unnamed-chunk-5-1.png)![](NPS_case_files/figure-markdown_github/unnamed-chunk-5-2.png)![](NPS_case_files/figure-markdown_github/unnamed-chunk-5-3.png)

``` r
# plot residuals vs fitted values    
nps_list_nested %>%
  walk(~ print(ggplot(data = ., 
                      aes(x = .$residuals,
                          y = .$fitted.values)) +
                 geom_point() +
                 labs(x = "residuals", 
                      y = "fitted values") +
                 coord_flip()))
```

![](NPS_case_files/figure-markdown_github/unnamed-chunk-5-4.png)![](NPS_case_files/figure-markdown_github/unnamed-chunk-5-5.png)![](NPS_case_files/figure-markdown_github/unnamed-chunk-5-6.png)

``` r
# extract R squared for each element
nps_list_nested %>%
  map_dbl(~summary(.)$adj.r.squared) %>%
  round(2)
```

    ##    total recreation data        tent camping data backcountry camping data 
    ##                     0.03                     0.05                     0.00

First, we create a dataframe of all possible combinations of year and region

``` r
pred_data <- 
  cross_df(.l = list("year" = unique(pluck(nps_list, 1, "year")),
                     "region" = levels(nps_list[[1]]$region))
)
```

For each visit type, we create a dataframe with predicted \# for each year/region

`se.fit = TRUE`

``` r
pred_list <- 
  nps_list_nested %>%
  map(.f = predict, newdata = pred_data, se.fit = TRUE) %>%
  map(~data_frame(fit = pluck(., "fit"), 
                  se = .$se.fit) %>%
        mutate(lcl = fit - qnorm(0.975) * se,
               ucl = fit + qnorm(0.975) * se)) %>%
  map(bind_cols, pred_data)  ## Add year and region onto each
```

We now want to create a plot faceted by region

``` r
plot_predicted <- 
  function(df, vscale, maintitle){

if(!all(c("fit", "se", "lcl", "ucl", "year", "region") %in% names(df))){
  stop("df should have columns fit, se, lcl, ucl, year, region")
}
    
## Create a plot faceted by region
p <- 
  ggplot(data = df, aes(x = year, y = fit)) +
  facet_wrap(~ region, nrow = 2) +
  geom_ribbon(aes(ymin = lcl, ymax = ucl, fill = region), alpha = 0.4) +
  geom_line(aes(color = region), size = 2) +
  scale_fill_viridis(option = vscale, discrete = TRUE, end = 0.75) +
  scale_colour_viridis(option = vscale, discrete = TRUE, end = 0.75) +
  labs(title = maintitle,
       x = NULL, y = "Log(Visitors)") +
  theme(legend.position = "none")

return(p)

}
```

We can now use the parallel mapping function `pmap`.

First, let’s set up a list of arguments. The list has 3 elements

-   A dataframe
-   a character vector `vscale`
-   a character vector `maintitle`

``` r
plot_args <- 
  list("df" = pred_list,
       "vscale" = c("D", "A", "C"),
       "maintitle" = c("Total Recreational Visits",
                       "Tent Campers", "Backcountry Campers"))
```

Now that we have our plotting function `plot_predicted` written and our arguments are set up in the list `plot_args`, we can get all our plots with one line:

``` r
nps_plots <- 
  pmap(plot_args, plot_predicted)

nps_plots %>%
  walk2(.x = c("rec.pdf", "tent.pdf", "backcountry.pdf"), 
        ., ggsave, width = 8, height = 6)
```
