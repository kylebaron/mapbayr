
# mapbayr <img align="right" src = "inst/logo2.png" width="135px">

mapbayr is a free and open source package for *maximum a posteriori*
bayesian estimation in R. Thanks to a single function, `mbrest()`, you
can estimate individual PK parameters from:

  - a data set of concentrations to fit (NM-TRAN format),
  - a population PK model coded in *mrgsolve*,

It was designed to be easily wrapped in [shiny
apps](https://shiny.rstudio.com/) in order to ease model-based
therapeutic drug monitoring, also referred to as Model-Informed
Prediction Dosing (MIPD).

## Installation:

mapbayr is not (yet) available on CRAN. You can install it from github
by executing the following code in R console.

``` r
install.packages("devtools")
devtools::install_github("FelicienLL/mapbayr")
```

mapbayr relies on
[mrgsolve](https://github.com/metrumresearchgroup/mrgsolve) for model
implementation and ordinary differential equation solving which requires
C++ compilers. Please refer to the [installation
guide](https://github.com/metrumresearchgroup/mrgsolve/wiki/mrgsolve-Installation)
of mrgsolve for additional information.

## Example:

``` r
library(mapbayr)
library(mrgsolve)

my_model <- mread("ex_mbr3.cpp", mbrlib())

my_data <- data.frame(
  ID = 1, 
  time = c(0, 6, 20),
  amt = c(100, NA, NA), 
  rate = c(20, 0, 0), 
  DV = c(NA, 5, 2), 
  cmt = 1, 
  evid = c(1,0,0), 
  mdv = c(1,0,0)
)

est <- my_model %>% 
  data_set(my_data) %>% 
  mbrest()
```

``` r
print(est$mapbay_tab)
```

    ## # A tibble: 3 x 10
    ##      ID  time  evid   amt   cmt  rate   mdv    DV IPRED  PRED
    ##   <dbl> <dbl> <dbl> <dbl> <dbl> <dbl> <dbl> <dbl> <dbl> <dbl>
    ## 1     1     0     1   100     1    20     1    NA  0    0    
    ## 2     1     6     0     0     1     0     0     5  5.68 2.80 
    ## 3     1    20     0     0     1     0     0     2  1.69 0.163

``` r
est %>% 
  mbrpred() %>% 
  mbrplot()
```

![](README_files/figure-gfm/unnamed-chunk-4-1.png)<!-- -->

If you are too lazy to build a dataset by yourself, you can pass
administration and observation information through pipe-friendly
functions, such as:

``` r
est <- my_model %>% 
  adm_lines(time = 0, amt = 100, rate = 20) %>% 
  obs_lines(time = 6, DV = 5) %>% 
  obs_lines(time = 20, DV = 2) %>% 
  mbrest()
```

## Features

mapbayr is a generalization of the “MAP Bayes estimation” tutorial
available on the [mrgsolve
blog](https://mrgsolve.github.io/blog/map_bayes.html). Additional
features are:

  - a unique function to perform the estimation: `mbrest()`.
  - handles multiple error models such as additive, proportional, mixed
    or exponential error (without prior log-transformation of data).
  - fit multiple patients.
  - fit both parent drugs and metabolites simultaneously.
  - accepts any king of models thanks to the flexibility of mrgsolve.
  - functions to easily pass administration and observation information,
    as well as plot methods to visualize predictions and parameter
    distribution.
  - a single output object to ease post-processing, depending on the
    purpose of the estimation.
  - several estimation algorithm available, such as “NEWUOA” (the
    default) or “L-BFGS-B”.

## Performance

Performance, in terms of quality of parameter predictions, was validated
against NONMEM for a wide variety of models and data. However
predictions might differ depending on the complexity of the user’s data
or model, or as function of the number of parameter to estimate. The
user is invited to investigate if discrepancies come from the prediction
of the concentrations (i.e. mrgsolve) or from the optimization process
*per se* (i.e. mapbayr). Feel free to contact us through the [issue
tracker](https://github.com/FelicienLL/mapbayr/issues).

## *mrgsolve* model specification

mapbayr contains a library of example model files (.cpp), accessible
with `mbrlib()`

``` r
my_model <- mread("ex_mbr1.cpp", mbrlib())
```

The user is invited to perform map-bayesian estimation with his/her own
mrgsolve models. These model files should be slightly modified in order
to be “read” by mapbayr with the subsequent specifications :

### 1\. `$PREAMBLE` block

Two lines with `- drug` and `- model_ref`, describing the name of the
drug and the bibliographic reference of the model, must be filled. This
block still accepts free text to make notes about the model.

``` c
$PREAMBLE
- drug: examplinib
- model_ref: Smith et al, J Pharmacokinet, 2020
```

### 2\. `$PARAM` block

Add as many ETA as there are parameters to estimate. Refer them as ETAn
(n being the number of ETA). Set 0 as default value. Plain text provided
description will be used as internal “parameter names”. Text provided in
parentheses will be used as internal “parameter units” (use of empty
parentheses is advised for parameters without units).

``` c
$PARAM @annotated
ETA1 : 0 : CL (L/h)
ETA2 : 0 : VC (L)
ETA3 : 0 : F ()
//do not write ETA(1)
//do not write iETA
```

A `@covariates` tag must be used to record covariates in the `$PARAM`
block. Set the reference value. Plain text provided description will be
used as internal “covariate names”. Text provided in parentheses will be
used as “covariate units” (description of 0/1 coding is advised for
categorical covariates)

``` c
$PARAM @annotated @covariates
BW : 70 : Body weight (kg)
SEX : 0 : Sex (0=Male, 1=Female)
//do not write
//$PARAM @annotated
//BW : 70 : Body weight (kg)
```

When time or dose are needed as covariates, an internal routine is
embedded in mapbayr. You can refer them as TOLA and AOLA (i.e. time of
last administration, amount of last administration).

``` c
$PARAM @annotated @covariates
TOLA : 0 : Time of last adm (h)
AOLA : 100 : Amt of last adm (mg)
```

### 3\. `$OMEGA` block

Please ensure that omega values correspond to the order of the ETAs
provided in `$PARAM`. Omega values can be recorded in multiple blocks.

``` c
$OMEGA
0.123 0.456 0.789
$OMEGA @block
0.111 
0.222 0.333
```

### 4\. `$SIGMA` block

Two (diagonal) values are expected. The first will be used for the
proportional error, the second for (log) additive error.

``` c
$SIGMA 0.111 0 // proportional error 
$SIGMA 0 0.222 // (log) additive error
$SIGMA 0.333 0.444 // mixed error
```

When a parent drug and its metabolite are fitted simultaneously, four
values are expected.

``` c
//example: correlated proportional error between parent and metabolite
$SIGMA 
0.050 // proportional error on parent drug
0.000 0.000 // additive error on parent drug
0.100 0.000 0.200 // proportional error on metabolite
0.000 0.000 0.000 0.000 // additive error on metabolite
```

### 5\. `$CMT` block

A `@annotated` tag must be used to record compartments. Text provided in
parentheses will be used as internal “concentration units” (possible
values: **mg/L**, **ng/mL** or **pg/mL**). Text provided in brackets
will be used to define which parameters correspond to administration
\[ADM\] and/or observation \[OBS\] compartment.

``` c
//example: model with dual zero and first order absorption in compartment 1 & 2, respectively, and observation of parent drug + metabolite 
$CMT @annotated
DEPOT: Depot () [ADM]
CENT_PAR: examplinib central (ng/mL) [ADM, OBS]
PERIPH : examplinib peripheral
CENT_MET : methylexamplinib central (ng/mL) [OBS] 
```

### 6\. `$TABLE` block

Please refer the concentration variable to fit as `DV`. For log additive
error models, there is no need to log transform the data. Please
describe error as exponential to concentrations.

``` c
$TABLE
double DV  = (CENTRAL / VC) * exp(EPS(1)) ;
```

For fitting parent drug and metabolite simultaneously, please refer to
them as PAR and MET, and define DV accordingly (DV will be used for
computation of OFV)

``` c
$TABLE
double PAR = (CENT_PAR / V) * (1 + EPS(1)) ;
double MET = (CENT_MET / V) * (1 + EPS(3)) ;
double DV = PAR ;
if(self.cmt == 4) DV = MET ;
```

### 7\. `$MAIN` block

Double every expression containing ETA information, with ETAn (used for
estimation of parameters) and ETA(n) (generated for simulations with
random effects)

``` c
$PK
double CL = TVCL * exp(ETA1 + ETA(1))
```

### 8\. `$CAPTURE` block

DV and ETAn must be captured, as well as PAR and MET for models with
parent + metabolite.

``` c
$CAPTURE 
DV ETA1 ETA2 ETA3 PAR MET
```
