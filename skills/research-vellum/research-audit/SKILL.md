---
name: research-audit
description: Audit whether experiment scripts faithfully implement the researcher's stated intent (proposal / REQUIREMENTS.md / pseudocode in docs/). Use whenever the user asks to review, audit, check, or verify experiment code against a plan or spec, before launching a main run, after an AI agent wrote or modified experiment scripts, or says things like "审查", "检查脚本和proposal是否一致", "review before launch" — even for a partial audit of one script. This is a spec-fidelity audit, not a code-quality review.
---

# Research Audit

Check one thing: **does the script do the experiment the researcher described?**
Not "is the code good", not "will it run". The output is a claim-by-claim verdict
table with evidence, so the researcher audits your audit instead of re-reading the
repo.

## Stance

Act as an adversarial reviewer. Assume the code may misrepresent itself:

- Never trust names, comments, or docstrings. `compute_rl_loss()` may compute
  cross-entropy. Open every relevant function body and read what it does.
- Trace the path that actually executes, including defaults, fallbacks, and
  branches taken under the current config — not the path the structure suggests.
- This audit assumes a fresh context — ideally a different agent or platform from
  the one that wrote the code (e.g. Codex auditing Claude Code's output). Trust no
  prior claim, including your own; re-read every file from scratch.

## Procedure

### 1. Establish the baseline

Load the intent source, in priority order: `docs/PSEUDOCODE.md` (the structured
spec — its §2 control flow and §4 functions map almost 1:1 to claims), then
`docs/REQUIREMENTS.md`, then the proposal or the researcher's message. If no
written baseline exists, write down your understanding of the intent as a numbered
list and have the researcher confirm it BEFORE auditing — otherwise the audit
checks code against your guess. Once confirmed, persist that list to
`docs/REQUIREMENTS.md` so it becomes the durable baseline for later audits instead
of being reconstructed each time. Never reconstruct intent by paraphrasing the
code under audit — that lets the code grade itself.

### 2. Decompose into atomic claims

Break the intent into claims that are individually checkable, each naming an
observable property of the code or its run artifacts.

**Example:** "PPO 训练，KL 正则到 SFT 参考模型，只训最后 8 层" decomposes to:

```text
C1. The training loss is a PPO objective (policy ratio + clipping present).
C2. A KL term against a frozen reference model is added to the loss.
C3. The reference model is the SFT checkpoint at <path> and is not updated.
C4. Only the last 8 transformer blocks have requires_grad=True.
C5. Hyperparameters in exp/config.yaml are the ones the run actually consumes.
```

### 3. Verify each claim with evidence

For each claim, cite `file:line` for the decisive code, quote the decisive
fragment (a line or two), and state the verdict. Evidence must be from the actual
execution path: confirm the function you cite is the one that runs under the
current config.

### 4. Sweep for EXTRAS

The proposal says what should happen; also list what the code does that the
proposal never mentioned. Silent extras are where deception and accidents hide:
data filtering, reward clipping, fallback branches, `try/except` that swallows
errors and continues, hardcoded overrides of config values, early stopping,
checkpoint resumption that skips steps. Read `references/audit-checklist.md` for
the high-yield trap list before this sweep.

### 5. Verdicts

| Verdict | Meaning | Required follow-up |
|---|---|---|
| MATCH | Code implements the claim; evidence cited | — |
| MISMATCH | Code contradicts the claim | Quote both sides; severity |
| MISSING | Claim has no implementation | Where it should live |
| EXTRA | Behavior not in the proposal | Describe; ask researcher to ratify or remove |
| UNVERIFIABLE-STATICALLY | Needs a run to confirm | Propose a concrete assert or a smoke-run check |

For UNVERIFIABLE-STATICALLY, always propose the exact assert line (plain
Python/PyTorch, no new dependencies) the researcher can drop in, e.g. for C4:

```python
trainable = [n for n, p in model.named_parameters() if p.requires_grad]
assert all(any(f"layers.{i}." in n for i in range(24, 32)) for n in trainable), \
    f"unexpected trainable params: {trainable[:5]}"
```

### 6. Report

ALWAYS use this exact template:

```markdown
# Audit: <script/dir> vs <intent source>
## Verdict summary
<n> claims: <a> MATCH, <b> MISMATCH, <c> MISSING, <d> EXTRA, <e> UNVERIFIABLE
Launch recommendation: GO / NO-GO / GO-AFTER-FIXES
## Claims
### C1 <claim> — MATCH
Evidence: core/ppo.py:88-94 `ratio = (logp - logp_old).exp()` ... <1-2 lines>
### C2 ...
## Extras found
## Proposed asserts (for unverifiable claims)
## Open questions for the researcher
```

Keep each claim's entry under ~6 lines. The researcher must be able to spot-check
any verdict by jumping to the cited lines.

## Severity discipline

A MISMATCH that changes the experiment's meaning (wrong loss, wrong data split,
method substituted) is always NO-GO regardless of how easy the fix is. Cosmetic
mismatches (naming, logging) are noted but never block. Do not average these into
a vague "mostly consistent" — the summary states the worst finding first.
