---
AIGC:
    Label: "1"
    ContentProducer: 001191440300708461136T1XGW3
    ProduceID: 60ea5d0126731e33de81577b27285889_986f3372bff911f1884b525400cd780f
    ReservedCode1: WqJ6pHrUEZQFUIZfIguV3uRl5d/7Wh9YK4yml6M0tAZQQmxVZPFiwIktACzpDrFHJlax+W4ZsM+5NenxIefDTdWM0Aa0+O5Fb61M5RAG3TjGIHryO271RfdV5N6c2Y6wXeyJn7VeWlxE18LfjQXtRvB7rE/ICJ/q7UX0QjAjLSX5+Db4D40tcJoLClo=
    ContentPropagator: 001191440300708461136T1XGW3
    PropagateID: 60ea5d0126731e33de81577b27285889_986f3372bff911f1884b525400cd780f
    ReservedCode2: WqJ6pHrUEZQFUIZfIguV3uRl5d/7Wh9YK4yml6M0tAZQQmxVZPFiwIktACzpDrFHJlax+W4ZsM+5NenxIefDTdWM0Aa0+O5Fb61M5RAG3TjGIHryO271RfdV5N6c2Y6wXeyJn7VeWlxE18LfjQXtRvB7rE/ICJ/q7UX0QjAjLSX5+Db4D40tcJoLClo=
---

# 教学说明：C4C 作业自动求解与排版技能

- 技能包：`lenovo_C4C_homework-solver.skill`（v2.0）
- 适用对象：C4C 课程学员，以及任何需要"用 AI + 国产大模型做数学求解"的自学者

---

## 一、这个技能能做什么

输入一份 `.md` 格式的数学作业文件，输出两个东西：

1. **带完整 LaTeX 公式的 Markdown 解答文件**：每个题号对应一个子文件，内含题目重述、分步求解过程、最终答案。
2. **编译好的 PDF 排版文件**：使用 MiKTeX（XeLaTeX）将 Markdown 渲染为学术风格 PDF，可直接提交或分享。

---

## 二、支持什么学科

当前技能覆盖 **3 个学科**：

| 学科 | 支持题型 | 示例 |
|------|----------|------|
| 线性代数 | 行列式计算、矩阵求逆、特征值/特征向量、线性方程组 | 4×4 行列式、2×2 逆矩阵、λ 求解、Ax=b |
| 常微分方程 | 一阶线性 ODE、一阶初值问题、二阶常系数齐次方程 | y'+2y=eˣ、y'=2ty(y(0)=1)、y''-3y'+2y=0 |
| 大学物理 | 运动学（匀加速直线运动）、动力学（牛顿第二定律）、电学（欧姆定律/基尔霍夫定律）、能量与振动 | 斜面滑块、弹簧振子、并联电路 |

---

## 三、快速上手

### 前置条件

| 依赖 | 说明 |
|------|------|
| Python 3.9+ | 用于运行求解器 |
| MiKTeX | 用于编译 PDF（XeLaTeX 引擎） |
| pip 包 | 安装 `sympy` |

```bash
pip install sympy
```

### 使用方法

```bash
# 解数学作业并生成 LaTeX
python scripts/pipeline.py 作业文件.md 输出目录

# 编译为 PDF
python scripts/pipeline.py 作业文件.md 输出目录 --compile

# 一步到位（求解 + 排版）
python scripts/pipeline.py 作业文件.md 输出目录 --compile --out-prefix 我的答案
```

### 目录结构

```
技能包/
├── SKILL.md              # 技能描述（给 AI 看）
├── README.md             # 使用说明（给人看）
└── scripts/
    ├── pipeline.py       # 主流程：提取 LaTeX → 分类 → 求解 → 组装
    ├── domain_solvers.py # 学科求解器（线性代数/ODE/物理）
    ├── llm_engine.py     # LLM 引擎（Qwen / Kimi / DeepSeek / 离线模板）
    ├── validate.py       # 校验器（L1 结构检查）
    ├── latex_utils.py    # LaTeX 工具（提取、公式渲染、数学模式检查）
    └── classify.py       # 题目类型分类器（线性代数/ODE/物理）
```

---

## 四、解题流程

每个数学题的处理链路：

```
1. 提取 LaTeX 公式    ← 用正则表达式从 Markdown 提取 $$ ... $$
2. 分类学科类型       ← 判断是线性代数、ODE 还是物理
3. 选择求解器         ← 线性代数 → SymPy；ODE → SymPy 符号求解；物理 → 符号 + 数值混合
4. 生成解答 LaTeX     ← 分步过程 + 最终答案
5. 组装 + 编译 PDF    ← 所有题汇总为完整文档，编译为学术风格 PDF
```

---

## 五、注意事项

1. **AI 不是万能的**：技能依赖 SymPy 符号求解和 LLM 推理，遇到无法解析的复杂表达式时，会如实标记"无法求解"，不会瞎编答案。
2. **LaTeX 编译环境**：需要本地安装 MiKTeX，并选择 XeLaTeX 引擎（支持中文）。
3. **学科覆盖**：当前仅支持线性代数、常微分方程、大学物理三个学科。如需扩展，可参考 `domain_solvers.py` 中的 `register_solver` 注册新求解器。
4. **结果可复现**：所有数学求解使用 SymPy（确定性引擎），非 LLM 生成的部分完全可复现。

---

## 六、扩展建议

- 如需支持更多学科（如概率论、复变函数、数值分析），可在 `domain_solvers.py` 中新增 `register_solver('概率论', 学科求解函数)`。
- 如需调整排版风格，可修改 `scripts/pipeline.py` 中 `build_latex_body` 函数的 LaTeX 模板。
- 如需批量处理，可将多份作业合并为一个目录，遍历 `pipeline.py`。

---

*本技能包基于 C4C 课程 starter kit 改造，将解题引擎从 Claude 迁移到国产大模型（Qwen3.6/Kimi2.5），并扩展学科范围到线性代数、常微分方程和大学物理。*
*（内容由AI生成，仅供参考）*
