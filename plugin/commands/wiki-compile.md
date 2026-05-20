# Compile Knowledge Base Wiki

Compile all configured markdown source files into a topic-based wiki.

## Instructions

1. **Read configuration** from `.wiki-compiler.json` in the project root (or nearest parent directory). If not found, tell the user to run `/wiki-init` first.

2. **Validate configuration:**
   - `sources[]` must have at least one entry
   - `output` must be set
   - Source paths must exist

3. **Read schema** from `{output}/schema.md` if it exists. Use it to guide topic/concept classification and naming. If it doesn't exist (first run), it will be generated in Phase 3.7.

4. **Invoke the `wiki-compiler` skill** to run the compilation:
   - Pass `article_sections` from config to the skill (if present — legacy configs may not have this field)
   - Phase 1: Scan sources (**file-list diff against `processed_sources`** — v2.2+, not timestamp-only)
   - Phase 2: Classify and discover topics (respecting schema if present)
   - Phase 3: Compile topic articles using `article_sections` for structure (use parallel agents when possible)
   - Phase 3.5: Discover and compile concept articles (cross-cutting patterns)
   - Phase 3.7: Generate or update schema.md
   - Phase 4: Update INDEX.md (now includes concepts section)
   - Phase 5: Update state and log (**write `processed_sources` registry** — accumulates across runs)

5. **Show completion summary** with: source scan results (new/modified/deleted counts), topics created/updated, concepts discovered, source count, and schema changes.

## Arguments

- No arguments: incremental compilation via file-list diff against `processed_sources`
- `--full`: force recompile all topics; rebuilds `processed_sources` from scratch (use when upgrading from pre-v2.2 state files or when state is suspected corrupt)
- `--topic {slug}`: recompile only the specified topic
- `--dry-run`: show what would be compiled (new/modified/deleted sets + topic plan) without writing files. **Recommended after upgrading the plugin** to verify the file-list diff produces expected results before committing.

## Upgrade note (v2.2+)

If your `.compile-state.json` was produced by a pre-v2.2 version (no `processed_sources` key), the first run after upgrade will **bootstrap** the registry by reverse-parsing existing topic articles' `## Sources` sections. This is a one-time cost — no source files are re-processed, only catalogued.

If you see "new files" reported that you know were already ingested, run with `--dry-run` first to inspect, and consider `--full` if the bootstrap mis-attributed some sources.
