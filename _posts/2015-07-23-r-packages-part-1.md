---
layout: post
title: "R packages part 1: up and running"
description: "R package set up, structure, DESCRIPTION file, R source code, documentation, NAMESPACE, etc."
tags: [R, R package]
modified: 2015-07-24
---

In part 1: R package set up, directory structure, the DESCRIPTION file, writing R code for packages, using roxygen2 to write documentation and define the package namespace, and a simple build protocol. 


Sources:

* [R packages](http://r-pkgs.had.co.nz/) by Hadley Wickham
* [Developer guidelines](http://www.bioconductor.org/developers/) from Bioconductor
* [Writing R extensions](https://cran.r-project.org/doc/manuals/r-release/R-exts.html) by R Core Team
* [Developing R packages](https://github.com/jtleek/rpackages) by Jeff Leek


## Choosing a package name

The package name should be all lower case with no special characters. 
To ensure the name is not already taken on CRAN or Bioconductor, check that `BiocInstaller::biocLite('<pkgname>')` fails. 


## Setting up in RStudio

Install or update devtools, roxygen2 and knitr. 

In RStudio: File > New Project > New Directory > R package (include git repo) > Create Project. 
This auto-generates the basic skeleton of necessary files and directories.
Delete the `Read-and-delete-me` file and the automatic package documentation file in `man/` (bizarre formatting fails `R CMD CHECK`).

In keeping with [Bioconductor's style preference](http://www.bioconductor.org/developers/how-to/buildingPackagesForBioc/#creating-packages-rstudio), go to code editing preferences and set tab width to 4 spaces and show a margin at column 80 (max line length). 


## R package source structure

#### Mandatory

* DESCRIPTION file of package metadata
* NAMESPACE file defining which objects to export and import
* R/ directory containing R code in `.R` files
* man/ directory of manual help pages in `.Rd` files

#### Optional

* src/ directory with any C or C++ code
* vignettes/ directory of tutorial vignettes in `.Rnw` or `.Rmd` files
* data/ directory containing saved R objects in `.rda` files (used for examples or unit tests)
* inst/ directory for other files including citation instructions, non-standard copyright definitions, non-standard data formats, etc. 


## DESCRIPTION file

#### Mandatory fields

{% highlight debcontrol %}
Package: pkgname
Title: Short one liner, no period at end
Description: A paragraph description, including information about the
   data types the package was designed to run on, and what the outputs are. 
Version: 0.1.0
Authors@R: person("first", "last", email="mail@place", role=c('aut', 'cre'))
License: GPL-3
{% endhighlight %}

For more on versioning packages, see [part 2]({{ site.url }}/r-packages-part-2). 

For more on using the `Authors@R` field to record the package author, maintainer, and other contributors or copyright holders, see `?person`. 

For more on choosing a license, see [Hadley Wickham's advice](http://r-pkgs.had.co.nz/description.html#license). To specify a non-standard license, write a file `LICENSE` in the top-level directory, and use `License: file LICENSE`. *For Sanger work, see CGP wiki on licensing code for external release.*

#### Optional fields

{% highlight debcontrol %}
Type: Package
Date: 2015-01-30
Depends: R(>= 3.1.2) 
Imports: MASS(>=7.3.0), dplyr
Suggests: knitr, BiocStyle 
biocViews: Software, StatisticalMethod, Clustering 
URL: package_url
BugReports: bug_url
LazyData: true
VignetteBuilder: knitr
LinkingTo: BH 
Collate: classes.R generics.R methods.R functions.R
{% endhighlight %}

Packages listed in Depends will be attached in their entirety, so only list here if your package is primarily designed for use in conjunction with another. Packages listed in Imports will load but not attach, and this is generally preferable to Depends. Packages that are only used in vignettes or examples should be listed in Suggests. 

For submission to Bioconductor, list a controlled set of keywords in biocViews (possible keywords can be found [here](http://bioconductor.org/packages/devel/BiocViews.html)). 

When including data with a package, use LazyData to turn on [lazy-loading](https://cran.r-project.org/doc/manuals/r-release/R-ints.html#Lazy-loading).

To use compiled code from another package, list it in LinkingTo. 

By default, R code files in the `R/` directory are sourced in alphanumeric order. To specify a different loading order, list the `.R` files in the Collate field (useful when defining your own S4 classes). 

Other possible fields are described in the [official docs](https://cran.r-project.org/doc/manuals/r-release/R-exts.html#The-DESCRIPTION-file). 


## R code

All R code goes in `.R` files in the `R/` directory. Group functions meaningfully to avoid either extreme of too many files in the directory, or too much code per file. 

Using Hadley's [recommended style](http://r-pkgs.had.co.nz/r.html#style), variables and functions should have lower case names with underscores separating words. Try to use nouns for variables and verbs for functions. To avoid potential conflict with other packages, try to define unique function names. Use comments to explain the why not the what, and break files up into readable chunks using commented lines of hyphens. Run `lintr::lint_package()` to find any breaches of lintr's default style guide. 

R package code should never change the user's R landscape. That is, the behaviour of any external function should be the same before and after use of your package. For this reason, never use `library()`, `require()`, or `source()` in a package. When changing `options()` or `par()`, save the old values first and reset when done.

{% highlight r %}
par_old <- par(no.readonly=TRUE)
on.exit(par(par_old), add=TRUE)
{% endhighlight %}

Package functions that are exported to the user should start with thorough input checks using `stop()` to throw errors. Following [Bioconductor guidelines](http://bioconductor.org/developers/package-guidelines/#messages), only pass messages back to the user using `stop()`, `warning()` and `message()`; reserve `cat()` and `print()` to display an object in a show method. 

When using functions from external packages, explicitly refer to the other package using `package::function()`. This makes it easier to see when external packages should be added to the Imports field of the DESCRIPTION file. When using an external function from a package that is merely in Suggests, first test whether or not the package is installed (`requireNamespace('pkg', quietly=TRUE)`), and handle either case. 


To display a message upon package start up, use the following template in a special `zzz.R` file:

{% highlight r %}
.onAttach <- function(libname, pkgname){
	packageStartupMessage("Welcome!")
}
{% endhighlight %}


## Documentation and the NAMESPACE file with roxygen2

Manual help pages to document exported functions or classes go in `.Rd` files in the `man/` directory, but don't write these files directly! Rather, use roxygen2 to generate the documentation for you. 

The NAMESPACE file defines the set of objects exported to end users, and the set of objects imported from other packages. Use roxygen2 to generate this file for you.  

In the `.R` files, add roxygen comments (starting `#' `) before each function as indicated below. Run `devtools::document()` to convert into appropriate NAMESPACE and `.Rd` files. 

To document the package itself, write the roxygen comments in a file called `<pkgname>.R`, and type `NULL` (uncommented) below. Include tags `@docType package` and `@name <pkgname>`. Delete any default package man page auto-generated at set up, so the roxygen version can replace it. 

{% highlight r %}
#' One liner description
#' 
#' A more detailed description of what the function is and how
#' it works. It may be a paragraph that should not be separated
#' by any spaces. 
#'
#' @param x A description of the input param \code{x}
#' @param y A description of the input param \code{y}
#'
#' @return output A description of the object the function outputs 
#'
#' @export
#' 
#' @seealso \url{some_url} for an explanation of blah,
#'   \code{\link{func_in_same_pkg}}, and \code{\link[otherpkg]{func_in_other_pkg}}
#' 
#' @aliases alias1 alias2
#'
#' @keywords keyword1 keyword2
#' 
#' @examples
#' R code here showing how your function works. Will run as part of R CMD CHECK. 

illustrate <- function(x, y){
    # example
    return(ans)
}
{% endhighlight %}


See [this guide](http://r-pkgs.had.co.nz/man.html#text-formatting) to text formatting in roxygen tags. 

The first sentence in the roxygen comment block becomes the title. The second paragraph becomes the description, and all subsequent paragraphs make the later Details section. Section headers may be added with the `@section Section name` tag. 

Include `@export` to make the function visible to end users (sends the function to NAMESPACE). Export the minimum number of functions an end user should need. 

Aliases are alternative searches that redirect to the function's help page. Keywords must be taken from [this predefined list](https://svn.r-project.org/R/trunk/doc/KEYWORDS). 

Examples should complete quickly (use toy data), and will run during checks. 

When using an external function from another package using `package::function()` (ensuring package is listed in Imports field of DESCRIPTION), there is no need to define an import through NAMESPACE, and thus no roxygen comments are required. However, using `package::function()` is slightly slower, so, for functions that are heavily used, import and *attach* the external function with the roxygen tag `@importFrom pkg func` and use the function without `::`.  


Other useful roxygen tags are:

* `@family` to link a family of related functions together
* `@inheritParams` to inherit the parameter descriptions from another function
* `@describeIn` or `@rdname` to [document multiple functions](http://r-pkgs.had.co.nz/man.html#dry2) in the same file 
* `@importClassesFrom` and `@importGenericsFrom` to use classes and generics from external packages



## Fast build and install

* Check code style `lintr::lint_package()`
* Update documentation `devtools::documents()`
* Click Check button in build panel (executes `devtools::check()`)
* Click Build & Reload button in build panel (executes `R CMD INSTALL`)

