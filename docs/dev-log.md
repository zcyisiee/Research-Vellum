# gen-script 开发问题记录

记录 `gen-script/` 各 sub-skill 开发过程中遇到的真实问题。不分析、不包装。

## 1. 目录命名与编号反复变动

初始规划用 `task1-script-gen/s01..s06`、`task2-hpo-run/s01..s04`。开发中改为 `gen-script/`、`run-hpo/`，且第一阶段由 `task1-script-gen/s01-research` 调整为 `gen-script/init`，research 后移到 `s02`。

后果：早期脚本中遗留 `task1_script_gen`、`task2_hpo_run` 两个下划线命名的空目录，需手动 `rmdir` 清理。

教训：在骨架未稳定前，不要给目录加编号。

## 2. "init" 的语义被误解

Agent 两次误解 init 的范围：
- 第一次把 init 理解为"初始化伪代码 spec 本身"，规划成了 `pseudocode-init`。
- 第二次把 init 理解为"从 spec 生成项目骨架"。

实际意图：init 只是创建工作目录脚手架（`core/`、`experiments/`、`results/`、`utils/`、`docs/`），不碰伪代码内容。

根因：没有在对话一开始就明确"init 的产出是什么"。

## 3. 写入空文件时触发工具校验

`Write` 工具要求对已存在的文件先 `Read`。`SKILL.md` 虽是空文件但已经存在，直接 `Write` 报 `File has not been read yet`。

解决：对空文件先调一次 `Read`，再 `Write`。

## 4. WebFetch 拉取 GitHub 内容超时

抓取 `anthropics/skills` 仓库时 `WebFetch` 被取消。改用 `curl` + GitHub API（`api.github.com/repos/.../contents/`）拿到目录与文件。

备用路径：`raw.githubusercontent.com` 直取单文件。

## 5. 软链跨平台风险

曾考虑让 `references/` 内的模板以软链指向仓库根原件。Windows 下软链需要特殊权限，最终改为直接拷贝，并明确"手动同步"的维护责任。

## 6. 项目语言默认值未对齐

Agent 默认用英文写 `SKILL.md`。用户明确指出项目描述应使用简洁中文。

修正：所有面向研究者的描述性文本（frontmatter `description`、章节正文）一律用中文；代码标识符、命令行、文件路径保持原形。

## 7. YAML frontmatter 格式需要明确范例

Agent 第一次写 frontmatter 时没把握 `name` 用 kebab-case 还是 plain，`description` 应该多长。参考 `anthropics/skills/template/SKILL.md` 后才确定：

```
---
name: init
description: 一句话讲清"做什么"和"何时用"。
---
```

`name` 用目录名（kebab-case 或 plain 均可），`description` 一段话覆盖用途与触发时机。

## 8. init 脚本错误地嵌套了一层项目目录

初版 `init_project.py` 把 `project_name` 当必传参数，在 `<root>/<project_name>/` 下创建脚手架。实际工作流是：研究者或 Agent 先 `mkdir` + `cd` 进项目目录，然后才运行 init——也就是说 cwd 已经是项目根，再套一层 `project_name` 会生成冗余目录。

修正：移除 `project_name` 必传参数，脚本直接在 `--root`（默认 cwd）下创建 `core/`、`experiments/` 等子目录；README 标题从目标目录的 basename 自动推断。CLI 签名从 `python init_project.py <name> --root <dir>` 简化为 `python init_project.py [--root <dir>]`。

教训：设计脚本前先确认调用者"此时站在哪里"。
