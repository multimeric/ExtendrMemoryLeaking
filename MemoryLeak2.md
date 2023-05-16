MemoryLeak2
================
2023-05-09

``` r
library(rextendr)
```

The idea of this function is to allocate a large vector, pass it to an
extendr function, then clean everything up and see if there is still an
overall increase in memory (a leak).

``` r
test_for_leaks <- function(func, make_vec){
  initial = lobstr::mem_used()
  cli::cli_alert_info("{format(initial)} used before allocation")
  vec = make_vec()
  cli::cli_alert_info("{lobstr::mem_used() |> format()} used after allocation")
  func(vec) |> try(silent = TRUE)
  cli::cli_alert_info("{lobstr::mem_used() |> format()} used after function call")
  rm(vec)
  gc(verbose = FALSE)
  final = lobstr::mem_used()
  cli::cli_alert_info("{lobstr::mem_used() |> format()} used after gc")
  if ((final - initial) > 1E7){
    cli::cli_alert_danger("Leak detected!")
  }
}
```

A baseline for Rust:

``` rust
#[extendr(use_try_from = true)]
fn identity_rust(x: Robj) -> Robj { x }
```

Panic, and no `try_from`:

``` rust
#[extendr]
fn throw_1(x: Robj) { panic!() }
```

Panic, and yes `try_from`:

``` rust
#[extendr(use_try_from = true)]
fn throw_2(x: Robj) { panic!() }
```

Throwing an error that doesn’t hold a Robj, with `try_from`:

``` rust
#[extendr(use_try_from = true)]
fn throw_3(x: Robj) -> Result<()> {
  Err(Error::Other(String::from("")))
}
```

Throwing an error that *does* hold a Robj, with `try_from`:

``` rust
#[extendr(use_try_from = true)]
fn throw_4(x: Robj) -> Result<()> {
  Err(Error::ExpectedList(x))
}
```

A baseline that doesn’t even use extendr:

``` r
identity_r <- function(x) x
```

These functions each produce a large but random vector of each of the 4
primitive types:

``` r
vecs = tibble::tribble(
  ~vec_name, ~make_vec,
  "integer", function() (1:1E7)[],
  "double", function() rnorm(1E7),
  "character", function() as.character(rnorm(1E7)),
  "logical", function() as.logical(rnorm(1E7))
)
```

Collect together all the functions we want to test:

``` r
funcs = tibble::tribble(
  ~func_name, ~func,
  "r_identity", identity_r,
  "rust_identity", identity_rust,
  "throw_1", throw_1,
  "throw_2", throw_2,
  "throw_3", throw_3
)
```

And the main test loop. For each combination of test function and input
vector, we test for leaks:

``` r
tidyr::expand_grid(vecs, funcs) |>
  purrr::pmap(function(vec_name, make_vec, func_name, func){
      cli::cli_alert_info("Testing {func_name} + {vec_name} for leaks")
      test_for_leaks(func, make_vec)
  })
```

    ## ℹ Testing r_identity + integer for leaks

    ## ℹ 61.00 MB used before allocation

    ## ℹ 101.27 MB used after allocation

    ## ℹ 101.29 MB used after function call

    ## ℹ 61.29 MB used after gc

    ## ℹ Testing rust_identity + integer for leaks

    ## ℹ 61.31 MB used before allocation

    ## ℹ 101.35 MB used after allocation

    ## ℹ 101.35 MB used after function call

    ## ℹ 61.35 MB used after gc

    ## ℹ Testing throw_1 + integer for leaks

    ## ℹ 61.34 MB used before allocation

    ## ℹ 101.35 MB used after allocation

    ## ℹ 101.35 MB used after function call

    ## ℹ 101.35 MB used after gc

    ## ✖ Leak detected!

    ## ℹ Testing throw_2 + integer for leaks

    ## ℹ 101.37 MB used before allocation

    ## ℹ 141.38 MB used after allocation

    ## ℹ 141.38 MB used after function call

    ## ℹ 101.38 MB used after gc

    ## ℹ Testing throw_3 + integer for leaks

    ## ℹ 101.37 MB used before allocation

    ## ℹ 141.38 MB used after allocation

    ## ℹ 141.38 MB used after function call

    ## ℹ 101.38 MB used after gc

    ## ℹ Testing r_identity + double for leaks

    ## ℹ 101.37 MB used before allocation

    ## ℹ 181.39 MB used after allocation

    ## ℹ 181.39 MB used after function call

    ## ℹ 101.39 MB used after gc

    ## ℹ Testing rust_identity + double for leaks

    ## ℹ 101.38 MB used before allocation

    ## ℹ 181.39 MB used after allocation

    ## ℹ 181.39 MB used after function call

    ## ℹ 101.39 MB used after gc

    ## ℹ Testing throw_1 + double for leaks

    ## ℹ 101.38 MB used before allocation

    ## ℹ 181.39 MB used after allocation

    ## ℹ 181.40 MB used after function call

    ## ℹ 181.40 MB used after gc

    ## ✖ Leak detected!

    ## ℹ Testing throw_2 + double for leaks

    ## ℹ 181.39 MB used before allocation

    ## ℹ 261.40 MB used after allocation

    ## ℹ 261.40 MB used after function call

    ## ℹ 181.40 MB used after gc

    ## ℹ Testing throw_3 + double for leaks

    ## ℹ 181.39 MB used before allocation

    ## ℹ 261.40 MB used after allocation

    ## ℹ 261.40 MB used after function call

    ## ℹ 181.40 MB used after gc

    ## ℹ Testing r_identity + character for leaks

    ## ℹ 181.39 MB used before allocation

    ## ℹ 261.41 MB used after allocation

    ## ℹ 261.41 MB used after function call

    ## ℹ 181.41 MB used after gc

    ## ℹ Testing rust_identity + character for leaks

    ## ℹ 181.40 MB used before allocation

    ## ℹ 261.41 MB used after allocation

    ## ℹ 261.41 MB used after function call

    ## ℹ 181.41 MB used after gc

    ## ℹ Testing throw_1 + character for leaks

    ## ℹ 181.40 MB used before allocation

    ## ℹ 261.41 MB used after allocation

    ## ℹ 261.41 MB used after function call

    ## ℹ 261.41 MB used after gc

    ## ✖ Leak detected!

    ## ℹ Testing throw_2 + character for leaks

    ## ℹ 261.40 MB used before allocation

    ## ℹ 341.41 MB used after allocation

    ## ℹ 341.41 MB used after function call

    ## ℹ 261.41 MB used after gc

    ## ℹ Testing throw_3 + character for leaks

    ## ℹ 261.40 MB used before allocation

    ## ℹ 341.42 MB used after allocation

    ## ℹ 341.42 MB used after function call

    ## ℹ 261.42 MB used after gc

    ## ℹ Testing r_identity + logical for leaks

    ## ℹ 261.41 MB used before allocation

    ## ℹ 301.42 MB used after allocation

    ## ℹ 301.42 MB used after function call

    ## ℹ 261.42 MB used after gc

    ## ℹ Testing rust_identity + logical for leaks

    ## ℹ 261.41 MB used before allocation

    ## ℹ 301.42 MB used after allocation

    ## ℹ 301.42 MB used after function call

    ## ℹ 261.42 MB used after gc

    ## ℹ Testing throw_1 + logical for leaks

    ## ℹ 261.41 MB used before allocation

    ## ℹ 301.42 MB used after allocation

    ## ℹ 301.42 MB used after function call

    ## ℹ 301.43 MB used after gc

    ## ✖ Leak detected!

    ## ℹ Testing throw_2 + logical for leaks

    ## ℹ 301.42 MB used before allocation

    ## ℹ 341.43 MB used after allocation

    ## ℹ 341.43 MB used after function call

    ## ℹ 301.43 MB used after gc

    ## ℹ Testing throw_3 + logical for leaks

    ## ℹ 301.42 MB used before allocation

    ## ℹ 341.43 MB used after allocation

    ## ℹ 341.43 MB used after function call

    ## ℹ 301.43 MB used after gc
