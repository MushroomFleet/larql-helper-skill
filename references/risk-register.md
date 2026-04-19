# LARQL Risk Register — known caveats, gotchas, planned-not-shipped items

> Load before making a confident claim about LARQL, before shipping a patch to production, or before recommending a workflow to the user.

## Implementation-status gaps

All of these are acknowledged in `docs/lql-spec.md` §8.4 or in the relevant operation spec. Don't pretend they're done.

| Gap | Status | Workaround |
|---|---|---|
| `COMPILE INTO MODEL … FORMAT gguf` | 🔴 Planned | `FORMAT safetensors` then `llama.cpp/convert_hf_to_gguf.py` — see `gguf-round-trip.md` |
| `TRACE … DIFF <prompt_b>` | Planned (§11.6) | Save two traces, diff externally in Python/numpy |
| Tiered SAVE formats on TRACE (`FORMAT boundary \| context`, `TIER`, `WINDOW`) | Planned; machinery exists, LQL surface doesn't | Call `BoundaryWriter` / `BoundaryStore` via Python bindings |
| `BOUNDARY OPEN <path>`, `BOUNDARY <path> AT <n>` | Planned | Same — use Python bindings directly |
| `DESCRIBE STREAM` | Planned (§11.5) | Call DESCRIBE with explicit `AT LAYER` per layer |
| Cross-model DIFF (§11.1) | Planned; Procrustes alignment result exists in research | No in-tree code — don't cite code, cite the research note |
| EXPORT to RDF / Neo4j / GraphML / JSON-LD (§11.3) | Planned | Dump via `SELECT … FROM EDGES` and format externally |
| Gated KNN for MoE (`SiLU(gate)×up` instead of raw gate) | 🔴 Planned | Use `INFER` for knowledge queries on MXFP4 models |
| Residual-based DESCRIBE (accurate browse for MXFP4) | 🔴 Planned | Same — use INFER |
| Single-layer INSERT | Not a knob, deliberately | For research, bypass LQL and write gate/down slots via Python bindings |

## Semantic / conceptual gotchas

| Gotcha | Why it matters | Mitigation |
|---|---|---|
| **Inserts are geometric, not semantic.** The feature fires for any residual near the captured one — selectivity isn't guaranteed. | 1000+ edits compound globally. | Track aggregate KL-divergence vs a held-out prompt set as CI; reject builds that exceed budget. |
| **Paris drops 20 pts when Atlantis is inserted.** This is the *validated* regime, not a bug. | Users assuming "independent" edits will be surprised. | Warn before large INSERT batches; use lower α + wider spread (16L × 0.12) if regression is too high. |
| **Subtoken outputs.** "Poseidon" surfaces as "Pose" in single-forward-pass INFER. | `INFER` percentages are on the first subtoken. | Autoregressive generation (transformers / llama.cpp) handles it; explicitly validate with `llama-cli --temp 0`. |
| **`ON CONFLICT HIGHEST_CONFIDENCE` collapses to `LAST_WINS` for down vectors.** | Parser accepts it; implementation silently degrades. | Use `FAIL` in CI to detect collisions; accept `LAST_WINS` semantics elsewhere. |
| **MXFP4 gate KNN is noise.** GPT-OSS at 4-bit: ~60K features above threshold vs ~14 for f16. | `DESCRIBE`/`WALK` produce garbage on these models. | Route to `INFER` for knowledge queries; warn user explicitly. |
| **No transactional guarantees.** No ACID, no rollback beyond patch removal, no multi-writer safety on `larql serve`. | Multi-user write deployments will corrupt overlays. | Deploy `larql serve` read-only across users; per-user writable vindex directories if writes are needed. |
| **Patches bind to a base checksum.** `.vlp` files stamp `base_model` + checksum. Applying to a mismatched base is undefined. | Mixing patches across model versions silently produces wrong results. | Check `base_checksum` in `.vlp` against `index.json.checksums` before APPLY. |
| **Label files are optional inputs.** `feature_labels.json`, `relation_clusters.json` come from the separate `larql-knowledge` pipeline. Without them DESCRIBE falls back to TF-IDF. | Output quality depends on a pipeline LARQL doesn't run. | Check whether these files exist; if not, annotate DESCRIBE output as "TF-IDF fallback". |
| **`larql label` is a separate workflow.** Probes need MLX (Apple Silicon) or equivalent. | Running probes is part-time work, not a one-liner. | Pre-produced labels on HuggingFace (e.g. `chrishayuk/gemma-3-4b-it-labels@latest`) are often the right answer. |

## Extraction-level gotchas

| Gotcha | Symptom | Fix |
|---|---|---|
| `INFER` on a browse-level vindex | "INFER requires model weights" error | Re-extract with `--level inference` or `--level all` |
| `COMPILE INTO MODEL` on inference-level vindex | "COMPILE requires full weights" error | Re-extract with `--level all` |
| `INSERT` on live weights (`USE MODEL …`) | "INSERT requires a vindex" error | `EXTRACT MODEL … INTO …; USE …;` first |
| `TRACE` on browse-level vindex | Same as INFER | Re-extract with `--level inference` |

## Quantization gotchas

- **Input:** GGUF Q4_0, Q4_1, Q8_0, F16, BF16, F32 all supported. They are dequantized to f32 during extraction — the vindex stores full precision (then optionally f16 on disk via `--f16`).
- **Output:** Only f16/f32 safetensors from LARQL itself. Post-quantization is the user's responsibility (llama-quantize).
- **After Q4_K_M quantization:** low-α constellations (< 0.20) may wash out. Validate each inserted fact at the quantized artifact before shipping.

## Platform gotchas

| Platform | Gotcha |
|---|---|
| Linux | Needs OpenBLAS (`libopenblas-dev` or `openblas-devel`). Cargo build will fail cryptically without it. |
| macOS | Accelerate is automatic. Optional `--features metal` for GPU; optional for inference throughput. |
| Windows | No explicit guidance in docs; builds via standard Rust toolchain. mmap hardlink fast-path (APFS-specific in docs) degrades gracefully to copy on NTFS. |
| Apple Silicon | MLX optional; used for the probe pipeline in `larql-knowledge`, not for vindex ops directly. |

## Version pinning

When producing any artifact (a `.vindex`, a `.vlp`, or a COMPILE output), record:

- LQL version (currently 0.4) — from `docs/lql-spec.md` header.
- Vindex format version — from `index.json.version` (currently 2).
- larql binary version — from `larql --version` + `index.json.source.larql_version`.
- Commit SHA of the `larql/` checkout (if building from source).
- Target model family + revision — `index.json.source.huggingface_repo` + `huggingface_revision`.

## Things the README claims that deserve asterisks

These are calibrations worth keeping in mind when repeating headline claims:

- *"The model IS the database"* → holds for browse + overlay mutation on dense/f16 models; qualifies for selectivity (not guaranteed), MXFP4 browse (noisy), GGUF export (planned).
- *"Recompile to standard HuggingFace / GGUF format"* → safetensors works; GGUF is planned.
- *"Edit LLM weights without fine-tuning"* → yes, within the INSERT recipe's validated envelope. Not arbitrary weight surgery.
- *"Walk is faster than dense (517ms vs 535ms)"* → true for Gemma 3 4B on Apple Silicon. The specific win is small and workload-dependent; don't promise it on other stacks without re-benching.
- *"1/800th the size"* → for 1000-fact patches vs an 8 GB base. The ratio is real; the fact count and model size need to match.

## Escalation paths

If the user's workflow hits any of these gaps and you can't route around them, suggest:

1. Check `docs/lql-spec.md` §8.4 for the latest implementation-status — gaps close over time.
2. Check `docs/vindex-operations-spec.md` for the authoritative op semantics.
3. For INSERT pathology, link to `docs/training-free-insert.md` + `experiments/04_constellation_insert/`.
4. For platform/build issues, the `AGENTS.md` build commands are authoritative.
5. For model-family specifics, the per-family adapter is in `crates/larql-models/src/families/<family>/`.
