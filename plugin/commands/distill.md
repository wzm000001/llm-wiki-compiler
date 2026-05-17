---
description: Distill the current Claude Code conversation into a wiki source and ingest it into matching topics. Works from any directory.
---

# Distill Conversation into Wiki Source

Synthesize the **valuable insights, decisions, and lessons** from the current Claude Code conversation into a structured markdown source file in the user's wiki, then ingest into the relevant topic(s). **Works from any cwd** — does not require launching Claude Code inside the wiki directory.

## Vault Resolution

Locate the target wiki via env var `$LLM_WIKI_GLOBAL_DIR`. All paths below resolve against `$VAULT = $LLM_WIKI_GLOBAL_DIR`.

If `$LLM_WIKI_GLOBAL_DIR` is unset, tell the user:
> "`$LLM_WIKI_GLOBAL_DIR` 未设。请先 `export LLM_WIKI_GLOBAL_DIR=/path/to/your/wiki` 再调用 `/distill`。"

## Arguments

- `$ARGUMENTS` (optional): one-sentence topic description. If empty, infer from conversation context.

## Workflow (strict order)

### Step 1 · Value Gate (critical)

Review the conversation. Ask:
- Any **new understanding** not already in the wiki?
- Any **new decisions** worth recording (with rationale)?
- Any **new gotchas/lessons**?
- Any **cross-cutting insights** spanning multiple topics?

If all 4 are "no" (pure execution / debugging without new insight / repeated operations), **stop and tell user**:
> "这段对话偏执行，没有值得沉淀的新认知。要继续吗？（你也可以告诉我具体想沉淀的角度）"

Wait for user decision.

### Step 2 · Read Vault Schema

Read these files to understand current wiki structure:

```
$VAULT/00-Wiki/INDEX.md           # topic catalog with coverage
$VAULT/00-Wiki/schema.md          # topic definitions, naming, decay rules
$VAULT/CLAUDE.md                  # vault business rules (content scope, etc.)
```

**Pay attention to vault CLAUDE.md** — especially content scope rules (some vaults only accept certain topics).

### Step 3 · Propose Scope (multi-topic split decision)

Identify how many topics this conversation touches. Then:

**If ≤3 topics touched** → propose ONE source file referenced by all touched topics. Show user (≤3 lines):
- Proposed topic for the source file
- List of topics that will reference it
- Rough section outline

**If 4+ topics touched and they cluster into 2+ unrelated groups** → propose SPLIT:
> "这次对话覆盖 N 个 topic，离散度高。建议拆成 K 个 source 文件：
> 1. {主题 A} → topic X, Y
> 2. {主题 B} → topic Z
> 还是合并成一个？"

Wait for "合并 / 拆成 K 个 / 改成 xxx".

### Step 4 · Write Source File(s)

For each source file, path is:

```
$VAULT/01-Sources/对话沉淀/{YYYY-MM-DD}-{slug}.md
```

If the directory doesn't exist, `mkdir -p` it.

Slug rules: lowercase-kebab-case or short Chinese name, ≤80 chars, derived from topic.

Frontmatter:

```yaml
---
tags: [对话沉淀, ...other relevant tags]
date: YYYY-MM-DD
source: "Claude Code 对话沉淀"
session_topic: "{主题}"
session_cwd: "{cwd when distill was invoked}"
duration_estimate: "{短/中/长}"
---
```

Body 7-section structure (aligned with vault topic conventions):

```markdown
# {主题}

## 起因
Why this conversation started · user's original problem · what was the goal

## 关键决策
Chronological decision points + why each was chosen + alternatives considered
(One paragraph per decision, 3-5 max)

## 工具与方法
Tools/commands/workflows actually used (comparison tables when applicable)

## 踩坑与教训
Where things broke · how found · how fixed · how to avoid in future
(Highest value section — write generously here)

## 最终方案 / 当前状态
Conclusive facts at conversation end (paste real paths / configs / data)

## 待解决问题
Open items at end · what to pick up next time

## 价值点（给未来的我）
Why this conversation matters · which parts will be reused · talking points for explaining to others
```

Writing requirements:
- **Denoise**: strip typos / polite "OK"s / raw tool outputs / repeated confirmations
- **Wikilinks**: use `[[wikilinks]]` for all tools/people/projects/concepts (per vault CLAUDE.md naming conventions)
- **Concrete not abstract**: paste real paths, real numbers, real commands; avoid vague phrases like "we did optimization"
- **Length**: 100-400 lines sweet spot; too short = no value, too long = dump
- **AI relevance**: if conversation topic is not strongly AI-related (e.g. pure debugging of unrelated project), first ask user "这条不是强 AI 主题，确定要进 wiki 吗？" — respect the vault's content scope defined in its CLAUDE.md

### Step 5 · Preview to User

Show user:
- Full path(s) of file(s) written
- 3-line summary (one sentence per "起因/关键决策/价值点")
- Total line count
- Ask "Ingest? Or revise?"

### Step 6 · Ingest

After user confirms, execute the standard ingest flow:

1. Read `$VAULT/00-Wiki/schema.md` to identify the best-fit topic(s)
2. For each touched topic, update `$VAULT/00-Wiki/topics/{slug}.md`:
   - Add wikilink to source in Sources section (path: `[[../../01-Sources/对话沉淀/{file}]]`)
   - Append Timeline entry if applicable
   - Bump `source_count` in frontmatter +1
   - Update `last_compiled` to today
3. Update `$VAULT/00-Wiki/INDEX.md`: bump source count for affected topic rows and the global total
4. Append `$VAULT/00-Wiki/LOG.md`:
   ```
   ## [YYYY-MM-DD] distill | {主题} → {topic1}[, {topic2}]
   - 来源：Claude Code 对话（{duration}，from cwd: {session_cwd}）
   - 文件：[[../01-Sources/对话沉淀/{file}]]
   - 关键 takeaway：{one-line core insight}
   ```
5. Update `$VAULT/00-Wiki/.compile-state.json`:
   - `last_ingest` = today
   - Add/update `last_compiled_at` ISO timestamp (helps hook stale detection avoid overcounting)

### Step 7 · Done

Tell user:
- Source file path(s)
- Topics updated
- Whether cross-references were triggered (multi-topic hits)
- Remind: "本次 distill 从 `{session_cwd}` 触发，文件已写入 vault"

## Usage Examples

```
/distill                                       # Auto-infer topic from any cwd
/distill wiki 初始化全流程                       # Explicit topic
/distill hook stale 算法虚报问题                 # Focused on one aspect
```

## Don'ts

- ❌ Don't dump raw conversation text — synthesize, don't transcribe
- ❌ Don't skip user confirmation in Step 1 / Step 3
- ❌ Don't forget to update `.compile-state.json` and LOG (state drift will break hook stale detection)
- ❌ Don't force a distill when there's no real new insight (better to skip than dilute)
- ❌ Don't refuse just because cwd isn't in vault — this command is explicitly designed to be cross-cwd
- ❌ Don't assume conversation is AI-relevant — non-AI topics need explicit user confirmation

## Quality Self-Check (before finishing)

- [ ] Will I (or future me) understand this 6 months later?
- [ ] Do key decisions explain "why" not just "what"?
- [ ] Does the gotchas section include specific error messages / fix commands?
- [ ] Are all important entities wrapped in `[[wikilinks]]`?
- [ ] Are frontmatter `source_count`, Sources list, and cross-refs all aligned?
- [ ] Do conversations comply with the vault's CLAUDE.md content scope?
