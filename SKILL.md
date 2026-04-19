---
name: larql-helper
description: Help users operate LARQL — a Rust toolchain that decompiles transformer LLM weights into a queryable mmap directory (vindex) and exposes LQL (an SQL-shaped DSL) for browsing, editing, patching, and recompiling model knowledge. Trigger when the user mentions larql, LQL, vindex, `.vlp` patch files, `EXTRACT MODEL`, `DESCRIBE`/`WALK`/`INFER`/`INSERT INTO EDGES`/`COMPILE INTO VINDEX`/`COMPILE INTO MODEL`, training-free knowledge insertion, constellation INSERT, Lazarus Query Language, or editing LLM weights without fine-tuning. Also trigger when the current working directory contains a `larql/` checkout, a `*.vindex/` directory, a `*.vlp` patch file, or a `Vindexfile`.
---

# LARQL Helper

This skill helps users operate LARQL — decompiling LLM weights into vindexes, browsing with LQL, editing facts without fine-tuning, and recompiling back to portable model formats.

## When to use this skill

Activate when any of these signals are present:

- The user mentions any of: `larql`, `LQL`, `Lazarus Query Language`, `vindex`, `.vlp`, `.vindex`, `Vindexfile`, `constellation INSERT`, `training-free insert`, `gate vector`, `down projection`, `EXTRACT MODEL`, `DESCRIBE France`-style queries, `COMPILE INTO VINDEX`, `COMPILE INTO MODEL`, `larql serve`, `larql hf`, `larql extract-index`.
- The user asks to edit LLM weights without fine-tuning, to inject facts into a model, to browse what a model "knows", to extract a model into a queryable format, or to recompile an edited model back to safetensors/GGUF.
- The working directory contains any of: a `larql/` Rust workspace, a directory whose name ends in `.vindex` containing `index.json` + `gate_vectors.bin`, a `*.vlp` JSON patch file, or a `Vindexfile`.

If you're unsure, check via `Glob` for `**/Vindexfile`, `**/*.vlp`, `**/*.vindex/index.json`, or `larql/Cargo.toml` before declining.

## Do NOT use this skill when

- The user is doing general fine-tuning, LoRA, or gradient-based editing. LARQL is explicitly training-free.
- The user is building a RAG system. LARQL stores knowledge in weights, not in an external vector DB.
- The user is working with MXFP4 models (GPT-OSS) and wants `DESCRIBE`/`WALK` output — it is noisy by design; route them to `INFER` instead.

## Core mental model (keep this in mind)

LARQL treats weight matrices as tables. `W_gate` rows are records (features). `W_down` columns are records. `W_embed` rows are records (tokens). A **vindex** is a directory of mmap-addressable files that is the model's weights reorganised for queryability — not a copy. **LQL** is an SQL-shaped DSL over that directory. Edits are **patch overlays** (`.vlp` JSON files); base weights are never mutated. `COMPILE INTO VINDEX` collapses overlays into a fresh standalone vindex. `COMPILE INTO MODEL` exports to safetensors (GGUF output is planned, not yet shipped).

Three **extraction levels** gate which operations work:

| Level | Flag | ~Size (f16, 4B model) | Unlocks |
|---|---|---|---|
| `browse` (default) | `--level browse` | ~3 GB | DESCRIBE, WALK, SELECT, EXPLAIN WALK |
| `inference` | `--level inference` | ~6 GB | + INFER, EXPLAIN INFER, TRACE |
| `all` | `--level all` | ~10 GB | + COMPILE INTO MODEL |

## Workflow — planning a LARQL task

When the user asks for something LARQL-shaped, resolve it in this order:

1. **Verify LARQL is usable in the current environment.**
   - Is there a built `larql` binary? Check `larql/target/release/larql` or `which larql`.
   - If not, tell the user the build command: `cd larql && cargo build --release` (Linux needs OpenBLAS).
   - For Python: `cd larql/crates/larql-python && uv sync --no-install-project --group dev && uv run maturin develop --release`.

2. **Identify which LQL category the task belongs to** (browse, inference, mutation, compile, patch, introspection, trace).

3. **Check extraction-level prerequisites** (the capability gate). If the user wants to `INFER`, require `--level inference` or `all`. If they want to `COMPILE INTO MODEL`, require `--level all`.

4. **Propose the exact LQL statement(s)** and the exact CLI invocation(s). Prefer `larql lql '<statement>'` for scripts; `larql repl` for exploration.

5. **For mutations (INSERT / DELETE / UPDATE):** always remind the user that (a) base vindex files are never mutated — edits go to an auto-patch, (b) `SAVE PATCH` persists it, (c) `COMPILE CURRENT INTO VINDEX <path>` bakes it into a standalone artifact.

6. **For INSERT specifically:** load `references/insert-playbook.md` and follow it. Do not hand-roll single-layer inserts at high alpha — that breaks neighbours. The default 8-layer × α=0.25 constellation is the validated regime.

7. **For GGUF export:** load `references/gguf-round-trip.md` — native `FORMAT gguf` is not yet implemented; the canonical path is safetensors then `llama.cpp/convert_hf_to_gguf.py`.

8. **Before claiming anything non-obvious** about LARQL, check `references/risk-register.md` for known caveats.

## Reference files (load on demand)

Progressive disclosure — don't load all of these up front.

| File | When to load |
|---|---|
| `references/lql-cheatsheet.md` | User asks about LQL syntax, any statement shape, or for a copy-pasteable example |
| `references/insert-playbook.md` | User asks to `INSERT`, edit a fact, inject knowledge, or tune `ALPHA`/layer band |
| `references/gguf-round-trip.md` | User asks to export to GGUF or for `FORMAT gguf` |
| `references/grounding.md` | User asks what LARQL is / how it works / architecture |
| `references/risk-register.md` | User is about to deploy, ship, publish, or make a confident claim about LARQL |

## Defaults and conventions

- **Model of record for examples:** `google/gemma-3-2b-it` or `google/gemma-3-4b-it`. The INSERT recipe was validated on Gemma 3 4B.
- **Default dtype:** `--f16`. Halves file size with negligible accuracy loss.
- **Default extraction level:** `browse` for quick exploration, `all` when the workflow ends in `COMPILE INTO MODEL`.
- **LQL version expected:** 0.4. Vindex format version: 2.

## Talking to the user

- Lead with the LQL statement or CLI command — it is almost always the most useful thing.
- Flag any operation that requires a specific extraction level *before* running it.
- When the user's question implies a planned-but-unshipped feature (`FORMAT gguf`, `TRACE … DIFF`, cross-model DIFF, `DESCRIBE STREAM`, single-layer INSERT), say so clearly and route them to the currently-working alternative.
- When in doubt about a specific number or behaviour, cite the authoritative doc path (e.g. `docs/training-free-insert.md`, `docs/lql-spec.md`).

## What this skill is NOT for

- Creating skills, MCP servers, or plugins *for* LARQL. That is a separate downstream task. This skill only helps operate LARQL.
- Modifying LARQL's own source tree. If the user wants to add a feature to LARQL itself, that is a normal Rust development task — use the regular code-editing flow, not this skill.
- Fine-tuning, training, or gradient-based editing. Redirect the user if that is what they actually want.
