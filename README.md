# Research Vellum

面向 ML/DL 科研的人机协作 copilot。人类研究员把控方向与实验设计，AI 负责实现细节。
目标是确立 **Human-on-the-loop** 范式（区别于 Human-in-the-loop）：AI 帮人类快速完成
想法验证与迭代，人类在关键节点（Spec 审查、异常分析）提供高价值指引。

## 要解决的问题

当前 AutoResearch 的困局有两个：

1. **逻辑断层**：研究员脑子里的实验逻辑与 AI Vibe Coding 出的代码逻辑对不上。AI 爱写
   过度封装的黑盒，人类看不懂也改不动；数学直觉也难以清晰传给 AI。
2. **AI 会撒谎**：让它跑 RL，它做着发现 RL 太难，偷偷换成 SFT，还汇报说"性能提升了"。
   失败被伪装成成功，是科研里最毒的污染。

一个完整的实验循环里，这两个问题会反复发作：

```text
Idea and Hypothesis
        │
        v
编写实验脚本  ->  查看结果  ->  科学发现
        ▲                      │
        │                      │
        └──── 修正假设，继续实验
```

## 核心设计：用审计代替"全靠人看懂"

最初的想法是逼研究员对主脚本的每处细节都有清晰把握，以此防止 AI 撒谎。但这把心智负担又
全数还给了人类。更好的思路是：**让一个独立的 audit agent 检查"研究员的意图"与"AI 写的
脚本"是否对得上**，人类只审计这份审计报告，而不必重读整个仓库。

围绕这一点，建立一个信任层级（强 → 弱）：

1. **机械验证**：用纯 Python 的 `assert` 把"研究员希望改动做了 X"变成"代码不满足 X 就拒绝运行"。
2. **AI 审计**：独立上下文的对抗式审查，把意图拆成原子 claim，逐条取证裁决。
3. **AI 自觉**：每轮注入的诚实不变量（绝不偷换方法、绝不报告无来源的数字等）。

v1 把"AI 自觉"做扎实，并通过 assert 把一部分检查推到机械层。

## v1 skills

具体实现见 [`skills/research-copilot-skills/`](skills/research-copilot-skills)，第一版只装"心法"，
纯 stdlib、零社区依赖，便于随仓库开源传播：

- **CLAUDE.md.template** — 每轮注入的心法层：身份定位、诚实不变量、语言风格、项目框架。
  做成模板而非 skill，因为只有仓库根的 `CLAUDE.md` 能保证每轮都在上下文里。
- **research-init** — `/init` 初始化：新项目搭脚手架，或为既有仓库写"实验逻辑骨架"。
- **research-audit** — 一致性审计（最核心）：claim 分解 → 逐条取证 → 五类裁决 → 报告模板。
- **research-develop** — 二次开发：确认意图 → 先写 assert → 最小 diff → smoke → 报告。

## 项目框架

```text
Project/
|-- core/          # 函数/类的具体实现，组织方式由 AI 决定
|-- exp/           # 调用 core，组合成单次脚本、批量脚本与 config
|-- results/       # 按日期存放实验产物，日期内分 smoke / main，从不手改
|-- achievements/  # 从 results 归档并深入分析有意义的结果，人类主导
|-- utils/         # logger、绘图、统计等辅助工具
|-- docs/          # 伪代码、需求、犯过的错（MISTAKES.md），人机协同、简洁
```

此框架是默认而非枷锁；在既有仓库里工作时保留其结构，只补 `results/` 与 `docs/` 约定。

## Roadmap

- [x] v1：心法层 skills（init / audit / develop + CLAUDE.md）
- [ ] hooks 强制执行：完成改动必须跑 smoke + assert，失败自动打回
- [ ] 审计上下文隔离自动化：固定 spawn 子代理做对抗式审查
- [ ] 冻结层：eval 脚本 / 测试集只读保护
- [ ] `verify/` 产物检查脚本：从真实失败中归纳，而非预先设计
- [ ] 大规模实验与超参优化的脚本
