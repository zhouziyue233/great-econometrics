# Econometric Methods in Stata

## Contents

1. [Linear Regression (OLS)](#linear-regression-ols)
2. [Panel Data Methods](#panel-data-methods)
3. [Difference-in-Differences](#difference-in-differences)
4. [Instrumental Variables and GMM](#instrumental-variables-and-gmm)
5. [Regression Discontinuity Design](#regression-discontinuity-design)
6. [Treatment Effects and Matching](#treatment-effects-and-matching)
7. [Time Series](#time-series)
8. [Maximum Likelihood Estimation](#maximum-likelihood-estimation)
9. [Bootstrap and Simulation](#bootstrap-and-simulation)
10. [Survey Data Analysis](#survey-data-analysis)
11. [Limited Dependent Variables](#limited-dependent-variables)
12. [Sample Selection](#sample-selection)
13. [Nonparametric Methods](#nonparametric-methods)

---

## Linear Regression (OLS)

### The regress Command

```stata
regress depvar indepvars [if] [in] [weight] [, options]

sysuse auto, clear
regress price mpg weight foreign
regress price mpg weight, beta           // Standardized coefficients
regress price mpg weight, noconstant     // Suppress constant
regress price mpg weight, level(90)      // 90% confidence level
```

### Post-Estimation Commands

**predict** - Fitted values and diagnostics:
```stata
predict price_hat                // Fitted values
predict residuals, residuals     // Residuals (y - yhat)
predict std_res, rstandard       // Standardized residuals
predict stud_res, rstudent       // Studentized residuals
predict se_pred, stdp            // SE of prediction
predict cooksd, cooksd           // Cook's distance
predict leverage, leverage       // Hat values
```

**margins** - Marginal effects:
```stata
regress wage c.education##c.experience i.gender

margins, dydx(education)                              // Average marginal effect
margins, at(education=(12 16 20))                     // Predictions at values
margins gender                                         // Predictive margins by category
marginsplot                                            // Visualize
```

**test** and **lincom** - Coefficient tests:
```stata
test education                        // H0: beta = 0
test education = 0.5                  // H0: beta = specific value
test education experience             // Joint test: both = 0
lincom education + experience         // Sum of coefficients
lincom 2*education + 3*experience     // Weighted combination
```

**Store and compare**:
```stata
estimates store model1
regress price mpg weight
estimates store model2
esttab model1 model2, se star(* 0.05 ** 0.01 *** 0.001)
```

### Standard Errors

**Robust (Huber-White) standard errors** - correct for heteroskedasticity:
```stata
regress wage education experience, vce(robust)
regress wage education experience, vce(hc2)  // HC2 variant (better small samples)
```

**Clustered standard errors** - for grouped/panel data:
```stata
regress test_score hours_studied class_size, vce(cluster school_id)
// Need ~50+ clusters for reliable inference
// Automatically robust to heteroskedasticity

// Two-way clustering
reghdfe wage education experience, absorb(i.firm_id i.year) vce(cluster firm_id year)
```

### Interaction Terms

**Factor variable operators**:
```stata
i.varname       // Categorical indicators
c.varname       // Continuous (explicit)
var1#var2       // Interaction only
var1##var2      // Main effects AND interaction

* Continuous x Continuous
regress wage c.education##c.experience
margins, dydx(education) at(experience=(5 10 15 20))

* Categorical x Continuous
regress wage i.gender##c.education
margins gender, at(education=(8(2)20))

* Reference categories
regress wage ib3.education_cat          // Category 3 as reference
regress wage ib(-1).time#1.treated      // base = time period -1
```

### Regression Diagnostics

```stata
rvfplot                    // Residual vs fitted
avplot mpg                 // Added-variable plot
avplots                    // All added-variable plots
qnorm residuals            // Q-Q plot for normality

estat hettest              // Breusch-Pagan (H0: homoskedasticity)
estat vif                  // VIF > 10 = multicollinearity
estat ovtest               // Ramsey RESET (H0: no omitted vars)
```

### Weighted Regression

| Weight Type | Syntax | When to Use |
|-------------|--------|-------------|
| Frequency | `[fweight=n]` | Aggregated data |
| Analytic | `[aweight=w]` | Inverse variance weights |
| Probability | `[pweight=w]` | Survey data; auto-robust SEs |

```stata
regress outcome predictor [aweight=n]
regress outcome predictor [pweight=survey_weight]
```

---

## Panel Data Methods

### Setting Up: xtset

```stata
xtset panelvar timevar [, options]
xtset country year
xtset firm_id year, yearly
xtdescribe                        // Check balance
```

**Time-series operators** (after xtset):
```stata
L.variable      // First lag (t-1)
L2.variable     // Second lag
D.variable      // First difference
F.variable      // First lead

xtreg consumption L.gdp L.income, fe
```

**Descriptive**:
```stata
xtsum ln_wage age tenure          // Between/within decomposition
xttab union                        // Cross-tabulation
xttrans union                      // Transition probabilities
```

### Fixed Effects (FE)

Eliminates time-invariant heterogeneity by demeaning. Cannot estimate effects of time-invariant variables.

```stata
xtreg ln_wage age tenure, fe
xtreg ln_wage age tenure, fe vce(robust)
xtreg ln_wage age tenure, fe vce(cluster idcode)

// Entity AND time fixed effects
xtreg ln_wage age tenure i.year, fe
testparm i.year                         // Test if time FEs needed

// High-dimensional FE (faster, more flexible)
reghdfe ln_wage age tenure, absorb(idcode)
reghdfe ln_wage age tenure, absorb(idcode year)
reghdfe ln_wage age tenure, absorb(idcode year) vce(cluster idcode year)
```

**Key output statistics**:
- **Within R-squared**: Variation explained within entities
- **sigma_u**: SD of entity-specific effects
- **rho**: Fraction of variance due to entity effects

### Random Effects (RE)

Uses GLS; assumes individual effects uncorrelated with regressors. More efficient than FE when assumption holds; can include time-invariant variables.

```stata
xtreg ln_wage age tenure, re
xtreg ln_wage age tenure female, re      // Can include time-invariant vars
```

### First Differences (FD)

Eliminates individual effects by differencing. Preferred when T is small.

```stata
xtreg gdp investment population, fd
xtreg gdp investment population, fd vce(robust)
```

### Hausman Test: FE vs RE

Tests whether individual effects correlated with regressors. H0: RE is consistent.

```stata
quietly xtreg ln_wage age tenure, fe
est store fixed
xtreg ln_wage age tenure, re
hausman fixed .
```

### Dynamic Panel Models

For lagged dependent variables; use Arellano-Bond/Blundell-Bond (requires `xtabond2`):

```stata
ssc install xtabond2
xtabond2 y L.y x1 x2, gmm(L.y, lag(1 3)) iv(x1 x2)
```

---

## Difference-in-Differences

### Classic 2x2 DiD

```stata
* Data structure: treated group, post-treatment period
gen post = (year >= 2010)
gen treat_post = treated * post

regress y treated post treat_post, vce(cluster id)
// treat_post coefficient = DiD effect
```

### Event Study (Uniform Timing)

```stata
* Create relative time variable (year - treatment_year)
gen rel_time = year - treatment_year

* Omit base period (e.g., year -1 before treatment)
reghdfe y ib(-1).rel_time#1.treated, absorb(id year) vce(cluster id)

* Test for parallel trends (pre-treatment coefficients should be zero)
testparm *(-5 -4 -3 -2).rel_time#1.treated
```

### Modern Staggered DiD (Callaway & Sant'Anna)

For settings where units are treated at different times.

```stata
ssc install csdid
csdid y x1 x2, ivar(id) time(year) gvar(first_treat) agg(event)
csdid_plot
```

**Key advantages over standard 2FE**: Robust to treatment effect heterogeneity across cohorts; does not mechanically average treatments with different signs.

---

## Instrumental Variables and GMM

### 2SLS (Instrumental Variables)

```stata
ivregress 2sls wage experience (education = parent_ed), first
// first: Shows first-stage regression

// Multiple instruments
ivregress 2sls wage experience (education = parent_ed sibling_ed)

// Multiple endogenous variables
ivregress 2sls y x1 (x2 x3 = z1 z2 z3)
```

**Critical gotcha**: All exogenous variables are automatically instruments. Don't omit them:
```stata
* WRONG: ivregress 2sls y (x1 = z)         -- missing exogenous x2
* RIGHT: ivregress 2sls y x2 (x1 = z)
```

**IV diagnostics**:
```stata
ivregress 2sls wage experience (education = parent_ed sibling_ed), first

estat firststage       // First-stage F-stat (F > 10 = strong instruments)
estat endogenous       // Durbin-Wu-Hausman test (H0: variable is exogenous)
estat overid           // Sargan/Hansen test (requires overidentification)
```

### Other IV Estimators

```stata
ivregress liml wage experience (education = parent_ed), first   // Better with weak instruments
ivregress gmm wage experience (education = parent_ed), wmatrix(robust)
```

### GMM Estimation

```stata
gmm y x1 x2, instruments(z1 z2 z3 x1 x2) onestep
gmm y x1 x2, instruments(z1 z2 z3 x1 x2) twostep

estat overid           // Overidentifying restrictions test
```

---

## Regression Discontinuity Design

For causal effects at a cutoff threshold. Sharp RD: treatment is deterministic; Fuzzy RD: treatment is probabilistic.

### Sharp RD

```stata
* Simple: regress on running variable, treatment dummy, interaction
gen treated = (running_var >= cutoff)
gen centered_running = running_var - cutoff

regress y treated c.centered_running c.centered_running#treated

* Visualize
twoway scatter y running_var if running_var < cutoff, color(blue) ///
    || scatter y running_var if running_var >= cutoff, color(red)
```

### Fuzzy RD

```stata
* Treatment assignment affected by cutoff but imperfect
gen assignment = (running_var >= cutoff)

ivregress 2sls y (treatment = assignment) centered_running, first
```

### rdrobust (RD Robust Package)

Recommended for publication-quality RD with optimal bandwidth selection:

```stata
ssc install rdrobust
rdrobust y running_var, c(cutoff) p(2) q(3)
rdplot y running_var, c(cutoff)
```

---

## Treatment Effects and Matching

### Potential Outcomes and ATE/ATT

- **ATE** (Average Treatment Effect): E[Y1 - Y0]
- **ATT** (Average Treatment Effect on Treated): E[Y1 - Y0 | treated=1]
- **ATET**: Average Treatment Effect on Treated (same as ATT)

### teffects - Stata's Built-In

```stata
* Regression Adjustment (RA)
teffects ra (y i.age i.education) (treated)

* Inverse Probability Weighting (IPW)
teffects ipw (y) (treated i.age i.education)

* IPW with Regression Adjustment (IPWRA)
teffects ipwra (y i.age i.education) (treated i.age i.education)

* Augmented IPW (AIPW)
teffects aipw (y i.age i.education) (treated i.age i.education)

* Extract results
tebalance summarize                   // Check covariate balance
```

### Propensity Score Matching

```stata
* Manual approach
logit treated age education income
predict ps, pr                         // Propensity score

* Nearest neighbor matching
psmatch2 treated age education income, outcome(earnings) ///
    pscore(ps) neighbor(1) caliper(.01) noreplacement

* Visualize balance before/after
pstest age education income, both graph
```

### Kernel/Local Polynomial Matching

```stata
ssc install lpdensity
teffects psmatch (y) (treated i.age i.education), nneighbor(5) kernel(tricube) atet
```

---

## Time Series

### Setting Up: tsset

```stata
tsset time_var [, options]
tsset year, yearly
tsset month, monthly
tsset date_var, delta(1 day)      // For daily data
```

### Stationarity Tests

```stata
dfuller y                           // Augmented Dickey-Fuller
pperron y                           // Phillips-Perron
* H0: unit root (non-stationary)
```

### ARIMA

```stata
arima y, arima(p,d,q)
arima y, arima(1,1,1)               // ARIMA(1,1,1)

* Forecasting
forecast create model1
predict yhat, y                     // Back-transform if d > 0
forecast evaluate                   // Evaluation metrics
```

**Gotcha after differencing (d > 0)**: Use `predict yhat, y` (not `xb`) to get predictions on original scale.

### VAR Models

```stata
var y1 y2 y3
irf create irfs, set(irfs) replace  // Impulse response functions
irf graph oirf                      // Plot IRFs
```

### Granger Causality

```stata
regress y L.y L.x
test L.x*                           // Does x Granger-cause y?
```

---

## Maximum Likelihood Estimation

### ml model - Custom Likelihood

For custom likelihood functions not available in built-in commands.

```stata
program define mylogit_lf
    args lnf xb
    tempvar p
    gen double `p' = invlogit(`xb')
    replace `lnf' = $ML_y1 * ln(`p') + (1 - $ML_y1) * ln(1 - `p')
end

ml model lf mylogit_lf (y: y = x1 x2)
ml init 0.5
ml maximize
```

### Accessing ML Results

```stata
ml display                          // Display estimates
estimates store mymodel
ereturn list
matrix list e(V)                    // Variance-covariance matrix
```

### Logit and Probit

Technically ML-based, commonly treated as standard estimators:

```stata
logit y x1 x2                      // Binary outcome
probit y x1 x2
margins, dydx(x1)                  // Marginal effects at means

mlogit y x1 x2                     // Multinomial logit
ologit y x1 x2                     // Ordered logit
```

---

## Bootstrap and Simulation

### Bootstrap

```stata
bootstrap, reps(1000) seed(12345): regress y x1 x2
bootstrap, reps(1000): estat gini   // Bootstrap of complex statistic

* Bootstrap with cluster
bootstrap, reps(1000) cluster(firm_id): regress y x1 x2

* Manual bootstrap for complex estimators
bsample      // Resample with replacement
// ... compute statistic ...
// repeat in loop
```

### Simulate

```stata
simulate, reps(1000): ttest y == 0
simulate beta_x = _b[x], seed(12345): regress y x
```

### Monte Carlo

```stata
set obs 1000
gen x = rnormal(0, 1)
gen y = 2 + 3*x + rnormal(0, 1)    // Simulate data with known DGP

* Run analysis
regress y x
```

---

## Survey Data Analysis

### Setting Up: svyset

```stata
svyset psu_var [pweight=weight_var]
svyset psu_var [pweight=weight_var], strata(strata_var)

* Complex design
svyset psu_var [pweight=weight_var], strata(strata_var) fpc(fpc_var)
```

### Survey Estimation

```stata
svy: mean age income
svy: proportion employed
svy: regress wage age education

svy: logit employed age education
margins, dydx(age)                  // Marginal effects

* Subpopulation
svy, subpop(if age > 25): regress wage age education

* Replicate weights
svyset [pweight=weight], replicates(rep1-rep50)
```

### Survey Design Effects and Diagnostics

```stata
estat effect                        // Design effects
estat headerinfo                    // Survey design details
```

---

## Limited Dependent Variables

### Logit and Probit (Binary)

```stata
logit y x1 x2
probit y x1 x2

* Marginal effects
margins, dydx(x1) at(mean)          // At sample means
margins, dydx(x1)                   // Average marginal effect

* Predictions
predict pr_y                        // Predicted probability
predict xb_y, xb                    // Linear prediction
```

### Multinomial Logit

```stata
mlogit outcome x1 x2
margins outcome
predict pr_*, pr
```

### Ordered Logit/Probit

```stata
ologit satisfaction x1 x2
* Assumes proportional odds (test with: brant)
```

### Count Data

```stata
poisson events x1 x2
nbreg events x1 x2                  // Negative binomial (if overdispersed)

* Zero-inflated models
zipoisson y x1, inflate(x2)         // Zero-inflated Poisson
zinb y x1, inflate(x2)              // Zero-inflated negative binomial
```

### Tobit (Censored)

```stata
tobit y x1 x2, ll(0)               // Left-censored at 0
tobit y x1 x2, ul(100)             // Right-censored at 100

margins, dydx(x1)                  // Marginal effect on latent y*
margins, dydx(x1) predict(pr)       // Marginal effect on Pr(uncensored)
```

---

## Sample Selection

### Heckman Selection Model

For data missing not at random (MNAR). Estimates both selection and outcome equations.

```stata
heckman wage education age, select(employed = education age married)

* Interpretation
display "Inverse Mills ratio (lambda): " e(lambda)
* Significant lambda indicates selection bias
```

### Heckprobit

```stata
heckprobit outcome x1, select(selection_var = x2 x3)
```

### Two-Stage Manual Approach

```stata
* Stage 1: Estimate selection (probit)
probit employed education age married
predict mills, scores               // Inverse Mills ratio

* Stage 2: Regress on main variables + Mills ratio
regress wage education age mills
```

---

## Nonparametric Methods

### Quantile Regression

Estimate effects on conditional quantiles (median, quartiles, etc.):

```stata
qreg y x1 x2, quantile(0.50)        // Median
qreg y x1 x2, quantile(0.25)        // 25th percentile

* All quantiles at once
sqreg y x1 x2, quantile(0.25 0.50 0.75)
```

### Kernel Density Estimation

```stata
kdensity x
kdensity x, bwidth(0.5) generate(xval density)
```

### Nonparametric Regression

```stata
npregress y x1 x2
margins                             // Partial effects estimated nonparametrically
```

### Rank-Based Tests

```stata
signrank y == 0                     // Wilcoxon signed-rank test
ranksum y, by(group)                // Mann-Whitney U test
```

---

## Quick Reference: When to Use What

| Problem | Method | Command |
|---------|--------|---------|
| Endogenous regressor | 2SLS | `ivregress 2sls` |
| Panel with time-invariant unobservable | Fixed Effects | `xtreg fe` |
| Selection on observables | Matching | `teffects` |
| Causal effect at threshold | RDD | `rdrobust` |
| Staggered treatment timing | Modern DiD | `csdid` |
| Complex survey | SVY | `svy: regress` |
| Censored outcome | Tobit | `tobit` |
| Non-random missingness | Heckman | `heckman` |
| Distributional effects | Quantile reg | `qreg` |
| Weak instruments | LIML | `ivregress liml` |
