# Panel Data — Advanced Reference

**Scope**: Advanced estimators not covered in the main `panel-data/SKILL.md`.
The main skill covers: pooled OLS, FE, RE, Hausman test, Mundlak/CRE, Arellano-Bond GMM, Driscoll-Kraay SE, two-way clustering.
This file covers: **first-difference (FD) estimator** and **panel models with limited dependent variables** (conditional logit, Poisson FE, Cox stratified hazard).

---

## 1. First-Difference (FD) Estimator

### When to Use FD vs FE

| Criterion | Prefer FD | Prefer FE (within) |
|---|---|---|
| T = 2 | ✓ FD = FE when T = 2 | — |
| Errors follow random walk (unit root) | ✓ FD eliminates unit-root component | FE SE are invalid |
| T is small and idiosyncratic errors are serially uncorrelated | — | ✓ FE is more efficient |
| T is large with MA(1) errors | ✓ FD is more efficient | — |

**Rule of thumb**: Use FD when T is small (T ≤ 5) and you suspect the idiosyncratic error has a random walk or when the Wooldridge test rejects no serial correlation after FE. Use FE otherwise.

### Estimating the FD Model

```python
# Python (linearmodels) — First Difference
from linearmodels.panel import FirstDifferenceOLS
import pandas as pd

df = df.set_index(["entity_id", "time_var"])

fd_model  = FirstDifferenceOLS(df["y"], df[["x1", "x2"]])
fd_result = fd_model.fit(cov_type="robust")    # HC-robust
# For clustered SE:
fd_result = fd_model.fit(cov_type="clustered", cluster_entity=True)
print(fd_result.summary)
```

```r
# R (plm) — First Difference
library(plm)
panel_df <- pdata.frame(df, index = c("entity_id", "time_var"))

fd_model <- plm(y ~ x1 + x2, data = panel_df, model = "fd")
summary(fd_model)

# Clustered SE at entity level
library(lmtest); library(sandwich)
coeftest(fd_model, vcov = vcovHC(fd_model, cluster = "group"))
```

```stata
* Stata — First Difference (manual via D. prefix)
xtset entity_id time_var

reg D.y D.x1 D.x2, cluster(entity_id)
* D. prefix applies first difference to each variable.
* Note: FD drops one period per entity; intercept suppressed.

* Alternative: xtreg with fd option (Stata 13+)
xtreg y x1 x2, fd cluster(entity_id)
```

### Serial Correlation Diagnostic for FD Residuals

After first differencing, no serial correlation in levels implies MA(1) correlation in the first-differenced residuals. If AR(2) or higher is present in ΔeᵢΤ, OLS on the first-differenced equation is still consistent but standard errors need adjustment.

```stata
* Wooldridge test for serial correlation in FD residuals
* H₀: no serial correlation in the FD residuals (MA(1) structure expected; test for MA(2)+)
ssc install xtserial
xtserial D.y D.x1 D.x2
* p < 0.05 → serial correlation in Δeᵢₜ beyond MA(1) → use Newey-West or GLS
```

```r
# R — Wooldridge-type serial correlation test
library(plm)
pbgtest(fd_model, order = 2)   # Breusch-Godfrey for panel
# p < 0.05 → autocorrelation present; apply Newey-West lags
```

### Interpretation Note

FD estimates identify the effect of **changes** in X on changes in Y. If X is binary (treatment switches on/off), the FD coefficient captures the average effect of switching from 0 to 1, net of any unit-level time trend. Time-invariant variables are automatically differenced out (no separate treatment needed).

---

## 2. Panel Models with Limited Dependent Variables

Standard FE with binary or count outcomes suffers from the **incidental parameters problem** when N is large and T is small: the entity fixed effects αᵢ cannot be estimated consistently, which contaminates the structural parameters.

| Outcome type | Recommended model | Incidental parameters? |
|---|---|---|
| Continuous | FE (within) | No (FE is unbiased) |
| Binary | Conditional logit (Chamberlain 1980) | Eliminated via sufficient statistics |
| Count | Poisson FE (PPML) | No (Poisson FE is consistent) |
| Duration / Time-to-event | Cox PH with strata | Eliminated via partial likelihood |

> **Probit with FE is inconsistent**: There is no sufficient statistic to condition out αᵢ in the probit model. Use conditional logit (binary) or reframe as a linear probability model (LPM) with FE as a robustness check.

---

### 2a. Conditional Logit (Binary FE — Chamberlain 1980)

**Identification**: Only units that **switch** treatment status contribute to the likelihood. Units with y = 0 or y = 1 in all periods are dropped (they provide no within-unit variation).

```python
# Python (statsmodels) — Conditional Logit
import statsmodels.formula.api as smf

# Conditional logit via fixed-effects logistic regression
# Using bife package (best Python implementation):
# pip install bife
import bife

model = bife.bife(df, "y", ["x1", "x2"], "entity_id")
result = model.fit()
print(result.summary())
```

```r
# R — Conditional Logit (survival::clogit or bife)
library(survival)

# clogit: each entity_id forms a stratum
clogit_model <- clogit(y ~ x1 + x2 + strata(entity_id), data = df)
summary(clogit_model)

# Alternative: bife package (faster for large panels)
library(bife)
bife_model <- bife(y ~ x1 + x2 | entity_id, data = df)
summary(bife_model)
# APEs (average partial effects):
bias_corr <- bias_corr(bife_model)
get_APEs(bias_corr)
```

```stata
* Stata — Conditional Logit
xtset entity_id time_var
xtlogit y x1 x2, fe           // Chamberlain conditional logit
* Marginal effects not directly available from xtlogit, fe
* Use margins after logit with dummies if needed (computationally expensive)

* Alternative: clogit
clogit y x1 x2, group(entity_id)
```

**Reporting**: Report log-odds coefficients and convert to Average Partial Effects (APEs) using `margins` (Stata) or `get_APEs()` (R `bife`). Log-odds are not directly interpretable as effect sizes.

---

### 2b. Poisson FE (PPML — Count and Non-negative Outcomes)

Poisson pseudo-maximum likelihood (PPML) with unit fixed effects is **consistent** even when the true DGP is not Poisson, as long as E[yᵢₜ | Xᵢₜ, αᵢ] = exp(Xᵢₜβ + αᵢ). This makes it the preferred estimator for trade flows, patent counts, export values, and any non-negative outcome.

```python
# Python — Poisson FE (pyfixest)
import pyfixest as pf

# feols formula with Poisson family:
poisson_fe = pf.fepois(
    "y ~ x1 + x2 | entity_id + time_var",
    data=df,
    vcov={"CRV1": "entity_id"}   # cluster by entity
)
poisson_fe.summary()

# Interpret coefficients as semi-elasticities:
# β = 0.10 → x1 increasing by 1 unit multiplies E[y] by exp(0.10) ≈ 1.105 (+10.5%)
```

```r
# R — Poisson FE (fixest)
library(fixest)

ppml_model <- fepois(y ~ x1 + x2 | entity_id + time_var,
                     data = df,
                     vcov = ~entity_id)
summary(ppml_model)

# Coefficient interpretation: exp(β) - 1 = % change in E[y]
exp(coef(ppml_model)["x1"]) - 1
```

```stata
* Stata — Poisson FE (ppmlhdfe preferred over xtpoisson, fe)
ssc install ppmlhdfe

ppmlhdfe y x1 x2, absorb(entity_id time_var) cluster(entity_id)

* xtpoisson alternative (slower for large panels):
xtset entity_id time_var
xtpoisson y x1 x2 i.time_var, fe vce(robust)
```

**Overdispersion check**: PPML is robust to overdispersion (Var[y] > E[y]), but if overdispersion is severe and there are many zeros, consider **Negative Binomial FE** or a **zero-inflated** model.

```stata
* Stata — overdispersion test after xtpoisson
* Compare xtpoisson (Poisson) vs xtnbreg (Negative Binomial)
xtset entity_id time_var
xtnbreg y x1 x2, fe
* If NB alpha is significantly > 0 → overdispersion matters
```

---

### 2c. Cox Proportional Hazard with Stratification (Survival Panel)

Use when the outcome is **time-to-event** (duration until job loss, firm exit, policy adoption) and unobserved unit heterogeneity may confound the baseline hazard. Stratifying by entity eliminates the entity-specific baseline hazard h₀ᵢ(t) non-parametrically, analogous to FE.

**Model**: h(t | Xᵢₜ, αᵢ) = h₀ᵢ(t) · exp(Xᵢₜβ), where h₀ᵢ(t) is stratified (unrestricted) by unit.

```python
# Python — Cox PH with strata (lifelines)
# pip install lifelines
from lifelines import CoxPHFitter
import pandas as pd

# Data must be in long format with: entity_id, duration, event (0/1), x1, x2
cph = CoxPHFitter()
cph.fit(
    df,
    duration_col  = "duration",
    event_col     = "event",
    strata        = ["entity_id"],   # stratify = entity-specific baseline
    formula       = "x1 + x2",
    robust        = True             # sandwich SE
)
cph.print_summary()
# exp(coef) = hazard ratio
```

```r
# R — Cox PH stratified by entity (survival package)
library(survival)

cox_model <- coxph(
  Surv(duration, event) ~ x1 + x2 + strata(entity_id),
  data    = df,
  cluster = df$entity_id,   # clustered SE
  ties    = "efron"         # Efron approximation for tied event times
)
summary(cox_model)
# Hazard ratio: exp(coef); 95% CI: exp(confint(cox_model))

# Schoenfeld residuals test for proportional hazards assumption:
cox.zph(cox_model)
# p < 0.05 for a covariate → PH assumption violated for that variable
```

```stata
* Stata — Stratified Cox PH
stset duration, failure(event==1) id(entity_id)

* Stratify on entity_id to absorb entity-specific baseline hazard:
stcox x1 x2, strata(entity_id) cluster(entity_id) efron

* Test proportional hazards assumption (Schoenfeld residuals):
estat phtest, detail
* p < 0.05 → PH violated; consider time-varying coefficients or parametric model
```

**Discrete-time alternative**: If the time variable is coarse (annual, quarterly), a **cloglog** or **logit** panel model on the discrete-time hazard is preferred:

```stata
* Stata — Discrete-time hazard model (complementary log-log)
xtcloglog event x1 x2 i.period, re    // RE cloglog (panel)
* FE version: use clogit with time dummies and entity-period structure
```

**Interpretation**: Exponentiated coefficients are **hazard ratios**. HR > 1 means covariate increases the instantaneous risk of the event; HR < 1 means it reduces it. Unlike OLS, there is no direct R² analogue — use Harrell's C-statistic or AIC for model comparison.

---

## Checklist: Choosing Among LDV Panel Estimators

```
Outcome is binary?
  → Conditional logit (clogit / bife)
  → Do NOT use probit FE (inconsistent)
  → LPM with FE is acceptable as robustness check

Outcome is count / non-negative continuous?
  → Poisson FE (PPML) — robust to distributional misspecification
  → If severe overdispersion + many zeros → Negative Binomial FE

Outcome is time-to-event?
  → Cox PH with strata(entity_id) — non-parametric baseline
  → If time is discrete → discrete-time cloglog or logit

Outcome is continuous but right-censored (e.g., wages, firm sales)?
  → Tobit FE (xttobit in Stata)
  → Note: Tobit FE also has incidental parameters problem when T is small
```

---

## Related References

- `panel-data/SKILL.md` — main skill: pooled OLS, FE/RE, Hausman, Mundlak/CRE, Arellano-Bond, Driscoll-Kraay SE
- Chamberlain (1980) "Analysis of Covariance with Qualitative Data" — conditional logit derivation
- Santos Silva & Tenreyro (2006) "The Log of Gravity" — PPML consistency argument
- Cox (1972) "Regression Models and Life Tables" — partial likelihood and stratification
