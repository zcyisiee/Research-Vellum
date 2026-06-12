---
name: research-init
description: Initialize a repository for human–AI collaborative ML/DL research. Use whenever the user wants to start a new experiment project, scaffold an experiment codebase, onboard or analyze an existing research repo, or says things like "/init", "set up the project", "帮我初始化", "分析一下这个仓库" — even if they do not mention scaffolding explicitly. Produces the directory skeleton, installs CLAUDE.md conventions, and writes pseudocode specs into docs/.
---

# Research Init

Initialize a repository so that all later collaboration (writing, auditing,
modifying experiment scripts) happens inside one shared framework. Two modes;
detect which one applies before doing anything.

**Mode selection:**
- An existing predecessor repo to build on → Mode B: analyze it so the researcher
  can quickly understand it and iterate.
- No repo to start from, the researcher states a goal (typical trigger:
  `/research-init 我希望进行……的实验，实验细节如下：…`) → Mode A: scaffold from that
  goal.

If the working directory already contains substantive source code, assume Mode B;
otherwise Mode A. If unclear, ask one question.

In both modes, finish by installing `assets/CLAUDE.md.template` from this skill as
the repository's `CLAUDE.md` (merge carefully if one already exists — never
overwrite researcher-written content).

## Pseudocode files

All pseudocode lives in `docs/` with the prefix `PSEUDOCODE-`:

```text
docs/PSEUDOCODE-train.md
docs/PSEUDOCODE-eval.md
docs/PSEUDOCODE-gcg-attack.md
```

Each file starts with YAML frontmatter:

```yaml
---
title: [实验脚本的一句话描述]
entry_scripts:
  - exp/train.py
  - exp/config.yaml
created: YYYY-MM-DD
modified: YYYY-MM-DD
status: draft | reviewed | verified
---
```

- `status: draft` — 刚生成，研究员尚未审批
- `status: reviewed` — 研究员已审批，可作为实现/审计基线
- `status: verified` — 已通过 research-audit 验证与代码一致

Pseudocode body follows the template in `references/pseudocode-spec.md`
(golden sample: `references/pseudocode-example-gcg.md`).

## Mode A — New project scaffolding

Scaffold only. Do not implement any experiment logic in this mode; implementation
happens later, against a recorded requirement.

1. **Understand the requirement first.** Interview the researcher until you can
   state, in a few sentences each: the research question; the method (at the level
   of experiment logic, e.g. "PPO on top of a frozen SFT checkpoint, reward from
   model X"); what a single run produces; what will be compared across runs
   (the hyperparameters that matter). Ask focused questions; do not proceed on
   guesses.
2. **Record the requirement** in `docs/REQUIREMENTS-{name}.md`, in the researcher's
   own words plus the clarifications they confirmed. This file is the baseline that
   the research-audit skill will later check scripts against, so it must be
   precise about anything that would change the experiment's meaning.
3. **Draft the pseudocode** in `docs/PSEUDOCODE-{name}.md` with YAML frontmatter
   (status: draft). Fill §0–§2 at minimum; keep §1a (searchable hyperparameters)
   and §1b (fixed experiment switches) strictly separated — that boundary is what
   the later HPO stage is allowed to touch. Pseudocode describes experiment logic
   only — do not write function signatures; AI decides `core/` function
   decomposition freely during implementation. Have the researcher approve the
   pseudocode before scaffolding code.
4. **Create the skeleton:**

```text
core/  exp/  results/  achievements/  utils/  docs/
```

   Add minimal placeholder files only where useful: `exp/config.yaml` with the
   §1a/§1b hyperparameters from the pseudocode (values may be TODO), `utils/logger.py`
   stub if the researcher wants one, `docs/MISTAKES.md` with just a header. Nothing
   else — do not implement experiment logic in `core/` or `exp/`.
5. **Install `CLAUDE.md`** from the template, then show the researcher a short
   summary: the tree, the recorded requirement, the pseudocode for their review,
   and the next step (implementing `core/` + `exp/` against the approved pseudocode).

## Mode B — Existing codebase

The goal is pseudocode the researcher can audit, not a refactor. Do not restructure
the legacy code; only add `docs/`, `results/` conventions, and `CLAUDE.md`.

**Two sub-modes** depending on whether the researcher has existing documentation:

### B1 — Researcher has documentation (proposal, README with experiment description, etc.)

1. **Trace the execution path.** Find the entrypoints (training/eval scripts, main
   configs) and read function bodies to understand what the code actually does.
   Do not infer behavior from names.
2. **Generate pseudocode.** For each experiment script/phase, create a
   `docs/PSEUDOCODE-{name}.md` file with YAML frontmatter (entry_scripts, dates,
   status: draft). The pseudocode describes what the code **actually does**,
   structured per the spec template (§0–§5).
3. **Generate diff report.** Create `docs/DIFF-{name}.md` comparing:
   - What the researcher's documentation says the experiment should do
   - What the code actually does (as captured in the pseudocode)
   Flag any discrepancies: silent extras, missing components, behavioral surprises.
   This is the Mode B equivalent of research-audit's claim decomposition — but
   done at init time to surface issues early.
4. **Install conventions:** create `results/` and `docs/MISTAKES.md` if absent,
   install `CLAUDE.md`, note any repo-specific layout deviations.
5. **Hand back a summary:** the list of generated pseudocode files, flagged
   discrepancies, and open questions. Do not start changing code.

### B2 — No documentation (researcher wants to understand the codebase)

1. **Trace the execution path.** Same as B1 step 1.
2. **Generate pseudocode.** Same as B1 step 2, but the pseudocode serves as the
   researcher's first map of what the codebase does. Be especially clear in §0
   (purpose) and §2 (control flow) — these sections must let the researcher
   understand the experiment logic without reading the source code.
3. **Install conventions.** Same as B1 step 4.
4. **Hand back a summary:** the list of generated pseudocode files, a brief
   overview of the experiment logic per file, and open questions.

## Quality bar

- Mode A: §1a and §1b in pseudocode are cleanly separated; §0–§2 are complete
  enough to review without reading code.
- Mode B: every pseudocode file accurately reflects what the code does; the
  researcher reading only `docs/PSEUDOCODE-*.md` can predict what a run will do.
- All pseudocode files have valid YAML frontmatter.
- All baseline docs are written for iteration: short sections, stable headings,
  so later diffs stay readable.
