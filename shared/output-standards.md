# 论文输出格式规范 (Publication Output Standards)

## 目的

本模块为所有输出阶段命令（`/code`、`/plot`、`/present`、`/write`）提供**统一的 LaTeX 和输出格式规范**，确保所有表格、图形和文档输出都符合 Top-5 期刊（AER、QJE、JPE、REStud、Econometrica）标准。

---

## 第一部分：表格规范 (Table Standards)

### 1.1 三线表 (Booktabs Format)

所有表格必须使用 `booktabs` 风格，包含三条规范的线条：`\toprule`、`\midrule`、`\bottomrule`。

**LaTeX 最小模板**：

```latex
\begin{table}[htbp]
\centering
\caption{[Table Title]}
\label{tab:[label]}
\begin{tabular}{lcccc}
\toprule
& \multicolumn{2}{c}{\textit{Panel A}} & \multicolumn{2}{c}{\textit{Panel B}} \\
\cmidrule(lr){2-3} \cmidrule(lr){4-5}
Variable & (1) & (2) & (3) & (4) \\
\midrule
Treatment & 0.123*** & 0.115** & 0.089 & 0.105** \\
          & (0.041) & (0.045) & (0.062) & (0.048) \\
Control 1 &         & 0.234** &        & 0.201** \\
          &         & (0.092) &        & (0.088) \\
\midrule
Observations & 1,000 & 1,000 & 500 & 500 \\
$R^2$ & 0.123 & 0.156 & 0.089 & 0.134 \\
\bottomrule
\end{tabular}
\begin{tablenotes}
\small
\item Standard errors clustered at the [unit level] are reported in parentheses.
*, **, *** denote significance at the 10\%, 5\%, and 1\% levels, respectively.
\end{tablenotes}
\end{table}
```

**关键规则**：

- **无竖线** (`|` 在列说明中会被移除)
- **三条线** (顶部 `\toprule`、中部 `\midrule`、底部 `\bottomrule`)
- **水平分割** (`\cmidrule(lr){col-col}` 用于多列标题下的分割)
- **标准误括号** (标准误独占一行，用圆括号)
- **标注信息** (表格下方 `\begin{tablenotes}` 说明 SE 类型、聚类层级、显著性记号)

### 1.2 描述性统计表 (Descriptive Statistics)

**结构**：观测数、均值、标准差、p25、p75（或最小/最大值）

```latex
\begin{table}[htbp]
\centering
\caption{Summary Statistics}
\begin{tabular}{lrrrrr}
\toprule
Variable & N & Mean & Std. Dev. & p25 & p75 \\
\midrule
[Y] & 10,000 & 5.234 & 2.891 & 3.125 & 7.456 \\
[D] & 10,000 & 0.347 & 0.476 & 0.000 & 1.000 \\
[Z] & 10,000 & 45.123 & 23.450 & 28.903 & 61.234 \\
\bottomrule
\end{tabular}
\end{table}
```

### 1.3 主回归结果表 (Main Regression Results)

**多列递进式规格**（从简到复杂，左到右）：

- 列 (1)：Baseline（仅处理变量）
- 列 (2)：+基本控制变量
- 列 (3)：+固定效应
- 列 (4)：首选规格（完整控制）

```latex
\begin{table}[htbp]
\centering
\caption{Main Results: Effect of [D] on [Y]}
\label{tab:main}
\begin{tabular}{lcccc}
\toprule
Dependent Variable: [Y] & (1) & (2) & (3) & (4) \\
\midrule
[D] & 0.245*** & 0.198** & 0.167* & 0.182** \\
    & (0.065) & (0.072) & (0.089) & (0.081) \\
\midrule
Controls & No & Yes & Yes & Yes \\
Individual FE & No & No & Yes & Yes \\
Year FE & No & No & Yes & Yes \\
\midrule
Observations & 5,000 & 5,000 & 5,000 & 5,000 \\
$R^2$ & 0.123 & 0.234 & 0.456 & 0.512 \\
\bottomrule
\end{tabular}
\begin{tablenotes}
\small
\item The dependent variable is [Y 的定义].
Robust standard errors clustered at [cluster level] are in parentheses.
*, **, *** denote significance at the 10\%, 5\%, and 1\% levels, respectively.
\end{tablenotes}
\end{table}
```

**显著性记号规范**：
- `*` p < 0.10
- `**` p < 0.05
- `***` p < 0.01

---

## 第二部分：图形规范 (Figure Standards)

### 2.1 尺寸与分辨率

| 排版位置 | 宽度 | 高度 | DPI | 用途 |
|---------|------|------|-----|------|
| 单栏 (Single Column) | 3.5 in | 2.8 in | 300 | 事件研究、系数图 |
| 双栏 (Double Column) | 7.0 in | 3.5 in | 300 | 多序列趋势、异质性对比 |
| 高比例（方案用） | 7.0 in | 2.2 in | 300 | 趋势图、时间序列 |

### 2.2 matplotlib rcParams 标准配置

**Python 论文质量默认值**：

```python
import matplotlib.pyplot as plt
import matplotlib as mpl

plt.rcParams.update({
    # 图形大小与 DPI
    'figure.figsize': (7, 3.5),        # 双栏宽度
    'figure.dpi': 300,                  # 矢量输出时被忽略，光栅输出时生效

    # 字体
    'font.family': 'serif',
    'font.serif': ['Times New Roman', 'Computer Modern Roman'],
    'font.size': 11,
    'axes.labelsize': 12,               # 坐标轴标签（x/y 标题）
    'axes.titlesize': 13,               # 图形标题
    'xtick.labelsize': 10,              # 坐标刻度文字
    'ytick.labelsize': 10,
    'legend.fontsize': 10,              # 图例文字

    # 线条与样式
    'axes.linewidth': 0.8,              # 坐标轴线宽
    'lines.linewidth': 1.5,             # 数据线宽
    'lines.markersize': 5,              # 数据点大小

    # 脊线与网格
    'axes.spines.top': False,           # 隐藏上脊线
    'axes.spines.right': False,         # 隐藏右脊线
    'axes.spines.left': True,
    'axes.spines.bottom': True,
    'axes.grid': True,                  # 启用网格
    'axes.grid.axis': 'y',              # 仅 y 方向网格
    'grid.color': '#cccccc',            # 浅灰色网格
    'grid.linewidth': 0.5,
    'grid.alpha': 0.3,

    # 输出格式
    'savefig.bbox': 'tight',            # 自动裁剪白边
    'savefig.pad_inches': 0.05,         # 最小边距
})
```

### 2.3 色彩方案（灰度优先）

**原则**：优先用线型和标记区分，颜色作为辅助。确保黑白打印可读。

```python
COLORS = [
    '#000000',      # 黑色 — 主要序列、处理组
    '#555555',      # 深灰 — 次要序列、控制组、合成控制
    '#999999',      # 中灰 — 第三序列
    '#cccccc',      # 浅灰 — 参考线、背景
]

MARKERS = ['o', 's', '^', 'D']              # 圆、方、三角、菱形
LINESTYLES = ['-', '--', '-.', ':']        # 实线、虚线、点线、点
```

### 2.4 轴标签与标题

**规范**：

- 坐标轴标签：清晰描述变量（包括单位）
- 标题：描述性、不超过 15 词
- 图例：放在图表右上或外部右侧，避免覆盖数据

```python
fig, ax = plt.subplots(figsize=(7, 3.5))

# ... 绘制数据 ...

ax.set_xlabel('Time Period (Years)', fontsize=12)
ax.set_ylabel('Log Income ($)', fontsize=12)
ax.set_title('Effect of Policy on Income Over Time', fontsize=13)
ax.legend(loc='upper left', frameon=True, fontsize=10)

# 隐藏上右脊线
ax.spines['top'].set_visible(False)
ax.spines['right'].set_visible(False)

plt.tight_layout()
plt.savefig('figure_name.pdf', dpi=300, bbox_inches='tight')
plt.savefig('figure_name.png', dpi=300, bbox_inches='tight')
```

### 2.5 事件研究图（Event Study）规范

**元素**：

1. **预处理期阴影** (浅灰色，alpha=0.06)
2. **信心区间** (95% 细线，90% 粗线双层)
3. **零线** (虚黑线，参考基线)
4. **处理时点标记** (竖虚线，突出政策时间)

```python
fig, ax = plt.subplots(figsize=(7, 3.5))

# 预处理期阴影
pre_mask = periods < treatment_time
if pre_mask.any():
    ax.axvspan(periods[pre_mask].min() - 0.5,
               treatment_time - 0.5,
               alpha=0.06, color='grey', zorder=0)

# 处理时点线
ax.axvline(treatment_time - 0.5, color='black', linestyle='--',
           linewidth=0.8, alpha=0.5)

# 95% CI（细线）
ax.errorbar(periods, coefs, yerr=ci95, fmt='none',
            color='#000000', linewidth=0.8, capsize=3, label='95% CI')

# 90% CI（粗线）
ax.errorbar(periods, coefs, yerr=ci90, fmt='none',
            color='#000000', linewidth=1.5, capsize=5, label='90% CI')

# 系数点
ax.scatter(periods, coefs, s=50, color='#000000', zorder=3)

# 零线
ax.axhline(0, color='black', linestyle='-', linewidth=1, alpha=0.3)

ax.set_xlabel('Period Relative to Policy Change')
ax.set_ylabel('Effect Size')
ax.legend()
plt.savefig('figure_event_study.pdf', dpi=300, bbox_inches='tight')
plt.savefig('figure_event_study.png', dpi=300, bbox_inches='tight')
```

---

## 第三部分：导出规范 (Export Standards)

### 3.1 表格导出

**双格式输出**（每个表格都应同时保存为 `.tex` 和 `.csv`）：

| 格式 | 用途 | 文件名约定 |
|------|------|-----------|
| `.tex` | 直接粘入 LaTeX 正文或通过 `\input{tables/...}` 引入 | `table_main.tex`、`table_robustness.tex` |
| `.csv` | 手工验证、再处理、上传数据仓库 | `table_main.csv`、`table_robustness.csv` |

**Python 导出示例**：

```python
import pyfixest as pf

# LaTeX 输出（booktabs 格式，不含 \begin{table} 包装）
pf.etable(
    [m1, m2, m3],
    type="tex",
    file="tables/table_main.tex"
)

# CSV 输出（验证用）
pf.etable(
    [m1, m2, m3],
    type="csv",
    file="tables/table_main.csv"
)
```

**Stata 导出示例**：

```stata
esttab m1 m2 m3 using "tables/table_main.tex", ///
    b(3) se(3) star(* 0.10 ** 0.05 *** 0.01) ///
    booktabs replace label ///
    mtitles("(1)" "(2)" "(3)") ///
    keep(treatment)

esttab m1 m2 m3 using "tables/table_main.csv", ///
    b(3) se(3) csv replace
```

### 3.2 图形导出

**双格式输出**（每个图都应同时保存为 `.pdf` 和 `.png`）：

| 格式 | 用途 | DPI | 文件名约定 |
|------|------|-----|-----------|
| `.pdf` | 论文最终输出（矢量，无损），嵌入 LaTeX | `figure_event_study.pdf` |
| `.png` | 初稿审阅、演讲幻灯片、网络共享 | `figure_event_study.png` |

```python
# 同时保存 PDF 和 PNG
plt.savefig('figures/figure_event_study.pdf', dpi=300, bbox_inches='tight', format='pdf')
plt.savefig('figures/figure_event_study.png', dpi=300, bbox_inches='tight', format='png')
```

### 3.3 文件命名约定

```
表格：
  table_descriptive.tex          # 描述性统计（Phase 4）
  table_balance.tex              # 平衡性检验（Phase 4）
  table_main.tex                 # 主回归结果（Phase 6）
  table_event_study.tex          # 事件研究结果（Phase 8）
  table_heterogeneity.tex        # 异质性分析（Phase 8）
  table_robustness.tex           # 稳健性检验（Phase 8）

图形：
  figure_eda_trend.pdf           # EDA 趋势图（Phase 4）
  figure_event_study.pdf         # 事件研究图（Phase 6/8）
  figure_coef_main.pdf           # 主系数及 CI（Phase 6）
  figure_heterogeneity.pdf       # 异质性对比（Phase 8）

均附带 .csv（表）和 .png（图）版本供备份和演讲用。
```

---

## 第四部分：期刊风格参考 (Journal Style Reference)

### 4.1 常见期刊格式差异

| 特征 | AER | QJE | REStud | Econometrica | JPE |
|------|-----|-----|--------|------------|-----|
| 表格位置 | 文末 | 文末 | 文末 | 文末 | 文末 |
| 线条风格 | booktabs | booktabs | booktabs | booktabs | booktabs |
| 显著性记号 | `*/**/**` | `*/**/**` | `*/**/**` | `*/**/**` | `*/**/**` |
| SE 位置 | 括号下行 | 括号下行 | 括号下行 | 括号下行 | 括号下行 |
| 图形嵌入 | 文内 | 文内 | 文内 | 文内 | 文内 |
| 字体 | Times New Roman | Times New Roman | Times New Roman | Times New Roman | Times New Roman |
| 图表标注语言 | 英文 | 英文 | 英文 | 英文 | 英文 |

**核心规则（所有期刊统一）**：
- 三线表（无竖线）
- 显著性：*** (1%), ** (5%), * (10%)
- SE 下置于括号
- 图形为矢量或高 DPI 光栅
- 表脚注说明 SE 类型和聚类方式

### 4.2 常见修改建议

**表格超出页宽**：
```latex
\begin{landscape}
  \begin{table}...
\end{landscape}
```

或使用较小字体：
```latex
\begin{table}
\small  % 或 \footnotesize
\begin{tabular}{...}
```

**图形质量不佳**（文字太小）：
```python
plt.rcParams['font.size'] = 11
plt.rcParams['axes.labelsize'] = 12
```

---

## 使用范例

### 范例：图表命令中的参考

在 `/plot`、`/code`、`/write` 等命令中，在涉及输出格式的章节添加：

```markdown
## [输出格式相关章节]

> 📎 **输出格式规范参见 [`shared/output-standards.md`](../shared/output-standards.md)**
> 所有表格必须使用 booktabs 三线格式；所有图形必须同时导出 PDF（矢量）和 PNG（高 DPI）。
```

---

## 维护说明

本文档为所有 LaTeX/输出相关命令的唯一权威来源。若需修改格式规范（如 matplotlib rcParams、表格模板等），在此文件中统一修改，所有引用该标准的命令自动采纳新规范。

版本历史：
- v1.0 (2026-03-30)：初版，覆盖表格、图形、导出、期刊风格规范
