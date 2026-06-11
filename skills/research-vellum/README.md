# research-copilot-skills（v1）

面向 ML/DL 科研的人机协作 skill 包。设计哲学：第一版只装"心法"——身份定位、诚实不变量、流程纪律、assert 模式，全部用纯 Python/stdlib 表达，不引入任何社区依赖，便于随代码仓库开源传播。具体工具选型（wandb 还是本地日志、Hydra 还是裸 YAML）留给使用这套 skill 的研究员自行决定。

## 结构与需求映射

```text
research-copilot-skills/
|-- README.md
|-- research-init/                  # 需求 1：/research-init 初始化
|   |-- SKILL.md                    #   Mode A 新项目脚手架+伪代码 / Mode B 既有仓库分析
|   |-- assets/
|   |   `-- CLAUDE.md.template      # 需求 2："每次都注入"的心法层（见下）
|   `-- references/
|       |-- pseudocode-spec.md      #   伪代码 spec 模板（docs/PSEUDOCODE.md 的结构）
|       `-- pseudocode-example-gcg.md  #   填好的黄金样例（GCG 主流程）
|-- research-audit/                 # 需求 3：proposal/伪代码 与脚本一致性审查
|   |-- SKILL.md                    #   claim 分解 -> 逐条取证 -> 五类裁决 -> 报告模板
|   `-- references/
|       `-- audit-checklist.md      #   高发陷阱清单（EXTRAS 扫描前必读）
`-- research-develop/               # 需求 4：二次开发
    |-- SKILL.md                    #   确认意图 -> 先写 assert -> 最小 diff -> smoke -> 报告
    `-- references/
        `-- assert-patterns.md      #   12 类 assert 模式目录 + 反模式
```

## 伪代码是研究员审查的抓手

`docs/PSEUDOCODE.md` 用自然语言+形式化描述每一步在做什么（实验逻辑，非代码调用），是
研究员批准后 AI 才据此实现 `core/`、`exp/` 的**最顶层抽象**，也是 research-audit 拆 claim
的主要来源。模板里 §1a（可搜索超参）与 §1b（固定实验开关）刻意分开：这条边界是第二阶段
超参搜索 Agent 的硬约束——只能动 §1a，碰 §1b 即违规。

## 审查的独立性

research-audit 假定在**独立上下文**里运行——最好是与写代码不同的 Agent 或平台（例如
Claude Code 写、Codex 审）。这把"AI 自己审自己"的可信度问题交给平台切换解决，skill 本身
只负责对抗式取证的方法。

## 一个重要的设计决定：心法为什么不是 skill

Claude Code 的 skill 是**按需触发**的——模型根据 description 决定是否读取，没有机制保证每轮注入。真正每轮都在上下文里的是仓库根目录的 `CLAUDE.md`。因此"身份定位 + 语言风格 + 项目框架 + 诚实不变量"做成了 `CLAUDE.md.template`，由 research-init 在初始化时装进目标仓库。这比做成 skill 更可靠：约束不依赖触发，且随仓库一起开源、一起被 fork。

三个 skill 则各管一个有明确入口的工作流（初始化 / 审查 / 改动），这正是 skill 机制擅长的形态。

## 安装

- 项目级：拷贝三个 skill 目录到目标仓库的 `.claude/skills/`。
- 个人级：拷贝到 `~/.claude/skills/`。
- `CLAUDE.md.template` 不需要手动安装——运行 research-init 时会装入仓库。

## v1 刻意不做的事（即扩展路线）

按上一轮讨论的信任层级（机械验证 > AI 审计 > AI 自觉），v1 的定位是把"AI 自觉"这一层做扎实，并通过 assert 把一部分检查推到机械层。以下各项是清晰的 v2+ 挂载点，目录结构已为此预留：

1. **hooks 强制执行**：目前"完成改动必须跑 smoke + assert"靠 CLAUDE.md 约定。v2 可加 PostToolUse / Stop hook 强制运行，失败自动打回。挂载点：各 SKILL.md 的 smoke 步骤。
2. **审查的上下文隔离**：v1 靠人工切换 Agent/平台（Claude Code 写、Codex 审）保证独立性。v2 可做成固定 spawn 子代理的命令，省掉手动切换。
3. **冻结层**：eval 脚本、测试集只读保护（文件权限 / CI 拒绝 diff）。这超出 skill 范畴，属仓库纪律，可写进 CLAUDE.md.template 的扩展段。
4. **`verify/` 产物检查脚本**：audit-checklist 中"UNVERIFIABLE-STATICALLY"项沉淀多了之后，自然长成一个 `verify/` 目录——从真实失败中归纳，而不是预先设计。
5. **触发词调优**：三个 description 已按"宁可多触发"的原则写得偏 pushy（含中文触发词）。实际使用一两周后，用 skill-creator 的 description 优化流程跑一遍真实 query。

## 命名备注

你原稿中的 `archievements/` 是 `achievements/` 的笔误，模板与 skill 中已统一为后者；如果你有意为之，全局替换回去即可。
