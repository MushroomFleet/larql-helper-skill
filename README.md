# larql-helper-skill

A [Claude Code](https://claude.com/claude-code) skill that teaches Claude how to operate [LARQL](https://github.com/MushroomFleet/larql) — a Rust toolchain that decompiles transformer LLM weights into a queryable mmap directory (a *vindex*) and exposes **LQL** (Lazarus Query Language), an SQL-shaped DSL for browsing, editing, patching, and recompiling model knowledge without fine-tuning.

Drop this skill into `~/.claude/skills/larql-helper/` and Claude will activate it automatically whenever you mention LARQL, LQL, vindexes, `.vlp` patches, or when the current working directory contains a LARQL checkout or vindex artifact.

## What the skill does

- **Plans LARQL workflows in natural language.** Ask "extract Gemma 3 2B and tell me everything it knows about France" or "insert four facts about Acme Corp and ship a GGUF" — the skill walks Claude through the right sequence of LQL statements, CLI invocations, and extraction-level prerequisites.
- **Writes correct LQL.** A copy-pasteable cheatsheet for every statement in the v0.4 spec, with the common gotchas (`AT LAYER` / `ALPHA` / `CONFIDENCE` knobs, `WITH INFERENCE` vs `WITH ALL`, auto-patch lifecycle, etc.).
- **Executes the training-free INSERT recipe correctly.** The 8-layer × α=0.25 constellation regime, with the residual-capture step, alpha sweep trade-offs, subtoken caveats, and neighbour-regression checks.
- **Routes around planned-not-shipped features.** Native `FORMAT gguf` output isn't implemented yet — the skill surfaces the standard `safetensors` → `llama.cpp/convert_hf_to_gguf.py` round-trip instead.
- **Flags the risks.** Selectivity is not guaranteed. MXFP4 browse is noisy. `HIGHEST_CONFIDENCE` collapses to `LAST_WINS` for down vectors. The risk register ships with the skill.

## Activation triggers

Claude auto-loads the skill when:

- You mention any of: `larql`, `LQL`, `Lazarus Query Language`, `vindex`, `.vlp`, `Vindexfile`, `EXTRACT MODEL`, `DESCRIBE`/`WALK`/`INFER`, `INSERT INTO EDGES`, `COMPILE INTO VINDEX`, `COMPILE INTO MODEL`, `larql serve`, `constellation INSERT`, `training-free insert`, editing LLM weights without fine-tuning.
- The working directory contains a `larql/` Rust workspace, a `*.vindex/` directory, a `*.vlp` patch file, or a `Vindexfile`.

## Install

```bash
# Clone into your user-level skills directory
git clone https://github.com/MushroomFleet/larql-helper-skill.git ~/.claude/skills/larql-helper

# Or as a submodule of a dotfiles repo
cd ~/.claude/skills && git submodule add https://github.com/MushroomFleet/larql-helper-skill larql-helper
```

No build step. No configuration. Claude discovers the skill on the next session and triggers it contextually.

## Structure

```
larql-helper/
├── SKILL.md                      — frontmatter + trigger + workflow + file index
└── references/
    ├── grounding.md              — architecture, vindex layout, invariants, benchmarks
    ├── lql-cheatsheet.md         — every LQL statement, copy-pasteable
    ├── insert-playbook.md        — the training-free INSERT recipe + caveats
    ├── gguf-round-trip.md        — safetensors → GGUF via llama.cpp (until FORMAT gguf ships)
    └── risk-register.md          — gaps, gotchas, version pinning, platform notes
```

Progressive disclosure: `SKILL.md` loads up front; references are pulled in on demand based on the task. Asking about INSERT loads the playbook; asking about GGUF loads the round-trip doc; etc.

## Example session

```
> I want to teach Gemma 3 2B that Acme Corp is headquartered in London,
> then ship it as a GGUF that Ollama can load.

[skill activates]

I'll walk you through the pipeline. This needs:
  1. EXTRACT at --level all (we'll be compiling back to a model)
  2. INSERT the fact via the validated 8-layer × α=0.25 constellation
  3. COMPILE CURRENT INTO VINDEX (bake the patch)
  4. COMPILE CURRENT INTO MODEL FORMAT safetensors
  5. llama.cpp/convert_hf_to_gguf.py (because native FORMAT gguf is planned,
     not yet shipped)

Commands:

  larql extract-index google/gemma-3-2b-it -o gemma3-2b.vindex --level all --f16

  larql lql '
  USE "gemma3-2b.vindex";
  BEGIN PATCH "acme.vlp";
  INSERT INTO EDGES (entity, relation, target)
    VALUES ("Acme Corporation", "headquarters", "London");
  SAVE PATCH;
  APPLY PATCH "acme.vlp";
  INFER "Acme Corporation is headquartered in" TOP 3;
  COMPILE CURRENT INTO VINDEX "gemma3-2b-acme.vindex";
  '

  larql lql '
  USE "gemma3-2b-acme.vindex";
  COMPILE CURRENT INTO MODEL "gemma3-2b-acme-hf/" FORMAT safetensors;
  '

  python3 llama.cpp/convert_hf_to_gguf.py ./gemma3-2b-acme-hf \
    --outfile ./gemma3-2b-acme-f16.gguf --outtype f16
```

## Related repositories

- [**MushroomFleet/larql**](https://github.com/MushroomFleet/larql) — the forked LARQL codebase this skill targets. Apache-2.0.
- [**MushroomFleet/larql-study**](https://github.com/MushroomFleet/larql-study) — the research bundle this skill was built from. Contains a seven-artifact audit of LARQL: identity, capability surface, operator manual, an end-to-end Gemma 2B INSERT + GGUF worked example, grounding references, and a final report stress-testing the "model IS the database" claim. `05-grounding-references.md` is the direct upstream of this skill.

## Compatibility

- **Claude Code** CLI, ≥ current release.
- **LARQL** v0.1.x targeting **LQL v0.4** and **vindex format v2**. If LARQL bumps its spec version, the skill's references may drift — check the implementation-status table in `larql/docs/lql-spec.md` §8.4.

## What this skill is NOT

- Not a plugin that ships LARQL itself — you still need a built `larql` binary on `PATH` (see the [LARQL README](https://github.com/MushroomFleet/larql) for build instructions).
- Not a fine-tuning helper. LARQL is explicitly training-free; ask Claude about LoRA/PEFT/DPO separately.
- Not a RAG builder. LARQL stores knowledge *in* the weights, not in an external vector DB.

## License

Apache-2.0, matching LARQL's license. Documentation excerpts quoted from the LARQL tree remain under the terms of that tree's `LICENSE`.

## 📚 Citation

### Academic Citation

If you use this codebase in your research or project, please cite:

```bibtex
@software{larql_helper_skill,
  title = {larql-helper-skill: a Claude Code skill for operating LARQL},
  author = {[Drift Johnson]},
  year = {2025},
  url = {https://github.com/MushroomFleet/larql-helper-skill},
  version = {1.0.0}
}
```

### Donate:


[![Ko-Fi](https://cdn.ko-fi.com/cdn/kofi3.png?v=3)](https://ko-fi.com/driftjohnson)
