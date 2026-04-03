---
name: code
description: Phase 6 Code Generation & Execution. Reads identification-memo.md, data-report.md, and model-spec.md, asks user to select software (Python / R / Stata), dispatches to the appropriate estimation skill, generates a reproducible analysis script with main regression, diagnostics, and output export, then executes and verifies results.
tools:
  - Read
  - Write
  - Bash
---

# /code — 代码执行与复现

## 定位

`/code` 是实证研究工作流的**第六阶段**，承接 Phase 5（`/model`）的 `model-spec.md`，将形式化的计量模型转化为可执行、可复现的分析代码。核心流程：

1. 读取上游文档，提取估计所需的全部参数
2. 用户选择目标统计软件（Python / R / Stata）
3. 根据识别策略 × 软件，调用对应的**估计技能**生成代码
4. 代码执行 + 结果验证
5. 输出规范化的 `.py` / `.R` / `.do` 脚本至 `code/`，结果表格至 `tables/`

---

## Step 0：读取上游输出

> 📎 **参见 [`shared/context-reader.md`](../shared/context-reader.md)**
> 本阶段所需文件：`identification-memo.md`（必需）、`data-report.md`（必需）、`model-spec.md`（必需）。

提取以下信息用于代码生成：

| 来源 | 提取内容 |
|------|---------|
| `model-spec.md` | 识别策略类型、主方程（LaTeX 转为代码描述）、Y/D/Z/控制变量名称、FE 层级、SE 类型与聚类变量、目标参数 |
| `data-report.md` | 清洗后数据路径（`data/clean/[project_name]_clean.[dta\|parquet]`）、样本量、面板结构、数据质量预警 |
| `identification-memo.md` | 识别假设的可检验形式（决定生成哪些诊断代码） |

提取完成后向用户确认：

> **Phase 6 启动确认**
>
> 识别策略：**[策略]**
> 主方程：$[LaTeX 概述]$
> 数据路径：`data/clean/[文件名]`
> 标准误：**[类型，聚类变量]**

---

## Step 1：软件选择

向用户呈现软件选项：

> *"请选择生成代码所使用的统计软件：*
>
> - **A. Python** ⚡ 推荐 — 代码生成后**自动在沙箱中执行**，结果即时可见
> - **B. R** — 生成 `.R` 脚本，**需在本地安装 R 后手动运行**
> - **C. Stata** — 生成 `.do` 脚本，**需在本地安装 Stata 后手动运行**
>

等待用户确认后进入 Step 2。

> **注**：选择 B 或 C 时，代码文件生成至 `code/` 目录后，Claude 不会在沙箱中执行；请用户下载脚本后在本地环境运行，运行结果可再上传供后续分析。

---

## Step 2：策略 × 软件派发

根据 `identification_strategy` × 软件选择，**调用对应的估计技能**，并传入 Step 0 提取的完整上下文。

### 2.1 派发表

| 识别策略 | 调用的估计技能 | Python 核心包 | R 核心包 | Stata 核心命令 |
|---------|-------------|-------------|---------|--------------|
| DiD / TWFE | `did-analysis` | `linearmodels`, `pyfixest` | `fixest`, `did` | `reghdfe`, `eventstudyinteract` |
| Event Study（交错 DiD）| `did-analysis` | `pyfixest`, `doubleml` | `fixest`, `did`, `sunab` | `reghdfe`, `did_imputation`, `csdid` |
| Sharp / Fuzzy RDD | `rdd-analysis` | `rdrobust` | `rdrobust`, `rddensity` | `rdrobust`, `rddensity` |
| 2SLS / IV | `iv-estimation` | `linearmodels` (`IV2SLS`) | `ivreg`, `AER` | `ivregress`, `ivreg2`, `ranktest` |
| 面板固定效应 | `panel-data` | `linearmodels` (`PanelOLS`) | `fixest`, `plm` | `reghdfe`, `xtreg` |
| 合成控制 | `synthetic-control` | `pysynth`, `synth_runner` | `Synth`, `SCtools` | `synth`, `synth_runner` |
| OLS / 截面回归 | `ols-regression` | `statsmodels`, `linearmodels` | `lm`, `estimatr` | `regress`, `reghdfe` |
| 双重机器学习 | `ml-causal` | `doubleml`, `econml` | `DoubleML`（R 包）| 无原生支持，建议用 Python |
| 时间序列 / VAR | `time-series` | `statsmodels` | `vars`, `urca` | `var`, `vecm`, `arima` |

### 2.2 技能调用格式

> *"调用 `[估计技能名]`，生成 [软件名] 代码。传入参数：*
> - 策略：[识别策略]
> - 主方程：[文字描述]
> - 因变量：`[Y_var]`
> - 处理变量：`[D_var]`
> - 识别变量：`[Z_var（若有）]`
> - 控制变量：`[control_vars 列表]`
> - 个体/时间变量：`[id_var]` / `[time_var]`
> - FE 层级：[个体 + 时间 / 仅个体 / 省 × 年...]
> - SE 类型：[聚类层级]
> - 数据路径：`data/clean/[文件名]`"*

### 2.3 Stata 专项处理

选择 Stata 时，除调用策略对应的估计技能外，**同时调用 `stata` skill** 确保 do-file 格式符合最佳实践（标准文件头、全局路径宏、日志文件、`assert` 验证、版本声明）。

`stata` skill 负责提供：
- `00_master.do` 的标准结构
- `reghdfe` / `ivreg2` / `rdrobust` 的安装命令
- `estout` / `coefplot` 的结果导出代码
- Stata 版本兼容性注意事项

---

## Step 3：代码质量标准

无论选择哪种软件，生成的代码必须满足以下要求，缺一不可：

### 3.1 文件头（Header）

每个脚本顶部必须包含：

```python
# ============================================================
# Project:    [研究问题一句话]
# Phase:      6 — Main Estimation
# Strategy:   [识别策略]
# Software:   Python [version] / R [version] / Stata [version]
# Author:     [project_name]
# Date:       [YYYY-MM-DD]
# Input:      data/clean/[文件名]
# Output:     tables/[结果表格], figures/[系数图]
# Model:      [主方程的文字描述，对应 model-spec.md §2]
# ============================================================
```

### 3.2 路径与环境

```python
# Python
import os
from pathlib import Path

ROOT = Path(__file__).parent.parent   # 项目根目录
DATA = ROOT / "data" / "clean"
OUT_TABLES  = ROOT / "tables"
OUT_FIGURES = ROOT / "figures"
OUT_TABLES.mkdir(exist_ok=True)
OUT_FIGURES.mkdir(exist_ok=True)

data_path = DATA / "[project_name]_clean.parquet"
```

```r
# R
root  <- here::here()   # 需 here 包
data_path   <- file.path(root, "data", "clean", "[project_name]_clean.rds")
out_tables  <- file.path(root, "tables");  dir.create(out_tables,  showWarnings=FALSE)
out_figures <- file.path(root, "figures"); dir.create(out_figures, showWarnings=FALSE)
```

```stata
* Stata
version 17
clear all
set more off
cap log close

global root    "[workspace 绝对路径]"
global data    "$root/data/clean"
global tables  "$root/tables"
global figures "$root/figures"

cap mkdir "$tables"
cap mkdir "$figures"

log using "$root/logs/main_estimation_`c(current_date)'.log", replace
```

### 3.3 数据加载 + 样本确认

```python
# 加载并报告基本维度
df = pd.read_parquet(data_path)
print(f"样本：{len(df):,} 行 × {df.shape[1]} 列")
print(f"时间范围：{df[time_var].min()} – {df[time_var].max()}")
print(f"个体数：{df[id_var].nunique()}")

# 与 data-report.md 中的样本量核对
assert len(df) == EXPECTED_N, f"样本量不符：预期 {EXPECTED_N}，实际 {len(df)}"
```

### 3.4 主回归（来自估计技能）

- 严格按照 `model-spec.md` §2 的方程规格，不擅自增减变量
- 使用 §4 指定的 SE 类型和聚类层级
- 代码注释中注明每个参数对应方程中的哪个符号（如 `# β: ATT estimate`）

### 3.5 诊断代码（执行 `model-spec.md §6` 的诊断计划）

诊断的**规格决策**（检验什么、通过标准、失败处理）已在 Phase 5（`/model` Step 3.5）中确定，写入 `model-spec.md §6`。本步骤的唯一职责是**按规格执行**：

1. 读取 `model-spec.md §6.1`（识别假设诊断规格）和 `§6.2`（统计假设诊断规格）
2. 生成 `code/02_diagnostics.[ext]`，严格按照 §6 指定的检验项目、通过标准和失败处理逻辑编写代码
3. 不得在代码阶段自行增减检验项目或修改通过标准

**代码生成规则：**
- §6.1 每一行 → 生成对应的识别诊断代码（`rddensity`、预趋势 F 检验、Hausman 等）
- §6.2 每一行 → 生成对应的统计诊断代码（VIF、BP、BG/Wooldridge、RESET、Cook's D）
- 若 §6 某行标注"失败则终止"→ 代码中加 `assert`（Python）/ `stop()`（R）/ `error`（Stata），失败时自动中断并打印返回建议
- 若 §6 某行标注"继续" → 代码记录 WARN 后继续执行，不中断

**各软件的诊断包对照：**

| 检验 | Python | R | Stata |
|------|--------|---|-------|
| VIF | `statsmodels.variance_inflation_factor` | `car::vif` | `estat vif` |
| 异方差 | `statsmodels.het_breuschpagan` | `lmtest::bptest` | `estat hettest` |
| 序列相关（截面）| `statsmodels.acorr_breusch_godfrey` | `lmtest::bgtest` | `estat bgodfrey` |
| 序列相关（面板）| `linearmodels` Wooldridge | `plm::pwartest` | `xtserial` |
| 截面相关 | `linearmodels` Pesaran CD | `plm::pcdtest` | `xtcsd` |
| RESET | `statsmodels.linear_reset` | `lmtest::resettest` | `estat ovtest` |
| Cook's D | `statsmodels.OLSInfluence` | `cooks.distance` | `predict cooksd` |
| McCrary | `rddensity` Python port | `rddensity::rddensity` | `rddensity` |
| 预趋势联合检验 | `pyfixest.wald_test` | `fixest::wald` | `test`（联合 F） |
| 第一阶段 F | `pyfixest` / `linearmodels` | `AER::ivreg` diagnostics | `estat firststage` |
| Hausman | `linearmodels.compare` | `plm::phtest` | `hausman` |

### 3.6 结果输出

> 📎 **输出格式规范参见 [`shared/output-standards.md`](../shared/output-standards.md)**

所有模型结果必须同时导出：

**回归系数表（`.tex` + `.csv`）：**

```python
# Python：使用 pyfixest 或 statsmodels 的 summary_col
from pyfixest.summarize import etable
etable([model_main, model_control1, model_control2],
       type="tex",
       file=str(OUT_TABLES / "table_main.tex"))
```

```r
# R：使用 modelsummary 或 fixest 的 etable
library(modelsummary)
modelsummary(list("Baseline"=m1, "Controls"=m2, "Full"=m3),
             stars=c("*"=0.1, "**"=0.05, "***"=0.01),
             gof_omit="AIC|BIC|Log",
             output=file.path(out_tables, "table_main.tex"))
```

```stata
* Stata：使用 esttab（estout 包）
esttab m1 m2 m3 using "$tables/table_main.tex", ///
    b(3) se(3) star(* 0.10 ** 0.05 *** 0.01) ///
    booktabs replace label ///
    mtitles("Baseline" "Controls" "Full") ///
    keep(`D_var') stats(N r2, fmt(0 3) labels("Obs." "R²"))
```

**系数图（`.png`）：**

```python
# Python：matplotlib + 回归结果
fig, ax = plt.subplots(figsize=(7, 4))
coef = result.params[D_var]
ci_lo, ci_hi = result.conf_int().loc[D_var]
ax.errorbar(0, coef, yerr=[[coef - ci_lo], [ci_hi - coef]],
            fmt="o", ms=8, capsize=5, color="#C0392B")
ax.axhline(0, ls="--", lw=1, color="gray")
ax.set_title(f"Main Estimate: Effect of {D_var} on {Y_var}")
plt.savefig(OUT_FIGURES / "coef_main.png", dpi=150, bbox_inches="tight")
```

```stata
* Stata：coefplot 包
coefplot m1 m2 m3, keep(`D_var') ///
    xline(0) msymbol(D) ///
    title("Main Estimates") ///
    xtitle("Coefficient (95% CI)") ///
    graphregion(color(white))
graph export "$figures/coef_main.png", replace
```

### 3.7 可复现性脚注

每个脚本末尾：

```python
# Python
import sys, platform
print(f"\n{'='*50}")
print(f"Python {sys.version}")
print(f"Platform: {platform.platform()}")
print(f"Date: {pd.Timestamp.now()}")
import importlib
for pkg in ["pandas", "numpy", "statsmodels", "linearmodels", "pyfixest"]:
    try:
        v = importlib.import_module(pkg).__version__
        print(f"  {pkg}: {v}")
    except Exception:
        pass
```

```r
# R
sessionInfo()
```

```stata
* Stata
which reghdfe
which rdrobust
di "Stata version: `c(stata_version)'"
di "Date: `c(current_date)'"
log close
```

---

## Step 4：代码文件结构

所有代码文件保存至 `code/` 目录，按编号命名，保持可单独运行：

```
code/
├── 00_master.[py|R|do]         # 主运行脚本：依次调用 01–04
├── 01_main_estimation.[py|R|do] # 主回归（对应 model-spec.md §2 主方程）
├── 02_diagnostics.[py|R|do]     # 识别假设诊断（§3.5 列出的全部检验）
├── 03_event_study.[py|R|do]     # 事件研究 / 动态效应（DiD/RDD 专用）
└── 04_output_tables.[py|R|do]   # 结果表格与系数图生成
```

> **说明**：若主回归、诊断和输出逻辑可在 100 行内完成，允许合并为单文件 `01_main_estimation.[py|R|do]`；复杂设计（交错 DiD、Fuzzy RDD、多种稳健性）应拆分文件。

**`00_master` 模板：**

```python
# 00_master.py — 按顺序运行全部分析脚本
import subprocess, sys

scripts = [
    "code/01_main_estimation.py",
    "code/02_diagnostics.py",
    "code/03_event_study.py",
    "code/04_output_tables.py",
]

for s in scripts:
    print(f"\n{'='*50}\n运行：{s}\n{'='*50}")
    result = subprocess.run([sys.executable, s], check=True)
```

```stata
* 00_master.do
do "$root/code/01_main_estimation.do"
do "$root/code/02_diagnostics.do"
do "$root/code/03_event_study.do"
do "$root/code/04_output_tables.do"
```

---

## Step 5：执行与验证

### 5.1 执行代码

**Python — 沙箱自动执行**

代码生成后，Claude 直接在沙箱中安装依赖并运行，无需用户任何操作：

```bash
pip install pyfixest linearmodels rdrobust doubleml pyreadstat --quiet
python code/00_master.py
```

执行完成后，结果表格（`tables/`）和图形（`figures/`）即时可见，Claude 将在对话中呈现关键数值并进行核查。

---

**R — 生成脚本，用户本地运行**

Claude 生成 `.R` 脚本至 `code/` 目录，**不在沙箱中执行**。用户下载后在本地 R 环境运行：

```r
# 本地 R 中：先安装依赖包（仅首次需要）
install.packages(c("fixest", "rdrobust", "ivreg", "modelsummary",
                   "here", "AER", "did", "Synth"),
                 repos = "https://cloud.r-project.org")

# 运行主脚本
source("[workspace]/code/00_master.R")
```

> *"R 脚本已生成至 `code/` 目录。请在本地 R（≥ 4.1）环境中运行 `00_master.R`。如需将运行结果（表格 / 图形）上传，我可继续协助核查和解读。"*

---

**Stata — 生成脚本，用户本地运行**

Claude 生成 `.do` 脚本至 `code/` 目录，**不在沙箱中执行**。用户下载后在本地 Stata 中运行：

> *"Stata do-file 已生成至 `code/` 目录。请在本地 Stata（≥ 15）中运行：*
> ```stata
> do "[workspace]/code/00_master.do"
> ```
> *首次运行前请先安装所需包（联网状态下在 Stata 命令窗口执行）：*
> ```stata
> ssc install reghdfe
> ssc install ftools
> ssc install ivreg2            // IV 策略
> ssc install rdrobust          // RDD 策略
> ssc install synth             // 合成控制
> ssc install eventstudyinteract // Event Study
> ssc install csdid             // Callaway-Sant'Anna
> ssc install estout            // 结果导出
> ssc install coefplot          // 系数图
> ```
> *如需将运行结果（日志 / 表格 / 图形）上传，我可继续协助核查和解读。"*

### 5.2 诊断报告生成与结果验证

代码执行后，汇总两层诊断结果，生成结构化报告并验证关键数值。

**生成 `diagnostic_report.md`（保存至项目根目录）：**

```
═══════════════════════════════════════════════════════
              MODEL DIAGNOSTIC REPORT
═══════════════════════════════════════════════════════
Project:  [project_name]
Strategy: [识别策略]
Model:    [Y_var] = f([D_var], [controls]) + [FE 层级]
N:        [样本量]
Date:     [YYYY-MM-DD]
═══════════════════════════════════════════════════════

【第一层：识别假设诊断】
────────────────────────────────────────────────────────
 检验                    统计量        结论
────────────────────────────────────────────────────────
 [策略对应的识别检验]    [值]          ✅ PASS / ⚠️ WARN / ❌ FAIL
────────────────────────────────────────────────────────

【第二层：统计假设诊断】
────────────────────────────────────────────────────────
 VIF（最大值）           [值]          ✅ / ❌
 Breusch-Pagan           p = [值]      ✅ / ⚠️（已用稳健 SE）
 Breusch-Godfrey         p = [值]      ✅ / ❌
 RESET                   p = [值]      ✅ / ❌
 Cook's D（最大值）      [值]          ✅ / ⚠️
────────────────────────────────────────────────────────

【诊断结论与建议】
────────────────────────────────────────────────────────
 ❌ 识别假设失败
    → 立即终止，返回 Phase [4/5] 修正：[具体说明]

 ⚠️ 异方差（已用聚类 SE，可接受）
 ✅ 无多重共线性问题
 ✅ 无序列相关
 ⚠️ Cook's D 超阈值（[N 个]高影响观测值）
    → 已纳入 Phase 8 稳健性检验：去除高影响值后重估
════════════════════════════════════════════════════════
```

**结果验证清单（逐项确认）：**

```
=== Phase 6 验证清单 ===
□ 主系数方向与 EDA 阶段的方向一致？
□ 样本量与 data-report.md 记录的 N 一致？
□ 标准误类型与 model-spec.md §4 一致（聚类层级正确）？

□ 【第一层识别诊断全部通过？】
  □ DiD：预趋势系数联合检验 p > 0.1
  □ RDD：McCrary 检验 p > 0.1；协变量无显著跳跃
  □ IV：一阶段 F > 10；J-test p > 0.05（过识别时）
  □ FE：Hausman 检验支持 FE 选择
  □ SC：处理单位预处理期 RMSPE 在捐赠者分布中居中

□ 【第二层统计诊断无阻断性问题？】
  □ VIF < 10（所有控制变量）
  □ 异方差：若 BP 失败，确认已使用稳健/聚类 SE
  □ RESET：p > 0.05，或有经济理论支持当前函数形式
  □ Cook's D：高影响值已列出并纳入稳健性

□ 结果表格已导出至 tables/（.tex + .csv）？
□ 系数图已导出至 figures/（.pdf + .png）？
□ diagnostic_report.md 已保存至项目根目录？
```

**若识别假设失败（❌）**：立即停止，不进入 Phase 7，明确告知用户返回哪个阶段修正，原因是什么。

**若统计假设存在 ⚠️**：继续流程，但将问题自动写入 diagnostic_report.md 的"建议"节，Phase 8 的 `/robustness` 命令读取该报告时将自动纳入对应的稳健性检验。

---

## Step 6：阶段确认与移交

向用户呈现阶段摘要：

> **Phase 6 产出摘要**
>
> ✅ 主回归代码：`code/01_main_estimation.[ext]`
> ✅ 诊断代码：`code/02_diagnostics.[ext]`
> ✅ 事件研究代码：`code/03_event_study.[ext]`（如适用）
> ✅ 输出脚本：`code/04_output_tables.[ext]`
> ✅ 主运行脚本：`code/00_master.[ext]`
> ✅ 主回归结果表：`tables/table_main.tex` + `.csv`
> ✅ 系数图：`figures/coef_main.png`
>
> ---
>
> **关键结果摘要：**
> - 主系数 $\hat{\beta}$ = [值]（SE = [值]，p = [值]）
> - 识别假设诊断：[通过 / 存疑，具体说明]
> - 样本量：N = [值]
>
> 代码与结果验证通过后，自动调用 `results-analysis` skill，进入 **Phase 7（结果分析与解释）**，产出 `results-memo.md`。
