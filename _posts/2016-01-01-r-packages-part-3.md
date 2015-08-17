---
layout: post
title: "R packages part 3: full tilt"
description: "topics"
tags: [R, R package]
modified: 2015-08-17
published: false
---

In part 3: compiled code, TRAVIS checking and BiocChecking, S4 classes, release on CRAN or Bioconductor, package at different stages. 

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


Define a new class as shown below. The class name should use UpperCamelCase. The slots can be of type `ANY` (no type restriction), a base type, S4 class, or S3 class registered with `setOldClass()`. To allow multiple classes in a slot, use `setClassUnion()`. The validity argument is a function of the object returning TRUE or FALSE. Other useful arguments to `setClass` are `contains` for class inheritance, and `prototype` to define default slot values. For class inheritance from another package, use `@importClassesFrom pkg ClassA` (add package to Imports field in DESCRIPTION). If you want others to extend your class, `@export` it; if you want others to create instances of the class but not extend it, just `@export` the constructor function. 

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


Functions of S4 classes may be written as "regular" functions, or as S4 methods dispatched via a generic function. As a general rule, write S4 generics and methods if the function is a common task that could have multiple class-specific implementations, e.g. plot, append, sort, unique. Setters and getters are also convenient as S4 methods. In contrast, if the function is highly specific to your package just implement it as a regular function (checking the input has the correct class).  

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


Slots of an S4 object can be accessed (get and set) via `@` or `slot()`. However, this requires specific knowledge of the implementation (slot names). A better approach for the user is to define accessor methods for getting/setting data. Remember to check `validObject(x)==TRUE` before returning an object with a new slot value. 

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

A special method called `show` controls how the object is printed to console when the object name is typed by itself. Note that the `show` generic is provided by the `methods` package. 

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

