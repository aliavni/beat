# generalized random forests <a href='https://grf-labs.github.io/grf/'><img src='https://raw.githubusercontent.com/grf-labs/grf/master/images/logo/grf_logo_wbg_cropped.png' align="right" height="120" /></a>

Fork from [https://github.com/grf-labs/grf](https://github.com/grf-labs/grf)
## What's Different:
* Original package use symbolic links for all C++ codes, but Windows users may fail to build it from source. All C++ code locations are  re-arranged. 
* Add target weight penalty in `regression_forest`
* penalty is calculated as following:
   1. calculate average sampled target weight;
   2. in the left node, calculate average target weight; 
   3. take l1 norm of the difference between the two vectors; 
* Main change: `src\src\splitting\RegressionSplittingRule.cpp`
* Package version is 1.2.1

## Build From Source:
```
git clone https://github.com/yu45020/grf
cd grf/r-package/grf
Use Rstudio to open it as a project. Or double click 'grf.Rproj'
Click Build --> Install and Restart
```

--

Sample usage:
```{R}
rm(list=ls())
library(grf)
library(data.table)
library(ggpubr)
r_2 = function(y_true, y_hat){
  mean_y_true = mean(y_true)
  top = (y_hat - mean_y_true)^2
  bottom = (y_true - mean_y_true)^2
  return(sum(top)/sum(bottom))
}
set.seed(1)
n <- 5000
p <- 10
X <- matrix(rnorm(n * p), n, p)
Z = matrix(c(sample(x=c(0,1), size=n, replace=TRUE, prob=c(0.3, 0.7)),
             sample(x=c(0,1), size=n, replace=TRUE)),
           n, 2)
# Z = matrix(c(sample(x=seq(-100,100), size=n, replace=TRUE ),
#              sample(x=seq(-1000,1000), size=n, replace=TRUE )),
#            n, 2)
Y <- X[, 1] * rnorm(n) +  Z[, 1] * rnorm(n)  +  Z[, 2] * rnorm(n)


r.forest <- regression_forest(X, Y, num.trees = 200, target.weights=Z) #

y_hat = predict(r.forest)

out = data.table(y_true = Y, y_hat = y_hat$predictions, Z=Z)
out[, idx := 1:.N]
out = melt(out, measure.vars = c("y_true", 'y_hat'))
out[, Z.V1:= as.character(Z.V1)]
out[, Z.V2:= as.character(Z.V2)]

p1 = ggpubr::ggdensity(data=out, x='value', color='Z.V1',facet.by = 'variable',
                       title='With Weights',
                       subtitle = sprintf("R2: %.4f", r_2(Y,y_hat$predictions)))

p2 = ggpubr::ggdensity(data=out, x='value', color='Z.V2',facet.by = 'variable'  )
p3 = ggpubr::ggdensity(data=out, x='value', color='variable'  )
ggarrange(p1,p2,p3)
ggsave("test_weights.png", width = 14, height=10)



r.forest <- regression_forest(X, Y, num.trees = 1000, target.weights=NULL) #

y_hat = predict(r.forest)
out = data.table(y_true = Y, y_hat = y_hat$predictions, Z=Z)
out[, idx := 1:.N]
out = melt(out, measure.vars = c("y_true", 'y_hat'))
out[, Z.V1:= as.character(Z.V1)]
out[, Z.V2:= as.character(Z.V2)]

p1 = ggpubr::ggdensity(data=out, x='value', color='Z.V1',facet.by = 'variable',
                       title='Without Weights',
                       subtitle = sprintf("R2: %.4f", r_2(Y,y_hat$predictions)))
p2 = ggpubr::ggdensity(data=out, x='value', color='Z.V2',facet.by = 'variable'  )
p3 = ggpubr::ggdensity(data=out, x='value', color='variable'  )
ggarrange(p1,p2,p3 )
ggsave("test_no_weights.png", width = 14, height=10)
```
Original ReadME
-----

[![CRANstatus](https://www.r-pkg.org/badges/version/grf)](https://cran.r-project.org/package=grf)
![CRAN Downloads overall](http://cranlogs.r-pkg.org/badges/grand-total/grf)
[![Build Status](https://travis-ci.com/grf-labs/grf.svg?branch=master)](https://travis-ci.com/grf-labs/grf)

A pluggable package for forest-based statistical estimation and inference. GRF currently provides non-parametric methods for least-squares regression, quantile regression, survival regression, and treatment effect estimation (optionally using instrumental variables), with support for missing values.

In addition, GRF supports 'honest' estimation (where one subset of the data is used for choosing splits, and another for populating the leaves of the tree), and confidence intervals for least-squares regression and treatment effect estimation.

Some helpful links for getting started:

- The [R package documentation](https://grf-labs.github.io/grf) contains usage examples and method reference.
- The [GRF reference](https://grf-labs.github.io/grf/REFERENCE.html) gives a detailed description of the GRF algorithm and includes troubleshooting suggestions.
- For community questions and answers around usage, see [Github issues labelled 'question'](https://github.com/grf-labs/grf/issues?q=label%3Aquestion).

The repository first started as a fork of the [ranger](https://github.com/imbs-hl/ranger) repository -- we owe a great deal of thanks to the ranger authors for their useful and free package.

### Installation

The latest release of the package can be installed through CRAN:

```R
install.packages("grf")
```

`conda` users can install from the [conda-forge](https://anaconda.org/conda-forge/r-grf) channel:

```
conda install -c conda-forge r-grf
```

The current development version can be installed from source using devtools.

```R
devtools::install_github("grf-labs/grf", subdir = "r-package/grf")
```

Note that to install from source, a compiler that implements C++11 is required (clang 3.3 or higher, or g++ 4.8 or higher). If installing on Windows, the RTools toolchain is also required.

### Usage Examples

The following script demonstrates how to use GRF for heterogeneous treatment effect estimation. For examples
of how to use types of forest, as for quantile regression and causal effect estimation using instrumental
variables, please consult the R [documentation](https://grf-labs.github.io/grf/reference/index.html) on the relevant forest methods (quantile_forest, instrumental_forest, etc.).

```R
library(grf)

# Generate data.
n <- 2000
p <- 10
X <- matrix(rnorm(n * p), n, p)
X.test <- matrix(0, 101, p)
X.test[, 1] <- seq(-2, 2, length.out = 101)

# Train a causal forest.
W <- rbinom(n, 1, 0.4 + 0.2 * (X[, 1] > 0))
Y <- pmax(X[, 1], 0) * W + X[, 2] + pmin(X[, 3], 0) + rnorm(n)
tau.forest <- causal_forest(X, Y, W)

# Estimate treatment effects for the training data using out-of-bag prediction.
tau.hat.oob <- predict(tau.forest)
hist(tau.hat.oob$predictions)

# Estimate treatment effects for the test sample.
tau.hat <- predict(tau.forest, X.test)
plot(X.test[, 1], tau.hat$predictions, ylim = range(tau.hat$predictions, 0, 2), xlab = "x", ylab = "tau", type = "l")
lines(X.test[, 1], pmax(0, X.test[, 1]), col = 2, lty = 2)

# Estimate the conditional average treatment effect on the full sample (CATE).
average_treatment_effect(tau.forest, target.sample = "all")

# Estimate the conditional average treatment effect on the treated sample (CATT).
# Here, we don't expect much difference between the CATE and the CATT, since
# treatment assignment was randomized.
average_treatment_effect(tau.forest, target.sample = "treated")

# Add confidence intervals for heterogeneous treatment effects; growing more trees is now recommended.
tau.forest <- causal_forest(X, Y, W, num.trees = 4000)
tau.hat <- predict(tau.forest, X.test, estimate.variance = TRUE)
sigma.hat <- sqrt(tau.hat$variance.estimates)
plot(X.test[, 1], tau.hat$predictions, ylim = range(tau.hat$predictions + 1.96 * sigma.hat, tau.hat$predictions - 1.96 * sigma.hat, 0, 2), xlab = "x", ylab = "tau", type = "l")
lines(X.test[, 1], tau.hat$predictions + 1.96 * sigma.hat, col = 1, lty = 2)
lines(X.test[, 1], tau.hat$predictions - 1.96 * sigma.hat, col = 1, lty = 2)
lines(X.test[, 1], pmax(0, X.test[, 1]), col = 2, lty = 1)

# In some examples, pre-fitting models for Y and W separately may
# be helpful (e.g., if different models use different covariates).
# In some applications, one may even want to get Y.hat and W.hat
# using a completely different method (e.g., boosting).

# Generate new data.
n <- 4000
p <- 20
X <- matrix(rnorm(n * p), n, p)
TAU <- 1 / (1 + exp(-X[, 3]))
W <- rbinom(n, 1, 1 / (1 + exp(-X[, 1] - X[, 2])))
Y <- pmax(X[, 2] + X[, 3], 0) + rowMeans(X[, 4:6]) / 2 + W * TAU + rnorm(n)

forest.W <- regression_forest(X, W, tune.parameters = "all")
W.hat <- predict(forest.W)$predictions

forest.Y <- regression_forest(X, Y, tune.parameters = "all")
Y.hat <- predict(forest.Y)$predictions

forest.Y.varimp <- variable_importance(forest.Y)

# Note: Forests may have a hard time when trained on very few variables
# (e.g., ncol(X) = 1, 2, or 3). We recommend not being too aggressive
# in selection.
selected.vars <- which(forest.Y.varimp / mean(forest.Y.varimp) > 0.2)

tau.forest <- causal_forest(X[, selected.vars], Y, W,
                            W.hat = W.hat, Y.hat = Y.hat,
                            tune.parameters = "all")

# Check whether causal forest predictions are well calibrated.
test_calibration(tau.forest)
```

### Developing

In addition to providing out-of-the-box forests for quantile regression and causal effect estimation, GRF provides a framework for creating forests tailored to new statistical tasks. If you'd like to develop using GRF, please consult the [algorithm reference](https://grf-labs.github.io/grf/REFERENCE.html) and [development guide](https://grf-labs.github.io/grf/DEVELOPING.html).

### Funding

Development of GRF is supported by the National Science Foundation, the Sloan Foundation, the Office of Naval Research (Grant N00014-17-1-2131) and Schmidt Futures.

### References

Susan Athey and Stefan Wager.
<b>Estimating Treatment Effects with Causal Forests: An Application.</b>
<i>Observational Studies</i>, 5, 2019.
[<a href="https://obsstudies.org/wp-content/uploads/2019/09/all-papers-compiled.pdf">paper</a>,
<a href="https://arxiv.org/abs/1902.07409">arxiv</a>]

Susan Athey, Julie Tibshirani and Stefan Wager.
<b>Generalized Random Forests.</b> <i>Annals of Statistics</i>, 47(2), 2019.
[<a href="https://projecteuclid.org/euclid.aos/1547197251">paper</a>,
<a href="https://arxiv.org/abs/1610.01271">arxiv</a>]

Rina Friedberg, Julie Tibshirani, Susan Athey, and Stefan Wager.
<b>Local Linear Forests.</b> 2018.
[<a href="https://arxiv.org/abs/1807.11408">arxiv</a>]

Imke Mayer, Erik Sverdrup, Tobias Gauss, Jean-Denis Moyer, Stefan Wager and Julie Josse.
<b>Doubly Robust Treatment Effect Estimation with Missing Attributes.</b>
<i>Annals of Applied Statistics</i>, forthcoming.
[<a href="https://arxiv.org/pdf/1910.10624.pdf">arxiv</a>]

Stefan Wager and Susan Athey.
<b>Estimation and Inference of Heterogeneous Treatment Effects using Random Forests.</b>
<i>Journal of the American Statistical Association</i>, 113(523), 2018.
[<a href="https://www.tandfonline.com/eprint/v7p66PsDhHCYiPafTJwC/full">paper</a>,
<a href="http://arxiv.org/abs/1510.04342">arxiv</a>]