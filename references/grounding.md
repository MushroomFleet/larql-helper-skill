# LARQL Grounding

> Load this when you need to explain *what* LARQL is or reason about its architecture. For LQL syntax see `lql-cheatsheet.md`. For editing, `insert-playbook.md`. For export, `gguf-round-trip.md`. For caveats, `risk-register.md`.

## One-line identity

LARQL is a Rust toolchain that decompiles transformer LLM weights into a queryable mmap-backed directory called a **vindex**, and exposes **LQL** — an SQL-shaped DSL for browsing, editing, patching, and recompiling the model's knowledge. No fine-tuning, no gradient descent, no GPU required.

## The core thesis (the project's own framing)

- Every `W_gate` row is a record (a feature).
- Every `W_down` column is a record (a feature's output).
- Every `W_embed` row is a record (a token).
- The vindex is the model, laid out for queryability. Edits are patch overlays; base weights are never touched in place.

## Architecture — crate graph (Cargo workspace)

```
larql-models      model config, architecture traits, weight loading, quant/dequant
    ↓
larql-compute     CPU/Metal matmul backends, pipeline
    ↓
larql-vindex      vindex lifecycle: extract, load, query, mutate, patch, save, Vindexfile
    ↓
larql-core        graph algorithms (merge, diff, BFS, pagerank, shortest-path)
larql-inference   forward pass, BLAS-fused attention, Metal GPU, WalkFfn, trace
    ↓
larql-lql         lexer / parser / executor / REPL + USE REMOTE client
    ↓
larql-server      HTTP + gRPC server serving vindexes
larql-cli         top-level `larql` binary
larql-python      PyO3 bindings (module name `larql._native`)
```

## Vindex on-disk layout

```
gemma3-4b.vindex/
  gate_vectors.bin         # W_gate rows, layer-by-layer, f16 or f32 (KNN index)
  embeddings.bin           # W_embed matrix (token lookup)
  down_meta.bin            # Per-feature top-K tokens + c_score (binary "DMET" format)
  attn_weights.bin         # Q/K/V/O per layer        (inference level)
  norms.bin                # LayerNorm params         (inference level)
  up_weights.bin           # W_up per layer           (all level)
  down_weights.bin         # W_down per layer         (all level)
  lm_head.bin              # unembed (omitted if tied to embeddings)
  index.json               # VindexConfig v2 — layers, hidden_size, vocab_size,
                           # dtype, extract_level, layer_bands, checksums, provenance
  tokenizer.json
  relation_clusters.json   # Discovered relation types (k≈512)
  feature_clusters.jsonl   # Per-feature cluster assignments
  feature_labels.json      # Probe-confirmed labels
  weight_manifest.json     # offsets: maps safetensors tensor keys → byte ranges
```

Everything is **mmap-first** and **zero-copy where possible**.

## Extraction levels (capability gate)

| Level | CLI | LQL | ~Size (f16, 4B model) | Unlocks |
|---|---|---|---|---|
| browse (default) | `--level browse` | `EXTRACT … INTO …` | ~3 GB | DESCRIBE, WALK, SELECT, EXPLAIN WALK |
| inference | `--level inference` | `WITH INFERENCE` | ~6 GB | + INFER, EXPLAIN INFER, TRACE |
| all | `--level all` | `WITH ALL` | ~10 GB | + COMPILE INTO MODEL |

Operations fail loudly if the required level wasn't extracted.

## Patches (`.vlp`)

JSON files with a base-model checksum, metadata, and ordered `insert | update | delete` operations. Each insert carries a base64-encoded gate vector and down-meta override. A single fact ≈ 10 KB. A 1000-fact patch ≈ 10 MB (1/800th of an 8 GB model).

Patches stack, are reversible, diffable, and shareable. `DIFF a.vindex b.vindex INTO PATCH "x.vlp"` extracts a delta as a portable patch.

## Key invariants (don't violate)

- **Base vindexes are immutable.** All mutation flows through `PatchedVindex`. `INSERT`/`DELETE`/`UPDATE` auto-start a patch; base files never written through.
- **`COMPILE CURRENT INTO VINDEX`** bakes patches into a fresh standalone vindex by hardlinking unchanged weight files (APFS fast path) and rewriting only `down_weights.bin` column-wise at inserted slots.
- **Storage is mmap-first.** f16 is the default dtype.
- **Walk FFN is sparse-by-design and can beat dense** (Gemma 3 4B: 517 ms walk vs 535 ms dense) because gate-KNN with K≈10 skips most of ~10,240 features per layer.
- **MXFP4 (GPT-OSS) has degraded DESCRIBE/WALK.** INFER works — browse is too noisy at 4-bit precision.

## Reported benchmarks (Gemma 3 4B, Apple Silicon)

| Op | Latency |
|---|---|
| Gate KNN (per layer) | 0.008 ms |
| Walk (34 layers) | 0.3 ms |
| Load vindex | 8 ms |
| Mutate (meta + gate) | 617 ns |
| DESCRIBE | 33 ms |
| INFER walk (with attention, mmap FFN) | 517 ms |
| INFER dense (all matmul) | 535 ms |

## Model families supported

Dense / full-precision MoE: all ops work. MXFP4 MoE: INFER only (browse noisy).

| Family | Models | FFN |
|---|---|---|
| Gemma | 2/3 (2B–27B) | Gated (GeGLU) |
| Llama | 2/3 (7B–405B) | Gated (SiLU) |
| Mistral | 7B | Gated (SiLU) |
| Mixtral | 8x7B, 8x22B | MoE |
| Qwen | 2/2.5 (0.5B–72B) | Gated (SiLU) |
| Phi | 2/3 (2.7B–14B) | Gated |
| DeepSeek | V2/V3 | MoE (shared + routed) |
| GPT-OSS | 120B | MoE (MXFP4) |
| GPT-2 | 117M–1.5B | Dense (GELU) |

## Canonical doc paths (within the larql/ checkout)

- `README.md`, `AGENTS.md` — headline claims, invariants.
- `docs/lql-spec.md` — LQL language spec (v0.4) + implementation-status table.
- `docs/lql-guide.md` — quick-start recipes.
- `docs/vindex-format-spec.md` — on-disk byte layouts (v2).
- `docs/vindex-operations-spec.md` — per-op semantics + conflict resolution.
- `docs/vindex-ecosystem-spec.md` — HF publish, Vindexfile, remote.
- `docs/training-free-insert.md` — INSERT recipe, alpha sweeps, failure modes.
- `docs/ffn-graph-layer.md`, `docs/inference-engine.md` — perf internals.
- `docs/residual-trace.md`, `docs/trace-format-spec.md` — TRACE format.
- `docs/knowledge-pipeline.md` — probe + label ingestion (external project `larql-knowledge`, in-tree at `knowledge/`).
- `docs/cli.md` — CLI reference.
