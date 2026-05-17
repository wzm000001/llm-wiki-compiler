# Fetch Bookmarks from an External Source

Pull bookmarks from an external service (X/Twitter today; Readwise, Pocket, GitHub stars planned) into a local directory that `/wiki-compile` can consume. Optional — only needed when you want wiki topics synthesized from external sources, not just files already on disk.

The command itself is a thin dispatcher. Each source has its own adapter file that handles dependency checks, auth, sync, and wiring.

## Instructions

### Step 1: Parse arguments

Accept one of these forms:

- `/fetch-bookmarks <source>` — sync once (default).
- `/fetch-bookmarks schedule <source>` — install a daily auto-sync job.
- `/fetch-bookmarks schedule <source> --uninstall` — remove the auto-sync job.
- `/fetch-bookmarks` (no args) — list available adapters and usage; then stop.

When listing adapters, glob `${CLAUDE_PLUGIN_ROOT}/skills/wiki-compiler/adapters/*.md`:
```
Usage:
  /fetch-bookmarks <source>                          Sync once
  /fetch-bookmarks schedule <source>                 Install daily auto-sync
  /fetch-bookmarks schedule <source> --uninstall     Remove auto-sync

Available sources:
- x — X/Twitter bookmarks (via Field Theory CLI)

More adapters planned: readwise, pocket, github-stars.
```

### Step 2: Locate the adapter

- Resolve the adapter file at `${CLAUDE_PLUGIN_ROOT}/skills/wiki-compiler/adapters/{source}.md`.
- If the file does not exist, print:
  ```
  No adapter for "{source}" yet. Available: {list}.
  Want to contribute one? See adapters/x.md for the contract.
  ```
  Then stop.

### Step 3a: One-off sync (default)

- Read the adapter file in full.
- Follow its numbered steps in order. Each adapter is self-contained and handles its own preflight, consent prompts, sync, and config updates.
- Surface the adapter's step-by-step progress to the user as you go. Do not retry failing steps silently — if a step fails, stop and show the error.
- After successful completion, print:
  ```
  Bookmarks fetched. Run /wiki-compile to synthesize them into topic articles.
  ```
  Do **not** auto-invoke `/wiki-compile`.

### Step 3b: Install auto-sync (`schedule <source>`)

- Read the adapter file and look for its **`## Scheduling`** section. It must document:
  - `sync_command` — the shell command to run on a schedule (e.g., `ft sync && ft md` for `x`)
  - `default_cadence` — recommended schedule (e.g., "daily 03:00 local")
  - `log_path` — where the sync command writes stdout/stderr
- If the adapter has no `## Scheduling` section, tell the user this adapter doesn't support auto-sync yet and stop.
- **macOS:** Generate a launchd plist at `~/Library/LaunchAgents/dev.llm-wiki-compiler.{source}-sync.plist` using the adapter's `sync_command` and `default_cadence`. Label: `dev.llm-wiki-compiler.{source}-sync`. Include `StandardOutPath` and `StandardErrorPath` pointing at `log_path`. Use `/bin/bash -lc "<sync_command>"` so the login shell loads PATH. Set `RunAtLoad` to `false`.
- Run `launchctl unload ~/Library/LaunchAgents/dev.llm-wiki-compiler.{source}-sync.plist 2>/dev/null` first (idempotent), then `launchctl load ~/Library/LaunchAgents/dev.llm-wiki-compiler.{source}-sync.plist`.
- Verify with `launchctl list | grep dev.llm-wiki-compiler.{source}-sync`.
- Print: next scheduled run time, log path, and the uninstall command. Example:
  ```
  ✓ Auto-sync installed. Next run: tomorrow 03:00 local.
  Log: ~/.fieldtheory/autosync.log
  Uninstall: /fetch-bookmarks schedule x --uninstall
  ```
- **Linux / WSL:** Don't auto-install. Print a crontab snippet the user can paste:
  ```
  # Add this with `crontab -e`:
  0 3 * * * /bin/bash -lc '<sync_command> >> <log_path> 2>&1'
  ```
- **Windows:** Print a Task Scheduler hint; don't auto-install.

### Step 3c: Uninstall auto-sync (`schedule <source> --uninstall`)

- **macOS:** Run `launchctl unload ~/Library/LaunchAgents/dev.llm-wiki-compiler.{source}-sync.plist 2>/dev/null`, then delete the plist file.
- Print confirmation.
- If the plist doesn't exist, say so and stop without error.

## Arguments

- `<source>` — slug matching an adapter file (currently: `x`).
- `schedule <source>` — install auto-sync for this source.
- `schedule <source> --uninstall` — remove auto-sync for this source.
- No arguments: prints help listing available adapters.

## Adapter contract (for contributors)

Each adapter is a self-contained markdown file that the dispatcher follows. Adapters must:

1. Preflight runtime and CLI dependencies. Abort with a friendly install link if a prerequisite is missing — never auto-install a runtime.
2. Get explicit user consent before installing any tool (`npm install`, `pip install`, etc.). Show the project URL and license.
3. Run the third-party sync command and surface its output verbatim so the user can respond to any prompts it raises.
4. Ensure the source's output ends up as markdown at a stable path.
5. Read `.wiki-compiler.json` from the project root. If that path is not already present in `sources[]`, ask the user to confirm adding it, then write the update preserving all other fields.
6. Print a "run `/wiki-compile`" suggestion. Never auto-invoke compile.
7. **Optional: `## Scheduling` section** documenting `sync_command`, `default_cadence`, and `log_path` so `/fetch-bookmarks schedule <source>` can wire launchd/cron. Adapters without this section don't support auto-sync.

See `plugin/skills/wiki-compiler/adapters/x.md` for the reference implementation.
