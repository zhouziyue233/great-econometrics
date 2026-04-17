# Great-Econometrics Plugin
<p align="center"><img src="./icon.png" width="200"></p>

> **A comprehensive econometrics plugin for end-to-end empirical research automation** — from research question scoping, literature review, and identification strategy design, through data fetching and cleaning, econometric modeling, code generation, results analysis, table and figure production, full-paper writing, and conference presentation.

<p align="center">
  <img src="https://img.shields.io/badge/version-2.2.6-blue" alt="Version">
</p>


---

## Installation

### Via Claude Code Plugin (Recommended)

```shell
# Step 1: Add the marketplace
/plugin marketplace add zhouziyue233/great-econometrics

# Step 2: Install the plugin
/plugin install econometrics@great-econometrics
```

### Manual Installation

```bash
git clone https://github.com/zhouziyue233/great-econometrics.git
cd great-econometrics
/plugin install .
```

## Overview

This plugin implements a **9-phase iterative empirical research workflow** targeting the standards of Top-5 economics journals (*AER*, *QJE*, *JPE*, *ReStud*, *Econometrica*). Each phase maps to dedicated slash commands, skills, and agents that work together under a unified system prompt in `CLAUDE.md`.

```
[Phase 1] Research Question
     ↕
[Phase 2] Literature Review
     ↕
[Phase 3] Identification Strategy
     ↕
[Phase 4] Data Preparation & EDA
     ↕
[Phase 5] Econometric Model
     ↕
[Phase 6] Code Execution
     ↕
[Phase 7] Results & Visualization
     ↕
[Phase 8] Robustness & Heterogeneity
     ↕
[Phase 9] Full Paper Writing
     ↕
Peer Review Response & Iteration
```

---

## File Structure

```
great-econometrics/
├── .claude-plugin/
│   ├── plugin.json              # Plugin metadata
│   └── marketplace.json         # Claude marketplace
├── CLAUDE.md                    # Agent 9-phase workflow
├── .env                         # Environment variables
│
├── shared/                      # Cross-command reusable modules
│   ├── context-reader.md        # Upstream file reading protocol (Step 0)
│   └── output-standards.md      # LaTeX/figure/table formatting standards
│
├── hooks/
│   └── hooks.json               # SessionStart banner + Stop phase-completion prompt
│
├── commands/                    # Slash commands — one per workflow phase
│   ├── question.md              # /question  · Phase 1: Research question scoping
│   ├── analyze.md               # /analyze   · Phase 3: Identification strategy memo
│   ├── data.md                  # /data      · Phase 4: Data fetch, clean & EDA
│   ├── model.md                 # /model     · Phase 5: Formal model specification
│   ├── code.md                  # /code      · Phase 6: Code generation & execution
│   ├── plot.md                  # /plot      · Phase 7: Tables & figures
│   ├── robustness.md            # /robustness· Phase 8: Robustness & hetero checks
│   ├── write.md                 # /write     · Phase 9: Full paper drafting
│   └── present.md               # /present   · Beamer slides (optional, no phase number)
│
├── agents/
│   └── checker.md         # Parallel robustness/heterogeneity/mechanism checks (Phase 8)
│
└── skills/                      # Estimation, output & utility skills
    │
    ├── — Causal Inference —
    ├── iv-estimation/           # IV / 2SLS
    ├── did-analysis/            # DID / TWFE, event study, staggered DID
    ├── rdd-analysis/            # Sharp / fuzzy RDD, bandwidth selection, validity tests
    ├── synthetic-control/       # Abadie SCM, augmented SCM, synthetic DID, etc
    ├── ml-causal/               # Causal forest (GRF), Double ML, etc
    │
    ├── — Regression & Panel —
    ├── ols-regression/          # OLS, assumption tests, robust / clustered SE
    ├── panel-data/              # FE / RE, two-way FE, Hausman, dynamic panels (GMM)
    │     └── references/
    │           ├── panel-reference.md       # Diagnostics: poolability, serial corr, unit roots, cointegration
    │           └── panel-ldv-advanced.md    # Advanced: FD estimator, conditional logit, PPML, Cox PH
    ├── time-series/             # ADF / KPSS, ARIMA, VAR / VECM, etc
    │
    ├── — Output & Presentation —
    ├── results-analysis/        # Descriptive stats, coefficient interpretation, etc
    ├── table/                   # Multi-model regression tables, LaTeX
    ├── figure/                  # Event study, binscatter, RDD, density, coefplot
    ├── beamer-ppt/              # LaTeX Beamer slide decks for conferences & seminars
    │
    ├── — Data & Literature —
    ├── data-pipeline/           # End-to-end: API fetch + clean + validate
    ├── literature-review/       # Phase 2: search, summarise & synthesise literature
    ├── scrapling/               # Web scraping toolkit for custom data collection
    │
    ├── — Writing —
    ├── paper-writing/           # Full paper drafting to Top-5 journal standards
    │
    └── — Software Reference —
        └── stata/               # Generates Stata .do files for manual execution;
```

---

## Commands

| Command | Phase | Description |
|---|---|---|
| `/question` | 1 | Transform a research idea into a clear, well-scoped research question |
| `/analyze` | 3 | Diagnose endogeneity threats, evaluate and select an identification strategy; produce an Identification Strategy Memo |
| `/data` | 4 | Fetch, clean, and explore data; produce a Data Report |
| `/model` | 5 | Specify the formal econometric model with LaTeX equations and SE strategy |
| `/code` | 6 | Generate and execute a reproducible analysis script in Python, R, or Stata |
| `/plot` | 7 | Audit and upgrade all tables and figures to top-journal standards |
| `/robustness` | 8 | Design and run robustness checks, heterogeneity analysis, and mechanism tests |
| `/write` | 9 | Draft any section or the complete paper to Top-5 journal standards |
| `/present` | — | Generate a fully compilable LaTeX Beamer slide deck from all upstream outputs |

---

## Skills

| Skill | Triggers | Description |
|---|---|---|
| **ols-regression** | "OLS", "linear regression", "最小二乘法", "线性回归", "跑回归" | OLS estimation, assumption testing, heteroskedasticity, robust SE, coefficient interpretation |
| **time-series** | "ARIMA", "unit root", "Granger causality", "时间序列", "平稳性" | Stationarity tests (ADF/KPSS), ARIMA, VAR/VECM, cointegration, forecasting |
| **panel-data** | "fixed effects", "Hausman test", "面板数据", "固定效应" | FE/RE models, two-way FE, clustered SE, dynamic panels |
| **iv-estimation** | "instrumental variables", "2SLS", "工具变量", "内生性", "两阶段最小二乘法" | IV/2SLS, first-stage diagnostics, weak instruments, Bartik instruments, judge designs, PSM |
| **did-analysis** | "difference-in-differences", "DID", "parallel trends", "双重差分", "平行趋势" | DID/TWFE, event study, staggered DID (CS, SA, BJS, dCDH), pre-trends power analysis |
| **rdd-analysis** | "regression discontinuity", "RDD", "断点回归", "带宽选择" | Sharp/fuzzy RDD, bandwidth selection, validity tests, discrete RV, multi-cutoff/multi-score RDD |
| **synthetic-control** | "synthetic control", "SCM", "合成控制", "合成控制法" | Abadie-Diamond-Hainmueller SCM, augmented SCM, synthetic DID, placebo inference, MSPE ratios |
| **ml-causal** | "causal forest", "double ML", "DML", "GRF", "因果森林", "双重机器学习" | Causal Forest (GRF), Double/Debiased ML, LASSO variable selection, CATE, BLP/CLAN analysis |
| **results-analysis** | "summary statistics", "Table 1", "balance table", "描述性统计", "结果解读" | Descriptive stats, balance tests, coefficient interpretation, effect sizes, causal language calibration |
| **table** | "regression table", "LaTeX table", "esttab", "modelsummary", "回归表格" | Multi-model regression tables, multi-panel layouts, AER/QJE journal formatting |
| **figure** | "publication figure", "event study plot", "binscatter", "论文图表" | Journal-quality figures: event study, binscatter, RDD, density, coefplot |
| **data-pipeline** | "FRED", "World Bank", "fetch data", "clean data", "获取数据", "数据清洗" | End-to-end data pipeline: API fetch (FRED, WB, IMF, BLS, OECD), cleaning, panel validation |
| **literature-review** | "literature review", "related work", "文献综述", "相关文献" | Search, summarize, and synthesize economics literature |
| **paper-writing** | "write paper", "draft paper", "论文写作", "学术写作" | Draft economics papers with proper structure, journal conventions, and academic style |
| **beamer-ppt** | "Beamer", "slides", "presentation", "PPT", "幻灯片", "演示文稿" | Professional academic presentations in LaTeX Beamer for conferences and seminars |
| **stata** | "stata code", "stata", "write stata code", "explain stata code" | Generates ready-to-run `.do` files; 4 themed reference guides + community packages (`reghdfe`, `estout`, `csdid`, `rdrobust`) |
| **scrapling** | "scrape", "web data", "crawl", "spider", "网页抓取" | Web scraping toolkit with static, dynamic, and stealthy fetching modes plus spider architecture |

---

## Phase–Tool Mapping

| Phase | Name | Command | Skills | Agents |
|---|---|---|---|---|
| **1** | Research Question | `/question` | — | — |
| **2** | Literature Review | — | `literature-review` | — |
| **3** | Identification Strategy | `/analyze` | `iv-estimation` · `did-analysis` · `rdd-analysis` · `synthetic-control` · `ml-causal` | — |
| **4** | Data Preparation | `/data` | `data-pipeline` · `scrapling` · `results-analysis` | — |
| **5** | Econometric Model | `/model` | estimation skill matching Phase 3 strategy | — |
| **6** | Code Execution | `/code` | `stata` · same as Phase 5 | — |
| **7** | Results & Visualization | `/plot` | `results-analysis` · `table` · `figure` | — |
| **8** | Robustness & Heterogeneity | `/robustness` | same as Phase 5 · `table` · `figure` | `checker` |
| **9** | Full Paper Writing | `/write` | `paper-writing` · `table` · `figure` | — |
| — | Academic Presentation | `/present` | `beamer-ppt` · `paper-writing` | — |

---

## Code Language Support

All skills and commands generate code in:

- **Python** — `statsmodels`, `linearmodels`, `rdrobust`, `arch`, `doubleml`, `econml`
- **R** — `fixest`, `plm`, `AER`, `rdrobust`, `did`, `grf`, `DoubleML`, `modelsummary`, `Synth`
- **Stata** — `reghdfe`, `ivreg2`, `rdrobust`, `csdid`, `ddml`, `synth`, `estout`

---

© All Rights Reserved.
