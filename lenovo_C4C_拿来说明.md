---
AIGC:
    Label: "1"
    ContentProducer: 001191440300708461136T1XGW3
    ProduceID: 60ea5d0126731e33de81577b27285889_9bde88d5bff911f1887c525400de85a5
    ReservedCode1: D2d9kF0qAxhonSoc8B21HZHbRZ8yyBTKxLDMmarANlIebwiYDd9xxo9O5n25fEHlN57XYgfdxtAFMUTIpxZXZ0csQQbKdHKOd3gjQaggyS/iwVC8WSmFvSROmJKlBV7mBrXrZ1KLZqW4t4jeDhyjYLZxBRQGXCI2WYScwVUTY2nV2D9RBSGsaWD5C0c=
    ContentPropagator: 001191440300708461136T1XGW3
    PropagateID: 60ea5d0126731e33de81577b27285889_9bde88d5bff911f1887c525400de85a5
    ReservedCode2: D2d9kF0qAxhonSoc8B21HZHbRZ8yyBTKxLDMmarANlIebwiYDd9xxo9O5n25fEHlN57XYgfdxtAFMUTIpxZXZ0csQQbKdHKOd3gjQaggyS/iwVC8WSmFvSROmJKlBV7mBrXrZ1KLZqW4t4jeDhyjYLZxBRQGXCI2WYScwVUTY2nV2D9RBSGsaWD5C0c=
---

# 使用说明：C4C 作业自动求解与排版技能

---

## 快速开始

将本技能包 `lenovo_C4C_homework-solver.skill`（ZIP 格式）解压后，按以下步骤使用：

### 1. 准备环境

- 安装 Python 3.9 或更高版本
- 安装 MiKTeX（用于 PDF 编译）
- 安装 SymPy：`pip install sympy`

### 2. 解一份作业

```bash
python scripts/pipeline.py 你的作业.md 输出目录
```

例如：

```bash
python scripts/pipeline.py lenovo_C4C_作业原件.md C:\Users\lenovo\Desktop\C4C_作业自动求解与排版_交付物\lenovo_C4C_交付物\作业解答
```

执行后，`输出目录` 中将生成：
- 每个题目的 Markdown 解答文件（含完整 LaTeX 公式）
- `lenovo_C4C_output.pdf`：编译好的学术风格 PDF

### 3. 编译 PDF

```bash
python scripts/pipeline.py 你的作业.md 输出目录 --compile --out-prefix 我的答案
```

编译完成后，`我的答案.pdf` 即为最终排版交付物。

---

## 技能包文件结构

```
lenovo_C4C_homework-solver/
├── SKILL.md          # 技能描述（供 AI Agent 读取）
├── README.md         # 使用说明（给人看）
└── scripts/
    ├── pipeline.py          # 主流程入口
    ├── domain_solvers.py    # 学科求解器（线性代数/ODE/物理）
    ├── llm_engine.py        # LLM 引擎（Qwen/Kimi/DeepSeek）
    ├── validate.py          # 结构校验器
    ├── latex_utils.py       # LaTeX 工具
    └── classify.py          # 题目类型分类器
```

---

## 常见问题

**Q: 技能包不能运行怎么办？**
A: 确认 Python 环境（`python --version` 应 >= 3.9）和 MiKTeX 已安装。MiKTeX 安装后可用 `miktex` 命令验证。

**Q: 支持的学科有哪些？**
A: 当前支持：线性代数（行列式/逆矩阵/特征值/方程组）、常微分方程（一阶线性/初值问题/二阶常系数）、大学物理（运动学/动力学/电学）。如需扩展，可参考 `domain_solvers.py` 中的注册机制。

**Q: PDF 编译失败？**
A: 确认 MiKTeX 已正确安装，且 `MiKTeX/bin/x64` 已加入系统 PATH。编译引擎为 XeLaTeX。

**Q: 某个题解不出？**
A: 技能会如实标记"无法求解"，不会瞎编。这比硬凑答案更有教学价值。

---

## 交付物清单

本技能挑战的完整交付物（8 件）：

| # | 文件 | 说明 |
|---|------|------|
| 1 | `lenovo_C4C_方案设计.md` | 架构设计、选型、实施计划 |
| 2 | `lenovo_C4C_homework-solver.skill` | 技能包 ZIP（v2.0） |
| 3 | `lenovo_C4C_作业原件.md` | 真实作业（10 题） |
| 4 | `lenovo_C4C_output.pdf` | 排版交付物（4 页） |
| 5 | `lenovo_C4C_验证报告.md` | 正确性验证（100%） |
| 6 | `lenovo_C4C_教学说明.md` | 教学文档（使用说明） |
| 7 | `lenovo_C4C_AI日志.md` | AI 使用日志（9 项迭代） |
| 8 | `lenovo_C4C_拿来说明.md` | 本文件（使用说明） |

---

*技能包基于 C4C 课程 starter kit 改造，将解题引擎从 Claude 迁移到国产大模型（Qwen3.6/Kimi2.5），并扩展学科范围到线性代数、常微分方程和大学物理。*
*（内容由AI生成，仅供参考）*
