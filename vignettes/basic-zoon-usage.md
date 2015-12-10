---
title: "Basic zoon usage"
author: "Tim Lucas"
date: "2015-12-07"
output:
  html_document:
    toc: yes
---
<!--
%\VignetteEngine{knitr::knitr}
%\VignetteIndexEntry{Basic zoon usage}
-->




An Introduction to the **zoon** package
=======================================


**Zoon** is a package to aid reproducibility and between-model comparisons in species distribution modelling. Each step in an analysis is a 'module'. These modules will include: 
+ Data collection of **occurrence** and environmental **covariate** data from online databases. 
+ **Process** steps such as removal of spatial autocorrelation in the data or generation of background pseudoabsences.
+ The fitting of **models**.
+ Model **output** including diagnostics, reports and vizualisation.



Getting set up
----------------------------

First install from github

```r
library(devtools)
install_github('zoonproject/zoon')
```

and load


```r
library(zoon)
```

Basic usage
----------------------------

A basic worklow is run using the `workflow` function. We must chose a module for each type: occurrence, covariate, process, model and output.


```r
work1 <- workflow(occurrence = UKAnophelesPlumbeus,
                  covariate  = UKAir,
                  process    = OneHundredBackground,
                  model      = RandomForest,
                  output     = PrintMap)
```

<img src="figure/basic-1.png" title="plot of chunk basic" alt="plot of chunk basic" style="display:block; margin: auto" style="display: block; margin: auto;" />

```r
class(work1)
```

```
## [1] "zoonWorkflow"
```

```r
str(work1, 1)
```

```
## List of 9
##  $ occurrence.output:List of 1
##  $ covariate.output :List of 1
##  $ process.output   :List of 1
##  $ model.output     :List of 1
##  $ report           :List of 1
##  $ call             : chr "workflow(occurrence = UKAnophelesPlumbeus, covariate = UKAir, process = OneHundredBackground, model = RandomForest, output = Pr"| __truncated__
##  $ call.list        :List of 5
##  $ session.info     :List of 7
##   ..- attr(*, "class")= chr "sessionInfo"
##  $ module.versions  :List of 5
##  - attr(*, "class")= chr "zoonWorkflow"
```

In this case we are using the following modules which do the following things:
+ `UKAnophelesPlumbeus`: Uses occurrence points of _Anopheles plumbeus_ in the UK collected from GBIF
+ `UKAir`: Uses NCEP air temperature data for the UK
+ `OneHundredBackground`: Randomly creates 100 pseudoabsence or background datapoints
+ `LogisticRegression`: Run a random forest to model the relationship between _A. plumbeus_ and air temperature
+ `PrintMap`: Predicts the model across the whole of the UK and prints to graphics device. 

For output we get an object of class "zoonWorkflow". This object is basically a big list with all the data, models and output we collected and created in our analysis.

Getting Help
--------------

To find a list of modules available on the online repository use


```r
GetModuleList()
```

To find help on a specific module use


```r
ModuleHelp(LogisticRegression)
```
Note that you can't use `?` as the modules are held on a repository. Therefore the module documentation files are not included with the basic zoon install.



More complex analyses
-----------------------

The syntax for including arguments to modules is simply `ModuleName(parameter = 'value')`. For example, to do two fold crossvalidation we do


```r
work2 <- workflow(occurrence = UKAnophelesPlumbeus,
                  covariate  = UKAir,
                  process    = BackgroundAndCrossvalid(k = 2),
                  model      = LogisticRegression,
                  output     = PerformanceMeasures)
```

```
## Loading required package: SDMTools
## 
## Attaching package: 'SDMTools'
## 
## The following object is masked from 'package:raster':
## 
##     distance
## 
## Model performance measures:
## auc :  0.675630417651694
## kappa :  0.451218048721949
## omissions :  0
## sensitivity :  1
## specificity :  0.37037037037037
## proportionCorrect :  0.810408921933085
## 
```

Here we are providing an argument to the module `BackgroundAndCrossvalid`. We are setting `k` (the number of cross validation folds) to 2.

We are using an output module `PerformanceMeasures` which calculates a number of measures of the effectiveness of our model: AUC, kappa, sensitivity, specificity etc.


### Multiple modules with Chain

We might want to combine multiple modules in our analysis. For this we use the function Chain.


```r
work3 <- workflow(occurrence = UKAnophelesPlumbeus,
                  covariate  = UKAir,
                  process    = Chain(OneHundredBackground, Crossvalidate),
                  model      = LogisticRegression,
                  output     = PerformanceMeasures)
```

```
## Warning in OneHundredBackground(.data = structure(list(df = structure(list(: There are fewer than 100 cells in the environmental raster.
## Using all available cells (81) instead
```

```
## Model performance measures:
## auc :  0.664827948515892
## kappa :  0.438052386308854
## omissions :  0
## sensitivity :  1
## specificity :  0.358024691358025
## proportionCorrect :  0.806691449814126
## 
```
Here we drawing some pseudoabsence background points, and doing crossvalidation (which is the same as `work2`, but explicitely using the separate modules.)

The effect of `Chain` depends on the module type: 
+`occurrence`: All data from chained modules are combined.
+`covariate`: All raster data from chained modules are stacked.
+`process`: The processes are run sequentially, the output of one going into the next.
+`model`: Model modules cannot be chained.
+`output`: Each output module that is chained is run separately on the output from other modules.

`Chain` can be used on as many module type as is required.

### Multiple modules with list

If you want to run separate analyses that can then be compared for example, specifiy a list of modules.


```r
work4 <- workflow(occurrence = UKAnophelesPlumbeus,
                  covariate  = UKAir,
                  process    = OneHundredBackground,
                  model      = list(LogisticRegression, RandomForest),
                  output     = SameTimePlaceMap)

str(work4, 1)
```

```
## List of 9
##  $ occurrence.output:List of 1
##  $ covariate.output :List of 1
##  $ process.output   :List of 1
##  $ model.output     :List of 2
##  $ report           :List of 2
##  $ call             : chr "workflow(occurrence = UKAnophelesPlumbeus, covariate = UKAir, process = OneHundredBackground, model = list(LogisticRegression, "| __truncated__
##  $ call.list        :List of 5
##  $ session.info     :List of 7
##   ..- attr(*, "class")= chr "sessionInfo"
##  $ module.versions  :List of 5
##  - attr(*, "class")= chr "zoonWorkflow"
```
Here, the analysis is split into two and both logistic regression and random forest (a machine learning algorithm) are used to model the data. Looking at the structure of the output we can see that the output from the first three modules are a list of length one. When the analysis splits into two, the output of the modules (in `work4$model.output` and `work4$report`) is then a list of length two. One for each branch of the split analysis.



### A larger example

Here is an example of a larger analysis.


```r
work5 <- workflow(occurrence = Chain(SpOcc(species = 'Eresus kollari', 
                                       extent = c(-10, 10, 45, 65)),
                                     SpOcc(species = 'Eresus sandaliatus', 
                                       extent = c(-10, 10, 45, 65))),
 
                  covariate  = UKAir,

                  process    = BackgroundAndCrossvalid(k = 2),

                  model      = list(LogisticRegression, RandomForest),

                  output     = Chain(SameTimePlaceMap, PerformanceMeasures)
         )
```

```
## Loading required package: spocc
## Installing package into 'C:/Users/tomaug/Documents/R/win-library/3.2'
## (as 'lib' is unspecified)
## also installing the dependencies 'leafletR', 'rbison', 'ecoengine', 'rebird', 'AntWeb', 'rvertnet', 'lubridate'
```

```
## package 'leafletR' successfully unpacked and MD5 sums checked
## package 'rbison' successfully unpacked and MD5 sums checked
## package 'ecoengine' successfully unpacked and MD5 sums checked
## package 'rebird' successfully unpacked and MD5 sums checked
## package 'AntWeb' successfully unpacked and MD5 sums checked
## package 'rvertnet' successfully unpacked and MD5 sums checked
## package 'lubridate' successfully unpacked and MD5 sums checked
## package 'spocc' successfully unpacked and MD5 sums checked
## 
## The downloaded binary packages are in
## 	C:\Users\tomaug\AppData\Local\Temp\RtmpyMMDln\downloaded_packages
```

```r
str(work5, 1)
```

```
## List of 7
##  $ occurrence.output:List of 1
##  $ covariate.output :List of 1
##  $ process.output   :List of 1
##  $ model.output     :List of 2
##  $ report           :List of 2
##  $ call             : chr "workflow(occurrence = Chain(SpOcc(species = \"Eresus kollari\", extent = c(-10, 10, 45,      65)), SpOcc(species = \"Eresus san"| __truncated__
##  $ call.list        :List of 5
##  - attr(*, "class")= chr "zoonWorkflow"
```

```r
par(mfrow=c(1,2))
plot(work5$report[[1]][[1]], 
  main = paste('Logistic Regression: AUC = ', 
             round(work5$report[[1]][[2]]$auc, 2)))
plot(work5$report[[2]][[1]],
  main = paste('Random forest: AUC = ', 
             round(work5$report[[2]][[2]]$auc, 2)))
```

<img src="figure/largeAnalysis-1.png" title="plot of chunk largeAnalysis" alt="plot of chunk largeAnalysis" style="display:block; margin: auto" style="display: block; margin: auto;" />

Here we are collecting occurrence data for two species, _Eresus kollari_ and _E. sandaliatus_ and combining them (having presumably decided that this is ecologically appropriate.) We are using the air temperature data from NCEP again. We are sampling 100 pseudo absence points and running two fold crossvalidation.

We run logistic regression and random forest on the data separately. We then predict the model back over the extent of our environmental data and calculate some measures of how good the models are. Collating the output into one plot we can see the very different forms of the models and can see that the random forest has a higher AUC (implying it predicts the data better.)







