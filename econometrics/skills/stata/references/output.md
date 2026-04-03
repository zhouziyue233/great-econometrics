# Stata Output: Graphics, Tables & Reporting

## Table of Contents

1. [Graphics Fundamentals](#graphics-fundamentals)
2. [Graph Schemes & Styling](#graph-schemes--styling)
3. [Coefficient & Event Study Plots](#coefficient--event-study-plots)
4. [Regression Tables](#regression-tables)
5. [Summary Tables](#summary-tables)
6. [Export to Multiple Formats](#export-to-multiple-formats)
7. [Graph Export Best Practices](#graph-export-best-practices)

---

## Graphics Fundamentals

### Twoway Plots

Foundational graph type for scatter, line, and combined plots.

```stata
* Scatter plot with marker customization
twoway scatter mpg weight, ///
    msymbol(square) msize(medium) mcolor(blue) ///
    mlabel(make) mlabposition(3) mlabsize(vsmall)

* Multiple overlays: scatter + fitted line + CI
twoway (scatter mpg weight) ///
       (lfitci mpg weight), ///
       legend(label(1 "Actual") label(2 "Linear Fit + 95% CI"))

* Area plot with filled confidence region
twoway (rarea upper lower x, fcolor(gray%20)) ///
       (line yhat x, lcolor(navy)), ///
       title("Confidence Band Plot")
```

### Bar and Box Plots

```stata
* Bar chart with groups and custom colors
graph bar (mean) mpg (mean) price, over(foreign) ///
    bar(1, color(navy)) bar(2, color(maroon)) ///
    blabel(bar, format(%4.1f))

* Box plot with multiple grouping dimensions
graph box mpg, over(foreign) over(rep78) ///
    box(1, fcolor(navy%50)) medtype(line) medline(lcolor(red))
```

### Histograms

```stata
histogram mpg, frequency bin(15) normal ///
    fcolor(navy%60) lcolor(navy) ///
    addlabel ylabel(, angle(0))
```

### Combining Graphs

```stata
* Create named graphs with nodraw
twoway scatter mpg weight, name(g1, replace) nodraw
histogram mpg, name(g2, replace) nodraw

* Combine with shared axes and custom layout
graph combine g1 g2, ///
    rows(1) ycommon xcommon ///
    imargin(medium) iscale(0.8) ///
    title("Multi-Panel Figure")
```

### Graph Options

**Titles and labels:**
```stata
title("Main Title") subtitle("Subtitle") ///
xtitle("X-Axis Label") ytitle("Y-Axis Label") ///
note("Source and notes here")
```

**Axis customization:**
```stata
xscale(range(0 100)) xlabel(0(10)100) ///
ylabel(, grid gstyle(dot) angle(0)) ///
xscale(log)  // log scale
```

**Legend options:**
```stata
legend(position(6) rows(1)) ///       // Bottom, 1 row
legend(ring(0) position(11)) ///      // Inside, 11 o'clock
legend(title("Legend")) ///
legend(order(2 "Label2" 1 "Label1")) // Reorder
```

**Regions and sizing:**
```stata
graphregion(color(white)) ///
plotregion(color(gs15) margin(medium)) ///
xsize(8) ysize(6) ///
aspectratio(1)  // Square plot
```

---

## Graph Schemes & Styling

Schemes control overall appearance (colors, fonts, sizes, lines). Choose scheme early; it affects all subsequent graphs in session.

### Available Schemes

```stata
* Built-in Stata schemes
set scheme s2color       // Default, colored
set scheme s1mono        // Black & white
set scheme economist     // Economist magazine style
set scheme sj            // Stata Journal style

* Community schemes (install first)
ssc install blindschemes, replace    // plotplain, colorblind-safe
ssc install schemepack, replace      // 538, tableau, gg_s2color, tufte
set scheme plotplain     // Publication-quality (most common)
set scheme white_tableau // Tableau colors, white background
```

### grstyle for On-the-Fly Customization

Dynamic customization without creating scheme files.

```stata
ssc install grstyle palettes colrspace

grstyle clear
grstyle init
grstyle set plain, box               // Clean white bg with box
grstyle set color tableau            // Tableau palette
grstyle set color viridis            // Colorblind-friendly
grstyle set symbol O D T S           // Marker symbols
grstyle set lpattern solid dash dot  // Line patterns
grstyle set size 11pt                // Font size
```

### Per-Graph Override

```stata
twoway scatter mpg weight, scheme(plotplain)
* Other graphs continue with session default
```

### Publication Recommendations

| Journal Type | Scheme | Notes |
|---|---|---|
| B&W print | `s1mono`, `plotplain` | No colors, clear lines |
| Color online | `plotplain`, `sj` | Clean, professional |
| Presentations | `538`, `white_tableau` | Bold, colorful |
| Colorblind accessibility | `white_cividis`, `plotplainblind` | Safe palettes |

---

## Coefficient & Event Study Plots

### coefplot: Publication-Ready Coefficient Plots

Drop-in visualization for saved estimation results with confidence intervals.

```stata
ssc install coefplot

* Basic plot from last estimation
regress price mpg weight foreign
coefplot, drop(_cons) xline(0) ///
    title("Price Determinants") ///
    xtitle("Coefficient Estimate")

* Multiple models side-by-side
quietly regress price mpg
estimates store m1
quietly regress price mpg weight
estimates store m2
coefplot m1 m2, drop(_cons) xline(0) ///
    msymbol(O D) mcolor(navy maroon) ///
    legend(order(1 "Model 1" 2 "Model 2"))

* Customization: markers, CIs, labels
coefplot, drop(_cons) ///
    msymbol(square) msize(large) mcolor(navy) ///
    ciopts(lwidth(thick) lcolor(navy)) ///
    levels(90 95) ///  // Multiple CI levels
    coeflabels(mpg="Fuel Efficiency" weight="Weight") ///
    xline(0, lpattern(dash)) ///
    scheme(plotplain)
```

**Common patterns:**
- `drop(_cons)` - Always drop constant
- `xline(0)` (or `yline(0)` if vertical) - Reference line
- `levels(95)` - Confidence level
- `vertical` - Vertical orientation
- `label` - Use variable labels
- `order()` - Reorder variables
- `headings()` - Add grouping headings

---

## Regression Tables

### esttab: Publication-Quality Tables (estout package)

Most flexible and widely-used table command. Supports LaTeX, Word, HTML, CSV.

```stata
ssc install estout  // Installs esttab, eststo, estpost, estadd

* Basic workflow
eststo clear
eststo: quietly regress price mpg
eststo: quietly regress price mpg weight
eststo: quietly regress price mpg weight foreign

* Display in Stata console
esttab

* Export to LaTeX with customization
esttab using "table1.tex", ///
    b(3) se(3) ///                         // 3 decimals for coef/SE
    star(* 0.10 ** 0.05 *** 0.01) ///    // Significance stars
    label ///                              // Use variable labels
    booktabs ///                           // Professional LaTeX lines
    mtitles("(1)" "(2)" "(3)") ///         // Column titles
    title("Table 1: Price Regressions") ///
    stats(N r2 r2_a, ///
        fmt(%9.0fc %9.3f %9.3f) ///
        labels("Observations" "R-squared" "Adj. R-squared")) ///
    note("Standard errors in parentheses." ///
         "* p<0.10, ** p<0.05, *** p<0.01") ///
    replace
```

**Key options:**
- `b(fmt)`, `se(fmt)` - Decimal formatting
- `keep()`, `drop()`, `order()` - Variable selection/ordering
- `varlabels()`, `coeflabels()` - Custom labels
- `star()` - Significance levels
- `scalars()` - Add model statistics
- `indicate()` - Show Yes/No for groups
- `mtitles()` - Column headers
- `mgroups()` - Panel headers
- `booktabs` - Professional formatting (LaTeX)
- `longtable` - Multi-page (LaTeX)

**Output formats:**
```stata
esttab using table.tex, replace      // LaTeX
esttab using table.rtf, replace      // Word (RTF)
esttab using table.html, replace html // HTML
esttab using table.csv, replace csv  // CSV
```

### outreg2: Quick Regression Tables

Faster alternative to esttab when you don't need maximum customization. Immediate export (no `eststo` needed).

```stata
ssc install outreg2

* First model creates file
regress price mpg weight
outreg2 using results.doc, replace ///
    ctitle("Model 1") label dec(3)

* Subsequent models append
regress price mpg weight foreign
outreg2 using results.doc, append ///
    ctitle("Model 2") label dec(3)

* LaTeX output
regress price mpg weight foreign
outreg2 using table.tex, replace tex ///
    label dec(3) ///
    addtext(Controls, Yes) ///
    addnote(Robust standard errors)
```

**Advantages:** Word/Excel native output, simple one-regression-at-a-time.
**Disadvantages:** Less control than esttab, sequential export.

### asdoc: Minimal-Syntax Table Export

Prefix command that exports results directly to Word without pre-storing estimates.

```stata
ssc install asdoc

* Just add 'asdoc' prefix
asdoc regress price mpg weight, replace save(results.doc)
asdoc regress price mpg weight foreign, append save(results.doc)

* Works with most Stata commands
asdoc summarize price mpg weight, replace
asdoc correlate price mpg weight, append
asdoc ttest price, by(foreign), append
```

**Use case:** Quick exploratory analysis, zero setup overhead.

---

## Summary Tables

### Summary Statistics with estpost + esttab

Most flexible approach for creating publication tables of means, medians, counts.

```stata
* Basic summary statistics table
estpost summarize price mpg weight length
esttab using summary.tex, replace ///
    cells("count(fmt(%9.0fc)) mean(fmt(%9.2fc)) sd(fmt(%9.2fc)) min(fmt(%9.2fc)) max(fmt(%9.2fc))") ///
    collabels("N" "Mean" "SD" "Min" "Max") ///
    label nomtitles ///
    title("Table 1: Descriptive Statistics\label{tab:summary}") ///
    booktabs
```

### Summary by Group with T-test

```stata
eststo clear
eststo domestic: estpost summarize price mpg weight if foreign==0
eststo foreign: estpost summarize price mpg weight if foreign==1
eststo diff: estpost ttest price mpg weight, by(foreign) unequal

esttab domestic foreign diff using comparison.tex, replace ///
    cells("mean(fmt(2)) sd(fmt(2) par) b(fmt(2) star)") ///
    mtitles("Domestic" "Foreign" "Difference") ///
    label nomtitles ///
    booktabs
```

### tabout: Flexible Cross-Tabulations

Powerful for two-way tables with custom statistics, percentages, tests.

```stata
ssc install tabout

* Frequency with row/column percentages
tabout foreign rep78 using crosstab.tex, ///
    c(freq row col) f(0 1 1) ///
    clab(N Row% Col%) ///
    ptotal(both) stats(chi2) ///
    style(tex) bt replace
```

### Baseline Characteristics Table (Table 1)

```stata
* Panel A: Categorical variables
tabout treatment using table1.tex, ///
    c(freq col) f(0 1) ///
    clab(N %) ///
    h1("Panel A: Categorical Variables") ///
    ptotal(single) ///
    style(tex) bt replace

* Panel B: Continuous summary by group
tabout treatment using table1.tex, ///
    sum ///
    c(N price mean price sd price) f(0 2 2) ///
    clab(N Mean SD) ///
    h1("Panel B: Continuous Variables") ///
    style(tex) bt append
```

---

## Export to Multiple Formats

### Excel with putexcel

Native Excel output with formatting, formulas, multiple sheets.

```stata
putexcel set "analysis.xlsx", replace

* Title with merged cells
putexcel A1 = "Descriptive Statistics"
putexcel A1:E1, merge bold fpattern(solid, "lightblue")

* Headers
putexcel A3 = "Variable" B3 = "N" C3 = "Mean" D3 = "SD"
putexcel A3:D3, bold border(bottom)

* Data
local row = 4
foreach var of varlist price mpg weight {
    quietly summarize `var'
    putexcel A`row' = "`var'"
    putexcel B`row' = r(N)
    putexcel C`row' = r(mean), nformat(number_d2)
    putexcel D`row' = r(sd), nformat(number_d2)
    local row = `row' + 1
}
```

### Word with putdocx

Native Word document with text, tables, graphs.

```stata
putdocx begin, font("Times New Roman", 11)

putdocx paragraph, style(Title)
putdocx text ("Analysis Report")

putdocx paragraph, style(Heading1)
putdocx text ("Summary Statistics")

* Data table from dataset
putdocx table tbl1 = data("price mpg weight length"), ///
    varnames border(all) ///
    title("Table 1: Descriptive Statistics")

* Add graph
graph export temp.png, replace width(1200)
putdocx paragraph
putdocx image temp.png, width(5)

putdocx save "report.docx", replace
```

### LaTeX

Build complete LaTeX documents or fragments for manual compilation.

```stata
* Professional regression table
esttab using results.tex, replace booktabs ///
    b(3) se(3) ///
    star(* 0.10 ** 0.05 *** 0.01) ///
    label nomtitles ///
    mgroups("Dependent Variable: log(Price)", ///
        pattern(1 0 0) ///
        prefix(\multicolumn{@span}{c}{) suffix(}) ///
        span erepeat(\cmidrule(lr){@span})) ///
    prehead("\begin{table}[htbp]\centering" ///
            "\caption{Regression Results}" ///
            "\label{tab:main}" ///
            "\begin{tabular}{l*{3}{c}}" ///
            "\toprule") ///
    posthead("\midrule") ///
    prefoot("\midrule") ///
    postfoot("\bottomrule" ///
             "\end{tabular}" ///
             "\end{table}")

* Include in LaTeX document:
* \input{results.tex}
```

---

## Graph Export Best Practices

### Vector Formats (for Publications)

```stata
* PDF - preferred for academic journals and LaTeX
graph export figure.pdf, replace

* EPS - older journals may require
graph export figure.eps, replace fontface(Times)

* SVG - web/interactive
graph export figure.svg, replace
```

### Raster Formats (for Web/Office)

```stata
* High resolution for print-quality rasters (~300 DPI)
graph export figure.png, width(3000) replace

* Medium resolution for presentations
graph export figure.png, width(1920) replace

* Web resolution
graph export figure.png, width(800) replace
```

### Multi-Panel Figures

```stata
* Create and name individual graphs
twoway scatter mpg weight, name(panel_a, replace) nodraw
twoway scatter mpg length, name(panel_b, replace) nodraw

* Combine with custom layout
graph combine panel_a panel_b, cols(2) ///
    title("Figure 1: Bivariate Relationships") ///
    graphregion(color(white))

* Export combined figure
graph export figure1.pdf, replace
```

### Batch Export

```stata
local graphs "g1 g2 g3"
foreach g in `graphs' {
    graph export `g'.pdf, name(`g') replace
}
```

---

## Publication Checklist

- [ ] Scheme set early (`plotplain` recommended for academic)
- [ ] Consistent decimal places (usually 2-3 in tables)
- [ ] Variable labels set before creating tables
- [ ] All models have same set of statistics reported
- [ ] Footnotes explain methodology (SEs type, FE included, etc.)
- [ ] Significance stars: * p<0.10, ** p<0.05, *** p<0.01 (or adjust for field)
- [ ] Graphs use professional color scheme (avoid rainbow, rainbow variations)
- [ ] Graphs have titles, axis labels, legends
- [ ] Graphs exported as PDF (vector) for LaTeX, or high-res PNG for Word
- [ ] Regression tables show both N and R-squared (or equivalent)
