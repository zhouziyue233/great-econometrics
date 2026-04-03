---
name: model
description: Phase 5 Econometric Model Construction. Reads identification-memo.md and data-report.md, writes formal model specification with LaTeX equations, discusses identification assumptions and SE strategy, calls the appropriate estimation skill, and produces model-spec.md.
tools:
  - Read
  - Write
  - Bash
---

# /model — 计量模型构建

## 定位

`/model` 是实证研究工作流的**第五阶段**，承接 Phase 3（`/analyze`）的 `identification-memo.md` 和 Phase 4（`/data`）的 `data-report.md`，完成：

1. 根据识别策略**形式化写出计量模型**（含完整 LaTeX 公式）
2. 明确**识别假设的可检验形式**及其在数据中的诊断状态
3. 确定**标准误策略**（聚类层级、异方差处理方式等）
4. 调用对应的**估计技能**执行回归
5. 产出 `model-spec.md`，供 Phase 6（代码执行）和 Phase 9（论文写作）使用

---

## Step 0：读取上游输出

> 📎 **参见 [`shared/context-reader.md`](../shared/context-reader.md)**
> 本阶段所需文件：`identification-memo.md`（必需）、`data-report.md`（可选）。

读取完成后提取并整理以下关键信息：

| 来源 | 提取内容 |
|------|---------|
| `identification-memo.md` | 识别策略类型、结果变量 Y、处理变量 D、识别变量 Z、控制变量列表、面板结构（id/time）、目标参数（ATE/ATT/LATE） |
| `data-report.md` | 样本量 N、面板维度（个体数 × 时期数）、关键变量缺失率、EDA 预警（平衡性不足、弱工具变量等）、平衡性检验结果 |

提取完成后向用户确认：

> **Phase 5 启动确认**
>
> 识别策略：**[策略名称]**
> 目标参数：**[ATE / ATT / LATE / 平均处理效应...]**
> 结果变量 Y：`[变量名]`
> 处理变量 D：`[变量名]`
> 识别变量 Z / 运行变量：`[变量名（如适用）]`
> 样本：[N] 观测，[个体数] 个个体 × [时期数] 期
>
> 继续构建模型，或告知是否需要调整。

---

## Step 1：模型类型与规格讨论

在形式化写出方程前，与用户确认以下**建模决策**。每个决策点需给出推荐选项及经济学理由：

### 1.1 主方程类型

| 识别策略 | 推荐主方程类型 | 备选 |
|---------|--------------|------|
| DiD（2×2 或少期） | 双向固定效应 TWFE | 加权 DiD、有控制变量的 OLS |
| DiD（交错、多期） | Callaway–Sant'Anna / Sun–Abraham | TWFE（需诊断异质性处理效应偏误） |
| RDD（精确服从） | Sharp RDD，局部线性 | 高阶多项式（通常不推荐） |
| RDD（模糊服从） | Fuzzy RDD（2SLS at cutoff） | — |
| IV | 2SLS（线性第一阶段） | LIML（弱工具变量时稳健） |
| 面板固定效应 | TWFE（个体 + 时间） | 随机效应（需 Hausman 检验） |
| 合成控制 | 加权合成控制（Abadie 2003） | 合成 DiD（Arkhangelsky 2021） |
| 截面 OLS | OLS + 稳健 SE | — |

### 1.2 控制变量决策

参考 `identification-memo.md` 的协变量列表和 `data-report.md` 的平衡性检验结果：

- 平衡性检验标准化差异 > 0.25 的变量：**必须列为重点控制变量**
- 理论上影响 Y 但与 D 无关的变量：加入可提升精度（降低残差方差），建议加入
- 与 D 相关且位于处理路径上的中介变量：**不得控制**（会引入中介偏误）

### 1.3 固定效应层级

向用户说明各层级固定效应控制的内容：

| 固定效应 | 控制内容 | 代价 |
|---------|---------|------|
| 个体 FE（$\alpha_i$） | 个体层面不随时间变化的遗漏变量 | 吸收所有个体层面截面变异 |
| 时间 FE（$\lambda_t$） | 共同时间趋势（商业周期、通货膨胀等） | 吸收所有时期层面共同冲击 |
| 个体 × 时间组合 FE | 更细粒度的组别趋势 | 大幅降低自由度 |
| 省 × 年 FE | 控制省级层面年度时变混淆 | 需要足够跨省 within 变异 |

---

## Step 2：形式化模型写作（LaTeX）

根据 Step 1 确认的策略类型，**输出完整的 LaTeX 模型规格**。每个模型必须包含：

① 主方程
② 下标与符号说明
③ 关键假设的形式化表述
④ 目标参数的经济学解释
⑤ 识别条件

---

### 模型 A：双向固定效应 DiD（TWFE）

**适用**：二元处理、平行趋势假设成立、处理效应同质或近似同质。

```latex
% ── 主方程 ──────────────────────────────────────────
\begin{equation}\label{eq:twfe}
  Y_{it} = \alpha_i + \lambda_t + \beta \, D_{it}
           + \mathbf{X}_{it}'\boldsymbol{\gamma} + \varepsilon_{it}
\end{equation}

% ── 符号说明 ─────────────────────────────────────────
% Y_{it}              结果变量（个体 i，时期 t）
% \alpha_i            个体固定效应（吸收不随时间变化的遗漏变量）
% \lambda_t           时间固定效应（吸收共同时间趋势）
% D_{it}              处理变量：= 1 若个体 i 在 t 期已受处理，否则 = 0
% \mathbf{X}_{it}     时变控制变量向量
% \varepsilon_{it}    误差项，假设 E[\varepsilon_{it} \mid \alpha_i, \lambda_t, D_{it}, \mathbf{X}_{it}] = 0

% ── 目标参数 ─────────────────────────────────────────
% \hat{\beta} 估计处理组的平均处理效应（ATT）
```

**平行趋势假设（Parallel Trends Assumption）的形式化表述：**

```latex
\begin{assumption}[Parallel Trends]
  E[Y_{it}(0) - Y_{it'}(0) \mid D_i = 1]
  = E[Y_{it}(0) - Y_{it'}(0) \mid D_i = 0],
  \quad \forall\, t \neq t'
\end{assumption}
% 含义：在反事实意义上，若处理组未受处理，
%       其结果变量的时间趋势与控制组相同。
```

---

### 模型 A'：事件研究（Event Study）

**适用**：检验平行趋势假设；展示处理效应的动态路径（预期效应、短期冲击、长期效应）。

```latex
% ── 事件研究方程 ─────────────────────────────────────
\begin{equation}\label{eq:eventstudy}
  Y_{it} = \alpha_i + \lambda_t
           + \sum_{\substack{k = k_{\min} \\ k \neq -1}}^{k_{\max}}
             \beta_k \cdot \mathbf{1}[\text{EventTime}_{it} = k]
           + \mathbf{X}_{it}'\boldsymbol{\gamma} + \varepsilon_{it}
\end{equation}

% EventTime_{it} = t - T_i^*：相对于个体 i 首次受处理时间 T_i^* 的事件时间
% k = -1（处理前一期）设为基准组，其系数归一化为零
% k_{\min} < 0 的系数：检验预趋势（应联合不显著于零）
% k \geq 0 的系数：处理效应的动态路径
%
% 平行趋势的可检验形式：
%   H_0: \beta_k = 0 \text{ for all } k < 0
```

---

### 模型 B：Sharp RDD（精确断点回归）

**适用**：处理分配完全由连续型分配变量（Running Variable）是否超过阈值 $c$ 决定。

```latex
% ── Sharp RDD 主方程 ─────────────────────────────────
\begin{equation}\label{eq:rdd_sharp}
  Y_i = \tau \cdot D_i + f(X_i - c) + \varepsilon_i
\end{equation}

% 其中：
%   D_i = \mathbf{1}[X_i \geq c]           \quad \text{（处理变量，精确服从）}
%   f(\cdot)                                \quad \text{（分配变量的灵活函数，通常为局部线性）}
%   c                                       \quad \text{（阈值，对应 X_i - c = 0）}
%
% 局部线性规格（推荐）：
\begin{equation}\label{eq:rdd_ll}
  Y_i = \alpha_0 + \tau \cdot D_i
      + \beta_1 (X_i - c)
      + \beta_2 (X_i - c) \cdot D_i
      + \varepsilon_i, \quad |X_i - c| \leq h
\end{equation}
% 带宽 h 由 \texttt{rdbwselect}（MSE 最优或 CER 最优）选择
% 参数 \tau 估计阈值处的局部平均处理效应（LATE at cutoff）

% ── 连续性假设 ────────────────────────────────────────
\begin{assumption}[Continuity at Cutoff]
  E[Y_i(0) \mid X_i = x] \text{ 和 } E[Y_i(1) \mid X_i = x]
  \text{ 在 } x = c \text{ 处连续}
\end{assumption}
% 等价于：分配变量在阈值处的密度无跳跃（McCrary 检验）
```

---

### 模型 B'：Fuzzy RDD（模糊断点回归）

**适用**：超过阈值显著提高处理概率，但非完全服从（部分个体跨越阈值但未受处理，或未跨越阈值但受处理）。

```latex
% ── 两阶段规格 ───────────────────────────────────────
% 第一阶段：
\begin{equation}\label{eq:rdd_fuzzy_fs}
  D_i = \pi_0 + \pi_1 \cdot \mathbf{1}[X_i \geq c]
      + g(X_i - c) + \nu_i, \quad |X_i - c| \leq h
\end{equation}

% 第二阶段（2SLS）：
\begin{equation}\label{eq:rdd_fuzzy_ss}
  Y_i = \alpha + \tau_{\text{FRD}} \cdot \hat{D}_i
      + f(X_i - c) + \varepsilon_i, \quad |X_i - c| \leq h
\end{equation}

% \tau_{\text{FRD}} = \frac{\lim_{x \to c^+} E[Y_i \mid X_i=x] - \lim_{x \to c^-} E[Y_i \mid X_i=x]}
%                          {\lim_{x \to c^+} E[D_i \mid X_i=x] - \lim_{x \to c^-} E[D_i \mid X_i=x]}
% 估计 Compliers 的局部平均处理效应（LATE）
```

---

### 模型 C：工具变量（2SLS）

**适用**：处理变量 D 内生，存在有效工具变量 Z（相关性 + 排他性限制）。

```latex
% ── 第一阶段 ─────────────────────────────────────────
\begin{equation}\label{eq:iv_first}
  D_i = \pi_0 + \pi_1 Z_i + \mathbf{X}_i'\boldsymbol{\delta} + \nu_i
\end{equation}

% ── 第二阶段 ─────────────────────────────────────────
\begin{equation}\label{eq:iv_second}
  Y_i = \beta_0 + \beta_1 \hat{D}_i + \mathbf{X}_i'\boldsymbol{\gamma} + \varepsilon_i
\end{equation}

% \hat{D}_i = \hat{\pi}_0 + \hat{\pi}_1 Z_i + \mathbf{X}_i'\hat{\boldsymbol{\delta}}
%              \quad \text{（第一阶段拟合值）}
%
% IV 有效性的三个条件：
% 1. 相关性（Relevance）：  \pi_1 \neq 0，经验检验：F_{\text{first stage}} > 10
% 2. 排他性（Exclusion）：  Z_i \perp \varepsilon_i \mid \mathbf{X}_i
%                           （Z 仅通过 D 影响 Y，不可直接检验，需理论论证）
% 3. 单调性（Monotonicity）：Z_i = 1 \Rightarrow D_i(1) \geq D_i(0)
%                           （无"Defiers"，LATE 解释的前提）
%
% \beta_1 \text{ 估计 Compliers 的 LATE（本地平均处理效应）}

% ── 简约式（Reduced Form） ────────────────────────────
\begin{equation}\label{eq:iv_rf}
  Y_i = \rho_0 + \rho_1 Z_i + \mathbf{X}_i'\boldsymbol{\phi} + \eta_i
\end{equation}
% \hat{\beta}_1^{\text{IV}} = \hat{\rho}_1 / \hat{\pi}_1
% 简约式应在论文中单独汇报，供读者核实
```

---

### 模型 D：面板固定效应（Panel FE）

**适用**：处理变量在个体内存在时变，个体固定效应足以控制截面异质性，无显著内生性（或已排除）。

```latex
% ── TWFE 主方程 ──────────────────────────────────────
\begin{equation}\label{eq:pfe}
  Y_{it} = \alpha_i + \lambda_t
           + \boldsymbol{\beta}' \mathbf{D}_{it}
           + \mathbf{X}_{it}'\boldsymbol{\gamma}
           + \varepsilon_{it}
\end{equation}

% \alpha_i：个体固定效应（消除个体层面遗漏变量偏误）
% \lambda_t：时间固定效应（控制共同时间趋势）
% \mathbf{D}_{it}：关键解释变量向量（可为多个政策变量）
% \mathbf{X}_{it}：时变控制变量
%
% 外生性假设（严格外生性）：
\begin{assumption}[Strict Exogeneity]
  E[\varepsilon_{it} \mid \mathbf{D}_{i1}, \ldots, \mathbf{D}_{iT},
    \mathbf{X}_{i1}, \ldots, \mathbf{X}_{iT}, \alpha_i] = 0
\end{assumption}
% 若严格外生性不满足（如 D 受滞后 Y 影响），需考虑 Arellano-Bond GMM

% ── Hausman 检验（FE vs RE） ──────────────────────────
% H_0: \text{随机效应模型一致}（\alpha_i \perp \mathbf{D}_{it}）
% H_1: \text{固定效应模型必要}（\alpha_i \text{ 与 } \mathbf{D}_{it} 相关）
% 若拒绝 H_0，使用 FE；否则 RE 更有效率
```

---

### 模型 E：合成控制（Synthetic Control）

**适用**：单一处理单元（或少数处理单元）、较长预处理期、处理组与任何单一控制单元匹配不理想。

```latex
% ── 合成控制估计量 ────────────────────────────────────
% 构造控制组合权重 w_j^* 使得合成控制单元与处理单元在预处理期最相似：
\begin{equation}\label{eq:synth_weights}
  (w_2^*, \ldots, w_{J+1}^*) = \arg\min_{\mathbf{w}}
  \left\| \mathbf{X}_1 - \mathbf{X}_0 \mathbf{w} \right\|_V
\end{equation}

% \mathbf{X}_1：处理单元的预处理期结果和预测变量向量
% \mathbf{X}_0：控制池中各供体单元的对应向量
% \mathbf{V}：对预测变量的重要性加权矩阵（由外层优化确定）
% 约束：w_j \geq 0，\sum_j w_j = 1

% ── 处理效应估计 ──────────────────────────────────────
\begin{equation}\label{eq:synth_effect}
  \hat{\alpha}_{1t} = Y_{1t} - \hat{Y}_{1t}^N
  = Y_{1t} - \sum_{j=2}^{J+1} w_j^* Y_{jt},
  \quad t > T_0
\end{equation}

% Y_{1t}          处理单元在 t 期的实际结果
% \hat{Y}_{1t}^N  合成控制（反事实）结果
% \hat{\alpha}_{1t} t 期的处理效应估计（可展示随时间变化的动态路径）

% ── 推断（排列检验） ─────────────────────────────────
% 对控制池中每个供体单元重复合成控制过程（"安慰剂"），
% p 值 = 处理单元的 MSPE 比 ≥ 该比值的供体单元比例
```

---

## Step 3：标准误策略

根据数据结构和识别策略，明确推荐标准误类型，并给出理由：

| 数据结构 | 推荐 SE 类型 | 关键考量 |
|---------|------------|---------|
| 纯截面，同质方差 | OLS SE | 需通过 Breusch-Pagan 检验 |
| 纯截面，异方差 | HC3 稳健 SE | 几乎始终优于 OLS SE |
| 面板，DiD/TWFE | 双向聚类 SE（个体 + 时间） | 控制个体内序列相关和时间截面相关 |
| 面板，处理变量在组内不变 | 按处理分配单元聚类 | 如政策以省为单位，则按省聚类 |
| RDD，带宽内样本 | 稳健 bias-corrected SE（rdrobust） | 标准 SE 低估 bias；Calonico 等 2014 |
| IV / 2SLS | 与主回归一致的聚类 SE | 两阶段需保持聚类层级一致 |
| 少量聚类（< 30） | Wild Bootstrap（cgmwildboot / boottest） | 聚类数不足时 t 近似失效 |

```latex
% 在论文中标准误说明的规范写法（放在表格 Notes 中）：
% "Standard errors clustered at the [province / firm / county] level
%  are reported in parentheses. *, **, *** denote significance
%  at the 10\%, 5\%, and 1\% levels, respectively."
```

---

## Step 3.5：诊断规格（Diagnostic Plan）

在进入代码执行之前，必须在模型规格阶段明确**要检验什么、为什么要检验、通过标准是什么、失败如何处理**。这些决策属于建模决策，不是执行细节。

诊断规格写入 `model-spec.md §6`，供 Phase 6（`/code`）直接读取执行，无需在代码阶段重新判断。

### 第一层：识别假设诊断规格

根据 Step 1 确认的策略，从以下表格中选取对应行，填入通过 / 失败时的具体数值标准：

| 策略 | 识别假设 | 诊断检验 | 通过标准 | 失败时处理 |
|------|---------|---------|---------|----------|
| DiD / TWFE | 平行趋势 | 事件研究图 + 预趋势系数联合 F 检验 | 联合检验 p > 0.1 | 返回 Phase 3，更换对照组或时间窗口 |
| 交错 DiD | 处理效应同质性 | Bacon 分解；C-S vs TWFE 系数对比 | TWFE / C-S 主系数差 < 20% | 改用 Callaway-Sant'Anna 估计量 |
| Sharp RDD | 分配变量连续性 | McCrary 密度检验（`rddensity`） | p > 0.1 | 返回 Phase 3，质疑 RDD 设计有效性 |
| Sharp / Fuzzy RDD | 预定变量平衡 | 对协变量做 RDD | 所有协变量 p > 0.05 | 加入协变量控制；检查数据清洗步骤 |
| Fuzzy RDD | 工具变量相关性 | 第一阶段 F；Anderson-Rubin CI | F > 10；AR CI 含 0 则无效 | 质疑 RDD 断点是否有效处理力度 |
| IV / 2SLS | 工具变量相关性 | 第一阶段 F（Stock-Yogo 临界值 16.38） | F > 10 | 寻找更强工具变量或改用 LIML |
| IV / 2SLS | 过识别约束（多 IV）| Sargan / Hansen J 检验 | p > 0.05 | 讨论哪个工具变量可能违反排他性 |
| 面板 FE | FE vs RE 选择 | Hausman 检验 | p < 0.05 → 选 FE | 若 p > 0.05，报告两种估计量并讨论 |
| 面板 FE | 严格外生性 | Wooldridge 序列相关检验 | p > 0.05 | 加滞后因变量；考虑 AB-GMM |
| 合成控制 | 预处理期拟合 | 预处理期 RMSPE 相对于供体分布 | 处理单位 RMSPE < 供体中位数 × 2 | 调整供体池；重新选择预测变量权重 |

### 第二层：统计假设诊断规格

以下检验对所有策略通用，无论识别策略如何均须执行。**注意：失败不等于终止流程——取决于失败项与 SE 策略的配合情况。**

| 检验 | 检验内容 | 通过标准 | 失败时处理 | 是否终止流程 |
|------|---------|---------|----------|------------|
| VIF（最大值）| 多重共线性 | VIF < 10 | 删除或合并共线变量，更新 §5 控制变量列表 | ⚠️ 视情况 |
| Breusch-Pagan | 异方差 | p > 0.05 | 已指定稳健/聚类 SE → 标记 WARN，继续 | 否（若已用稳健 SE）|
| Breusch-Godfrey | 序列相关（截面）| p > 0.05 | 加滞后因变量或使用 Newey-West SE | ⚠️ 视情况 |
| Wooldridge | 序列相关（面板）| p > 0.05 | 双向聚类 SE 或 AR(1) 误差 | 否（若已双向聚类）|
| RESET | 函数形式误设 | p > 0.05 | 尝试对数变换或加二次项；写入稳健性清单 | 否 |
| Cook's D（最大）| 强影响观测值 | < 4/N | 列出高影响观测值；纳入 Phase 8 稳健性 | 否 |

**关键原则：**
- **识别假设失败（第一层）→ 立即终止，返回对应阶段**：识别失败意味着因果解释无效，后续结果无意义
- **统计假设警告（第二层）→ 记录于 `model-spec.md §6`，继续**：统计问题通常可通过 SE 策略或稳健性检验应对

---

## Step 4：调用估计技能

根据 Step 1 确认的策略类型，调用对应的专属估计 skill：

| 策略 | 调用的 skill | 传入上下文 |
|------|------------|----------|
| DiD / TWFE / Event Study | `did-analysis` | 主方程规格、FE 层级、SE 聚类层级、事件窗口 |
| Sharp / Fuzzy RDD | `rdd-analysis` | 运行变量名、阈值值、带宽选择方法、局部多项式阶次 |
| 2SLS / IV | `iv-estimation` | 工具变量名、第一/第二阶段规格、SE 策略 |
| Panel FE | `panel-data` | FE 层级、Hausman 检验、SE 策略 |
| 合成控制 | `synthetic-control` | 供体池列表、预处理期、预测变量权重矩阵 |
| OLS + FE | `ols-regression` | 主方程规格、控制变量、SE 类型 |

调用格式：

> *"现在调用 `[skill 名称]`，执行 [识别策略名称] 估计。传入以下参数：*
> *主方程：[LaTeX 公式中文概述]*
> *标准误：[聚类层级]*
> *数据路径：`data/clean/[project_name]_clean.[dta|parquet]`"*

---

## Step 5：产出 `model-spec.md`

估计 skill 完成后，将本阶段所有决策整合为阶段交付文档：

```
Write: [workspace]/model-spec.md
```

**文档结构：**

```markdown
# Model Specification
**项目：** [研究问题一句话]
**版本：** v1.0
**日期：** [YYYY-MM-DD]

---

## 1. 识别策略

- **策略类型：** [DiD / RDD / IV / Panel FE / Synthetic Control]
- **目标参数：** [ATE / ATT / LATE，含经济学含义]
- **识别变量：** [Z / 运行变量 / 政策时点]

## 2. 主方程

[LaTeX 方程，直接可粘贴至论文]

$$
[主方程 LaTeX]
$$

**参数说明：**

| 符号 | 含义 |
|------|------|
| $Y_{it}$ | [结果变量描述] |
| $\beta$ | [处理效应的经济学解释，含量级] |
| ... | ... |

## 3. 识别假设

| 假设 | 形式化表述 | 诊断状态 |
|------|-----------|---------|
| [假设名] | $[LaTeX]$ | ✅ 通过 / ⚠️ 存疑 / ❌ 不满足 |

诊断状态来源于 `data-report.md` 的 EDA 结果（平行趋势图、McCrary 检验等）。

## 4. 标准误策略

- **类型：** [聚类 SE / 稳健 SE / Wild Bootstrap]
- **聚类层级：** [省 / 企业 / 县]
- **理由：** [经济学依据]

## 5. 控制变量

[变量列表及加入理由；注明哪些变量因平衡性不足被强制加入]

## 6. 诊断计划（Diagnostic Plan）

> 本节由 `/model` Step 3.5 生成，供 `/code` Phase 6 直接读取执行。不得在代码阶段修改诊断逻辑。

### 6.1 第一层：识别假设诊断

| 假设 | 检验方法 | 通过标准 | 失败时处理 |
|------|---------|---------|----------|
| [填入该策略对应的识别假设] | [检验名称] | [具体数值标准] | [返回阶段 + 修正方向] |

### 6.2 第二层：统计假设诊断

| 检验 | 通过标准 | 失败时处理 | 是否终止 |
|------|---------|----------|--------|
| VIF（最大值）| < 10 | [处理方式] | ⚠️ |
| Breusch-Pagan | p > 0.05 | 已用 [SE类型] → WARN，继续 | 否 |
| Breusch-Godfrey / Wooldridge | p > 0.05 | [处理方式] | ⚠️ |
| RESET | p > 0.05 | [处理方式] | 否 |
| Cook's D | < 4/N | 纳入 Phase 8 稳健性 | 否 |

### 6.3 诊断输出

`/code` 执行后生成 `diagnostic_report.md`，汇总两层诊断结果（✅ / ⚠️ / ❌）。
⚠️ 项自动纳入 Phase 8 `/robustness` 检验清单；❌ 项触发流程终止并返回对应阶段。

## 7. 数据质量预警与应对

[来自 data-report.md 的预警事项，及在模型设计中的应对方式]

## 8. 待执行的稳健性检验

[进入 Phase 8 前预先列出的稳健性检验清单，如：
- 替换控制变量集合
- 更换带宽（RDD）
- 安慰剂检验
- 剔除特殊样本]
```

---

## Step 6：阶段确认与移交

向用户呈现阶段摘要：

> **Phase 5 产出摘要**
>
> ✅ 主方程（LaTeX）：`model-spec.md` §2
> ✅ 识别假设形式化：`model-spec.md` §3
> ✅ 标准误策略：`model-spec.md` §4
> ✅ 估计代码：由 `[skill 名称]` 生成，见 `code/`
> ✅ 模型规格文档：`model-spec.md`
>
> ---
>
> 待您确认模型规格后，进入 Phase 6（代码执行与复现）
