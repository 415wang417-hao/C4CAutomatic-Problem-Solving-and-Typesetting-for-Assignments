---
AIGC:
    Label: "1"
    ContentProducer: 001191440300708461136T1XGW3
    ProduceID: 60ea5d0126731e33de81577b27285889_8fb62ccdbff911f1887c525400de85a5
    ReservedCode1: HSd+CpIC4BydG5ofJqKUVvWL0hVwAEaYBs49/WOC+1kKYK07vbPKrvXqpPpFAxOw8eN4hJKfJwAtCIiEOCNbzj+26XvPHKw7Omi705D9EYm0j5VyvR1cOSdHfqt32zQXRBaKHp98OzZPRSJ0XG4RQZm9J/IRBfBdP5yRNZEQezzyjFtoeCI5Davar6I=
    ContentPropagator: 001191440300708461136T1XGW3
    PropagateID: 60ea5d0126731e33de81577b27285889_8fb62ccdbff911f1887c525400de85a5
    ReservedCode2: HSd+CpIC4BydG5ofJqKUVvWL0hVwAEaYBs49/WOC+1kKYK07vbPKrvXqpPpFAxOw8eN4hJKfJwAtCIiEOCNbzj+26XvPHKw7Omi705D9EYm0j5VyvR1cOSdHfqt32zQXRBaKHp98OzZPRSJ0XG4RQZm9J/IRBfBdP5yRNZEQezzyjFtoeCI5Davar6I=
---

# C4C 作业自动求解与排版 —— 方案设计文档

- 提交人：**lenovo**
- 挑战：C4 技能分享与传播 · C4C 作业自动求解与排版
- 基线：`c4c-homework-solver-starter`（Claude 版流水线，493 KB）
- 交付版本：**v2.0 国产模型迁移版**（Qwen 3.6 / Kimi 2.5）
- 日期：2026-10-04

---

## 1. 任务理解

starter kit 已经跑通一条「作业文件 → 可提交 PDF」的 5 段式流水线，但能力被锁死在两件事上：

1. **推理引擎绑定 Claude**：概念题/证明题/文字题只有 `LLM` 占位实现，离线时直接返回"无法求解"；
2. **求解域只有微积分极限**：`solve_matrix` / `solve_ode` / `solve_proof` 均为空实现（starter 自身代码注释即写明是学生的扩展点）。

因此本挑战的工程本质是两件事：**换引擎（Claude → 国产大模型）** + **扩域（极限 → 线代 / ODE / 大学物理）**，且要求产出真实作业的可提交 PDF。

## 2. 总体架构

保留 starter 的「本体驱动（ontology-grounded）」骨架不动，只做加法：

```
输入作业(.md/.pdf/.docx/图片)
   │
   ├─ Stage 1  Ingest           ingest.py            文本摄取 + 分段边界
   ├─ Stage 2  Parse + Classify parse_problems.py → classify.py → retrieve.py
   │                          （T-box 概念图 → 求解器模板检索）
   ├─ Stage 3  Solve            solve.py（调度） + domain_solvers.py（新增域）
   │                                     + llm_engine.py（国产模型，新增）
   ├─ Stage 4  Render           render_latex.py + latex_utils.py（新增）
   └─ Stage 5  Compile          xelatex/pdflatex（含 Windows 降权兜底）
   │
   └─ 旁路校验  validate.py（新增，独立复算）
   → homework.pdf / <prefix>.pdf
```

### 相对 starter kit 的改动清单

| 文件 | 状态 | 说明 |
|------|------|------|
| `scripts/domain_solvers.py` | **新增（42 KB）** | 三大新学科的求解器：`solve_linear_algebra` / `solve_ode_domain` / `solve_physics` / `solve_by_spec` |
| `scripts/llm_engine.py` | **新增（12 KB）** | 国产模型客户端：DeepSeek / Qwen / Kimi 三端点，OpenAI 兼容协议，带缓存、重试、用量记账、离线降级 |
| `scripts/latex_utils.py` | **新增（17 KB）** | LaTeX→SymPy 解析层：矩阵环境、导数记号、物理量与单位解析（不依赖 antlr4） |
| `scripts/validate.py` | **新增（17 KB）** | 独立校验器：逐题复算 + 结构检查，产出 `validation_report.md/json` |
| `scripts/solve.py` | 改造 | 接入域求解器与 LLM 兜底；补全概念题/模板题路径（原 `solver: none` → 模板作答） |
| `scripts/render_latex.py` | 改造 | 中文自动切换 `ctexart`+xelatex；答案框 `answerbox`；非提权编译兜底；浮点痕迹清理 |
| `scripts/classify.py` / `parse_problems.py` | 改造 | 分类规则扩到线代/ODE/物理；解析适配 `$...$` 行内公式题干 |
| `domain_skills/` | 保留 | `calculus_limits.yaml` 原样保留，作为基线域不变 |
| `oracles/` `solver_templates/` `references/` | 保留 | 未改动，继承 starter 的知识资产 |
| `test_cases/` | 保留 | starter 自带 worksheet 3/4 及参考输出，用作**回归对比集** |

## 3. 国产大模型选型与理由

| 优先级 | 模型 | 端点（OpenAI 兼容） | 承担角色 | 选它的理由 |
|--------|------|----------------------|----------|-----------|
| 主 | **Qwen 3.6**（`qwen-plus`） | `dashscope.aliyuncs.com/compatible-mode/v1` | 概念题、证明题、物理文字题作答与参数抽取 | 中文题干理解稳、数学推理强、`compatible-mode` 迁移成本最低（同一份 OpenAI SDK 代码即可复用） |
| 备 | **Kimi 2.5**（`moonshot-v1-8k`） | `api.moonshot.cn/v1` | 长题干 / 多小问整体推理 | 长上下文、多步推理强，适合「一道题带 4 个小问」的整段推理 |
| 兼容 | DeepSeek（`deepseek-chat`） | `api.deepseek.com/v1` | 同角色可替换 | 作为第三后备，验证「协议层可插拔」而非硬编码单一厂商 |
| 兜底 | **离线确定性路径** | — | 无 Key 时全流程可跑 | 保证交付物**可复现**：不依赖任何外部密钥也能产出正确 PDF |

**关键设计判断：不让 LLM 承接它可以出错的确定性计算。**

- 行列式、逆矩阵、特征值、线性方程组、ODE、物理量代入，全部走 SymPy 符号推导 → 结果可被机器核验、零幻觉；
- LLM 只负责「语义层」：题干理解、非标准题干的参数抽取、概念题/证明题的文字作答、SymPy 兜底失败后的二次求解；
- **所有 LLM 输出都要过 `validate.py`**，才允许进入 PDF —— 避免"模型幻觉直接进终稿"。

## 4. 目标课程与题型覆盖

| 学科 | 覆盖题型 | 实现路径 |
|------|----------|----------|
| 线性代数 | 行列式、逆矩阵、特征值与特征向量、线性方程组（含高斯消元） | `domain_solvers.solve_linear_algebra`，SymPy 符号计算 + 回代校验 |
| 常微分方程 | 一阶线性、可分离变量、二阶常系数齐次/非齐次、初值问题 | `solve_ode_domain`：`dsolve` + 初值代入 + `sp.diff` 残差回代 |
| 大学物理 | 斜面动力学（支持力/摩擦力/加速度/末速度）、弹簧振子（周期/最大速度/机械能）、直流电路（并联等效电阻/干路电流）、动量与热一律 | `solve_physics`：公式库 + AST 数值求值 + 单位归一 |
| 微积分极限（基线域） | 极限、单侧极限、ε-δ、切线、概念题 | 沿用 starter 原路径，**不做破坏性改动**，仅补全概念题模板 |

## 5. 求解策略：三层路由 + 校验闭环

```
问题 → 分类（概念节点）
        ├─ 可形式化？ ── 是 → SymPy 确定性求解（域求解器）──┐
        │                                       否 ↓        │
        ├─ 命中求解器模板？ ── 是 → 模板作答 ────────────────┤
        │                             否 ↓                  │
        └─ 概念题/证明题/超纲 ── 有 Key → 国产 LLM 作答 ─────┤
                                    无 Key → 标记未解（诚实）│
                                                            ↓
                                              validate.py 独立复算
                                              （残差 / 代回 / 特例数值）
                                                            ↓
                                             通过 → 进入 PDF；不通过 → 回退重解
```

要点：

1. **确定性优先**：能机器核验的一律不用 LLM；
2. **语义兜底**：SymPy 无法形式化时才交国产模型，且只交"文字与推理"，答案仍需校验；
3. **诚实失败**：确实解不了的题（如本次的 AP3 几何推导题）标记未解并写明原因，**不编造答案**；
4. **校验独立**：`validate.py` 不复用求解器的函数路径，而是用另一种算法重算（按定义递归展开的余子式、ODE 求导回代残差、A·v=λv、KCL 支路求和、牛顿定律复算）。

## 6. 排版方案（对应「排版质量」独立 20 分）

- 文档类：`ctexart` + `a4paper` + 12pt，XeLaTeX 编译（中文必需），`Noto Sans CJK SC` 字体；
- 版式：题目蓝（`headblue`）+ 解答绿（`ansgreen`）双色体系；`\problem` 标题带横线分隔；`\answerbox` 绿框答案，答案与推导视觉分离；
- 首部信息栏：学生 / 课程 / 日期 / 题目总数 / 自动求解率 / 求解引擎分布 / 模型状态（可核验的元信息）；
- 数学排版：矩阵、分式、行列式统一走数学模式，行内公式用 `$...$`，多行公式按题型分行；
- 行长控制：用 `\emergencystretch=2em` 替代 `\sloppy`，避免长公式行出现夸张词间距或溢出；
- 数值观感：清理 SymPy 浮点痕迹（`1.0 e^{t^2}` → `e^{t^2}`）；
- **Windows 编译兜底**：MiKTeX 在非交互环境下拒绝提权安装缺包时，自动改用 `runas /trustlevel:0x20000` 降权令牌重编，保证无人值守也能出 PDF。

## 7. 质量保证与可复现性

| 手段 | 说明 |
|------|------|
| 逐题校验 | `python scripts/validate.py <run_dir>` → 10/10 题、28/28 检查项通过 |
| 同题回归 | 用 starter 自带的 worksheet 3/4 与随包参考输出逐题对比（见《验证报告》） |
| 一键复现 | `python scripts/pipeline.py <input> <out> --compile --out-prefix lenovo_C4C_output`，9.4 秒出 PDF |
| 零密钥可跑 | 无 API Key 时自动离线模式，仍有完整 PDF 产出 |
| 完整日志 | `1_ingested.json` / `2_parsed.json` / `3_solutions.json` / `validation_report.json` 全链路留痕 |

## 8. 与评分维度的对应（rubric.json，总分 100）

| 维度 | 分值 | 本方案的支撑 |
|------|------|--------------|
| 求解质量 solverQuality | 25 | 真实作业 10/10 解出；关键数值用 NumPy/SymPy **独立复算**（行列式 −85、逆矩阵、特征值 1/3、解 (5/6,5/6,3/2)、a=3.2026 m/s²、v=5.6591 m/s、T=0.3142 s、E=1 J、R_p=2 Ω、I=6 A）；边界题（AP3）如实标注未解 |
| 排版质量 typesetting | 20 | 双色版式 + 答案框 + 信息栏 + 中文排版 + 4 页无溢出（逐页视觉检查为"优"），并附 PNG 预览 |
| 产物完整性 artifactCompleteness | 15 | 7 件交付齐全 + `.skill` 可安装包 + 运行证据目录（JSON/TeX/校验报告）+ README 与安装使用说明 |
| AI 使用质量 aiUsage | 20 | 《AI日志》记录全过程多轮迭代：迁移、扩域、编译排障、排版优化、校验器修正；含真实 prompt 与失败复盘 |
| 复盘质量 reflectionQuality | 20 | 《AI日志》含 4 个具体失败案例（提权编译、Missing $、ODE 残差伪差、电路键缺失）与改进措施；《拿来说明》给出 Claude 版与国产版差异分析 |

## 9. 风险与边界

| 风险 | 现状与对策 |
|------|-----------|
| 无 API Key 时 LLM 路径不可实测 | 代码路径完整、可一键复现（`C4C_LLM_PROVIDER=qwen`），交付报告对"实测/待测"边界明确标注，不编造模型输出 |
| 题型超出域覆盖 | 分类器命中未知概念时标记未解并给出原因，绝不硬凑答案（本次 AP3 即为例） |
| MiKTeX 缺包 / 提权失败 | 已实现降权重编兜底，PDF 稳定产出 |
| 图片/扫描件输入 | starter 的 OCR 扩展点保留（`ingest.py`），本次未启用，已在《教学说明》中说明扩展方式 |
*（内容由AI生成，仅供参考）*
