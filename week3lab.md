The glory of dplyr!
===================

The fancy printing of `help` files, etc., in this document is handled by
`printr`:

    library(printr)

    ## Registered S3 method overwritten by 'printr':
    ##   method                from     
    ##   knit_print.data.frame rmarkdown

Today’s exercise requires some imagination on your part. We will not
give you a massive data set. Rather, we provide you the foundation to
efficiently handle some very large computations.

Some of you won’t “get it”. Not now at least. But, [a recent
paper](https://www.genetics.org/content/213/4/1513) of mine literally
would not have been possible without the methods outlined here.

The problem
-----------

1.  You have a very large amount of data. It cannot fit into RAM, so you
    cannot use regular `R/dplyr` or `pandas` methods to process it.
2.  There’s no pre-existing tool for your analysis. For example, this
    isn’t something that `samtools` or `bedtools` or some other
    domain-specific software already do.

Thinking about a solution
-------------------------

Let’s reason through a few things:

1.  The data can’t fit in RAM, so they need to go in a file.
2.  So, we need a file format.
3.  We also need code to process it.

Rectangular data and split/apply/combine
----------------------------------------

Let’s fire up the venerable `mtcars` data set and take a look:

    data(mtcars)
    help(mtcars)

    ## Motor Trend Car Road Tests
    ## 
    ## Description:
    ## 
    ##      The data was extracted from the 1974 _Motor Trend_ US magazine,
    ##      and comprises fuel consumption and 10 aspects of automobile design
    ##      and performance for 32 automobiles (1973-74 models).
    ## 
    ## Usage:
    ## 
    ##      mtcars
    ##      
    ## Format:
    ## 
    ##      A data frame with 32 observations on 11 (numeric) variables.
    ## 
    ##        [, 1]  mpg   Miles/(US) gallon                        
    ##        [, 2]  cyl   Number of cylinders                      
    ##        [, 3]  disp  Displacement (cu.in.)                    
    ##        [, 4]  hp    Gross horsepower                         
    ##        [, 5]  drat  Rear axle ratio                          
    ##        [, 6]  wt    Weight (1000 lbs)                        
    ##        [, 7]  qsec  1/4 mile time                            
    ##        [, 8]  vs    Engine (0 = V-shaped, 1 = straight)      
    ##        [, 9]  am    Transmission (0 = automatic, 1 = manual) 
    ##        [,10]  gear  Number of forward gears                  
    ##        [,11]  carb  Number of carburetors                    
    ##       
    ## Note:
    ## 
    ##      Henderson and Velleman (1981) comment in a footnote to Table 1:
    ##      'Hocking [original transcriber]'s noncrucial coding of the Mazda's
    ##      rotary engine as a straight six-cylinder engine and the Porsche's
    ##      flat engine as a V engine, as well as the inclusion of the diesel
    ##      Mercedes 240D, have been retained to enable direct comparisons to
    ##      be made with previous analyses.'
    ## 
    ## Source:
    ## 
    ##      Henderson and Velleman (1981), Building multiple regression models
    ##      interactively.  _Biometrics_, *37*, 391-411.
    ## 
    ## Examples:
    ## 
    ##      require(graphics)
    ##      pairs(mtcars, main = "mtcars data", gap = 1/4)
    ##      coplot(mpg ~ disp | as.factor(cyl), data = mtcars,
    ##             panel = panel.smooth, rows = 1)
    ##      ## possibly more meaningful, e.g., for summary() or bivariate plots:
    ##      mtcars2 <- within(mtcars, {
    ##         vs <- factor(vs, labels = c("V", "S"))
    ##         am <- factor(am, labels = c("automatic", "manual"))
    ##         cyl  <- ordered(cyl)
    ##         gear <- ordered(gear)
    ##         carb <- ordered(carb)
    ##      })
    ##      summary(mtcars2)

The first several rows are:

    head(mtcars)

<table>
<thead>
<tr class="header">
<th style="text-align: left;"></th>
<th style="text-align: right;">mpg</th>
<th style="text-align: right;">cyl</th>
<th style="text-align: right;">disp</th>
<th style="text-align: right;">hp</th>
<th style="text-align: right;">drat</th>
<th style="text-align: right;">wt</th>
<th style="text-align: right;">qsec</th>
<th style="text-align: right;">vs</th>
<th style="text-align: right;">am</th>
<th style="text-align: right;">gear</th>
<th style="text-align: right;">carb</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td style="text-align: left;">Mazda RX4</td>
<td style="text-align: right;">21.0</td>
<td style="text-align: right;">6</td>
<td style="text-align: right;">160</td>
<td style="text-align: right;">110</td>
<td style="text-align: right;">3.90</td>
<td style="text-align: right;">2.620</td>
<td style="text-align: right;">16.46</td>
<td style="text-align: right;">0</td>
<td style="text-align: right;">1</td>
<td style="text-align: right;">4</td>
<td style="text-align: right;">4</td>
</tr>
<tr class="even">
<td style="text-align: left;">Mazda RX4 Wag</td>
<td style="text-align: right;">21.0</td>
<td style="text-align: right;">6</td>
<td style="text-align: right;">160</td>
<td style="text-align: right;">110</td>
<td style="text-align: right;">3.90</td>
<td style="text-align: right;">2.875</td>
<td style="text-align: right;">17.02</td>
<td style="text-align: right;">0</td>
<td style="text-align: right;">1</td>
<td style="text-align: right;">4</td>
<td style="text-align: right;">4</td>
</tr>
<tr class="odd">
<td style="text-align: left;">Datsun 710</td>
<td style="text-align: right;">22.8</td>
<td style="text-align: right;">4</td>
<td style="text-align: right;">108</td>
<td style="text-align: right;">93</td>
<td style="text-align: right;">3.85</td>
<td style="text-align: right;">2.320</td>
<td style="text-align: right;">18.61</td>
<td style="text-align: right;">1</td>
<td style="text-align: right;">1</td>
<td style="text-align: right;">4</td>
<td style="text-align: right;">1</td>
</tr>
<tr class="even">
<td style="text-align: left;">Hornet 4 Drive</td>
<td style="text-align: right;">21.4</td>
<td style="text-align: right;">6</td>
<td style="text-align: right;">258</td>
<td style="text-align: right;">110</td>
<td style="text-align: right;">3.08</td>
<td style="text-align: right;">3.215</td>
<td style="text-align: right;">19.44</td>
<td style="text-align: right;">1</td>
<td style="text-align: right;">0</td>
<td style="text-align: right;">3</td>
<td style="text-align: right;">1</td>
</tr>
<tr class="odd">
<td style="text-align: left;">Hornet Sportabout</td>
<td style="text-align: right;">18.7</td>
<td style="text-align: right;">8</td>
<td style="text-align: right;">360</td>
<td style="text-align: right;">175</td>
<td style="text-align: right;">3.15</td>
<td style="text-align: right;">3.440</td>
<td style="text-align: right;">17.02</td>
<td style="text-align: right;">0</td>
<td style="text-align: right;">0</td>
<td style="text-align: right;">3</td>
<td style="text-align: right;">2</td>
</tr>
<tr class="even">
<td style="text-align: left;">Valiant</td>
<td style="text-align: right;">18.1</td>
<td style="text-align: right;">6</td>
<td style="text-align: right;">225</td>
<td style="text-align: right;">105</td>
<td style="text-align: right;">2.76</td>
<td style="text-align: right;">3.460</td>
<td style="text-align: right;">20.22</td>
<td style="text-align: right;">1</td>
<td style="text-align: right;">0</td>
<td style="text-align: right;">3</td>
<td style="text-align: right;">1</td>
</tr>
</tbody>
</table>

The data are “rectangular”, meaning that they are set up like a
spreadsheet.

A typical analysis of such data involves:

1.  **Split** the data up into groups. For example, break the data set
    up according to number of cylinders or numb of gears. One could also
    split by unique `(cylinder, gear)` combos.
2.  **Apply** a function to each group. For example, get the average
    `mpg` for each group.
3.  **Combine** (“aggregate”) the results back into a new data frame.

In `base` `R`, we can do these analyses using `aggregate`. These
analyses use `R`’s formula notation, `response ~ factors`. This kind of
formula notation is common in statistical languages, and you’ll see it
in many corners of `R`.

Let’s get the mean `mpg` after grouping by `cyl`:

    aggregate(mpg ~ cyl, data=mtcars, mean)

<table>
<thead>
<tr class="header">
<th style="text-align: right;">cyl</th>
<th style="text-align: right;">mpg</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td style="text-align: right;">4</td>
<td style="text-align: right;">26.66364</td>
</tr>
<tr class="even">
<td style="text-align: right;">6</td>
<td style="text-align: right;">19.74286</td>
</tr>
<tr class="odd">
<td style="text-align: right;">8</td>
<td style="text-align: right;">15.10000</td>
</tr>
</tbody>
</table>

Now, get the means for all `(cyl, gear)` combos:

    aggregate(mpg ~ cyl + gear, data=mtcars, mean)

<table>
<thead>
<tr class="header">
<th style="text-align: right;">cyl</th>
<th style="text-align: right;">gear</th>
<th style="text-align: right;">mpg</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td style="text-align: right;">4</td>
<td style="text-align: right;">3</td>
<td style="text-align: right;">21.500</td>
</tr>
<tr class="even">
<td style="text-align: right;">6</td>
<td style="text-align: right;">3</td>
<td style="text-align: right;">19.750</td>
</tr>
<tr class="odd">
<td style="text-align: right;">8</td>
<td style="text-align: right;">3</td>
<td style="text-align: right;">15.050</td>
</tr>
<tr class="even">
<td style="text-align: right;">4</td>
<td style="text-align: right;">4</td>
<td style="text-align: right;">26.925</td>
</tr>
<tr class="odd">
<td style="text-align: right;">6</td>
<td style="text-align: right;">4</td>
<td style="text-align: right;">19.750</td>
</tr>
<tr class="even">
<td style="text-align: right;">4</td>
<td style="text-align: right;">5</td>
<td style="text-align: right;">28.200</td>
</tr>
<tr class="odd">
<td style="text-align: right;">6</td>
<td style="text-align: right;">5</td>
<td style="text-align: right;">19.700</td>
</tr>
<tr class="even">
<td style="text-align: right;">8</td>
<td style="text-align: right;">5</td>
<td style="text-align: right;">15.400</td>
</tr>
</tbody>
</table>

Okay, cool. The code isn’t too bad, and it uses a common syntax that is
all over the `R` world (the “formula”). Let’s look at the `dplyr`
version now.

The first example is:

    library(dplyr)

    ## 
    ## Attaching package: 'dplyr'

    ## The following objects are masked from 'package:stats':
    ## 
    ##     filter, lag

    ## The following objects are masked from 'package:base':
    ## 
    ##     intersect, setdiff, setequal, union

    results = mtcars %>%
        group_by(cyl) %>%
        summarise(mean_mpg = mean(mpg))

    ## `summarise()` ungrouping output (override with `.groups` argument)

    results

<table>
<thead>
<tr class="header">
<th style="text-align: right;">cyl</th>
<th style="text-align: right;">mean_mpg</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td style="text-align: right;">4</td>
<td style="text-align: right;">26.66364</td>
</tr>
<tr class="even">
<td style="text-align: right;">6</td>
<td style="text-align: right;">19.74286</td>
</tr>
<tr class="odd">
<td style="text-align: right;">8</td>
<td style="text-align: right;">15.10000</td>
</tr>
</tbody>
</table>

The second analysis becomes:

    results = mtcars %>%
        group_by(cyl, gear) %>%
        summarise(mean_mpg = mean(mpg))

    ## `summarise()` regrouping output by 'cyl' (override with `.groups` argument)

    as.data.frame(results)

<table>
<thead>
<tr class="header">
<th style="text-align: right;">cyl</th>
<th style="text-align: right;">gear</th>
<th style="text-align: right;">mean_mpg</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td style="text-align: right;">4</td>
<td style="text-align: right;">3</td>
<td style="text-align: right;">21.500</td>
</tr>
<tr class="even">
<td style="text-align: right;">4</td>
<td style="text-align: right;">4</td>
<td style="text-align: right;">26.925</td>
</tr>
<tr class="odd">
<td style="text-align: right;">4</td>
<td style="text-align: right;">5</td>
<td style="text-align: right;">28.200</td>
</tr>
<tr class="even">
<td style="text-align: right;">6</td>
<td style="text-align: right;">3</td>
<td style="text-align: right;">19.750</td>
</tr>
<tr class="odd">
<td style="text-align: right;">6</td>
<td style="text-align: right;">4</td>
<td style="text-align: right;">19.750</td>
</tr>
<tr class="even">
<td style="text-align: right;">6</td>
<td style="text-align: right;">5</td>
<td style="text-align: right;">19.700</td>
</tr>
<tr class="odd">
<td style="text-align: right;">8</td>
<td style="text-align: right;">3</td>
<td style="text-align: right;">15.050</td>
</tr>
<tr class="even">
<td style="text-align: right;">8</td>
<td style="text-align: right;">5</td>
<td style="text-align: right;">15.400</td>
</tr>
</tbody>
</table>

To summarise:

1.  We have a new syntax. Boo!
2.  This syntax is “weird”, maybe?
3.  It is a bit longer/more verbose.

So, why bother?

1.  As your analysis becomes more complex, the `dplyr` variant will tend
    to be *less* verbose than the `base` version.
2.  The `dplyr` solution will be *vastly* faster.
3.  Finally, `dplyr` has magic super powers that help solve the problem
    of data sets too big to fit in memory. This is the part that
    requires imagination, as our examples so far are in memory. Trust
    me, though, the payoff is coming…

### Some technical bits

1.  The `%>%` code chains together a series of function calls. To read
    it, to left to right starting after the `=`:
    -   Take the `mtcars` data.
    -   Pass it to `group_by`, which is our **split** step.
    -   Pass each group to `summarise`, which is our **apply** step.
    -   Return the result into `results`, which is our **combine** step.
2.  The `as.data.frame` business in the last example is only needed so
    that `printr` gives this page a nice rendering. Technically, these
    operations return a `tibble`, which is a fancified version of a
    `data.frame`. You can read more about `tibble`s in the [R for Data
    Science](https://r4ds.had.co.nz/) book. Most of the time, though,
    you can take them for granted and pretend you have a vanilla
    `data.frame`. (That is intentional! A `tibble` is designed to be as
    close to a drop-in replacement as possible.)

A solution for very large data sets
-----------------------------------

So, if our data can be arranged as one or more “tidy” spreadsheets and
our analysis will be “split/apply/combine”, then the solution to our
problem is:

1.  Store each data frame/“spreadsheet” as a *table* in a *relational
    database* software.
2.  Use [dplyr](https://dplyr.tidyverse.org/) for our analysis.

### Relational databases

To oversimplify a bit, these are spreadsheets/data frames stored on
disk. Key examples of open-source software versions are:

-   MySQL
-   PostgreSQL
-   sqlite3

All of these have `SQL` in the name, which means “structured query
language”. This is an “English-adjacent” language for performing
split/apply/combine analysis on databases. Each program uses a slightly
different `SQL` “dialect”, which is quite irritating.

The first two tools (My and Postgre) require administrative privileges
to use. The latter, sqlite3, does not, making it amenable to using in
your pipelines. You give up some speed, though.

### Databases plus dplyr

This is the payoff: `dplyr` can analyze data *in a relational database*
using the **same code** that you use for in-memory *data frames*!!!

This capability of `dplyr` is due to some very impressive software
engineering! It will write the `SQL` syntax for you. It can do so for
several `SQL` dialects.

The only substantive difference is that your input to a `dplyr` analysis
will be a connection to a database table rather than a data frame that’s
already in memory.

#### Caveat

You can only perform analyses if the `dplyr` “verbs” you use have a
direct analog in the `SQL` dialect of your database. This limitation
will not affect you now, but may in the future.

#### Documentation

-   [dbplyr](https://dbplyr.tidyverse.org/).
-   [DBI](https://dbi.r-dbi.org/)

### A working example

    ## [1] TRUE

First, get our data into an `sqlite3` database:

    library(dbplyr)

    ## 
    ## Attaching package: 'dbplyr'

    ## The following objects are masked from 'package:dplyr':
    ## 
    ##     ident, sql

    # Create a connection ("con") to a database file:
    con <- DBI::dbConnect(RSQLite::SQLite(), "mtcars.sqlite3")
    # Write our data frame to the database in a table called "mtcars"
    DBI::dbWriteTable(con, "mtcars", mtcars)
    # Disconnect from our database
    DBI::dbDisconnect(con)

#### Warning

The `dbplyr` docs use the following code, which **did not** work for me:

    copy_to(con, mtcars)

Let’s check that we have a database whose file is  &gt; 0:

    ls -lhrt *.sqlite3

    ## -rw-r--r-- 1 molpopgen molpopgen  12K Jan 20 12:01 mtcars_from_pandas.sqlite3
    ## -rw-r--r-- 1 molpopgen molpopgen 8.0K Jan 20 12:04 mtcars.sqlite3

#### Analyze our database using dplyr

    con <- DBI::dbConnect(RSQLite::SQLite(), "mtcars.sqlite3")
    mtcars2 <- tbl(con, "mtcars")
    g = mtcars2 %>% 
        group_by(cyl) %>%
        summarise(mean_mpg=mean(mpg))

The object `g` is **not** (yet) a `dplyr` result! It is an `SQL` query:

    g %>% show_query()

    ## Warning: Missing values are always removed in SQL.
    ## Use `mean(x, na.rm = TRUE)` to silence this warning
    ## This warning is displayed only once per session.

    ## <SQL>
    ## SELECT `cyl`, AVG(`mpg`) AS `mean_mpg`
    ## FROM `mtcars`
    ## GROUP BY `cyl`

We need to `collect` it to execute the query:

    result = g %>% collect()
    as.data.frame(result)

<table>
<thead>
<tr class="header">
<th style="text-align: right;">cyl</th>
<th style="text-align: right;">mean_mpg</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td style="text-align: right;">4</td>
<td style="text-align: right;">26.66364</td>
</tr>
<tr class="even">
<td style="text-align: right;">6</td>
<td style="text-align: right;">19.74286</td>
</tr>
<tr class="odd">
<td style="text-align: right;">8</td>
<td style="text-align: right;">15.10000</td>
</tr>
</tbody>
</table>

### This looks the same–what is the point?

Well, the point is that it (mostly) looks the same! However, there is a
*big* difference! The *query* is executed by the `sqlite3` database
software. By doing so, the entire database is **not** loaded into
memory.

In other words:

-   When your data fit into memory, use `dplyr`.
-   When your data do not fit into memory, use `dplyr` (to analyze stuff
    in a database).

**One toolkit can handle all of your needs.**

Python
------

    ## [1] TRUE

Python users can play these games, too.

    library(reticulate)

Grab `mtcars` into Python:

    mtcars = r.mtcars
    mtcars.head()

    ##                     mpg  cyl   disp     hp  drat  ...   qsec   vs   am  gear  carb
    ## Mazda RX4          21.0  6.0  160.0  110.0  3.90  ...  16.46  0.0  1.0   4.0   4.0
    ## Mazda RX4 Wag      21.0  6.0  160.0  110.0  3.90  ...  17.02  0.0  1.0   4.0   4.0
    ## Datsun 710         22.8  4.0  108.0   93.0  3.85  ...  18.61  1.0  1.0   4.0   1.0
    ## Hornet 4 Drive     21.4  6.0  258.0  110.0  3.08  ...  19.44  1.0  0.0   3.0   1.0
    ## Hornet Sportabout  18.7  8.0  360.0  175.0  3.15  ...  17.02  0.0  0.0   3.0   2.0
    ## 
    ## [5 rows x 11 columns]

[pandas](https://pandas.pydata.org)’ `DataFrame` object has the
split/apply/combine stuff built in.

Repeat our split/apply/combine analyses:

    mtcars.groupby(['cyl'])['mpg'].mean()

    ## cyl
    ## 4.0    26.663636
    ## 6.0    19.742857
    ## 8.0    15.100000
    ## Name: mpg, dtype: float64

And the other one:

    mtcars.groupby(['cyl', 'gear'])['mpg'].mean()

    ## cyl  gear
    ## 4.0  3.0     21.500
    ##      4.0     26.925
    ##      5.0     28.200
    ## 6.0  3.0     19.750
    ##      4.0     19.750
    ##      5.0     19.700
    ## 8.0  3.0     15.050
    ##      5.0     15.400
    ## Name: mpg, dtype: float64

### Pandas to sqlite

Simple:

    import sqlite3 # Built into the Python language!
    con = sqlite3.connect("mtcars_from_pandas.sqlite3")
    # Add our data frame to the mtcars table in the database
    mtcars.to_sql("mtcars", con)
    con.close()

Check that it worked:

    ls -lhrt *.sqlite3

    ## -rw-r--r-- 1 molpopgen molpopgen 8.0K Jan 20 12:04 mtcars.sqlite3
    ## -rw-r--r-- 1 molpopgen molpopgen  12K Jan 20 12:04 mtcars_from_pandas.sqlite3

**NOTE:** The two data base files differ in size! This is because `R`
and `pandas` make different default decisions about the column data
types. Thus, your database may be bigger if you create it from `pandas`
without changing the default, meaning you may be using more disk space
than you need! It is an “exercise for the interested reader” to learn
how to change those defaults.

### Read it back in

This is where `R` is ahead of the curve. Sadly, to get data back into a
`pandas.DataFrame`, we must use “raw” `SQL`:

    import pandas as pd

    con = sqlite3.connect("mtcars_from_pandas.sqlite3")
    df = pd.read_sql("select * from mtcars", con)
    df.head()

    ##                index   mpg  cyl   disp     hp  ...   qsec   vs   am  gear  carb
    ## 0          Mazda RX4  21.0  6.0  160.0  110.0  ...  16.46  0.0  1.0   4.0   4.0
    ## 1      Mazda RX4 Wag  21.0  6.0  160.0  110.0  ...  17.02  0.0  1.0   4.0   4.0
    ## 2         Datsun 710  22.8  4.0  108.0   93.0  ...  18.61  1.0  1.0   4.0   1.0
    ## 3     Hornet 4 Drive  21.4  6.0  258.0  110.0  ...  19.44  1.0  0.0   3.0   1.0
    ## 4  Hornet Sportabout  18.7  8.0  360.0  175.0  ...  17.02  0.0  0.0   3.0   2.0
    ## 
    ## [5 rows x 12 columns]

To do our analyses:

    df = pd.read_sql("select cyl, avg(mpg) from mtcars group by cyl", con)
    df.head()

    ##    cyl   avg(mpg)
    ## 0  4.0  26.663636
    ## 1  6.0  19.742857
    ## 2  8.0  15.100000

    df = pd.read_sql("select cyl, gear, avg(mpg) from mtcars group by cyl, gear", con)
    df.head()

    ##    cyl  gear  avg(mpg)
    ## 0  4.0   3.0    21.500
    ## 1  4.0   4.0    26.925
    ## 2  4.0   5.0    28.200
    ## 3  6.0   3.0    19.750
    ## 4  6.0   4.0    19.750

Most of the time, I’d use `dplyr` here instead. The `SQL` shown here is
deceptively simple. Trust me–it gets complex fast!

### More things to learn

-   `dplyr` and `pandas` are both capable of joining multiple data
    frames together. `dplyr` is able to join `SQL` tables together.
-   For big data sets, you usually add data to tables in “chunks” as
    they come. So, you need to distinguish between *creating* a table vs
    *appending* to an existing one. Often, *append* is the default. So,
    when you make mistakes, you need to either `rm -f` your database
    file or learn to `drop` tables. (That’s another “exercise for the
    reader”.)

Your lab exercise for this week.
--------------------------------

Repeat the above code blocks! Couldn’t be simpler, right! (Famous last
words.)

### Stuff to install

#### R (via RStudio)

At the very minimum, you’ll need:

-   dplyr
-   dbplyr
-   RSQLite

#### Python (via conda)

-   sqlite3

You should have `pandas` from last week’s exercise.
