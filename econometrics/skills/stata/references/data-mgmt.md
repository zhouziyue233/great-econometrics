# Stata Data Management & Specialized Models Reference

## Table of Contents
1. Core Data Operations
2. String Functions
3. Date & Time Functions
4. Mathematical Functions
5. Descriptive Statistics
6. Missing Data Handling
7. Mata Data Access & Matrix Operations
8. Specialized Models (LDV, Survival, SEM, Spatial, ML)

---

## 1. Core Data Operations

### Generate and Replace

```stata
generate len_ft = length / 12
generate log_price = log(price)
generate high_price = 1 if price > 10000

replace price = price * 1.1
replace mpg = . if mpg < 0
```

### Recode and Egen

```stata
recode mpg (min/18=1) (19/23=2) (24/max=3)
recode age (0/17=1 "Child") (18/64=2 "Adult") (65/max=3 "Senior"), gen(age_group)

egen avg_score = rowmean(test1 test2 test3)
egen max_score = rowmax(test1 test2 test3)
bysort country: egen avg_income = mean(income)
eigen region_id = group(region)
```

### Merge and Append

```stata
// One-to-one merge
merge 1:1 person_id using income_data
tab _merge
drop if _merge != 3
drop _merge

// Many-to-one merge
merge m:1 school_id using school_characteristics, keep(3) nogenerate

// Append datasets
append using dataset2
use dataset1, clear
append using dataset2 dataset3
```

**Gotcha:** Always `tab _merge` before `assert _merge == 3` — if assert fails, tab output shows why the merge failed.

### Reshape (Wide/Long)

```stata
reshape long income, i(famid) j(year)
reshape wide income, i(person_id) j(year)
reshape long ht wt, i(famid birth) j(age)
```

### Collapse

```stata
// By-group statistics (preserves original data)
bysort foreign: egen mean_price = mean(price)

// Full collapse (destructive - use preserve first)
preserve
collapse (mean) avg_wage=wage (sd) sd_wage=wage, by(industry)
save groupstats, replace
restore
merge m:1 industry using groupstats
```

### Drop, Keep, and Duplicates

```stata
drop temp_var var1-var5
keep id name age income*
drop if age < 18
duplicates drop id year, force
bysort id year (revenue): keep if _n == _N
```

### Rename and Label

```stata
rename rep78 repair_record
rename (v1 v2 v3) (score1 score2 score3)
rename *, lower
label variable income "Annual household income"
label define yesno 0 "No" 1 "Yes"
label values married yesno
```

### Encode and Decode

```stata
encode gender_str, gen(gender)
label define race_lbl 1 "White" 2 "Black" 3 "Asian"
encode race_str, gen(race) label(race_lbl)
decode gender, gen(gender_str)
```

---

## 2. String Functions

| Function | Purpose | Example |
|----------|---------|---------|
| `strlen(s)` | Length in bytes | `strlen(name) == 5` |
| `substr(s, n1, n2)` | Extract substring | `substr(text, 1, 5)` |
| `strpos(s1, s2)` | Find first occurrence (0 if not found) | `strpos(email, "@") + 1` |
| `strrpos(s1, s2)` | Find last occurrence | `strrpos(name, " ")` |
| `subinstr(s1, s2, s3, n)` | Replace substring | `subinstr(text, "$", "", .)` |
| `strupper(s)` / `strlower(s)` / `strproper(s)` | Case conversion | `strupper(state)` |
| `trim(s)` / `ltrim(s)` / `rtrim(s)` | Remove spaces | `trim(stritrim(text))` |

### Regular Expressions

```stata
gen has_number = regexm(text, "[0-9]")
gen valid_email = regexm(email, "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$")
gen clean = regexr(text, "[0-9]+", "")              // Replace first match
gen has_code = regexm(id, "^([A-Z]{3})([0-9]{4})")
gen code = regexs(1) if has_code                    // Extract captured group
```

---

## 3. Date & Time Functions

### Creating and Converting Dates

| Function | Purpose |
|----------|---------|
| `mdy(m,d,y)` | Month, day, year → daily date |
| `ym(y,m)` | Year, month → monthly date |
| `yq(y,q)` | Year, quarter → quarterly date |
| `mofd(d)` | Daily → monthly |
| `qofd(d)` | Daily → quarterly |
| `dofm(m)` | Monthly → daily (first day) |
| `date(string, "YMD")` | Parse string dates |
| `today()` | Current date |
| `now()` | Current datetime |

```stata
gen date_var = mdy(month, day, year)
format date_var %td

gen monthly = mofd(date_var)
format monthly %tm

gen date1 = date("2023-02-15", "YMD")
gen date2 = date("15/02/2023", "DMY")
```

### Date Arithmetic

```stata
gen days_elapsed = date2 - date1
gen end_date = start_date + 30
gen duration_hours = (end_time - start_time) / (60 * 60 * 1000)
gen years_diff = datediff(date1, date2, "year")           // Stata 17+
```

### Extracting Components

```stata
gen year = year(date_var)
gen month = month(date_var)
gen day = day(date_var)
gen dow = dow(date_var)        // Day of week (0=Sun to 6=Sat)
gen age_years = floor((today() - birthdate) / 365.25)
gen fiscal_year = year(date_var) + (month(date_var) >= 4)
```

---

## 4. Mathematical Functions

| Function | Purpose |
|----------|---------|
| `abs(x)` | Absolute value |
| `sqrt(x)` | Square root |
| `exp(x)` / `ln(x)` / `log10(x)` | Exponential / natural log / base-10 |
| `max(x1, x2, ...)` / `min(x1, x2, ...)` | Maximum / minimum |
| `round(x)` / `floor(x)` / `ceil(x)` | Rounding functions |
| `mod(x, y)` | Modulus (remainder) |

### Random Numbers

```stata
set seed 12345
generate u = runiform()              // Uniform [0,1)
generate z = rnormal()               // Standard normal
generate z2 = rnormal(75, 10)        // N(75, sd=10)
generate b = rbinomial(n, p)         // Binomial
```

**Gotcha:** `rnormal(m, s)` takes **standard deviation**, not variance. For N(0, 4) errors, use `rnormal(0, 2)` since sd = sqrt(4) = 2.

### Probability and Distributions

```stata
display normal(z)                      // CDF: P(Z <= z)
display normalden(z)                   // PDF
display invnormal(0.975)               // z-score for 97.5th percentile
display ttail(df, t)                   // Upper tail P(T > t)
display chi2tail(df, x)                // Upper tail P(Chi2 > x)
```

---

## 5. Descriptive Statistics

### Basic Commands

```stata
summarize price mpg weight              // N, mean, sd, min, max
summarize price, detail                 // Add percentiles, skewness, kurtosis
tabstat price mpg, by(foreign) statistics(mean sd median)
tabulate foreign                        // Frequencies and percentages
tabulate foreign rep78, chi2            // Cross-tab with chi-squared
```

### Correlation and Association

```stata
correlate price mpg weight
pwcorr price mpg weight, sig star(0.05)     // Pairwise with p-values
```

### Creating Summary Variables

```stata
summarize price, meanonly
local mean_price = r(mean)
tabstat income, by(group) statistics(p25 p50 p75) save
matrix summary = r(Stat1)
```

---

## 6. Missing Data Handling

### Identifying and Understanding Missingness

```stata
misstable summarize
misstable patterns wage age grade hours, frequency
generate wage_missing = missing(wage)
tabstat age grade hours, by(wage_missing) statistics(mean sd)
```

### Multiple Imputation Setup and Imputation

```stata
mi set mlong
mi register imputed wage age grade
mi register regular hours married
mi describe

// Simple imputation
mi impute pmm wage age grade hours i.race married, add(20) rseed(12345) knn(5)

// Chained equations (multiple variables)
mi impute chained ///
    (pmm, knn(5)) wage hours ///
    (truncreg, ll(0)) tenure ///
    (logit) married ///
    = age i.race, add(25) rseed(12345) burnin(10)
```

### Analyzing with Missing Data

```stata
mi estimate: regress wage age grade hours
mi estimate, vartable: regress wage age grade hours
mi estimate, post: regress wage age grade
margins, dydx(*)
mi test age
```

**Choose imputations: M = % missing information; M >= 20 for estimation, M >= 100 for hypothesis tests.**

---

## 7. Mata Data Access & Matrix Operations

### Data Access

```mata
// Copy data to matrix
X = st_data(., ("price", "mpg", "weight"))
foreign_data = st_data(., ("price", "mpg"), "foreign")

// Create view (no copy, read-only)
st_view(X=., ., ("price", "mpg", "weight"))

// Write back to Stata
mpg_sq = st_data(., "mpg") :^ 2
st_store(., "mpg_squared", mpg_sq)

// Critical: filter missings before aggregating
clean = select(data, missing(data) :== 0)
result = mean(clean)
```

### Matrix Operations

```mata
A = (1, 2, 3 \ 4, 5, 6 \ 7, 8, 9)   // Create matrix
I = I(3)                              // Identity
B = A'                                // Transpose
C = inv(A)                            // Inverse (use invsym for symmetric)
d = det(A)                            // Determinant
t = trace(A)                          // Trace
symeigensystem(A, V=., L=.)          // Eigendecomposition
```

### Efficient Patterns

```mata
// Bad: repeated lookups
for (i=1; i<=100; i++) result[i] = mean(st_data(., "price"))

// Good: lookup once, reuse
price_data = st_data(., st_varindex("price"))
for (i=1; i<=100; i++) result[i] = mean(price_data)
```

---

## 8. Specialized Models (LDV, Survival, SEM, Spatial, ML)

### Limited Dependent Variable Models

```stata
// Binary
logit union age grade south
logistic union age grade south               // Odds ratios
predict pr, pr                               // Predicted probability

// Ordered
ologit health age grade i.race, or

// Multinomial (unordered)
mlogit occ_cat age grade i.race, baseoutcome(1) rrr

// Count
poisson doctor_visits age grade i.married, irr
nbreg doctor_visits age grade i.married, irr // Overdispersion

// Censored/truncated
tobit hours age grade south, ll(0)
margins, dydx(*) predict(ystar(0,.))

// Always use margins for interpretation in LDV models
margins, dydx(*) at(age=30)
```

### Survival Analysis

```stata
stset time, failure(event)           // Setup
stset time, failure(event) id(id)    // Panel data
stsum                                 // Summary
sts graph, by(group)                  // Kaplan-Meier
sts test group                         // Log-rank test
stcox treatment age                   // Cox model
estat phtest                          // Test proportional hazards
streg treatment age, dist(weibull)    // Parametric model
```

### SEM and Factor Analysis

```stata
// Confirmatory Factor Analysis
sem (Ability -> x1 x2 x3 x4), standardized
estat gof, stats(all)
estat mindices, minchi(3.84)

// Full SEM
sem (F1 -> x1 x2 x3) (F2 -> x4 x5 x6) (Y <- F1 F2), standardized
estat teffects

// Exploratory Factor Analysis
factor varlist, factors(3) pcf
rotate, varimax

// Reliability
alpha item1-item10
alpha item1-item10, item detail gen(scale)
```

### Spatial Analysis

```stata
spset county_id                       // Setup spatial data
spmatrix create contiguity W, queen normalize(row)    // Weights
estat moran income, weight(W)         // Test autocorrelation
spregress income education, dvarlag(W)   // SAR model
estat impact education                // Spillover effects
spmap income using shp, id(_ID) clmethod(quantile) clnumber(5)
```

### Machine Learning

```stata
// Lasso
lasso linear price mpg weight length headroom, selection(cv)
lassocoef, display(nonzero)
cvplot
predict price_pred

// Cross-validation
generate fold = mod(_n, 5) + 1
forvalues k = 1/5 {
    quietly regress price mpg weight if fold != `k'
    predict temp if fold == `k'
}

// Elastic net
elasticnet linear price mpg weight, alpha(0.5) selection(cv)

// Lasso goodness of fit
lassogof, postselection
```

---

## Best Practices Summary

1. **Merge/append:** Always `tab _merge` and check identify before asserting
2. **Reshape:** Use `fillin` to create missing combinations; check for duplicates first
3. **Missing data:** Filter before Mata aggregation; use `mi estimate` for inference
4. **Dates:** Always `format` after creating; use `tsset` for time series
5. **Strings:** Use `subinstr` or regex for cleaning; `encode` after standardizing
6. **Limited DV:** Never interpret raw coefficients; always use `margins`
7. **Spatial:** Row-standardize weights; test residuals before modeling
8. **ML:** Always split data before any analysis; evaluate on test set only
