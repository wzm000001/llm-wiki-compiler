# Lint Knowledge Base Wiki

Run health checks on the compiled wiki to find issues.

## Instructions

1. **Read configuration** from `.wiki-compiler.json`. If not found, tell the user to run `/wiki-init` first.

2. **Read the wiki state:**
   - Read `{output}/INDEX.md` for current topics
   - Read `{output}/schema.md` for expected structure (if exists)
   - Read `{output}/.compile-state.json` for last compilation state

3. **Run these checks:**

### Check 1: Stale Articles
Compare source file modification dates against `.compile-state.json`. Flag topics whose sources have changed since last compile.

### Check 2: Orphan Pages
Check each wiki article's Sources section. Flag articles where:
- Source files no longer exist (were deleted or moved)
- Article has 0 sources listed

### Check 3: Missing Cross-References
For each pair of topics, count shared sources. If two topics share 3+ sources but neither references the other, suggest a cross-reference.

### Check 4: Low Coverage Sections
Scan all articles for `[coverage: low]` tags. List them as improvement candidates -- these sections should either be expanded with more sources or flagged as known gaps.

### Check 5: Contradictions
Compare key facts across articles. Look for:
- Different dates for the same event in different articles
- Conflicting metrics (e.g., "D1 is 17.5%" in one article, "D1 is 13.3%" in another)
- Decisions described differently across topics

### Check 6: Schema Drift
If `schema.md` exists:
- Topics in `topics/` directory not listed in schema.md
- Topics listed in schema.md that don't have a corresponding article
- Article sections that don't match the schema's Article Structure

### Check 7: Orphan Source Files (NEW in v2.2)

Detects files that exist in source directories but were never ingested into any topic. This is the canonical "10 bookmarks landed in symlink but no compile picked them up" case — pre-v2.2 incremental compile could silently skip files whose mtime was older than `last_compiled`.

**Algorithm:**

1. Read `.compile-state.json` → `processed_sources` (path → `{topics, last_processed, content_hash}`)
2. For each entry in `sources[]`, list all `.md` files (Glob, applying `exclude`)
3. Build `current_files` = set of all relative paths
4. Build `known_files` = keys of `processed_sources`
5. **Orphan sources** = `current_files - known_files` — file exists on disk but not in registry
6. **Broken back-references** = entries in `processed_sources` where the file's claimed `topics[]` don't all reference the file in their `## Sources` section (state thinks file is processed but topic article disagrees)
7. **Dead registry entries** = `known_files - current_files` — file referenced in state but no longer on disk

For each **orphan source**, report:
- File path
- mtime (when it appeared)
- Suggested topic(s) based on directory structure, filename, and frontmatter tags
- Suggested fix: `/wiki-ingest {path}` or batch `/wiki-compile`

For each **broken back-reference**, report:
- File path
- Topic that's missing the reference
- Suggested fix: rerun `/wiki-compile --topic {slug}` to rebuild

For each **dead registry entry**, ask user: was this file moved (need to update path), deleted intentionally (purge from state), or accidentally removed (restore)? Do not auto-modify.

### Check 8: Source/Topic Round-Trip Consistency (NEW in v2.2)

For each topic article in `{output}/topics/*.md`:
1. Parse its `## Sources` section, extract all `[[../../...]]` wikilink targets
2. For each target, look up in `processed_sources`
3. If target exists in registry but topic's slug NOT in `processed_sources[target].topics[]`: registry is missing a cross-reference (state out of sync with article)
4. If target NOT in registry: article references a source the registry doesn't know — also out of sync

Report any divergence. Suggest `/wiki-compile --full` to rebuild state from scratch when divergence is large (>10 entries).

4. **Output a summary:**

```
Wiki Lint: "{name}"
──────────────────────────
Stale:           {N} topics (sources changed since last compile)
Orphans:         {N} articles with missing sources
Cross-refs:      {N} missing links suggested
Low coverage:    {N} sections across {N} topics
Contradictions:  {N} found
Schema drift:    {N} mismatches
Orphan sources:  {N} files in source dirs not in any topic  (NEW)
Broken back-refs:{N} state-vs-article mismatches            (NEW)
Dead state:      {N} registry entries with no on-disk file  (NEW)

{Details for each finding, grouped by check}
```

5. **Suggest fixes:**
   - Stale topics: "Run `/wiki-compile` to refresh"
   - Orphans: "Source was deleted -- recompile to remove stale references"
   - Cross-refs: "Consider adding a reference to [[topic-b]] in topic-a's Summary"
   - Contradictions: "Check {source1} vs {source2} for the correct value"
   - Schema drift: "Add {topic} to schema.md" or "Remove {topic} from schema.md"
   - Orphan sources: "Run `/wiki-ingest {path}` for each, or `/wiki-compile` to batch-process"
   - Broken back-refs: "Run `/wiki-compile --topic {slug}` to rebuild that topic's article"
   - Dead state: "Confirm with user; never auto-purge state entries"

6. **Log the lint run** by appending to `{output}/log.md`:
```markdown
### {date} — Lint
- Stale: {N}, Orphans: {N}, Cross-refs: {N}, Low: {N}, Contradictions: {N}, Drift: {N}, OrphanSrc: {N}, BrokenRefs: {N}, DeadState: {N}
```
