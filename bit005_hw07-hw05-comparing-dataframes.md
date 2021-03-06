# Comparing two almost-identical data.frames
Dean Attali & Jenny Bryan  
December 15, 2014  

## Overview

In [hw07](http://stat545-ubc.github.io/hw07_data-wrangling-grand-finale.html), many accepted the challenge to split the [Gapminder data](https://github.com/jennybc/gapminder) by country, write a country-specific delimited file, then reverse the process. Note this homework was all about data wrangling. Several students used `identical()` and `all.equal()` to compare before vs after and learned that they were not *exactly* the same. But they look the same to human eyes, so what's up?

In this document, we walk through that example to show a workflow for determining *how* two objects differ.

## Load packages and data

We're going to use `plyr` and `dplyr` for a lot of data manipulation, and `gapminder` for data.  `gapminder` is available on [GitHub](https://github.com/jennybc/gapminder) and can be installed as follows:

```
install.packages("devtools")
devtools::install_github("jennybc/gapminder")
```

Let's load the packages:


```r
library(gapminder)
library(plyr)
library(dplyr)
library(knitr)
```

Just a quick sanity check to ensure we have gapminder properly loaded.


```r
tbl_df(gapminder)
```

```
## Source: local data frame [1,704 x 6]
## 
##        country continent year lifeExp      pop gdpPercap
## 1  Afghanistan      Asia 1952  28.801  8425333  779.4453
## 2  Afghanistan      Asia 1957  30.332  9240934  820.8530
## 3  Afghanistan      Asia 1962  31.997 10267083  853.1007
## 4  Afghanistan      Asia 1967  34.020 11537966  836.1971
## 5  Afghanistan      Asia 1972  36.088 13079460  739.9811
## 6  Afghanistan      Asia 1977  38.438 14880372  786.1134
## 7  Afghanistan      Asia 1982  39.854 12881816  978.0114
## 8  Afghanistan      Asia 1987  40.822 13867957  852.3959
## 9  Afghanistan      Asia 1992  41.674 16317921  649.3414
## 10 Afghanistan      Asia 1997  41.763 22227415  635.3414
## ..         ...       ...  ...     ...      ...       ...
```

## Split gapminder to one file per country

First, it's a good idea to create a separate directory for all these files
we're about to create.


```r
outdir <- file.path("tmp-gapminder")
dir.create(outdir, recursive = TRUE, showWarnings = FALSE)
```

Now we use `d_ply()` to split the data by country and write the data for
each country to its own file.


```r
d_ply(gapminder, ~ country, function(x) {
  filename <- file.path(outdir, paste0(x$country[1], ".tsv"))
  write.table(x, file = filename, quote = FALSE, sep = "\t" ,row.names = FALSE)
})
```

Why do we use `d_ply()`? Because a data.frame goes in (`d`) and nothing comes out (`_`), ergo `d_ply()`. Remember to use this over `ddply()` or `dlply()` when there's no return value, i.e. when you're submitting the command for a side effect such as writing a file.

Also note the use of `file.path()` and `paste0()`, which is shortcut for `paste(..., sep = "")`. The use of `file.path()` to build paths will make you code more portable across operating systems.

## Read the files to re-create gapminder

Let's read the files back in and try to recreate the `gapminder` data.frame.

We know that the data files are `.tsv`, so we use a regular expression
pattern to find all the files.


```r
fileRegex <- "^.*\\.tsv$"
out_files <- list.files(outdir, pattern = fileRegex, full.names = TRUE)
out_files %>% head
```

```
## [1] "tmp-gapminder/Afghanistan.tsv" "tmp-gapminder/Albania.tsv"    
## [3] "tmp-gapminder/Algeria.tsv"     "tmp-gapminder/Angola.tsv"     
## [5] "tmp-gapminder/Argentina.tsv"   "tmp-gapminder/Australia.tsv"
```

OK, looks like we have a list with all our files, so let's read them into
`dejavu` and see if it matches the original `gapminder`!


```r
dejavu <- ldply(out_files, read.delim)
all.equal(gapminder, dejavu)
```

```
## [1] "Component \"country\": Attributes: < Component \"levels\": 2 string mismatches >"  
## [2] "Component \"country\": 24 string mismatches"                                       
## [3] "Component \"continent\": Attributes: < Component \"levels\": 4 string mismatches >"
## [4] "Component \"lifeExp\": Mean relative difference: 0.09774601"                       
## [5] "Component \"pop\": Mean relative difference: 0.1702937"                            
## [6] "Component \"gdpPercap\": Mean relative difference: 0.1890264"
```

Bummer.  

It looks like we have non-equalness in every variable except "year".

Every time you run into such a problem, it will be a unique problem, and you
must *read the message* for hints on where the differences lie.

## Digging into the non-identical-ness

I find that the last three messages, about numerical differences,
are the most self-explanatory, so I tackle that first. I'll look at `gdpPercap` first since it has the largest difference. See the appendix for some background on the pitfalls of checking for equality of floating point numbers and on `all.equal()` versus `identical()`.

#### GDP per capita

How many observations have a GDP discrepancy? The threshold is somewhat arbitrary but also informed by reading the message re: mean relative difference.


```r
check_gdp_diff <- abs(gapminder$gdpPercap - dejavu$gdpPercap) > 0.01
sum(check_gdp_diff)
```

```
## [1] 24
```
Hmm.. 24 observations have `gdpPercap` that is different by more than 0.01
between the two sources.  Let's see which observations these are.


```r
which(check_gdp_diff)
```

```
##  [1] 613 614 615 616 617 618 619 620 621 622 623 624 625 626 627 628 629
## [18] 630 631 632 633 634 635 636
```

They seem to all be in a row, suggesting it's a block of data from two countries. Let's take a closer look at what the data from the two data.frames looks like
at these observations.


```r
data.frame(
  with(gapminder, data.frame(
       continent, country, year, gdpPercap)),
  with(dejavu, data.frame(
    continent, country, year, gdpPercap))
  ) %>%
  filter(check_gdp_diff) %>%
  kable
```



continent   country          year   gdpPercap  continent.1   country.1        year.1   gdpPercap.1
----------  --------------  -----  ----------  ------------  --------------  -------  ------------
Africa      Guinea           1952    510.1965  Africa        Guinea-Bissau      1952      299.8503
Africa      Guinea           1957    576.2670  Africa        Guinea-Bissau      1957      431.7905
Africa      Guinea           1962    686.3737  Africa        Guinea-Bissau      1962      522.0344
Africa      Guinea           1967    708.7595  Africa        Guinea-Bissau      1967      715.5806
Africa      Guinea           1972    741.6662  Africa        Guinea-Bissau      1972      820.2246
Africa      Guinea           1977    874.6859  Africa        Guinea-Bissau      1977      764.7260
Africa      Guinea           1982    857.2504  Africa        Guinea-Bissau      1982      838.1240
Africa      Guinea           1987    805.5725  Africa        Guinea-Bissau      1987      736.4154
Africa      Guinea           1992    794.3484  Africa        Guinea-Bissau      1992      745.5399
Africa      Guinea           1997    869.4498  Africa        Guinea-Bissau      1997      796.6645
Africa      Guinea           2002    945.5836  Africa        Guinea-Bissau      2002      575.7047
Africa      Guinea           2007    942.6542  Africa        Guinea-Bissau      2007      579.2317
Africa      Guinea-Bissau    1952    299.8503  Africa        Guinea             1952      510.1965
Africa      Guinea-Bissau    1957    431.7905  Africa        Guinea             1957      576.2670
Africa      Guinea-Bissau    1962    522.0344  Africa        Guinea             1962      686.3737
Africa      Guinea-Bissau    1967    715.5806  Africa        Guinea             1967      708.7595
Africa      Guinea-Bissau    1972    820.2246  Africa        Guinea             1972      741.6662
Africa      Guinea-Bissau    1977    764.7260  Africa        Guinea             1977      874.6859
Africa      Guinea-Bissau    1982    838.1240  Africa        Guinea             1982      857.2504
Africa      Guinea-Bissau    1987    736.4154  Africa        Guinea             1987      805.5725
Africa      Guinea-Bissau    1992    745.5399  Africa        Guinea             1992      794.3484
Africa      Guinea-Bissau    1997    796.6645  Africa        Guinea             1997      869.4498
Africa      Guinea-Bissau    2002    575.7047  Africa        Guinea             2002      945.5836
Africa      Guinea-Bissau    2007    579.2317  Africa        Guinea             2007      942.6542

Looks like the rows for Guinea and Guinea-Bissau have swapped their order. In `gapminder`, it's Guinea then Guinea-Bissau. In `dejavu` it's Guinea-Bissau then Guinea.

Let's see what R thinks the order of these two words is:


```r
sort(c("Guinea-Bissau", "Guinea"))
```

```
## [1] "Guinea"        "Guinea-Bissau"
```

Puzzling ... this agrees with the `gapminder` order. So why is `dejavu` different?

Inspect the order of the two countries in our vector of file names.


```r
grep("Guinea", out_files, value = TRUE)
```

```
## [1] "tmp-gapminder/Equatorial Guinea.tsv"
## [2] "tmp-gapminder/Guinea-Bissau.tsv"    
## [3] "tmp-gapminder/Guinea.tsv"
```

So here is the explanation of the country order in `dejavu` -- it's exactly the order from the filename vector. But why does Guinea-Bissau come before Guinea in this vector? 

We obtained the filenames using `list.files()`.  The documentation for `list.files()`
says under the **Value** section that "The files are sorted in alphabetical
order".  

OK... so the files are supposed to be sorted alphabetically, yet that's not
what we're seeing - we just saw earlier than R knows that Guinea comes before
Guinea-Bissau!  

Or at least that's what I initially thought. If you take a closer look at the
files, you'll realize what's going on here is that R sorts the files based on
the full file path, including its extension. So it's comparing 
tmp-gapminder/Guinea-Bissau.tsv to
tmp-gapminder/Guinea-Bissau.tsv.  

Now do you see the problem? Both files are identical up to the end of "Guinea",
at which point one file has a `-Bissau.tsv` and the other has a `.tsv`. So
R was behaving correctly all along, just not the way we wanted it to.  We want
to sort by the country name, not the full file path. Just to make sure our reasoning is correct, let's confirm that a dash does indeed come before a period


```r
sort(c(".", "-"))
```

```
## [1] "-" "."
```

Yes. Looks like we understand why we got the problem.  

Now that we understand why `dejavu` has a different order, we just
need to think of a way to fix it. The approach I will use it to sort the
files based on the country name, before reading the files.  
The way I do it is by extracting the country name from each file and
sorting the file paths based on the country order. I slightly modified
the `fileRegex` from above by just adding parentheses `()` around the `.*`
so that we can backreference the country name (we learned about regex in hw08).


```r
fileRegex <- "^(.*)\\.tsv$"
out_files <- out_files[order(sub(fileRegex, "\\1", basename(out_files)))]
```

If this line is confusing to you, just break it up into each piece
and think for a minute what each step is doing. Reordering vectors using a
similar approach can be very useful, so do take a minute to take that in.

Now let's ensure that the order of the Guineas is correct in the new vector.


```r
grep("Guinea", out_files, value = TRUE)
```

```
## [1] "tmp-gapminder/Equatorial Guinea.tsv"
## [2] "tmp-gapminder/Guinea.tsv"           
## [3] "tmp-gapminder/Guinea-Bissau.tsv"
```

Success!  
So let's construct `dejavu` again.


```r
dejavu <- ldply(out_files, read.delim)
all.equal(gapminder, dejavu)
```

```
## [1] "Component \"continent\": Attributes: < Component \"levels\": 4 string mismatches >"
```

Sweet, that got rid of most of the mismatches.  Only one remains - and the error
is not too cryptic and tells us that there are 4 mismatches in the levels of
the continent variable. What odes this mean? Well, I'm not sure exactly, so one 
powerful tool that we always have to inspect any object is `str()`. Let's try to
see if we see a difference between `gapminder` and `dejavu` continent variables.


```r
str(gapminder$continent)
```

```
##  Factor w/ 5 levels "Africa","Americas",..: 3 3 3 3 3 3 3 3 3 3 ...
```

```r
str(dejavu$continent)
```

```
##  Factor w/ 5 levels "Asia","Europe",..: 1 1 1 1 1 1 1 1 1 1 ...
```

Okay, we definitely see a difference. But this doesn't show us enough
information.  We know that the message said something about the levels, so let's
look at those more closely.


```r
data.frame(gapminder = levels(gapminder$continent),
           dejavu = levels(dejavu$continent))
```

```
##   gapminder   dejavu
## 1    Africa     Asia
## 2  Americas   Europe
## 3      Asia   Africa
## 4    Europe Americas
## 5   Oceania  Oceania
```

Interesting! `gapminder` levels seem to be alphabetical, while `dejavu` levels
are not.  Now we just have to figure out what order the `dejavu` continent
levels are following.  
The first and most logical guess is that the continent levels are ordered
according to the order in which they are seen in the data. An easy way to 
check what order they appear in the data is to look what the unique continents
are.


```r
unique(gapminder$continent)
```

```
## [1] Asia     Europe   Africa   Americas Oceania 
## Levels: Africa Americas Asia Europe Oceania
```

Uh huh, we're on to something here! This output tells us that Asia is the
first continent that appears in the data (meaning that the first row is an
Asian country), and Europe is next (meaning that the first non-Asian row
is European). Notice how the order of the continents does not match the order
of the levels in the case of `gapminder`.

_Note: It's important to understand the difference between looking at the unique
continents as they appear in the data vs looking at the continent levels, which
are also unique, but can have a different ordering than the order in which they
appear in the data._

If we run the same command on `dejavu`, we expect to see the same order of
continents in the data since the data.frame is the same, but we know that the
order of the levels is different.


```r
unique(dejavu$continent)
```

```
## [1] Asia     Europe   Africa   Americas Oceania 
## Levels: Asia Europe Africa Americas Oceania
```

Yup, just as we suspected! See how in the case of `dejavu`, the order in which
the continents are seen in the data is the same as the levels order. This is
good, we're learning.

So now we know that in order to make our two data.frames the same, we just need
to reorder the continent levels of one of them.  Alphabetical vs chronological
doesn't really make a difference, since both are not terribly useful for
visualization. Remember that we often want to reorder factors for plotting, as
we learned in hw05. Let's choose to reorder the `dejavu` continent levels
to be alphabetical. Since that is the default when creating a new factor, we just need to convert continent to character and back again.


```r
dejavu <- dejavu %>%
  mutate(continent = factor(continent %>% as.character))
all.equal(gapminder, dejavu)
```

```
## [1] TRUE
```

Success!  
We were able to find all the differences between the original
dataset and the written-to-files dataset.  As you can see, there isn't
a clear step-by-step guide that will you can always follow. This is 
mostly problem-solving (sometimes combined with bashing your head against the
wall) and you need to use the tools we have in R to solve the question of
"what's different?".  Some common tools are to use the output from `all.equal()`,
`str)_`, `levels()`, `unique()`, or simply just looking at the raw data. Hopefully
this can make you more comfortable in tackling a similar problem with different
data in the future.

## Bonus: eliminating the problem at the source

How could we have eliminated all this faffing around? By deliberately sorting `dejavu` on `country` and only converting `continent` to factor after the aggregation was done. Watch.


```r
fileRegex <- "^.*\\.tsv$"
out_files <- list.files(outdir, pattern = fileRegex, full.names = TRUE)
dejavu <- ldply(out_files, read.delim, stringsAsFactors = FALSE)
all.equal(gapminder, dejavu)
```

```
## [1] "Component \"country\": 'current' is not a factor"            
## [2] "Component \"continent\": 'current' is not a factor"          
## [3] "Component \"lifeExp\": Mean relative difference: 0.09774601" 
## [4] "Component \"pop\": Mean relative difference: 0.1702937"      
## [5] "Component \"gdpPercap\": Mean relative difference: 0.1890264"
```

```r
dejaNEW <- dejavu %>%
	arrange(country) %>%
	mutate_each(funs(factor), country, continent)
all.equal(gapminder, dejaNEW)
```

```
## [1] TRUE
```

Lesson #1: never rely on row order "just because". Always check or force things to be the way you need them to be.

Lesson #2: it is often easiest to keep things as character for data munging THEN convert to factor. Don't be passive about factor level order.

## Bonus: getting the data.frames to be IDENTICAL

As mentioned before, `identical()` checks for exact identity, and often times
will return FALSE even when two objects seem completely similar to you.  I
wanted to see if we can get `dejavu` to be `identical()` to `gapminder`, so 
let's go!


```r
dejavu <- dejaNEW # reset dejavu to it's fixed version
identical(gapminder, dejavu)
```

```
## [1] FALSE
```

So we know that `all.equal()` thinks they are equal, but `identical()` doesn't. 
We can't use the output from `all.equal()` to direct us to the possible mismatches
anymore - we're on our own now. Scary!

Let's see if we can see any obvious differences... Using `str` again:


```r
str(gapminder)
```

```
## 'data.frame':	1704 obs. of  6 variables:
##  $ country  : Factor w/ 142 levels "Afghanistan",..: 1 1 1 1 1 1 1 1 1 1 ...
##  $ continent: Factor w/ 5 levels "Africa","Americas",..: 3 3 3 3 3 3 3 3 3 3 ...
##  $ year     : num  1952 1957 1962 1967 1972 ...
##  $ lifeExp  : num  28.8 30.3 32 34 36.1 ...
##  $ pop      : num  8425333 9240934 10267083 11537966 13079460 ...
##  $ gdpPercap: num  779 821 853 836 740 ...
```

```r
str(dejavu)
```

```
## 'data.frame':	1704 obs. of  6 variables:
##  $ country  : Factor w/ 142 levels "Afghanistan",..: 1 1 1 1 1 1 1 1 1 1 ...
##  $ continent: Factor w/ 5 levels "Africa","Americas",..: 3 3 3 3 3 3 3 3 3 3 ...
##  $ year     : int  1952 1957 1962 1967 1972 1977 1982 1987 1992 1997 ...
##  $ lifeExp  : num  28.8 30.3 32 34 36.1 ...
##  $ pop      : num  8425333 9240934 10267083 11537966 13079460 ...
##  $ gdpPercap: num  779 821 853 836 740 ...
```

Oh, I do see a difference! The "year" variable is numeric in `gapminder` and
`int` in `dejavu`. I'm not purposely going in circles here - this is how you
troubleshoot these problems in the real world: incrementally, and with
mistakes along the way. So let's change `dejavu` to match `gapminder`:


```r
dejavu <- dejavu %>%
  mutate(year = as.numeric(year))
identical(gapminder,dejavu)
```

```
## [1] FALSE
```

Alright, still not identical. No more easy answers. Next I'll have to try to
zoom in on every single variable and see which ones are causing the
non-identical-ness.


```r
identicalVars <-
  sapply(colnames(gapminder), function(x){
    identical(gapminder[[x]], dejavu[[x]])
  })
(nonIdenticalVars <- names(identicalVars[!identicalVars]))
```

```
## [1] "pop"       "gdpPercap"
```

It looks like the pop and gdpPercap columns are the ones that are causing
non-identity of the two data,frames.  Those two columns are numeric - could it
be that the floating-point number issue described above is the cause of this?


```r
which(gapminder$pop != dejavu$pop)
```

```
## [1] 289
```

```r
which(gapminder$gdpPercap != dejavu$gdpPercap)
```

```
## [1] 289
```

OK, looks like there is one row in which both the population and gdpPercap
aren't the same in both datasets. Is the difference tiny as we suspect?


```r
idxDiff <- which(gapminder$pop != dejavu$pop)
gapminder$pop[idxDiff] - dejavu$pop[idxDiff]
```

```
## [1] -4.768372e-07
```

```r
gapminder$gdpPercap[idxDiff] - dejavu$gdpPercap[idxDiff]
```

```
## [1] -1.136868e-13
```

Yup, both are tiny differences that could be attributable to computer storage.
The difference these numbers is so small that unless we're dealing with
astrophysics computations, I doubt this matters.  I would normally just leave
our data in its current state and come to terms with the fact that it's not
"identical" to `gapminder`, but if it bothers you too much, there is a cheat
we can apply to fix it. There might be better ways, but this is what I can think
of immediately. It is a big hack so don't repeat this at home!


```r
dejavu[idxDiff, nonIdenticalVars] <- gapminder[idxDiff, nonIdenticalVars]
identical(gapminder, dejavu)
```

```
## [1] TRUE
```
There, I hope you're happy now :) The last step is more for pleasing your
OCD than anything else, but it's nice to know that if we force these two
numbers to be the same, `identical()` is finally TRUE!

## Cleanup

Don't forget to clean up after yourself.

```r
unlink(outdir, recursive = TRUE)
```

## Extras

#### Ensuring your filenames are valid
One potential problem you might run into when creating files with filenames
based on data is that not all filenames are legal or desirable.  

For example, we've discussed many times in class how having spaces in a filename
is a bad idea, and some countries do contain a space, so you might want to
remove spaces from filenames.  One possibility is to simply strip spaces, but
another solution would be to replace spaces with underscores. (Of course, then
you can ask "but what about a country name with an underscore?" Well, there
are always ways around everything, but I'll ignore that case for now.)

To do this, all you have to do is slightly modify the code that writes files:
```
d_ply(gapminder, ~ country, function(x) {
  country <- gsub(" ", "_", x$country[1])
  filename <- file.path(outdir, paste0(country, ".tsv"))
  write.table(x, file = filename, quote = FALSE, sep = "\t" ,row.names = FALSE)
})
```

You can get even fancier and more advanced if you need.  On Windows, filenames
cannot contain colons or quotes, so you can replace all those characters
in country names as well, but we don't have to worry about that.

#### Floating point numbers

This is somewhat technical, so don't worry if you don't understand it,
but the main take home message is this: R (and any other programming language)
can be a little inaccurate when representing certain numbers.  
**Explanation**: Because of the way computers work and store information, only
integers and fractions whose denominator is a power of 2 can be represented
exactly. Any other number has to be approximated and rounded off, though you
generally don't see this behavior because the rounding is performed at a very
insignificant digit. As a result, two fractions that should be equal might not
be equal in R, simply because different algorithms are used to compute them,
so they may be rounded a little bit differently.  

For example, anyone can tell you that 1 - 0.9 = 0.1, right? Let's see
what R says


```r
1 - 0.9
```

```
## [1] 0.1
```

No problem, what was I making such a big fuss about?? Let's look again...


```r
1 - 0.9 == 0.1
```

```
## [1] FALSE
```

That's the problem I was talking about. Another way to see this more clearly:

```r
1 - 0.9 - 0.1
```

```
## [1] -2.775558e-17
```

As you can see, the difference is tiny. Most people can live their lives
perfectly happy never knowing about this seemingly horrible shortcoming of
computers, but it's good to keep in mind.  

You can Google for this to learn more, or [read this StackOverflow
thread](http://stackoverflow.com/questions/9508518/why-are-these-numbers-not-equal)
to see a nice detailed discussion that is tailored to R. It is important to
remember that this is NOT an R-specific problem though, so don't hate on R
because of this. (Though, granted, there are [many other reasons to hate on
R](http://www.burns-stat.com/pages/Tutor/R_inferno.pdf) if you so wish.)

#### Difference between `all.equal` and `identical`

According to the `identical()` documentation:  "The safe and reliable way to test
two objects for being exactly equal". Notice the stress on "exactly equal".  
According to the `all.equal()` documentation: "a utility to compare R objects x and
y testing 'near equality'".  What exactly do they mean by "near equality"? 
Unfortunately the documentation doesn't tell us that exactly, but generally
`all.equal()` will return `TRUE` if the two objects are equal enough for (almost)
all intents and purposes, while `identical()` is a perfectionist with a gold medal
in OCD that will only return `TRUE` if everything about the two objects is
absolutely exactly the same.  

For example, `all.equal` doesn't care about the exact storage type of numerics
if they pretty much represent the same thing, whereas `identical()` does.


```r
all.equal(as.integer(5), as.double(5))
```

```
## [1] TRUE
```

```r
identical(as.integer(5), as.double(5))
```

```
## [1] FALSE
```

Another example is that `all.equal()` is aware of the floating-point numbers
problem described above, so it is a little lenient if two numbers are not
exactly identical.  As shown above, 1 - 0.9 - 0.1 is represented in R as
-2.7755576\times 10^{-17}.  But `all.equal()` realizes this is an insignificant difference
and that for most everyday uses, that number is as good as `0`.


```r
all.equal(1 - 0.9 - 0.1, 0)
```

```
## [1] TRUE
```

```r
identical(1 - 0.9 - 0.1, 0)
```

```
## [1] FALSE
```

You can actually change the error tolerance of `all.equal()`, which is
1.4901161\times 10^{-8} by default (varies by computer). This means that
any two numbers differing by less than 1.4901161\times 10^{-8} will be 
reported as the same.


```r
all.equal(1 - 0.9 - 0.1, 0, tolerance = 1e-15)
```

```
## [1] TRUE
```

```r
all.equal(1 - 0.9 - 0.1, 0, tolerance = 1e-20)
```

```
## [1] "Mean relative difference: 1"
```

We're getting side-tracked, so if you really want to understand the checks that
`all.equal()` performs or what the output means, you can look at the source code
of all the different `all.equal()` methods: there is an `all.equal.numeric()` for
numeric data, `all.equal.character()` for character data, etc. Just use tab
completion to find out what the different `all.equal.*` methods are.  In case you
didn't know, if you type the name of a function without parentheses, it will
show you the function code (since a function is really just a variable) instead
of running that function. So if you want to see the code of `all.equal.numeric()`,
simply run `all.equal.numeric()`.

----------
