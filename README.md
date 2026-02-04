# worktree-skill-plugin

A Claude Code skill that sets up git worktree management with isolated databases, caches, and ports for parallel development.

## Install

```bash
npx skills add flrngel/worktree-skill-plugin
```

## Usage

In any git project, run the skill in Claude Code:

```
/worktree-setup
```

Claude analyzes your project — env vars, database type, cache, package manager, available CLI tools — and generates customized worktree management scripts tailored to your stack. After setup, use standalone git commands with no dependency on Claude Code.

## Commands

```bash
git wt create feat/my-feature        # Create worktree from current HEAD
git wt create feat/my-feature main   # Create worktree from a specific branch
git wt list                          # List all managed worktrees
git wt delete feat/my-feature        # Delete worktree and clean up all resources
```

## What It Does

When you run `/worktree-setup`, Claude:

1. **Analyzes your project** — detects language/framework, package manager, database, cache, env var patterns, ports, Docker Compose services, and available CLI tools
2. **Generates `.worktree/config.json`** — stores detected settings and tracks active worktrees
3. **Generates `.worktree/bin/` scripts** — standalone bash scripts customized to your project's stack and tools
4. **Configures local git** — adds `git wt` alias and excludes `.worktree` from tracking

### What `git wt create` does

- Creates a git worktree in `.worktree/<branch-name>`
- Clones your database (adapts to Postgres, MySQL, or whatever you use)
- Allocates an isolated cache namespace (e.g., Redis DB number)
- Allocates a unique port
- Copies env files and patches connection strings, ports, and other per-instance values
- Runs your dependency install command

### What `git wt delete` does

- Drops the cloned database
- Cleans up the allocated cache namespace
- Removes the git worktree
- Updates config

## Design

- **Zero git changes** — `.worktree` is excluded via `.git/info/exclude` (not `.gitignore`), alias is local git config only
- **Standalone** — after setup, scripts work without Claude Code or the skill installed
- **Adaptive** — Claude picks the right tools based on what's available on your machine (e.g., `jq` vs `python3` for JSON, `createdb` vs `psql -c` for Postgres)
- **Resilient** — database/cache operations warn on failure but don't block worktree creation; missing tools are detected and skipped gracefully
- **Portable** — scripts work on both macOS and Linux, handle `sed` differences, use dynamic path resolution

## Supported Stacks

The skill detects and adapts to your project, including but not limited to:

| Ecosystem | Detected By |
|-----------|-------------|
| Node.js | `package.json` + lockfiles (npm, yarn, pnpm, bun) |
| Python | `requirements.txt`, `Pipfile`, `pyproject.toml`, `poetry.lock` |
| Ruby | `Gemfile` |
| Go | `go.mod` |
| Rust | `Cargo.toml` |
| PHP | `composer.json` |
| Java/Kotlin | `build.gradle`, `pom.xml` |
| Elixir | `mix.exs` |

Database support: PostgreSQL, MySQL, and others based on detected connection patterns.
Cache support: Redis and others based on detected configuration.

## License

MIT
