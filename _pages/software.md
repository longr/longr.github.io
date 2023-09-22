---
layout: archive
title: "Software"
author_profile: true
permalink: /software/
---
### `changepoint`: Methods for Changepoint dectection in R

I created and a changepoint detection library in R called [`changepoint`](https://cran.r-project.org/package=changepoint).  It implements various mainstream and specialised changepoint methods for finding single and multiple changepoints within data. Many popular non-parametric and frequentist methods are included. The `cpt.mean()`, `cpt.var()`, `cpt.meanvar()` functions should be your first point of call.

This package is available of [CRAN](https://cran.r-project.org/package=changepoint)

Suggestions for future improvements or bug fixes are welcomed Email me or post an issue or pull request on [Github](https://github.com/rkillick/changepoint)

Accompanying introductory paper Copy available [here](http://www.jstatsoft.org/v58/i03/)

___

### `changepoint.np`: Methods for Nonparametric Changepoint Detection in R

This package implements the multiple changepoint algorithm PELT with a nonparametric cost function based on the empirical distribution of the data. This package extends the `changepoint` package.

This package is available on [CRAN](https://cran.r-project.org/package=changepoint.np)

Suggestions for future improvements or bug fixes are welcomed Email me or post an issue or pull request on [Github](https://github.com/KayleaHaynes/changepoint.np)

___

### `EnvCpt`: Detection of Structural Changes in Climate and Environment Time Series in R

Tools for automatic model selection and diagnostics for Climate and Environmental data. In particular the `envcpt()` function does automatic model selection between a variety of trend, changepoint and autocorrelation models. The `envcpt()` function should be your first port of call.

This package is	available on [CRAN](https://cran.r-project.org/package=EnvCpt)

Suggestions for future improvements or bug fixes are welcomed Email me or post an issue or pull request on [Github](https://github.com/rkillick/EnvCpt)

___

### `changepoint.geo`: Title: Geometrically Inspired Multivariate Changepoint Detection in R

Implements the high-dimensional changepoint detection method GeomCP and the related mappings used for changepoint detection. These methods view the changepoint problem from a geometrical viewpoint and aim to extract relevant geometrical features in order to detect changepoints. The geomcp() function should be your first point of call.

This package is	available on [CRAN](https://cran.r-project.org/package=changepoint.geo)

Suggestions for future improvements or bug fixes are welcomed Email me or post an issue or pull request on [Github](https://github.com/grundy95/changepoint.geo)

___

### `changepoint.influence`: Package to Calculate the Influence of the Data on a Changepoint Segmentation in R

Allows users to input their data, segmentation and function used for the segmentation (and additional arguments) and the package calculates the influence of the data on the changepoint locations, see Wilms et al. (2021) <arXiv:2107.10572>. Currently this can only be used with the changepoint package functions to identify changes, but we plan to extend this. There are options for different types of graphics to assess the influence.

This package is	available on [CRAN](https://cran.r-project.org/package=changepoint.influence)

Suggestions for future improvements or bug fixes are welcomed Email me or post an issue or pull request on [Github](https://github.com/rkillick/changepoint.influence)
