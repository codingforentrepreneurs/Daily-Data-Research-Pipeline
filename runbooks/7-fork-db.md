---
title: Fork Ghost DB for Agents
runtime: bash
---

# Fork Ghost DB for Agents

Create one disposable fork of a Ghost Postgres database for each existing Hermes
agent profile in `GHOST_AGENT_FORK_LIST`, then write that fork's read-only
connection string to the profile `.env` as `DATABASE_URL`.

Defaults: `GHOST_NAME=daily-research` and `GHOST_AGENT_FORK_LIST=jnn-triage`.

Example invocation: `GHOST_AGENT_FORK_LIST="jnn-camera-researcher,jnn-3d-print-researcher" wb run runbooks/7-fork-db.md`.

Missing Hermes profiles are skipped. No fork is created for a missing profile.


## Fork Database for Agents

```bash
set -euo pipefail

export CWD="$(pwd)"
if [ "$(basename "$CWD")" = "runbooks" ]; then
  export REPO_ROOT="$(dirname "$CWD")"
else
  export REPO_ROOT="$CWD"
fi

export GHOST_NAME="${GHOST_NAME:-daily-research}"
export GHOST_AGENT_FORK_LIST="${GHOST_AGENT_FORK_LIST:-jnn-triage}"
export FORK_PROFILE_ROOT="${FORK_PROFILE_ROOT:-/Users/${USER:-cfe}/.hermes/profiles}"
export FORK_RUNTIME_DIR="${FORK_RUNTIME_DIR:-$REPO_ROOT/.wb-ghost-runtime}"
export FORK_STAMP="${FORK_STAMP:-$(date +%Y%m%d%H%M%S)}"
export FORK_PREFIX="${FORK_PREFIX:-${GHOST_NAME}-fork}"

mkdir -p "$FORK_RUNTIME_DIR"

for tool in ghost jq awk; do
  if ! command -v "$tool" >/dev/null 2>&1; then
    echo "Missing required tool: $tool"
    exit 1
  fi
done

if [ -z "${GHOST_API_KEY:-}" ]; then
  if ghost list --json >/dev/null 2>&1; then
    echo "GHOST_API_KEY is not set; using existing ghost CLI authentication."
  else
    echo "Missing GHOST_API_KEY and ghost CLI cannot list databases."
    exit 1
  fi
fi

case "$GHOST_NAME" in
  *[!A-Za-z0-9._-]*|'') echo "Unsafe GHOST_NAME: $GHOST_NAME"; exit 1 ;;
esac

case "$FORK_PREFIX" in
  *[!A-Za-z0-9._-]*|'') echo "Unsafe FORK_PREFIX: $FORK_PREFIX"; exit 1 ;;
esac

case "$FORK_STAMP" in
  *[!0-9]*|'') echo "FORK_STAMP must contain only digits"; exit 1 ;;
esac

export FORK_AGENT_FILE="$FORK_RUNTIME_DIR/fork-agent-profiles.txt"
: > "$FORK_AGENT_FILE"

printf '%s\n' "$GHOST_AGENT_FORK_LIST" \
  | tr ',' '\n' \
  | awk '{$1=$1}; NF && !seen[$0]++ { print }' \
  | while IFS= read -r AGENT_PROFILE; do
      case "$AGENT_PROFILE" in
        *[!A-Za-z0-9._-]*|'')
          echo "Unsafe Hermes profile name: $AGENT_PROFILE"
          exit 1
          ;;
      esac

      if [ ! -d "$FORK_PROFILE_ROOT/$AGENT_PROFILE" ]; then
        echo "Skipping missing Hermes profile: $AGENT_PROFILE"
        continue
      fi

      printf '%s\n' "$AGENT_PROFILE" >> "$FORK_AGENT_FILE"
    done

if [ ! -s "$FORK_AGENT_FILE" ]; then
  echo "No existing Hermes profiles found. Nothing to fork."
  exit 0
fi

echo "GHOST_NAME=$GHOST_NAME"
echo "GHOST_AGENT_FORK_LIST=$GHOST_AGENT_FORK_LIST"
echo "FORK_PROFILE_ROOT=$FORK_PROFILE_ROOT"
echo "FORK_RUNTIME_DIR=$FORK_RUNTIME_DIR"

export DB_ID="$(ghost list --json | jq -r --arg name "$GHOST_NAME" '.[] | select(.name == $name) | .id' | head -1)"

if [ -z "$DB_ID" ]; then
  echo "Could not find Ghost database named: $GHOST_NAME"
  echo "Create it first with: ghost create --name $GHOST_NAME --wait"
  exit 1
fi

echo "Ghost database: $GHOST_NAME"
echo "DB_ID: $DB_ID"

export FORKS_JSONL="$FORK_RUNTIME_DIR/forks-${FORK_STAMP}.jsonl"
: > "$FORKS_JSONL"

while IFS= read -r AGENT_PROFILE; do
  export AGENT_PROFILE
  export FORK_NAME="${FORK_PREFIX}-${AGENT_PROFILE}-${FORK_STAMP}"
  export AGENT_ENV="$FORK_PROFILE_ROOT/$AGENT_PROFILE/.env"

  case "$FORK_NAME" in
    *[!A-Za-z0-9._-]*|'') echo "Unsafe fork name: $FORK_NAME"; exit 1 ;;
  esac

  if [ "$(ghost list --json | jq -r --arg name "$FORK_NAME" 'any(.[]; .name == $name)')" = "true" ]; then
    echo "Fork already exists: $FORK_NAME"
    exit 1
  fi

  echo "Forking $GHOST_NAME for $AGENT_PROFILE as $FORK_NAME"
  echo "Ghost fork creation can take a few minutes."
  export FORK_JSON="$(ghost fork "$DB_ID" --name "$FORK_NAME" --wait --json)"
  export FORK_DB_ID="$(printf '%s' "$FORK_JSON" | jq -r 'if type == "array" then .[0] else . end | .id // .database.id // empty')"

  if [ -z "$FORK_DB_ID" ]; then
    echo "Could not read fork ID from Ghost response:"
    printf '%s\n' "$FORK_JSON"
    exit 1
  fi

  export DATABASE_URL="$(ghost connect --read-only "$FORK_DB_ID")"
  if [ -z "$DATABASE_URL" ]; then
    echo "ghost connect returned an empty DATABASE_URL for $FORK_NAME"
    exit 1
  fi

  umask 077
  AGENT_ENV_TMP="$(mktemp "${AGENT_ENV}.tmp.XXXXXX")"
  if [ -f "$AGENT_ENV" ]; then
    grep -v -E '^DATABASE_URL=' "$AGENT_ENV" > "$AGENT_ENV_TMP" || true
  fi
  printf 'DATABASE_URL=%s\n' "$DATABASE_URL" >> "$AGENT_ENV_TMP"
  mv "$AGENT_ENV_TMP" "$AGENT_ENV"
  chmod 600 "$AGENT_ENV"

  printf '%s\n' "$FORK_JSON" > "$FORK_RUNTIME_DIR/fork-${AGENT_PROFILE}.json"
  jq -n -c \
    --arg ghost_name "$GHOST_NAME" \
    --arg db_id "$DB_ID" \
    --arg fork_name "$FORK_NAME" \
    --arg fork_db_id "$FORK_DB_ID" \
    --arg agent_profile "$AGENT_PROFILE" \
    --arg agent_env "$AGENT_ENV" \
    '{
      ghost_name: $ghost_name,
      db_id: $db_id,
      fork_name: $fork_name,
      fork_db_id: $fork_db_id,
      agent_profile: $agent_profile,
      env_key: "DATABASE_URL",
      agent_env: $agent_env
    }' >> "$FORKS_JSONL"

  ghost list --json \
    | jq -r --arg prefix "${FORK_PREFIX}-${AGENT_PROFILE}-" --arg keep "$FORK_NAME" '
        .[]
        | select(.name != $keep)
        | select(.name | startswith($prefix))
        | .id
      ' \
    | while IFS= read -r OLD_FORK_DB_ID; do
        if [ -n "$OLD_FORK_DB_ID" ]; then
          echo "Deleting previous fork for $AGENT_PROFILE: $OLD_FORK_DB_ID"
          ghost delete "$OLD_FORK_DB_ID" --confirm
        fi
      done

  echo "Updated Hermes profile: $AGENT_PROFILE"
done < "$FORK_AGENT_FILE"

echo "Created fork metadata: $FORKS_JSONL"
echo "Wrote DATABASE_URL to each updated Hermes profile .env."
echo "Restart any already-running agent so it reads the new .env."
```
