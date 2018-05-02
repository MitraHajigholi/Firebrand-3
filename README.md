Applied Data Science
================

Packages
--------

Packages we'll look at today:

-   odbc / readxl / readr / dbplyr for data access
-   tidyverse for data manipulation
-   DataExplorer for providing of our data
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

    ## -- Attaching packages ---------------------------------------------------------------- tidyverse 1.2.1 --

    ## v ggplot2 2.2.1     v purrr   0.2.4
    ## v tibble  1.4.2     v dplyr   0.7.4
    ## v tidyr   0.8.0     v stringr 1.3.0
    ## v readr   1.1.1     v forcats 0.2.0

    ## -- Conflicts ------------------------------------------------------------------- tidyverse_conflicts() --
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

Some of our code could fail in that section so we used 'error=TRUE' to be able to carry on even if some of the code errored. Great for optional code or things with bad connections.

Exploratory
-----------

``` r
## eval = FALSE, should evaluate this chunk of code or not, when I nit it.
flights_tbl %>% 
  as_data_frame() %>% 
  DataExplorer::GenerateReport()
```
