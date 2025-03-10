
# Tidyup 6: Ordering of `dplyr::group_by()`

**Champion**: Davis

**Co-Champion**: Hadley

**Status**: Accepted

## Abstract

The current algorithm used by `group_by()` computes the group locations
and orders the unique grouping keys in two separate steps. Recently,
vctrs has gained a new function, `vec_locate_sorted_groups()`, that can
efficiently do both of these operations in one step. The purpose of this
tidyup is to propose switching `group_by()`’s internal algorithm to use
`vec_locate_sorted_groups()`, outlining potential drawbacks from doing
so.

## Motivation

``` r
library(vctrs) # r-lib/vctrs#1441
library(withr)
set.seed(42)
```

Building on the fast radix ordering function, `vec_order()`, mentioned
in [Radix Ordering in
`dplyr::arrange()`](https://github.com/tidyverse/tidyups/blob/main/003-dplyr-radix-ordering.md),
vctrs has gained another new function, `vec_locate_sorted_groups()`,
that efficiently sorts and returns the locations corresponding to each
occurrence of the unique values found in a vector.

``` r
x <- data_frame(x = c(1, 2, 1, 1, 2), y = c("a", "b", "b", "a", "b"))
x
```

    ##   x y
    ## 1 1 a
    ## 2 2 b
    ## 3 1 b
    ## 4 1 a
    ## 5 2 b

``` r
# The row containing (1, a) occurs twice, at row 1 and row 4
vec_locate_sorted_groups(x)
```

    ##   key.x key.y  loc
    ## 1     1     a 1, 4
    ## 2     1     b    3
    ## 3     2     b 2, 5

The result of this function exactly matches the internal structure used
by dplyr for carrying grouping information around, and is often faster
than the current two step method used by dplyr, especially with many
groups.

``` r
# Current algorithm
dplyr_locate_sorted_groups <- function(x) {
  out <- vec_group_loc(x)
  vec_slice(out, vec_order_base(out$key))
}
```

``` r
# 10 million rows, with 1 million unique grouping values
n_row <- 1e7
n_unique_groups <- 1e6
grouping_column <- sample(n_unique_groups, n_row, replace = TRUE)

bench::mark(
  dplyr = dplyr_locate_sorted_groups(grouping_column),
  vctrs = vec_locate_sorted_groups(grouping_column),
  iterations = 5
)
#> # A tibble: 2 × 6
#>   expression      min   median `itr/sec` mem_alloc `gc/sec`
#>   <bch:expr> <bch:tm> <bch:tm>     <dbl> <bch:byt>    <dbl>
#> 1 dplyr         1.67s    2.13s     0.481     186MB    0.673
#> 2 vctrs      213.44ms 232.12ms     3.15      189MB    5.05
```

It is particularly useful with character columns containing many unique
values.

``` r
# 10 million rows, with 1 million unique grouping values
n_row <- 1e7
n_unique_groups <- 1e6
unique_values <- stringi::stri_rand_strings(n_unique_groups, length = 10)
grouping_column <- sample(unique_values, n_row, replace = TRUE)

bench::mark(
  dplyr = dplyr_locate_sorted_groups(grouping_column),
  vctrs = vec_locate_sorted_groups(grouping_column),
  iterations = 2,
  check = FALSE
)
#> # A tibble: 2 × 6
#>   expression      min   median `itr/sec` mem_alloc `gc/sec`
#>   <bch:expr> <bch:tm> <bch:tm>     <dbl> <bch:byt>    <dbl>
#> 1 dplyr         10.7s   10.97s    0.0911     194MB    0.137
#> 2 vctrs          1.3s    1.35s    0.742      308MB    1.11
```

The reason for the improved performance in the character columns is also
the largest potential issue with this proposed change. The current
algorithm used by dplyr respects the system locale when ordering
character vectors, but the one used by vctrs unconditionally orders
character vector groups in the C locale. Ordering in the C locale is
much faster, but often produces different results from the system locale
(which might be set to English, French, etc.).

This change in group ordering would be seen when a user calls
`summarise()` after grouping (or any other function that uses the
`group_data()`). If the C locale differs from their system locale, then
the grouping columns will be returned in a different order than before.
For English users, the main difference with the C locale is in how
uppercase vs lowercase letters are ordered:

``` r
library(dplyr)

df <- tibble(g = c("a", "A", "B", "b", "A", "b"))
gdf <- group_by(df, g)

# With CRAN dplyr
summarise(gdf, n = n())
#> # A tibble: 4 × 2
#>   g         n
#>   <chr> <int>
#> 1 a         1
#> 2 A         2
#> 3 b         2
#> 4 B         1

# With the changes in this tidyup
summarise(gdf, n = n())
#> # A tibble: 4 × 2
#>   g         n
#>   <chr> <int>
#> 1 A         2
#> 2 B         1
#> 3 a         1
#> 4 b         2
```

## Solution

We feel that the potential performance benefits from switching to
`vec_locate_sorted_groups()` outweigh the issues that could arise from
`group_by()` ordering in the C locale, so the proposed plan is to switch
out the current `group_by()` algorithm for the new vctrs implementation,
which will begin to order character groups in the C locale
unconditionally. We expect all other types to retain the same ordering,
having improved performance everywhere, especially when there are many
groups.

Documentation for both `summarise()` and `group_by()` would be updated
to mention the new behavior with character vectors, and would encourage
the usage of `arrange()` after `summarise()` if the returned ordering is
important. A pre-release blog post would also mention this change.

To ease the transition, a new *temporary* global option,
`dplyr.legacy_group_by_locale`, will be added and documented inside
`group_by()`. If set to `TRUE`, the old grouping algorithm will be used,
which will again respect the system locale. This should be used
*extremely* sparingly, and we only expect this to be used in long
analysis scripts for a quick fix. In a future minor version of dplyr,
this option will be deprecated and ultimately removed, so it is
encouraged that code be updated with an explicit `arrange()` call after
the grouped operation rather than using this option.

## Implementation

-   vctrs PR for exporting `vec_locate_sorted_groups()`

    -   <https://github.com/r-lib/vctrs/pull/1441>

-   dplyr PR for converting to `vec_locate_sorted_groups()`

    -   <https://github.com/tidyverse/dplyr/pull/6018>

## Backwards compatibility

### Code breakage

It is difficult to know how much “in the wild” user code will break as a
result of changing from using the system locale to the C locale in
`group_by()`. Ideally, users won’t be relying on row order for most of
their analysis scripts, but we are empathetic to the fact that this is
not always the case. Our plan is to run reverse dependency checks and
help fix any package code with issues that arise from this, and to
provide a pre-release blog post noting this change so that users can
update their code before this version releases. Combined with the
`dplyr.legacy_group_by_locale` global option, we hope that these
precautions will make the transition period as painless as possible.

### Usage with `arrange()`

Initially, we worried that the following snippet of code would cause
some confusion:

``` r
df %>%
  arrange(chr, .locale = "en") %>%
  group_by(chr) %>% # internally would sort with C locale
  summarise() # result ordered in C locale
```

Prior to dplyr 1.1.0, `arrange()` and `group_by()` both used the system
locale, so they always sorted in a consistent way, meaning that this was
never an issue.

After gathering some
[feedback](https://github.com/tidyverse/tidyups/issues/21#issuecomment-917958890),
it seems that arranging directly before computing a grouped summary is
rather rare. Arranging *after* the summary is much more common since it:

-   Matches hows this is done in SQL

-   Often requires a modifier like `desc()` to tweak the ordering

-   Is often the last step in a pipeline, used only for better
    readability

Because of the assumed rarity of the appearance of this code chunk, we
are willing to accept the (hopefully) minimal amount of confusion it may
cause in favor of the benefits we get from this change.

## Alternatives

### Order by first appearance

We briefly considered letting `group_by()` internally compute groups
ordered by first appearance. This has a number of benefits, outlined in
[this issue
comment](https://github.com/tidyverse/dplyr/issues/5664#issuecomment-907232443),
but was ultimately determined to be too large of a change from the
current behavior. This is something that we may revisit in a potential
second edition of dplyr.

### Give `group_by()` a `.locale` argument

In [Radix Ordering
in](https://github.com/tidyverse/tidyups/blob/main/003-dplyr-radix-ordering.md)`dplyr::arrange()`,
a `.locale` argument was added to `arrange()` to explicitly control the
locale used while sorting. An argument could be made to add a `.locale`
argument to `group_by()` as well, which would allow users to change to
something other than the C locale while grouping. However, because
`group_by()` returns a grouped data frame that has to recompute its
group information at various times, many other functions besides
`group_by()` would also have to gain a `.locale` argument. Additionally,
for methods like `[.grouped_df`, we would need some way to “remember”
that the user originally called `group_by()` with a particular locale,
so that we can recompute groups in the same way. Ultimately this was
deemed too complicated for not enough benefit.
