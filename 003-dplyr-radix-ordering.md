
# Tidyup 3: Radix Ordering in `dplyr::arrange()`

**Champion**: Davis

**Co-Champion**: Hadley

**Status**: Accepted

## Abstract

As of dplyr 1.0.6, `arrange()` uses a modified version of
`base::order()` to sort columns of a data frame. Recently, vctrs has
gained `vec_order()`, a fast radix ordering function with first class
support for data frames and custom vctrs types, along with enhanced
character ordering. The purpose of this tidyup is to propose switching
`arrange()` from `order()` to `vec_order()` in the most user-friendly
way possible.

## Motivation

Thanks to the data.table team, R &gt;= 3.3 gained support for an
extremely fast radix ordering algorithm in `order()`. This has become
the default algorithm for ordering most atomic types, with the notable
exception of character vectors. Radix ordering character vectors is only
possible in the C locale, but the shell sort currently in use by
`order()` respects the system locale. Because R is justifiably hesitant
to break backwards compatibility, if *any* character vector is present,
the entire ordering procedure falls back to a much slower shell sort.
Because dplyr uses `order()` internally, the performance of `arrange()`
is negatively affected by this fallback.

Inspired by the performance of the radix ordering algorithm, and by its
many practical applications for data science, a radix order-based
`vec_order()` was added to vctrs, which has the following benefits:

-   Radix ordering on all atomic types.

-   First class support for data frames, including specifying sorting
    direction on a per column basis.

-   Support for an optional character transformation function, which
    generates an intermediate *sort key* that is unique to a particular
    locale. When sorted in the C locale, the sort key returns an
    ordering that would be equivalent to directly sorting the original
    vector in the locale that the sort key was generated for.

It is worth looking at a quick example that demonstrates just how fast
this radix ordering algorithm is when compared against the defaults of
`order()`.

``` r
library(stringi)
library(vctrs) # r-lib/vctrs#1435
set.seed(123)

# 10,000 random strings, sampled to a total size of 1,000,000
n_unique <- 10000L
n_total <- 1000000L

dictionary <- stri_rand_strings(
  n = n_unique, 
  length = sample(1:30, n_unique, replace = TRUE)
)

x <- sample(dictionary, size = n_total, replace = TRUE)

head(x)
```

    ## [1] "vW5VN"                          "qdNNzemEw1sXdoaqsLz1mJc3bGuixU"
    ## [3] "mljKvuznJRP"                    "22wLX7L"                       
    ## [5] "wcIz5PS93kRUC"                  "2yy09KfokjQoBwumnUascCD"

``` r
# Force `order()` to use the C locale to match `vec_order()`
bench::mark(
  base = withr::with_locale(c(LC_COLLATE = "C"), order(x)),
  vctrs = vec_order(x)
)
```

    ## # A tibble: 2 x 6
    ##   expression      min   median `itr/sec` mem_alloc `gc/sec`
    ##   <bch:expr> <bch:tm> <bch:tm>     <dbl> <bch:byt>    <dbl>
    ## 1 base          1.61s    1.61s     0.621    3.89MB      0  
    ## 2 vctrs       18.22ms  20.77ms    46.9     12.71MB     21.9

``` r
# Force `vec_order()` to use the American English locale, which is also
# my system locale. To do that we'll need to generate a sort key, which
# we can sort in the C locale, but the result will be like we sorted
# directly in the American English locale.
bench::mark(
  base = order(x),
  vctrs = vec_order(x, chr_transform = ~stri_sort_key(.x, locale = "en_US"))
)
```

    ## Warning: Some expressions had a GC in every iteration; so filtering is disabled.

    ## # A tibble: 2 x 6
    ##   expression      min   median `itr/sec` mem_alloc `gc/sec`
    ##   <bch:expr> <bch:tm> <bch:tm>     <dbl> <bch:byt>    <dbl>
    ## 1 base          6.48s    6.48s     0.154    3.81MB     0   
    ## 2 vctrs      642.76ms 642.76ms     1.56    20.21MB     3.11

``` r
# Generating the sort key takes most of the time
bench::mark(
  sort_key = stri_sort_key(x, locale = "en_US")
)
```

    ## Warning: Some expressions had a GC in every iteration; so filtering is disabled.

    ## # A tibble: 1 x 6
    ##   expression      min   median `itr/sec` mem_alloc `gc/sec`
    ##   <bch:expr> <bch:tm> <bch:tm>     <dbl> <bch:byt>    <dbl>
    ## 1 sort_key      656ms    656ms      1.53    7.63MB     1.53

In dplyr, we’d like to utilize `vec_order()` while breaking as little
code as possible. Switching to `vec_order()` has many potential positive
impacts, including:

-   Improved performance when ordering character vectors.

    -   Which also results in improved overall performance when ordering
        character vectors alongside other atomic types, since it would
        no longer cause the whole procedure to fall back to a shell
        sort.

-   Improved reproducibility across sessions by ensuring that the
    default behavior doesn’t depend on an environment variable,
    `LC_COLLATE`, which is used by `order()` to determine the locale.

-   Improved reproducibility across OSes where `LC_COLLATE` might
    differ, even for the same locale. For example, `"en_US"` on a Mac is
    approximately equivalent to `"English_United States.1252"` on
    Windows.

-   Improved consistency within the tidyverse. Specifically, with
    stringr, which defaults to the `"en"` locale and has an explicit
    argument for changing the locale,
    i.e. `stringr::str_order(locale = "en")`.

However, making this switch also has potential negative impacts:

-   Breaking code that relies strongly on the ordering resulting from
    using the `LC_COLLATE` environment variable.

-   Surprising users if the new ordering does not match the previous
    one.

## Solution

To switch to `vec_order()` internally while surprising the least amount
of users, it is proposed that the data frame method for `arrange()` gain
a new argument, `.locale`, with the following properties:

-   Defaults to `dplyr_locale()`, see below.
-   If stringi is installed, allow a string locale identifier, such as
    `"fr"` for French, for explicitly adjusting the locale. If stringi
    is not installed, and the user has explicitly specified a locale
    identifier, an error will be thrown.
-   Allow `"C"` to be specified as a special case, which is an explicit
    way to request the C locale. If the exact details of the ordering
    are not critical, this is often much faster than specifying a locale
    identifier. This does not require stringi.

`dplyr_locale()` would be a new exported helper function. Its purpose is
to return a string containing the default locale to sort with. It has
the following properties:

-   If stringi is installed, the American English locale, `"en"`, is
    used as the default.

-   If stringi is not installed, the C locale, `"C"`, will be used. A
    warning will be thrown informing the user of this fallback behavior,
    and encouraging them to silence the warning by either installing
    stringi or explicitly specifying `.locale = "C"`.

-   Alternatively, to globally override the above default behavior, the
    global option, `dplyr.locale`, can be set to either `"C"` or a
    string locale identifier. Setting this to anything except `"C"`
    would require stringi.

American English has been chosen as the default solely because we
believe it is the most used locale among R users, so it is the least
likely to change existing results. That said, we understand that
non-English speakers will not want to set the `.locale` argument *every
time* they call `arrange()`, so the global option provides a way to
change that default for their scripts. Feedback has indicated that while
a global option can decrease reproducibility between sessions, in this
case the benefits of it outweigh the costs.

The global option `dplyr.locale` should be used sparingly, as it has the
potential to reduce reproducibility across R sessions and affect
indirect calls to `arrange()`. It should be viewed as a *convenience
option*, which can be helpful for quickly adapting an existing script to
the new behavior of `arrange()`, but ideally should not be used in
production code.

On certain systems, stringi can be a difficult dependency to install.
Because of this, this proposal recommends that stringi only be
*suggested* so that users without stringi can still use dplyr.

This proposal relies on `stringi::stri_sort_key()`, which generates the
sort key mentioned under Motivation as a proxy that can be ordered in
the C locale. However, sort key generation is expensive. In fact, it is
often the most expensive part of the entire sorting process. That said,
generating a sort key + sorting it in the C locale is generally still
5-10x faster than using `order()` directly. If performance is critical,
users can specify `.locale = "C"` to get the maximum benefits of radix
ordering.

## Implementation

-   Using `vec_order()` in `arrange()`, and adding `.locale`

    -   <https://github.com/tidyverse/dplyr/pull/5942>

-   Renaming `vec_order_radix()` to `vec_order()`

    -   <https://github.com/r-lib/vctrs/pull/1435>

## Backwards Compatibility

### arrange()

The proposal outlined above is purposefully as conservative as possible,
preserving the results of programs using the American English locale,
which is the most widely used locale in R, while sacrificing a bit of
performance from the generation of the sort key.

That said, this proposal will impact non-English Latin script languages.
For example, in a majority of Latin script languages, including `"en"`,
ø sorts after o, but before p. However, a few languages, such as Danish,
sort ø as a unique character after z. Danish users that have
`LC_COLLATE` set to Danish may be surprised that `arrange()` would now
be placing ø in the “wrong order” even though they have set that global
option. The fix would be to either set `.locale = "da"` in their calls
to `arrange()`, or to set `options(dplyr.locale = "da")` to override
this default globally.

``` r
library(dplyr) # tidyverse/dplyr#5942
library(withr)

tbl <- tibble(x = c("ø", "o", "p", "z"))

# `"en"` default
arrange(tbl, x)
```

    ## # A tibble: 4 x 1
    ##   x    
    ##   <chr>
    ## 1 o    
    ## 2 ø    
    ## 3 p    
    ## 4 z

``` r
# Set the locale in `arrange()` directly
arrange(tbl, x, .locale = "da")
```

    ## # A tibble: 4 x 1
    ##   x    
    ##   <chr>
    ## 1 o    
    ## 2 p    
    ## 3 z    
    ## 4 ø

### arrange\_at/if/all()

While these three variants of `arrange()` are superseded, we have
decided to add a `.locale` argument to each of them anyways.

## Unresolved Questions

stringi provides various options for fine tuning the sorting method
through `stringi::stri_opts_collator()`. The most useful of these is
`numeric`, which allows for natural sorting of strings containing a mix
of alphabetical and numeric characters.

``` r
library(stringi)

x <- c("A1", "A100", "A2")

# Compares the 2nd character of each string as 1 <= 1 <= 2
stri_sort(x, locale = "en")
```

    ## [1] "A1"   "A100" "A2"

``` r
# Compares 1 <= 2 <= 100
opts <- stri_opts_collator(locale = "en", numeric = TRUE)
stri_sort(x, opts_collator = opts)
```

    ## [1] "A1"   "A2"   "A100"

Feedback suggested that it might be useful to allow `.locale` to accept
a `stri_opts_collator()` list to fine tune the procedure used by
`arrange()`, but ultimately we decided not to add this at this time
because most of the arguments to `stri_opts_collator()` are extreme
special cases. We may return to this in the future if it proves to be a
popular request.

## Alternatives

### No global option

The original proposal defaulted to the American English locale, but did
not include a way to globally override this. This had the benefit of
being more reproducible across sessions, since as long as stringi was
installed it was guaranteed that the locale was always American English
unless specified otherwise through `.locale`. However, in the tidyverse
meeting on 2021-06-14 it was determined that not including a way to
globally override this default was too aggressive of a change for
non-English Latin script users. With a large script containing multiple
calls to `arrange()`, each call would have to be updated. Additionally,
if a function from a package that a user didn’t control called
`arrange()` internally, then without a global option they would have no
way to adjust the locale until that package author exposed the new
`.locale` argument. Ultimately, in this case the benefits of the global
option outweigh the potential reproducibility issues that might arise
from it.

### Defaulting to the C locale

Another alternative is to default `arrange()` to the C locale, while
still allowing users to specify `.locale` for ordering in alternative
locales.

-   This has the benefit of making it clearer that stringi is an
    optional dependency, which would only be used if a user requests a
    specific locale identifier. The default behavior would never require
    stringi.

-   Additionally, the performance improvements would be even more
    substantial since no sort key would be generated by default.

-   However, this would have the potential to alter nearly every call to
    `arrange()`, since the C locale is not identical to the American
    English locale. In particular, in the C locale all capital letters
    are ordered before lowercase ones, such as: `c(A, B, a, b)` , while
    in the American English locale letters are first grouped together
    regardless of capitalization, and then lowercase letters are placed
    before uppercase ones, like: `c(a, A, b, B)`. This may look like a
    small difference, but it would surprise enough users to justify not
    defaulting to the C locale.

-   This is also not as consistent with stringr, which defaults to
    `"en"`.

### Tagged character vectors

A final proposed alternative is to implement a “tagged” character vector
class, with an additional attribute specifying the locale to be used
when ordering. This would remove the need for any `.locale` argument,
and the locale would even be allowed to vary between columns. If no
locale tag was supplied, `arrange()` would default to either `"en"` or
`"C"` for the locale. This approach is relatively clean, but is
practically very difficult because it would require cross package effort
to generate and retain these locale tags. Additionally, it doesn’t solve
the problem of avoiding breakage for existing code that uses a
non-English locale. Lastly, it would require an additional learning
curve for users to understand how to use them in conjunction with
`arrange()`.
