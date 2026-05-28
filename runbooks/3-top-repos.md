---
title: Top Repos This Week
runtime: bash
---

# Top Repos This Week

Collect the top GitHub repos created in the past 7 days, normalize to CSV, load into Ghost Postgres, and verify.


## Setup Environment

### Location Config
```bash
export CWD="$(pwd)"
export PARENT="$(dirname "$CWD")"
cd "$PARENT"
```

### Data Config
```bash
export GHOST_NAME="remember-me"
export RAW_DIR="raw/github"
export TMP_DIR="/tmp/delacruz-github"

export TOP_REPOS_JSON="$RAW_DIR/top_repos.json"
export TOP_REPOS_CSV="$TMP_DIR/top_repos.csv"
```

```bash
mkdir -p "$TMP_DIR"
mkdir -p "$RAW_DIR"
```


## Download Top Repos This Week

### Authenticate GitHub CLI
```bash
GH_AUTH_STATUS=$(gh auth status 2>&1)
if ! echo "$GH_AUTH_STATUS" | grep -q "✓ Logged in"; then
  echo "Not logged in to GitHub"
  echo "use 'gh auth login' to authenticate"
  exit 1
fi
```

### Export top repos from the past 7 days
```bash
# Portable date: macOS uses -v, Linux uses -d
if WEEK_AGO=$(date -v-7d +%Y-%m-%d 2>/dev/null); then
  :
else
  WEEK_AGO=$(date -d '7 days ago' +%Y-%m-%d)
fi

TOP_REPOS_QUERY="created:>$WEEK_AGO"
TOP_REPOS_STARS_FILTER="stars:>50"

gh search repos "$TOP_REPOS_QUERY $TOP_REPOS_STARS_FILTER" \
  --limit 1000 \
  --json name,url,description,stargazersCount,language,updatedAt \
  > "$TOP_REPOS_JSON"
```


## Validate Inputs

```bash
if [ ! -f "$TOP_REPOS_JSON" ]; then
  echo "Missing required file: $TOP_REPOS_JSON"
  exit 1
fi
```


## Normalize JSON to CSV

### Top repos CSV
```bash
jq -r '
  [
    "name","url","description","language","stargazers_count","updated_at"
  ],
  (
    .[] | [
      .name,
      .url,
      .description,
      .language,
      .stargazersCount,
      .updatedAt
    ]
  )
  | @csv
' "$TOP_REPOS_JSON" > "$TOP_REPOS_CSV"
```


## Verify CSV

```bash
head "$TOP_REPOS_CSV"
```


## Ghost Database

```bash
export DB_ID=$(ghost list --json | jq -r --arg name "$GHOST_NAME" '.[] | select(.name == $name) | .id')
export PG_HOST=$(ghost connect $DB_ID)
echo "DB_ID: $DB_ID"
```

```bash
psql "$PG_HOST" <<SQL
CREATE TABLE IF NOT EXISTS github_top_repos (
  id BIGSERIAL PRIMARY KEY,
  name TEXT NOT NULL,
  url TEXT NOT NULL,
  description TEXT,
  language TEXT,
  stargazers_count INTEGER,
  updated_at TIMESTAMPTZ,
  db_added_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE github_top_repos ADD COLUMN IF NOT EXISTS db_added_at TIMESTAMPTZ DEFAULT NOW();
SQL
```

```bash
psql "$PG_HOST" <<PSQL
\copy github_top_repos (name, url, description, language, stargazers_count, updated_at) FROM '$TOP_REPOS_CSV' WITH (FORMAT csv, HEADER true)
PSQL
```


## Remove duplicates

```bash
psql "$PG_HOST" <<SQL
DELETE FROM github_top_repos a USING github_top_repos b
WHERE a.id > b.id AND a.url = b.url;
SQL
```


## Verify rows

```bash
psql "$PG_HOST" -c "
SELECT 'github_top_repos' AS table_name, COUNT(*) FROM github_top_repos;
"
```
