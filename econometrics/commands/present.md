---
name: present
description: Phase 10 (optional) Beamer-Style PPTX Generation. Reads all upstream outputs and paper/sections/, maps research content to a presentation-type-specific slide outline, calls beamer-ppt skill to generate a Beamer-style .pptx file (navy-blue Metropolis theme, publication-quality), and produces slides/ directory.
tools:
  - Read
  - Write
  - Bash
---

# /present — 学术报告幻灯片生成

## 定位

`/present` 是实证研究工作流的**可选阶段**，通常在 Phase 9（`/write`）完成后运行，也可在初稿完成前并行进行。核心职责：

1. 读取所有上游输出，提取幻灯片所需的关键内容
2. 根据演示类型（会议报告 / 学术讲座 / 求职报告）确定幻灯片数量和深度
3. 调用 `beamer-ppt` skill，用 python-pptx 生成 **Beamer 学术风格的 `.pptx` 文件**（海军蓝配色、16:9 宽屏、标题栏+进度条）
4. 验证幻灯片结构并输出文件

---

## Step 0：读取上游输出

### 0.1 读取文件

```
Read: [workspace]/research-question.md
Read: [workspace]/results-memo.md
Read: [workspace]/robustness-report.md
Read: [workspace]/model-spec.md
Read: [workspace]/data-report.md
```

若 `paper/sections/` 目录存在，同时读取：

```
Read: [workspace]/paper/sections/introduction.tex   # 提取贡献定位
Read: [workspace]/paper/sections/results.tex        # 提取主要发现措辞
```

**若 `results-memo.md` 不存在**，立即停止：

> *"未找到 `results-memo.md`。幻灯片的核心内容（主系数、量级解读、识别可信度）来自 Phase 7 产出。请先完成 Phase 7（`results-analysis` skill），再运行 `/present`。"*

### 0.2 提取幻灯片关键信息

从各文件中提取以下内容，作为幻灯片**内容素材库**：

| 来源 | 提取内容 | 用于哪页幻灯片 |
|------|---------|--------------|
| `research-question.md` | 研究问题一句话、政策背景数字 | Motivation、This Paper |
| `model-spec.md §1–2` | 识别策略名称、主方程（简化版）| Identification Strategy |
| `model-spec.md §3` | 识别假设（可检验形式）| Identification Strategy |
| `data-report.md §2` | 样本量 N、时间跨度、数据来源 | Data slide |
| `results-memo.md §1` | 主系数 β̂、SE、p 值 | Main Results |
| `results-memo.md §2` | 经济显著性（量级换算）| Main Results（Takeaway） |
| `results-memo.md §4` | 识别可信度 + 因果语言建议 | 全程语言控制 |
| `robustness-report.md §1` | 稳健性一句话结论 | Robustness slide |
| `robustness-report.md §2` | 关键异质性发现 | Heterogeneity slide |
| `robustness-report.md §3` | 机制证据 | Mechanism slide（若有）|

---

## Step 1：演示参数确认

### 1.1 演示类型

向用户确认：

> *"请选择演示类型（决定幻灯片数量和深度）：*
>
> **A. 15–20 分钟 会议报告**（≤ 15 张幻灯片）
> — NBER/AEA 分会场报告；时间有限，直奔主题
>
> **B. 45–60 分钟 学术讲座/研讨班**（≤ 30 张幻灯片）
> — 院系 seminar；受众深度参与，需完整展示方法细节
>
> **C. 求职报告（Job Market Talk）**（≤ 20 张幻灯片）
> — 求职季使用；精炼呈现贡献、方法和主要结果"*

### 1.2 受众确认

> *"目标受众的领域背景：*
> *A. 同领域经济学家（可用技术语言和方程）*
> *B. 跨领域经济学家（需简化技术细节，强调直觉）*
> *C. 政策制定者 / 非经济学家（需要去专业化，强调政策含义）"*

### 1.3 主题风格

> *"幻灯片主题（或接受推荐）：*
> *A. Metropolis（推荐）— 简洁现代，当前学术界最流行*
> *B. 极简自定义（海军蓝 + 白色）— 稳重，适合求职和顶刊作者*
> *C. Madrid — 传统学术风格，适合保守场合*"*

---

## Step 2：幻灯片大纲设计

根据 Step 1 确认的演示类型，生成**幻灯片大纲**并向用户确认后再生成 LaTeX。

### 标准大纲（各类型幻灯片数量对照）

| 幻灯片板块 | A（15 min，≤15）| B（45 min，≤30）| C（求职，≤20）|
|----------|----------------|----------------|--------------|
| 标题页 | 1 | 1 | 1 |
| Motivation | 1–2 | 2–3 | 2–3 |
| This Paper（贡献预告）| 1 | 1 | 1 |
| Related Literature | — | 1–2 | 1–2 |
| Data | 1 | 2 | 2 |
| Identification Strategy | 2 | 3–4 | 3 |
| Main Results | 3 | 5–7 | 4–5 |
| Robustness | 1 | 2–3 | 2 |
| Heterogeneity & Mechanism | — | 2–3 | 1–2 |
| Conclusion / Takeaways | 1 | 1 | 1 |
| **合计上限** | **≤ 15** | **≤ 30** | **≤ 20** |

向用户呈现大纲并确认：

```
═══════════════════════════════════════════════════
演示大纲（[类型]，共 [N] 张，上限 [L] 张）
═══════════════════════════════════════════════════
[1]   Title
[2]   Motivation: [研究问题背景]
[3]   This Paper: [一句话问题 + 一句话发现]
[4]   Data & Sample
[5]   Identification Strategy
[6]   Validity Check: [策略对应的诊断图/检验]
[7]   Main Results（表格）
[8]   Main Results（图形/事件研究）
[9]   Economic Magnitude
[10]  Robustness
[11]  Takeaways
═══════════════════════════════════════════════════
```

待用户确认或修改大纲后，进入 Step 3 确认各页内容规范，再到 Step 4 生成 PPTX 文件。

---

## Step 3：各类幻灯片内容规范

> 📎 **输出格式规范参见 [`shared/output-standards.md`](../shared/output-standards.md)**

以下规范是经济学学术报告的约定俗成，逐条实施于每张幻灯片的内容生成。

---

### 3.1 标题页（Title Slide）

```latex
\begin{frame}[plain]
\titlepage
\end{frame}
```

标题格式要求：
- 论文标题：冒号后接副标题，副标题描述识别策略或数据
  - 好标题："The Effect of [X] on [Y]: Evidence from [Natural Experiment]"
  - 差标题："An Empirical Study of X and Y"（过于笼统）
- 作者行：姓名、机构（单位两行以内）
- 日期行：会议/研讨班名称 + 年月

---

### 3.2 Motivation（动机幻灯片，1–2 张）

**第 1 张：为什么关心这个问题**

每个条目不超过一行，用数字锚定重要性：

```latex
\begin{frame}{Why Does [TOPIC] Matter?}
\begin{itemize}
    \item<1-> \textbf{[政策/社会重要性]:} [一句话 + 具体数字]
    \item<2-> \textbf{[经济学理论相关性]:} [一句话]
    \item<3-> \textbf{[识别缺口]:} Existing work relies on [OLS/cross-section]
              — likely [overstates/understates] true effect
\end{itemize}
\vspace{1em}
\only<4>{
\begin{alertblock}{This paper's question}
    Does [D] causally affect [Y]?
\end{alertblock}
}
\end{frame}
```

**禁忌**：
- ❌ 超过 5 个条目
- ❌ 只有文字没有数字（"很多研究关注这个问题"）
- ❌ 第一张就放方程

---

### 3.3 This Paper（核心预告幻灯片，1 张，必须）

**这是经济学幻灯片中最重要的一张**，通常是第 2–3 张，让听众在 5 分钟内知道这场报告值不值得继续听。

必须包含：
1. 研究问题（一句话）
2. 识别策略（一句话，直觉层面）
3. 主要发现（一句话，含量级数字）
4. 贡献定位（1–2 条，简洁）

```latex
\begin{frame}{This Paper}

\textbf{Question:} Does [D] causally affect [Y]?

\vspace{0.8em}

\textbf{What we do:} Exploit [NATURAL EXPERIMENT / INSTRUMENT / CUTOFF]
to isolate exogenous variation in [D].
\quad $\Rightarrow$ [DATA SOURCE], [N] [UNITS], [YEARS]

\vspace{0.8em}

\textbf{Main finding:} [D] [increases/decreases] [Y] by
\alert{[MAGNITUDE] [UNITS/\%]},
[significant at X\% level].

\vspace{0.8em}

\textbf{Contribution:}
\begin{enumerate}
    \item First causal evidence on [TOPIC] using [METHOD]
    \item [SECOND CONTRIBUTION, if any — be brief]
\end{enumerate}

\end{frame}
```

---

### 3.4 Data（数据幻灯片，1–2 张）

**第 1 张：数据来源与样本（来自 `data-report.md §2`）**

```latex
\begin{frame}{Data}
\begin{columns}
\begin{column}{0.5\textwidth}
\textbf{Data Sources:}
\begin{itemize}
    \item [数据集1]：[描述，年份]
    \item [数据集2]：[描述，链接方式]
\end{itemize}
\vspace{0.5em}
\textbf{Sample:}
\begin{itemize}
    \item Unit: [个体/企业/县/省]
    \item Period: [起止年份]
    \item N = [样本量，加逗号分隔]
\end{itemize}
\end{column}
\begin{column}{0.5\textwidth}
\begin{table}
\caption{Summary Statistics}
\tiny
\begin{tabular}{lcc}
\toprule
Variable & Mean & SD \\
\midrule
[Y var]      & [值]  & [值] \\
[D var]      & [值]  & [值] \\
[Control 1]  & [值]  & [值] \\
\bottomrule
\end{tabular}
\end{table}
\end{column}
\end{columns}
\end{frame}
```

---

### 3.5 Identification Strategy（识别策略，2–3 张）

**第 1 张：直觉层面的识别逻辑（非技术）**

先给直觉，再给方程——不要一上来就放方程：

```latex
\begin{frame}{Identification Challenge}
\textbf{Problem:} [D] is endogenous because...
\begin{itemize}
    \item \textbf{OVB:} [一句话描述混淆路径]
    \item \textbf{Reverse causality:} [若适用]
\end{itemize}

\vspace{0.8em}

\textbf{Our solution:} Exploit [外生变异来源]

\begin{center}
\begin{tikzpicture}[node distance=2cm, auto]
% 简洁 DAG：D → Y，Z → D，Z ⊥ Y|D
\node (Z) {$Z$};
\node (D) [right of=Z] {$D$};
\node (Y) [right of=D] {$Y$};
\node (U) [above of=D] {$U$ (unobserved)};
\draw[->] (Z) -- (D) node[midway,below]{\tiny First stage};
\draw[->] (D) -- (Y);
\draw[->, dashed] (U) -- (D);
\draw[->, dashed] (U) -- (Y);
\draw[->, red, thick] (Z) to[bend left=40]
    node[above]{\tiny \textcolor{red}{Excluded}} (Y);
\end{tikzpicture}
\end{center}
\end{frame}
```

**第 2 张：识别假设 + 主方程**

```latex
\begin{frame}{Empirical Specification}
\textbf{Key assumption:} [识别假设一句话直觉表述]

\vspace{0.5em}

\begin{block}{Main Equation}
\begin{equation*}
    [主方程 LaTeX — 来自 model-spec.md §2，简化版]
\end{equation*}
\end{block}

\vspace{0.5em}

\begin{itemize}
    \item $[β]$: [系数的经济学含义]
    \item SE clustered at [聚类层级]
    \item [FE 说明]
\end{itemize}

\vspace{0.5em}

\textbf{Evidence for assumption:} $\Rightarrow$ [下一张幻灯片]
\end{frame}
```

**第 3 张：识别假设检验（策略特异性）**

| 识别策略 | 专属视觉化幻灯片 | 图形文件 |
|---------|---------------|---------|
| DiD / 交错 DiD | 事件研究图（Event Study Plot）| `figures/fig*_event_study.pdf` |
| RDD | 断点处散点图（Binscatter at Cutoff）| `figures/fig*_rdd_binscatter.pdf` |
| IV / 2SLS | 第一阶段散点图（First Stage Scatter）| `figures/fig*_iv_scatter.pdf` |
| 面板 FE | 平行趋势目视图 | `figures/fig*_parallel_trends.pdf` |
| 合成控制 | 预处理期拟合图 | `figures/fig*_sc_gap.pdf` |

```latex
% DiD 事件研究图幻灯片示例
\begin{frame}{Parallel Trends: Event Study}
\begin{center}
    \includegraphics[width=0.85\textwidth]{figures/fig01_event_study.pdf}
\end{center}
\vspace{-0.5em}
{\small \textit{Notes:} Coefficients from Eq.~(\ref{eq:eventstudy}).
95\% CI. Pre-period joint F-test: $p = [值]$ (cannot reject $H_0$: all pre-period coefficients $= 0$).}
\end{frame}
```

---

### 3.6 Main Results（主要结果，2–3 张）

**原则：用图而不是表，如果必须用表则精简到极致。**

**第 1 张：主结果表（精简版）**

幻灯片中的回归表**必须精简**，规则：
- 最多 3–4 列（不要把完整的 6 列表格搬上来）
- 只保留主系数行 + SE + N + FE 指示器
- 用 `\alert{}` 或加粗突出首选规格的主系数
- 字号：`\footnotesize` 或 `\scriptsize`

```latex
\begin{frame}{Main Results}
\begin{table}
\centering
\footnotesize
\begin{tabular}{lccc}
\toprule
 & (1) & (2) & \textbf{(3)} \\
 & Baseline & + Controls & \textbf{Preferred} \\
\midrule
[D var] & [β̂_1]*** & [β̂_2]*** & \alert{[β̂_3]***} \\
        & ([SE_1])  & ([SE_2])  & \alert{([SE_3])}  \\
\midrule
FE & No & No & Yes \\
$N$ & [N1] & [N2] & [N3] \\
\bottomrule
\multicolumn{4}{l}{\tiny SE clustered at [level]. *p<0.1 **p<0.05 ***p<0.01}
\end{tabular}
\end{table}

\vspace{0.3em}
\textbf{Takeaway:} [D] [increases/decreases] [Y] by
\alert{[COEFFICIENT] [UNITS]}.
\end{frame}
```

**第 2 张：经济量级（Economic Magnitude）**

单独一张幻灯片专门讲量级——这是听众最想要的信息：

```latex
\begin{frame}{Economic Magnitude}

\begin{center}
\Large
\textbf{[D] [increases/decreases] [Y] by \alert{[MAGNITUDE]}}
\end{center}

\vspace{1em}

\begin{itemize}
    \item Relative to mean [Y] of [MEAN]: \alert{[PCT]\% change}
    \item In SD units: \alert{[SD-COEF] $\sigma_{Y}$}
              ([小/中/大] effect by Cohen's benchmark)
    \item[vs.] Closest prior estimate ([AUTHOR Year]):
              [他们的系数]
              — our estimate is [larger/smaller] because [识别策略差异]
\end{itemize}

\vspace{0.5em}
{\small \textit{Policy implication:} [一句话政策含义]}
\end{frame}
```

**第 3 张（若 DiD）：动态效应图（Event Study）**

直接引用 `figures/fig*_event_study.pdf`，比静态系数更有说服力：

```latex
\begin{frame}{Dynamic Effects}
\begin{center}
    \includegraphics[width=0.85\textwidth]{figures/fig01_event_study.pdf}
\end{center}
\vspace{-0.5em}
{\small \textbf{Takeaway:} No pre-trend. Effect appears [immediately/gradually]
after treatment. [Long-run/Short-run] effect: [β̂_long] ([SE], [sig]).}
\end{frame}
```

---

### 3.7 Robustness（稳健性，1–3 张）

**原则：结论先行，不要把所有检验逐条念一遍。**

```latex
\begin{frame}{Robustness}

\textbf{Main finding is robust to:}

\begin{columns}[t]
\begin{column}{0.5\textwidth}
\textbf{Inference:}
\begin{itemize}
    \item[\checkmark] HC3 robust SE
    \item[\checkmark] Two-way clustered SE
    \item[\checkmark] Wild bootstrap (if < 30 clusters)
\end{itemize}
\vspace{0.5em}
\textbf{Sample:}
\begin{itemize}
    \item[\checkmark] Exclude top/bottom 1\% of $Y$
    \item[\checkmark] Drop high-influence observations
\end{itemize}
\end{column}
\begin{column}{0.5\textwidth}
\textbf{Specification:}
\begin{itemize}
    \item[\checkmark] Log transformation
    \item[\checkmark] Alternative control sets
    \item[\checkmark] Oster $\delta = [值] \gg 1$
\end{itemize}
\vspace{0.5em}
\textbf{Identification:}
\begin{itemize}
    \item[\checkmark] Placebo [treatment/cutoff/instrument]
    \item[\checkmark] [策略特异性检验]
\end{itemize}
\end{column}
\end{columns}

\vspace{0.3em}
\small Coefficients range from [MIN] to [MAX] across all specifications.
\end{frame}
```

---

### 3.8 Takeaways（结论幻灯片，1 张）

**经济学报告的最后一张幻灯片绝对不能是 "Thank you / Questions?"**，必须是内容幻灯片，让听众带着结论离开。

```latex
\begin{frame}{Takeaways}

\begin{enumerate}
    \item \textbf{Causal evidence:}
          [D] [causes] [Y] to [increase/decrease] by
          \alert{[MAGNITUDE]}.
          [量级的直觉类比]

    \item \textbf{Why it matters:}
          [政策含义 / 理论贡献，一句话]

    \item \textbf{Mechanism:}
          [机制证据一句话（若有）]
\end{enumerate}

\vspace{1.5em}

\begin{center}
\normalsize [Author Name] \quad \texttt{[email]}
\end{center}

\end{frame}
```

---

## Step 4：PPTX 文件生成（Beamer 学术风格）

调用 `beamer-ppt` skill，使用 **python-pptx** 按 Step 3 的内容规范逐张生成幻灯片，输出 `.pptx` 文件。

### 4.1 配色主题方案

按 Step 1.3 用户选定的主题使用以下 RGB 配色：

| 主题 | 标题栏背景 | 强调色 | 正文背景 |
|------|-----------|--------|---------|
| **A. Metropolis（推荐）** | 海军蓝 `(0, 35, 82)` | 强调红 `(180, 30, 30)` | 浅灰 `(245, 245, 245)` |
| **B. 极简自定义** | 海军蓝 `(0, 35, 82)` | 海军蓝 `(0, 35, 82)` | 白色 `(255, 255, 255)` |
| **C. Madrid（传统）** | 深蓝 `(31, 73, 125)` | 金色 `(189, 152, 44)` | 白色 `(255, 255, 255)` |

### 4.2 幻灯片母版基础设置

```python
from pptx import Presentation
from pptx.util import Inches, Pt, Emu
from pptx.dml.color import RGBColor
from pptx.enum.text import PP_ALIGN
from pptx.enum.shapes import MSO_SHAPE_TYPE

# 16:9 宽屏（对应 Beamer aspectratio=169）
prs = Presentation()
prs.slide_width  = Inches(13.33)
prs.slide_height = Inches(7.5)

# 配色（以 Metropolis 主题为例）
NAVY  = RGBColor(0, 35, 82)
RED   = RGBColor(180, 30, 30)
LGRAY = RGBColor(245, 245, 245)
WHITE = RGBColor(255, 255, 255)
BLACK = RGBColor(30, 30, 30)
MGRAY = RGBColor(100, 100, 100)
```

### 4.3 标题栏辅助函数（Frame Title）

每张内容幻灯片必须有海军蓝标题栏（高 1.1 英寸，白色加粗 24pt），模拟 Beamer `\frametitle`：

```python
from pptx.util import Inches, Pt
from pptx.dml.color import RGBColor

def add_slide_bg(slide, prs, color):
    """全屏背景色块"""
    bg = slide.shapes.add_shape(
        1, 0, 0, prs.slide_width, prs.slide_height)
    bg.fill.solid()
    bg.fill.fore_color.rgb = color
    bg.line.fill.background()
    return bg

def add_frame_title(slide, prs, title_text, bg_color=None, fg_color=None):
    """Beamer 风格标题栏：海军蓝背景 + 白色粗体标题"""
    bg_color = bg_color or RGBColor(0, 35, 82)
    fg_color = fg_color or RGBColor(255, 255, 255)
    bar = slide.shapes.add_shape(
        1, 0, 0, prs.slide_width, Inches(1.1))
    bar.fill.solid()
    bar.fill.fore_color.rgb = bg_color
    bar.line.fill.background()
    tf = bar.text_frame
    tf.word_wrap = False
    p = tf.paragraphs[0]
    p.text = title_text
    p.font.bold = True
    p.font.size = Pt(24)
    p.font.color.rgb = fg_color
    p.alignment = PP_ALIGN.LEFT
    # 左边距
    from pptx.util import Inches
    tf.margin_left = Inches(0.3)
    tf.margin_top  = Inches(0.2)
```

### 4.4 底部进度条（Metropolis 风格）

```python
def add_progress_bar(slide, prs, current, total):
    """Metropolis 风格底部进度条（可选）"""
    bar_h = Inches(0.06)
    bar_top = prs.slide_height - bar_h
    # 灰色底条
    bg_bar = slide.shapes.add_shape(
        1, 0, bar_top, prs.slide_width, bar_h)
    bg_bar.fill.solid()
    bg_bar.fill.fore_color.rgb = RGBColor(200, 200, 200)
    bg_bar.line.fill.background()
    # 海军蓝进度条
    prog_w = int(prs.slide_width * current / total)
    prog = slide.shapes.add_shape(1, 0, bar_top, prog_w, bar_h)
    prog.fill.solid()
    prog.fill.fore_color.rgb = RGBColor(0, 35, 82)
    prog.line.fill.background()
```

### 4.5 逐类幻灯片生成

按 Step 2 大纲依次生成，每张幻灯片内容来自 Step 0.2 的**内容素材库**。

#### 标题页（深色背景全屏）

```python
def make_title_slide(prs, title, subtitle, author, institute, date):
    slide = prs.slides.add_slide(prs.slide_layouts[6])  # blank
    add_slide_bg(slide, prs, NAVY)
    # 论文标题（白色，36pt，加粗，居中）
    tb = slide.shapes.add_textbox(Inches(1), Inches(1.8), Inches(11.33), Inches(2))
    tf = tb.text_frame; tf.word_wrap = True
    p = tf.paragraphs[0]
    p.text = title; p.font.bold = True
    p.font.size = Pt(36); p.font.color.rgb = WHITE
    p.alignment = PP_ALIGN.CENTER
    # 副标题（浅灰，22pt）
    p2 = tf.add_paragraph()
    p2.text = subtitle; p2.font.size = Pt(22)
    p2.font.color.rgb = LGRAY; p2.alignment = PP_ALIGN.CENTER
    # 作者 + 机构（白色，18pt）
    tb2 = slide.shapes.add_textbox(Inches(1), Inches(4.5), Inches(11.33), Inches(1.2))
    tf2 = tb2.text_frame
    tf2.paragraphs[0].text = author
    tf2.paragraphs[0].font.size = Pt(18); tf2.paragraphs[0].font.color.rgb = WHITE
    tf2.paragraphs[0].alignment = PP_ALIGN.CENTER
    p3 = tf2.add_paragraph()
    p3.text = institute; p3.font.size = Pt(16); p3.font.color.rgb = LGRAY
    p3.alignment = PP_ALIGN.CENTER
    # 日期/会议（浅灰，14pt）
    tb3 = slide.shapes.add_textbox(Inches(1), Inches(6.2), Inches(11.33), Inches(0.8))
    tf3 = tb3.text_frame
    tf3.paragraphs[0].text = date
    tf3.paragraphs[0].font.size = Pt(14); tf3.paragraphs[0].font.color.rgb = LGRAY
    tf3.paragraphs[0].alignment = PP_ALIGN.CENTER
    return slide
```

#### 通用内容幻灯片（条目列表）

```python
def make_content_slide(prs, title, bullets, current=None, total=None):
    """标题栏 + 条目列表正文"""
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    add_slide_bg(slide, prs, LGRAY)
    add_frame_title(slide, prs, title)
    # 正文区
    tb = slide.shapes.add_textbox(
        Inches(0.5), Inches(1.3), Inches(12.33), Inches(5.8))
    tf = tb.text_frame; tf.word_wrap = True
    for i, (lvl, text) in enumerate(bullets):
        p = tf.paragraphs[i] if i == 0 else tf.add_paragraph()
        p.text = text; p.level = lvl
        p.font.size = Pt(20 if lvl == 0 else 17)
        p.font.color.rgb = BLACK
        p.space_before = Pt(6 if lvl == 0 else 3)
    if current and total:
        add_progress_bar(slide, prs, current, total)
    return slide
```

#### 图形幻灯片

```python
def make_figure_slide(prs, title, img_path, caption="", current=None, total=None):
    """标题栏 + 居中图形 + 图注"""
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    add_slide_bg(slide, prs, LGRAY)
    add_frame_title(slide, prs, title)
    slide.shapes.add_picture(
        img_path, Inches(1.17), Inches(1.3), Inches(11), Inches(5.2))
    if caption:
        cap = slide.shapes.add_textbox(
            Inches(0.5), Inches(6.6), Inches(12.33), Inches(0.7))
        cap.text_frame.paragraphs[0].text = caption
        cap.text_frame.paragraphs[0].font.size = Pt(11)
        cap.text_frame.paragraphs[0].font.color.rgb = MGRAY
    if current and total:
        add_progress_bar(slide, prs, current, total)
    return slide
```

#### 回归表幻灯片

```python
def make_table_slide(prs, title, headers, rows, footnote="",
                     highlight_col=-1, current=None, total=None):
    """标题栏 + 精简回归表（≤4列）"""
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    add_slide_bg(slide, prs, LGRAY)
    add_frame_title(slide, prs, title)
    nc = len(headers); nr = len(rows) + 1
    tbl = slide.shapes.add_table(
        nr, nc, Inches(0.5), Inches(1.4), Inches(12.33), Inches(4.5)).table
    # 表头：海军蓝背景，白色字体
    for j, h in enumerate(headers):
        cell = tbl.cell(0, j)
        cell.text = h
        cell.text_frame.paragraphs[0].font.bold = True
        cell.text_frame.paragraphs[0].font.size = Pt(14)
        cell.text_frame.paragraphs[0].font.color.rgb = WHITE
        cell.fill.solid(); cell.fill.fore_color.rgb = NAVY
    # 数据行
    for i, row in enumerate(rows):
        for j, val in enumerate(row):
            cell = tbl.cell(i + 1, j)
            cell.text = str(val)
            cell.text_frame.paragraphs[0].font.size = Pt(13)
            if j == (nc + highlight_col) % nc:   # 首选规格列加粗
                cell.text_frame.paragraphs[0].font.bold = True
    # 脚注
    if footnote:
        fn = slide.shapes.add_textbox(
            Inches(0.5), Inches(6.0), Inches(12.33), Inches(1.2))
        fn.text_frame.paragraphs[0].text = footnote
        fn.text_frame.paragraphs[0].font.size = Pt(10)
        fn.text_frame.paragraphs[0].font.color.rgb = MGRAY
    if current and total:
        add_progress_bar(slide, prs, current, total)
    return slide
```

### 4.6 图形文件预处理

若 `figures/` 目录存在 **PDF 向量图**，先转为高分辨率 PNG 再嵌入 PPTX：

```python
import subprocess, os

def pdf_to_png(pdf_path, dpi=200):
    """PDF → PNG（需系统安装 poppler）"""
    png_base = pdf_path.replace(".pdf", "")
    subprocess.run([
        "pdftoppm", "-r", str(dpi), "-png", "-singlefile",
        pdf_path, png_base
    ], check=True)
    return png_base + ".png"

# 若 pdftoppm 不可用，回退 pdf2image
def pdf_to_png_fallback(pdf_path, dpi=200):
    from pdf2image import convert_from_path
    imgs = convert_from_path(pdf_path, dpi=dpi)
    png_path = pdf_path.replace(".pdf", ".png")
    imgs[0].save(png_path, "PNG")
    return png_path
```

安装依赖：

```bash
pip install python-pptx pdf2image --break-system-packages
apt-get install -y poppler-utils 2>/dev/null || true
```

### 4.7 演讲者备注

为每张关键幻灯片在 `notes_slide` 中添加备注，包含：核心论点（1–2 句）、预期Q&A 回答、时间提示。

```python
def add_speaker_notes(slide, notes_text):
    notes_slide = slide.notes_slide
    tf = notes_slide.notes_text_frame
    tf.text = notes_text
```

---

## Step 5：文件输出

### 5.1 文件目录

```
slides/
├── slides.pptx          # Beamer 风格 PPTX（可用 PowerPoint / LibreOffice 编辑）
└── slides.pdf           # PDF 版本（用于投稿、邮件分发、归档）
```

### 5.2 保存 PPTX 文件

```python
import os

output_dir = os.path.join(workspace, "slides")
os.makedirs(output_dir, exist_ok=True)

pptx_path = os.path.join(output_dir, "slides.pptx")
prs.save(pptx_path)
print(f"✅ PPTX 已保存：{pptx_path}")
print(f"   共 {len(prs.slides)} 张幻灯片")
```

### 5.3 导出 PDF

用 LibreOffice headless 将 PPTX 转为 PDF（推荐，保留字体和布局）：

```python
import subprocess, shutil

def pptx_to_pdf(pptx_path, output_dir):
    """使用 LibreOffice headless 将 PPTX 转为 PDF"""
    result = subprocess.run(
        ["libreoffice", "--headless", "--convert-to", "pdf",
         "--outdir", output_dir, pptx_path],
        capture_output=True, text=True
    )
    pdf_path = pptx_path.replace(".pptx", ".pdf")
    if os.path.exists(pdf_path):
        print(f"✅ PDF 已导出：{pdf_path}")
        return pdf_path
    else:
        print(f"⚠️  LibreOffice 转换失败：{result.stderr}")
        return None

pdf_path = pptx_to_pdf(pptx_path, output_dir)
```

若 LibreOffice 不可用，回退方案（需安装 `unoconv` 或 `soffice`）：

```bash
# 检查 LibreOffice 是否可用
which libreoffice soffice 2>/dev/null || echo "未找到 LibreOffice"

# Ubuntu/Debian 安装
apt-get install -y libreoffice 2>/dev/null || true
```

### 5.4 验证检查

```python
from pptx import Presentation

# 验证 PPTX
verify = Presentation(pptx_path)
assert len(verify.slides) > 0, "PPTX 文件为空！"
pptx_kb = os.path.getsize(pptx_path) // 1024

# 验证 PDF（若已生成）
pdf_ok = pdf_path and os.path.exists(pdf_path)
pdf_kb = os.path.getsize(pdf_path) // 1024 if pdf_ok else 0

print(f"✅ 验证通过")
print(f"   PPTX：{len(verify.slides)} 张，{pptx_kb} KB")
if pdf_ok:
    print(f"   PDF ：{pdf_kb} KB")
else:
    print(f"   PDF ：未生成（可在 PowerPoint 中手动导出）")
```

---

## Step 6：阶段确认与移交

> **演示文稿产出摘要**
>
> ✅ Beamer 风格幻灯片：`slides/slides.pptx`（共 [N] 张，上限 [L] 张）
> ✅ PDF 版本：`slides/slides.pdf`（用于投稿、邮件分发、归档）
> ✅ 主题：[A Metropolis / B 极简 / C Madrid]，16:9 宽屏，海军蓝配色
> ✅ PPTX 可直接用 PowerPoint 或 LibreOffice Impress 打开和编辑
>
> ---
>
> **幻灯片质量自检（报告前建议核查）：**
> - [ ] 总张数是否在上限以内（A ≤ 15 / B ≤ 30 / C ≤ 20）？
> - [ ] "This Paper" 幻灯片是否包含主系数数值和量级？
> - [ ] 回归表是否精简到 ≤ 3–4 列？首选规格列是否加粗高亮？
> - [ ] 最后一张是否为 Takeaways（不是 "Thank you / Questions?"）？
> - [ ] 图形是否以 ≥ 200 DPI PNG 嵌入，清晰无锯齿？
> - [ ] 每张关键幻灯片是否有演讲者备注？
> - [ ] 总时长估算：[N 张] × 1.5 分钟/张 ≈ [总分钟数]，是否合理？

---

## 常见问题处理

**Q：图形文件不存在（`figures/` 目录为空），如何处理？**

先运行 `/plot` 命令（Phase 7 的绘图输出），生成所有分析图形，再运行 `/present`。若用户希望先生成幻灯片框架，用占位矩形替代：

```python
# 占位图（灰色矩形 + 说明文字）
from pptx.util import Inches, Pt
ph = slide.shapes.add_shape(
    1, Inches(1), Inches(1.5), Inches(11.33), Inches(4.5))
ph.fill.solid(); ph.fill.fore_color.rgb = RGBColor(200, 200, 200)
ph.line.fill.background()
tf = ph.text_frame
tf.paragraphs[0].text = "[图形占位：figures/figNN_name.png]"
tf.paragraphs[0].font.size = Pt(18)
tf.paragraphs[0].font.color.rgb = RGBColor(100, 100, 100)
```

**Q：python-pptx 未安装？**

```bash
pip install python-pptx pdf2image --break-system-packages
```

**Q：PDF 图形转换失败？**

检查是否安装了 poppler：`which pdftoppm`。若未安装：

```bash
# macOS
brew install poppler
# Ubuntu/Debian
apt-get install poppler-utils
```

若环境受限，直接将 PNG 图形（来自 `/plot` 阶段输出的 `figures/*.png`）嵌入，跳过 PDF→PNG 转换。
