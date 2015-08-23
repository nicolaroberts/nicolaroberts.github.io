---
layout: post
title: "R packages part 3: full tilt"
description: "S4 classes, compiled code, and automated checking with Travis CI"
tags: [R, R package]
modified: 2015-08-23
published: false
---

In part 3: S4 classes and methods, compiled code, and automated checking with Travis CI.  

See [part 1]({{ site.url }}/r-packages-part-1) for R package set up, directory structure, the DESCRIPTION file, writing R code for packages, using roxygen2 to write documentation and define the package namespace, and a simple build protocol. 

See [part 2]({{ site.url }}/r-packages-part-2) for versioning, GitHub release, data, other files, testing, and vignettes.



Sources:

* [R packages](http://r-pkgs.had.co.nz/) by Hadley Wickham
* [Advanced R](http://adv-r.had.co.nz/) by Hadley Wickham
* [Developer guidelines](http://www.bioconductor.org/developers/) from Bioconductor
* [Writing R extensions](https://cran.r-project.org/doc/manuals/r-release/R-exts.html) by R Core Team
* [R object oriented programming](https://github.com/lgatto/roo) by Robert Stojnić and Laurent Gatto



## S4 classes and methods

If defining new S4 classes and methods, add the `methods` package to the Imports field in DESCRIPTION, and include the following in `<pkgname>.R`:  
{% highlight r %}
#' @import methods 
NULL
{% endhighlight %}


Define a new class as shown below. The class name should use UpperCamelCase. The slots can be of type `ANY` (no type restriction), a base type, S4 class, or S3 class registered with `setOldClass()`. To allow multiple classes in a slot, use `setClassUnion()`. The validity argument is a function of the object returning TRUE or FALSE. Other useful arguments to `setClass` are `contains` for class inheritance, and `prototype` for default slot values. For class inheritance from another package, use `@importClassesFrom pkg ClassA` (and add package to Imports field in DESCRIPTION). If you want others to extend your class, `@export` it; if you want others to create instances of the class but not extend it, just `@export` the constructor function. 

{% highlight r %}
#' An example S4 class for members of The Beatles
#'
#' @slot name First name
#' @slot ranking Your ranking of favourites, from 1-4
setClass("BeatlesMember",
         slots = list(
             name = "character",
             ranking = "numeric"),
         validity = function(object) {
             is_valid <- TRUE
             if (! object@name %in% c("John", "Paul", "George", "Ringo")) {
                 is_valid <- FALSE
                 message("Name is not one of John, Paul, George, or Ringo")
             }
             if (! object@ranking %in% 1:4) {
                 is_valid <- FALSE
                 message("Ranking must be 1, 2, 3 or 4")
             }
             return(is_valid)
         })


# Constructor function
#' Create an instance of BeatlesMember
#'
#' @param name First name of Beatle's member
#' @param ranking Your ranking of favourites, from 1-4
#'
#' @export
#'
#' @examples
#' beatles_member('John', 1)
beatles_member <- function(name, ranking){
    ans <- new("BeatlesMember", name=name, ranking=ranking)
    return(ans)
}
{% endhighlight %}


Functions of S4 classes may be written as "regular" functions, or as S4 methods dispatched via a generic function. As a general rule, write S4 generics and methods if the function is a common task that could have multiple class-specific implementations, e.g. plot, append, sort, unique, as.data.frame etc. Setters and getters are also convenient as S4 methods. In contrast, if the function is highly specific to your package, just implement it as a regular function (checking the input has the correct class).  

A generic function dispatches a method implementation specific to the class of the argument. The generic function must be defined *before* the method. If it has already been defined in another package (e.g. `BiocGenerics`), then use that pre-existing definition by including the roxygen comment `#' @importMethodsFrom pkg GenericA` above your method definition (and add package to Imports field in DESCRIPTION). To define a new generic, use the `setGeneric` function, and possibly `@export` it to users. 

To define a method, use the `setMethod` function as shown below, with the argument name/s to the function exactly matching the argument name/s in the generic (even if it was defined by someone else). `@export` every method, and possibly use `@describeIn` to merge documentation with the class or the generic. 

{% highlight r %}
#' @importMethodsFrom BiocGenerics as.data.frame
#' @describeIn BeatlesMember convert to data.frame
#' @param x Object of class BeatlesMember
#' @export
setMethod("as.data.frame",
          signature = "BeatlesMember",
          definition = function(x, ...) {
              ans <- data.frame(name=x@name, ranking=x@ranking)
              return(ans)
          })
{% endhighlight %}


Slots of an S4 object can be accessed (get and set) via `@` or `slot()`. However, this requires specific knowledge of the implementation (slot names). Defined accessor methods are a better approach for general users. Remember to check `validObject(x)==TRUE` before returning an object with a new slot value. 

{% highlight r %}
# only set the generic like this if it does not already exist
#' Get 'name' slot from S4 class
#' @param x Object of S4 class with slot 'name'
setGeneric("name", function(x, ...) standardGeneric("name"))

#' @describeIn BeatlesMember get name value
#' @export
setMethod("name", "BeatlesMember", function(x) x@name)

#' Set 'name' slot from S4 class
#' @param x Object of S4 class with slot 'name'
setGeneric("name<-", function(x, value) standardGeneric("name<-"))

#' @describeIn BeatlesMember set name value
#' @param value replacement value
#' @export
setReplaceMethod("name",
                 "BeatlesMember",
                 function(x, value) {
                     x@name <- value
                     if (validObject(x)) return(x)
                 })
{% endhighlight %}

A special method called `show` controls how the object is printed to console. Note that the `show` generic is provided by the `methods` package. 

{% highlight r %}
setMethod("show",
          "BeatlesMember",
          function(object) {
              cat("Object of class", class(object), "\n")
              cat(" name:", object@name, "\n")
              cat(" ranking:", object@ranking, "\n")
          })
{% endhighlight %} 

When working with S4, the code must be loaded in the order: classes, generics, methods+other. The default is for R package code to load alphabetically by file name. To ensure classes and generics are loaded first, they could be placed in files `aaa-classes.R` and `aaa-generics.R` respectively. Alternatively, use the `@include` roxygen tag at the top of a file to list all the other source code files that should be loaded beforehand. This information is used to set the `Collates` field in DESCRIPTION (specifies a non-default load order). 


## Compiled code

Any C/C++ code (including header files) belong in the `src/` directory. 

#### Instructions for C++ or C

Set up a `src/.gitignore` file to ignore `*.o`, `*.so` and `*.dll` files (this will be auto-generated if you run `devtools::use_rcpp()`). 

To access compiled C/C++ functions from R through the `.Call()` function, include the roxygen tag `@useDynLib pkgname` (for all compiled routines, place in `pkgname.R`) or `@useDynLib pkgname routine` (for a specific routine, place alongside wrapper R function).

Write `.onUnload()` function (place in `pkgname.R`) to clean up when the package is unloaded. 

{% highlight r %}
.onUnload <- function(libpath){
  library.dynam.unload("pkgname", libpath)
}
{% endhighlight %}



#### C with the R API

Using C in R packages is only recommended for legacy code. 

C files must include:

{% highlight c %}
#include <R.h>
#include <Rinternals.h>
{% endhighlight %}

To interface with R, C functions must both input and output SEXP (S expression) types. The actual work of the function is easiest to perform with native C data structures, so the first and last steps are usually conversion from/to SEXP to/from C types. Remember to `PROTECT()` (and later `UNPROTECT()`) any SEXP object created in C to save it from R's garbage collector. 

Compiled functions should be called via a wrapper R function, and the classes of the inputs can be checked within this wrapper. Roxygen documentation should be written alongside this wrapper. 

{% highlight r %}
illustrate <- function(x, y) {
  .Call('illustrate', PACKAGE='pkgname', x, y)
}
{% endhighlight %}

More info on using C with R is available [here](http://adv-r.had.co.nz/C-interface.html) and [here](http://r-pkgs.had.co.nz/src.html#clang).


#### C++ with Rcpp

Run `devtools::use_rcpp()` to add Rcpp to the LinkingTo and Imports fields in DESCRIPTION, set up the `.gitignore` file as described above. 

Include roxygen tag `@importFrom Rcpp sourceCpp` in `pkgname.R` (don't actually need `sourceCpp`, but a bug in R means something has to be imported so the internal Rcpp code gets properly loaded). 

C++ files must include:

{% highlight r %}
#include <Rcpp.h>
using namespace Rcpp;
{% endhighlight %}

Rcpp will do the hard work of setting up functions of SEXP objects for you, so just write the C++ functions in terms of native C++ types, and preface the function with the special comment `// [[Rcpp::export]]`. Run `devtools::document()` and then build and reload the package. This automatically calls `Rcpp::compileAttributes()` and auto-generates the files `src/RcppExports.cpp` and `R/RcppExports.R`. 

The auto-generated functions in `src/RcppExports.cpp` act as the go-between for the SEXP types passed to/from R and the C++ types passed to/from your other C++ functions. 

The auto-generated `R/RcppExports.R` file contains wrapper functions for calling the compiled C++ functions. This file shouldn't be edited directly, so any documentation should be written alongside the source C++ code. Write roxygen comment blocks in C++ as in R, just using `//' ` at the start of each line instead of `#' `. Note that the roxygen line `//' @export` makes the R wrapper function available to the user, while the non-roxygen line `// [[Rcpp::export]]` just makes the C++ function available to the R wrapper function (via the SEXP translator function made in `src/RcppExports.cpp`).

More info on using C++ with R is available [here](http://adv-r.had.co.nz/Rcpp.html) and [here](http://r-pkgs.had.co.nz/src.html#cpp).


## Automated checking with Travis CI

During development, run `devtools::check()` and strive to eliminate all errors, warnings, and notes from [these checks](http://r-pkgs.had.co.nz/check.html#check-checks). 

To automatically run `R CMD check` after every push to GitHub: 

* Run `devtools::use_travis()` to generate the `.travis.yml` config file and update `.Rbuildignore` accordingly.  The basic config options are automatically included, with more advanced options described in the [docs](http://docs.travis-ci.com/user/languages/r/). Push to GitHub. 
* Log in to Travis (linked to your GitHub account), and turn on the repo you want to test (will automatically run after every push to GitHub). 
* Embed Travis status image (failing/passing) in the README.md:


{% highlight r %}
[![Build Status](https://travis-ci.org/<USR>/<REPO>.svg?branch=master)](https://travis-ci.org/<USR>/<REPO>)
{% endhighlight %}


For BioConductor, the package must also pass `BiocCheck::BiocCheck("/path/to/pkg")`. To automate with Travis, add the following code to `.travis.yml` (taken from [Przemol](https://github.com/Przemol/seqplots/blob/master/.travis.yml)).

{% highlight r %}
bioc_required: true
bioc_packages:
  - BiocCheck

after_script:
  - ls -lah
  - FILE=$(ls -1t *.tar.gz | head -n 1)
  - Rscript -e "library(BiocCheck); BiocCheck(\"${FILE}\")"
{% endhighlight %}


## Submission to CRAN or Bioconductor

Info on [submitting package to CRAN](http://r-pkgs.had.co.nz/release.html).

Info on submitting package to BioConductor is [here](http://bioconductor.org/developers/package-submission/), [here](http://bioconductor.org/developers/package-guidelines/#responsibilities) and [here](http://bioconductor.org/developers/how-to/git-mirrors/).  

