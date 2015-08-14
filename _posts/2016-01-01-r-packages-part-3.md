---
layout: post
title: "R packages part 3: full tilt"
description: "topics"
tags: [R, R package]
modified: 2015-08-14
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

If defining new S4 classes and methods, add the `methods` package to the Imports field in DESCRIPTION, and include the following in `<pkgname>.R`.  
{% highlight r %}
#' @import methods 
NULL
{% endhighlight %}


Define a new class as shown below. The class name should use UpperCamelCase. The slots can be of type `ANY` (no type restriction), a base type, S4 class, or S3 class registered with `setOldClass()`. Other useful arguments to `setClass` are: `contains` for class inheritance; `validity` for object testing; and `prototype` to define default slot values. To enable class inheritance from another package, use `@importClassesFrom pkg ClassA`. If you want others to extend your class, `@export` it; if you want others to create instances of the class but not extend it, just `@export` the constructor function. 

{% highlight r %}
#' An example S4 class
#'
#' @slot slota An example character value
#' @slot slotb An example numeric value
#' @slot slotc An example list value
setClass("ClassName",
	slots = list(
		slota = "character", 
		slotb = "numeric", 
		slotc = "list"))


# Constructor function for users to create an instance
#' Create an instance of ClassName
#' 
#' Description of ClassName
#' 
#' @param a Some character value
#' @param b Some numeric value
#' @param c Some list value
#' 
#' @export
#' 
#' @examples
#' classname(a='test', b=1, c=list(foo=3, bar=4))

classname <- function(a, b, c){
	ans <- new('ClassName', slota=a, slotb=b, slotc=c)
	return(ans)
}
{% endhighlight %}

A generic function dispatches a method implementation specific to the class of the argument. To define a method for an S4 class, the generic function must first be defined. If it has already been defined in another package (e.g. `BiocGenerics`), then use that pre-existing definition by including the roxygen comment `#' @importMethodsFrom pkg GenericA` above your method definition. If you want a newly defined generic to be publically usable, `@export` it. 

{% highlight r %}
# only set the generic like this if it does not already exist
#' do method
#' @param x input of some class, e.g. ClassName
#' @seealso class?ClassName
setGeneric("methodname", function(x, ...) standardGeneric("methodname"))
{% endhighlight %}

Must `@export` every method. The argument name/s to `function` in `setMethod` must match the argument name/s in the generic. For a pre-defined generic, just type the name to see what the argument names must be. Method documentation could go:

* in the class - if you've written the class (use `@describeIn`)
* in the generic - if you've written the generic (use `@describeIn`)
* in its own file - if you've not written the class or generic

{% highlight r %}
#' @describeIn ClassName Method for this class
#' @param x Object of class ClassName
#' @export 
setMethod("methodname", 
	signature = "ClassName", 
	definition = function(x) {
		print(x)
	})
{% endhighlight %}


Slots of an S4 object can be accessed (get and set) via `@` or `slot()`. However, this requires specific knowledge of the implementation (slot names). A better approach for the user is to define accessor methods for getting/setting data. 

{% highlight r %}
setGeneric("slota", function(x, ...) standardGeneric("slota"))

setMethod("slota", "ClassName", function(x) x@slota)

setGeneric("slota<-", function(x, value) standardGeneric("slota<-"))

setReplaceMethod("slota", 
	"ClassName",
	function(x, value) {
		x@slota <- value
		return(x)
	})

{% endhighlight %}


### Load order

When working with S4, the code must be loaded in the order: classes, generic, methods+other. The default is for R package code to load alphabetically by file name. To ensure classes and generics are loaded first, they could be placed in files `aaa-classes.R` and `aaa-generics.R` respectively. Alternatively, use the `@include` roxygen tag at the top of a file to list all the other source code files that should be loaded beforehand. This information gets collected up and used to set the `Collates` field in DESCRIPTION (specifies a non-default load order). 

