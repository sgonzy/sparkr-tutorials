# Subsetting DataFrames
Sarah Armstrong, Urban Institute  
July 1, 2016  




**Objective**: Now that we understand what a SparkR DataFrame (DF) really is (remember, it's not actually data!) and can write expressions using essential DataFrame operations, such as `agg`, we are ready to start subsetting DFs using more advanced transformation operations. This tutorial discusses various ways of subsetting DFs, as well as how to work with a randomly sampled subset as a local data.frame in RStudio:

* Subset a DF by row
* Subset a DF by a list of columns
* Subset a DF by column expressions
* Drop a column from a DF
* Subset a DF by taking a random sample
* Collect a random sample as a local data.frame
* Export a data.frame as a single .csv file

**SparkR/R Operations Discussed**: `filter`, `where`, `select`, `sample`, `collect`, `write.table`

***

<span style="color:red">**Warning**</span>: Before beginning this tutorial, please visit the SparkR Tutorials README file (found [here](https://github.com/UrbanInstitute/sparkr-tutorials/blob/master/README.md)) in order to load the SparkR library and subsequently initiate your SparkR and SparkR SQL contexts.



You can confirm that you successfully initiated these contexts by looking at the global environment of RStudio. Only proceed if you can see `sc` and `sqlContext` listed as values in the global environment or RStudio.

***

**Read in initial data as DF**: Throughout this tutorial, we will use the loan performance example dataset that we exported at the conclusion of the SparkR Basics I tutorial. Note that we are __persisting__ the DataFrame since we will use it throughout this tutorial.


```r
df <- read.df(sqlContext, "s3://sparkr-tutorials/hfpc_ex", header='false', inferSchema='true')
cache(df)
```

_Note_: documentation for the quarterly loan performance data can be found at http://www.fanniemae.com/portal/funding-the-market/data/loan-performance-data.html.

Let's check the dimensions our DF `df` and its column names so that we can compare the dimension sizes of `df` with those of the subsets that we will  define throughout this tutorial:


```r
nrow(df)
## [1] 13216516
ncol(df)
## [1] 14
columns(df)
##  [1] "loan_id"       "period"        "servicer_name" "new_int_rt"   
##  [5] "act_endg_upb"  "loan_age"      "mths_remng"    "aj_mths_remng"
##  [9] "dt_matr"       "cd_msa"        "delq_sts"      "flag_mod"     
## [13] "cd_zero_bal"   "dt_zero_bal"
```

***


### Subset DataFrame by row:

The SparkR operation `filter` allows us to subset the rows of a DF according to specified conditions. Before we begin working with `filter` to see how it works, let's print the schema of `df` since the types of subsetting conditions we are able to specify depend on the datatype of each column in the DF: 


```r
printSchema(df)
## root
##  |-- loan_id: long (nullable = true)
##  |-- period: string (nullable = true)
##  |-- servicer_name: string (nullable = true)
##  |-- new_int_rt: double (nullable = true)
##  |-- act_endg_upb: double (nullable = true)
##  |-- loan_age: integer (nullable = true)
##  |-- mths_remng: integer (nullable = true)
##  |-- aj_mths_remng: integer (nullable = true)
##  |-- dt_matr: string (nullable = true)
##  |-- cd_msa: integer (nullable = true)
##  |-- delq_sts: string (nullable = true)
##  |-- flag_mod: string (nullable = true)
##  |-- cd_zero_bal: integer (nullable = true)
##  |-- dt_zero_bal: string (nullable = true)
```

We can subset `df` into a new DF, `f1`, that includes only those loans for which JPMorgan Chase is the servicer with the expression:


```r
f1 <- filter(df, df$servicer_name == "JP MORGAN CHASE BANK, NA" | df$servicer_name == "JPMORGAN CHASE BANK, NA" | df$servicer_name == "JPMORGAN CHASE BANK, NATIONAL ASSOCIATION")
nrow(f1)
## [1] 102733
```

Notice that the `filter` considers normal logical syntax (e.g. logical conditions and operations), making working with the operation very straightforward. We can specify `filter` with SQL statement strings. For example, here we have the preceding example written in SQL statement format:


```r
filter(df, "servicer_name = 'JP MORGAN CHASE BANK, NA' or servicer_name = 'JPMORGAN CHASE BANK, NA' or servicer_name = 'JPMORGAN CHASE BANK, NATIONAL ASSOCIATION'")
```

Or, alternatively, in a syntax similar to how we subset data.frames by row in base R:


```r
df[df$servicer_name == "JP MORGAN CHASE BANK, NA" | df$servicer_name == "JPMORGAN CHASE BANK, NA" | df$servicer_name == "JPMORGAN CHASE BANK, NATIONAL ASSOCIATION",]
```

Another example of using logical syntax with `filter` is that we can subset `df` such that the new DF only includes those loans for which the servicer name is known, i.e. the column `"servicer_name"` is not equa to an empty string or listed as `"OTHER"`:


```r
f2 <- filter(df, df$servicer_name != "OTHER" & df$servicer_name != "")
nrow(f2)
## [1] 226264
```

Or, if we wanted to only consider observations with a `"loan_age"` value of greater than 60 months (five years), we would evaluate:


```r
f3 <- filter(df, df$loan_age > 60)
nrow(f3)
## [1] 1714413
```

An alias for `filter` is `where`, which reads much more intuitively, particularly when `where` is embedded in a complex statement. For example, the following expression can be read as "aggregate and return the mean loan age and count values for observations in `df` where loan age is less than 60 months":


```r
f7 <- agg(groupBy(where(df, df$loan_age < 60), where(df, df$loan_age < 60)$servicer_name), loan_age_avg = avg(where(df, df$loan_age < 60)$loan_age), count = n(where(df, df$loan_age < 60)$loan_age))
head(f7)
##                                servicer_name loan_age_avg count
## 1                IRWIN MORTGAGE, CORPORATION     38.84615    13
## 2 FIRST TENNESSEE BANK, NATIONAL ASSOCIATION     23.45820 12774
## 3                         FLAGSTAR BANK, FSB     42.82895    76
## 4                   PHH MORTGAGE CORPORATION     20.77649  8872
## 5  JPMORGAN CHASE BANK, NATIONAL ASSOCIATION     21.32419 46917
## 6                  USAA FEDERAL SAVINGS BANK     20.35909  3080
```

***


### Subset DataFrame by column:

The operation `select` allows us to subset a DF by a specified list of columns. In the expression below, for example, we create a subsetted DF that includes only the number of calendar months remaining until the borrower is expected to pay the mortgage loan in full (remaining maturity) and adjusted remaining maturity:


```r
s1 <- select(df, "mths_remng", "aj_mths_remng")
ncol(s1)
## [1] 2
```

We can also reference the column names through the DF name, i.e. `select(df, df$mths_remng, df$aj_mths_remng)`. Or, we can save a list of columns as a combination of strings. If we wanted to make a list of all columns that relate to remaining maturity, we could evaluate the expression `remng_mat <- c("mths_remng", "aj_mths_remng")` and then easily reference our list of columns later on with `select(df, remng_mat)`.


Besides subsetting by a list of columns, we can also subset `df` by column expressions, or by both as we do in the example below. The DF `s2` includes the columns `"mths_remng"` and `"aj_mths_remng"` as in `s1`, but now with a column that lists the absolute value of the difference between the unadjusted and adjusted remaining maturity:


```r
s2 <- select(df, df$mths_remng, df$aj_mths_remng, abs(df$aj_mths_remng - df$mths_remng))
ncol(s2)
## [1] 3
head(s2)
##   mths_remng aj_mths_remng abs((aj_mths_remng - mths_remng))
## 1        360           359                                 1
## 2        359           358                                 1
## 3        358           357                                 1
## 4        357           356                                 1
## 5        356           355                                 1
## 6        355           355                                 0
```

Note that, just as we can subset by row with syntax similar to that in base R, we can similarly acheive subsetting by column. The following expressions are equivalent:


```r
select(df, df$period)
df[,"period"]
df[,2]
```

To simultaneously subset by column and row specifications, you can simply embed a `where` expression in a `select` operation (or vice versa). The following expression creates a DF that lists loan age values only for observations in which servicer name is unknown:


```r
s3 <- select(where(df, df$servicer_name == "" | df$servicer_name == "OTHER"), "loan_age")
head(s3)
##   loan_age
## 1        0
## 2        1
## 3        2
## 4        3
## 5        4
## 6        5
```

Note that we could have also written the above expression as `df[df$servicer_name == "" | df$servicer_name == "OTHER", "loan_age"]`.


#### Drop a column from a DF:

We can drop a column from a DF very simply by assigning `NULL` to a DF column. Below, we drop `"aj_mths_remng"` from `s1`:


```r
head(s1)
##   mths_remng aj_mths_remng
## 1        360           359
## 2        359           358
## 3        358           357
## 4        357           356
## 5        356           355
## 6        355           355
s1$aj_mths_remng <- NULL
head(s1)
##   mths_remng
## 1        360
## 2        359
## 3        358
## 4        357
## 5        356
## 6        355
```

***


### Subset a DF by taking a random sample:

Perhaps the most useful subsetting operation is `sample`, which returns a randomly sampled subset of a DF. With `subset`, we can specify whether we want to sample with or without replace, the approximate size of the sample that we want the new DF to call and whether or not we want to define a random seed. If our initial DF is so massive that performing analysis on the entire dataset requires a more expensive cluster, we can: sample the massive dataset, interactively develop our analysis in SparkR using our sample and then evaluate the resulting program using our initial DF, which calls the entire massive dataset, only as is required. This strategy will help us to minimize wasting resources.

Below, we take a random sample of `df` without replacement that is, in size, approximately equal to 1% of `df`. Notice that we must define a random seed in order to be able to reproduce our random sample.


```r
df_samp1 <- sample(df, withReplacement = FALSE, fraction = 0.01)  # Without set seed
df_samp2 <- sample(df, withReplacement = FALSE, fraction = 0.01)
count(df_samp1)
## [1] 131811
count(df_samp2)
## [1] 132152

df_samp3 <- sample(df, withReplacement = FALSE, fraction = 0.01, seed = 0)  # With set seed
df_samp4 <- sample(df, withReplacement = FALSE, fraction = 0.01, seed = 0)
count(df_samp3)
## [1] 132070
count(df_samp4)
## [1] 132070
```


#### Collect a random sample as a local data.frame:

An additional use of `sample` is to collect a random sample of a massive dataset as a local data.frame in R. This would allow us to work with a sample dataset in a traditional analysis environment that is likely more representative of the population since we are sampling from a larger set of observations than we are normally doing so. This can be achieved by simply using `collect` to create a local data.frame:


```r
typeof(df_samp4)  # DFs are of class S4
## [1] "S4"
dat <- collect(df_samp4)
typeof(dat)
## [1] "list"
```

If we want to export this data.frame from RStudio as a single .csv file that we can work with in any environment, we can export the data as we normally do in R:


```r
write.table(dat, file = "hfpc_samp.csv",row.names=FALSE, na="",col.names=FALSE, sep=",")
```

__Warning__: we cannot collect a DF as a data.frame unless it is sufficiently small in size since it must fit onto a single node!

__End of tutorial__ - Next up is [Insert next tutorial]