# worktree-skill-plugin

A Claude Code skill that sets up git worktree management with isolated Postgres databases, Redis instances, and ports for parallel development.

## Install

```bash
npx skills add flrngel/worktree-skill-plugin
```

## Usage

In any git project, run the skill in Claude Code:

```
/worktree-setup
```

Claude will analyze your project (env vars, database, Redis, package manager) and generate customized worktree management scripts. After setup, use standalone git commands — no skill required.

## Commands

```bash
git wt create feat/my-feature        # Create worktree from current HEAD
git wt create feat/my-feature main   # Create worktree from a specific branch
git wt list                          # List all managed worktrees
git wt delete feat/my-feature        # Delete worktree and clean up all resources
```

## What It Does

When you run `/worktree-setup`, Claude:

1. **Analyzes your project** — detects package manager, database, Redis, env vars, and ports
2. **Generates `.worktree/config.json`** — stores detected settings and tracks worktrees
3. **Generates `.worktree/bin/` scripts** — fully customized bash scripts for your project
4. **Configures local git** — adds `git wt` alias and excludes `.worktree` from tracking

### What `git wt create` does

- Creates a git worktree in `.worktree/<branch-name>`
- Clones your Postgres database (via `createdb -T` or `pg_dump | psql` fallback)
- Allocates a unique Redis DB number
- Allocates a unique port
- Copies `.env` and patches DATABASE_URL, REDIS_URL, and PORT
- Runs your install command (`npm install`, `yarn install`, etc.)

### What `git wt delete` does

- Drops the cloned Postgres database
- Flushes the allocated Redis DB
- Removes the git worktree
- Updates config

## Design

- **Zero git changes** — `.worktree` is excluded via `.git/info/exclude` (not `.gitignore`), alias is local git config
- **Standalone** — after setup, scripts work without Claude Code or the skill installed
- **Project-aware** — Claude adapts scripts to your exact env var format, database type, and package manager
- **Safe** — branch names are sanitized, ports are checked for availability, database operations have fallbacks

## Requirements

- `git` (with worktree support)
- `jq` (required for config management)
- `psql` / `createdb` / `dropdb` (optional, for Postgres support)
- `redis-cli` (optional, for Redis support)

## Supported Project Types

| Ecosystem | Detected By | Init Command |
|-----------|-------------|-------------|
| Node.js | `package.json`, lockfiles | `npm/yarn/pnpm/bun install` |
| Python | `requirements.txt`, `Pipfile`, `pyproject.toml` | `pip install` / `pipenv install` |
| Ruby | `Gemfile` | `bundle install` |
| Go | `go.mod` | `go mod download` |
| Rust | `Cargo.toml` | `cargo build` |
| PHP | `composer.json` | `composer install` |

## License

MIT
