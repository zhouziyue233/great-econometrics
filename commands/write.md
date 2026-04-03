---
name: write
description: Phase 9 Full Paper Writing. Reads all upstream outputs (model-spec.md, results-memo.md, robustness-report.md, literature-review-report.md) and drafts any section or the complete paper to Top-5 economics journal standards following a narrative-driven structure; assembles LaTeX, compiles to PDF, and exports DOCX via pandoc, producing a final paper/ directory.
tools:
  - Read
  - Write
  - Bash
---

# /write — 全文写作

## 定位

`/write` 是实证研究工作流的**第九阶段**，承接前八个阶段的全部产出，将研究从"有结果"推进到"有论文"。核心职责：

1. 读取所有上游输出文件，提取可直接嵌入论文的数字、表格引用和写作建议
2. 按 Top-5 期刊（AER/QJE/JPE/REStud/Econometrica）的写作规范，以**叙事推进逻辑**分节起草论文
3. 正文图表与附录图表按重要性分流；横向宽表独占横排页面
4. 组装完整 LaTeX 全文，编译验证，产出 `paper/paper.pdf`；并用 pandoc 导出 `paper/paper.docx`

---

## Step 0：读取上游输出

> 📎 **参见 [`shared/context-reader.md`](../shared/context-reader.md)**
> 本阶段所需文件：`model-spec.md`（必需）、`results-memo.md`（必需）。可选文件：`research-question.md`、`literature-review-report.md`、`identification-memo.md`、`data-report.md`、`robustness-report.md`、`tables/table_main.csv`。

### 0.2 提取写作关键信息

从各文件中提取以下内容，构建**写作信息底稿**，后续各节直接调用：

| 来源 | 提取内容 | 用于哪节 |
|------|---------|---------|
| `research-question.md` | 研究问题一句话、政策背景 | 摘要、引言 |
| `literature-review-report.md` | 文献综述主线、本文贡献定位、近年引用 | 引言、文献综述 |
| `identification-memo.md` | 识别策略逻辑（三层辩护）、核心假设 | 实证策略节 |
| `model-spec.md §2` | 主方程 LaTeX、符号说明 | 实证策略节 |
| `model-spec.md §4` | 标准误策略及理由 | 实证策略节 |
| `data-report.md §2–3` | 样本量 N、时间跨度、变量定义 | 数据节 |
| `results-memo.md §1` | 主系数 β̂、SE、p 值、95% CI | 摘要、引言、结果节 |
| `results-memo.md §2` | 经济显著性（σ单位效应量、相对均值%） | 结果节 |
| `results-memo.md §4` | 识别可信度及语言强度建议 | 全文因果语言 |
| `results-memo.md §5` | 外部有效性局限性 | 讨论节、结论节 |
| `robustness-report.md §1` | 稳健性一句话结论（范围 + 显著性） | 摘要、稳健性节 |
| `robustness-report.md §2` | 异质性结论（分组 + 交互项显著性）| 异质性/讨论节 |
| `robustness-report.md §3` | 机制证据及语言强度 | 机制/讨论节 |
| `robustness-report.md §5` | Phase 9 写作建议（直接使用）| 所有节 |

提取完成后向用户输出写作信息底稿确认：

```
═══════════════════════════════════════════════════════
Phase 9 写作信息底稿
═══════════════════════════════════════════════════════
研究问题：[X 对 Y 的影响]
识别策略：[策略名称]
目标参数：[ATE / ATT / LATE]
主系数：β̂ = [值]（SE = [值]，p = [值]）
因果语言强度：[因果语言 / 审慎因果 / 相关性语言]
稳健性结论：[一句话]
异质性关键发现：[有/无，一句话]
机制证据：[有直接证据 / 间接相容证据 / 无]
═══════════════════════════════════════════════════════
```

---

## Step 1：写作任务确认

### 1.1 节次选择

向用户确认写作范围：

> *"请选择写作目标：*
>
> - **A. 完整论文**（推荐）：按顺序起草所有节，组装 LaTeX 全文并编译
> - **B. 指定节次**：选择一个或多个节次单独起草
>
> 可用节次：*
> ① 摘要（Abstract）*
> ② 引言（Introduction）*
> ③ 文献综述（Literature Review）*
> ④ 数据节（Data）*
> ⑤ 实证策略节（Empirical Strategy）*
> ⑥ 结果节（Results）*
> ⑦ 稳健性节（Robustness）*
> ⑧ 异质性与机制节（Heterogeneity & Mechanisms）*
> ⑨ 讨论节（Discussion）*
> ⑩ 结论节（Conclusion）"*

### 1.2 语言确认

> *"论文语言：中文 / 英文？（默认：英文，符合国际期刊投稿惯例）"*

### 1.3 目标期刊（可选）

若用户指定目标期刊，按该期刊的格式要求调整：

| 期刊 | 字数上限 | 表格位置 | 引注格式 | 特殊要求 |
|------|---------|---------|---------|---------|
| AER | ~12,000 词 | 文末 | Author (Year) | 贡献宣示简洁，不宜超过 3 条 |
| QJE | 无硬性上限 | 文末 | Author (Year) | Introduction 要求极高，通常 4–6 页 |
| JPE | ~12,000 词 | 文末 | Author (Year) | 偏好简洁，讨论节可与结果节合并 |
| REStud | ~12,000 词 | 文末 | Author (Year) | 理论/方法结合型论文偏多 |
| Econometrica | 无硬性上限 | 文末 | Author (Year) | 数学严谨性要求最高 |

---

## Step 1.5：叙事推进原则（写作前必读）

**经济学实证论文本质上是一个因果故事**，读者从第一段开始就应该被带着走完整个研究旅程。写作时必须遵循以下原则，而非生成孤立段落的堆砌。

### 核心叙事弧

```
引言（提问）
  → 文献（已知边界）
    → 数据（研究舞台）
      → 识别策略（解题思路）
        → 主要结果（答案 + 量级）
          → 稳健性（为什么可信）
            → 机制/异质性（为什么如此）
              → 结论（So what）
```

每一节都应以**承接上一节的一句话**开篇，并以**预告下一节的一句话**结尾，形成连贯的叙事链条。

### 图表叙事配合规则

每一张引用的表格或图形，正文中必须做到三件事：

| 步骤 | 说明 | 示例 |
|------|------|------|
| **① 预告**（在引用前） | 告诉读者即将看到什么 | *"Table 2 presents our main results."* |
| **② 解读**（与引用同段）| 从表/图中提炼一句核心结论，包含具体数字 | *"The coefficient on D is 0.12 (s.e. = 0.04), implying..."* |
| **③ 评价**（引用后） | 说明这个结果对研究故事意味着什么 | *"This is consistent with our identification assumption."* |

**禁止**只写 *"See Table X."* 或 *"Figure Y shows the results."* 而不给出数字和解读。

### 正文 vs. 附录的图表分流规则

**放正文的图表**（直接支撑主叙事的发现）：

| 类型 | 条件 |
|------|------|
| 描述性统计表（Table 1）| 必须在正文 |
| 主回归表（通常为 Table 2）| 必须在正文 |
| 核心识别图（事件研究图、RDD 断点图、IV 一阶段图）| 必须在正文 |
| 主要异质性结果表（若构成核心贡献之一）| 在正文 |

**放附录的图表**（支撑性或补充性发现）：

| 类型 | 理由 |
|------|------|
| 稳健性检验表（多列规格对比）| 验证性，不是新发现 |
| 平衡性检验表（DiD/IV 的 balance table）| 技术性，编辑通常接受 |
| 变量定义表 | 参考性 |
| 额外的异质性分组 | 超出核心贡献的延伸 |
| 数据清洗流程表（样本筛选漏斗）| 技术性 |
| 补充机制检验图 | 探索性 |

### 横向宽表（景观页）规则

**判断标准**：回归表超过 **5 列**，或在标准页面宽度（`\textwidth`）内会产生 `Overfull \hbox > 5pt` 时，**必须**独占一页横向排版：

```latex
\begin{landscape}
\begin{table}[p]         % [p] = 独占浮动页
\caption{[表格标题]}
\label{tab:xxx}
\footnotesize            % 稍小字号
\begin{tabular}{...}
...
\end{tabular}
\end{table}
\end{landscape}
```

适用场景：主稳健性表（多规格对比）、异质性分组表（多子组）、平行趋势前后期分列表。

---

## Step 2：各节写作规范

> 📎 **输出格式规范参见 [`shared/output-standards.md`](../shared/output-standards.md)**

**分工说明**：本步骤定义的是**内容规范**——每节该写什么、从哪个上游文件取数据、禁止哪些表达。**LaTeX 模板结构**（preamble、各节骨架、编译流程）由 `paper-writing` skill 提供，执行写作时同时参照两者。

调用 `paper-writing` skill 并传入以下上下文（Mode A，跳过 skill 内部 Q&A）：
- 研究类型：实证
- 目标期刊：`[来自 Step 1.3]`
- 论文语言：`[来自 Step 1.2]`
- 写作范围：`[来自 Step 1.1]`
- 因果语言强度：`[来自 results-memo.md §4]`

各节内容规范如下，这是 Top-5 期刊区别于一般期刊的核心要求：

---

### 2.1 摘要（Abstract）

**格式**：150–200 词，5–6 句话，结构固定：

```
句1：研究问题（What question）
句2–3：方法与数据（How + Where）
句4–5：主要发现，必须包含具体数字（What we find）
句6：贡献/政策含义（So what）
```

**必填项**：
- 必须含主系数量级（绝对值或相对均值百分比）
- 必须含识别策略名称（"exploiting..."/"using a difference-in-differences design..."）
- 不得仅报告统计显著性而不报告量级

**模板**：

```latex
\begin{abstract}
\noindent
[RESEARCH QUESTION IN ONE SENTENCE].
Using [DATA SOURCE] covering [N] [UNITS] over [PERIOD],
we exploit [IDENTIFICATION STRATEGY] to identify
the causal effect of [D] on [Y].
We find that [MAIN FINDING WITH COEFFICIENT]:
[D] [increases/decreases] [Y] by [MAGNITUDE]
[UNITS/PERCENT/SD],
[SIGNIFICANCE STATEMENT].
This effect is [heterogeneous/concentrated in/robust to],
with [KEY HETEROGENEITY OR ROBUSTNESS FINDING].
Our results [CONTRIBUTION: inform/challenge/extend]
[POLICY/THEORY IMPLICATION].
\end{abstract}
```

---

### 2.2 引言（Introduction）

Top-5 期刊的 Introduction 通常为 4–6 页（约 1,500–2,000 词），严格遵循以下**五段结构**：

**第一段：研究问题 + 主要发现（数字必须出现）**

> 要求：第一段结束前，读者必须知道（1）研究的问题是什么；（2）主要发现是什么；（3）效应的量级是多少。这是 AER/QJE 的不成文规范。

```latex
% 第一段模板
[HOOK SENTENCE — 一个引发政策或理论关注的事实/数字].
[RESEARCH QUESTION]. This paper provides causal evidence that
[MAIN FINDING]: [D] [causes/is associated with] a
[MAGNITUDE] [increase/decrease] in [Y],
[SIGNIFICANCE — e.g., "statistically significant at the 1\% level"].
```

**第二段：为什么这个问题难回答（内生性威胁）**

> 说明 OLS 的偏误方向和来源，为识别策略做铺垫。

```latex
% 第二段模板
Estimating the causal effect of [D] on [Y] is challenging
for three reasons. First, [OVB THREAT]. Second,
[REVERSE CAUSALITY THREAT]. Third, [MEASUREMENT ERROR/SELECTION].
Prior work using [OLS/cross-section] is likely to
[overstate/understate] the true effect because [DIRECTION OF BIAS].
```

**第三段：本文的识别策略与数据**

> 清晰描述外生变异来源，让读者理解为什么这里的估计是可信的。

**第四段：主要发现汇总（含异质性 + 稳健性一句话）**

> 量级、方向、主要异质性维度、稳健性结论。

**第五段：贡献定位 + 路线图**

> 对应 `literature-review-report.md` 的文献定位，明确说明与最近相关论文的区别；最后一句是路线图。

**引言写作禁忌**：

```
❌ 第一句话是 "This paper examines..."（太弱，直接陈述发现）
❌ 只报告统计显著性，不报告量级（"我们发现显著正效应"）
❌ 贡献宣示超过 4 条（审稿人会认为过度宣传）
❌ 路线图段超过 5 句话（简洁：一节一句）
❌ 在引言中出现数学公式或技术细节
```

---

### 2.3 文献综述节（Literature Review / Related Work）

**结构**：按研究主题组织，**不按时间顺序**。通常 2–4 个子主题，每个主题 1–3 段。

**必须包含**：

1. **与本文最相关的 3–5 篇论文**：说明它们做了什么、发现了什么、本文如何不同
2. **识别策略的先例**：哪些论文用过类似方法研究类似问题
3. **明确的贡献定位**：用 "Unlike [PAPER], we..." 句式结束每个子主题

**来源**：直接使用 `literature-review-report.md` 的结构，不重新检索。

---

### 2.4 数据节（Data）

**结构**：

```
§ 数据来源与样本构建
§ 核心变量定义
§ 描述性统计（Table 1，引用 tables/table1_descriptive.tex）
[§ 平衡性检验，若 DiD/IV，引用 tables/table2_balance.tex]
```

**规范**：

- 样本量、时间跨度、地域范围必须用具体数字（来自 `data-report.md §2`）
- 变量定义精确，与 `model-spec.md §2` 的符号保持一致
- 异常值处理方式需在此节说明（来自 `data-report.md §4`）
- Table 1 必须覆盖 Y、D、Z 及主要控制变量，格式：N、均值、标准差、p25、p75

**Table 1 引用格式**：

```latex
Table~\ref{tab:sumstats} reports summary statistics for our
main analysis sample. The average [Y] is [MEAN] ([SD]),
and [D PERCENT]\% of [UNITS] are [TREATED].
[KEY OBSERVATION ABOUT THE DATA — e.g., variation, distribution].
```

---

### 2.5 实证策略节（Empirical Strategy）

**这是审稿人审查最仔细的节**，必须覆盖：

1. **主方程**：直接使用 `model-spec.md §2` 的 LaTeX 方程，不重新推导
2. **识别假设**：正式表述每个假设（来自 `identification-memo.md §4`）
3. **识别假设的可检验性**：说明如何检验并预告结果（"In Section X, we show..."）
4. **标准误策略**：类型 + 聚类层级 + 经济学理由（来自 `model-spec.md §4`）

**识别假设陈述模板**（以平行趋势为例）：

```latex
% 识别假设正式陈述
The key identifying assumption is that, in the absence of treatment,
treated and control units would have followed parallel trends:
\begin{equation*}
  E[Y_{it}(0) - Y_{it'}(0) \mid D_i = 1]
  = E[Y_{it}(0) - Y_{it'}(0) \mid D_i = 0], \quad \forall\, t \neq t'.
\end{equation*}
This assumption would be violated if [SPECIFIC THREAT].
We provide evidence supporting this assumption
in Section~\ref{sec:robustness}, where [BRIEF PREVIEW OF TEST RESULT].
```

**语言强度规范**：

| 识别可信度（来自 results-memo.md §4）| 识别假设表述 | 结果表述 |
|--------------------------------------|------------|---------|
| 高（所有假设 ✅）| "The causal effect of D on Y..." | "causes", "increases", "reduces" |
| 中（含 ⚠️）| "The effect of D on Y, as estimated by [STRATEGY]..." | "is associated with", "predicts" |
| 低（含 ❌）| "We document a correlation between D and Y..." | "is correlated with", "we observe" |

---

### 2.6 结果节（Results）

**Lead with numbers**（Top-5 期刊的核心规范）：

> 结果节第一句话必须包含主系数数值。不得写"Table 2 shows our main results"而不给出具体数字。

**节内结构**：

```
§ 主要结果（引用 Table 2 主回归表）
  - 第一句：β̂ = [值]，显著性，量级解读
  - 逐列介绍：Baseline → +控制变量 → +FE → 首选规格
  - 经济显著性讨论（必须！）
[§ 事件研究 / 动态效应（若 DiD，引用 Fig. 1 事件研究图）]
[§ 主要异质性发现（若重要，引用 Table 3 异质性表）]
```

**经济显著性讨论格式**（来自 `results-memo.md §2`）：

```latex
% 经济显著性段落模板
To assess economic significance, note that the sample mean of
[Y] is [MEAN]. Our estimate of [COEFFICIENT] implies that
[D] is associated with a [MAGNITUDE PERCENT]\% [increase/decrease]
in [Y] relative to the mean, or approximately [SD-UNIT] standard deviations.
[COMPARISON TO LITERATURE: This is [larger/smaller] than the estimate
of \citet{CLOSEST PAPER}, who find [THEIR ESTIMATE] using [THEIR METHOD].]
```

**逐列介绍规范**：

```latex
% 逐列介绍模板
Column (1) presents the baseline specification without controls.
The coefficient on [D] is [β̂_1] (s.e. = [SE_1]),
suggesting [DIRECTION] of [Y]. Adding [CONTROL SET] in column (2)
[leaves the estimate stable / reduces the estimate to [β̂_2]],
consistent with [INTERPRETATION OF CHANGE].
Our preferred specification in column ([N]) includes [FULL CONTROLS]
and yields [FINAL ESTIMATE] ([SE], [SIGNIFICANCE]).
```

---

### 2.7 稳健性节（Robustness）

**节内结构**：

```
§ 开篇结论句（先给结论，再展示细节）
§ Panel A：推断方法稳健性（替代标准误）
§ Panel B：样本稳健性（子样本、异常值剔除）
§ Panel C：规格稳健性（函数形式、控制变量集合）
§ Panel D：识别假设检验（安慰剂、预趋势等，策略特异性）
[§ Oster δ 系数稳定性]
```

**开篇结论句**（直接使用 `robustness-report.md §5` 的建议语言）：

```latex
% 稳健性节开篇句模板
Table~\ref{tab:robustness} reports a battery of robustness checks.
Our main finding is robust: [MAIN COEFFICIENT RANGE — e.g.,
"the coefficient on [D] ranges from [MIN] to [MAX] across specifications,
remaining statistically significant at the [X]\% level in all cases"].
```

**对任何系数发生实质性变化的规格**，必须主动解释（不得只呈现表格）：

```latex
% 系数变化的解释模板
In column ([N]), [WHAT CHANGES — e.g., "we restrict the sample to..."].
The estimate [increases/decreases] to [NEW ESTIMATE], reflecting
[ECONOMIC INTERPRETATION — e.g., "the composition of the subgroup"].
This change is consistent with [MECHANISM/THEORY], and does not
challenge our main conclusion because [REASON].
```

---

### 2.8 异质性与机制节（Heterogeneity and Mechanisms）

此节在很多 Top-5 论文中与 Results 节合并，也可单独成节。按以下顺序叙述：

**异质性分析（来自 `robustness-report.md §2`）**：

1. 先陈述理论预测（为什么预期存在异质性）
2. 再呈现表格结果（引用 `tables/table_heterogeneity.tex`）
3. 结论需回应理论预测（是否符合预期，如不符合需解释）

**机制分析（来自 `robustness-report.md §3`）**：

语言强度按 `robustness-report.md §5` 建议：
- "直接机制证据"（有外生识别）→ "provides direct evidence for"
- "间接相容证据"（仅描述性）→ "is consistent with" / "provides suggestive evidence"
- 排除竞争性解释 → "rules out" / "cannot be explained by"

---

### 2.9 结论节（Conclusion）

**Top-5 期刊的结论节通常只有 1–2 页**，结构：

```
① 复述问题 + 方法（1句）
② 主要发现（2–3句，含量级）
③ 稳健性摘要（1句）
④ 政策含义或理论贡献（2–3句）
⑤ 局限性（1–2句，诚实但简洁）
⑥ 未来研究方向（1–2句）
```

**结论禁忌**：

```
❌ 引入任何在正文中未出现的新结果
❌ 重复复述整个论文（读者已经读了）
❌ 只说"我们做了X发现了Y"不谈 So what
❌ 结论超过 2 页（审稿人会注意到）
```

---

## Step 3：LaTeX 全文组装

### 3.1 文件目录结构

```
paper/
├── paper.tex             # 主文件（\input 各节）
├── sections/
│   ├── abstract.tex
│   ├── introduction.tex
│   ├── literature.tex
│   ├── data.tex
│   ├── strategy.tex
│   ├── results.tex
│   ├── robustness.tex
│   ├── heterogeneity.tex  # 异质性与机制节
│   ├── discussion.tex
│   ├── conclusion.tex
│   └── appendix.tex       # 附录（附录图表统一放此处）
├── tables/               # 符号链接或复制自 ../tables/
├── figures/              # 符号链接或复制自 ../figures/
└── references.bib
```

**附录节（appendix.tex）的标准结构**（根据可用上游文件自动填充）：

```latex
\appendix
\renewcommand{\thesection}{\Alph{section}}   % 附录编号：A, B, C...
\renewcommand{\thetable}{A\arabic{table}}    % 表编号：A1, A2...
\renewcommand{\thefigure}{A\arabic{figure}}  % 图编号：A1, A2...
\setcounter{table}{0}
\setcounter{figure}{0}

\section{附加稳健性检验}
\label{sec:appendix_robustness}
% 来源：tables/table_robustness_*.tex

\section{平衡性检验}
\label{sec:appendix_balance}
% 来源：tables/table_balance.tex

\section{变量定义}
\label{sec:appendix_variables}
% 来源：data-report.md 中的变量定义表

\section{补充图形}
\label{sec:appendix_figures}
% 来源：figures/ 目录中非核心图形
```

> **附录图表引用规则**：正文中凡引用附录图表，需明确标注位置，例如：*"Appendix Table A1 reports the balance test results."* 或 *"(see Appendix Figure A1)"*。

### 3.2 主文件（paper.tex）

**LaTeX preamble 和前言页结构以 `paper-writing` skill 中的模板为准**（见该 skill 的"LaTeX Preamble (Authoritative)"节）。此处不再重复 preamble 代码，以 skill 为唯一来源，避免版本分歧。

组装时注意以下两点与内容规范直接相关：

1. **`\graphicspath{{../figures/}}`**：路径相对于 `paper/`，指向上级目录的 `figures/`，不要写成 `./figures/`（`paper/` 子目录下不存在 `figures/`）。
2. **`\input` 顺序**：依次为 introduction → literature → data → strategy → results → robustness → heterogeneity → discussion → conclusion，与 Step 3.1 目录结构一致。

### 3.3 参考文献（references.bib）

根据 `literature-review-report.md` 中引用的论文，生成标准 BibTeX 格式：

```bibtex
@article{Author2020,
  author  = {Author, First and Coauthor, Second},
  title   = {Title of the Paper},
  journal = {American Economic Review},
  year    = {2020},
  volume  = {110},
  number  = {3},
  pages   = {101--130},
  doi     = {10.1257/aer.XXXXXXXX}
}
```

---

## Step 4：编译验证 + DOCX 导出

### 4.1 LaTeX → PDF 编译

```bash
cd [workspace]/paper/
pdflatex -interaction=nonstopmode paper.tex   # 第1次：生成 .aux
bibtex paper                                  # 处理参考文献
pdflatex -interaction=nonstopmode paper.tex   # 第2次：嵌入书目
pdflatex -interaction=nonstopmode paper.tex   # 第3次：确保交叉引用

# 检查常见错误
grep -i "overfull\|undefined\|missing\|error" paper.log
grep "Overfull .hbox" paper.log      # > 10pt = 可见溢出，必须修复
grep "File.*not found" paper.log     # 缺包
grep "Citation.*undefined" paper.log # 未定义引用
```

**常见错误自动修复**：

| 错误 | 原因 | 修复方式 |
|------|------|---------|
| `File 'siunitx.sty' not found` | 表格使用 `S` 列 | 改为 `c` 列或 `D{.}{.}{-1}` |
| `Unicode character U+XXXX` | Python 写入 `≥`、`→` | 替换为 `$\geq$`、`$\rightarrow$` |
| `Overfull \hbox (>10pt)` | 表格超出页宽 | 加 `\begin{landscape}` + `\footnotesize` |
| `Citation xxx undefined` | bib 条目缺失 | 补充到 `references.bib` 后重跑 bibtex |

### 4.2 LaTeX → DOCX 导出

PDF 编译成功后，使用 `pandoc` 将 LaTeX 转为 DOCX，供非 LaTeX 用户审阅和批注：

```bash
cd [workspace]/paper/

# 方式一：pandoc（推荐，保留公式和引用）
pandoc paper.tex \
  --bibliography=references.bib \
  --citeproc \
  --reference-doc=../shared/docx_template.docx \   # 若存在自定义模板
  -o paper.docx

# 若无自定义模板（通用方式）
pandoc paper.tex \
  --bibliography=references.bib \
  --citeproc \
  -o paper.docx

echo "✅ DOCX 已导出：paper/paper.docx"
```

若 pandoc 不可用，回退方式：

```bash
# 方式二：LibreOffice headless（较低保真度，但不需要 pandoc）
libreoffice --headless --convert-to docx paper.pdf --outdir .
echo "✅ DOCX 已导出（via LibreOffice）：paper/paper.docx"
```

安装 pandoc（若缺失）：

```bash
# Ubuntu/Debian
apt-get install -y pandoc 2>/dev/null || true

# macOS
brew install pandoc 2>/dev/null || true

# 检查版本
pandoc --version | head -1
```

### 4.3 编译验证摘要

```
编译验证结果
─────────────────────────────────────────────────────
✅ pdflatex 编译成功（3 次，含 bibtex）
✅ 无 undefined references
✅ 无 missing citations
⚠️ Overfull \hbox (2.3pt) — 次要，可接受
─────────────────────────────────────────────────────
✅ DOCX 导出成功（pandoc / LibreOffice）
─────────────────────────────────────────────────────
输出文件：paper/paper.pdf（[N] 页）
         paper/paper.docx
```

---

## Step 5：产出文件清单

```
Write: [workspace]/paper/paper.tex
Write: [workspace]/paper/sections/abstract.tex
Write: [workspace]/paper/sections/introduction.tex
Write: [workspace]/paper/sections/literature.tex
Write: [workspace]/paper/sections/data.tex
Write: [workspace]/paper/sections/strategy.tex
Write: [workspace]/paper/sections/results.tex
Write: [workspace]/paper/sections/robustness.tex
Write: [workspace]/paper/sections/heterogeneity.tex
Write: [workspace]/paper/sections/discussion.tex
Write: [workspace]/paper/sections/conclusion.tex
Write: [workspace]/paper/sections/appendix.tex   # 附录（附录图表）
Write: [workspace]/paper/references.bib
```

**版本管理**：每次修改后，在文件名中加版本号：

```
paper_v1.0_YYYYMMDD.tex   # 初稿
paper_v1.1_YYYYMMDD.tex   # 第一轮修改
paper_v2.0_YYYYMMDD.tex   # 大幅修改
```

---

## Step 6：阶段确认与移交

向用户呈现阶段摘要：

> **Phase 9 产出摘要**
>
> ✅ 摘要：`paper/sections/abstract.tex`
> ✅ 引言：`paper/sections/introduction.tex`
> ✅ 文献综述：`paper/sections/literature.tex`
> ✅ 数据节：`paper/sections/data.tex`
> ✅ 实证策略节：`paper/sections/strategy.tex`
> ✅ 结果节：`paper/sections/results.tex`
> ✅ 稳健性节：`paper/sections/robustness.tex`
> ✅ 异质性与机制节：`paper/sections/heterogeneity.tex`
> ✅ 讨论节：`paper/sections/discussion.tex`
> ✅ 结论节：`paper/sections/conclusion.tex`
> ✅ 附录：`paper/sections/appendix.tex`（含附录图表）
> ✅ 主文件 + 参考文献：`paper/paper.tex` + `paper/references.bib`
> ✅ PDF：`paper/paper.pdf`（[N] 页）
> ✅ DOCX：`paper/paper.docx`（供审阅批注）
>
> 完成初稿后，可运行 `/present` 将论文转化为学术报告幻灯片。
