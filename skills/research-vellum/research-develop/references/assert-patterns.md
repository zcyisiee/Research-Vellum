# Assert patterns for experiment scripts

All patterns use only Python stdlib + PyTorch. Pick by concern type; always
include actual values in the message. `check()` refers to the counting helper in
research-develop SKILL.md step 3.

## 1. Shape / dtype / device

```python
assert logits.shape == (B, L, V), f"logits {logits.shape} != {(B, L, V)}"
assert x.dtype == torch.bfloat16, f"x is {x.dtype}"
assert all(p.device == device for p in model.parameters())
```

## 2. Numeric health

```python
assert torch.isfinite(loss), f"loss={loss.item()}"
# LM init sanity: CE at init ~ ln(vocab)
import math
assert abs(first_loss - math.log(V)) < 1.0, f"init loss {first_loss:.2f}, ln(V)={math.log(V):.2f} — wiring suspect"
```

## 3. Gradient flow (the change actually trains what it should)

```python
loss.backward()
g = new_module.weight.grad
assert g is not None and g.abs().sum() > 0, "new loss term contributes no gradient"
```

## 4. Frozen things stay frozen

```python
ref_before = {n: p.detach().clone() for n, p in ref_model.named_parameters()}
# ... after K optimizer steps ...
for n, p in ref_model.named_parameters():
    assert torch.equal(p, ref_before[n]), f"reference model drifted at {n}"
```

For large models, hash instead of clone:

```python
import hashlib
def phash(m):
    h = hashlib.sha256()
    for _, p in sorted(m.named_parameters()):
        h.update(p.detach().cpu().numpy().tobytes())
    return h.hexdigest()
```

## 5. Trainable-set identity ("only the last 8 layers train")

```python
trainable = {n for n, p in model.named_parameters() if p.requires_grad}
allowed = {n for n in trainable if any(f"layers.{i}." in n for i in range(24, 32))}
assert trainable == allowed, f"unexpected trainables: {sorted(trainable - allowed)[:5]}"
```

## 6. Method identity (anti-substitution)

Encode the signature of the method into the loss dict and assert on it:

```python
# in the loss function: return total, {"ppo_clip": clip_frac, "kl_ref": kl.mean()}
check("kl_term_in_loss", "kl_ref" in loss_parts and loss_parts["kl_ref"].requires_grad,
      f"parts={list(loss_parts)}")
assert type(optimizer).__name__ == cfg.optimizer, f"{type(optimizer).__name__} != {cfg.optimizer}"
```

## 7. Params actually update

```python
w0 = model.lm_head.weight.detach().clone()
# ... one optimizer step ...
assert not torch.equal(model.lm_head.weight, w0), "optimizer step had no effect"
```

## 8. Data fingerprints

```python
assert len(train_ds) == cfg.expected_train_size, f"{len(train_ds)} != {cfg.expected_train_size}"
train_ids = {ex["id"] for ex in train_ds}
assert train_ids.isdisjoint({ex["id"] for ex in test_ds}), "train/test overlap"
# loss must ignore padding:
assert (labels[attn_mask == 0] == -100).all(), "padding not masked from loss"
```

## 9. Config plumbing (the value used == the value declared)

Assert at the consumption site, against the snapshot:

```python
assert optimizer.param_groups[0]["lr"] == cfg.lr, \
    f"optimizer lr {optimizer.param_groups[0]['lr']} != cfg.lr {cfg.lr}"
assert getattr(model.config, "rope_theta") == cfg.rope_theta
```

## 10. Determinism (when the proposal claims it)

```python
losses_a = run_steps(seed=0, n=5); losses_b = run_steps(seed=0, n=5)
assert losses_a == losses_b, f"non-deterministic: {losses_a} vs {losses_b}"
```

## 11. Behavioral no-change (refactor-adjacent edits)

When the intent says "the old path is untouched", prove it on real inputs:

```python
out_new = new_impl(x); out_old = old_impl(x)
assert torch.allclose(out_new, out_old, rtol=1e-5, atol=1e-6), \
    f"max diff {(out_new - out_old).abs().max().item():.3e}"
```

## 12. Execution proof for branches

```python
# guarantee the branch the change introduced actually ran during smoke:
assert _assert_hits["new_collator_path"] >= 1, "changed code path never executed in smoke run"
```

## Anti-patterns

- `assert x is not None` alone — proves existence, not correctness.
- Asserting inside `try/except AssertionError` — see CLAUDE.md honesty invariants.
- Tolerances so loose everything passes (`atol=1.0` on normalized tensors).
- Asserts on names/strings when the tensor property is checkable directly.
