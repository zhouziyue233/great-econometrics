# 上游输出文件读取规范 (Context Reader Protocol)

## 目的

本模块为 5 个命令提供**统一的文件读取与验证流程**，避免重复代码和维护混乱。任何涉及上游输出的命令应遵循本规范。

---

## 第一部分：上游输出文件完整注册表

| 文件名 | 产出阶段 | 简述 |
|--------|---------|------|
| `research-question.md` | Phase 1 `/question` | 研究问题、政策背景、实际意义 |
| `literature-review-report.md` | Phase 2 `literature-review` | 文献综述、研究空白、本文贡献定位 |
| `identification-memo.md` | Phase 3 `/analyze` | 识别策略选择、假设、可检验形式、三层辩护 |
| `data-report.md` | Phase 4 `/data` | 数据来源、样本描述、EDA、数据质量预警 |
| `model-spec.md` | Phase 5 `/model` | 模型方程（LaTeX）、SE 策略、诊断规格 |
| `diagnostic_report.md` | Phase 6 `/code` | 统计诊断结果（✅/⚠️/❌） |
| `results-memo.md` | Phase 7 `results-analysis` | 系数、显著性、经济显著性、识别可信度 |
| `robustness-report.md` | Phase 8 `/robustness` | 稳健性、异质性、机制检验结果 |

---

## 第二部分：文件读取标准流程

### 2.1 检查与读取

对每个**必需文件**，执行以下步骤：

```
1. 使用 Read 工具尝试打开文件
   Read: [workspace]/[filename].md

2. 若文件存在 → 读取并解析关键信息（见 2.2）

3. 若文件不存在且为"必需" → 立即停止
   显示模板错误信息：
   > "未找到 [filename]。请先完成 Phase [X]（`[命令名]`）。"

4. 若文件不存在但为"可选" → 记录警告但继续
   显示模板警告信息：
   > "未找到 [filename]。将基于其他可用信息继续，但功能可能受限。"
```

### 2.2 信息提取与结构化

读取后，将关键信息提取到**结构化确认块**中，格式如下：

```
═══════════════════════════════════════════
Phase [N] [阶段名] 启动确认
═══════════════════════════════════════════
[关键字段 1]：[提取值]
[关键字段 2]：[提取值]
...
═══════════════════════════════════════════
```

确认块中的字段完全由该命令的具体需求决定（见下文第三部分）。

---

## 第三部分：命令级需求规范

### `/analyze` (Phase 3)

**必需文件**：
- `research-question.md`（必需）
- `literature-review-report.md`（必需）

**提取内容**：
- `research-question.md` → 研究问题、研究对象、初判识别层级、数据源
- `literature-review-report.md` → 文献主流方法、代表性估计量、已知威胁

**确认块名称**：Phase 3 研究背景信息

---

### `/model` (Phase 5)

**必需文件**：
- `identification-memo.md`（必需）

**可选文件**：
- `data-report.md`（可选但强烈推荐）

**提取内容**：
- `identification-memo.md` → 策略类型、结果变量 Y、处理变量 D、识别变量 Z、控制变量列表、目标参数、面板结构
- `data-report.md` → 样本量 N、面板维度、关键变量缺失率、平衡性检验结果、EDA 预警

**确认块名称**：Phase 5 启动确认

---

### `/code` (Phase 6)

**必需文件**：
- `identification-memo.md`（必需）
- `data-report.md`（必需）
- `model-spec.md`（必需）

**提取内容**：
- `model-spec.md` → 策略类型、主方程、Y/D/Z/控制变量名、FE 层级、SE 类型与聚类变量、目标参数
- `data-report.md` → 清洗后数据路径、样本量、面板结构
- `identification-memo.md` → 可检验的识别假设（用于诊断代码）

**确认块名称**：Phase 6 启动确认

---

### `/robustness` (Phase 8)

**必需文件**：
- `model-spec.md`（必需）

**可选文件**：
- `diagnostic_report.md`（可选）
- `results-memo.md`（可选）

**提取内容**：
- `model-spec.md` → 策略类型、识别假设及诊断状态、预列稳健性检验
- `diagnostic_report.md` → 第二层统计诊断 ⚠️ 项（自动转为稳健性检验）
- `results-memo.md` → Phase 7 建议的优先检验

**确认块名称**：Phase 8 启动确认

---

### `/write` (Phase 9)

**必需文件**：
- `model-spec.md`（必需）
- `results-memo.md`（必需）

**可选文件**：
- `research-question.md`
- `literature-review-report.md`
- `identification-memo.md`
- `data-report.md`
- `robustness-report.md`
- `tables/table_main.csv`

**提取内容**：
- `model-spec.md` → 主方程 LaTeX、符号说明、标准误策略
- `results-memo.md` → 主系数、显著性、经济显著性、识别可信度、外部有效性
- 可选文件 → 按需补充各节的具体细节和数字

**确认块名称**：Phase 9 写作信息底稿

---

## 第四部分：通用错误处理

### 文件缺失时的标准错误消息

**模板格式**：

```
未找到 [filename]。
请先完成 Phase [X]（`[command 或 skill]`），产出 [filename]。
```

**具体例示**：

```
未找到 identification-memo.md。
请先完成 Phase 3（`/analyze`），产出 identification-memo.md。
```

### 文件存在但内容不完整时

若关键字段缺失，记录警告但不停止：

```
⚠️ [filename] 中缺失关键字段 [fieldname]。
将使用默认值继续，但建议检查上游阶段的完整性。
```

---

## 第五部分：命令 ↔ 文件依赖矩阵

| 命令 | 必需文件 | 可选文件 | 调用时机 |
|------|---------|---------|---------|
| `/analyze` | research-question.md, literature-review-report.md | — | Step 0 |
| `/model` | identification-memo.md | data-report.md | Step 0 |
| `/code` | identification-memo.md, data-report.md, model-spec.md | — | Step 0 |
| `/robustness` | model-spec.md | diagnostic_report.md, results-memo.md | Step 0 |
| `/write` | model-spec.md, results-memo.md | research-question.md, literature-review-report.md, identification-memo.md, data-report.md, robustness-report.md | Step 0 |

---

## 使用范例

### 范例：`/model` 命令的 Step 0

```markdown
## Step 0：读取上游输出

> 📎 **参见 [`shared/context-reader.md`](../shared/context-reader.md)**
> 本阶段所需文件：`identification-memo.md`（必需）、`data-report.md`（可选）。

读取 `identification-memo.md` 和 `data-report.md`（若存在）：

[Extract information from files using Read tool]

提取完成后向用户确认：

> **Phase 5 启动确认**
>
> 识别策略：**[策略名称]**
> 目标参数：**[ATE / ATT / LATE]**
> 结果变量 Y：`[变量名]`
> 处理变量 D：`[变量名]`
> 样本：[N] 观测，[个体数] 个个体 × [时期数] 期
```

---

## 维护说明

本文档为所有涉及文件读取的命令的唯一权威来源。若需：

- **新增命令**：在第三部分添加对应的"命令级需求规范"小节
- **修改文件名**：同步更新第一部分注册表
- **调整依赖**：同步更新第五部分矩阵

版本历史：
- v1.0 (2026-03-30)：初版，覆盖 5 条命令、8 类文件
