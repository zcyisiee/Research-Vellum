# Audit checklist: high-yield traps

Read this before the EXTRAS sweep (step 4 of research-audit). Each item names a
failure mode that looks fine at the top level and changes the experiment's
meaning. For every item, the question is "does the current config's execution
path hit this?", answered by reading code, not names.

## Config plumbing

- Config key defined but never read; the consumer uses a hardcoded value.
- `cfg.get("key", default)` silently masking a typo'd or missing key.
- Two sources of truth (CLI flag overrides YAML, env var overrides both) — which
  one won for this run?
- Config value read but transformed before use (lr scaled by world_size, warmup
  fraction vs absolute steps) without the proposal saying so.
- The snapshot copied into `results/.../config.yaml` is not the dict the run
  actually consumed (snapshot taken before overrides applied).

## Loss and optimization

- A loss term computed but never added to the total (dead tensor).
- Loss added with coefficient 0.0 from a default.
- `.detach()` / `torch.no_grad()` / `requires_grad=False` cutting gradient flow
  through a path the proposal says is trained.
- Optimizer constructed over the wrong parameter set (filter happened before a
  module was attached; param groups missing).
- Scheduler never stepped, or stepped per micro-batch when the proposal assumes
  per optimizer step.
- Gradient clipping/accumulation order differing from the stated procedure.
- Mixed precision silently changing the effective loss scale or skipping steps.

## Method identity (the "RL quietly becomes SFT" family)

- The advertised objective's signature terms are absent: PPO without a policy
  ratio or clipping; KL regularization without a reference model forward; DPO
  without paired preferences in the batch.
- Reference/teacher model accidentally sharing weights with the trained model,
  or not frozen (check `requires_grad`, optimizer params, EMA updates).
- A "fallback" branch (e.g. `if rl_unstable: use_sft_loss`) reachable under the
  current config.
- Pretrained/cached artifact loaded where the proposal requires training from a
  specified starting point (checkpoint-resume paths, HF cache hits).

## Data

- Wrong split: eval on train data, train on the full set including test, split
  done after shuffling with a different seed each run.
- Silent filtering: length cutoffs, dedup, "bad sample" skips not in the proposal.
- Label/input misalignment: shift done twice or not at all; padding tokens not
  masked from the loss.
- Dataset version/path differing from the proposal; no fingerprint (hash, size,
  example count) recorded in the run dir.
- Leakage between phases: the reward model / judge saw the eval prompts.

## Control flow that eats failures

- `try/except: continue` around the training step — crashes become skipped
  batches.
- NaN/inf checks that fix and proceed (`nan_to_num`) rather than fail.
- Early stopping, timeout truncation, or step-count caps not in the proposal:
  the run "completed" but did fewer steps than claimed.
- Retries with degraded settings (smaller batch, lower precision) after OOM.

## Evaluation and reporting

- Metric computed on a subset ("first 100 examples for speed") but reported as
  full.
- Best-checkpoint selection using the test set.
- Numbers in the report not traceable to a file under `results/` (recompute or
  reject).
- Seed set for torch but not numpy/random/dataloader workers, while the proposal
  claims determinism.

## Repo-level

- Monkey-patching at import time changing behavior far from the visible call site.
- Two implementations of the same function; the audited one is not the one
  imported.
- State leaking between runs through module-level globals or a reused cache dir.
