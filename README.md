# dev — Project Launcher & Worktree Manager

A thin wrapper around [mise](https://mise.jdx.dev) for consistent project startup and git worktree management across diverse codebases.

## How It Works

Each project defines a `mise.toml` with:
- `[tools]` — language versions (replaces `.tool-versions` / asdf)
- `[env]` — environment variables (replaces `.envrc` / direnv)
- `[tasks]` — lifecycle commands (up, down, deps, db:setup)

The `dev` script adds worktree management on top: creating isolated worktrees with copied dependencies, cloned databases, and auto-generated `mise.local.toml` overrides.

## Install

```bash
# 1. Install mise
brew install mise
echo 'eval "$(mise activate zsh)"' >> ~/.zshrc
source ~/.zshrc

# 2. Install dev
git clone <repo-url> ~/path/to/dev
ln -sf ~/path/to/dev/dev ~/bin/dev
```

**3. Add the shell wrapper to `~/.zshrc`** so `dev wt create` automatically cd's into the new worktree:

```bash
# dev wrapper — transparently cd into worktree after `dev wt create`
dev() {
  if [[ "$1" == "wt" && "$2" == "create" ]]; then
    local name="$3"
    local project_root
    project_root=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
    command dev "$@"
    local wt_dir="${project_root}/.worktrees/${name}"
    if [[ -d "$wt_dir" ]]; then
      cd "$wt_dir"
    fi
  else
    command dev "$@"
  fi
}
```

## Quick Start

```bash
cd ~/my-project
dev up     # Start everything
dev down   # Stop services
```

## Worktree Management

After `dev wt create`, your shell is automatically cd'd into the new worktree — no manual navigation needed.

```bash
dev wt create my-feature   # Create worktree, copy deps, clone DB — lands you inside it
dev up                      # Runs with isolated DB (via mise.local.toml)
dev wt list                 # Show all worktrees
dev wt remove my-feature   # Clean up worktree + drop DB
```

## Commands

| Command | Description |
|---------|-------------|
| `dev up` | Delegates to `mise run up` (with worktree context) |
| `dev down` | Delegates to `mise run down` |
| `dev setup` | Delegates to `mise run setup` |
| `dev init` | Initialise a new project (generates `mise.toml`) |
| `dev info` | Show project info (tools, tasks, env) |
| `dev wt create NAME` | Create git worktree with full setup |
| `dev wt remove NAME` | Remove worktree and drop its database |
| `dev wt list` | List all worktrees with DB names |

## Adding a New Project

Run `dev init` to generate a `mise.toml` interactively, or write one manually:

```toml
[tools]
elixir = "1.19.1-otp-28"
erlang = "28.1"

[env]
PORT = "4000"
DB_NAME = "myapp_dev"

# Worktree config (read by dev wt commands)
WT_COPY_DIRS = "deps _build"
WT_SYMLINK_FILES = ".env.local"

[tasks.deps]
run = "mix deps.get"

[tasks."db:setup"]
run = ["mix ecto.create 2>/dev/null || true", "mix ecto.migrate"]

[tasks.up]
depends = ["deps", "db:setup"]
run = "iex -S mix phx.server"
raw = true

[tasks.down]
run = "echo 'nothing to stop'"
```

That's it. `dev up` and `dev wt create` just work.

## Design Opinions

- **mise as the foundation**: One tool for versions, env, and tasks. No custom config format.
- **`mise.toml` is the source of truth**: No auto-detection magic. Each project declares what it needs.
- **`dev` only adds worktrees**: The script is ~200 lines. All project logic lives in mise.toml.
- **`mise.local.toml` for worktree overrides**: Auto-generated, gitignored. Each worktree gets its own DB.
- **Template DB cloning**: Copies from the main DB instead of running full setup — much faster for Phoenix projects with seeds.
- **Dep copying**: Copy compiled deps from main checkout instead of rebuilding from scratch.
