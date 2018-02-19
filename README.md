OpenCancer package: cancer incidence
================
Lino Galiana
01 décembre 2017

Installing `OpenCancer`
=======================

`OpenCancer` package installation might fail when `CARET` is not already installed in the computer. As it is stated in [CARET documentation](https://cran.r-project.org/web/packages/caret/vignettes/caret.pdf), to install it, you should run

``` r
install.packages("caret", dependencies = c("Depends", "Suggests"))
```

This installation process might be quite long. Once `CARET` have been installed, you can run

``` r
install.packages("devtools")
devtools::install_github("EpidemiumOpenCancer/OpenCancer")
```

By default, `Vignettes` are not built. If you want to build `Vignettes`, use `devtools::install_github("EpidemiumOpenCancer/OpenCancer", build_vignettes = TRUE)`. However, `Vignettes` building might be time-consuming. You can find them [here](http://htmlpreview.github.io/?https://github.com/EpidemiumOpenCancer/OpenCancer/blob/master/vignettes/import_data.html) and [here](http://htmlpreview.github.io/?https://github.com/EpidemiumOpenCancer/OpenCancer/blob/master/vignettes/estimation_pointers.html)

`OpenCancer` package has been designed to help anyone wanting to work on cancer data to build a dataset. As an example, we use colon cancer data. However, functions are general enough to be applied to any similar data (change `C18` code to the one you want when using `import_training`). One of the main challenges of the `Epidemium` dataset is that it requires high-dimensional statistical techniques. `OpenCancer` package allows to

-   Easily import and merge [Epidemium](http://qa.epidemium.cc/data/epidemiology_dataset/) datasets
-   Build a clean training table
-   Perform feature selection
-   Apply statistical models on selected features

Vignettes have been written to help any users working with `OpenCancer` and can be accessed using `browseVignettes("OpenCancer")`

-   [Import Epidemium data and build training table](/vignettes/import_data.Rmd). HTML version can be found [here](http://htmlpreview.github.io/?https://github.com/EpidemiumOpenCancer/OpenCancer/blob/master/vignettes/import_data.html)
-   [Use pointers to build statistical models](/vignettes/estimation_pointers.Rmd). HTML version can be found [here](http://htmlpreview.github.io/?https://github.com/EpidemiumOpenCancer/OpenCancer/blob/master/vignettes/estimation_pointers.html)

If you want to see this `README` with the code output in HTML, go [there](http://htmlpreview.github.io/?https://github.com/EpidemiumOpenCancer/OpenCancer/blob/master/README.html)

It might be hard to work with Epidemium data because they require lots of RAM when working with R. It is a challenge to take advantage of the statistical power of R packages without being limited by the R memory handling system. Many functions of the `OpenCancer`, relying on the `bigmemory` package, implement memory efficient techniques based on C++ pointer.

`OpenCancer` package has been designed such that it is possible to work with pointers (`big.*` functions) or apply equivalent functions when working with standard dataframes (same functions names without `big.*` prefix). In this tutorial, we will use pointers since it is less standard and might require some explanations. More examples are available [here](http://htmlpreview.github.io/?https://github.com/EpidemiumOpenCancer/OpenCancer/blob/master/vignettes/estimation_pointers.html)

Importing Epidemium data using pointers
=======================================

You can find [here](/vignettes/import_data.Rmd) a `Vignette` describing how to create a clean training table. After installing `OpenCancer` package, we import training data, stored as `csv`, using pointers

``` r
library(OpenCancer)
datadir <- paste0(getwd(),"/vignettes/inst")

X <- bigmemory::read.big.matrix(paste0(datadir,"/exampledf.csv"), header = TRUE)
```

This `markdown` presents a standard methodology :

-   Feature selection using LASSO
-   Linear regression on selected features

`Epidemium` data are multi-level panel data. An individual unit is defined by a series of variables that are overlapping levels (*country, region, sex, age levels and sometimes ethnicity*). `OpenCancer` functions allow to apply this methodology on groups that are defined independently by a series of variables. Some function executions can be parallelized.

The main interest of using pointers rather than dataframes is that data are never imported in memory, avoiding to sature computer's RAM.

Feature selection
=================

Feature selection is performed using LASSO. Given a penalization parameter *λ* and a set of *p* explanatory variables, we want to solve the following program
$$\\widehat{\\beta}\\in\\arg\\min\_{\\beta\\ \\in\\mathbb{R}^p}\\ \\frac{1}{2}\\ \\left|\\right|y-X\\beta\\left|\\right|\_2^2 + \\lambda ||\\beta||\_1 $$
 using standard matrix notations. The *λ* parameter is of particular importance. Its value determines the sparsity of the model: the higher *λ* is, the stronger the ℓ<sub>1</sub> constraint is and the more *β* coefficients will be zero.

The optimal set of parameters can be selected using cross validation (though not recommended, `OpenCancer` package also allows not to perform cross validation).

`big.simplelasso` has been designed to perform LASSO using `biglasso` package (for the non-pointers version, `simplelasso` the `glmnetUtils` is used). Assume we want to use a pooled model, i.e. we do not define independent groups. In that case, the following command can be used

``` r
lassomodel <- big.simplelasso(X,yvar = 'incidence', labelvar = c("cancer", "age", "sex",
  "Country_Transco", "year"),
  crossvalidation = T,
  nfolds = 10, returnplot = F)
```

where we excluded a few variables - that are labelling variables, not explanatory - from the set of covariates. The `returnplot` option, when it is set to `TRUE` will produce the following plot

``` r
plot(lassomodel$model)
```

![](README_files/figure-markdown_github/unnamed-chunk-4-1.png)

The LASSO performance is the following

``` r
summary(lassomodel$model)
```

    ## lasso-penalized linear regression with n=5928, p=8185
    ## At minimum cross-validation error (lambda=1.6596):
    ## -------------------------------------------------
    ##   Nonzero coefficients: 12
    ##   Cross-validation error (deviance): 7765.92
    ##   R-squared: 0.01
    ##   Signal-to-noise ratio: 0.01
    ##   Scale estimate (sigma): 88.124

From an initial number of parameters of 8186, LASSO selects 12 variables

If parallelization is wanted, assuming one core is let aside of computations,

``` r
big.simplelasso(X,yvar = 'incidence', labelvar = c("cancer", "age", "sex",
  "Country_Transco", "year", "area.x", "area.y"), crossvalidation = T,
  nfolds = 10, returnplot = F,
  ncores = parallel::detectCores() - 1)
```

Linear regression after feature selection
=========================================

`big.simplelasso` is useful to select features. To go further, we can perform linear regression on selected variables. The `big.model.FElasso` allows to launch a `big.simplelasso` routine, extract non-zero coefficients and use, afterwards, linear regression.

``` r
pooledOLS <- big.model.FElasso(X,yvar = "incidence",
                           labelvar = c("cancer", "age",
  "Country_Transco", "year", "area.x", "area.y"),
  returnplot = F, groupingvar = NULL)

summary(pooledOLS)
```

    ## Large data regression model: biglm(formula = formula, data = data, ...)
    ## Sample size =  5928 
    ##                    Coef       (95%       CI)        SE      p
    ## (Intercept)     43.1951 -4217.8538 4304.2439 2130.5244 0.9838
    ## sex              4.7179     0.1425    9.2934    2.2877 0.0392
    ## `6741..72040` -103.7412 -8142.3828 7934.9003 4019.3208 0.9794
    ## `6690..5110`    -0.0090    -0.7658    0.7477    0.3784 0.9810
    ## `1057..5112`     0.0001    -0.0111    0.0114    0.0056 0.9811
    ## `372..5510`      0.0000    -0.0007    0.0007    0.0003 0.9698
    ## `2600..5510`     0.0000    -0.0002    0.0002    0.0001 0.9730
    ## `1062..5510`     0.0001    -0.0058    0.0060    0.0029 0.9797
    ## `1780..5510`     0.0001    -0.0083    0.0085    0.0042 0.9779
    ## `51..5510`       0.0000    -0.0010    0.0010    0.0005 0.9803
    ## `882..5510`     -0.0001    -0.0083    0.0080    0.0041 0.9779
    ## `1765..5510`     0.0000    -0.0003    0.0003    0.0001 0.9805
    ## `1770..5510`         NA         NA        NA        NA     NA
    ## `2738..5300`         NA         NA        NA        NA     NA

The `DTsummary.biglm` can be used to produce an HTML summary table.

``` r
DTsummary.biglm(pooledOLS)
DTsummary.biglm(pooledOLS)
```

Apply same methodology with independent groups
==============================================

`big.model.FElasso` allows to apply the same methodology with dataframes splitted by groups. For instance, defining independent groups by sex and age class

``` r
panelOLS <- big.model.FElasso(X,yvar = "incidence",
                              groupingvar = c('sex','age'),
                              labelvar = c('year','Country_Transco')
)

DTsummary.biglm(panelOLS[[2]])
```

Importing data after feature selection
======================================

Once features have been selected, using pointers is no longer so relevant since the dataset with a few columns not being so large. It is thus possible, once features have been selected, to import data with selected features back in the memory. `recover_data` has been designed for that. Taking a pointer as input, it performs LASSO to select feature and imports relevant variables back in the memory as a `tibble`.

``` r
df <- unique(recover_data(X))
```

``` r
DT::datatable(df[sample.int(nrow(df),10),1:7])
```

It is henceforth possible to use standard statistical and visualization tools. For instance, assume we want to perform random forest using `CARET`

``` r
train.index <- sample.int(n = nrow(df), size = 0.8*nrow(df))
trainData <- df[train.index,-which(colnames(df) == "year")]
testData  <- df[-train.index,-which(colnames(df) == "year")]

rfctrl <- trainControl(method = "cv", number = 5)
randomforest <- caret::train(incidence ~ ., data = trainData,
                             trControl = rfctrl, method = "rf")

knitr::kable(data.frame(yhat = predict(randomforest,testData),
                        y = testData$incidence)[1:10,])
```
