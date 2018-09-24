Sparklyr Introduction
================

In this notebook we walk through some basic examples of how to use sparklyr in R. This notebook is based on the guide provided at <http://spark.rstudio.com/>.

Connecting to Spark
-------------------

Before we start we must install a local instance of Spark

``` r
library(sparklyr)
spark_install(version = "2.1.0")
```

We can connect to clusters and local instances of Spark as follows (I had to create a new Spark connection first):

``` r
sc <- spark_connect(master = "local")
```

    ## * Using Spark: 2.1.0

Using dplyr
-----------

Next we will show that we cn use all of the standard dplyr verbs against tables in the cluster. First we must populate the cluster with data:

``` r
library(dplyr)
```

    ## 
    ## Attaching package: 'dplyr'

    ## The following objects are masked from 'package:stats':
    ## 
    ##     filter, lag

    ## The following objects are masked from 'package:base':
    ## 
    ##     intersect, setdiff, setequal, union

``` r
iris_tbl <- copy_to(sc, iris)
flights_tbl <- copy_to(sc, nycflights13::flights, "flights")
batting_tbl <- copy_to(sc, Lahman::Batting, "batting")
src_tbls(sc)
```

    ## [1] "batting" "flights" "iris"

Visiting the SparkUI we see that this is a relatively small database (only about 50MB). We could imagine in real cases we would be working with larger databases. Note that the data tables lie `iris_tbl` are lists and are not stored in memory.

Let's try a simple filtering example:

``` r
# filter by departure delay and print the first few records
flights_tbl %>% filter(dep_delay == 2) %>% head()
```

    ## # Source:   lazy query [?? x 19]
    ## # Database: spark_connection
    ##    year month   day dep_time sched_dep_time dep_delay arr_time
    ##   <int> <int> <int>    <int>          <int>     <dbl>    <int>
    ## 1  2013     1     1      517            515      2.00      830
    ## 2  2013     1     1      542            540      2.00      923
    ## 3  2013     1     1      702            700      2.00     1058
    ## 4  2013     1     1      715            713      2.00      911
    ## 5  2013     1     1      752            750      2.00     1025
    ## 6  2013     1     1      917            915      2.00     1206
    ## # ... with 12 more variables: sched_arr_time <int>, arr_delay <dbl>,
    ## #   carrier <chr>, flight <int>, tailnum <chr>, origin <chr>, dest <chr>,
    ## #   air_time <dbl>, distance <dbl>, hour <dbl>, minute <dbl>,
    ## #   time_hour <dttm>

Let's quickly review some dplyr grammar: The pipe operator `%>%` indicates feeding the expression on its left-hand side into the function on the right-hand side. This allows us to chain many functions on a dataframe while maintaining readability of the command (compare the above with the equivalent expression `head(filter(flights_tbl, dep_delay==2))` ).

A more coplicated example:

``` r
delay <- flights_tbl %>% 
  group_by(tailnum) %>%
  summarise(count = n(), dist = mean(distance), delay = mean(arr_delay)) %>%
  filter(count > 20, dist < 2000, !is.na(delay)) %>%
  collect
```

    ## Warning: Missing values are always removed in SQL.
    ## Use `AVG(x, na.rm = TRUE)` to silence this warning

    ## Warning: Missing values are always removed in SQL.
    ## Use `AVG(x, na.rm = TRUE)` to silence this warning

Plot the queried data:

``` r
library(ggplot2)
ggplot(delay, aes(dist, delay)) +
  geom_point(aes(size = count), alpha = 1/2) +
  geom_smooth() +
  scale_size_area(max_size = 2)
```

    ## `geom_smooth()` using method = 'gam'

![](intro-to-sparklyr_files/figure-markdown_github/unnamed-chunk-6-1.png)

Now we review ggplot2 syntax: The first line specifies the x-y axes of the plot. Each additional line preceded by a '+' indicates another layer to be added. The first layer added is a scatter plot where the point size is determined by the counts of instances. The second layer is a smoothed conditional mean with standard errors. The conditional mean is modeled by a generalized additive model (GAM) and the confidence intervals are determined by asymptotics (I assume; verify this). We will not take the smoothing function too seriously, it is simply here to roughly illustrate the conditional mean.
