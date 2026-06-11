---
name: research-init
description: Initialize a repository for human–AI collaborative ML/DL research. Use whenever the user wants to start a new experiment project, scaffold an experiment codebase, onboard or analyze an existing research repo, or says things like "/init", "set up the project", "帮我初始化", "分析一下这个仓库" — even if they do not mention scaffolding explicitly. Produces the directory skeleton, installs CLAUDE.md conventions, and writes the experiment-logic skeleton into docs/.
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

## Mode A — New project scaffolding

Scaffold only. Do not implement any experiment logic in this mode; implementation
happens later, against a recorded requirement.

1. **Understand the requirement first.** Interview the researcher until you can
   state, in a few sentences each: the research question; the method (at the level
   of experiment logic, e.g. "PPO on top of a frozen SFT checkpoint, reward from
   model X"); what a single run produces; what will be compared across runs
   (the hyperparameters that matter). Ask focused questions; do not proceed on
   guesses.
2. **Record the requirement** in `docs/REQUIREMENTS.md`, in the researcher's own
   words plus the clarifications they confirmed. This file is the baseline that
   the research-audit skill will later check scripts against, so it must be
   precise about anything that would change the experiment's meaning.
3. **Draft the pseudocode** in `docs/PSEUDOCODE.md` using the template in
   `references/pseudocode-spec.md` (golden sample: `references/pseudocode-example-gcg.md`).
   This is the **top-level abstraction the researcher reviews** before any code is
   written, and the primary claim source for research-audit. Fill §0–§2 at minimum;
   keep §1a (searchable hyperparameters) and §1b (fixed experiment switches)
   strictly separated — that boundary is what the later HPO stage is allowed to
   touch. Leave §4 function bodies as signatures + intent; do not implement.
   Have the researcher approve the pseudocode before scaffolding code.
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

The goal is a map the researcher can audit, not a refactor. Do not restructure the
legacy code; only add `docs/`, `results/` conventions, and `CLAUDE.md`.

1. **Find the entrypoints** (training/eval scripts, main configs) and trace the
   actual execution path. Read function bodies; do not infer behavior from names.
2. **Write `docs/REPO_SKELETON.md`.** Its backbone must be **experiment logic**,
   not call-graph logic. Experiment logic is the researcher's mental model of the
   procedure; code logic is how the repo happens to implement it. Structure: one
   section per experiment-logic step, and under each step, where and how the code
   realizes it, with `file:line` references.

**Example skeleton fragment (LLM training):**

```markdown
## Experiment logic: one training step
1. Forward pass — model produces logits for the batch
   - entry: train.py:212 `model(batch)`; attention impl swapped in
     models/patch.py:40 (monkey-patches HF attention at import time — note:
     this happens implicitly, flag for researcher)
2. Loss computation — token-level CE, ignore_index=-100
   - losses/ce.py:18; label shifting done in data collator, not in loss
     (collate.py:77)
3. Backward — grad accumulation x8, clipping at 1.0
   - train.py:230-241; clipping BEFORE accumulation boundary — unusual, flag
4. Update — AdamW, cosine schedule
   - train.py:245; scheduler stepped per micro-batch, not per optimizer step
     — likely bug or intentional, flag
```

   The "flag" items matter most: anywhere the code's behavior would surprise a
   researcher reading only the top level, say so explicitly.
3. **Inventory the configs:** where hyperparameters live, which are actually read,
   which are silently defaulted. Put this in the same doc.
4. **Add the run conventions:** create `results/` and `docs/MISTAKES.md` if absent,
   install `CLAUDE.md`, and note in it any repo-specific deviations from the
   standard layout (per the template's own escape clause).
5. **Hand back a summary:** the skeleton's table of contents, the flagged
   surprises, and open questions for the researcher. Do not start changing code.

## Quality bar

- Every claim in `REPO_SKELETON.md` carries a `file:line` reference.
- A researcher who reads only `docs/` should be able to predict what a run will do.
- Mode A: §1a and §1b in `PSEUDOCODE.md` are cleanly separated; §0–§2 are complete
  enough to review without reading code.
- All baseline docs (`REQUIREMENTS.md`, `PSEUDOCODE.md`, `REPO_SKELETON.md`) are
  written for iteration: short sections, stable headings, so later diffs stay readable.
