---
name: robustness
description: Phase 8 Robustness, Heterogeneity & Mechanism Tests. Reads model-spec.md, diagnostic_report.md, and results-memo.md to build a personalised checklist, then runs method-specific robustness checks, heterogeneity analysis, and mechanism tests; generates code; and produces robustness-report.md.
tools:
  - Read
  - Write
  - Bash
---

# /robustness — 稳健性、异质性与机制检验

## 定位

`/robustness` 是实证研究工作流的**第八阶段**，承接 Phase 6（`/code`）的 `diagnostic_report.md` 和 Phase 7（`results-analysis` skill）的 `results-memo.md`，完成三类核心检验：

1. **稳健性检验（Robustness Checks）**：验证主要结论在不同规格、样本和推断方法下的稳定性
2. **异质性分析（Heterogeneity Analysis）**：识别处理效应在不同子群体中的差异
3. **机制检验（Mechanism Tests）**：提供因果机制的间接证据，区分竞争性解释

本阶段产出 `robustness-report.md`，直接供 Phase 9（`/write`）的实证结果和稳健性讨论节使用。

---

## Step 0：读取上游输出，构建个性化检验清单

> 📎 **参见 [`shared/context-reader.md`](../shared/context-reader.md)**
> 本阶段所需文件：`model-spec.md`（必需）、`diagnostic_report.md`（可选）、`results-memo.md`（可选）。

### 0.2 提取关键信息

从各文件中提取以下信息，构建**个性化稳健性检验清单**：

| 来源 | 提取内容 | 用途 |
|------|---------|------|
| `model-spec.md §1` | 识别策略类型 | 选择方法特异性检验菜单 |
| `model-spec.md §3` | 识别假设及诊断状态（✅/⚠️/❌） | ⚠️ 项自动转为稳健性检验 |
| `model-spec.md §8` | 预列的稳健性检验清单 | 直接纳入执行清单 |
| `diagnostic_report.md` | 第二层统计诊断 ⚠️ 项 | 对应的稳健性处理方式 |
| `results-memo.md §6` | Phase 7 建议的优先检验 | 调整执行优先级 |

### 0.3 输出启动确认

向用户输出Phase 8 启动摘要：

```
═══════════════════════════════════════════════════════
Phase 8 启动确认
═══════════════════════════════════════════════════════
识别策略：[策略类型]
主系数：β̂ = [值]（p = [值]）
识别可信度（Phase 7）：[高/中/低]

【已继承的待检验项】（来自上游阶段）
  model-spec.md §8：[N1] 项预列稳健性检验
  diagnostic_report.md ⚠️ 项：[N2] 项统计诊断警告
  results-memo.md §6 优先建议：[N3] 项

【本阶段将执行】
  ① 方法特异性稳健性检验（见 Step 2）
  ② 异质性分析（见 Step 3）
  ③ 机制检验（见 Step 4）
═══════════════════════════════════════════════════════
```

待用户确认后继续，或询问是否有额外的稳健性检验需求。

---

## Step 1：构建稳健性检验主清单

综合三个来源，生成本次执行的**完整稳健性检验清单**，按优先级排序：

```
稳健性检验主清单
─────────────────────────────────────────────────────
编号 | 来源       | 检验类型        | 执行优先级
─────────────────────────────────────────────────────
R1   | 方法通用    | [具体检验名称]  | 高（必须执行）
R2   | model-spec §8 | [具体检验名称] | 高（预列）
R3   | diagnostic ⚠️ | Cook's D 高影响观测值 | 高（已标记）
R4   | results §6  | [建议检验名称]  | 中（建议执行）
R5   | 主动设计    | [检验名称]      | 中（主动发现）
...
─────────────────────────────────────────────────────
```

**主动性原则（Activeness）**：在继承上游清单基础上，主动识别用户未明确提出但**审稿人很可能质疑**的检验点，自动补充到清单并标注"主动设计"。

---

## Step 1.5：写入上下文 → 自动调用 Checker Agent（并行执行）

> **此步骤是 Steps 2–4 的替代入口**。完成检验清单后，将上下文序列化并委托 `checker` agent 并行执行三轨检验，而非在主线程中串行运行 Steps 2/3/4。

### 1.5.1 序列化检验上下文

将 Step 0–1 提取的信息写入 `phase8/context.json`：

```json
{
  "data_path":       "<来自 model-spec.md 的数据路径>",
  "depvar":          "<因变量>",
  "treatment":       "<处理变量>",
  "controls":        ["<控制变量列表>"],
  "fe":              ["<固定效应>"],
  "cluster":         "<聚类层级>",
  "method":          "<识别策略：OLS/IV/DID/RDD/Panel FE>",
  "baseline_coef":   <来自 results-memo.md 的基准系数>,
  "baseline_se":     <基准标准误>,
  "N":               <样本量>,
  "checklist": {
    "robustness":    ["R1: ...", "R2: ...", "R3: ..."],
    "heterogeneity": ["H1: ...", "H2: ..."],
    "mechanism":     ["M1: ...", "M2: ..."]
  }
}
```

`checklist` 字段直接来自 Step 1 构建的个性化清单，供各 subagent 优先执行用户研究特定的检验。

### 1.5.2 读取 Checker Agent 规格

```
Read: agents/checker.md
```

### 1.5.3 按 Checker agent spec 执行并行派发

严格按照 `agents/checker.md` 的 **Step 3** 规定，在**单条消息**中同时发出三个 Agent tool call：

- **Agent A**：稳健性检验 → 写入 `phase8/robustness/`
- **Agent B**：异质性分析 → 写入 `phase8/heterogeneity/`
- **Agent C**：机制检验 → 写入 `phase8/mechanism/`

每个 subagent 的 prompt 应包含：
1. 指向 `phase8/context.json` 的读取指令
2. 来自 `checklist` 字段的个性化检验项（优先于 `agents/checker.md` 中的通用菜单）
3. 来自 Steps 2–4 技术规范的对应方法代码（按 `method` 字段选择对应小节）

### 1.5.4 等待汇总

三路 subagent 完成后，由 checker 执行其 Step 4（汇总 `phase8_report.md`）和 Step 5（呈现给用户确认）。

> **Steps 2–4 以下内容为各 subagent 可调用的技术规范库**，不在主线程中串行执行。

---

## Step 2：方法特异性稳健性检验

根据 `model-spec.md §1` 中的识别策略，执行对应检验菜单。**同一研究中可能同时适用多个策略的检验**（如 DiD + IV 组合设计）。

---

### 2.A OLS / 截面回归稳健性

适用于主方程为 OLS 的情形。

**1. 控制变量敏感性（Oster 2019 系数稳定性）**

逐步添加控制变量，并计算 Oster δ 统计量：

```python
# δ = (β_受控 - β_基准) / (β_受控 - 假设的 OVB 估计量)
# |δ| > 1 表示 OVB 解释全部效应所需的选择性相对可观测性失调程度 > 1，不合理
# 论文惯例：报告 β_受控 在 R² → R²_max = 1.3 × R²_全控 时对应的 δ

delta_oster = (beta_controlled - beta_baseline) / (beta_controlled - 0)
print(f"Oster δ = {delta_oster:.3f}（|δ| > 1 说明 OVB 不足以推翻结论）")
```

**2. 替代标准误**

并排报告 HC1、HC3 和聚类 SE：

```python
# Python — statsmodels
import statsmodels.formula.api as smf

m_ols  = smf.ols(formula, data).fit()
m_hc1  = smf.ols(formula, data).fit(cov_type='HC1')
m_hc3  = smf.ols(formula, data).fit(cov_type='HC3')
m_clus = smf.ols(formula, data).fit(cov_type='cluster',
                                     cov_kwds={'groups': data[cluster_var]})
```

**3. 异常值剔除**

- 剔除结果变量 Y 前后 1% 极端值
- 剔除 `diagnostic_report.md` 中 Cook's D 超阈值的高影响观测值

**4. 函数形式检验**

- RESET 检验（来自 Phase 6 诊断；若 ⚠️ 则此处测试对数变换规格）
- 对数变换：`log(Y)` 和 `log(D)`（若 Y/D 右偏分布）
- 二次项：在主方程中加入 $D^2$，检验非线性效应

**5. 子样本稳健性**

按关键维度分组估计，检验主效应是否由特定子样本驱动：
- 地理维度（如东/西部；高/低收入地区）
- 时间维度（前半段 vs. 后半段样本期）
- 规模维度（大/小企业；高/低收入群体）

---

### 2.B 双重差分（DiD / TWFE）稳健性

适用于主方程为 DiD 或交错 DiD 设计。

**1. 预趋势检验（最高优先级）**

事件研究图 + 预处理期系数联合 F 检验（此项若在 Phase 6 已通过仍须在报告中呈现）：

```stata
* Stata — reghdfe + coefplot
reghdfe Y i.event_time##i.treated controls, ///
    absorb(id year) cluster(id)

* 联合检验预处理期系数 = 0
test [event_time = -4].treated [event_time = -3].treated [event_time = -2].treated
```

**2. 安慰剂处理时点（Placebo in Time）**

将处理时点提前 1–2 年重新估计；安慰剂效应应不显著：

```python
# 将 post 变量提前 1 年（安慰剂）
df['post_placebo'] = (df[time_var] >= treatment_year - 1).astype(int)
df['treat_post_placebo'] = df['treated'] * df['post_placebo']
# 在原始处理时点前的样本中估计，效应应趋于零
```

**3. 替代控制组**

- 限制为处理时点前特征更相似的对照单元（倾向得分匹配后构建控制组）
- 剔除政策可能有溢出效应的相邻单元

**4. 交错 DiD 估计量比较（Staggered Adoption）**

若存在交错处理，对比 TWFE 与异质性稳健估计量：

```r
# R — 对比四种估计量
library(did);  library(fixest);  library(sunab)

# (1) TWFE（可能有负权重偏误）
m_twfe <- feols(Y ~ treat_post | id + year,
                cluster = ~id, data = df)

# (2) Callaway-Sant'Anna
m_cs <- att_gt(yname="Y", tname="year", idname="id",
               gname="first_treated", data=df)
m_cs_agg <- aggte(m_cs, type="simple")

# (3) Sun-Abraham
m_sa <- feols(Y ~ sunab(first_treated, year) | id + year,
              cluster = ~id, data = df)

# (4) Bacon 分解 — 检查负权重比例
library(bacondecomp)
bacon_result <- bacon(Y ~ treat_post, data=df,
                      id_var="id", time_var="year")
```

结果比较标准：若 TWFE 与 C-S / S-A 主系数差异 > 20%，优先使用异质性稳健估计量，并在论文中解释。

**5. 推断方法稳健性**

- 若聚类数 < 30：报告 Wild Cluster Bootstrap p 值

```r
library(fwildclusterboot)
boot_res <- boottest(m_twfe, clustid = "id",
                     param = "treat_post", B = 9999)
print(boot_res)
```

---

### 2.C 工具变量（IV / 2SLS）稳健性

适用于主方程为 2SLS 设计。

**1. 工具变量强度诊断**

```stata
* Stata — ivreg2 完整诊断
ivreg2 Y controls (D = Z), robust ffirst
* 报告：第一阶段 F、Kleibergen-Paap rk Wald F、Cragg-Donald F
* Stock-Yogo (2005) 10% 偏误临界值：16.38（单一 IV）

* 弱工具变量稳健置信区间（Anderson-Rubin）
weakiv
```

**2. 2SLS vs. LIML vs. JIVE 对比**

弱工具变量时，LIML 比 2SLS 具有更好的小样本性质：

```stata
ivregress 2sls  Y controls (D = Z), robust   // 2SLS
ivregress liml  Y controls (D = Z), robust   // LIML
ivregress gmm   Y controls (D = Z), wmatrix(robust) // GMM
```

**3. 排他性限制的间接检验（Reduced Form 对比）**

```stata
* 简约式（Reduced Form）应显著
reg Y Z controls, robust    // 简约式
reg D Z controls, robust    // 第一阶段
* IV 估计量 = 简约式系数 / 第一阶段系数
* 若简约式不显著但 IV 显著，说明工具变量可能无效
```

**4. 安慰剂工具变量**

使用一个理论上应无第一阶段效应的变量代替 Z，验证排他性：
- 若安慰剂 IV 估计量显著，说明主工具变量可能违反排他性

**5. 控制变量集合敏感性**

在第一阶段和第二阶段中分别增减控制变量，检验 IV 估计量的稳定性。

---

### 2.D 断点回归（RDD）稳健性

适用于 Sharp 或 Fuzzy RDD 设计。

**1. 带宽敏感性（最重要的稳健性检验）**

在 MSE 最优带宽的 50%、75%、100%、125%、150% 下并排报告：

```r
# R — rdrobust
library(rdrobust)

bw_opt <- rdbwselect(Y, X, c=cutoff)$bws["mserd","left"]

for (mult in c(0.5, 0.75, 1.0, 1.25, 1.5)) {
    rdd_res <- rdrobust(Y, X, c=cutoff, h=bw_opt*mult)
    cat(sprintf("h = %.2f×opt: β = %.4f (SE = %.4f, p = %.3f)\n",
                mult, rdd_res$coef[1],
                rdd_res$se[3], rdd_res$pv[3]))
}
```

**2. 多项式阶次敏感性**

对比局部线性（p=1）、局部二次（p=2）、局部三次（p=3）：

```r
for (p in 1:3) {
    rdd_res <- rdrobust(Y, X, c=cutoff, p=p)
    cat(sprintf("p=%d: β = %.4f\n", p, rdd_res$coef[1]))
}
```

**3. 核函数敏感性**

三角核（triangular）、矩形核（uniform）、Epanechnikov 核结果对比。

**4. Donut-Hole 检验**

剔除阈值附近最近的 $\delta$ 个单位（$\delta = 1, 2, 3$），检验精确操纵：

```r
# 检验是否有精确堆积在 cutoff 处（刚好等于阈值的观测值）
for (donut in c(1, 2, 3)) {
    df_donut <- df[abs(df$X - cutoff) > donut, ]
    rdd_donut <- rdrobust(df_donut$Y, df_donut$X, c=cutoff)
    cat(sprintf("Donut Δ=%d: β = %.4f\n", donut, rdd_donut$coef[1]))
}
```

**5. 安慰剂截断点（Placebo Cutoffs）**

在真实阈值左右各取 25%、50% 分位点作为虚假阈值，效应应为零：

```r
placebo_cutoffs <- quantile(df$X[df$X < cutoff], c(0.25, 0.5, 0.75))
for (c_fake in placebo_cutoffs) {
    rdd_p <- rdrobust(df$Y[df$X < cutoff],
                      df$X[df$X < cutoff], c=c_fake)
    cat(sprintf("Placebo c=%.2f: β = %.4f (p = %.3f)\n",
                c_fake, rdd_p$coef[1], rdd_p$pv[3]))
}
```

**6. 预定协变量平衡（Covariate Balance at Cutoff）**

对所有预处理协变量做 RDD，估计量应接近零：
- 若协变量不平衡，说明存在操纵或其他处理同时发生

---

### 2.E 面板固定效应稳健性

适用于面板 TWFE 主方程。

**1. 聚类层级比较**

并排报告不同聚类层级的标准误：
- 个体层级聚类 vs. 行业/省级聚类 vs. 双向聚类（个体 + 时间）

**2. 样本时间窗口敏感性**

缩短/延长样本期，检验是否由特定年份驱动：
- 逐年剔除（Jackknife by year）

**3. 逐单位 Jackknife**

逐个剔除个体单元，检验是否存在极端影响的单元：

```python
jackknife_coefs = []
for unit in df[id_var].unique():
    df_jack = df[df[id_var] != unit]
    model_j = run_model(df_jack)  # 调用主方程估计函数
    jackknife_coefs.append(model_j.params[D_var])

import numpy as np
print(f"Jackknife 系数范围：[{min(jackknife_coefs):.4f}, {max(jackknife_coefs):.4f}]")
print(f"Jackknife 均值：{np.mean(jackknife_coefs):.4f}（主系数：{beta_main:.4f}）")
```

**4. 固定效应规格对比**

| 规格 | FE 层级 | β̂ | SE | N |
|------|---------|---|---|---|
| 仅个体 FE | $\alpha_i$ | ... | ... | ... |
| 双向 FE（主规格）| $\alpha_i + \lambda_t$ | ... | ... | ... |
| 个体 + 行业×年 FE | $\alpha_i + \delta_{j,t}$ | ... | ... | ... |

**5. 动态规格（加入滞后因变量）**

若主规格不含滞后因变量，加入 $Y_{i,t-1}$ 检验估计量稳定性（但注意 Nickell 偏误）。

---

### 2.F 合成控制稳健性

适用于合成控制（SCM）主方案。

**1. 供体池敏感性**

- 逐一剔除供体单元，检验合成控制估计是否对特定供体依赖过强（Leave-one-out）
- 扩大/缩小供体池条件（地理 / 经济相似性标准）

**2. 安慰剂检验（Permutation Test）**

对供体池每个单元重复合成控制，计算"虚假处理效应"分布，报告：
- p 值 = 处理单元 MSPE 比 ≥ 该值的供体比例
- 可视化：安慰剂效应路径图（灰色）+ 处理单元路径（黑色加粗）

```r
library(Synth)
placebos <- generate.placebos(synth_out, synth_data, strategy="multisession")
plot_placebos(placebos)
mspe_test(placebos)  # 精确 p 值
```

**3. 预测变量权重矩阵（V 矩阵）敏感性**

分别使用：等权重、主成分确定权重、PCA 降维后的权重，对比合成路径拟合质量（pre-RMSPE）和处理效应。

---

## Step 3：异质性分析（Heterogeneity Analysis）

**异质性分析是 Top-5 期刊近年日益强调的内容**，要求从理论假设出发设计分组，而非事后数据挖掘。

### 3.1 异质性分析设计原则

设计异质性检验前，必须在 `robustness-report.md` 中**先写出理论预测**：

```
异质性分析设计记录
─────────────────────────────────────────────────────
分组变量：[变量名]
理论预测：[为什么该变量应调节处理效应大小或方向？]
预期模式：[预期哪个子组效应更大/更小/方向相反？]
检验方式：[交互项 / 子组分别估计]
─────────────────────────────────────────────────────
```

**无理论支撑的事后分组分析不得纳入论文正文**；若需报告，只能放入附录并标注为"探索性"。

### 3.2 子组分组估计

根据研究背景和理论，选择 2–3 个最重要的分组维度：

```python
# Python — 子组分别估计
heterogeneity_vars = ['group_A', 'group_B', 'group_C']  # 替换为实际分组变量

subgroup_results = {}
for var in heterogeneity_vars:
    for val in df[var].unique():
        df_sub = df[df[var] == val]
        res_sub = run_main_model(df_sub)
        subgroup_results[f"{var}={val}"] = {
            'coef': res_sub.params[D_var],
            'se':   res_sub.bse[D_var],
            'n':    len(df_sub)
        }
```

```stata
* Stata — 分组估计 + 系数对比（附交互项显著性检验）
* 高组
reghdfe Y D controls if group == 1, absorb(id year) cluster(id)
estimates store m_high

* 低组
reghdfe Y D controls if group == 0, absorb(id year) cluster(id)
estimates store m_low

* 交互项检验（形式上等价的 Wald 检验）
reghdfe Y c.D##i.group controls, absorb(id year) cluster(id)
test c.D#1.group  // 交互项是否显著
```

### 3.3 交互项规格（首选方法）

**推荐做法**：在主方程中加入 $D \times \text{Moderator}$ 交互项，而非单独分组估计，以正式检验异质性的统计显著性。

```latex
% 含调节变量的主方程扩展
\begin{equation}\label{eq:heterogeneity}
  Y_{it} = \alpha_i + \lambda_t
           + \beta_1 D_{it}
           + \beta_2 D_{it} \times M_{it}
           + \beta_3 M_{it}
           + \mathbf{X}_{it}'\boldsymbol{\gamma}
           + \varepsilon_{it}
\end{equation}
% M_{it}：调节变量（分组变量）
% \beta_2：异质性处理效应估计量（审稿人重点关注此系数）
% 若 M 为虚拟变量，\beta_1 为 M=0 时的效应；\beta_1+\beta_2 为 M=1 时的效应
```

### 3.4 常见异质性维度（按识别策略推荐）

| 识别策略 | 建议分组维度 |
|---------|-----------|
| DiD（政策评估）| 政策合规强度高/低；处理密度强/弱；首次处理 vs. 后续处理（SUTVA 测试）|
| RDD | 阈值附近更靠近处理区 vs. 控制区（连续度量）；人口密度；城乡差异 |
| IV | Compliers 子群体特征（低/高 first-stage 值）；工具变量强度的空间分布 |
| Panel FE | 行业周期敏感度；企业规模；地区市场化程度 |

---

## Step 4：机制检验（Mechanism Tests）

机制检验的目标是**提供支持特定因果渠道的间接证据**，而非直接证明机制（通常不可能）。

### 4.1 机制检验设计框架

在执行机制检验前，先写出竞争性机制的逻辑链：

```
机制假设分析
═══════════════════════════════════════════════════════
主要机制假设：
  D → [中间变量 M] → Y
  理论基础：[经济学/制度学解释]
  可观测预测：[若此机制成立，应观察到什么额外规律？]

竞争性机制（需要排除）：
  D → Y（直接效应，绕过 M）
  D → [其他中间变量 M'] → Y
  排除方法：[如何区分？]
═══════════════════════════════════════════════════════
```

### 4.2 中介变量分析（Mediation Analysis）

**注意**：社会科学中的"中介分析"通常仅为描述性，因为中介变量本身往往是内生的。应明确声明局限性。

```python
# 三方程中介框架（Baron-Kenny 1986；在论文中标注局限性）

# 方程 1：D → Y（总效应）
total_effect = run_model(Y ~ D + controls)

# 方程 2：D → M（D 对中介变量的影响）
first_stage_M = run_model(M ~ D + controls)

# 方程 3：D + M → Y（条件效应）
direct_effect = run_model(Y ~ D + M + controls)

# 若 β_D 在方程 3 中减小（衰减），说明 M 部分中介 D→Y
# 注意：此处需明确说明中介变量 M 本身的内生性问题
print(f"总效应：{total_effect.params[D_var]:.4f}")
print(f"直接效应（控制M后）：{direct_effect.params[D_var]:.4f}")
print(f"间接效应（近似）：{total_effect.params[D_var] - direct_effect.params[D_var]:.4f}")
```

### 4.3 排除竞争性解释

对每个竞争性机制，设计一个直接检验加以排除：

| 竞争性解释 | 排除检验 | 预期结果（若此解释不成立）|
|----------|---------|----------------------|
| [解释 A] | [检验方法] | [效应应为零/方向相反] |
| [解释 B] | [检验方法] | [效应应为零] |

### 4.4 机制检验的合规性判断

**关键原则**：若中间变量 M 位于 D→Y 的因果路径上（即 M 是处理的结果），**绝对不能**在主回归中控制 M，否则引入中介偏误（Over-control Bias）。机制检验应以独立的回归方程形式呈现。

```
合规性检查
□ 中介变量 M 是否在主回归中被控制？
  → 若是，立即移除！M 在主方程中是碰撞变量或中介变量
□ 机制分析是否声明"描述性证据"局限？
□ 机制回归是否使用与主方程一致的标准误策略？
```

---

## Step 5：代码生成与执行

### 5.1 软件确认

稳健性检验代码与 Phase 6 使用相同软件。若上游文件中未记录，向用户确认：

> *"请确认稳健性检验代码的软件：*
> *A. Python（自动执行）/ B. R（本地运行）/ C. Stata（本地运行）"*

### 5.2 代码文件结构

```
code/
├── 05_robustness.[py|R|do]          # 方法特异性稳健性检验
├── 06_heterogeneity.[py|R|do]       # 异质性分析
└── 07_mechanism.[py|R|do]           # 机制检验
```

每个脚本头部使用与 Phase 6 一致的文件头格式：

```python
# ============================================================
# Project:    [研究问题一句话]
# Phase:      8 — Robustness, Heterogeneity & Mechanism
# Script:     05_robustness.py / 06_heterogeneity.py / 07_mechanism.py
# Strategy:   [识别策略]
# Date:       [YYYY-MM-DD]
# Input:      data/clean/[文件名]，tables/table_main.csv
# Output:     tables/table_robustness.tex/.csv
#             tables/table_heterogeneity.tex/.csv
#             tables/table_mechanism.tex/.csv
#             figures/fig_robustness_coefplot.png
# ============================================================
```

### 5.3 执行规范

**Python（自动执行）**：

```bash
pip install pyfixest linearmodels rdrobust fwildclusterboot doubleml --quiet
python code/05_robustness.py
python code/06_heterogeneity.py
python code/07_mechanism.py
```

**R / Stata（用户本地执行）**：
生成脚本至 `code/` 目录，提示用户在本地运行后上传结果。

### 5.4 稳健性结果验证清单

```
=== Phase 8 验证清单 ===
□ 主系数在稳健性检验中是否保持显著且符号一致？
□ 若某规格下系数发生实质性变化，是否已明确解释原因？
□ 异质性交互项是否有理论支撑（避免数据挖掘）？
□ 机制分析中是否避免控制了中介变量？
□ 稳健性表格是否同时有 .tex 和 .csv 输出？
□ 系数比较图（coefplot）是否已生成？
```

---

## Step 6：产出稳健性汇总表（LaTeX）

### 6.1 稳健性结果表格规范

稳健性表格通常较宽，**使用横向排版（landscape）**，规格为行、数字列（β̂、SE、N）为列：

```latex
\begin{landscape}
\begin{table}[ht]
\centering
\caption{\textbf{Robustness Checks}}
\label{tab:robustness}
\begin{threeparttable}
{\footnotesize\setlength{\tabcolsep}{5pt}
\begin{tabular}{l p{5.5cm} c c c c c}
\toprule
& & \multicolumn{5}{c}{Dependent Variable: \textit{[Y]}} \\
\cmidrule(lr){3-7}
Spec. & Description & $\hat{\beta}$ & SE & $p$-val & $N$ & $R^2$ \\
\midrule
\textbf{(1)} & \textbf{Main specification} & & & & & \\
\midrule
\multicolumn{7}{l}{\textit{Panel A: Standard Error Alternatives}} \\
(2) & HC3 robust SE        & & & & & \\
(3) & Two-way clustered SE & & & & & \\
\midrule
\multicolumn{7}{l}{\textit{Panel B: Sample Restrictions}} \\
(4) & Exclude top/bottom 1\% of $Y$ & & & & & \\
(5) & Exclude high Cook's $D$ obs.  & & & & & \\
\midrule
\multicolumn{7}{l}{\textit{Panel C: Alternative Specifications}} \\
(6) & Log transformation of $Y$ & & & & & \\
(7) & Add quadratic $D^2$        & & & & & \\
\midrule
\multicolumn{7}{l}{\textit{Oster (2019) Coefficient Stability}} \\
\quad $\delta$ &
  \multicolumn{6}{l}{$\delta = [VALUE]$;\quad
    $|\delta| \gg 1$ implies OVB implausibly large to explain result.} \\
\bottomrule
\end{tabular}}
\begin{tablenotes}[flushleft]\small
\item \textit{Notes}: [样本描述]. SE clustered at [level] unless stated.
$^{***}p<0.01$, $^{**}p<0.05$, $^{*}p<0.10$.
\end{tablenotes}
\end{threeparttable}
\end{table}
\end{landscape}
```

**Oster δ 格式规范**（延续已有设计）：

若 δ 文本超过一行，移入 tablenotes 节，正文单元格只放数值：

```latex
\quad $\delta$ & \multicolumn{6}{l}{$\delta = [VALUE]$} \\
% tablenotes 中：
\item Oster (2019) $\delta$: ratio of selection on unobservables to observables
needed to fully explain the result with omitted variable bias.
$|\delta| \gg 1$ implies the result is robust to OVB.
```

### 6.2 异质性结果表格

异质性表格的标准格式：主效应 + 分组效应 + 交互项列：

```latex
% 异质性表格列结构（示例）
\begin{tabular}{lccc}
\toprule
 & Full Sample & High [M] & Low [M] \\
\midrule
$D$ (treatment) & [β̂] & [β̂_H] & [β̂_L] \\
 & ([SE]) & ([SE]) & ([SE]) \\
$D \times$ High [M] & & [β̂_int]$^{**}$ & \\
 & & ([SE]) & \\
\midrule
$N$ & [N] & [N_H] & [N_L] \\
\bottomrule
\end{tabular}
```

### 6.3 系数比较图（Coefplot）

生成稳健性检验主系数及 95% CI 的并排系数图：

```python
# Python — 稳健性系数比较图
import matplotlib.pyplot as plt
import numpy as np

specs = ["Main", "HC3 SE", "Clustered SE", "No Outliers", "Log Y", ...]
coefs = [beta_main, beta_hc3, beta_clus, beta_noout, beta_log, ...]
ci_lo = [ci_lo_main, ...]
ci_hi = [ci_hi_main, ...]

fig, ax = plt.subplots(figsize=(8, 5))
y_pos = np.arange(len(specs))
for i, (spec, coef, lo, hi) in enumerate(zip(specs, coefs, ci_lo, ci_hi)):
    ax.errorbar(coef, i, xerr=[[coef-lo], [hi-coef]],
                fmt='o', ms=6, capsize=4,
                color='#C0392B' if i==0 else '#2C3E50')
ax.axvline(0, color='gray', ls='--', lw=1)
ax.set_yticks(y_pos); ax.set_yticklabels(specs)
ax.set_xlabel("Coefficient (95% CI)")
ax.set_title("Robustness Checks: Main Coefficient")
plt.tight_layout()
plt.savefig("figures/fig_robustness_coefplot.png", dpi=150)
```

```stata
* Stata — coefplot
coefplot (m_main, label("Main")) ///
         (m_hc3,  label("HC3 SE")) ///
         (m_clus, label("Clustered SE")) ///
         (m_nout, label("No Outliers")) ///
         , keep(D_var) xline(0) ///
         title("Robustness Checks: Main Coefficient") ///
         graphregion(color(white))
graph export "figures/fig_robustness_coefplot.png", replace
```

---

## Step 7：产出 `robustness-report.md`

整合本阶段所有结果，写入工作目录：

```
Write: [workspace]/robustness-report.md
```

**文档结构：**

```markdown
# Robustness Report
**项目：** [研究问题一句话]
**版本：** v1.0
**日期：** [YYYY-MM-DD]
**主系数（Phase 6）：** β̂ = [值]（SE = [值]，p = [值]）

---

## 1. 稳健性检验汇总

### 1.1 检验清单完成情况

| 编号 | 检验类型 | 来源 | 结果 | 结论 |
|------|---------|------|------|------|
| R1 | [检验名] | 方法通用 | β̂=[值], p=[值] | ✅ 稳健 / ⚠️ 有变化 / ❌ 不稳健 |
| ... | | | | |

### 1.2 主要结论

[2–3句话：哪些规格下结论稳健？哪些规格下系数有变化及其经济学解释]

**主系数稳健性判断：**
- 符号一致性：[在所有/绝大多数/部分规格下符号一致]
- 量级稳定性：[系数范围 X 至 Y，主规格为 Z]
- 统计显著性：[在所有/绝大多数规格下在 X% 水平显著]

---

## 2. 异质性分析结果

### 2.1 [分组维度 1]

**理论预测：** [预设的异质性方向]
**结果：** [β̂_高 = X vs. β̂_低 = Y；交互项 p = Z]
**解释：** [与理论预测是否一致？如不一致，给出经济学解释]

### 2.2 [分组维度 2]

[同上格式]

---

## 3. 机制检验结果

### 3.1 主要因果渠道

**机制假设：** D → [M] → Y
**支持证据：**
  - D 对 M 的效应：β̂_{D→M} = [值]（p = [值]）[✅ 显著 / ❌ 不显著]
  - 控制 M 后 D 对 Y 的效应：β̂_{直接} = [值]（原始效应：[主系数值]）
  - 间接效应（近似）：[值]，占总效应的 [%]%

**注意事项：** [中介分析的内生性局限性]

### 3.2 竞争性解释排除

| 竞争性解释 | 排除检验 | 结果 | 结论 |
|----------|---------|------|------|
| [解释 A] | [检验] | p = [值] | ✅ 已排除 / ⚠️ 部分支持 |

---

## 4. 产出文件清单

- `tables/table_robustness.tex` / `.csv`
- `tables/table_heterogeneity.tex` / `.csv`
- `tables/table_mechanism.tex` / `.csv` （如适用）
- `figures/fig_robustness_coefplot.png`
- `code/05_robustness.[ext]`
- `code/06_heterogeneity.[ext]`
- `code/07_mechanism.[ext]`（如适用）

---

## 5. Phase 9 写作建议

根据本阶段结果，给出论文 Robustness Section 的写作建议：

- **稳健性结论一句话概括**（可直接用于论文）：
  "[主系数] 在各种稳健性检验中保持稳定，系数范围为 [X, Y]，均在 [Z]% 水平显著（表 X）。"
- **需要重点解释的偏差**：[若某规格系数变化较大，建议在论文中主动解释]
- **异质性发现的定位**：[异质性分析应进主结果节还是附录？]
- **机制讨论的语言强度建议**：[直接机制证据 / 间接相容证据 / 排除竞争解释]
```

---

## Step 8：阶段确认与移交

向用户呈现阶段摘要：

> **Phase 8 产出摘要**
>
> ✅ 稳健性检验代码：`code/05_robustness.[ext]`
> ✅ 异质性分析代码：`code/06_heterogeneity.[ext]`
> ✅ 机制检验代码：`code/07_mechanism.[ext]`（如适用）
> ✅ 稳健性结果表：`tables/table_robustness.tex` + `.csv`
> ✅ 异质性结果表：`tables/table_heterogeneity.tex` + `.csv`
> ✅ 系数比较图：`figures/fig_robustness_coefplot.png`
> ✅ 稳健性报告：`robustness-report.md`
>
> ---
>
> **Phase 8 结论摘要：**
>
> - 稳健性结论：[主系数在 X/Y 项检验中保持一致，符号稳定，量级范围 ...]
> - 最重要的异质性发现：[...]
> - 机制证据：[支持 / 不支持 / 无结论性证据]
>
> 待您确认稳健性结论后，进入 **Phase 9（全文写作）**，运行 `/write` 指令。

---

## 常见问题处理

**Q：某稳健性检验下主系数不再显著，怎么办？**

按以下优先级判断和处理：
1. 先检查规格变化是否引入了经济上合理的不同识别（如带宽缩窄导致样本代表性变化）
2. 若系数量级变化不大但 SE 增大（如小样本子组），诚实汇报而非选择性不报
3. 若系数量级实质缩小，在论文中主动讨论该规格反映了什么不同假设
4. **不得选择性只报告显著结果**（p-hacking），所有预设检验均须报告

**Q：异质性分析发现了意外的差异模式，但缺乏理论支撑，该怎么处理？**

意外的异质性发现有两种处理方式：
1. 若能事后提供合理的经济学机制解释，可纳入论文但必须明确标注"探索性发现"
2. 若无法提供合理解释，建议只在附录中报告，正文不作过度解读

**Q：机制分析中的中介变量本身是内生的，还有意义吗？**

有意义，但需在论文中清晰声明局限性。标准做法是：
1. 说明中介分析仅提供"与机制一致的描述性证据"
2. 若中介变量有外生变异来源，可进一步使用 IV 或 DiD 验证中介路径
3. 若无法解决中介变量内生性，重新定位该检验为"排除竞争性解释"（检验 D→M 是否显著）而非完整中介分析

---

## 与其他指令/技能的衔接

- **上游**：`/code`（Phase 6，产出 `diagnostic_report.md`）→ `results-analysis` skill（Phase 7，产出 `results-memo.md`）
- **本阶段调用的技能**：
  - `did-analysis` — 交错 DiD 估计量比较（Callaway-Sant'Anna、Sun-Abraham）
  - `iv-estimation` — IV 稳健性（弱工具变量检验、LIML、Anderson-Rubin CI）
  - `rdd-analysis` — RDD 稳健性（带宽敏感性、安慰剂截断点）
  - `panel-data` — 面板稳健性（聚类层级比较、Jackknife）
  - `ml-causal` — 异质性处理效应的机器学习估计（可选）
- **下游**：`/write`（Phase 9，读取 `robustness-report.md` 写作稳健性节）
