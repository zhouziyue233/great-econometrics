# Stata Basics and Programming Guide

## Contents

1. [Getting Started](#getting-started)
2. [Command Syntax and Data Types](#command-syntax-and-data-types)
3. [Variables and Operators](#variables-and-operators)
4. [Data Import and Export](#data-import-and-export)
5. [Programming Fundamentals](#programming-fundamentals)
6. [Advanced Programming](#advanced-programming)
7. [Mata Introduction](#mata-introduction)
8. [Mata Programming](#mata-programming)
9. [External Tools Integration](#external-tools-integration)
10. [Workflow Best Practices](#workflow-best-practices)

---

## Getting Started

### Basic Command Syntax

```stata
[by varlist:] command [varlist] [=exp] [if exp] [in range] [weight] [, options]
```

#### Key Syntax Rules

1. **Commands are case-sensitive for variable names** but not for command names
2. **Abbreviations** - Most commands can be abbreviated: `sum age` = `summarize age`
3. **Variable lists** can use wildcards and ranges: `describe age-income` or `describe age*`
4. **Comments**: `* line comment` or `// inline comment` or `/* multi-line */`
5. **Line continuation**: `///`

#### Qualifiers

```stata
summarize income if age > 30
list in 1/10              // First 10 observations
list in -10/l             // Last 10 observations
summarize income in 20/50 // Observations 20-50
```

### Essential Commands

```stata
* Data input/output
use "mydata.dta", clear
use age income using "mydata", clear  // Load specific variables
save "mydata.dta", replace

* Data exploration
describe                  // All variables
describe, short           // Brief summary
summarize                 // All numeric variables
summarize age, detail     // Detailed stats (percentiles, etc.)
list in 1/10              // First 10 observations
browse age income         // Data browser (read-only)
codebook, compact         // Compact variable info
tabulate gender           // One-way frequency table
tab gender married, row col   // Two-way cross-tabulation

* Data modification
generate age_squared = age^2
generate high_income = (income > 50000)
replace income = income * 1.03
replace age = . if age < 0          // Set invalid to missing
drop age                  // Drop a variable
keep age income education // Keep only these variables
sort age
gsort -age                // Descending order

* System/file management
pwd                       // Print working directory
cd "/path/to/project"     // Change directory
log using "analysis.log", replace
log close
```

### Working Directory Management

**Best Practice**: Use global macros for paths

```stata
global projdir "/path/to/project"
cd "$projdir"
use "$projdir/data/mydata.dta"

* Recommended structure:
* MyProject/
* ├── data/           (raw data)
* ├── dofiles/        (analysis scripts)
* ├── results/        (output datasets)
* ├── logs/           (log files)
* └── graphs/         (figures)
```

### Getting Help

```stata
help summarize               // Help for specific command
search "panel data"          // Search for topics
findit "robust regression"   // Search including user-written programs
ssc install commandname      // Install user-written commands
```

---

## Command Syntax and Data Types

### Numeric Storage Types

| Type | Bytes | Range | Use |
|------|-------|-------|-----|
| `byte` | 1 | -127 to 100 | Small integers |
| `int` | 2 | -32,767 to 32,740 | Medium integers |
| `long` | 4 | -2,147M to 2,147M | Large integers |
| `float` | 4 | ±10^38 | Floating point (6-7 digit precision) |
| `double` | 8 | ±10^307 | High precision floating point |

Only `float` and `double` hold decimals.

```stata
describe          // View current variable types
recast byte age   // Change storage type
recast double price
```

### String Variables

- `str1` to `str2045`: Fixed-length strings
- `strL`: Variable-length strings (up to 2 billion characters)

**Best Practice**: Categorical variables should be numeric with value labels, not strings.

```stata
gen str20 name = "John Doe"
recast strL description
```

### Date and Time Variables

Stored as numbers representing units since January 1, 1960:

| Type | Format | Unit |
|------|--------|------|
| Daily | `%td` | Days |
| Weekly | `%tw` | Weeks |
| Monthly | `%tm` | Months |
| Quarterly | `%tq` | Quarters |
| Yearly | `%ty` | Years |
| Datetime | `%tc` | Milliseconds |

```stata
gen date_var = date(string_date, "DMY")
format date_var %td

gen year = year(date_var)
gen month = month(date_var)

gen double datetime_var = clock(string_datetime, "DMYhms")
format datetime_var %tc
```

### Variable Labels and Value Labels

```stata
label variable age "Age in years"
label define yesno 0 "No" 1 "Yes"
label values married yesno
```

---

## Variables and Operators

### Variable Naming Rules

- Up to 32 characters
- Must start with letter or underscore
- Letters, numbers, underscores only
- **Case-sensitive**: `Age` and `age` are different
- Cannot use reserved words (`if`, `in`, `using`)
- Use lowercase with underscores for readability

### Operators

#### Arithmetic
| Operator | Description |
|----------|-------------|
| `+`, `-`, `*`, `/` | Basic arithmetic |
| `^` | Exponentiation |
| `-` (prefix) | Negation |

#### Relational (return 1 if true, 0 if false)
| Operator | Description |
|----------|-------------|
| `==` | Equal to |
| `!=` or `~=` | Not equal to |
| `>`, `>=`, `<`, `<=` | Comparisons |

#### Logical
| Operator | Description |
|----------|-------------|
| `&` | AND |
| `|` | OR |
| `!` or `~` | NOT |

#### Operator Precedence (highest to lowest)
1. Parentheses `( )`
2. Negation: `-`, `!`, `~`
3. `^`
4. `*`, `/`
5. `+`, `-`
6. Relational: `>`, `>=`, `<`, `<=`, `==`, `!=`
7. `&`
8. `|`

**Key**: `&` binds tighter than `|`. Use parentheses for clarity.

```stata
gen flag = (age > 18 & income > 50000) | (student == 1)
gen fullname = firstname + " " + lastname  // String concatenation
```

### Missing Values

Stata has 27 missing values: `.`, `.a` through `.z`

#### CRITICAL: Missing as Positive Infinity

**Missing values are treated as larger than any number.**

```stata
* WRONG - includes missing values!
list if age > 60

* CORRECT
list if age > 60 & !missing(age)
* or
list if age > 60 & age < .
```

### Common Missing Value Patterns

```stata
misstable summarize              // Overview of missing data
misstable patterns               // Missing patterns
count if missing(age)

replace income = . if income == 999 | income == -99
mvdecode _all, mv(999=. \ -99=.)    // Batch recode to missing

* Forward fill
bysort id (date): replace value = value[_n-1] if missing(value)
```

### if and in Qualifiers

```stata
* if qualifier: logical conditions
summarize income if age > 25
count if age >= 18 & age <= 65
gen eligible = 1 if (age >= 18 & citizen == 1) | (age >= 21 & resident == 1)
list if inlist(state, 1, 2, 5, 10)
count if inrange(age, 18, 65)

* in qualifier: observation ranges
list in 1/10          // First 10
list in -10/l         // Last 10 ('l' = last)
list in f             // First observation

* Note: in is evaluated first, then if
```

**Gotcha**: Wildcards (`*`) do NOT work in `if` expressions. They only work in variable lists.

### Variable Lists and Wildcards

```stata
describe age-education       // Range of variables
summarize inc*               // Wildcard at end
describe *_id                // Wildcard at beginning
drop _*                      // Variables starting with underscore

* By variable type
ds, has(type numeric)        // All numeric variables
ds, has(type string)         // All string variables
ds, has(vallabel)            // Variables with value labels
```

### Temporary Variables

Automatically deleted when program/do-file ends.

```stata
tempvar sumsq
gen `sumsq' = x^2          // Reference with backticks

tempname myscalar mymatrix  // For scalars/matrices
scalar `myscalar' = 100

tempfile mydata             // For temporary datasets
save `mydata'
```

---

## Data Import and Export

### File Format Summary

| Format | Import Command | Export Command |
|--------|---------------|----------------|
| Stata (.dta) | `use` | `save` |
| CSV/TXT | `import delimited` | `export delimited` |
| Excel | `import excel` | `export excel` |
| dBASE | `import dbase` | `export dbase` |
| ODBC | `odbc load` | `odbc insert` |
| SAS XPORT | `import sasxport5` | `export sasxport5` |

### Import Delimited Files

```stata
import delimited [using] filename [, options]
```

**Key Options**:
- `delimiter("char")` / `delimiter(tab)`: Specify delimiter
- `varnames(#)` / `varnames(nonames)`: Row with variable names
- `clear`: Replace data in memory
- `case(preserve|lower|upper)`: Variable name case
- `encoding("UTF-8")`: Character encoding
- `rowrange([start][:end])`: Import specific rows
- `colrange([start][:end])`: Import specific columns
- `stringcols(numlist)`: Force columns to string

**Examples**:
```stata
import delimited "data.csv", clear
import delimited "data.txt", delimiter(tab) clear
import delimited "data.csv", delimiter(";") clear          // European
import delimited "data.csv", varnames(1) rowrange(2:1000) case(preserve) clear
import delimited "data.csv", stringcols(1 3 5) clear
import delimited "data.csv", encoding("UTF-8") clear
```

### Import Excel Files

```stata
import excel [using] filename [, options]
```

**Key Options**:
- `sheet("sheetname")`: Specify worksheet
- `cellrange([start][:end])`: Import specific cell range
- `firstrow`: First row as variable names
- `allstring`: Import all data as strings (useful for mixed types)

**Examples**:
```stata
import excel "data.xlsx", firstrow clear
import excel "workbook.xlsx", sheet("Sales_2023") firstrow clear
import excel "data.xlsx", sheet("Data") cellrange(A1:E100) firstrow clear

* Import all as strings then convert
import excel "data.xlsx", firstrow allstring clear
destring var1 var2 var3, replace

* Describe Excel file before importing
import excel "workbook.xlsx", describe
```

### Use and Save Commands

```stata
* Load entire dataset
use "data/mydata.dta", clear

* Load specific variables
use id name age income using "data/mydata.dta", clear

* Load subset of observations
use "data/mydata.dta" if age > 18, clear

* Describe file without loading
describe using "data/mydata.dta"

* Save
save "data/mydata.dta", replace

* Save in older format for compatibility
saveold "data/mydata_v13.dta", replace version(13)

* Preserve and restore
preserve
    keep if year == 2020
    summarize
restore
```

### Export Commands

```stata
* Export delimited
export delimited using "output/data.csv", replace
export delimited using "output/data.txt", delimiter(tab) replace
export delimited id name income using "output/subset.csv", replace
export delimited using "output/data.csv", nolabel replace     // Values not labels
export delimited using "output/data.csv", quote replace        // Quote strings

* Export Excel
export excel using "output/data.xlsx", firstrow(variables) replace
export excel using "output/data.xlsx", firstrow(varlabels) replace
export excel using "output/data.xlsx", sheet("Results") firstrow(variables) replace
export excel using "output/data.xlsx", sheet("Data") cell(B2) firstrow(variables) replace

* Add sheet to existing workbook
export excel using "output/existing.xlsx", sheet("NewData") firstrow(variables) modify
```

### ODBC and SQL Database Connections

```stata
* List available data sources and tables
odbc list
odbc query "MyDataSource"

* Load from database
odbc load, dsn("MyDatabase") table("customers") clear

* Execute SQL query
odbc load, exec("SELECT * FROM sales WHERE year = 2023") dsn("MyDatabase") clear

* Connection string (instead of DSN)
odbc load, exec("SELECT * FROM employees") ///
    connectionstring("Driver={SQL Server};Server=myserver;Database=mydb;Trusted_Connection=yes") clear

* Insert data into database
odbc insert, table("analysis_results") dsn("MyDatabase") create

* Execute SQL commands
odbc exec("UPDATE employees SET salary = salary * 1.05 WHERE dept = 'Sales'"), dsn("MyDatabase")
```

### Web Data Import

```stata
* Import CSV from URL
import delimited "https://example.com/data/mydata.csv", clear

* Stata's webuse for example datasets
webuse auto, clear
webuse nlswork, clear

* Download then import Excel
copy "https://example.com/data.xlsx" "temp_data.xlsx", replace
import excel "temp_data.xlsx", firstrow clear
erase "temp_data.xlsx"

* JSON via third-party command
ssc install jsonio
jsonio kv, file("data.json") flatten
```

### Common Import Issues and Solutions

**Variables imported as wrong type**:
```stata
* Force columns during import
import delimited "data.csv", numericcols(2 3 4 5) clear
import delimited "data.csv", stringcols(1 6 7) clear

* Convert after import
destring var1 var2, replace ignore("$" "," "%")
tostring var3 var4, replace
```

**Leading zeros dropped** (e.g., ID "00123" becomes "123"):
```stata
import delimited "data.csv", stringcols(1) clear
* Or format after import:
tostring id, replace format(%05.0f)
```

**Dates imported as strings**:
```stata
generate date_var = date(string_date, "YMD")   // "2023-01-15"
format date_var %td

generate date_var = date(string_date, "MDY")   // "01/15/2023"
format date_var %td

generate datetime_var = clock(string_datetime, "YMDhms")
format datetime_var %tc
```

**Missing values coded as strings**:
```stata
foreach var of varlist _all {
    replace `var' = "" if inlist(`var', "NA", "N/A", ".", "missing", "-999")
}
destring, replace
```

---

## Programming Fundamentals

### Do-files and Ado-files

#### Do-files

```stata
* Running
do "path/to/analysis.do"
quietly do "path/to/analysis.do"

* Best practice template
version 17
clear all
set more off
log using "analysis_log.log", replace

sysuse auto, clear
* ... analysis ...

log close
```

#### Ado-files

Ado-files extend Stata's functionality and are auto-loaded when called.

```stata
* mysumm.ado
program define mysumm
    version 17
    syntax varlist [if] [in]
    marksample touse
    foreach var of local varlist {
        quietly summarize `var' if `touse'
        display as text "`var': " as result %9.2f r(mean) ///
            " (SD: " %9.2f r(sd) ")"
    }
end
```

### Local and Global Macros

#### Local Macros

Exist only within the current program or do-file. Referenced with backtick-quote: `` `name' ``.

```stata
local filename "mydata.dta"
use "`filename'", clear

local n = 100
display "Sample size: `n'"

local demog_vars "age gender education income"
summarize `demog_vars'

* Extended macro functions
local price_label : variable label price
local foreign_vallabel : value label foreign
local count : word count `mylist'
local third : word 3 of `mylist'
```

#### Global Macros

Persist throughout the session. Referenced with `$name`.

```stata
global datadir "/home/user/data"
use "$datadir/myfile.dta", clear

global controls "age gender education income"
regress outcome treatment $controls
```

**Caution**: Globals create conflicts across programs. Prefer locals. Clear when done: `macro drop _all`.

### Loops

#### foreach Loop

```stata
* Over variable names
foreach var of varlist price mpg weight {
    quietly summarize `var'
    display "`var': mean = " r(mean)
}

* Over strings
foreach country in "USA" "Canada" "Mexico" {
    count if country == "`country'"
}

* Over a local macro
local outcomes "income health education"
foreach outcome of local outcomes {
    regress `outcome' `controls'
    estimates store model_`outcome'
}

* Over all numeric variables
foreach var of varlist _all {
    capture confirm numeric variable `var'
    if !_rc {
        summarize `var'
    }
}

* Nested loops
foreach year of local years {
    foreach region of local regions {
        * ...
    }
}
```

#### forvalues Loop

**GOTCHA: No C-style increment operators.** Stata has no `++`, `--`, `+=`, etc.

```stata
forvalues i = 1/10 {
    display "Iteration `i'"
}

forvalues year = 2000(5)2020 {
    display "Year: `year'"
}

forvalues i = 1/5 {
    generate category`i' = (category == `i')
}
```

### Conditional Statements

```stata
* Basic if
if _N > 50 {
    display "Large dataset"
}

* if/else
capture confirm variable price
if !_rc {
    summarize price
}
else {
    display "Variable 'price' not found"
}

* if/else if/else
if `age' < 18 {
    local category "Minor"
}
else if `age' >= 18 & `age' < 65 {
    local category "Adult"
}
else {
    local category "Senior"
}

* capture for error handling
capture drop newvar
generate newvar = oldvar * 2

capture confirm file "mydata.dta"
if _rc == 0 {
    use mydata.dta, clear
}
else {
    display "File not found"
}
```

### Programs and Functions

#### Basic Structure

```stata
program define programname
    version 17
    syntax [varlist] [if] [in] [, options]
    * Program code
end
```

#### Program with syntax

```stata
program define myreg
    version 17
    syntax varlist(min=2) [if] [in] [, Robust CLuster(varname)]

    local depvar : word 1 of `varlist'
    local indepvars : list varlist - depvar

    if "`robust'" != "" {
        regress `depvar' `indepvars' `if' `in', robust
    }
    else if "`cluster'" != "" {
        regress `depvar' `indepvars' `if' `in', cluster(`cluster')
    }
    else {
        regress `depvar' `indepvars' `if' `in'
    }
end

myreg price mpg weight, robust
```

#### Ado-file with Return Values

```stata
*! version 1.0.0
program define mystats, rclass
    version 17
    syntax varname [if] [in] [, Detail]
    marksample touse

    quietly summarize `varlist' if `touse', detail

    return scalar N = r(N)
    return scalar mean = r(mean)
    return scalar sd = r(sd)

    if "`detail'" != "" {
        return scalar p50 = r(p50)
    }

    display as text "Variable: " as result "`varlist'"
    display as text "Mean: " as result %9.2f return(mean)
end
```

### return and ereturn

#### r-class (general results)

```stata
program define compute_stats, rclass
    syntax varname
    quietly summarize `varlist'
    return scalar mean = r(mean)
    return scalar cv = r(sd) / r(mean)
    return local varname "`varlist'"
end

compute_stats price
display "CV: " r(cv)
```

**Gotcha**: r-class results are overwritten by the next r-class command. Save to locals immediately.

#### e-class (estimation results)

```stata
program define myregress, eclass
    syntax varlist [if] [in]
    marksample touse
    local depvar : word 1 of `varlist'
    local indepvars : list varlist - depvar
    regress `depvar' `indepvars' if `touse'
    ereturn local cmd "myregress"
    ereturn local depvar "`depvar'"
end
```

**Key difference**: e-class results persist until the next estimation command.

### preserve and restore

```stata
preserve
collapse (mean) avg_price=price, by(foreign)
list
restore
* Original data unchanged

* In loops
foreach condition of local groups {
    preserve
    keep if `condition'
    summarize price mpg weight
    restore
}
```

### quietly and noisily

```stata
* Suppress output
quietly summarize price
display "Mean: " r(mean)

* Block suppression
quietly {
    use mydata, clear
    generate newvar = oldvar * 2
    summarize newvar
}

* Force display inside quiet block
quietly {
    noisily summarize newvar
    regress y x
}

* In loops (faster)
foreach var of varlist price mpg weight {
    quietly summarize `var'
    display "`var': mean = " %9.2f r(mean)
}
```

---

## Advanced Programming

### Syntax Parsing

#### Complete Syntax Specification

```stata
program define advanced_syntax_demo
    version 17
    #delimit ;
    syntax [varlist(min=1 max=5 numeric default=none)]
           [if] [in] [using/]
           [fweight aweight pweight iweight]
           [,
           Required(varname numeric)
           OPTional(varname)
           STRing(string)
           INTeger(integer 10)
           REAL(real 0.5)
           NUMlist(numlist min=1 max=10 >0 <=100)
           VARList(varlist numeric min=2)
           GENerate(name)
           REPlace
           BY(varlist)
           Level(cilevel)
           *                        // Catch additional options
           ];
    #delimit cr
    marksample touse
end
```

#### Varlist Specifications

```stata
syntax varlist(numeric)              // Numeric only
syntax varlist(string)               // String only
syntax varlist(min=2 max=10)         // Count constraints
syntax [varlist(default=none)]       // Optional, empty default
syntax varlist(ts)                   // Time-series operators allowed
syntax varlist(fv)                   // Factor variables allowed
```

#### Option Types

```stata
* Flags
syntax [, Replace NOIsily Detail]
if "`replace'" != "" { ... }

* Typed arguments
syntax [, SAVing(string) Level(cilevel) GENerate(name) BY(varlist)]

* Integer/real with defaults and validation
syntax [, Reps(integer 1000) Alpha(real 0.05)]
if `reps' < 100 {
    display as error "reps() must be at least 100"
    exit 198
}

* Numlist
syntax [, NUMlist(numlist min=1 >=0 <=1)]
```

### gettoken and tokenize

#### gettoken: Extract Tokens One at a Time

```stata
gettoken token rest : string [, parse(chars) quotes]

* Separate dependent from independent variables
gettoken depvar indepvars : varlist

* Custom delimiter
local mystring "one,two,three"
gettoken first rest : mystring, parse(",")
// first = "one", rest = ",two,three"
```

#### tokenize: Split into Numbered Macros

```stata
tokenize `mystring'
display "`1'"  "`2'"  "`3'"

* Count tokens
local i = 1
while "``i''" != "" {
    local ++i
}
local ntokens = `i' - 1
```

### Matrix Programming

#### Matrix Basics

```stata
matrix A = (1, 2, 3 \ 4, 5, 6 \ 7, 8, 9)
matrix accum R = price mpg weight       // From variables
matrix I = I(3)                         // Identity
matrix Z = J(3, 4, 0)                  // Zeros

* Operations
matrix D = B + C
matrix E = B * C'                       // Multiply
matrix F = 2 * B                        // Scalar multiply

* Functions
matrix B = A'                           // Transpose
matrix Ainv = syminv(A)                 // Symmetric inverse
matrix d = vecdiag(A)                   // Diagonal
matrix symeigen X L = A                 // Eigendecomposition
```

#### Custom OLS via Matrices

```stata
program define matrix_ols, rclass
    version 17
    syntax varlist
    gettoken depvar indepvars : varlist
    tempname X y XX Xy b V

    mkmat `indepvars', matrix(`X')
    mkmat `depvar', matrix(`y')
    local n = rowsof(`X')
    matrix ones = J(`n', 1, 1)
    matrix `X' = ones, `X'

    matrix `XX' = `X'' * `X'
    matrix `Xy' = `X'' * `y'
    matrix `b' = syminv(`XX') * `Xy'

    matrix yhat = `X' * `b'
    matrix e = `y' - yhat
    matrix e2 = e' * e
    local s2 = e2[1,1] / (`n' - colsof(`X'))
    matrix `V' = `s2' * syminv(`XX')

    return matrix b = `b'
    return matrix V = `V'
    return scalar N = `n'
end
```

### Debugging

```stata
* assert and confirm
assert price > 0
assert !missing(price, mpg, weight)
isid make                               // Assert unique ID

capture confirm variable `varname'
if _rc { display "`varname' not found" }

capture confirm numeric variable `varname'
capture confirm file "`filename'"

* trace
set trace on
set tracedepth 2                        // Limit depth
myprogram price mpg weight
set trace off

* Debug levels pattern
program define myprogram, rclass
    syntax varlist [, Debug Level(integer 0)]
    marksample touse

    if "`debug'" != "" | `level' > 0 {
        display "Debug: Variables = `varlist'"
        quietly count if `touse'
        display "Debug: N = " r(N)
    }
end
```

### Error Handling

#### Common Error Codes

| Code | Meaning |
|------|---------|
| 7 | Type mismatch |
| 111 | Variable not found |
| 198 | Invalid syntax |
| 416 | Missing values |
| 601 | File not found |

#### Custom Error Messages

```stata
if !inrange(`threshold', 0, 1) {
    display as error "threshold() must be between 0 and 1"
    exit 198
}
```

---

## Mata Introduction

### What is Mata?

Mata is Stata's compiled matrix programming language.

| Aspect | Stata | Mata |
|--------|-------|------|
| Execution | Interpreted | Compiled to bytecode |
| Speed | Good for data ops | 10-1000x faster |
| Data model | Observations/variables | Matrices/scalars |
| Syntax | Command-based | C-style |
| Primary use | Data mgmt, stats | Matrix ops, algorithms |

**Use Mata when**: loops are slow, matrix operations needed, custom estimators, simulations.
**Use Stata when**: data management, standard statistical procedures, interactive exploration.

### Mata Environment and Syntax

```mata
mata                    // Enter interactive mode
// ... Mata code ...
end                     // Return to Stata

mata: x = I(5)          // Single-line execution
```

#### Syntax Basics

```mata
// Assignment
x = 5
name = "Hello"
matrix = (1, 2, 3 \ 4, 5, 6)

// Comments: // or /* ... */
// Semicolons optional (separate statements on one line)
// Case-sensitive: x and X are different

// Display
x                       // Type name to display
printf("Value: %f\n", x)
```

### Mata Data Types

#### Numeric

```mata
x = 3.14159             // Real scalar
z = .                   // Missing value
c = 3 + 4i             // Complex
magnitude = abs(c)
real_part = Re(c)
```

#### String

```mata
name = "Stata"
upper = strupper(name)
length = strlen(name)
substr = substr(name, 1, 3)
concat = name + " 18"
```

#### Pointer

```mata
x = 10
p = &x                 // Address-of
*p                      // Dereference: returns 10
*p = 20                 // Modify through pointer

// Function pointer
fp = &square()
(*fp)(5)                // Call function via pointer

// Pointer matrices
ptrs = J(3, 3, NULL)
a = 1
ptrs[1,1] = &a
*ptrs[1,1]      // 1
```

#### Structure

```mata
struct person {
    string scalar name
    real scalar age
    real scalar income
}

struct person scalar p
p.name = "Alice"
p.age = 30
```

---

## Mata Programming

### Functions

#### Defining Functions

```mata
function real scalar add_numbers(real scalar a, real scalar b) {
    return(a + b)
}

function real matrix matrix_multiply(real matrix A, real matrix B) {
    if (cols(A) != rows(B)) _error(3200)  // conformability error
    return(A * B)
}

function void display_message(string scalar msg) {
    printf("Message: %s\n", msg)
}

* Type declarations: real scalar, real vector, real matrix, string scalar, pointer, transmorphic
```

#### Statistical Function Example

```mata
function real scalar compute_mean(real colvector x) {
    if (rows(x) == 0) return(.)
    return(sum(x) / rows(x))
}

function real scalar compute_variance(real colvector x) {
    real scalar n, m
    n = rows(x)
    if (n <= 1) return(.)
    m = compute_mean(x)
    return(sum((x :- m):^2) / (n - 1))
}
```

### Flow Control

#### If Statements

```mata
if (condition) { ... }
else if (condition2) { ... }
else { ... }

* Gotcha: Conditions must evaluate to a single scalar. For matrices use all() or any():
if (all(x :> 0)) printf("All positive\n")
if (any(x :== 0)) printf("At least one zero\n")
```

#### Loops

```mata
* For loops
for (i = 1; i <= 10; i++) {
    printf("%f\n", i)
}
// Prefer vectorized operations over element loops

* While and do-while
while (condition) { ... }

do {
    ...
} while (condition)      // Guarantees at least one execution

* Loop control
break       // Exit loop immediately
continue    // Skip to next iteration
```

### Pointers and Structures

```mata
* Basic pointer operations
x = 10
p = &x          // Address-of
*p              // Dereference: 10
*p = 20         // Modify through pointer

* Pointer to matrix
X = (1, 2 \ 3, 4)
p = &X
(*p)[2,2]       // Access element: 4

* Function pointers
fp = &square()
(*fp)(5)         // 25

* Practical: generic function application
function real scalar apply_op(real scalar x, real scalar y,
                              pointer(function) scalar op) {
    return((*op)(x, y))
}
result = apply_op(5, 3, &add())

* Structures
struct point {
    real scalar x
    real scalar y
}

struct point scalar p
p.x = 5.0
p.y = 3.0

* Functions with structures
struct point scalar create_point(real scalar x, real scalar y) {
    struct point scalar p
    p.x = x; p.y = y
    return(p)
}

function real scalar distance(struct point scalar p1, struct point scalar p2) {
    return(sqrt((p2.x - p1.x)^2 + (p2.y - p1.y)^2))
}
```

### Matrix Operations in Mata

#### Matrix Creation

```mata
// Constants and special matrices
A = J(3, 4, 0)                 // 3x4 zeros
B = J(2, 2, 1)                 // 2x2 ones
I3 = I(3)                      // 3x3 identity
D = diag((1, 2, 3, 4))        // Diagonal matrix

// Direct input
v = (1, 2, 3, 4)              // Row vector
w = (1 \ 2 \ 3 \ 4)           // Column vector
M = (1, 2, 3 \ 4, 5, 6)      // Matrix (\ = new row)

// Sequences and random
x = range(1, 10, 1)            // 1 to 10 step 1
U = runiform(3, 4)             // Uniform [0,1)
Z = rnormal(3, 4, 0, 1)       // Standard normal
rseed(12345)                    // Set seed

// Extract diagonal
d = diagonal(M)
```

#### Matrix Arithmetic

```mata
C = A + B                       // Addition
D = A - B                       // Subtraction
E = A * B                       // Matrix multiplication
F = A :* B                      // Element-wise multiply
G = A :/ 2                      // Element-wise divide
H = A :^ 2                      // Element-wise power
K = 2 * A                       // Scalar multiply
L = A :+ 10                     // Scalar add

// Transpose
At = A'
// Complex conjugate transpose
Ct = conj(C')
```

#### Matrix Functions

```mata
* Dimensions
rows(A); cols(A); length(v)

* Element-wise
sqrt(A); exp(A); log(A); abs(A); floor(A); ceil(A)

* Summary (WARNING: these propagate missing values silently)
sum(A)                          // All elements (. if any missing)
colsum(A); rowsum(A)            // (. propagates per column/row)
mean(A)                         // Column means
variance(A)                     // Covariance matrix
min(A); max(A)

* Manipulation
C = A \ B                       // Vertical concatenation
D = A, B                        // Horizontal concatenation
F = rowshape(E, 3)             // Reshape to 3 rows
M = colshape(v, 2)             // Reshape to 2 columns

* Decompositions
symeigensystem(A, eigenvectors=., eigenvalues=.)
evals = symeigenvalues(A)       // Ascending order
```

### Stata-Mata Data Access

#### st_data() -- Copy Data

Creates an independent copy of Stata data in a Mata matrix.

```mata
// All observations, named variables
X = st_data(., ("price", "mpg", "weight"))

// Specific observations
X = st_data((1::10), ("mpg", "weight"))

// With selection variable (obs included where selectvar != 0)
foreign_data = st_data(., ("price", "mpg"), "foreign")

// By variable index
X = st_data(., (1, 2, 3))

// CRITICAL: Missing Values in Mata
// Mata's mean(), variance(), correlation() etc. propagate missing values silently
// A single missing value in a column makes mean() return "." for that column
// ALWAYS filter missings before computing:

// WRONG: if any value in X is missing, mean() returns "."
result = mean(X)

// RIGHT: drop rows with any missing values
Xclean = select(X, rowmissing(X) :== 0)
result = mean(Xclean)

// Missing-value detection functions:
// missing(x) -- 1 if scalar x is missing, element-wise for matrices
// hasmissing(X) -- 1 if any element of X is missing
// rowmissing(X) -- Column vector: count of missings per row
// colmissing(X) -- Row vector: count of missings per column
```

#### st_view() -- Create View (No Copy)

Creates a reference into Stata data. Changes to the view modify Stata data.

```mata
st_view(X=., ., ("price", "mpg", "weight"))
X[1,1] = 999                   // Modifies Stata data!

// With selection
st_view(domestic=., ., ("price", "mpg"), "!foreign")

// Specific rows
st_view(subset=., (1::10), ("price", "mpg"))

// When to use which:
// st_data(): Need independent copy, data may change, small datasets
// st_view(): Large datasets (memory), want to modify Stata data, read-only computation
```

#### st_store() -- Write Data

Variables must exist before storing. Matrix dimensions must match.

```mata
// Store to existing variable
mpg_sq = st_data(., "mpg") :^ 2
st_store(., "mpg_squared", mpg_sq)

// Store to specific observations
st_store((1::10), "mpg_squared", J(10, 1, 0))

// With selection
st_store(., "mpg", foreign_mpg, "foreign")

// Creating new variables
idx = st_addvar("double", "log_price")
st_store(., idx, log(st_data(., "price")))
```

---

## External Tools Integration

### Python Integration (Stata 16+)

```stata
* Execute single Python command
python: print("Hello from Python!")

* Execute Python block
python:
import numpy as np
arr = np.array([1, 2, 3, 4, 5])
print(f"Mean: {arr.mean()}")
end

* Use Python script file
python script "analysis.py"

* Exchanging data between Stata and Python
sysuse auto, clear

python:
from sfi import Data, Scalar, Macro

# Read Stata data into Python
price = Data.get("price")
mpg = Data.get("mpg")

# Perform Python calculations
import numpy as np
correlation = np.corrcoef(price, mpg)[0,1]

# Send result back to Stata
Scalar.setValue("r(corr)", correlation)
end

* Access result in Stata
display "Correlation: " r(corr)

* Reading all data into pandas
python:
import pandas as pd
from sfi import Data, Macro

df = pd.DataFrame({
    'price': Data.get("price"),
    'mpg': Data.get("mpg"),
    'weight': Data.get("weight")
})

summary = df.groupby(pd.cut(df['mpg'], bins=3)).agg({
    'price': ['mean', 'std'],
    'weight': 'mean'
})
print(summary)
end
```

### R Integration

```stata
* Execute R code inline
rsource: print("Hello from R")

* R code block
rsource {
    library(tidyverse)
    data <- data.frame(x = c(1,2,3), y = c(4,5,6))
    cor(data$x, data$y)
}

* Exchanging data with R
sysuse auto, clear

rsource {
    stata_data <- .readStata()         // Read Stata data
    cor_matrix <- cor(stata_data)
    write.csv(cor_matrix, "correlations.csv")
    .writeStata(stata_data)             // Write back to Stata
}
```

### Shell Commands and Git

```stata
* Execute shell commands
shell ls -la /path/to/directory
shell python script.py

* Git operations
shell git status
shell git add .
shell git commit -m "Analysis update"
shell git push
```

### Best Practices

- **Encapsulate**: Wrap external tools in Stata programs for reproducibility
- **Error Handling**: Use `capture` when calling external tools
- **Documentation**: Comment code and explain why you're using external tools
- **Data Verification**: Always check that data was successfully exchanged
- **Version Control**: Track which Python/R versions and packages your code requires

---

## Workflow Best Practices

### Project Organization

Recommended directory layout:

```
MyProject/
├── data/
│   ├── raw/              # Original, untouched data (READ ONLY)
│   ├── intermediate/     # Partially processed data
│   └── final/           # Analysis-ready datasets
├── code/
│   ├── 00_master.do     # Master script that runs everything
│   ├── 01_import.do     # Data import scripts
│   ├── 02_clean.do      # Data cleaning
│   ├── 03_construct.do  # Variable construction
│   ├── 04_analysis.do   # Statistical analysis
│   └── 05_tables.do     # Table and figure generation
├── output/
│   ├── tables/
│   ├── figures/
│   └── logs/
├── documentation/
│   ├── codebook.xlsx
│   └── methods.md
├── .gitignore
└── README.md
```

### Master Do-File Template

```stata
/*******************************************************************************
MASTER DO-FILE: Project Title
Author: Your Name | Date: 2025-11-23
Description: Runs the complete analysis pipeline.
Requires: Stata 17+, estout, reghdfe
*******************************************************************************/

version 17
clear all
macro drop _all
set more off
set varabbrev off              // Require full variable names

* Per-user project root
if "`c(username)'" == "johndoe" {
    global project "C:/Users/johndoe/Projects/MyProject"
}
else {
    display as error "Unknown user: `c(username)'. Add your path to setup."
    exit
}

* Define subdirectories
global data       "$project/data"
global code       "$project/code"
global output     "$project/output"
global logs       "$output/logs"

* Run all modules
do "$code/01_import.do"
do "$code/02_clean.do"
do "$code/03_construct.do"
do "$code/04_analysis.do"
do "$code/05_tables.do"

display as text "Analysis completed: " as result c(current_date) " " c(current_time)
```

### Data Management Best Practices

```stata
* Load and inspect
import delimited "raw_data.csv", clear varnames(1)
describe
codebook, compact

* Clean
rename *, lower                 // lowercase all varnames
destring income, replace force  // convert string to numeric
replace income = . if income < 0

* Label
label variable income "Annual household income (USD)"
label define yesno 0 "No" 1 "Yes"
label values employed yesno

* Verify
assert income >= 0 if !missing(income)
duplicates report id
misstable summarize

* Save
compress
save "clean_data.dta", replace
```

### Code Commenting and Documentation

- Comment every major section with `* Description`
- Use `//` for inline clarifications
- Document why, not what: `// Drop zeros because they represent non-respondents`
- Use descriptive variable names: `age_at_survey_start` not `x1`
- Version control: include version history in ado-files

### Common Pitfalls to Avoid

1. **Not using `clear` before loading data** — always `use "data.dta", clear`
2. **Confusing `=` and `==`** — `=` assigns, `==` compares
3. **Ignoring missing values** — missing values sort to +infinity
4. **Using `by` without sorting** — always use `bysort`
5. **Forgetting closing quotes in macros** — `` `name' `` not `` `name ``
6. **Not checking `_merge`** — always `tab _merge` before `assert _merge == 3`
7. **Silent merge failures** — `merge` doesn't error if IDs don't match, use `assert`
8. **Modifying global data without `preserve`** — always save state if changing data
9. **Not testing code incrementally** — run in sections, not all at once
10. **Mixing absolute and relative paths** — use globals and absolute paths

---

## Key Gotchas Reference

### Missing Values Sort to +Infinity

```stata
* WRONG — includes observations where income is missing!
gen high_income = (income > 50000)

* RIGHT
gen high_income = (income > 50000) if !missing(income)
```

### `=` vs `==`

```stata
* WRONG — syntax error
gen employed = 1 if status = 1

* RIGHT
gen employed = 1 if status == 1
```

### Local Macro Syntax

Locals use `` `name' `` (backtick + single-quote). Forgetting the closing quote is the #1 macro bug.

```stata
local controls "age education income"
regress wage `controls'        // correct
regress wage `controls         // WRONG — missing closing quote
```

### `by` Requires Prior Sort (Use `bysort`)

```stata
* WRONG — error if data not sorted by id
by id: gen first = (_n == 1)

* RIGHT — bysort sorts automatically
bysort id: gen first = (_n == 1)
```

### Factor Variable Notation (`i.` and `c.`)

```stata
* WRONG — treats race as continuous
regress wage race education

* RIGHT — creates dummies automatically
regress wage i.race education

* Interactions
regress wage i.race##c.education    // full interaction
regress wage i.race#c.education     // interaction only
```

### `merge` Always Check `_merge`

```stata
merge 1:1 id using other.dta
tab _merge                      // ALWAYS tab before assert
assert _merge == 3              // fails silently without tab output
drop _merge
```

### Weights Are Not Interchangeable

- `fweight` — frequency weights (replication)
- `aweight` — analytic/regression weights (inverse variance)
- `pweight` — probability/sampling weights (survey data, implies robust SE)
- `iweight` — importance weights (rarely used)

### Line Continuation Uses `///`

```stata
regress y x1 x2 x3 ///
    x4 x5 x6, ///
    vce(robust)
```

### Stored Results: `r()` vs `e()` vs `s()`

- `r()` — r-class commands (summarize, tabulate, etc.)
- `e()` — e-class commands (estimation: regress, logit, etc.)
- `s()` — s-class commands (parsing)

A new estimation command **overwrites** previous `e()` results. Store them first: `estimates store model1`
