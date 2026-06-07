---
name: init
description: 初始化标准化研究项目脚手架（core/、experiments/、results/、utils/、docs/，以及 README、.gitignore 与占位文档）。当研究者想开始一个新实验项目、为 gen-script 流水线创建工作目录，或把已有项目整理成统一布局时使用。
---

# Init — 项目脚手架初始化

在当前目录下创建一个符合 `gen-script` 流水线约定的研究项目布局。Agent 默认已身处项目目录，只需调用 `scripts/init_project.py` 并校验产物目录树符合下方契约。

## 何时使用

- 研究者有了新实验想法，需要一个干净、可审计的工作区。
- 需要把已有项目规整成标准目录布局。
- 作为 `gen-script` 流水线的第一步，在 `research`、`discuss`、`build` 之前执行。

## 如何运行

Agent 默认已身处项目目录内，直接在项目根调用即可：

```bash
python <skill_dir>/scripts/init_project.py
```

可选参数：
- `--root <target_dir>`：指定目标目录（默认当前工作目录）。
- `--no-git`：跳过自动 `git init`。

脚本仅使用 Python 标准库（`pathlib`、`argparse`、`subprocess`），在 Windows 10+、macOS 12+、Ubuntu 20.04+ 上无需修改即可运行。脚本是幂等的：重复运行不会覆盖已有文件，只会补建缺失部分。

## 期望的目录布局

脚本直接在目标目录下创建以下结构：

```
<target_dir>/
├── core/                          # 函数 / 类的具体实现
├── experiments/                   # 顶层脚本，组合调用 core/
├── results/                       # YYYY-MM-DD/{smoke,main}/<run_id>/
├── utils/                         # logger、seed 等纯工具
├── docs/                          # 研究者主导维护；Agent 每次运行前必须全量读取
│   ├── pseudocode.md              # （占位）后续填入伪代码 spec
│   ├── research-notes.md          # （占位）研究者的调研笔记
│   └── bug-fixes.md               # （占位）对应 spec §5 的 Gotcha 账本
├── README.md                      # 说明目录约定
└── .gitignore                     # results/、__pycache__/、.venv/、.DS_Store 等
```

当 `git` 可执行且未传 `--no-git` 时，会在项目根创建 `.git/`。脚本**不会**执行 `git commit`——首次提交由研究者自行决定。

## 校验清单

脚本运行完成后，Agent 必须逐项确认以下结果，全部通过才能报告成功：

1. 五个子目录（`core`、`experiments`、`results`、`utils`、`docs`）均存在且可读。
2. `docs/pseudocode.md`、`docs/research-notes.md`、`docs/bug-fixes.md` 均存在。
3. `README.md` 和 `.gitignore` 存在于项目根。
4. `.git/` 存在于项目根，除非传了 `--no-git` 或 `git` 不可用（后者必须向研究者说明原因）。

任一项失败都要报告错误，**不得**继续进入流水线下一阶段。

## 边界情况

| 场景 | 行为 |
|---|---|
| 目标目录已存在且非空 | 仅补建缺失部分，不触碰已有文件 |
| 目标目录无写权限 | 脚本报错退出 |
| `git` 不在 PATH 中 | 跳过 git 步骤，Agent 需向研究者说明 |
| 目标目录已是 git 仓库 | 跳过 `git init`，不修改任何状态 |

## 本阶段不做的事

- 不写伪代码（那是研究者的工作，可由后续阶段辅助）。
- 不生成 `core/` 或 `experiments/` 内的任何代码。
- 不执行 `git commit`。
- 不触碰新项目目录之外的任何文件。
- 不依赖其他任何 sub-skill 的产物。

## 参考资料（仅供参考，不自动加载）

两份文件位于 `<skill_dir>/references/`，是仓库根对应文件的**独立拷贝**——不用软链，保证跨平台兼容性。如果源模板演进，需由研究者手动同步。

| 文件 | 作用 |
|---|---|
| `pseudocode-spec-template.md` | 标准伪代码 spec 模板，研究者据此填写 `docs/pseudocode.md` |
| `pseudocode-gcg-example.md` | GCG 攻击的完整伪代码范例，供后续阶段作为黄金样例 |
