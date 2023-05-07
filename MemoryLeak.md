MemoryLeaks
================
2023-05-04

``` r
knitr::opts_chunk$set(
  engine.opts = list(
    extendr_deps = list(
      `extendr-api` = list(
        git = "https://github.com/yutannihilation/extendr", 
        branch = "poc/drop-error"
      )
    )
  )
)
```

``` r
library(rextendr)
```

The below function reports memory usage at various times during the call
of a Rust function. By default we use an integer vector as the input,
but the `vec` argument is customized ocasionally.

**Note that the actual error messages from extendr are shown here**.

``` r
test_for_leaks <- function(func, vec = (1:10E7)[]){
  lobstr::mem_used() |> format() |> paste(" used before allocation") |> print()
  force(vec)
  lobstr::mem_used() |> format() |> paste(" used after allocation") |> print()
  func(vec) |> try(silent = TRUE) 
  lobstr::mem_used() |> format() |> paste(" used after panic") |> print()
  rm(vec)
  gc(verbose = FALSE)
  lobstr::mem_used() |> format() |> paste(" used after gc") |> print()
}
```

# Case 1

- `use_try_from` is false
- Panic with no message

``` rust
#[extendr]
fn throw_1(x: Robj) {
    panic!()
}
```

``` r
test_for_leaks(throw_1)
```

    ## [1] "57.14 MB  used before allocation"
    ## [1] "457.43 MB  used after allocation"
    ## [1] "457.46 MB  used after panic"
    ## [1] "457.46 MB  used after gc"

Definitely a leak.

# Case 2

- `use_try_from` is true
- Panic with no message

``` rust
#[extendr(use_try_from = true)]
fn throw_2(x: Robj) {
    panic!()
}
```

``` r
test_for_leaks(throw_2)
```

    ## [1] "458.33 MB  used before allocation"
    ## [1] "858.33 MB  used after allocation"
    ## [1] "858.33 MB  used after panic"
    ## [1] "458.33 MB  used after gc"

No leak!

``` r
test_for_leaks(throw_2, vec = rep(LETTERS, 1E7))
```

    ## [1] "458.33 MB  used before allocation"
    ## [1] "2.54 GB  used after allocation"
    ## [1] "2.54 GB  used after panic"
    ## [1] "2.54 GB  used after gc"

But when using a string vector, it seems to leak.

# Case 3

- `use_try_from` is true
- Return string error

``` rust
#[extendr(use_try_from = true)]
fn throw_3(x: Robj) -> Result<()> {
  Err(Error::Other(String::from("")))
}
```

``` r
test_for_leaks(throw_3)
```

    ## [1] "459.15 MB  used before allocation"
    ## [1] "859.14 MB  used after allocation"
    ## [1] "859.14 MB  used after panic"
    ## [1] "459.14 MB  used after gc"

Also no leak!

``` r
test_for_leaks(throw_3, vec = rep(LETTERS, 1E7))
```

    ## [1] "459.14 MB  used before allocation"
    ## [1] "2.54 GB  used after allocation"
    ## [1] "2.54 GB  used after panic"
    ## [1] "2.54 GB  used after gc"

As above, a string vector leaks.

# Case 4

- `use_try_from` is true
- Return error that takes a Robj

``` rust
#[extendr(use_try_from = true)]
fn throw_4(x: Robj) -> Result<()> {
  Err(Error::ExpectedList(x))
}
```

``` r
test_for_leaks(throw_4, vec = (1:1E5)[])
```

    ## [1] "459.95 MB  used before allocation"
    ## [1] "460.34 MB  used after allocation"
    ## [1] "460.34 MB  used after panic"
    ## [1] "460.34 MB  used after gc"

It’s a bit hard to see from the small memory increase, but this is a
leak.

``` r
test_for_leaks(throw_4, vec = rep(LETTERS, 1E4))
```

    ## [1] "459.94 MB  used before allocation"
    ## [1] "462.02 MB  used after allocation"
    ## [1] "462.02 MB  used after panic"
    ## [1] "462.02 MB  used after gc"

As above.
