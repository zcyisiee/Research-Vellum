---
name: research-develop
description: Modify existing experiment scripts for a new experiment, a follow-up change, or secondary development, while guarding the researcher's intent with plain-Python asserts. Use whenever the user asks to change, extend, or adapt experiment code — "改一下脚本", "加一个消融", "在现有基础上做新实验", "switch the dataset", "add a loss term" — even for small edits, as long as the change touches experimental behavior. Not for pure refactors with zero behavioral change.
---

# Research Develop

Every behavioral change to experiment code follows: **confirm intent → assert the
intent → minimal diff → smoke run → report**. The asserts are the point: they turn
"the researcher hopes the change does X" into "the code refuses to run unless X",
using nothing but plain Python/PyTorch — no new dependencies, safe to open-source.

## Workflow

### 1. Restate intent

Restate the requested change in at most 3 sentences: what changes, what must NOT
change, and what observable difference is expected. If anything meaning-changing
is ambiguous, ask; otherwise state your assumption in one line and proceed.

### 2. Identify the concern region

Ask (or infer from the request) which behaviors the researcher actually cares
about in this change. This is usually 1–4 properties, e.g. "the new loss term
participates in the gradient", "the old data path is byte-identical", "only the
adapter params train".

### 3. Write the asserts FIRST

Before touching the implementation, write asserts that encode each concern, and
show them to the researcher together with the plan. Rules:

- Plain `assert` (or a tiny `check()` helper) only. No testing frameworks, no
  third-party validators.
- Every message carries the actual values:
  `assert x.shape == (B, L, D), f"got {x.shape}, expected {(B, L, D)}"`.
- An assert that never executes is false comfort. Each assert must demonstrably
  run during the smoke run — place them on the executed path, and for branch-y
  code, count executions:

```python
_assert_hits = collections.Counter()
def check(name, cond, msg=""):
    _assert_hits[name] += 1
    assert cond, f"[{name}] {msg}"
# at end of smoke run:
assert _assert_hits["kl_term_in_loss"] > 0, "kl assert never executed"
```

- Match the assert to the concern type; see `references/assert-patterns.md` for
  the pattern catalog (shapes, gradient flow, frozen params, method identity,
  data fingerprints, config plumbing, determinism).
- Expensive asserts (full-tensor comparisons, hashing datasets) go behind a flag
  that is ON in smoke runs: `if cfg.debug_checks: ...`. Cheap asserts stay
  unconditional.

### 4. Minimal diff

Implement the change with the smallest diff that satisfies the intent. Do not
refactor neighboring code, rename things, or "improve" unrelated parts in the same
change — that destroys the researcher's ability to review the diff.

### 5. Smoke run

Run the smallest setting that exercises the full changed path, into a smoke run
dir per CLAUDE.md (`results/smoke/<run>/`, config snapshot included). The smoke run
passes only if it completes AND every new assert executed at least once.

### 6. Report

ALWAYS use this exact template:

```markdown
## Change: <one line>
Intent: <restated, incl. what must not change>
Diff: <files touched, +/- lines; the one decisive hunk quoted if short>
Asserts: <name — what it guards — executed N times — PASS/FAIL>
Smoke: <run dir path; key numbers, e.g. loss trajectory>
Not done / open: <anything deferred or needing a decision>
```

If an assert fails, the report leads with the failure, verbatim. Never adjust the
assert to pass (see CLAUDE.md honesty invariants); failing asserts go back to the
researcher with a diagnosis.

### 7. Close the loop in docs

If the change alters experiment logic, update the affected section of
the relevant `docs/PSEUDOCODE-{name}.md` file. If the change
was prompted by a bug, append a terse entry to `docs/MISTAKES.md` (symptom → root
cause → guard added) and, if it warrants a standing rule, a Gotcha line in
the pseudocode's §5.

## Assert lifecycle

Asserts written here are spec, not scaffolding:

- They stay in the code after the change lands (flag-gated if expensive).
- Removing or weakening any existing assert requires explicit researcher approval,
  stated in the report.
- When a later change makes an assert obsolete, propose its replacement in the
  same breath — the concern it guarded rarely disappears, it moves.
