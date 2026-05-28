---
title: Fork Ghost Database
runtime: bash
---

# Fork Ghost Database

Fork the `remember-me` Ghost Postgres database and save the fork metadata for the next demo runbook.

This runbook does not write a Postgres connection string to disk. It only stores Ghost database IDs in `demo/.ghost-fork.env`, which is gitignored.


## Setup Environment

### Location Config
```bash
export CWD="$(pwd)"
if [ "$(basename "$CWD")" = "demo" ]; then
  export REPO_ROOT="$(dirname "$CWD")"
else
  export REPO_ROOT="$CWD"
fi
```

### Demo Config
```bash
export GHOST_SOURCE_NAME="${GHOST_SOURCE_NAME:-remember-me}"
export GHOST_FORK_NAME="${GHOST_FORK_NAME:-remember-me-codex-demo-$(date +%Y%m%d%H%M%S)}"
export DEMO_STATE_FILE="$REPO_ROOT/demo/.ghost-fork.env"
```

### Check required tools
```bash
for tool in ghost jq psql; do
  if ! command -v "$tool" >/dev/null 2>&1; then
    echo "Missing required tool: $tool"
    exit 1
  fi
done
```


## Find Source Database

```bash
export GHOST_SOURCE_DB_ID=$(ghost list --json | jq -r --arg name "$GHOST_SOURCE_NAME" '.[] | select(.name == $name) | .id' | head -1)

if [ -z "$GHOST_SOURCE_DB_ID" ]; then
  echo "Could not find Ghost database named: $GHOST_SOURCE_NAME"
  echo "Create it first with: ghost create --name $GHOST_SOURCE_NAME --wait"
  exit 1
fi

echo "Source database: $GHOST_SOURCE_NAME"
echo "Source ID: $GHOST_SOURCE_DB_ID"
```


## Fork Database

```bash
export GHOST_FORK_JSON=$(ghost fork "$GHOST_SOURCE_DB_ID" --name "$GHOST_FORK_NAME" --wait --json)
export GHOST_FORK_DB_ID=$(printf '%s' "$GHOST_FORK_JSON" | jq -r 'if type == "array" then .[0] else . end | .id // .database.id // empty')

if [ -z "$GHOST_FORK_DB_ID" ]; then
  echo "Could not read fork database ID from Ghost response:"
  printf '%s\n' "$GHOST_FORK_JSON"
  exit 1
fi

printf '%s\n' "$GHOST_FORK_JSON" | jq .
```


## Save Demo State

```bash
{
  printf 'export GHOST_SOURCE_NAME=%q\n' "$GHOST_SOURCE_NAME"
  printf 'export GHOST_SOURCE_DB_ID=%q\n' "$GHOST_SOURCE_DB_ID"
  printf 'export GHOST_FORK_NAME=%q\n' "$GHOST_FORK_NAME"
  printf 'export GHOST_FORK_DB_ID=%q\n' "$GHOST_FORK_DB_ID"
  printf 'export GHOST_FORK_CREATED_AT=%q\n' "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
} > "$DEMO_STATE_FILE"

echo "Saved fork state to $DEMO_STATE_FILE"
```


## Verify Fork Is Queryable

```bash
export SOURCE_PG_HOST=$(ghost connect "$GHOST_SOURCE_DB_ID")
export FORK_PG_HOST=$(ghost connect "$GHOST_FORK_DB_ID")

echo "Source database tables:"
psql "$SOURCE_PG_HOST" -c "
SELECT table_name
FROM information_schema.tables
WHERE table_schema = 'public'
  AND table_type = 'BASE TABLE'
ORDER BY table_name;
"

echo "Fork database tables:"
psql "$FORK_PG_HOST" -c "
SELECT table_name
FROM information_schema.tables
WHERE table_schema = 'public'
  AND table_type = 'BASE TABLE'
ORDER BY table_name;
"
```
