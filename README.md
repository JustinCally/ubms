# ubms: Unmarked Bayesian Models with Stan

[![Build
Status](https://travis-ci.org/kenkellner/ubms.svg?branch=master)](https://travis-ci.org/kenkellner/ubms)
[![codecov](https://codecov.io/gh/kenkellner/ubms/branch/master/graph/badge.svg)](https://codecov.io/gh/kenkellner/ubms)

In-development `R` package for fitting Bayesian hierarchical models of
animal occurrence and abundance. The package has a formula-based
interface compatible with
[unmarked](https://cran.r-project.org/web/packages/unmarked/index.html),
but the model is fit using MCMC with [Stan](https://mc-stan.org/)
instead of using maximum likelihood. Right now there are Stan versions
of unmarked functions `occu`, `occuRN`, `colext`, `pcount`, `distsamp`,
and `multinomPois`. These functions follow the `stan_` prefix naming
format established by
[rstanarm](https://cran.r-project.org/web/packages/rstanarm/index.html).
For example, the Stan version of the `unmarked` function `occu` is
`stan_occu`.

Advantages compared to `unmarked`:

1.  Obtain posterior distributions of parameters and derived parameters
2.  Include random effects in parameter formulas (same syntax as `lme4`)
3.  Assess model fit using WAIC and LOO via the
    [loo](https://cran.r-project.org/web/packages/loo/index.html)
    package

Disadvantages compared to `unmarked`:

1.  MCMC is slower than maximum likelihood
2.  Not all model types are supported
3.  Potential for convergence issues

### Example

Simulate occupancy data including a random effect on occupancy:

``` r
library(ubms)

set.seed(123)
dat_occ <- data.frame(x1=rnorm(500))
dat_p <- data.frame(x2=rnorm(500*5))

y <- matrix(NA, 500, 5)
z <- rep(NA, 500)

b <- c(0.4, -0.5, 0.3, 0.5)

re_fac <- factor(sample(letters[1:26], 500, replace=T))
dat_occ$group <- re_fac
re <- rnorm(26, 0, 1.2)
re_idx <- as.numeric(re_fac)

idx <- 1
for (i in 1:500){
  z[i] <- rbinom(1,1, plogis(b[1] + b[2]*dat_occ$x1[i] + re[re_idx[i]]))
  for (j in 1:5){
    y[i,j] <- z[i]*rbinom(1,1, 
                    plogis(b[3] + b[4]*dat_p$x2[idx]))
    idx <- idx + 1
  }
}
```

Create `unmarked` frame:

``` r
umf <- unmarkedFrameOccu(y=y, siteCovs=dat_occ, obsCovs=dat_p)
```

Fit a model with a random intercept:

``` r
options(mc.cores=3) #number of parallel cores to use
(fm <- stan_occu(~x2 ~x1 + (1|group), umf, refresh=0))
```

    ## Warning: Some Pareto k diagnostic values are slightly high. See help('pareto-k-diagnostic') for details.

    ## 
    ## Call:
    ## stan_occu(formula = ~x2 ~ x1 + (1 | group), data = umf, refresh = 0)
    ## 
    ## Occupancy (logit-scale):
    ##                 Estimate    SD   2.5%  97.5% n_eff  Rhat
    ## (Intercept)        0.328 0.294 -0.244  0.909   997 1.003
    ## x1                -0.466 0.115 -0.695 -0.248  5433 0.999
    ## sigma [1|group]    1.397 0.286  0.936  2.074  2529 0.999
    ## 
    ## Detection (logit-scale):
    ##             Estimate     SD  2.5% 97.5% n_eff Rhat
    ## (Intercept)    0.381 0.0622 0.259 0.503  6282    1
    ## x2             0.588 0.0632 0.466 0.714  7106    1
    ## 
    ## LOOIC: 2268.24

Examine residuals for occupancy and detection submodels (following
[Wright et al. 2019](https://doi.org/10.1002/ecy.2703)). Each panel
represents a draw from the posterior distribution.

``` r
plot(fm)
```

![](README_figs/README-resids-1.png)<!-- -->

Assess model goodness-of-fit with a posterior predictive check, using
the MacKenzie-Bailey chi-square test:

``` r
(fm_fit <- gof(fm, draws=500, quiet=TRUE))
```

    ## MacKenzie-Bailey Chi-square 
    ## Point estimate = 30.381
    ## Posterior predictive p = 0.482

``` r
plot(fm_fit)
```

![](README_figs/README-gof-1.png)<!-- -->

Look at the marginal effect of `x2` on detection:

``` r
plot_marginal(fm, "det")
```

![](README_figs/README-marginal-1.png)<!-- -->
