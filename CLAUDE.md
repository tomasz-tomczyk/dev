# dev — Development Guide

## What This Is

A thin bash script (`dev`) that wraps mise for project lifecycle and adds git worktree management. Lives at `~/Server/side/dev/`, symlinked to `~/bin/dev`.

## Project Structure

```
dev/
├── dev              # Main script (~200 lines)
├── README.md        # User-facing docs
└── CLAUDE.md        # This file
```

## Architecture

- `dev up/down/setup` → delegates to `mise run up/down/setup`
- `dev wt create/remove/list` → direct worktree management
- `dev info` → shows mise config summary
- Each project's `mise.toml` defines tools, env vars, and tasks
- Worktree overrides stored in `mise.local.toml` (auto-generated, gitignored)

## Key Environment Variables

Set in each project's `mise.toml [env]` section:
- `PORT` — dev server port (worktrees get offset)
- `DB_NAME` — database name (worktrees get `_<name>` suffix)
- `DB_PORT`, `DB_USER`, `DB_HOST` — database connection
- `WT_COPY_DIRS` — space-separated dirs to copy for fast worktree setup
- `WT_SYMLINK_FILES` — space-separated files to symlink from main worktree

## Known Issues

- **mise Tera templating**: `{{.Names}}` Go template syntax conflicts with mise's Tera engine. Use `docker ps --filter "name=..."` instead.
- **`docker compose` vs `docker-compose`**: This machine uses the standalone `docker-compose`. Use hyphenated form in all tasks.
- **Interactive tasks**: Use `raw = true` (not `interactive = true`) for iex/node processes that need stdin.

## Shell Integration

`~/.zshrc` wraps `dev` as a shell function so `dev wt create` automatically cd's into the new worktree. The function calls `command dev` to invoke the actual binary and bypass the wrapper.

## Testing Changes

After modifying the script:
1. `dev info` in a Phoenix+Docker project and a bun project
2. `dev up` in a Phoenix project — verify mise delegation works
3. `dev wt create test && dev wt list && dev wt remove test` in a Phoenix project
4. Verify mise.local.toml generation in a worktree

## Conventions

- `_` prefix for internal helpers (`_db_exists`, `_db_drop`)
- `cmd_*` for top-level commands (`cmd_info`, `cmd_wt`)
- `wt_*` for worktree subcommands (`wt_create`, `wt_remove`)
- `log`, `warn`, `err`, `info` for colored output
