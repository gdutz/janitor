Overview of janitor functions
================
2020-04-07

-   [Major functions](#major-functions)
    -   [Cleaning](#cleaning)
        -   [Clean data.frame names with `clean_names()`](#clean-data.frame-names-with-clean_names)
        -   [Do those data.frames actually contain the same columns?](#do-those-data.frames-actually-contain-the-same-columns)
    -   [Exploring](#exploring)
        -   [`tabyl()` - a better version of `table()`](#tabyl---a-better-version-of-table)
        -   [Explore records with duplicated values for specific combinations of variables with `get_dupes()`](#explore-records-with-duplicated-values-for-specific-combinations-of-variables-with-get_dupes)
-   [Minor functions](#minor-functions)
    -   [Cleaning](#cleaning-1)
        -   [Manipulate vectors of names with `make_clean_names()`](#manipulate-vectors-of-names-with-make_clean_names)
        -   [`remove_empty()` rows and columns](#remove_empty-rows-and-columns)
        -   [`remove_constant()` columns](#remove_constant-columns)
        -   [Directionally-consistent rounding behavior with `round_half_up()`](#directionally-consistent-rounding-behavior-with-round_half_up)
        -   [Round decimals to precise fractions of a given denominator with `round_to_fraction()`](#round-decimals-to-precise-fractions-of-a-given-denominator-with-round_to_fraction)
        -   [Fix dates stored as serial numbers with `excel_numeric_to_date()`](#fix-dates-stored-as-serial-numbers-with-excel_numeric_to_date)
        -   [Convert a mix of date and datetime formats to date](#convert-a-mix-of-date-and-datetime-formats-to-date)
        -   [Elevate column names stored in a data.frame row](#elevate-column-names-stored-in-a-data.frame-row)
    -   [Exploring](#exploring-1)
        -   [Count factor levels in groups of high, medium, and low with `top_levels()`](#count-factor-levels-in-groups-of-high-medium-and-low-with-top_levels)

The janitor functions expedite the initial data exploration and cleaning that comes with any new data set. This catalog describes the usage for each function.

Major functions
===============

Functions for everyday use.

Cleaning
--------

### Clean data.frame names with `clean_names()`

Call this function every time you read data.

It works in a `%>%` pipeline, and handles problematic variable names, especially those that are so well-preserved by `readxl::read_excel()` and `readr::read_csv()`.

-   Parses letter cases and separators to a consistent format.
    -   Default is to snake\_case, but other cases like camelCase are available
-   Handles special characters and spaces, including transliterating characters like `œ` to `oe`.
-   Appends numbers to duplicated names
-   Converts "%" to "percent" and "\#" to "number" to retain meaning
-   Spacing (or lack thereof) around numbers is preserved

``` r
# Create a data.frame with dirty names
test_df <- as.data.frame(matrix(ncol = 6))
names(test_df) <- c("firstName", "ábc@!*", "% successful (2009)",
                    "REPEAT VALUE", "REPEAT VALUE", "")
```

Clean the variable names, returning a data.frame:

``` r
test_df %>%
  clean_names()
#>   first_name abc percent_successful_2009 repeat_value repeat_value_2  x
#> 1         NA  NA                      NA           NA             NA NA
```

Compare to what base R produces:

``` r
make.names(names(test_df))
#> [1] "firstName"            "ábc..."               "X..successful..2009."
#> [4] "REPEAT.VALUE"         "REPEAT.VALUE"         "X"
```

This function is powered by the underlying exported function **`make_clean_names()`**, which accepts and returns a character vector of names (see below). This allows for cleaning the names of *any* object, not just a data.frame. `clean_names()` is retained for its convenience in piped workflows, and can be called on an `sf` simple features object or a `tbl_graph` tidygraph object in addition to a data.frame.

### Do those data.frames actually contain the same columns?

#### Check with `compare_df_cols()`

For cases when you are given a set of data files that *should* be identical, and you wish to read and combine them for analysis. But then `dplyr::bind_rows()` or `rbind()` fails, because of different columns or because the column classes don't match across data.frames.

`compare_df_cols()` takes unquoted names of data.frames / tibbles, or a list of data.frames, and returns a summary of how they compare. See what the column types are, which are missing or present in the different inputs, and how column types differ.

``` r
df1 <- data.frame(a = 1:2, b = c("big", "small")) # a factor by default
df2 <- data.frame(a = 10:12, b = c("medium", "small", "big"), c = 0, stringsAsFactors = FALSE)
df3 <- df1 %>%
  dplyr::mutate(b = as.character(b))

compare_df_cols(df1, df2, df3)
#>   column_name     df1       df2       df3
#> 1           a integer   integer   integer
#> 2           b  factor character character
#> 3           c    <NA>   numeric      <NA>

compare_df_cols(df1, df2, df3, return = "mismatch")
#>   column_name    df1       df2       df3
#> 1           b factor character character
compare_df_cols(df1, df2, df3, return = "mismatch", bind_method = "rbind") # default is dplyr::bind_rows
#>   column_name    df1       df2       df3
#> 1           b factor character character
#> 2           c   <NA>   numeric      <NA>
```

`compare_df_cols_same()` returns `TRUE` or `FALSE` indicating if the data.frames can be successfully row-bound with the given binding method:

``` r
compare_df_cols_same(df1, df3)
#>   column_name    ..1       ..2
#> 1           b factor character
#> [1] FALSE
compare_df_cols_same(df2, df3)
#> [1] TRUE
```

Exploring
---------

### `tabyl()` - a better version of `table()`

`tabyl()` is a tidyverse-oriented replacement for `table()`. It counts combinations of one, two, or three variables, and then can be formatted with a suite of `adorn_*` functions to look just how you want. For instance:

``` r
mtcars %>%
  tabyl(gear, cyl) %>%
  adorn_totals("col") %>%
  adorn_percentages("row") %>%
  adorn_pct_formatting(digits = 2) %>%
  adorn_ns() %>%
  adorn_title()
#>              cyl                                    
#>  gear          4          6           8        Total
#>     3  6.67% (1) 13.33% (2) 80.00% (12) 100.00% (15)
#>     4 66.67% (8) 33.33% (4)  0.00%  (0) 100.00% (12)
#>     5 40.00% (2) 20.00% (1) 40.00%  (2) 100.00%  (5)
```

Learn more in the [tabyls vignette](http://sfirke.github.io/janitor/articles/tabyls.html).

### Explore records with duplicated values for specific combinations of variables with `get_dupes()`

This is for hunting down and examining duplicate records during data cleaning - usually when there shouldn't be any.

For example, in a tidy data.frame you might expect to have a unique ID repeated for each year, but no duplicated pairs of unique ID & year. Say you want to check for and study any such duplicated records.

`get_dupes()` returns the records (and inserts a count of duplicates) so you can examine the problematic cases:

``` r
get_dupes(mtcars, wt, cyl) # or mtcars %>% get_dupes(wt, cyl) if you prefer to pipe
#>                 wt cyl dupe_count  mpg  disp  hp drat  qsec vs am gear
#> Merc 280      3.44   6          2 19.2 167.6 123 3.92 18.30  1  0    4
#> Merc 280C     3.44   6          2 17.8 167.6 123 3.92 18.90  1  0    4
#> Duster 360    3.57   8          2 14.3 360.0 245 3.21 15.84  0  0    3
#> Maserati Bora 3.57   8          2 15.0 301.0 335 3.54 14.60  0  1    5
#>               carb
#> Merc 280         4
#> Merc 280C        4
#> Duster 360       4
#> Maserati Bora    8
```

Minor functions
===============

Smaller functions for use in particular situations. More human-readable than the equivalent code they replace.

Cleaning
--------

### Manipulate vectors of names with `make_clean_names()`

Like base R's `make.names()`, but with the stylings and case choice of the long-time janitor function `clean_names()`. While `clean_names()` is still offered for use in data.frame pipeline with `%>%`, `make_clean_names()` allows for more general usage, e.g., on a vector.

It can also be used as an argument to `.name_repair` in the newest version of `tibble::as_tibble`:

``` r
tibble::as_tibble(iris, .name_repair = janitor::make_clean_names)
#> New names:
#> * Sepal.Length -> sepal_length
#> * Sepal.Width -> sepal_width
#> * Petal.Length -> petal_length
#> * Petal.Width -> petal_width
#> * Species -> species
#> # A tibble: 150 x 5
#>    sepal_length sepal_width petal_length petal_width species
#>           <dbl>       <dbl>        <dbl>       <dbl> <fct>  
#>  1          5.1         3.5          1.4         0.2 setosa 
#>  2          4.9         3            1.4         0.2 setosa 
#>  3          4.7         3.2          1.3         0.2 setosa 
#>  4          4.6         3.1          1.5         0.2 setosa 
#>  5          5           3.6          1.4         0.2 setosa 
#>  6          5.4         3.9          1.7         0.4 setosa 
#>  7          4.6         3.4          1.4         0.3 setosa 
#>  8          5           3.4          1.5         0.2 setosa 
#>  9          4.4         2.9          1.4         0.2 setosa 
#> 10          4.9         3.1          1.5         0.1 setosa 
#> # … with 140 more rows
```

### `remove_empty()` rows and columns

Does what it says. For cases like cleaning Excel files that contain empty rows and columns after being read into R.

``` r
q <- data.frame(v1 = c(1, NA, 3),
                v2 = c(NA, NA, NA),
                v3 = c("a", NA, "b"))
q %>%
  remove_empty(c("rows", "cols"))
#>   v1 v3
#> 1  1  a
#> 3  3  b
```

Just a simple wrapper for one-line functions, but it saves a little thinking for both the code writer and the reader.

### `remove_constant()` columns

Drops columns from a data.frame that contain only a single constant value (with an `na.rm` option to control whether NAs should be considered as different values from the constant).

`remove_constant` and `remove_empty` work on matrices as well as data.frames.

``` r
a <- data.frame(good = 1:3, boring = "the same")
a %>% remove_constant()
#>   good
#> 1    1
#> 2    2
#> 3    3
```

### Directionally-consistent rounding behavior with `round_half_up()`

R uses "banker's rounding", i.e., halves are rounded to the nearest *even* number. This function, an exact implementation of <https://stackoverflow.com/questions/12688717/round-up-from-5/12688836#12688836>, will round all halves up. Compare:

``` r
nums <- c(2.5, 3.5)
round(nums)
#> [1] 2 4
round_half_up(nums)
#> [1] 3 4
```

### Round decimals to precise fractions of a given denominator with `round_to_fraction()`

Say your data should only have values of quarters: 0, 0.25, 0.5, 0.75, 1, etc. But there are either user-entered bad values like `0.2` or floating-point precision problems like `0.25000000001`. `round_to_fraction()` will enforce the desired fractional distribution by rounding the values to the nearest value given the specified denominator.

There's also a `digits` argument for optional subsequent rounding.

### Fix dates stored as serial numbers with `excel_numeric_to_date()`

Ever load data from Excel and see a value like `42223` where a date should be? This function converts those serial numbers to class `Date`, with options for different Excel date encoding systems, preserving fractions of a date as time (in which case the returned value is of class `POSIXlt`), and specifying a time zone.

``` r
excel_numeric_to_date(41103)
#> [1] "2012-07-13"
excel_numeric_to_date(41103.01) # ignores decimal places, returns Date object
#> [1] "2012-07-13"
excel_numeric_to_date(41103.01, include_time = TRUE) # returns POSIXlt object
#> [1] "2012-07-13 01:14:24 EDT"
excel_numeric_to_date(41103.01, date_system = "mac pre-2011")
#> [1] "2016-07-14"
```

### Convert a mix of date and datetime formats to date

Building on `excel_numeric_to_date()`, the new functions `convert_to_date()` and `convert_to_datetime()` are more robust to a mix of inputs. Handy when reading many spreadsheets that *should* have the same column formats, but don't.

For instance, here a vector with a date and an Excel datetime sees both values successfully converted to Date class:

``` r
convert_to_date(c("2020-02-29", "40000.1"))
#> [1] "2020-02-29" "2009-07-06"
```

### Elevate column names stored in a data.frame row

If a data.frame has the intended variable names stored in one of its rows, `row_to_names` will elevate the specified row to become the names of the data.frame and optionally (by default) remove the row in which names were stored and/or the rows above it.

``` r
dirt <- data.frame(X_1 = c(NA, "ID", 1:3),
           X_2 = c(NA, "Value", 4:6))

row_to_names(dirt, 2)
#>   ID Value
#> 3  1     4
#> 4  2     5
#> 5  3     6
```

Exploring
---------

### Count factor levels in groups of high, medium, and low with `top_levels()`

Originally designed for use with Likert survey data stored as factors. Returns a `tbl_df` frequency table with appropriately-named rows, grouped into head/middle/tail groups.

-   Takes a user-specified size for the head/tail groups
-   Automatically calculates a percent column
-   Supports sorting
-   Can show or hide `NA` values.

``` r
f <- factor(c("strongly agree", "agree", "neutral", "neutral", "disagree", "strongly agree"),
            levels = c("strongly agree", "agree", "neutral", "disagree", "strongly disagree"))
top_levels(f)
#>                            f n   percent
#>        strongly agree, agree 3 0.5000000
#>                      neutral 2 0.3333333
#>  disagree, strongly disagree 1 0.1666667
top_levels(f, n = 1)
#>                         f n   percent
#>            strongly agree 2 0.3333333
#>  agree, neutral, disagree 4 0.6666667
#>         strongly disagree 0 0.0000000
```