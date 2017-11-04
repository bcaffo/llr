[![Travis-CI Build Status](https://travis-ci.org/dirkschumacher/llr.svg?branch=master)](https://travis-ci.org/dirkschumacher/llr) [![AppVeyor Build Status](https://ci.appveyor.com/api/projects/status/github/dirkschumacher/llr?branch=master&svg=true)](https://ci.appveyor.com/project/dirkschumacher/llr) [![Coverage Status](https://img.shields.io/codecov/c/github/dirkschumacher/llr/master.svg)](https://codecov.io/github/dirkschumacher/llr?branch=master)

llr
===

Lisp-like-R (llr). A work in progress, *just for fun* package to implement a lisp interpreter in R. The idea is to write a LISP that compiles to R's abstract syntax tree. It is implemented more as an exercise for me to better understand LISP, but already works quite ok.

All of this is very experimental :)

It also helps that R itself is a bit of a lisp with a non-lisp syntax:

> We started by writing a small lisp interpreter. Next we expanded its data structures with atomic vector types and altered its evaluation semantics to include lazy evaluation of closure arguments and argument binding by tag as well as order. Finally we added some syntactic sugar to make it look somewhat like S. We call the result "R". [An Updated History of R](http://blog.revolutionanalytics.com/2017/10/updated-history-of-r.html)

Install
-------

``` r
devtools::install_github("dirkschumacher/llr")
```

Features
--------

### Datatypes

``` r
llr("[1 2 3]") # list
llr("(quote (1 2 3))") # list
llr("1:10") # vector
llr("(seq 0 1 0.1)") # a single numeric vector (no list)
llr("pi") # single value
llr("0.3")
llr("3")
llr("3L")
```

### Special forms

``` r
llr("(def x 1)") # bind a value to a symbol
llr("(defmacro hello [name] (paste0 \"Hello \" name))") # create a macro
```

### Functions

``` r
llr("(fn [a] (> a 1))") # anonymous function
llr("(utils::head [1, 2, 3] 1)") # namspaced functions
llr("(Sys.Date)") # function without arguments
```

It does not yet support variadic functions, meaning functions with a variable number of arguments. Also at the moment you cannot yet define functions with `...` directly in R.

### Macros

LLR supports macros and expands these at compile time. Macros can be both written in R and LLR and work on R's data structures. You can use macros extend the language. In the example below we write a macro that rewrites `(a + b)` to `(+ a b)` at compile time, so you can write your binary additions in infix notation.

With `quote` you can quote code (i.e. prevent it from being evaluated) and with `UQ`, you can unquote it. Those two constructs can be used to modify data structures. Powered by [rlang](https://github.com/tidyverse/rlang).

``` r
# a macro that evaluates expressions in infix notation
# just for binary operations to make it simple
# also R <-> LLR interop
first <- function(x) x[[1L]]
rest <- function(x) tail(x, length(x) - 1L)
llr("
(defmacro infix [code]
  (quote
    ((UQ (first (rest code)))
     (UQ (first code))
     (UQ (first (rest (rest code)))))))    
(infix (40 + 2))
")
#> [1] 42
llr("((fn [x] (infix (2 + x))) 40)")
#> [1] 42
```

Examples
--------

``` r
llr("(Map (fn [x] (+ x 10)) (quote (1 2 3)))")
#> [[1]]
#> [1] 11
#> 
#> [[2]]
#> [1] 12
#> 
#> [[3]]
#> [1] 13
```

``` r
llr("
  (def x [1 2 3 4 5])
  (Reduce (fn [acc x] (+ acc x)) 
    (Filter (fn [y] (> y 20)) (Map (fn [x] (* x 10)) x)))
")
#> [1] 120
```

``` r
llr("
 (library tibble)
 (as_tibble (dplyr::filter mtcars (> hp 180) (> cyl 6)))
")
#> # A tibble: 7 x 11
#>     mpg   cyl  disp    hp  drat    wt  qsec    vs    am  gear  carb
#>   <dbl> <dbl> <dbl> <dbl> <dbl> <dbl> <dbl> <dbl> <dbl> <dbl> <dbl>
#> 1  14.3     8   360   245  3.21 3.570 15.84     0     0     3     4
#> 2  10.4     8   472   205  2.93 5.250 17.98     0     0     3     4
#> 3  10.4     8   460   215  3.00 5.424 17.82     0     0     3     4
#> 4  14.7     8   440   230  3.23 5.345 17.42     0     0     3     4
#> 5  13.3     8   350   245  3.73 3.840 15.41     0     0     3     4
#> 6  15.8     8   351   264  4.22 3.170 14.50     0     1     5     4
#> 7  15.0     8   301   335  3.54 3.570 14.60     0     1     5     8
```

``` r
llr("
  (def x 1:7)
  (as.numeric (purrr::keep x (fn [x] (> x 5))))
")
#> [1] 6 7
```

``` r
llr("pi")
#> [1] 3.141593
```

``` r
llr("
  (def some_fun (fn [] (runif 1)))
  (some_fun)
")
#> [1] 0.194763
```

``` r
llr("(dplyr::filter mtcars (> hp 150))")
#>     mpg cyl  disp  hp drat    wt  qsec vs am gear carb
#> 1  18.7   8 360.0 175 3.15 3.440 17.02  0  0    3    2
#> 2  14.3   8 360.0 245 3.21 3.570 15.84  0  0    3    4
#> 3  16.4   8 275.8 180 3.07 4.070 17.40  0  0    3    3
#> 4  17.3   8 275.8 180 3.07 3.730 17.60  0  0    3    3
#> 5  15.2   8 275.8 180 3.07 3.780 18.00  0  0    3    3
#> 6  10.4   8 472.0 205 2.93 5.250 17.98  0  0    3    4
#> 7  10.4   8 460.0 215 3.00 5.424 17.82  0  0    3    4
#> 8  14.7   8 440.0 230 3.23 5.345 17.42  0  0    3    4
#> 9  13.3   8 350.0 245 3.73 3.840 15.41  0  0    3    4
#> 10 19.2   8 400.0 175 3.08 3.845 17.05  0  0    3    2
#> 11 15.8   8 351.0 264 4.22 3.170 14.50  0  1    5    4
#> 12 19.7   6 145.0 175 3.62 2.770 15.50  0  1    5    6
#> 13 15.0   8 301.0 335 3.54 3.570 14.60  0  1    5    8
```

REPL
----

It also has a repl :)

``` r
llr::repl() # exit by typing (llr:exit)
```

[![asciicast](https://asciinema.org/a/129308.png)](https://asciinema.org/a/129308)

One more thing
--------------

You can use LLR directly in knitr documents thank's to knitrs language engines 🤗.

``` clojure
(Reduce + 
   (Map (fn [x] (* x 10)) [1 2 3 4 5]))
#> [1] 150
```

You can also communicate with other chunks through a shared environment:

``` clojure
(def x "hello other chunk")
#> [1] "hello other chunk"
```

``` clojure
(print x)
#> [1] "hello other chunk"
```

### More macro fun

#### Piping ggplot2

The following macro concatenates expressions by `+`.

``` clojure
(defmacro +> [...] 
  (Reduce (fn [acc el] 
            (quote (+ (UQ acc) (UQ el))))
    (rlang::dots_list ...)))
```

``` clojure
(library ggplot2)
(+> 
  (ggplot mtcars)
  (aes mpg cyl)
  (geom_point))    
```

#### Magrittr like piping

Another macro similiar to Clojure's [-&gt;](https://clojuredocs.org/clojure.core/-%3E) that works like `magrittr` pipes:

``` clojure
(defmacro => [...] 
    (Reduce (fn [acc el] 
      (quote 
        (do.call
          (UQ (first el))
          (UQ (c (list (UQ acc)) 
                 (unlist (as.list (UQ (rest el))) FALSE))))))
    (rlang::dots_list ...)))
```

``` clojure
(library dplyr)
(=>
  mtcars
  (filter (> hp 100) (> cyl 2))
  (group_by cyl)
  (summarise (n))) 
#> # A tibble: 3 x 2
#>     cyl `n()`
#>   <dbl> <int>
#> 1     4     2
#> 2     6     7
#> 3     8    14
```

Inspiration
-----------

-   [Peter Norvig's article](http://norvig.com/lispy.html)
-   [Make-a-lisp](https://github.com/kanaka/mal) - Great overview how to build a LISP. In particular I am currently using the regexp to tokenize the code.
-   [Clojure](https://clojure.org/)
-   [Hy](https://github.com/hylang/hy) - a lisp that compiles to pyhtons AST

Code of conduct
---------------

Please note that this project is released with a [Contributor Code of Conduct](CONDUCT.md). By participating in this project you agree to abide by its terms.

Tests
-----

``` r
covr::package_coverage()
#> llr Coverage: 84.23%
#> R/knitr.R: 0.00%
#> R/llr.R: 18.52%
#> R/parser.R: 99.38%
#> R/ast.R: 100.00%
#> R/compiler.R: 100.00%
#> R/tokenizer.R: 100.00%
```
