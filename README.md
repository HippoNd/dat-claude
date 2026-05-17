# datclaude

Version-controlled mirror of selected files from `~/.claude/`. Files live here as the
canonical copy; `~/.claude/` holds symlinks pointing into this directory.

## Layout

Every file under `datclaude/` (excluding `bin/`, `.git/`, `*.md`, `.gitignore`) mirrors
its position under `~/.claude/`. For example, `datclaude/plugins/blocklist.json` is the
canonical file for `~/.claude/plugins/blocklist.json`.

## Commands

- `bin/dc-track <relative-path>` — start tracking. Moves `~/.claude/<path>` here and
  replaces it with a symlink. Refuses to run if a `claude` process is alive (override
  with `DC_FORCE=1` if you know the file isn't being written).
- `bin/dc-untrack <relative-path>` — reverse it. Run `git rm` yourself afterwards.
- `bin/dc-sync` — recreate every symlink from the current datclaude tree. Idempotent;
  fails loudly if `~/.claude/<path>` already holds a real file or wrong symlink. Use
  this on a fresh machine after `git clone`.

## Fresh machine setup

```
git clone <url> ~/datclaude
~/datclaude/bin/dc-sync
```

If `~/.claude/` already has its own real `settings.json` (etc.) from Claude's first run,
`dc-sync` refuses to overwrite. Reconcile manually: diff the two copies, decide which
wins, delete the loser, re-run.

## Templated files

Some files cannot be safely symlinked because (a) Claude Code rewrites them via
atomic-rename on plugin install/uninstall (clobbering the symlink — see below), and
(b) they embed absolute paths under `/Users/<name>/.claude/...` that differ per
machine.

These files are stored in datclaude as templates using `$HOME` as a literal
placeholder. `dc-sync` materializes them into `~/.claude/...` as real files by
substituting `$HOME` with the current home directory. The list lives in
`bin/dc-sync` under `TEMPLATE_FILES`:

- `plugins/installed_plugins.json`
- `plugins/known_marketplaces.json`

Capturing local changes back: when Claude adds/removes a plugin or marketplace,
the real file at `~/.claude/plugins/<file>.json` diverges from the template.
Copy it back manually, replace your `/Users/<you>` prefix with literal `$HOME`,
and commit. (No automated `dc-capture` yet.)

## Known failure modes

### Atomic-rewrite clobber

Programs (including Claude Code on plugin install/uninstall) often write files via
"write to tmp file, `rename(2)` over the original." This replaces the symlink at the
target path with a regular file. The datclaude copy goes stale and out of sync.

For the plugin metadata files this is now handled via the template mechanism above.
For other tracked files: run `ls -la ~/.claude/<path>` after a suspect operation.
If it's no longer a symlink, that's the clobber.

Recovery: decide which copy is current (likely the now-real file in `~/.claude/`),
move it back into `datclaude/`, run `bin/dc-sync` to re-establish the symlink.

### Permissions drift on clone

Plugin JSONs were originally `0600`. Git restores files as `0644` on clone. `dc-sync`
runs `chmod 600` on `plugins/*.json` before linking. If you add other sensitive files,
extend the chmod logic.

## Intentionally not tracked

- `~/.claude.json` (home dir) — global state, may contain auth.
- `~/.claude/.claude/settings.local.json` — workspace-local override created when this
  dir is opened as a Claude Code working directory.
- All runtime/cache: `projects/`, `todos/`, `tasks/`, `sessions/`, `session-env/`,
  `file-history/`, `ide/`, `debug/`, `cache/`, `paste-cache/`, `shell-snapshots/`,
  `statsig/`, `telemetry/`, `transcripts/`, `plans/`, `backups/`.
- Runtime files: `history.jsonl`, `mcp-needs-auth-cache.json`, `policy-limits.json`,
  `stats-cache.json`, `.last-cleanup`.
- `plugins/cache/` and `plugins/marketplaces/` — externally fetched.
- `skills/find-skills` — symlinked to `~/.agents/skills/find-skills`, managed elsewhere.
