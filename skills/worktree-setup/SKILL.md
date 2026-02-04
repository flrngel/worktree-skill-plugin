---
name: worktree-setup
description: Set up git worktree management with isolated Postgres, Redis, and ports for parallel development. Run once per project to generate standalone scripts.
disable-model-invocation: true
allowed-tools: Bash, Read, Write, Glob, Grep, Edit
---

# Worktree Setup Skill

You are setting up git worktree management for this project. Follow each phase below in order. Adapt all generated scripts and config to what you detect in the project.

## Phase 1: Analyze Project

### 1.1 Verify Prerequisites

Run these checks:

```bash
# Must be in a git repo
git rev-parse --show-toplevel

# Must not already be set up
test -d .worktree && echo "ALREADY_EXISTS" || echo "OK"
```

If `.worktree` already exists, ask the user if they want to re-run setup (which will overwrite existing scripts but preserve config.json worktree entries).

### 1.2 Detect Project Type

Check for these files at the repo root to determine the project type and package manager:

| File | Type | Init Command |
|------|------|-------------|
| `package-lock.json` | Node.js | `npm install` |
| `yarn.lock` | Node.js | `yarn install` |
| `pnpm-lock.yaml` | Node.js | `pnpm install` |
| `bun.lockb` | Node.js | `bun install` |
| `package.json` (no lockfile) | Node.js | `npm install` |
| `requirements.txt` | Python | `pip install -r requirements.txt` |
| `Pipfile` | Python | `pipenv install` |
| `pyproject.toml` | Python | `pip install -e .` |
| `Gemfile` | Ruby | `bundle install` |
| `go.mod` | Go | `go mod download` |
| `Cargo.toml` | Rust | `cargo build` |
| `composer.json` | PHP | `composer install` |

Store the detected init command(s) for later.

### 1.3 Detect Environment Variables

Read `.env` file (if it exists) and look for these patterns:

- **DATABASE_URL**: Extract host, port, user, password, database name. Common formats:
  - `postgresql://user:pass@host:port/dbname`
  - `postgres://user:pass@host:port/dbname`
  - `mysql://user:pass@host:port/dbname`
- **REDIS_URL**: Extract host, port, DB number. Common formats:
  - `redis://host:port/db`
  - `redis://host:port`
- **PORT**: The application port number

Also check for individual DB vars like `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`, `PGDATABASE`, `PGHOST`, etc.

### 1.4 Check Docker Compose

If `docker-compose.yml` or `docker-compose.yaml` or `compose.yml` or `compose.yaml` exists, read it to find:
- Postgres service: image, port mapping, environment vars (POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB)
- Redis service: image, port mapping

Use these values if `.env` doesn't provide them.

### 1.5 Check Available Tools

```bash
command -v jq && echo "jq: yes" || echo "jq: no"
command -v psql && echo "psql: yes" || echo "psql: no"
command -v createdb && echo "createdb: yes" || echo "createdb: no"
command -v redis-cli && echo "redis-cli: yes" || echo "redis-cli: no"
command -v lsof && echo "lsof: yes" || echo "lsof: no"
```

`jq` is **required** — if missing, tell the user to install it and stop.

### 1.6 Detect Dev Server Command

For Node.js projects, check `package.json` for scripts:
- Prefer: `dev`, `start:dev`, `serve`, `start`
- Note the command for use in the summary

## Phase 2: Generate Config

Create `.worktree/config.json` with the detected values. Use this exact schema:

```json
{
  "base_port": <detected PORT or 3000>,
  "db_type": "<postgres|mysql|none>",
  "postgres": {
    "host": "<detected or localhost>",
    "port": <detected or 5432>,
    "user": "<detected or postgres>",
    "password": "<detected or postgres>",
    "main_db": "<detected database name>"
  },
  "redis": {
    "enabled": <true|false>,
    "host": "<detected or localhost>",
    "port": <detected or 6379>
  },
  "init_commands": ["<detected install command>"],
  "env_file": "<.env or null if no env file>",
  "env_vars": {
    "database_url_key": "<DATABASE_URL or DB var pattern detected>",
    "redis_url_key": "<REDIS_URL or null>",
    "port_key": "<PORT or null>"
  },
  "worktrees": {}
}
```

Omit sections that don't apply (e.g., no `postgres` key if no database detected, no `redis` key if no Redis detected).

## Phase 3: Generate Scripts

Create these scripts in `.worktree/bin/`. **IMPORTANT**: Adapt the scripts based on Phase 1 detection. The templates below show the full logic — fill in project-specific values where indicated by `__PLACEHOLDER__` comments.

### 3.1 `.worktree/bin/git-wt`

```bash
#!/usr/bin/env bash
set -euo pipefail

# Find repo root
REPO_ROOT="$(git rev-parse --show-toplevel)"
WT_DIR="$REPO_ROOT/.worktree"
BIN_DIR="$WT_DIR/bin"
CONFIG="$WT_DIR/config.json"

if [[ ! -f "$CONFIG" ]]; then
  echo "Error: .worktree/config.json not found."
  echo "Run /worktree-setup in Claude Code to initialize."
  exit 1
fi

if ! command -v jq &>/dev/null; then
  echo "Error: jq is required but not installed."
  echo "Install it: brew install jq (macOS) or apt-get install jq (Linux)"
  exit 1
fi

CMD="${1:-help}"
shift || true

case "$CMD" in
  create)  exec "$BIN_DIR/wt-create.sh" "$@" ;;
  delete|rm|remove)  exec "$BIN_DIR/wt-delete.sh" "$@" ;;
  list|ls) exec "$BIN_DIR/wt-list.sh" "$@" ;;
  help|--help|-h)
    echo "Usage: git wt <command> [args]"
    echo ""
    echo "Commands:"
    echo "  create <branch> [from]   Create a new worktree with isolated resources"
    echo "  delete <branch>          Delete a worktree and clean up resources"
    echo "  list                     List all managed worktrees"
    echo ""
    echo "Examples:"
    echo "  git wt create feat/login          # Branch from current HEAD"
    echo "  git wt create feat/login main     # Branch from main"
    echo "  git wt delete feat/login          # Full cleanup"
    echo "  git wt list                       # Show all worktrees"
    ;;
  *)
    echo "Unknown command: $CMD"
    echo "Run 'git wt help' for usage."
    exit 1
    ;;
esac
```

### 3.2 `.worktree/bin/wt-create.sh`

Write this script, adapting the DATABASE_URL/REDIS_URL/PORT sed patterns based on what was detected in Phase 1. Here is the full template:

```bash
#!/usr/bin/env bash
set -euo pipefail

REPO_ROOT="$(git rev-parse --show-toplevel)"
WT_DIR="$REPO_ROOT/.worktree"
CONFIG="$WT_DIR/config.json"

# --- Argument parsing ---
BRANCH="${1:?Usage: git wt create <branch> [from-ref]}"
FROM_REF="${2:-HEAD}"

# --- Sanitize branch name for directory/db use ---
SANITIZED="$(echo "$BRANCH" | sed 's/[^a-zA-Z0-9._-]/-/g' | sed 's/--*/-/g' | sed 's/^-//;s/-$//')"
WT_PATH="$WT_DIR/$SANITIZED"

# --- Check if already exists ---
if jq -e ".worktrees[\"$SANITIZED\"]" "$CONFIG" &>/dev/null; then
  echo "Error: Worktree '$SANITIZED' already exists."
  echo "Run 'git wt delete $BRANCH' first."
  exit 1
fi

if [[ -d "$WT_PATH" ]]; then
  echo "Error: Directory $WT_PATH already exists."
  exit 1
fi

echo "Creating worktree '$SANITIZED' from $FROM_REF..."

# --- Allocate port ---
BASE_PORT=$(jq -r '.base_port' "$CONFIG")
USED_PORTS=$(jq -r '[.worktrees[].port] | sort | .[]' "$CONFIG" 2>/dev/null || true)
NEW_PORT=$((BASE_PORT + 1))
while echo "$USED_PORTS" | grep -qx "$NEW_PORT" 2>/dev/null; do
  NEW_PORT=$((NEW_PORT + 1))
done

# Check port availability
if command -v lsof &>/dev/null && lsof -i :"$NEW_PORT" &>/dev/null; then
  echo "Warning: Port $NEW_PORT is in use, finding next available..."
  while lsof -i :"$NEW_PORT" &>/dev/null 2>&1; do
    NEW_PORT=$((NEW_PORT + 1))
  done
fi

echo "  Port: $NEW_PORT"

# --- Allocate Redis DB ---
REDIS_DB=""
# __REDIS_BLOCK_START__ (include this block only if redis is detected)
REDIS_ENABLED=$(jq -r '.redis.enabled // false' "$CONFIG")
if [[ "$REDIS_ENABLED" == "true" ]]; then
  USED_DBS=$(jq -r '[.worktrees[].redis_db // empty] | sort | .[]' "$CONFIG" 2>/dev/null || true)
  REDIS_DB=1
  while echo "$USED_DBS" | grep -qx "$REDIS_DB" 2>/dev/null; do
    REDIS_DB=$((REDIS_DB + 1))
  done
  echo "  Redis DB: $REDIS_DB"
fi
# __REDIS_BLOCK_END__

# --- Create Postgres DB ---
PG_DB=""
# __POSTGRES_BLOCK_START__ (include this block only if postgres is detected)
DB_TYPE=$(jq -r '.db_type // "none"' "$CONFIG")
if [[ "$DB_TYPE" == "postgres" ]]; then
  PG_HOST=$(jq -r '.postgres.host' "$CONFIG")
  PG_PORT=$(jq -r '.postgres.port' "$CONFIG")
  PG_USER=$(jq -r '.postgres.user' "$CONFIG")
  PG_PASS=$(jq -r '.postgres.password' "$CONFIG")
  MAIN_DB=$(jq -r '.postgres.main_db' "$CONFIG")
  PG_DB="${MAIN_DB}_wt_${SANITIZED}"

  export PGPASSWORD="$PG_PASS"

  echo "  Database: $PG_DB (cloning from $MAIN_DB)"

  # Try template copy first (faster, but fails if main_db has active connections)
  if ! createdb -h "$PG_HOST" -p "$PG_PORT" -U "$PG_USER" -T "$MAIN_DB" "$PG_DB" 2>/dev/null; then
    echo "  Template copy failed (active connections?), using pg_dump..."
    createdb -h "$PG_HOST" -p "$PG_PORT" -U "$PG_USER" "$PG_DB" 2>/dev/null || true
    if ! pg_dump -h "$PG_HOST" -p "$PG_PORT" -U "$PG_USER" "$MAIN_DB" | psql -h "$PG_HOST" -p "$PG_PORT" -U "$PG_USER" "$PG_DB" > /dev/null 2>&1; then
      echo "  Warning: Could not clone database. Created empty DB '$PG_DB'."
      createdb -h "$PG_HOST" -p "$PG_PORT" -U "$PG_USER" "$PG_DB" 2>/dev/null || true
    fi
  fi

  unset PGPASSWORD
fi
# __POSTGRES_BLOCK_END__

# --- Create git worktree ---
echo "  Creating git worktree..."
git worktree add "$WT_PATH" -b "$BRANCH" "$FROM_REF"

# --- Copy and patch .env ---
# __ENV_BLOCK_START__ (include this block only if env_file is detected)
ENV_FILE=$(jq -r '.env_file // empty' "$CONFIG")
if [[ -n "$ENV_FILE" && -f "$REPO_ROOT/$ENV_FILE" ]]; then
  cp "$REPO_ROOT/$ENV_FILE" "$WT_PATH/$ENV_FILE"
  echo "  Patching $ENV_FILE..."

  # __ENV_SED_COMMANDS__
  # Claude: Replace this section with sed commands based on detected env var patterns.
  # Examples of what to generate:
  #
  # If DATABASE_URL=postgresql://user:pass@host:port/dbname was detected:
  #   sed -i.bak "s|/MAIN_DB_NAME|/$PG_DB|g" "$WT_PATH/$ENV_FILE"
  #
  # If individual DB vars (DB_NAME=myapp):
  #   sed -i.bak "s/^DB_NAME=.*/DB_NAME=$PG_DB/" "$WT_PATH/$ENV_FILE"
  #
  # If REDIS_URL=redis://host:port/0 was detected:
  #   sed -i.bak "s|redis://\([^/]*\)/[0-9]*|redis://\1/$REDIS_DB|g" "$WT_PATH/$ENV_FILE"
  #
  # If PORT=3000 was detected:
  #   sed -i.bak "s/^PORT=.*/PORT=$NEW_PORT/" "$WT_PATH/$ENV_FILE"
  #
  # Always clean up backup:
  #   rm -f "$WT_PATH/$ENV_FILE.bak"

fi
# __ENV_BLOCK_END__

# --- Run init commands ---
INIT_CMDS=$(jq -r '.init_commands[]' "$CONFIG" 2>/dev/null || true)
if [[ -n "$INIT_CMDS" ]]; then
  echo "  Running init commands..."
  cd "$WT_PATH"
  while IFS= read -r cmd; do
    echo "    \$ $cmd"
    eval "$cmd" || echo "    Warning: '$cmd' failed"
  done <<< "$INIT_CMDS"
  cd "$REPO_ROOT"
fi

# --- Update config ---
CREATED_AT=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
jq --arg name "$SANITIZED" \
   --arg branch "$BRANCH" \
   --argjson port "$NEW_PORT" \
   --arg pg_db "$PG_DB" \
   --arg redis_db "$REDIS_DB" \
   --arg created "$CREATED_AT" \
   --arg path "$WT_PATH" \
   '.worktrees[$name] = {
     branch: $branch,
     port: $port,
     pg_db: $pg_db,
     redis_db: (if $redis_db == "" then null else ($redis_db | tonumber) end),
     path: $path,
     created: $created
   }' "$CONFIG" > "$CONFIG.tmp" && mv "$CONFIG.tmp" "$CONFIG"

echo ""
echo "Worktree '$SANITIZED' created successfully!"
echo ""
echo "  Path:     $WT_PATH"
echo "  Branch:   $BRANCH"
echo "  Port:     $NEW_PORT"
[[ -n "$PG_DB" ]] && echo "  Database: $PG_DB"
[[ -n "$REDIS_DB" ]] && echo "  Redis DB: $REDIS_DB"
echo ""
echo "To start working:"
echo "  cd $WT_PATH"
```

**IMPORTANT**: When writing this script, you MUST replace the `__ENV_SED_COMMANDS__` section with actual sed commands that match the specific env var patterns detected in Phase 1. Do NOT leave placeholders. For example:

- If you detected `DATABASE_URL=postgresql://postgres:postgres@localhost:5432/myapp` in `.env`, write:
  ```bash
  sed -i.bak "s|/myapp|/$PG_DB|g" "$WT_PATH/$ENV_FILE"
  ```
- If you detected `PORT=3000`, write:
  ```bash
  sed -i.bak "s/^PORT=.*/PORT=$NEW_PORT/" "$WT_PATH/$ENV_FILE"
  ```
- If you detected `REDIS_URL=redis://localhost:6379/0`, write:
  ```bash
  sed -i.bak "s|redis://\([^/]*\)/[0-9]*|redis://\1/$REDIS_DB|g" "$WT_PATH/$ENV_FILE"
  ```

Also remove the `__BLOCK_START__/__BLOCK_END__` comment markers — they are only template guidance. Keep or remove the actual blocks based on detection.

### 3.3 `.worktree/bin/wt-delete.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

REPO_ROOT="$(git rev-parse --show-toplevel)"
WT_DIR="$REPO_ROOT/.worktree"
CONFIG="$WT_DIR/config.json"

BRANCH="${1:?Usage: git wt delete <branch>}"

# Sanitize to match config key
SANITIZED="$(echo "$BRANCH" | sed 's/[^a-zA-Z0-9._-]/-/g' | sed 's/--*/-/g' | sed 's/^-//;s/-$//')"

# Look up in config (try both original and sanitized)
if ! jq -e ".worktrees[\"$SANITIZED\"]" "$CONFIG" &>/dev/null; then
  echo "Error: Worktree '$SANITIZED' not found in config."
  echo "Available worktrees:"
  jq -r '.worktrees | keys[]' "$CONFIG" 2>/dev/null || echo "  (none)"
  exit 1
fi

WT_PATH=$(jq -r ".worktrees[\"$SANITIZED\"].path" "$CONFIG")
PG_DB=$(jq -r ".worktrees[\"$SANITIZED\"].pg_db // empty" "$CONFIG")
REDIS_DB=$(jq -r ".worktrees[\"$SANITIZED\"].redis_db // empty" "$CONFIG")

echo "Deleting worktree '$SANITIZED'..."

# --- Drop Postgres DB ---
# __POSTGRES_DELETE_BLOCK_START__
DB_TYPE=$(jq -r '.db_type // "none"' "$CONFIG")
if [[ "$DB_TYPE" == "postgres" && -n "$PG_DB" && "$PG_DB" != "null" ]]; then
  PG_HOST=$(jq -r '.postgres.host' "$CONFIG")
  PG_PORT=$(jq -r '.postgres.port' "$CONFIG")
  PG_USER=$(jq -r '.postgres.user' "$CONFIG")
  PG_PASS=$(jq -r '.postgres.password' "$CONFIG")
  export PGPASSWORD="$PG_PASS"

  echo "  Dropping database: $PG_DB"
  dropdb -h "$PG_HOST" -p "$PG_PORT" -U "$PG_USER" --if-exists "$PG_DB" 2>/dev/null || \
    echo "  Warning: Could not drop database $PG_DB"

  unset PGPASSWORD
fi
# __POSTGRES_DELETE_BLOCK_END__

# --- Flush Redis DB ---
# __REDIS_DELETE_BLOCK_START__
REDIS_ENABLED=$(jq -r '.redis.enabled // false' "$CONFIG")
if [[ "$REDIS_ENABLED" == "true" && -n "$REDIS_DB" && "$REDIS_DB" != "null" ]]; then
  REDIS_HOST=$(jq -r '.redis.host' "$CONFIG")
  REDIS_PORT=$(jq -r '.redis.port' "$CONFIG")
  echo "  Flushing Redis DB: $REDIS_DB"
  redis-cli -h "$REDIS_HOST" -p "$REDIS_PORT" -n "$REDIS_DB" FLUSHDB &>/dev/null || \
    echo "  Warning: Could not flush Redis DB $REDIS_DB"
fi
# __REDIS_DELETE_BLOCK_END__

# --- Remove git worktree ---
echo "  Removing git worktree..."
if [[ -d "$WT_PATH" ]]; then
  git worktree remove "$WT_PATH" --force 2>/dev/null || \
    rm -rf "$WT_PATH"
fi
git worktree prune 2>/dev/null || true

# --- Update config ---
jq "del(.worktrees[\"$SANITIZED\"])" "$CONFIG" > "$CONFIG.tmp" && mv "$CONFIG.tmp" "$CONFIG"

echo ""
echo "Worktree '$SANITIZED' deleted successfully."
```

### 3.4 `.worktree/bin/wt-list.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

REPO_ROOT="$(git rev-parse --show-toplevel)"
WT_DIR="$REPO_ROOT/.worktree"
CONFIG="$WT_DIR/config.json"

WORKTREES=$(jq -r '.worktrees | keys[]' "$CONFIG" 2>/dev/null || true)

if [[ -z "$WORKTREES" ]]; then
  echo "No worktrees found."
  echo ""
  echo "Create one with: git wt create <branch-name>"
  exit 0
fi

# Header
printf "%-25s %-25s %-6s %-30s %-10s %s\n" "NAME" "BRANCH" "PORT" "PG_DB" "REDIS_DB" "CREATED"
printf "%-25s %-25s %-6s %-30s %-10s %s\n" "----" "------" "----" "-----" "--------" "-------"

# Rows
while IFS= read -r name; do
  BRANCH=$(jq -r ".worktrees[\"$name\"].branch // \"-\"" "$CONFIG")
  PORT=$(jq -r ".worktrees[\"$name\"].port // \"-\"" "$CONFIG")
  PG_DB=$(jq -r ".worktrees[\"$name\"].pg_db // \"-\"" "$CONFIG")
  REDIS_DB=$(jq -r ".worktrees[\"$name\"].redis_db // \"-\"" "$CONFIG")
  CREATED=$(jq -r ".worktrees[\"$name\"].created // \"-\"" "$CONFIG")
  printf "%-25s %-25s %-6s %-30s %-10s %s\n" "$name" "$BRANCH" "$PORT" "$PG_DB" "$REDIS_DB" "$CREATED"
done <<< "$WORKTREES"
```

## Phase 4: Configure Git (Local Only)

### 4.1 Add to git excludes

```bash
# Add .worktree to local excludes (NOT .gitignore)
EXCLUDES="$(git rev-parse --show-toplevel)/.git/info/exclude"
mkdir -p "$(dirname "$EXCLUDES")"
if ! grep -qx '.worktree' "$EXCLUDES" 2>/dev/null; then
  echo '.worktree' >> "$EXCLUDES"
fi
```

### 4.2 Set up git alias

```bash
# Local git alias only (not global)
git config alias.wt '!.worktree/bin/git-wt'
```

### 4.3 Make scripts executable

```bash
chmod +x .worktree/bin/*
```

## Phase 5: Report Summary

After completing all phases, print a summary like this:

```
=== Worktree Setup Complete ===

Project detected:
  Type:        Node.js (npm)
  Database:    PostgreSQL (myapp @ localhost:5432)
  Redis:       Yes (localhost:6379)
  Base port:   3000
  Env file:    .env

Generated files:
  .worktree/config.json     - Configuration
  .worktree/bin/git-wt      - Main command router
  .worktree/bin/wt-create.sh - Create worktree + resources
  .worktree/bin/wt-delete.sh - Delete worktree + cleanup
  .worktree/bin/wt-list.sh   - List worktrees

Git config (local only):
  .git/info/exclude         - Added .worktree
  git alias 'wt'            - Configured

Usage:
  git wt create feat/my-feature        # Create from current HEAD
  git wt create feat/my-feature main   # Create from main branch
  git wt list                          # List all worktrees
  git wt delete feat/my-feature        # Delete and clean up

Zero files modified in git — 'git status' will show a clean tree.
```

Adapt the summary based on what was actually detected and configured. If no database was found, don't mention it. If no Redis, don't mention it. Be accurate about what was set up.

## Important Notes for Claude

1. **Do NOT leave placeholders in scripts.** Every script must be fully functional with real values.
2. **Adapt sed commands** to the actual env var format detected. Don't assume patterns — check the actual `.env` content.
3. **Remove template blocks** that don't apply. If no Redis was detected, remove the Redis blocks entirely from the scripts.
4. **macOS sed compatibility**: Use `sed -i.bak` (not `sed -i ''`) for portability, then `rm -f *.bak`.
5. **Test the git alias** after setup by running `git wt help` and confirming it works.
6. **If jq is not installed**, stop and inform the user — it's the one hard requirement.
7. **Config path references** in scripts should always use `git rev-parse --show-toplevel` for portability.
