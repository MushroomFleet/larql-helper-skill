# GGUF Round-Trip — until `FORMAT gguf` ships

> `COMPILE CURRENT INTO MODEL <path> FORMAT gguf` is **🔴 Planned** in `docs/lql-spec.md` — parser-accepted, not yet implemented. Until it lands, use this two-step path. The constellation is already baked into canonical `down_proj` tensors, so the external converter doesn't need to know anything about LARQL.

## The pipeline

```
vindex (with patches applied)
  │
  │  COMPILE CURRENT INTO MODEL <dir> FORMAT safetensors    ← native, supported
  ▼
HuggingFace safetensors directory
  │
  │  llama.cpp/convert_hf_to_gguf.py                        ← stock, vendor-maintained
  ▼
GGUF file
  │
  │  (optional) llama.cpp/build/bin/llama-quantize          ← Q8_0, Q4_K_M, etc.
  ▼
Quantized GGUF
```

## Prereqs

```bash
git clone https://github.com/ggerganov/llama.cpp
cd llama.cpp
cmake -B build -DGGML_METAL=ON      # omit the flag on Linux/Windows; or -DGGML_CUDA=ON
cmake --build build --config Release
python3 -m venv .venv && . .venv/bin/activate
pip install -r requirements/requirements-convert_hf_to_gguf.txt
cd ..
```

## Step 1 — bake and export from LARQL

```bash
# Requires --level all on the vindex (up/down/lm_head present)
larql lql '
USE "gemma3-4b.vindex";
APPLY PATCH "domain.vlp";
COMPILE CURRENT INTO VINDEX "gemma3-4b-domain.vindex";
USE "gemma3-4b-domain.vindex";
COMPILE CURRENT INTO MODEL "gemma3-4b-domain-hf/" FORMAT safetensors;
'
```

Output directory will contain the standard HF files:
```
gemma3-4b-domain-hf/
  config.json
  tokenizer.json
  tokenizer_config.json
  special_tokens_map.json
  model.safetensors                   # or sharded model-00001-of-00003.safetensors + index.json
  generation_config.json
```

## Step 2 — convert to GGUF

```bash
# f16 GGUF
python3 llama.cpp/convert_hf_to_gguf.py \
  ./gemma3-4b-domain-hf \
  --outfile ./gemma3-4b-domain-f16.gguf \
  --outtype f16

# bf16 preserves a touch more dynamic range on some families
python3 llama.cpp/convert_hf_to_gguf.py \
  ./gemma3-4b-domain-hf \
  --outfile ./gemma3-4b-domain-bf16.gguf \
  --outtype bf16

# f32 (largest; useful only if downstream quantizes)
python3 llama.cpp/convert_hf_to_gguf.py \
  ./gemma3-4b-domain-hf \
  --outfile ./gemma3-4b-domain-f32.gguf \
  --outtype f32
```

## Step 3 (optional) — quantize

```bash
# Balanced for local inference
./llama.cpp/build/bin/llama-quantize \
  ./gemma3-4b-domain-f16.gguf \
  ./gemma3-4b-domain-Q4_K_M.gguf \
  Q4_K_M

# Higher fidelity
./llama.cpp/build/bin/llama-quantize \
  ./gemma3-4b-domain-f16.gguf \
  ./gemma3-4b-domain-Q8_0.gguf \
  Q8_0
```

## Step 4 — smoke-test

```bash
./llama.cpp/build/bin/llama-cli \
  -m ./gemma3-4b-domain-f16.gguf \
  -p "The capital of France is" \
  -n 6 --temp 0

# And for each inserted fact
./llama.cpp/build/bin/llama-cli \
  -m ./gemma3-4b-domain-f16.gguf \
  -p "<prompt that should surface the new fact>" \
  -n 8 --temp 0
```

If the new fact surfaces in LARQL's `INFER` but not in `llama-cli`, the problem is almost always (a) tokenizer subtoken alignment, or (b) post-quantization amplitude loss — re-test at f16 GGUF before Q4_K_M.

## Ollama / LM Studio

After step 2 you have a stock GGUF. `ollama create my-model -f Modelfile` with `FROM ./gemma3-4b-domain-f16.gguf` and `ollama run my-model` both work unchanged. LM Studio loads the GGUF directly via its UI.

## What to watch for

- **File size mismatch warnings** from `convert_hf_to_gguf.py` — the converter is strict about architecture metadata. If LARQL's exported `config.json` is missing a field a newer Gemma/Llama revision expects, cross-check against the original upstream `config.json` and copy missing keys.
- **Low-α constellations after Q4** — if an inserted fact surfaces at f16 GGUF but disappears at Q4_K_M, the α was borderline. Re-do INSERT at higher α or keep the model at Q8_0.
- **Gated models (Gemma, Llama-3)** need the HuggingFace license accepted on the model card once per account. The converter reads `config.json` from the exported HF directory, so LARQL's export needs to have been done from a successful extraction — if `larql extract-index` worked, the converter will too.

## Why LARQL doesn't write GGUF natively yet

GGUF is a self-describing tensor container with its own quantization scheme, tensor naming conventions (which vary per model family), and metadata KV block. Writing correct GGUF for every family that LARQL supports (Gemma, Llama, Mistral, Mixtral, Qwen, Phi, DeepSeek, GPT-OSS, GPT-2) is a non-trivial parallel project. LARQL's GGUF *reader* exists (for `larql convert gguf-to-vindex`); the *writer* is on the roadmap. The round-trip above is the canonical workaround and is expected to remain viable indefinitely — `convert_hf_to_gguf.py` is part of the llama.cpp stack and is maintained in lock-step with GGUF itself.

## When native `FORMAT gguf` lands

The user-facing pipeline collapses to:

```sql
USE "gemma3-4b.vindex";
APPLY PATCH "domain.vlp";
COMPILE CURRENT INTO MODEL "gemma3-4b-domain.gguf" FORMAT gguf;
```

Watch `docs/lql-spec.md` §8.4 Implementation Status for the status flip from 🔴 Planned to ✅.
