# Econometrics Plugin

A comprehensive econometrics plugin for end-to-end empirical research automation, covering the complete workflow from research question scoping, literature review, and identification strategy design, through data fetching and cleaning, econometric modeling, code generation, results analysis, table and figure production, full-paper writing, and conference presentation.

## File Structure

```
econometrics/
├── .claude-plugin/
│   └── plugin.json              # Plugin metadata (name, version, author)
├── CLAUDE.md                    # Agent system prompt & 9-phase workflow
├── .env                         # Environment variables (API keys)
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
    │     └── zotero-setup.md   # Zotero MCP installation & configuration guide
    ├── scrapling/               # Web scraping toolkit for custom data collection
    │
    ├── — Writing —
    ├── paper-writing/           # Full paper drafting to Top-5 journal standards
    │
    └── — Software Reference —
        └── stata/               # Generates Stata .do files for manual execution;
    
```

## Skills

| Skill | Triggers | Description |
|-------|----------|-------------|
| **ols-regression** | "OLS", "linear regression", "最小二乘法", "线性回归", "跑回归" | OLS estimation, assumption testing, heteroskedasticity, robust SE, coefficient interpretation |
| **time-series** | "ARIMA", "unit root", "Granger causality", "时间序列", "平稳性" | Stationarity tests (ADF/KPSS), ARIMA, VAR/VECM, cointegration, forecasting |
| **panel-data** | "fixed effects", "Hausman test", "面板数据", "固定效应" | FE/RE models, two-way FE, clustered SE, dynamic panels |
| **iv-estimation** | "instrumental variables", "2SLS", "工具变量", "内生性", "两阶段最小二乘法" | IV/2SLS, first-stage diagnostics, weak instruments, Bartik instruments, judge designs, PSM |
| **did-analysis** | "difference-in-differences", "DID", "parallel trends", "双重差分", "平行趋势" | DID/TWFE, event study, staggered DID (CS, SA, BJS, dCDH), pre-trends power analysis |
| **rdd-analysis** | "regression discontinuity", "RDD", "断点回归", "带宽选择" | Sharp/fuzzy RDD, bandwidth selection, validity tests, discrete RV, multi-cutoff/multi-score RDD |
| **synthetic-control** | "synthetic control", "SCM", "合成控制", "合成控制法" | Abadie-Diamond-Hainmueller SCM, augmented SCM, synthetic DID, placebo inference, MSPE ratios |
| **ml-causal** | "causal forest", "double ML", "DML", "GRF", "因果森林", "双重机器学习" | Causal Forest (GRF), Double/Debiased ML, LASSO variable selection, CATE, BLP/CLAN analysis |
| **results-analysis** | "summary statistics", "Table 1", "balance table", "interpret results", "economic significance", "描述性统计", "平衡性检验", "结果解读" | Descriptive stats, balance tests, coefficient interpretation, effect sizes, causal language calibration, results memos |
| **table** | "regression table", "LaTeX table", "esttab", "modelsummary", "回归表格" | Multi-model regression tables, multi-panel layouts, journal-specific formatting (AER, QJE) |
| **figure** | "publication figure", "event study plot", "binscatter", "coefficient plot", "论文图表" | Journal-quality figures (AER/QJE standards): event study, binscatter, RDD, density, coefplot |
| **data-pipeline** | "FRED", "World Bank", "fetch data", "clean data", "merge datasets", "获取数据", "数据清洗" | End-to-end data pipeline: API fetch (FRED, WB, IMF, BLS, OECD), cleaning, variable construction, panel validation |
| **literature-review** | "literature review", "related work", "文献综述", "相关文献" | Search, summarize, and synthesize economics literature; identify gaps and organize findings. **Zotero integration**: auto-searches your local Zotero library before querying external APIs, and imports the final reference list into a new Zotero collection (requires [zotero-mcp](skills/literature-review/zotero-setup.md)) |
| **paper-writing** | "write paper", "draft paper", "论文写作", "学术写作" | Draft economics papers with proper structure, journal conventions, and academic style |
| **beamer-ppt** | "Beamer", "slides", "presentation", "PPT", "幻灯片", "演示文稿" | Create professional academic presentations in LaTeX Beamer for conferences and seminars |
| **stata** | "stata code", "stata", "write stata code", "explain stata code" | Generates ready-to-run `.do` files for manual execution in Stata (no CLI/MCP integration); 4 themed reference guides (basics, methods, data-mgmt, output) + community packages (reghdfe, estout, csdid, rdrobust, etc.) |
| **scrapling** | "scrape", "web data", "crawl", "spider", "网页抓取" | Web scraping toolkit with static, dynamic, and stealthy fetching modes plus spider architecture for large-scale data collection |

## Commands

| Command | Phase | Description |
|---------|-------|-------------|
| `/question` | 1 | Transform a research idea into a clear, well-scoped research question |
| `/analyze` | 3 | Diagnose endogeneity threats, evaluate and select an identification strategy; produce an Identification Strategy Memo |
| `/data` | 4 | Fetch, clean, and explore data; produce a Data Report |
| `/model` | 5 | Specify the formal econometric model with LaTeX equations and SE strategy; produce model-spec.md |
| `/code` | 6 | Generate and execute a reproducible analysis script in Python, R, or Stata |
| `/plot` | 7 | Audit and upgrade all tables and figures to top-journal standards |
| `/robustness` | 8 | Design and run robustness checks, heterogeneity analysis, and mechanism tests |
| `/write` | 9 | Draft any section or the complete paper to Top-5 journal standards; compile LaTeX |
| `/present` |  | Generate a fully compilable LaTeX Beamer slide deck from all upstream outputs |

## Code Language Support

All skills and commands generate code in:
- **Python** — using `statsmodels`, `linearmodels`, `rdrobust`, `arch`, `doubleml`, `econml`
- **R** — using `fixest`, `plm`, `AER`, `rdrobust`, `did`, `grf`, `DoubleML`, `modelsummary`, `Synth`
- **Stata** — using standard and SSC packages (`reghdfe`, `ivreg2`, `rdrobust`, `csdid`, `ddml`, `synth`)
