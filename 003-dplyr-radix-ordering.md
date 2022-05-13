
# Tidyup 3: Radix Ordering in `dplyr::arrange()`

**Champion**: Davis

**Co-Champion**: Hadley

**Status**: Accepted

## Abstract

As of dplyr 1.0.9, `arrange()` uses a modified version of
`base::order()` to sort columns of a data frame. Recently, vctrs gained
`vec_order_radix()`, a fast radix ordering function with first class
support for data frames and custom vctrs types, along with enhanced
character ordering. The purpose of this tidyup is to propose switching
`arrange()` from `order()` to `vec_order_radix()` in the most
user-friendly way possible.

## Motivation

Thanks to the data.table team, R \>= 3.3 gained support for an extremely
fast radix ordering algorithm in `order()`. This has become the default
algorithm for ordering most atomic types, with the notable exception of
character vectors. Radix ordering character vectors is only possible in
the C locale, but the shell sort currently in use by `order()` respects
the system locale. Because R is justifiably hesitant to break backwards
compatibility, if *any* character vector is present, the entire ordering
procedure falls back to a much slower shell sort. Because dplyr uses
`order()` internally, the performance of `arrange()` is negatively
affected by this fallback.

Inspired by the performance of the radix ordering algorithm, and by its
many practical applications for data science, a radix order-based
`vec_order_radix()` was added to vctrs, which has the following
benefits:

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
library(vctrs)
vec_order_radix <- vctrs:::vec_order_radix
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
# Force `order()` to use the C locale to match `vec_order_radix()`
bench::mark(
  base = withr::with_locale(c(LC_COLLATE = "C"), order(x)),
  vctrs = vec_order_radix(x)
)
```

    ## # A tibble: 2 × 6
    ##   expression      min   median `itr/sec` mem_alloc `gc/sec`
    ##   <bch:expr> <bch:tm> <bch:tm>     <dbl> <bch:byt>    <dbl>
    ## 1 base          1.61s    1.61s     0.622    8.34MB      0  
    ## 2 vctrs       32.89ms  36.96ms    27.4      12.7MB     32.0

``` r
# Force `vec_order_radix()` to use the American English locale, which is also
# my system locale. To do that we'll need to generate a sort key, which
# we can sort in the C locale, but the result will be like we sorted
# directly in the American English locale.
bench::mark(
  base = order(x),
  vctrs = vec_order_radix(
    x = x, 
    chr_proxy_collate = ~stri_sort_key(.x, locale = "en_US")
  )
)
```

    ## Warning: Some expressions had a GC in every iteration; so filtering is disabled.

    ## # A tibble: 2 × 6
    ##   expression      min   median `itr/sec` mem_alloc `gc/sec`
    ##   <bch:expr> <bch:tm> <bch:tm>     <dbl> <bch:byt>    <dbl>
    ## 1 base          6.25s    6.25s     0.160    3.81MB     0   
    ## 2 vctrs      661.42ms 661.42ms     1.51    20.21MB     1.51

``` r
# Generating the sort key takes most of the time
bench::mark(
  sort_key = stri_sort_key(x, locale = "en_US")
)
```

    ## # A tibble: 1 × 6
    ##   expression      min   median `itr/sec` mem_alloc `gc/sec`
    ##   <bch:expr> <bch:tm> <bch:tm>     <dbl> <bch:byt>    <dbl>
    ## 1 sort_key      609ms    609ms      1.64    7.63MB        0

In dplyr, we’d like to utilize `vec_order_radix()` while breaking as
little code as possible. Switching to `vec_order_radix()` has many
potential positive impacts, including:

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
    stringr, which has an explicit argument for changing the locale,
    i.e. `stringr::str_order(locale = "en")` that utilizes the locale
    identifiers from stringi.

However, making this switch also has potential negative impacts:

-   Breaking code that relies strongly on the ordering resulting from
    using the `LC_COLLATE` environment variable.

-   Surprising users if the new ordering does not match the previous
    one.

## Solution

To switch to `vec_order_radix()` internally while surprising the least
amount of users, it is proposed that the data frame method for
`arrange()` gain a new argument, `.locale`, with the following
properties:

-   Defaults to `dplyr_locale()`, which typically returns the `"C"`
    locale, but can be overriden, see below.
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

-   It defaults to returning `"C"`, for the C locale.

-   Alternatively, to globally override the above default behavior, the
    global option, `dplyr.locale`, can be set to either `"C"` or a
    string locale identifier. Setting this to anything except `"C"`
    would require stringi.

C has been chosen as the default because it is the most reproducible
option. The C locale is available in all versions of R and across all
operating systems, and works the same everywhere. This satisfies one of
the main goals of this tidyup, improving the reproducibility of
`arrange()` when compared with the current behavior of `LC_COLLATE`. The
C locale is also fairly close to the American English locale
(i.e. `"en"`), with the main difference being how case-sensitivity is
treated, but this rarely makes a practical difference in a data
analysis.

The global option `dplyr.locale` has been provided as an “escape hatch”
that should be used sparingly, as it has the potential to reduce
reproducibility across R sessions. It has two main uses:

-   Quickly adapting an existing script to the new behavior of
    `arrange()` by overriding the locale at the top of the script to
    switch back to the previous behavior. In the long term, we suggest
    explicitly supplying `.locale` in all affected calls to `arrange()`
    instead.

-   Locally overriding the locale when `arrange()` is called from within
    a function that you don’t control. In this case, we recommend using
    `rlang::with_options()` to limit the scope in which the locale is
    overriden.

Feedback has indicated that while a global option can decrease
reproducibility between sessions, in this case the benefits of it
outweigh the costs.

On certain systems, stringi can be a difficult dependency to install.
Because of this, this proposal recommends that stringi only be
*suggested* so that users without stringi can still use dplyr.

This proposal relies on `stringi::stri_sort_key()` when a stringi locale
identifier is supplied, which generates the sort key mentioned under
Motivation as a proxy that can be ordered in the C locale. However, sort
key generation is expensive. In fact, it is often the most expensive
part of the entire sorting process, which is one of the reasons that the
C locale is the default. That said, generating a sort key + sorting it
in the C locale is generally still 5-10x faster than using `order()`
directly.

## Implementation

-   Using `vec_order_radix()` in `arrange()`, and adding `.locale`

    -   <https://github.com/tidyverse/dplyr/pull/6263>

## Backwards Compatibility

### arrange()

The proposal outlined above should preserve the results of most programs
using the American English locale. The main difference between the C
locale and American English is related to case-sensitivity. In the C
locale, the English alphabet is grouped by *case*, while in English
locales the alphabet is grouped by *letter*:

``` r
library(dplyr) # tidyverse/dplyr#6263

df <- tibble(x = c("a", "b", "C", "B", "c"))
df
```

    ## # A tibble: 5 × 1
    ##   x    
    ##   <chr>
    ## 1 a    
    ## 2 b    
    ## 3 C    
    ## 4 B    
    ## 5 c

``` r
# The C locale groups the English alphabet by case, placing uppercase letters
# before lowercase letters
arrange(df, x)
```

    ## # A tibble: 5 × 1
    ##   x    
    ##   <chr>
    ## 1 B    
    ## 2 C    
    ## 3 a    
    ## 4 b    
    ## 5 c

``` r
# The American English locale groups the alphabet by letter
arrange(df, x, .locale = "en")
```

    ## # A tibble: 5 × 1
    ##   x    
    ##   <chr>
    ## 1 a    
    ## 2 b    
    ## 3 B    
    ## 4 c    
    ## 5 C

This rarely has a practical difference in a data analysis. The C locale
will return identical results to the American English locale as long as
the case is consistent between observations, like with the following two
IDs: `"AD25" < "AG66"`, or with proper nouns, which are often either
capitalized consistently or not capitalized at all:
`"America" < "Japan"` or `"america" < "japan"`.

With non-English Latin script languages, we expect there to be more
meaningful differences. For example, in Spanish the `n` with a tilde
above it is sorted directly after `n`, but in the C locale this is
placed after `z`.

``` r
tbl <- tibble(x = c("\u00F1", "n", "z"))
tbl
```

    ## # A tibble: 3 × 1
    ##   x    
    ##   <chr>
    ## 1 ñ    
    ## 2 n    
    ## 3 z

``` r
arrange(tbl, x)
```

    ## # A tibble: 3 × 1
    ##   x    
    ##   <chr>
    ## 1 n    
    ## 2 z    
    ## 3 ñ

``` r
arrange(tbl, x, .locale = "es")
```

    ## # A tibble: 3 × 1
    ##   x    
    ##   <chr>
    ## 1 n    
    ## 2 ñ    
    ## 3 z

Spanish users that have `LC_COLLATE` set to Spanish may be surprised
that `arrange()` would now be placing this character in the “wrong
order” even though they have set that global option. The fix would be to
either set `.locale = "es"` in their calls to `arrange()`, or to set
`options(dplyr.locale = "es")` to override this default globally. We
expect this issue to affect a small number of users, due to the sheer
magnitude of users that use English as their system locale.

### arrange_at/if/all()

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

### Defaulting to the American English locale if stringi was installed

A previous version of this proposal changed the behavior of
`dplyr_locale()` so that it:

-   Defaulted to `"en"` if stringi was installed

-   Fell back to `"C"` with a warning if stringi wasn’t installed

-   Still allowed `dplyr.locale` to override this behavior

This seemed reasonable since most R users use an English locale, and
that would mean their scripts had very little chance of breaking from
this change. However, on 2022-05-11 we realized that in practice this
would cause more confusion than it is worth. In particular, if you are a
package developer relying on dplyr, then the default behavior of
`arrange()` would change for you depending on whether or not stringi was
installed. This would result in difficult to debug situations where you
might locally write tests with stringi installed, but then on CI your
tests might fail or throw warnings if you forget to `Import` stringi
(even if you weren’t working with character columns!). By defaulting to
the C locale, the default behavior of `arrange()` will work the same
everywhere, and if package developers explicitly set `.locale` to
anything else, such as `"en"`, then this will be a clear signal that
they have opted into an optional behavior and need to add stringi as a
dependency of their package.

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
