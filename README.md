Applied Data Science
================

Packages
--------

Packages we'll look at today:

-   odbc / readxl / readr / dbplyr for data access
-   tidyverse for data manipulation
-   DataExplorer for helping us providing our EDA of the data
-   modelr / rsamples for sampling strategy
-   recipes for performing feature engineering
-   glmnet / glmnetUtils / h2o / FFTrees for building models
-   yardstick / broom for evaluation
-   rmarkdown for documentation

Working with databases
----------------------

We need a database connection before we can do anything with our database

``` r
library(DBI)  # talk with databases, a driver
library(odbc) # allows us to talk with DBI drivers

driver = "SQL server" # prgram that allows us to talk with the database
server = "fbmcsads.database.windows.net"
database = "WideWorldImporters-Standard"
uid = "adatumadmin"
pwd = "Pa55w.rdPa55w.rd"


con <- dbConnect(odbc(),
                 driver = driver,
                 server = server,
                 database = database,
                 uid = uid,
                 pwd = pwd)
```

Now that we have a DB connection, we can write SQL in a code chunk.

``` sql
select top 5 * from flights
```

|  year|  month|  day|  dep\_time|  sched\_dep\_time|  dep\_delay|  arr\_time|  sched\_arr\_time|  arr\_delay| carrier |  flight| tailnum | origin | dest |  air\_time|  distance|  hour|  minute| time\_hour          |
|-----:|------:|----:|----------:|-----------------:|-----------:|----------:|-----------------:|-----------:|:--------|-------:|:--------|:-------|:-----|----------:|---------:|-----:|-------:|:--------------------|
|  2013|      1|    1|        517|               515|           2|        830|               819|          11| UA      |    1545| N14228  | EWR    | IAH  |        227|      1400|     5|      15| 2013-01-01 05:00:00 |
|  2013|      1|    1|        533|               529|           4|        850|               830|          20| UA      |    1714| N24211  | LGA    | IAH  |        227|      1416|     5|      29| 2013-01-01 05:00:00 |
|  2013|      1|    1|        542|               540|           2|        923|               850|          33| AA      |    1141| N619AA  | JFK    | MIA  |        160|      1089|     5|      40| 2013-01-01 05:00:00 |
|  2013|      1|    1|        544|               545|          -1|       1004|              1022|         -18| B6      |     725| N804JB  | JFK    | BQN  |        183|      1576|     5|      45| 2013-01-01 05:00:00 |
|  2013|      1|    1|        554|               600|          -6|        812|               837|         -25| DL      |     461| N668DN  | LGA    | ATL  |        116|       762|     6|       0| 2013-01-01 06:00:00 |

We can use dbplyr to construct dplyr commands that work on the DB.

``` r
library(dbplyr) # translates into sql 
library(tidyverse) # group_by and filter and stuff in this package
```

    ## -- Attaching packages ----------------------------------------------------------------- tidyverse 1.2.1 --

    ## v ggplot2 2.2.1     v purrr   0.2.4
    ## v tibble  1.4.2     v dplyr   0.7.4
    ## v tidyr   0.8.0     v stringr 1.3.0
    ## v readr   1.1.1     v forcats 0.2.0

    ## -- Conflicts -------------------------------------------------------------------- tidyverse_conflicts() --
    ## x dplyr::filter() masks stats::filter()
    ## x dplyr::ident()  masks dbplyr::ident()
    ## x dplyr::lag()    masks stats::lag()
    ## x dplyr::sql()    masks dbplyr::sql()

``` r
flights_tbl <- tbl(con, "flights")  # talk with database con and flights table

flights_tbl %>% 
  filter(month<=6) %>%  
  group_by(origin) %>% 
  summarise(n = n(), # n is number of rows
            mean_dist = mean(distance)) %>% 
  show_query()
```

    ## <SQL>
    ## SELECT "origin", COUNT(*) AS "n", AVG("distance") AS "mean_dist"
    ## FROM "flights"
    ## WHERE ("month" <= 6.0)
    ## GROUP BY "origin"

We can also work with tables that aren't in the default schema.

``` r
purchaseorders_tbl <-tbl(con, in_schema("purchasing", "purchaseorders")) # selects purchaseorders in purchasing 

purchaseorders_tbl %>%  
  top_n(5)
```

    ## Selecting by LastEditedWhen

    ## # Source:   lazy query [?? x 12]
    ## # Database: Microsoft SQL Server
    ## #   12.00.0300[dbo@fbmcsads/WideWorldImporters-Standard]
    ##   PurchaseOrderID SupplierID OrderDate  DeliveryMethodID ContactPersonID
    ##             <int>      <int> <chr>                 <int>           <int>
    ## 1            2073          4 2016-05-31                7               2
    ## 2            2074          7 2016-05-31                2               2
    ## 3            2071          4 2016-05-30                7               2
    ## 4            2072          7 2016-05-30                2               2
    ## 5            2068          4 2016-05-27                7               2
    ## 6            2069          7 2016-05-27                2               2
    ## 7            2070          4 2016-05-28                7               2
    ## # ... with 7 more variables: ExpectedDeliveryDate <chr>,
    ## #   SupplierReference <chr>, IsOrderFinalized <lgl>, Comments <chr>,
    ## #   InternalComments <chr>, LastEditedBy <int>, LastEditedWhen <chr>

We can use the 'Id()' function from DBI to work with schema more generically within a database. This means we aren't restricted to just SELECT statements.

``` r
# error = true, we will se the error code it generated
dbGetQuery(con,"CREATE SCHEMA DBIexample5")  # insert a number for your examlpe
```

    ## Error: <SQL> 'CREATE SCHEMA DBIexample5'
    ##   nanodbc/nanodbc.cpp:1587: 42S01: [Microsoft][ODBC SQL Server Driver][SQL Server]There is already an object named 'DBIexample5' in the database.

``` r
dbWriteTable(con,"iris", iris, overwrite = TRUE)
#Read from newly written table
head(dbReadTable(con,"iris"))
```

    ##   Sepal.Length Sepal.Width Petal.Length Petal.Width Species
    ## 1          5.1         3.5          1.4         0.2  setosa
    ## 2          4.9         3.0          1.4         0.2  setosa
    ## 3          4.7         3.2          1.3         0.2  setosa
    ## 4          4.6         3.1          1.5         0.2  setosa
    ## 5          5.0         3.6          1.4         0.2  setosa
    ## 6          5.4         3.9          1.7         0.4  setosa

``` r
#Read from a table in a schema
head(dbReadTable(con, Id(schema="20774A", table = "CustomerTransactions")))
```

    ## Note: method with signature 'DBIConnection#SQL' chosen for function 'dbQuoteIdentifier',
    ##  target signature 'Microsoft SQL Server#SQL'.
    ##  "OdbcConnection#character" would also be valid

    ##                  CustomerName TransactionAmount OutstandingBalance
    ## 1             Aakriti Byrraju           2645.00                  0
    ## 2                  Bala Dixit            465.75                  0
    ## 3 Tailspin Toys (Head Office)            103.50                  0
    ## 4 Tailspin Toys (Head Office)            511.98                  0
    ## 5                Sara Huiting            809.60                  0
    ## 6                Alinne Matos            494.50                  0
    ##   TaxAmount PKIDDate TransactionDate
    ## 1    345.00 20130101      2013-01-01
    ## 2     60.75 20130101      2013-01-01
    ## 3     13.50 20130101      2013-01-01
    ## 4     66.78 20130101      2013-01-01
    ## 5    105.60 20130101      2013-01-01
    ## 6     64.50 20130101      2013-01-01

``` r
#If a write methid is supportewd by the driver, this will work
dbWriteTable(con, Id(schema="DBIexampleMitra", table = "iris", iris, overwrite = TRUE))
```

    ## Error in validObject(.Object): invalid class "Id" object: invalid object for slot "name" in class "Id": got class "list", should be or extend class "character"

Some of our code could fail in that section so we used 'error=TRUE' to be able to carry on even if some of the code error-ed. Great for optional code or things with bad connections.

Exploratory
-----------

``` r
## eval = FALSE, should evaluate this chunk of code or not, when I nit it.
flights_tbl %>% 
  as_data_frame() %>% 
  DataExplorer::GenerateReport()
```

Questions arising from the basic report:

1.  Why is there a day with double the number of flights?
2.  We need to address the high correlation between time columns
3.  Why is there negative correlation between 'flight' and 'distance'?
4.  Do we need to do anything about missings or can we just remove the rows
5.  look up why there is a peak in middle of the month?

Things to implement later in the workflow due to the EDA (explorer tree data analysis):

1.  We need to address the high correlation between time columns
2.  We need to group low freq airlines carries
3.  Bi variate for analyzing two things in relation to each other

### Answering our questions

> Why is there a day with double the number of flights?

Are there duplicate rows?

``` r
flights_tbl %>% 
  filter(day == 15) %>% 
  distinct()  %>%  
  summarise(n())  %>%  # count the data
  as_data_frame() ->  # force it to give it the content and not sql code
  distict_count  # create a uniqe list of using distict count
  
  # get all rows if dublicate or not
flights_tbl %>% 
  filter(day == 15) %>% 
  summarise(n())  %>%  
  as_data_frame() ->
  row_count 

# if the sructure and values are the same return true,
identical(row_count, distict_count) #one row per observed flight?
```

    ## [1] TRUE

But are the number of rows unusual?

``` r
library(ggplot2)
flights_tbl %>% 
  group_by(day) %>% 
  summarise(n = n(), n_distinct(flight)) %>% 
  arrange(day)
```

    ## # Source:     lazy query [?? x 3]
    ## # Database:   Microsoft SQL Server
    ## #   12.00.0300[dbo@fbmcsads/WideWorldImporters-Standard]
    ## # Ordered by: day
    ##      day     n `n_distinct(flight)`
    ##    <int> <int>                <int>
    ##  1     1 11036                 2532
    ##  2     2 10808                 2542
    ##  3     3 11211                 2491
    ##  4     4 11059                 2449
    ##  5     5 10858                 2463
    ##  6     6 11059                 2484
    ##  7     7 10985                 2427
    ##  8     8 11271                 2436
    ##  9     9 10857                 2496
    ## 10    10 11227                 2504
    ## # ... with more rows

``` r
## to plot the data instead : 
flights_tbl %>% 
  group_by(day) %>% 
  summarise(n = n(), n_distinct(flight)) %>%
  as_data_frame() %>% 
  ggplot(aes(day,y = n)) + geom_col()  # does not do any binning of the data
```

![](README_files/figure-markdown_github/unnamed-chunk-8-1.png) Data is fine, the problem is in our visualization. Doing histogram, split continues numbers have to group them two in a day. the spike looks to be a problem but its not.

Looks like the jump in the histogram is an artifact of binning the data. s'oh!

### Bivariate analysis

``` r
flights_tbl %>% 
  select_if(is.numeric) %>% 
  as_data_frame() %>%   # need to convert to a data frame from sql data
  gather(col,val,-dep_delay) %>% # dont pivot the dep_delay column, gather and pivot creates the variables col and val
  filter(col!= "arr_delay", dep_delay < 500) %>%  # filter out arr_delay to see better result
  ggplot(aes(x=val, y=dep_delay)) + 
  #geom_point() +  # this takes long time since its plotting row by row
  geom_bin2d() +
  facet_wrap(~col, scales = "free") + # takes different parts of our data to produce them as charts
  scale_fill_gradientn(colours= viridisLite::viridis(256, option = "D"))
```

    ## Applying predicate on the first 100 rows

    ## Warning: Removed 1631 rows containing non-finite values (stat_bin2d).

    ## Warning: Computation failed in `stat_bin2d()`:
    ## 'from' must be a finite number

![](README_files/figure-markdown_github/unnamed-chunk-9-1.png)

### Sampling

Our options for sampling data with large class imbalance are:

-   Down sampling takes as many majority rows and there are minority rows
    -   No over fit from individual rows
    -   Can drastically reduce training data size
-   Up sampling or sampling repeats minority rows until they meet some defined class ratio
    -   Risks over fitting
    -   Doesn't reduce training data set
-   Synthesizing data makes extra records that are like the minority class
    -   Does not reduce training set
    -   Avoids some of the over fit risk of up sampling
    -   Can weaken predictions if minority data is very similar to majority
    -   synthpop is a package used.

We need to think about whether we need to k-fold cross-validation explicitly.

-   Run the same model and assess robustness of the coefficients
-   We have an algorithm that needs explicit cross validation because it does not do it internally
-   When we are going to run lots of models with hyper-parameter tuning so the results are more consistent.

We use bootstrapping when we want to fit a single model and ensure the results are robust. This will often do many more iterations than k-fold cross validation, making it better in cases where there is relatively small amounts of data. (Bootstrapping is used to arrive at a single robust model)

Packages we can use for sampling include:

-   modelr which facilitates boostrap and cross validation strategies
-   rsample allows us to bootstrap and perform a wide variety of cross validation tasks
-   recipes allows us to up sample and down sample
-   synthpop allows us to build synthesized samples

More notes: - we have many underlays (= majority of the data), how can we predict when the flights are late, now it will always say that the flights are on time... what to do? - Down sampling = if we have alot of data, take randome small sample of the not delayed majority class data and still have a huge amount of data, having more data of delayed. - upsampling = a model to predict 3 % that are delay, repeat those rows which are delays, known minority class rows, until we have 15 - 20% delays. - synthesize data, is the third option

Hyperparameter tunings - Use training data - Split our data into training and test set. 4/5 split,

Need to test different models to select the one that gives the best result.

### Install

from github, it is CRAN but we have an earlier version
======================================================

run in console &gt;
===================

install.packages("devtools")
============================

devtools::install\_github("topepo/recipes")
===========================================

### Practical

First we need to split our data into test and train.

``` r
flights_tbl %>% 
  as_data_frame() ->
  flights

flights %>% 
  mutate(was_delayed = ifelse(arr_delay>5, "Delayed", "Not Delayed"), week = ifelse(day %/% 7 >3, 3, day %/% 7))  -> # flight delayed with 5 min & %/% makes absolute division (decimal < 1 becomes 0 when doing division and 1 when one 7 fits to the value, 2 for 2 7:ns osv..), we create weeks, 
  flights

flights %>% 
  modelr::resample_partition(c(train=0.7, test =0.3)) ->
  splits

splits %>% 
  pluck("train") %>% 
  as_data_frame() ->
  train_raw

splits %>% 
  pluck("test") %>% 
  as_data_frame() ->
  test_raw
```

Druring the investigation, we will look at the impact of upsampling. We will see it in action in a bit. First prepping our basic features!

``` r
library(recipes)
```

    ## Loading required package: broom

    ## 
    ## Attaching package: 'recipes'

    ## The following object is masked from 'package:stringr':
    ## 
    ##     fixed

    ## The following object is masked from 'package:stats':
    ## 
    ##     step

``` r
basic_fe <- recipe(train_raw, was_delayed ~ .)  # ~ . = by everything to predict was_delayed data # feature engineering

basic_fe %>% 
  step_rm(ends_with("time"), ends_with("delay"), 
          year, day, minute, time_hour, tailnum, flight) %>% 
#  step_corr(all_predictors()) %>%  # remove highly corr variables
  step_zv(all_predictors()) %>% # remove parameters with near zero variance
  step_nzv(all_predictors()) %>%  # drop param. if freq is less than 5 % that has differences
  step_naomit(all_predictors()) %>% #remove recors that are NA, because they are only 3%
  step_naomit(all_outcomes()) %>%
  step_other(all_nominal(), threshold = 0.03) ->  # if they have values with low incident rate to a low category, other category. nominal = categorical variable
 # step_discretize(month, day) ->  #convert from numbers to category variables
  colscleaned_fe
  
colscleaned_fe
```

    ## Data Recipe
    ## 
    ## Inputs:
    ## 
    ##       role #variables
    ##    outcome          1
    ##  predictor         20
    ## 
    ## Operations:
    ## 
    ## Delete terms ends_with("time"), ends_with("delay"), year, ...
    ## Zero variance filter on all_predictors()
    ## Sparse, unbalanced variable filter on all_predictors()
    ## Removing rows with NA values in all_predictors()
    ## Removing rows with NA values in all_outcomes()
    ## Collapsing factor levels for all_nominal()

``` r
colscleaned_fe <- prep(colscleaned_fe, verbose = TRUE)
```

    ## oper 1 step rm [training] 
    ## oper 2 step zv [training] 
    ## oper 3 step nzv [training] 
    ## oper 4 step naomit [training] 
    ## oper 5 step naomit [training] 
    ## oper 6 step other [training]

``` r
colscleaned_fe
```

    ## Data Recipe
    ## 
    ## Inputs:
    ## 
    ##       role #variables
    ##    outcome          1
    ##  predictor         20
    ## 
    ## Training data contained 235743 data points and 6627 incomplete rows. 
    ## 
    ## Operations:
    ## 
    ## Variables removed dep_time, sched_dep_time, arr_time, ... [trained]
    ## Zero variance filter removed no terms [trained]
    ## Sparse, unbalanced variable filter removed no terms [trained]
    ## Removing rows with NA values in all_predictors()
    ## Removing rows with NA values in all_outcomes()
    ## Collapsing factor levels for carrier, origin, dest, was_delayed [trained]

``` r
# do what we prepped to do
train_prep1 <- bake(colscleaned_fe, train_raw)
```

Now we need to process our numeric variables.

``` r
colscleaned_fe %>% 
  step_num2factor(month, week, hour) %>% # factors the numbers into classifications
  step_rm(tailnum) %>% #hack! now removed!
  step_log(distance) ->
  numscleaned_fe

numscleaned_fe <- prep(numscleaned_fe, verbose = TRUE)
```

    ## oper 1 step rm [pre-trained]
    ## oper 2 step zv [pre-trained]
    ## oper 3 step nzv [pre-trained]
    ## oper 4 step naomit [pre-trained]
    ## oper 5 step naomit [pre-trained]
    ## oper 6 step other [pre-trained]
    ## oper 7 step num2factor [training] 
    ## oper 8 step rm [training] 
    ## oper 9 step log [training]

``` r
numscleaned_fe
```

    ## Data Recipe
    ## 
    ## Inputs:
    ## 
    ##       role #variables
    ##    outcome          1
    ##  predictor         20
    ## 
    ## Training data contained 235743 data points and 6627 incomplete rows. 
    ## 
    ## Operations:
    ## 
    ## Variables removed dep_time, sched_dep_time, arr_time, ... [trained]
    ## Zero variance filter removed no terms [trained]
    ## Sparse, unbalanced variable filter removed no terms [trained]
    ## Removing rows with NA values in all_predictors()
    ## Removing rows with NA values in all_outcomes()
    ## Collapsing factor levels for carrier, origin, dest, was_delayed [trained]
    ## Factor variables from month, week, hour [trained]
    ## Variables removed tailnum [trained]
    ## Log transformation on distance [trained]

``` r
train_prep1 <- bake(numscleaned_fe, train_raw)
```

W00t its upsampling time!

``` r
# mean(train_prep1$was_delayed,na.rm =TRUE)  # just to check the mean
numscleaned_fe %>% 
  step_upsample(all_outcomes(), ratio = 1.) %>%  # increase to 50 % ratio between 
  prep(retain = TRUE) %>% 
  juice() %>% 
  #hack because juice is not reducing the column set
  bake(numscleaned_fe, .) ->
  train_prep2
```

Building models
---------------

Decide which types of models you want to consider -- perhaps using Microsoft's lovely [cheat sheet](https://docs.microsoft.com/en-us/azure/machine-learning/studio/algorithm-cheat-sheet). Then determine if any need any special processing to the data beyond what you have done so far.

### A basic logistic regression model

``` r
glm_unbal <- glm(was_delayed~ . -1,"binomial", data = train_prep1) # "was_delayed"" = input variable AND "~ ."" = output variable
glm_bal <- glm(was_delayed~ . -1, "binomial", data = train_prep2) # general linearized model, general regression model
```

Then we can see how these models are constructed and how they perform

``` r
library(broom)
glance(glm_unbal)  # model obj linear model, coeff values, how we change prop of an outcome...
```

    ##   null.deviance df.null    logLik      AIC      BIC deviance df.residual
    ## 1      118009.7   85126 -49955.19 100014.4 100500.7 99910.39       85074

``` r
# BIC = how much information we are collecting
# logLik = measure of what predicted and what happened ?? come back to this later
```

Get the coefficients of the model

``` r
tidy(glm_unbal)
```

    ##         term    estimate  std.error     statistic      p.value
    ## 1     month1  0.29425891 4.20831411   0.069923228 9.442548e-01
    ## 2    month10  0.63502820 4.20820993   0.150902215 8.800529e-01
    ## 3    month11  0.52336367 4.20814728   0.124369143 9.010230e-01
    ## 4    month12 -0.34873287 4.20819761  -0.082869890 9.339550e-01
    ## 5     month2  0.12649737 4.20810752   0.030060393 9.760189e-01
    ## 6     month3  0.26077905 4.20822706   0.061968864 9.505876e-01
    ## 7     month4 -0.02498763 4.20821342  -0.005937825 9.952623e-01
    ## 8     month5  0.38984130 4.20814166   0.092639777 9.261897e-01
    ## 9     month6 -0.15368069 4.20808978  -0.036520297 9.708675e-01
    ## 10    month7 -0.18594769 4.20827421  -0.044186210 9.647560e-01
    ## 11    month8  0.14146003 4.20824519   0.033614968 9.731842e-01
    ## 12    month9  0.94538519 4.20830024   0.224647754 8.222533e-01
    ## 13 carrierAA  0.38744395 0.06350483   6.101015565 1.053966e-09
    ## 14 carrierB6 -0.08142250 0.06301651  -1.292082080 1.963287e-01
    ## 15 carrierDL  0.44842990 0.06348505   7.063551707 1.622998e-12
    ## 16 carrierEV -0.25302041 0.07610555  -3.324598395 8.854598e-04
    ## 17 carrierMQ -0.25932517 0.06967655  -3.721843135 1.977739e-04
    ## 18 carrierUA  0.12494876 0.06450648   1.936995468 5.274590e-02
    ## 19 carrierUS  0.39278124 0.07148829   5.494344149 3.921651e-08
    ## 20 carrierWN -0.83419714 0.32524851  -2.564799241 1.032355e-02
    ## 21 originJFK  0.01005491 0.02445096   0.411227446 6.809058e-01
    ## 22 originLGA -0.02080014 0.02357197  -0.882409937 3.775552e-01
    ## 23   destBOS  0.77837976 0.87259564   0.892028022 3.723779e-01
    ## 24   destCLT  0.15203400 0.20856867   0.728939766 4.660385e-01
    ## 25   destFLL  0.11908087 0.22038350   0.540334791 5.889662e-01
    ## 26   destLAX  0.28799439 0.75090473   0.383529862 7.013269e-01
    ## 27   destMCO  0.32567151 0.14197536   2.293859363 2.179858e-02
    ## 28   destMIA  0.25095647 0.23314121   1.076414011 2.817421e-01
    ## 29   destORD  0.33386411 0.04519274   7.387560808 1.495467e-13
    ## 30   destSFO  0.15209325 0.77795265   0.195504508 8.449980e-01
    ## 31  distance  0.07579355 0.63508759   0.119343461 9.050033e-01
    ## 32    hour11  0.11978778 0.05117889   2.340570141 1.925432e-02
    ## 33    hour12 -0.13045649 0.04626323  -2.819874188 4.804248e-03
    ## 34    hour13 -0.19088179 0.04620127  -4.131526580 3.603620e-05
    ## 35    hour14 -0.30148700 0.04637223  -6.501455767 7.954643e-11
    ## 36    hour15 -0.53679355 0.04353068 -12.331385694 6.138198e-35
    ## 37    hour16 -0.50112002 0.04563982 -10.979887026 4.775206e-28
    ## 38    hour17 -0.66003779 0.04357935 -15.145656441 8.094237e-52
    ## 39    hour18 -0.75934978 0.04508545 -16.842456112 1.191852e-63
    ## 40    hour19 -0.66157746 0.04585553 -14.427429117 3.478056e-47
    ## 41    hour20 -0.68078456 0.04797995 -14.188937989 1.072710e-45
    ## 42    hour21 -0.63885034 0.05168246 -12.361067183 4.244836e-35
    ## 43    hour22 -0.87196712 0.13561651  -6.429653064 1.278955e-10
    ## 44    hour23 -0.97042535 0.27770119  -3.494494727 4.749600e-04
    ## 45     hour5  0.54341998 0.11906215   4.564170846 5.014723e-06
    ## 46     hour6  0.39180333 0.04441829   8.820766535 1.136788e-18
    ## 47     hour7  0.36107780 0.04702018   7.679209127 1.600737e-14
    ## 48     hour8  0.18133259 0.04584929   3.954970596 7.654407e-05
    ## 49     hour9  0.14471633 0.04616087   3.135043460 1.718287e-03
    ## 50     week1 -0.34627908 0.02365766 -14.637077716 1.629216e-48
    ## 51     week2 -0.10908877 0.02391284  -4.561932473 5.068495e-06
    ## 52     week3 -0.16670648 0.02206819  -7.554152605 4.215956e-14

Takes original data and suppliment with predicte data and predicted error and associate with those...????

``` r
head(augment(glm_unbal))
```

    ##   .rownames was_delayed month carrier origin dest distance hour week
    ## 1         3 Not Delayed     1      DL    LGA  ATL 6.635947    6    0
    ## 2         4     Delayed     1      UA    EWR  ORD 6.577861    5    0
    ## 3         5     Delayed     1      B6    EWR  FLL 6.970730    6    0
    ## 4         6 Not Delayed     1      B6    JFK  MCO 6.850126    6    0
    ## 5         7     Delayed     1      AA    LGA  ORD 6.597146    6    0
    ## 6         8     Delayed     1      UA    JFK  LAX 7.813996    6    0
    ##    .fitted    .se.fit     .resid         .hat   .sigma      .cooksd
    ## 1 1.616654 0.04878218  0.6018679 0.0003289257 1.083699 1.256829e-06
    ## 2 1.795051 0.12095496 -1.9741997 0.0017872322 1.083680 2.076406e-04
    ## 3 1.252057 0.05048571 -1.7340867 0.0004407073 1.083685 2.966820e-05
    ## 4 1.459562 0.04912918  0.6463950 0.0003692665 1.083699 1.651117e-06
    ## 5 1.886591 0.05042784 -2.0138187 0.0002906771 1.083679 3.689743e-05
    ## 6 1.701311 0.05196149 -1.9333362 0.0003523159 1.083681 3.716249e-05
    ##   .std.resid
    ## 1  0.6019669
    ## 2 -1.9759663
    ## 3 -1.7344689
    ## 4  0.6465144
    ## 5 -2.0141114
    ## 6 -1.9336769

Plot predictive's vs actuals

``` r
glm_unbal %>% 
  augment() %>% 
  ggplot(aes(x=.fitted, group=was_delayed, fill= was_delayed)) + 
  geom_density(alpha=0.5) + 
  geom_vline(aes(xintercept=0))
```

![](README_files/figure-markdown_github/unnamed-chunk-18-1.png)

``` r
# if its x>0, then its more than 50 % chance that is not delayed. 
# if x<0 then its more than 50 % chance that its delayed.
```

#### Prep and predict on test data

``` r
test_raw %>% 
  bake(numscleaned_fe, .) %>% 
  modelr:: add_predictions(glm_unbal,var = "glm_unbal") ->
  test_scored
```

``` r
test_scored %>% 
  ggplot(aes(x=glm_unbal, group=was_delayed, fill= was_delayed)) + 
  geom_density(alpha=0.5) + 
  geom_vline(aes(xintercept=0))
```

    ## Warning: Removed 61871 rows containing non-finite values (stat_density).

![](README_files/figure-markdown_github/unnamed-chunk-20-1.png)

``` r
# if its x>0, then its more than 50 % chance that is not delayed. 
# if x<0 then its more than 50 % chance that its delayed.
```

But how many did we get right etc?

``` r
library(yardstick)
```

    ## 
    ## Attaching package: 'yardstick'

    ## The following object is masked from 'package:readr':
    ## 
    ##     spec

``` r
test_scored %>% 
  mutate(glm_unbal_class = as.factor(ifelse(glm_unbal<0, "Delayed", "Not Delayed"))) %>% 
  conf_mat(was_delayed, glm_unbal_class)
```

    ##              Truth
    ## Prediction    Delayed Not Delayed
    ##   Delayed        1970        1444
    ##   Not Delayed    9584       23361

``` r
test_scored %>% 
  mutate(glm_unbal_class = as.factor(
    ifelse(glm_unbal<0, "Delayed", "Not Delayed"))) %>% 
  accuracy(was_delayed, glm_unbal_class)
```

    ## [1] 0.6966913
