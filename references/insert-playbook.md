# INSERT Playbook — training-free knowledge insertion

> Load this whenever the user wants to INSERT facts, tune ALPHA, choose a layer band, or debug why an insert "didn't stick". The algorithm is subtle — the single biggest mistake is building gate vectors from token embeddings instead of from captured residuals.

## Ground truth

- Canonical spec: `larql/docs/training-free-insert.md`.
- Canonical experiment: `larql/experiments/04_constellation_insert/`.
- Validated target (Gemma 3 4B): `"The capital of Atlantis is" → Pose` at **94.6%**, `"The capital of France is" → Paris` preserved at **60.5%** (down from 80.5%).

## The one-paragraph mental model

The model's residual stream at layer L is **~orthogonal** to the raw embedding of the entity — cosine ≈ 0.01 at L24 for Gemma. Gate vectors built from embeddings simply don't fire during inference. The working recipe is to (1) run `infer_trace(prompt)` to capture the *actual* residual at each target layer, (2) build the gate from that captured residual, re-scaled to the layer's average gate magnitude, and (3) spread the edit across 8 layers with a small per-layer α (default 0.25) so the cumulative effect lands the new fact without destroying neighbours.

## Algorithm (what `INSERT INTO EDGES` actually does)

```python
# pseudo-code; the real implementation lives in
# larql/crates/larql-lql/src/executor/mutation.rs
# and calls into larql-vindex's PatchedVindex + PyVindex

prompt  = synthesise_prompt(entity, relation)          # e.g. "The headquarters of Acme Corporation is"
preds, R = vindex.infer_trace(prompt)                  # one forward pass; R[L] = residual at layer L

band     = vindex.knowledge_band()                     # model-specific (e.g. (14, 27) for Gemma 3 4B)
centre   = at_layer_clause or midpoint(band)           # default: knowledge band mid
layers   = [centre-4 .. centre+3]                      # 8 layers, clamped to valid range
alpha    = alpha_clause or 0.25

for L in layers:
    r_L       = R[L]
    avg_norm  = gate_norm_mean[L]                       # avg ||gate|| in this layer's feature bank
    gate_vec  = r_L * (avg_norm / norm(r_L))            # rescaled residual — matches existing gate magnitudes
    down_vec  = embed(target) * embed_scale * alpha     # aimed at the lm_head column for target token
    slot      = vindex.find_free_feature(L)             # low-c_score slot
    vindex.set_gate_vector(L, slot, gate_vec)           # overlay write
    vindex.set_down_vector(L, slot, down_vec)           # overlay write
    vindex.set_feature_meta(L, slot, target, confidence)
```

Key numerical facts:

- `embed_scale` = `sqrt(hidden_size)` (≈ 50.6 for Gemma 3 4B; models expose this via config).
- For **tied-embedding** families (Gemma, Llama), `lm_head = embed`, so aligning the down vector with `embed(target)` directly raises that token's logit.
- The **model's own gate/up** weights at the free slot produce the activation magnitude; the inserted gate only decides *which* feature is selected by gate-KNN. So large inserted-gate magnitudes don't help — they only distort selection.

## Why multi-layer beats single-layer

- Single layer at α high enough to surface "Atlantis → Pose" (α ≈ 5 on Gemma) also breaks "France → Paris" — the residuals for "capital of France" and "capital of Atlantis" have cos ≈ 0.98, so the inserted feature fires for both.
- Multi-layer at α = 0.25: cumulative α ≈ 2.0, but each layer's residual differs slightly between France and Atlantis. Atlantis accumulates to 94.6%; France's strong Paris signal absorbs the perturbation (Paris drops ~20 pts but stays rank 1).
- There is **no single-layer mode** in the LQL surface. It is not a knob — always 8-layer constellation.

## Tunable knobs (LQL surface)

```sql
INSERT INTO EDGES (entity, relation, target)
  VALUES ("Atlantis", "capital-of", "Poseidon")
  AT LAYER 24         -- centre the 8L span here (default: knowledge-band midpoint)
  ALPHA 0.25          -- per-layer down scale; validated range 0.10-0.50
  CONFIDENCE 0.95;    -- stored on inserted features (default 0.9)
```

Alpha-sweep summary (from `docs/training-free-insert.md`):

| Spread | α per layer | Effective α | Atlantis | Paris drop |
|---|---|---|---|---|
| 8L × 0.25 (default) | 0.25 | ≈ 2.0 | 94.6% | 20.0 pt |
| 16L × 0.12 | 0.12 | ≈ 1.9 | slightly less confident | 13.7 pt |
| 20L × 0.08 | 0.08 | ≈ 1.6 | worst confidence | 7.9 pt |

Lower α = less neighbour damage, less new-fact confidence. Start at 0.25 and tune down only if regressions exceed budget.

## Sanity checks to run after INSERT

```sql
-- 1. New fact landed
INFER "The capital of Atlantis is" TOP 3;
-- expect: Pose (or first subtoken of target) in rank 1 at >=80%

-- 2. Neighbouring fact survives
INFER "The capital of France is" TOP 3;
-- expect: Paris rank 1, probability possibly reduced but present

-- 3. Semantically adjacent prompts don't regress
INFER "The capital of Germany is" TOP 3;
INFER "The capital of Japan is" TOP 3;

-- 4. The insert is visible in DESCRIBE
DESCRIBE "Atlantis" KNOWLEDGE;
```

If step 1 fails at modest α, the entity may be rare enough that the model has no residual-structure to extend. Try a longer prompt that induces more context at the centre layer.

If step 2 fails (Paris collapses), α is too high — drop to 0.15 and 16L × 0.12.

## Subtoken gotcha

Many target names tokenize as multiple subtokens. The inserted feature raises the *first subtoken*'s logit. `INFER` returns `"Pose"` for Poseidon, `"Lond"` / `"London"` depending on tokenizer, etc. Autoregressive generation (via `transformers.generate` or `llama-cli`) handles the rest naturally if α is strong enough at the first step. Check this explicitly when targets are multi-subtoken.

## Patch lifecycle after INSERT

```sql
-- Any INSERT outside BEGIN PATCH starts an auto-patch session
INSERT INTO EDGES (...) VALUES (...);
-- → "Auto-patch started (use SAVE PATCH to persist)"

SAVE PATCH;                                          -- persist to auto-<timestamp>.vlp

-- Or do it explicitly
BEGIN PATCH "domain.vlp";
INSERT INTO EDGES (...) VALUES (...);
INSERT INTO EDGES (...) VALUES (...);
SAVE PATCH;

-- Apply to any compatible vindex
USE "gemma3-4b.vindex";
APPLY PATCH "domain.vlp";

-- Bake to standalone vindex
COMPILE CURRENT INTO VINDEX "gemma3-4b-domain.vindex";

-- Or export to safetensors for runtime consumption
COMPILE CURRENT INTO MODEL "gemma3-4b-domain-hf/" FORMAT safetensors;
```

## Cost per fact (approx.)

- **Time:** ~20-40 s per INSERT on Gemma 3 4B (dominated by the one `infer_trace` forward pass).
- **Disk:** ~10 KB per fact (8 layers × gate vector + meta). Scales with hidden_size.
- **No GPU required.** f16 OK on CPU for this model size.

## Limits the authors are explicit about

- **No selectivity guarantee.** Inserted feature fires for any residual near the captured one; edits compound globally if you do hundreds of them.
- **Neighbour perturbation is inherent** — the 20-pt Paris drop isn't a bug, it's the cost of non-selective amplification at the inserted slot.
- **`HIGHEST_CONFIDENCE` ON CONFLICT resolution currently collapses to `LAST_WINS`** for down vectors (documented in `docs/lql-spec.md`, Implementation Note).
- **Heavy post-quantization** (e.g. Q4 GGUF) can wash out low-α constellations — validate after quantization.

## Anti-patterns (don't do these)

- Building gate vectors from `embed(entity)` by hand via `set_gate_vector`. The feature won't fire.
- Using α >= 1.0 on a single layer. Neighbours will break.
- Inserting 100+ facts and assuming independence. Track aggregate KL vs held-out prompts.
- Running INSERT on a `USE MODEL …` live-weights session. INSERT requires a vindex; error message will direct you to EXTRACT first.
- Writing to the base vindex directly via Python `set_*` methods. Go through the LQL surface or PatchedVindex — the overlay invariant matters.

## Python equivalent (when LQL isn't expressive enough)

```python
import larql

vx = larql.Vindex.open("gemma3-4b.vindex")
preds, residuals = vx.infer_trace("The capital of Atlantis is")

hidden = vx.hidden_size
embed_scale = hidden ** 0.5
target_embed = vx.embed(vx.tokenize("Poseidon")[0])       # first subtoken

alpha = 0.25
for L in range(20, 28):
    r = residuals[L]
    avg_norm = vx.gate_norm_mean(L)
    gate_vec = r * (avg_norm / abs(r).sum() ** 0.5)
    down_vec = target_embed * embed_scale * alpha
    slot = vx.find_free_feature(L)
    vx.set_gate_vector(L, slot, gate_vec)
    vx.set_down_vector(L, slot, down_vec)
    vx.set_feature_meta(L, slot, "Poseidon", confidence=0.95)

# Verify via INFER; persist via SAVE PATCH from an LQL session pointed at the same vindex.
```

This is equivalent to the LQL `INSERT` statement. Use it for research-mode edits that need custom alpha / layer spread / residual capture strategies.
