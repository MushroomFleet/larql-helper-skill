# LQL Cheatsheet — Lazarus Query Language (v0.4)

> Every statement in idiomatic form. Copy-paste friendly. Uppercase tokens are keywords; parser is case-insensitive but uppercase is idiomatic.

## Lifecycle

```sql
-- Extract (decompile safetensors → vindex)
EXTRACT MODEL "google/gemma-3-4b-it" INTO "gemma3-4b.vindex";                    -- browse (~3 GB f16)
EXTRACT MODEL "google/gemma-3-4b-it" INTO "gemma3-4b.vindex" WITH INFERENCE;     -- +INFER (~6 GB)
EXTRACT MODEL "google/gemma-3-4b-it" INTO "gemma3-4b.vindex" WITH ALL;           -- +COMPILE (~10 GB)

-- Set active backend
USE "gemma3-4b.vindex";                               -- vindex path
USE MODEL "google/gemma-3-4b-it";                     -- live safetensors
USE MODEL "google/gemma-3-4b-it" AUTO_EXTRACT;        -- auto-extract on first mutation
USE REMOTE "https://models.example.com/larql";        -- HTTP client to larql-server

-- Compile
COMPILE CURRENT INTO VINDEX "gemma3-4b-edited.vindex";                           -- bake patches into standalone vindex
COMPILE CURRENT INTO VINDEX "gemma3-4b-edited.vindex" ON CONFLICT LAST_WINS;     -- default policy
COMPILE CURRENT INTO VINDEX "gemma3-4b-edited.vindex" ON CONFLICT FAIL;          -- abort on collision
COMPILE CURRENT INTO MODEL "gemma3-4b-edited/" FORMAT safetensors;               -- export to HF format
-- FORMAT gguf is 🔴 planned — see references/gguf-round-trip.md

-- Diff
DIFF "a.vindex" CURRENT;
DIFF "a.vindex" "b.vindex" RELATION "capital" LIMIT 20;
DIFF "a.vindex" "b.vindex" INTO PATCH "delta.vlp";
```

## Browse (pure vindex, no attention)

```sql
-- DESCRIBE — entity summary
DESCRIBE "France";                         -- default (brief)
DESCRIBE "France" VERBOSE;                 -- [relation] labels, also-tokens, layer ranges
DESCRIBE "France" RAW;                     -- no labels, pure model signal
DESCRIBE "France" ALL LAYERS;              -- syntax + knowledge + output bands
DESCRIBE "France" KNOWLEDGE;               -- L14-27 only (for Gemma 3 4B)
DESCRIBE "France" SYNTAX;                  -- L0-13 only
DESCRIBE "France" OUTPUT;                  -- L28-33 only
DESCRIBE "Mozart" AT LAYER 26;
DESCRIBE "France" RELATIONS ONLY;          -- only labelled edges

-- WALK — feature scan for a prompt (no attention)
WALK "France" TOP 10;
WALK "The capital of France is" TOP 10 LAYERS 24-33;

-- SELECT — edge-table queries
SELECT entity, relation, target, confidence
  FROM EDGES
  WHERE entity = "France"
  ORDER BY confidence DESC
  LIMIT 10;

SELECT entity, target
  FROM EDGES
  WHERE relation = "capital" AND confidence > 0.5;

SELECT *
  FROM EDGES
  WHERE layer = 27 AND feature = 9515;

SELECT entity, target, distance
  FROM EDGES
  NEAREST TO "Mozart"
  AT LAYER 26
  LIMIT 20;

-- EXPLAIN WALK — per-layer feature trace (no attention)
EXPLAIN WALK "The capital of France is";
EXPLAIN WALK "The capital of France is" LAYERS 24-33 TOP 3 VERBOSE;
```

## Inference (requires inference or all extraction level)

```sql
INFER "The capital of France is" TOP 5;
INFER "The capital of France is" TOP 5 COMPARE;       -- dense vs walk side by side
EXPLAIN INFER "The capital of France is" TOP 5;
EXPLAIN INFER "The capital of France is" TOP 5 WITH ATTENTION;
```

## Mutation (auto-patch; base files never modified)

```sql
-- INSERT — always multi-layer constellation (8L × α=0.25 default; see insert-playbook.md)
INSERT INTO EDGES (entity, relation, target) VALUES ("John Coyle", "lives-in", "Colchester");

INSERT INTO EDGES (entity, relation, target)
  VALUES ("Atlantis", "capital-of", "Poseidon")
  AT LAYER 24                 -- centre the 8-layer band here
  CONFIDENCE 0.95             -- stored on inserted features (default 0.9)
  ALPHA 0.30;                 -- per-layer down scale (default 0.25; range 0.10-0.50)

-- DELETE
DELETE FROM EDGES WHERE entity = "John Coyle" AND relation = "lives-in";

-- UPDATE (entity scan)
UPDATE EDGES
  SET target = "London"
  WHERE entity = "John Coyle" AND relation = "lives-in";

-- UPDATE (fast path — (layer, feature) direct)
UPDATE EDGES
  SET target = "London", confidence = 0.95
  WHERE layer = 26 AND feature = 8821;

-- MERGE across vindexes
MERGE "medical.vindex"
  INTO "gemma3-4b.vindex"
  ON CONFLICT HIGHEST_CONFIDENCE;   -- KEEP_SOURCE | KEEP_TARGET | HIGHEST_CONFIDENCE
```

## Patches

```sql
BEGIN PATCH "medical.vlp";
INSERT INTO EDGES (entity, relation, target) VALUES ("aspirin", "treats", "headache");
INSERT INTO EDGES (entity, relation, target) VALUES ("aspirin", "side-effect", "bleeding");
SAVE PATCH;

APPLY PATCH "medical.vlp";
APPLY PATCH "company-facts.vlp";
SHOW PATCHES;
REMOVE PATCH "company-facts.vlp";

DIFF "base.vindex" "edited.vindex" INTO PATCH "changes.vlp";
```

## Trace (residual stream; requires inference+ level)

```sql
TRACE "The capital of France is";                           -- default, last token
TRACE "The capital of France is" FOR "Paris";               -- track a specific target
TRACE "The capital of France is" DECOMPOSE LAYERS 22-27;    -- attn vs FFN attribution
TRACE "The capital of France is" POSITIONS ALL SAVE "france.trace";

-- 🔴 Planned (not yet in the LQL surface):
-- TRACE ... DIFF <prompt_b>
-- TRACE ... FORMAT boundary|context TIER <n> WINDOW <n>
-- BOUNDARY OPEN <path>; BOUNDARY <path> AT <n>
```

## Introspection

```sql
SHOW RELATIONS;                            -- probe-confirmed relations
SHOW RELATIONS WITH EXAMPLES VERBOSE;
SHOW LAYERS;                               -- layer band summary
SHOW LAYERS RANGE 22-27;
SHOW FEATURES 26 LIMIT 20;
SHOW FEATURES 26 WHERE confidence > 0.5 LIMIT 20;
SHOW MODELS;                               -- available vindexes
SHOW PATCHES;
STATS;
STATS "gemma3-4b.vindex";
```

## WHERE operators

```
= != > < >= <= LIKE IN AND OR
WHERE entity LIKE "Fran%"
WHERE entity IN ("France", "Germany")
WHERE layer >= 20 AND layer <= 30
```

## Pipe

```sql
WALK "The capital of France is" TOP 5
  |> EXPLAIN WALK "The capital of France is";
```

## CLI idioms

```bash
# One-shot
larql lql 'USE "gemma3-4b.vindex"; DESCRIBE "France";'

# Batch via stdin
echo 'USE "gemma3-4b.vindex"; STATS;' | larql lql -

# Interactive
larql repl

# Non-LQL subcommands
larql extract-index google/gemma-3-4b-it -o gemma3-4b.vindex --level all --f16
larql convert gguf-to-vindex model.gguf -o model.vindex --f16
larql hf download chrishayuk/gemma-3-4b-it-vindex
larql hf publish ./my.vindex user/my-vindex
larql build . --stage prod
larql serve ./gemma3-4b.vindex --port 8080
larql verify ./gemma3-4b.vindex
```

## Implementation-status callouts (v0.4)

- ✅ Parser, REPL, USE, STATS, SHOW, SELECT, DESCRIBE, WALK, EXPLAIN WALK, INFER, EXPLAIN INFER, EXTRACT, INSERT, DELETE, UPDATE, MERGE, COMPILE INTO VINDEX, COMPILE INTO MODEL (safetensors), BEGIN/SAVE/APPLY/SHOW/REMOVE PATCH, auto-patch, TRACE (basic + FOR + DECOMPOSE + SAVE), WeightBackend (USE MODEL), Vindexfile, USE REMOTE, larql serve.
- 🔴 Planned: `FORMAT gguf`, `TRACE ... DIFF`, tiered SAVE formats, `BOUNDARY OPEN`, `DESCRIBE STREAM`, cross-model DIFF, gated KNN for MoE, residual-based DESCRIBE.
- 🟡 Known limitation: MXFP4 gate-KNN is noisy (use INFER for knowledge queries on GPT-OSS).
- ⚠️ `ON CONFLICT HIGHEST_CONFIDENCE` for down vectors currently collapses to `LAST_WINS` (parser-accepted, forward-compat; documented).
