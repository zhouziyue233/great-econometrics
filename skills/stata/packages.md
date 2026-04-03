# Stata Community Packages Reference

## Table of Contents

1. [Package Management](#package-management)
2. [Estimation: High-Dimensional Fixed Effects & IV](#estimation-high-dimensional-fixed-effects--iv)
3. [Causal Inference: Difference-in-Differences](#causal-inference-difference-in-differences)
4. [Causal Inference: Regression Discontinuity, Matching, Synthetic Control](#causal-inference-regression-discontinuity-matching-synthetic-control)
5. [Dynamic Panel Data](#dynamic-panel-data)
6. [Visualization & Binning](#visualization--binning)
7. [Data Manipulation & Tools](#data-manipulation--tools)
8. [Diagnostics & Testing](#diagnostics--testing)

---

## Package Management

### Installing Packages

```stata
* SSC (most common -- ~3500 packages)
ssc install reghdfe, replace

* Stata Journal
net from http://www.stata-journal.com/software/sj16-3
net install st0448

* GitHub (requires github package first)
net install github, from("https://haghish.github.io/github/")
github install mcaceresb/stata-gtools

* Search before installing
findit "difference in differences"   // Most comprehensive
ssc describe reghdfe, all             // SSC package info
```

### Updating and Maintenance

```stata
ado update, update                     // Install user package updates
update all                             // Install official Stata updates
ado dir                                // List installed packages
which reghdfe                          // Find package location
ado uninstall packagename              // Remove package
```

### Reproducibility Script

```stata
* Auto-install missing packages
local packages reghdfe ftools estout coefplot ivreg2 ranktest

foreach pkg of local packages {
    capture which `pkg'
    if _rc {
        ssc install `pkg', replace
    }
}
```

---

## Estimation: High-Dimensional Fixed Effects & IV

### reghdfe: High-Dimensional Fixed Effects Regression

Absorbs multiple fixed effects efficiently without creating dummy variables.

```stata
ssc install reghdfe ftools

* Two-way FE with clustering
reghdfe wage tenure education, absorb(firm_id year) ///
    vce(cluster firm_id)

* Save fixed effects for later analysis
reghdfe wage tenure education, ///
    absorb(fe_firm=firm_id fe_year=year) ///
    vce(cluster firm_id)

* Interacted FE (e.g., industry-year effects)
reghdfe sales advertising, absorb(firm_id industry#year) ///
    cluster(state)

* Key diagnostics
ereturn list  // Shows e(r2_within), e(N), etc.
```

**Gotcha:** Singletons (units appearing only once) are automatically dropped. Check with `verbose(3)` to see how many.

### ivreghdfe: IV Estimation with High-Dimensional FE

Combines IV with `reghdfe` for fast estimation with multiple fixed effects.

```stata
ssc install ivreghdfe

* Basic 2SLS with two-way FE
ivreghdfe wage (education = parent_educ), absorb(firm_id year) ///
    cluster(firm_id)

* First-stage diagnostics
ivreghdfe wage (education = parent_educ sibling_educ), ///
    absorb(firm_id year) cluster(firm_id) first
* Check e(widstat) for Kleibergen-Paap F-stat (rule of thumb: F > 10)

* Save first-stage estimates
ivreghdfe wage (education = parent_educ sibling_educ), ///
    absorb(firm_id year) cluster(firm_id) first savefirst
estimates restore _ivreg2_education  // First-stage results
```

### ivreg2: Comprehensive IV with Diagnostics

Extended IV command with weak instrument tests, overidentification tests, and multiple estimators.

```stata
ssc install ivreg2 ranktest

* Basic 2SLS with full diagnostics
ivreg2 wage (education = parent_educ sibling_educ) ///
    experience, first robust

* Key diagnostics from output
* e(widstat): Kleibergen-Paap F (always report)
* e(j): Hansen J statistic (p > 0.05 = valid instruments)
* e(endogp): Endogeneity test p-value

* LIML for weak instruments (less biased than 2SLS)
ivreg2 wage (education = parent_educ) experience, ///
    liml robust first

* Fuller's modified LIML (even more robust)
ivreg2 wage (education = parent_educ) experience, ///
    fuller(1) robust first
```

**Decision rule:** If Kleibergen-Paap F < 10, use LIML + Anderson-Rubin test instead of 2SLS.

---

## Causal Inference: Difference-in-Differences

### Modern DiD Packages (addressing staggered treatment)

TWFE can produce biased estimates with staggered adoption. Use at least one heterogeneity-robust estimator.

### csdid: Callaway-Sant'Anna

Estimates group-time treatment effects with flexible aggregation (by group, by time, event study).

```stata
ssc install csdid drdid

* Setup: treatment_year = 0/missing for never-treated
generate treatment_year = .
replace treatment_year = 2010 if state==1
replace treatment_year = 2015 if state==2

csdid outcome age education, ///
    ivar(state) time(year) gvar(treatment_year) ///
    method(dripw) notyet

* Aggregate to overall ATT
estat simple

* By treatment cohort
estat group

* Event study
estat event, window(-5 10)
csdid_plot, name(event_study, replace)

* Pre-trend test
estat pretrend, pre(5)
```

### did_multiplegt: Built-in Placebo Tests & Dynamics

Robust heterogeneity-consistent estimator with integrated placebo tests and dynamic effects.

```stata
ssc install did_multiplegt

did_multiplegt outcome unit_id time treatment, ///
    robust_dynamic dynamic(5) placebo(3) ///
    cluster(unit_id) breps(100)

* Results stored in e():
* e(effect): Instantaneous treatment effect
* e(dynamic_k): Effect k periods after treatment
* e(placebo_k): Placebo effect k periods before treatment
* e(p_jointplacebo): Joint test of all placebos
```

**Joint placebo p-value > 0.05:** pre-trends look parallel (good sign).

### Bacon Decomposition: DiD Diagnostics

Decomposes TWFE into component 2x2 comparisons to reveal problematic weights.

```stata
ssc install bacondecomp

bacondecomp outcome treatment, ddetail stub(bacon_)

* Comparison types:
* "Earlier T vs Later C": Early-treated vs not-yet-treated (GOOD)
* "Later T vs Earlier C": Late-treated vs already-treated (PROBLEMATIC)
* Weight > 20% on "Later T vs Earlier C" suggests heterogeneity

* Visualize
scatter bacon_estimate bacon_weight, ///
    yline(0) ylabel(, angle(0))
```

---

## Causal Inference: Regression Discontinuity, Matching, Synthetic Control

### rdrobust Suite: Regression Discontinuity

Robust RD estimation with bias-correction, optimal bandwidth selection, and manipulation tests.

```stata
net install rdrobust, ///
    from("https://raw.githubusercontent.com/rdpackages/rdrobust/master/stata")

* Main RD estimation (automatic bandwidth)
rdrobust outcome running_var, bwselect(cerrd)
* e(tau_bc): Bias-corrected RD estimate
* e(se_tau_rb): Robust SE

* RD plot with CI
rdplot outcome running_var, ci(95) shade

* Manipulation test (McCrary-type)
rddensity running_var
* p < 0.05: evidence of manipulation

* Fuzzy RD (treatment is discontinuous but not deterministic)
rdrobust outcome running_var, fuzzy(treatment_received)
```

**Key diagnostic:** Pre-treatment covariates should show no discontinuity at cutoff.

### psmatch2: Propensity Score Matching

Flexible matching algorithm with multiple estimators (NN, kernel, local linear) and diagnostics.

```stata
ssc install psmatch2

* Estimate propensity score
logit treatment age education income married, robust
predict pscore, pr

* Match with common support restriction
psmatch2 treatment, outcome(earnings) pscore(pscore) ///
    neighbor(1) caliper(0.05) common

* Check balance
pstest age education income married, both graph
* %bias before/after should drop significantly

* Robustness: try multiple algorithms
foreach algo in "neighbor(1)" "neighbor(3)" "kernel" "llr" {
    psmatch2 treatment, outcome(earnings) pscore(pscore) `algo' common
}
```

**Caliper rule of thumb:** 0.25 * SD(propensity score).

### synth: Synthetic Control Methods

Constructs weighted average of control units to match treated unit's pre-treatment trajectory.

```stata
ssc install synth synth_runner

* California Prop 99 example
synth cigsale ///
    cigsale(1988) cigsale(1980) cigsale(1975) ///
    retprice(1980(1)1988) beer(1980(1)1988) age15to24(1980(1)1988), ///
    trunit(3) trperiod(1989) ///
    mspeperiod(1970(1)1988) resultsperiod(1970(1)2000) ///
    keep(results.dta) replace

use results.dta, clear
generate gap = _Y_treated - _Y_synthetic

* Assessment: pre-treatment RMSPE should be small (< 5% of outcome mean)
* Post-treatment gap = treatment effect

* Placebo test: apply synth to each control unit (should see no effect)
synth_runner cigsale cigsale(1988) cigsale(1980) retprice(1980(1)1988), ///
    trunit(1/39) trperiod(1989) gen_vars
```

---

## Dynamic Panel Data

### xtabond2: Arellano-Bond/Blundell-Bond GMM

Estimates dynamic panel models with lagged dependent variable (avoids Nickell bias from FE).

```stata
ssc install xtabond2 ranktest

* System GMM (recommended default)
xtabond2 y L.y x1 x2, ///
    gmm(L.y, lag(1 2) collapse) ///    // Instrument for lagged DV
    iv(x1 x2) ///                       // x1, x2 are exogenous
    robust twostep orthogonal small     // Windmeijer correction for two-step SEs

* Key diagnostics -- must check:
* e(ar2p): Arellano-Bond AR(2) test p > 0.05 (no serial correlation)
* e(hansenp): Hansen overid test p in [0.10, 0.90] (valid instruments)
* e(j) < e(N_g): Instruments < groups (not overfitted)

* Treated variables as endogenous
xtabond2 y L.y x1 x2, ///
    gmm(L.y, lag(1 2) collapse) ///
    gmm(x2, lag(1 1)) ///              // x2 is endogenous/predetermined
    iv(x1) ///
    robust twostep small
```

**Gotcha:** Without `collapse`, instruments proliferate (T*(T-1)/2 per variable). Use `collapse` always for system GMM. Aim for j < N_g.

---

## Visualization & Binning

### coefplot: Coefficient Plots

Visualize regression coefficients with confidence intervals and multiple models.

```stata
ssc install coefplot

* See Graphics section (references/output.md) for details and examples
regress price mpg weight foreign
coefplot, drop(_cons) xline(0) ///
    title("Price Determinants") ///
    msymbol(D) mcolor(navy) ///
    scheme(plotplain)
```

### binsreg: Modern Binscatter with Rigorous Inference

Optimal binning, pointwise/uniform confidence bands, hypothesis testing (linearity, monotonicity).

```stata
net install binsreg, ///
    from("https://raw.githubusercontent.com/nppackages/binsreg/master/stata")

* Basic binscatter (automatic bin selection)
binsreg wage education

* With confidence bands and polynomial overlay
binsreg wage education experience age, ///
    nbins(20) dots(0,0) line(3,3) cb(3,3) polyreg(2) ///
    title("Wage-Education Relationship")

* Test linearity
binstest wage education experience age, testmodelpoly(1)

* Test monotonicity (increasing)
binstest wage education experience age, testshaper(0) deriv(1)
```

**Key advantage over old `binscatter`:** Proper covariate adjustment (not residualization), formal inference.

### nprobust: Nonparametric Kernel Estimation with Robust CI

Local polynomial regression and kernel density with bias-corrected inference.

```stata
net install nprobust, ///
    from("https://raw.githubusercontent.com/nppackages/nprobust/master/stata")

* Local linear regression with robust CIs
lprobust wage age, genvars plot ///
    graph_options(title("Age-Wage Profile"))

* Kernel density estimation
kdrobust price, plot

* Derivative estimation (marginal effects)
lprobust mpg weight, deriv(1) genvars  // First derivative/slope
```

---

## Data Manipulation & Tools

### gtools: Fast Data Operations

Drop-in replacements for `collapse`, `egen`, `reshape`, etc. using C plugins. 5-20x speedup on large data.

```stata
ssc install gtools ftools

* Fast aggregation
gcollapse (mean) avg_wage=wage (sd) sd_wage=wage ///
    (count) n=wage, by(firm_id year)

* Fast egen operations
gegen mean_wage = mean(wage), by(firm_id)
gegen rank_wage = rank(wage), by(firm_id)
gegen tag_first = tag(firm_id)  // Tag first obs in each group

* Fast reshape
greshape long income, i(id) j(year)
```

**Rule of thumb:** Use gtools for N > 100K, repeated operations, or panel grouping.

### rangestat: Moving Window Statistics

Computes statistics within arbitrary ranges -- rolling means, event windows, lagged/lead averages.

```stata
ssc install rangestat

tsset firm_id year

* 7-day backward-looking moving average
rangestat (mean) ma7=price, interval(date -6 0) by(firm_id)

* Event windows: pre vs post
rangestat (mean) pre_avg=outcome, interval(year -5 -1) by(firm_id)
rangestat (mean) post_avg=outcome, interval(year 0 5) by(firm_id)
```

### distinct / unique: Uniqueness Checks

```stata
ssc install distinct unique

* Count distinct values
distinct state              // How many unique states?
distinct firm_id year, joint // Unique firm-year combinations?

* Tag observations
unique firm_id year, gen(tag_first)  // Tag first obs per firm-year
list if tag_first==1               // Show one observation per group
```

### winsor2: Winsorizing (Cap Extremes)

Replaces extreme values at percentiles rather than deleting.

```stata
ssc install winsor2

* Symmetric: cap at 1st and 99th percentiles
winsor2 price, cuts(1 99) suffix(_w)

* By-group (within industry)
winsor2 roa leverage, cuts(1 99) suffix(_w) by(industry)

* Common cutoffs: 1%/99% (conservative), 5%/95% (aggressive)
```

---

## Diagnostics & Testing

### bacondecomp: DiD Decomposition

See Causal Inference section above.

### xttest3: Panel Heteroskedasticity

Tests whether error variance differs across panel units.

```stata
ssc install xttest3

* Run after xtreg FE estimation
xtreg y x1 x2, fe
xttest3

* If p < 0.05: heteroskedasticity detected
* Remedy: Use vce(robust) or vce(cluster id)
xtreg y x1 x2, fe vce(robust)
```

### xtserial: Serial Correlation in Panels

Wooldridge test for first-order autocorrelation in panel data.

```stata
ssc install xtserial

xtserial y x1 x2

* If p < 0.05: serial correlation present
* Remedy: Cluster SE by unit id, or use dynamic panel GMM
xtreg y x1 x2, fe vce(cluster id)
xtabond2 y L.y x1 x2, gmm(L.y) iv(x1 x2) robust
```

### xtcsd: Cross-Sectional Dependence

Tests for common shocks/spatial correlation across panel units.

```stata
ssc install xtcsd

xtreg y x1 x2, fe
xtcsd, pesaran abs

* If p < 0.05: CSD detected
* Remedy: Driscoll-Kraay SE (handles both serial + CSD)
ssc install xtscc
xtscc y x1 x2, fe lag(2)
```

### collin / vif: Multicollinearity

```stata
regress y x1 x2 x3
vif                         // Built-in VIF

ssc install collin
collin x1 x2 x3             // Enhanced: condition number, variance decomp
```

**VIF rule of thumb:** < 5 (OK), 5-10 (caution), > 10 (problematic).

---

## Installation Quick Reference

### Essential Packages by Research Area

**Econometrics Core:**
```stata
ssc install reghdfe ftools        // High-dim FE
ssc install ivreg2 ranktest       // IV diagnostics
ssc install xtabond2               // Dynamic panels
```

**Causal Inference (DiD):**
```stata
ssc install bacondecomp            // DiD decomposition
net install csdid, from("https://friosavila.github.io/playingwithstata/drdid")
ssc install did_multiplegt         // DID with placebos
ssc install did_imputation         // Borusyak et al.
```

**Causal Inference (RD):**
```stata
net install rdrobust, from("https://raw.githubusercontent.com/rdpackages/rdrobust/master/stata")
net install rddensity, from("https://raw.githubusercontent.com/rdpackages/rddensity/master/stata")
```

**Visualization:**
```stata
ssc install coefplot               // Coefficient plots
net install binsreg, from("https://raw.githubusercontent.com/nppackages/binsreg/master/stata")
net install nprobust, from("https://raw.githubusercontent.com/nppackages/nprobust/master/stata")
```

**Data Tools:**
```stata
ssc install gtools                 // Fast operations
ssc install rangestat              // Moving windows
ssc install winsor2                // Winsorizing
```

**Diagnostics:**
```stata
ssc install xttest3 xtserial xtcsd // Panel diagnostics
ssc install collin                 // Multicollinearity
```

**Output Tables (see references/output.md):**
```stata
ssc install estout                 // esttab, eststo, estpost, estadd
ssc install outreg2                // Quick tables
ssc install asdoc                  // Minimal-syntax export
ssc install tabout                 // Cross-tabulations
```

---

## Reproducibility & Version Control

```stata
* Document installed packages in do-file
version 17.0
mata: mata clear

* At top of project
local packages reghdfe ftools estout coefplot ivreg2

foreach pkg of local packages {
    capture which `pkg'
    if _rc {
        ssc install `pkg', replace
    }
}

* Save package versions
log using package_log.log, text replace
ado dir
log close
```
