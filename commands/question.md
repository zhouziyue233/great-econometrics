---
name: question
description: Help user transform their idea into a clear research question through interactive dialogue.
tools:
  - Read
  - Write
  - Bash
---

# /question — Research Question Scoping

When the user invoke /question, transforms their preliminary idea into a confirmed, well-scoped research question through focused and multi-round conversations. The output is a single clear research question statement.

---

## Step 1: Draw Out the Idea

Ask the user to describe their idea in 1–3 sentences. Frame it as low-stakes:

> "请用1–3句话描述你初步的研究想法。"

After they respond, identify the most unclear part and ask **one targeted follow-up** — focusing on whichever is most ambiguous:

- **谁受影响？**（研究对象：个体、企业、地区、国家？）
- **什么在变化？**（关键自变量或政策冲击是什么？）
- **关心什么结果？**（因变量：工资、健康、生产率、犯罪？）

Do not ask all three at once. One follow-up at a time.

---

## Step 2: Identify the Motivation

Once you understand the core idea, tell the user which type of research motivation this is. This helps them articulate the "why this paper" framing later.

| 类型 | 描述 | 例子 |
|------|------|------|
| **Type 1: 有趣的现象** | 数据中观察到的规律，需要解释 | 靠近高速公路的农村土地反而规模化更快 |
| **Type 2: 理论与现实的矛盾** | 理论预测X，但现实显示¬X | 最低工资理论上减少就业，但实证结果不一致 |
| **Type 3: 新制度/政策/技术** | 外生变化创造了研究机会 | 某省2015年推行宅基地改革 |
| **Type 4: 新数据** | 过去无法回答的问题现在数据可及 | 首次获得某平台的个体级交易数据 |

Say something like: *"你的研究动机最接近 Type X——[简短解释]。"*

---

## Step 3: Quick Check

Apply three quick checks. Flag any failure immediately.

**可检验 (Testable)**
Can the question be stated as "X 对 Y 的影响"，where both X and Y are measurable variables?

- ✅ Pass: both X and Y can be operationalized as data columns
- ❌ Fail: question is too conceptual, normative, or tautological → help the user operationalize

**可识别 (Identifiable)**
Is there a plausible source of variation to answer this question credibly?

- ✅ Level A (因果): 有准实验变异（政策、阈值、工具变量）
- ✅ Level B (可信相关): 面板数据 + 固定效应 + 充分控制
- ⚠️ Level C (描述): 截面OLS，需降低因果主张的力度
- ❌ Fail: 完全内生，且无可用变异 → 需重新设计或降格为描述性研究

Be honest about the identification level. If it's Level C, say so clearly and note what it means for the kind of claims the paper can make.

**可操作 (Feasible)**
Is the required data obtainable and the method executable?

- Is there a publicly available dataset, or does the user have their own data?
- Is the scope answerable in one paper (one main question, not five)?

If any check fails, help the user either reformulate or scope down before proceeding.

---

## Step 4: Confirm the Research Question

Once the three checks pass (or the user accepts a scoped-down version), produce the confirmed research question in this format:

```
研究问题：[X] 如何影响 [Y]？
研究对象：[总体/单位]，[时间/地域范围]
识别层级：Level [A/B/C]
数据来源（初步）：[数据集名称或类型]

研究假设：
H₀：X 对 Y 无显著影响（β = 0）
H₁：X 对 Y 有 [正向/负向/显著] 影响（β ≠ 0）
预期方向：[理论或直觉上的预期，简要说明原因]
```

Then ask: *"这个研究问题符合你的想法吗？有什么需要调整的？"*

After the user explicitly confirms, save the output as a `research-question.md` document in your working directory. Then move to next phase `literature-review`.

---

## Common Reformulation Examples

**Too broad → Narrowed**

- ❌ "数字化对经济的影响"
- ✅ "工业机器人的普及是否降低了中国制造业城市的就业率（2000–2015）？"

**Fails identifiable → Redesigned**
- ❌ "ESG评分高的企业财务表现更好吗？"（OLS内生）
- ✅ "强制ESG披露政策（2008年深交所要求）是否改善了中小上市公司的财务绩效？"（DiD）

**Too vague → Operationalized**
- ❌ "教育对人的影响"
- ✅ "义务教育年限延长对个人30岁时工资的影响（利用教育改革作为工具变量）"
