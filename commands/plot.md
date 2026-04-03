---
name: plot
description: Publication Polish. Runs after results-analysis (Phase 7). Audits all tables and figures produced in Phases 4–7, upgrades them to top-journal standards by calling the table and figure skills.
tools:
  - Read
  - Write
  - Bash
---

# /plot — 表格和图表可视化

## 定位

`/plot` 是 Phase 7（`results-analysis`）与 Phase 9（`/write`）之间的**图表可视化阶段**。将 Phase 4–7 产出的所有功能性表格和图形，统一升级为符合 Top5 顶刊排版标准的最终版本，并检查是否有识别策略要求的标准图形尚未生成。

完成后产出**输出清单 `output-manifest.md`**，供 `/write`（Phase 9）直接引用，不再需要手动查找文件路径。

---

## Step 0：读取上下文

```
Read: [workspace]/results-memo.md        # Phase 7 产出：识别策略、主要结果、建议图表
Read: [workspace]/model-spec.md          # 识别策略类型（决定标准图形集合）
Read: [workspace]/data-report.md         # 变量名、样本定义
```

同时扫描已有的输出目录：

```python
import os
from pathlib import Path

tables  = sorted(Path("tables").glob("*"))
figures = sorted(Path("figures").glob("*"))

print("=== 现有表格 ===")
for f in tables:  print(f"  {f.name}")

print("\n=== 现有图形 ===")
for f in figures: print(f"  {f.name}")
```

根据识别策略，对照**标准输出清单**（见 Step 1）检查缺漏，输出审计报告：

```
=== 输出审计 ===
策略：[识别策略]

表格
  ✅ table1_descriptive.tex     — 已有，待精修
  ✅ table2_balance.tex         — 已有，待精修
  ✅ table_main.tex             — 已有，待精修
  ❌ table_event_study.tex      — 缺失，需生成

图形
  ✅ eda_did_trend.png          — 已有，待升级为 PDF
  ❌ figure_event_study.pdf     — 缺失，需生成（DiD 标准图）
  ✅ coef_main.png              — 已有，待升级
```

---

## Step 1：按识别策略确定标准输出集合

> 📎 **输出格式规范参见 [`shared/output-standards.md`](../shared/output-standards.md)**

不同识别策略有约定俗成的"必备图表"，缺少会影响审稿通过率：

### 标准表格集合（所有策略共用）

| 编号 | 文件名 | 内容 | 来源阶段 |
|------|--------|------|---------|
| Table 1 | `table1_descriptive.tex` | 描述性统计 | Phase 4 `results-analysis` |
| Table 2 | `table2_balance.tex` | 平衡性检验（若有处理变量）| Phase 4 `results-analysis` |
| Table 3 | `table_main.tex` | 主回归结果（多列规格）| Phase 6 `/code` |
| Table A1 | `table_robustness.tex` | 稳健性检验汇总 | Phase 8 `/robustness`（待生成）|

### 标准图形集合（按识别策略）

#### DiD / TWFE

| 编号 | 文件名 | 内容 | 优先级 |
|------|--------|------|--------|
| Figure 1 | `figure_event_study.pdf` | 事件研究系数图（动态处理效应）| **必须** |
| Figure 2 | `figure_parallel_trends.pdf` | 处理组 vs 控制组时间趋势 | **必须** |
| Figure A1 | `figure_bacon_decomp.pdf` | Bacon 分解（交错 DiD）| 若为交错设计 |

#### RDD

| 编号 | 文件名 | 内容 | 优先级 |
|------|--------|------|--------|
| Figure 1 | `figure_rdd_main.pdf` | RDD 主图：散点 + 两侧拟合线 + 跳跃 | **必须** |
| Figure 2 | `figure_mccrary.pdf` | McCrary 密度检验图 | **必须** |
| Figure A1 | `figure_rdd_bandwidth.pdf` | 带宽敏感性图 | 推荐 |

#### IV / 2SLS

| 编号 | 文件名 | 内容 | 优先级 |
|------|--------|------|--------|
| Figure 1 | `figure_iv_first_stage.pdf` | 第一阶段：Z vs D 散点 + 拟合线 | **必须** |
| Figure 2 | `figure_iv_reduced_form.pdf` | 简约式：Z vs Y 散点 + 拟合线 | **必须** |

#### Panel FE

| 编号 | 文件名 | 内容 | 优先级 |
|------|--------|------|--------|
| Figure 1 | `figure_coef_main.pdf` | 主系数图（多规格并列）| **必须** |
| Figure 2 | `figure_fe_variance.pdf` | Within/Between 方差分解 | 推荐 |

#### 合成控制

| 编号 | 文件名 | 内容 | 优先级 |
|------|--------|------|--------|
| Figure 1 | `figure_sc_gap.pdf` | Gap 图：处理单元 vs 合成控制 | **必须** |
| Figure 2 | `figure_sc_placebo.pdf` | In-space 安慰剂检验图 | **必须** |
| Figure 3 | `figure_sc_weights.pdf` | 供体权重条形图 | 推荐 |

---

## Step 2：表格精修（调用 `table` skill）

对 Step 0 审计中标注"✅ 待精修"的所有 `.tex` 表格，调用 `table` skill 执行以下标准化操作：

**调用格式：**

> *"调用 `table` skill，精修 `[文件名]`。要求：*
> - *格式标准：booktabs（`\toprule` / `\midrule` / `\bottomrule`），无竖线*
> - *多列对齐：系数列居中，标准误括号行与系数行紧邻*
> - *显著性标注：\*, \*\*, \*\*\*（10%/5%/1%），置于系数右上角*
> - *Notes 行：说明标准误类型、聚类层级、样本限制*
> - *宽度：`\textwidth` 自适应或指定 `tabular` 列格式*
> - *输出：覆盖原文件，同时保存 `_final` 后缀版本备份"*

---

## Step 3：图形精修与生成（调用 `figure` skill）

对审计中"✅ 待升级"的图形执行精修，对"❌ 缺失"的标准图形从头生成。

**调用格式：**

> *"调用 `figure` skill，[精修/生成] `[图形名称]`。要求：*
> - *期刊标准：AER/QJE 风格（无顶框/右框，浅灰网格）*
> - *尺寸：单栏 3.5 in × 2.8 in / 双栏 7 in × 4 in*
> - *字体大小：轴标签 10pt，标题 11pt，图注 9pt*
> - *颜色：灰度优先，彩色图须灰度可读（无红绿对）*
> - *置信区间：阴影带（`fill_between`）或误差棒，透明度 0.2*
> - *输出格式：PDF（矢量，投稿用）+ PNG 300 DPI（草稿用）*
> - *文件名：`figures/[figure_name].pdf` + `.png`"*

各策略核心图形的具体实现代码（DiD 事件研究图、RDD 主图、合成控制 Gap 图、多规格系数图）统一维护在 **`skills/figure/SKILL.md`**。调用 `figure` skill 时传入图形名称和识别策略，由该 skill 负责选择对应模板并执行。

---

## Step 4：产出 `output-manifest.md`

精修完成后，生成完整的输出清单，供 `/write` 直接引用：

```
Write: [workspace]/output-manifest.md
```

**文档结构：**

```markdown
# Output Manifest
**项目：** [研究问题一句话]
**生成日期：** [YYYY-MM-DD]
**精修标准：** AER / QJE / JPE

---

## 表格

| 论文编号 | 文件路径 | 内容 | LaTeX 标签 | 状态 |
|---------|---------|------|-----------|------|
| Table 1 | `tables/table1_descriptive.tex` | 描述性统计 | `\ref{tab:descriptive}` | ✅ 精修完成 |
| Table 2 | `tables/table2_balance.tex` | 平衡性检验 | `\ref{tab:balance}` | ✅ 精修完成 |
| Table 3 | `tables/table_main.tex` | 主回归结果 | `\ref{tab:main}` | ✅ 精修完成 |
| Table A1 | `tables/table_robustness.tex` | 稳健性检验 | `\ref{tab:robustness}` | ⏳ Phase 8 后生成 |

## 图形

| 论文编号 | 文件路径（PDF） | 内容 | LaTeX 标签 | 状态 |
|---------|--------------|------|-----------|------|
| Figure 1 | `figures/figure_event_study.pdf` | 事件研究图 | `\ref{fig:eventstudy}` | ✅ 精修完成 |
| Figure 2 | `figures/figure_parallel_trends.pdf` | 平行趋势图 | `\ref{fig:trends}` | ✅ 精修完成 |
| Figure A1 | `figures/figure_bacon_decomp.pdf` | Bacon 分解 | `\ref{fig:bacon}` | ⏳ Phase 8 后生成 |

## /write 引用说明

在论文中插入表格：
```latex
\input{tables/table_main.tex}
```

在论文中插入图形：
```latex
\begin{figure}[htbp]
  \centering
  \includegraphics[width=\textwidth]{figures/figure_event_study.pdf}
  \caption{Dynamic Treatment Effects}
  \label{fig:eventstudy}
\end{figure}
```
```

---

## Step 5：阶段确认与移交

向用户呈现精修摘要：

> **Phase 7 产出摘要**
>
> ✅ 精修表格：[N] 个（`.tex`，booktabs 规范）
> ✅ 精修图形：[N] 个（`.pdf` 矢量 + `.png` 300 DPI）
> ⏳ 待 Phase 8 补充：稳健性检验表格、异质性图形
> ✅ 输出清单：`output-manifest.md`（含 LaTeX 引用代码）
>
> ---
>
> **下一步选择：**
> - 进入 **Phase 8**（`/robustness`）—— 稳健性、异质性与机制检验
> - 或直接进入 **Phase 9**（`/write`）—— 若稳健性检验已在 `/code` 阶段完成

---

## 常见问题处理

**Q：图形中文字用中文还是英文？**

论文图形一律用**英文**（变量名、轴标签、图注），以备直接投稿。描述性文字和分析讨论用中文或英文均可，但图形本身须为英文。

**Q：PDF 与 PNG 都需要吗？**

投稿期刊用 **PDF**（矢量，无限缩放），Word 稿件或 Overleaf 草稿用 **PNG**（300 DPI 足够屏幕和打印）。两者同步生成，按需取用。

**Q：表格宽度超出单栏怎么办？**

若变量多、列数多，调用 `table` skill 使用 `\resizebox{\textwidth}{!}{...}` 或 `landscape` 环境；或将控制变量行改为"Yes/No"标记，折叠系数展示。

**Q：Phase 8 的稳健性图表如何纳入？**

`output-manifest.md` 中已为 Phase 8 输出预留 ⏳ 占位符。`/robustness` 完成后，再次调用 `/plot` 将新图表加入精修流程，并更新 `output-manifest.md`。
