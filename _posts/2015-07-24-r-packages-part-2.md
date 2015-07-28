---
layout: post
title: "R packages part 2: on a roll"
description: "Versioning, GitHub release, data, other files, testing, and vignettes."
tags: [R, R package]
modified: 2015-07-28
published: false
---

In part 2: versioning, GitHub release, data, other files, testing, and vignettes.

See [part 1]({{ site.url }}/r-packages-part-1) for R package set up, directory structure, the DESCRIPTION file, writing R code for packages, using roxygen2 to write documentation and define the package namespace, and a simple build protocol. 


Sources:

* [R packages](http://r-pkgs.had.co.nz/) by Hadley Wickham
* [Developer guidelines](http://www.bioconductor.org/developers/) from Bioconductor
* [Writing R extensions](https://cran.r-project.org/doc/manuals/r-release/R-exts.html) by R Core Team
* [Developing R packages](https://github.com/jtleek/rpackages) by Jeff Leek


## Versioning

Use `x.y.z` versioning scheme, starting with 0.1.0 as recommended by [Jeff Leek](https://github.com/jtleek/rpackages#versioning-your-package). 
Before release on CRAN or Bioconductor, keep `x` at 0 and increase `y` with every major redesign. Each time a change is made public (pushed to GitHub), increase `z` by one. 


When submitting a package to Bioconductor, submit as version 0.99.0 so it gets bumped up to 1.0.0 on the next Bioconductor release. 
[Bioconductor](http://www.bioconductor.org/developers/version-numbering/) uses even `y` for packages in release and odd `y` for packages in development. Every time the Bioconductor release version increases `y` to the next even, bump up the GitHub (devel) version to the next odd. 
Continue to increase `z` with every public change (bumps back to zero with every `y` increase). 
To signify a major redesign with increased `x`, set `y` to 99 in the development version, and then the next Bioconductor release will be `(x+1).0.0`. 


## GitHub for R packages

For an in-depth explanation of version control and code release using git and GitHub for R package development, see [Hadley Wickham's chapter](http://r-pkgs.had.co.nz/git.html). In brief:

* use git version control for local package development from the start (can turn on within RStudio)
* create GitHub repo with same name as the package
* set remote origin of the local git repo to `git@github.com:username/packagename.git`
* write a `README.md` file to describe the package to GitHub users - include installation instructions e.g. `devtools::install_github('username/packagename')`
* push public versions to GitHub, bumping up the version number `z` each time, and ensuring `R CMD CHECK` passes

The URL and BugReports fields in the DESCRIPTION file can point to the package's GitHub site and issues page. Use [git tags](http://git-scm.com/book/en/v2/Git-Basics-Tagging) to mark important versions.  


## Data

Data can be included in an R package as a means of: data release and sharing; checking package behavior with tests at build time; and demonstrating package functions through examples or vignettes. 

If the package is primarily a vehicle for data release and sharing, then the included functions should be minimal. If the package is primarily designed as analysis software, then the included data should be small. Large examples and vignettes for software packages can use large datasets from separate data packages. Note that data packages in Bioconductor are not limited by the same size restrictions that apply to software packages ([see here](http://bioconductor.org/developers/how-to/buildingPackagesForBioc/#external-data-dirs)). 

R data objects to be exported to the user (for examples and vignettes) go in `data/`. The code to generate these objects goes in a `data-raw/` directory. Run `devtools::use_data_raw()` to both create the `data-raw/` directory, and add it to `.Rbuildignore`. In the data generation scripts, use `devtools::use_data(object)` to create a `.rda` file with the same name as the object in `data/`.   Set `LazyData: true` in the DESCRIPTION file. Lazy-loading allows these data objects to be accessed directly in examples or vignettes (don't require explicit loading), and only take up memory in an R session when called upon.

Objects in `data/` are exported to the user (no export tag required), and should, therefore, be documented. Write a roxygen comment block for each data object in the `R/<pkgname>.R` file, with the data object name as an uncommented string beneath. Example from [ggplot2 package](https://github.com/hadley/ggplot2):

{% highlight r %}
#' Prices of 50,000 round cut diamonds.
#'
#' A dataset containing the prices and other attributes of almost 54,000
#' diamonds.
#'
#' @format A data frame with 53940 rows and 10 variables:
#' \describe{
#'   \item{price}{price, in US dollars}
#'   \item{carat}{weight of the diamond, in carats}
#'   ...
#' }
#' @source \url{http://www.diamondse.info/}
"diamonds"
{% endhighlight %}

R data objects to be hidden from the user (for internal use within functions) go in `R/sysdata.rda`. As above, put the code to generate these objects in `data-raw/`, and use `devtools::use_data(object1, object2, internal=TRUE)` to save them to `R/sysdata.rda`. These objects will be available internally via lazy-loading (no need to explicitly load). 

Raw data (not parsed into an R data object) is stored in `inst/extdata/`, and can be used to give examples of loading and parsing data from scratch. Reach these files using `system.file('extdata', 'filename.csv', package='<pkgname>')`.  

Note that some data can also be directly encoded in the R source files.  


## Installed files

The `inst/` directory can house any miscellaneous files, and these are copied into the top-level directory when the package is installed. Some common files include:

* inst/AUTHOR to describe non-standard authorship,
* inst/COPYRIGHT to describe non-standard copyright,
* inst/LICENSE to describe non-standard licensing,
* inst/CITATION to give [citation instructions](http://r-pkgs.had.co.nz/inst.html#inst-citation), 
* inst/extdata as described above.

A message at pacakge start up (see [part 1]({{ site.url }}/r-packages-part-1)) is a good way to direct users to this information. For example, users could be instructed to run `citation('<pkgname>')` to see citation instructions, and `file.show(system.file("LICENSE", package='<pkgname>'))` to see the LICENSE file. 


## Vignettes

Vignettes are tutorials stepping through one or more use cases of the package, using small real datasets. 

Run `devtools::use_vignette('vig_title')` to both create the template file `vignettes/vig_title.Rmd`, and add `knitr` to the Suggests and VignetteBuilder fields in DESCRIPTION. In addition, manually add `rmarkdown` and `BiocStyle` to the Suggests field in DESCRIPTION. 

Example metadata header:

{% highlight yaml %}
---
title: "Vignette Title"
author: "Vignette Author"
date: "`r Sys.Date()`"
vignette: >
  %\VignetteIndexEntry{Vignette Title}
  %\VignetteEngine{knitr::rmarkdown}
  %\VignetteEncoding{UTF-8}
output: BiocStyle::html_document:
  toc: true
  fig_caption: yes
---
{% endhighlight %}

Include this code chunk at the beginning: 

{% highlight r %}
```{r style, echo = FALSE, results = 'asis'}
BiocStyle::markdown()
```
{% endhighlight %}

Write the text of the vignette in [R-flavoured markdown](http://r-pkgs.had.co.nz/vignettes.html#markdown), using [BiocStyle macros](http://www.bioconductor.org/packages/devel/bioc/vignettes/BiocStyle/inst/doc/HtmlStyle.html#style-macros) to refer to other R packages. 

Weave code and results through the text using [knitr](http://r-pkgs.had.co.nz/vignettes.html#knitr) syntax. The data used for the vignette examples can be stored in `inst/extdata` (demonstrate how to load raw data) or in `data/` (demonstrate how to work with loaded data). Remember to use Shift+Alt+K to see RStudio shortcuts - includes shortcuts for running code chunks. 

At the end of the vignette, include a Session Information section showing the output from `devtools::session_info()`. 
