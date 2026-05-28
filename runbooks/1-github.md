---
title: GitHub
runtime: bash
---

# GitHub

Collect GitHub repo data with the GitHub CLI, normalize it into CSV files, load it into Ghost Postgres, and verify the rows exist.


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

export STARS_JSON="$RAW_DIR/stars.json"
export REPOS_JSON="$RAW_DIR/repos.json"
export TOP_REPOS_JSON="$RAW_DIR/top_repos.json"
export TRENDING_JSON="$RAW_DIR/trending_recent.json"

export STARS_CSV="$TMP_DIR/stars.csv"
export REPOS_CSV="$TMP_DIR/repos.csv"
export TOP_REPOS_CSV="$TMP_DIR/top_repos.csv"
export TRENDING_CSV="$TMP_DIR/trending_recent.csv"
```

```bash
mkdir -p "$TMP_DIR"
mkdir -p "$RAW_DIR"
```


## Optional: Download Raw GitHub Data

These commands do not need to run every time. They are here so the collection workflow lives with the load workflow.

### Authenticate GitHub CLI
```bash
if [ "${GITHUB_ACTIONS:-}" = "true" ] && [ -z "${GH_TOKEN:-}" ]; then
  echo "Missing GH_TOKEN in GitHub Actions"
  echo "Set a user personal access token in the 'GH_TOKEN' repository secret"
  echo "The default Actions GITHUB_TOKEN cannot access user-scoped endpoints like /user/starred"
  exit 1
fi

GH_AUTH_STATUS=$(gh auth status 2>&1)
if ! echo "$GH_AUTH_STATUS" | grep -q "✓ Logged in"; then
  echo "Not logged in to GitHub"
  echo "use 'gh auth login' to authenticate"
  exit 1
fi
```

In GitHub Actions, use a user PAT stored as the `GH_TOKEN` secret. The default `GITHUB_TOKEN` is an installation token and cannot call user-scoped endpoints such as `user/starred` and `user/repos`.

### Export starred repos
```bash
gh api --paginate user/starred > "$STARS_JSON"
```

### Export owned repos
```bash
gh api --paginate user/repos > "$REPOS_JSON"
```

### Export top repos
```bash
gh search repos "stars:>50000" \
  --limit 1000 \
  --json name,url,description,stargazersCount,language,updatedAt \
  > "$TOP_REPOS_JSON"
```

### Export recently trending repos
```bash
# Portable date: macOS uses -v, Linux uses -d
if WEEK_AGO=$(date -v-7d +%Y-%m-%d 2>/dev/null); then
  :
else
  WEEK_AGO=$(date -d '7 days ago' +%Y-%m-%d)
fi

TRENDING_QUERY="created:>$WEEK_AGO"
TRENDING_STARS_FILTER="stars:>50"

gh search repos "$TRENDING_QUERY $TRENDING_STARS_FILTER" \
  --limit 200 \
  --json name,url,description,stargazersCount,language,updatedAt \
  > "$TRENDING_JSON"
```


## Validate Inputs

```bash
for f in "$STARS_JSON" "$REPOS_JSON" "$TOP_REPOS_JSON" "$TRENDING_JSON"; do
  if [ ! -f "$f" ]; then
    echo "Missing required file: $f"
    exit 1
  fi
done
```


## Normalize JSON to CSV

### Starred repos CSV
```bash
jq -r '
  [
    "github_id","full_name","name","html_url","description","language",
    "private","fork","archived","visibility","stargazers_count","watchers_count",
    "forks_count","open_issues_count","created_at","updated_at","pushed_at",
    "homepage","default_branch","owner_login","owner_type"
  ],
  (
    .[] | [
      .id,
      .full_name,
      .name,
      .html_url,
      .description,
      .language,
      .private,
      .fork,
      .archived,
      .visibility,
      .stargazers_count,
      .watchers_count,
      .forks_count,
      .open_issues_count,
      .created_at,
      .updated_at,
      .pushed_at,
      .homepage,
      .default_branch,
      .owner.login,
      .owner.type
    ]
  )
  | @csv
' "$STARS_JSON" > "$STARS_CSV"
```

### Owned repos CSV
```bash
jq -r '
  [
    "github_id","full_name","name","html_url","description","language",
    "private","fork","archived","visibility","stargazers_count","watchers_count",
    "forks_count","open_issues_count","created_at","updated_at","pushed_at",
    "homepage","default_branch","owner_login","owner_type"
  ],
  (
    .[] | [
      .id,
      .full_name,
      .name,
      .html_url,
      .description,
      .language,
      .private,
      .fork,
      .archived,
      .visibility,
      .stargazers_count,
      .watchers_count,
      .forks_count,
      .open_issues_count,
      .created_at,
      .updated_at,
      .pushed_at,
      .homepage,
      .default_branch,
      .owner.login,
      .owner.type
    ]
  )
  | @csv
' "$REPOS_JSON" > "$REPOS_CSV"
```

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

### Trending repos CSV
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
' "$TRENDING_JSON" > "$TRENDING_CSV"
```


## Verify CSVs

```bash
head "$STARS_CSV"
head "$REPOS_CSV"
head "$TOP_REPOS_CSV"
head "$TRENDING_CSV"
```


## Ghost Database

```bash
export DB_ID=$(ghost list --json | jq -r --arg name "$GHOST_NAME" '.[] | select(.name == $name) | .id')
export PG_HOST=$(ghost connect $DB_ID)
echo "DB_ID: $DB_ID"
```

```bash
psql "$PG_HOST" <<SQL
CREATE TABLE IF NOT EXISTS github_stars (
  id BIGSERIAL PRIMARY KEY,
  github_id BIGINT,
  full_name TEXT NOT NULL,
  name TEXT NOT NULL,
  html_url TEXT NOT NULL,
  description TEXT,
  language TEXT,
  private BOOLEAN,
  fork BOOLEAN,
  archived BOOLEAN,
  visibility TEXT,
  stargazers_count INTEGER,
  watchers_count INTEGER,
  forks_count INTEGER,
  open_issues_count INTEGER,
  created_at TIMESTAMPTZ,
  updated_at TIMESTAMPTZ,
  pushed_at TIMESTAMPTZ,
  homepage TEXT,
  default_branch TEXT,
  owner_login TEXT,
  owner_type TEXT,
  db_added_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS github_repos (
  id BIGSERIAL PRIMARY KEY,
  github_id BIGINT,
  full_name TEXT NOT NULL,
  name TEXT NOT NULL,
  html_url TEXT NOT NULL,
  description TEXT,
  language TEXT,
  private BOOLEAN,
  fork BOOLEAN,
  archived BOOLEAN,
  visibility TEXT,
  stargazers_count INTEGER,
  watchers_count INTEGER,
  forks_count INTEGER,
  open_issues_count INTEGER,
  created_at TIMESTAMPTZ,
  updated_at TIMESTAMPTZ,
  pushed_at TIMESTAMPTZ,
  homepage TEXT,
  default_branch TEXT,
  owner_login TEXT,
  owner_type TEXT,
  db_added_at TIMESTAMPTZ DEFAULT NOW()
);

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

CREATE TABLE IF NOT EXISTS github_trending_recent (
  id BIGSERIAL PRIMARY KEY,
  name TEXT NOT NULL,
  url TEXT NOT NULL,
  description TEXT,
  language TEXT,
  stargazers_count INTEGER,
  updated_at TIMESTAMPTZ,
  db_added_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE github_stars ADD COLUMN IF NOT EXISTS db_added_at TIMESTAMPTZ DEFAULT NOW();
ALTER TABLE github_repos ADD COLUMN IF NOT EXISTS db_added_at TIMESTAMPTZ DEFAULT NOW();
ALTER TABLE github_top_repos ADD COLUMN IF NOT EXISTS db_added_at TIMESTAMPTZ DEFAULT NOW();
ALTER TABLE github_trending_recent ADD COLUMN IF NOT EXISTS db_added_at TIMESTAMPTZ DEFAULT NOW();
SQL
```

```bash
psql "$PG_HOST" <<PSQL
\copy github_stars (github_id, full_name, name, html_url, description, language, private, fork, archived, visibility, stargazers_count, watchers_count, forks_count, open_issues_count, created_at, updated_at, pushed_at, homepage, default_branch, owner_login, owner_type) FROM '$STARS_CSV' WITH (FORMAT csv, HEADER true)
\copy github_repos (github_id, full_name, name, html_url, description, language, private, fork, archived, visibility, stargazers_count, watchers_count, forks_count, open_issues_count, created_at, updated_at, pushed_at, homepage, default_branch, owner_login, owner_type) FROM '$REPOS_CSV' WITH (FORMAT csv, HEADER true)
\copy github_top_repos (name, url, description, language, stargazers_count, updated_at) FROM '$TOP_REPOS_CSV' WITH (FORMAT csv, HEADER true)
\copy github_trending_recent (name, url, description, language, stargazers_count, updated_at) FROM '$TRENDING_CSV' WITH (FORMAT csv, HEADER true)
PSQL
```


## Remove duplicates

```bash
psql "$PG_HOST" <<SQL
DELETE FROM github_stars a USING github_stars b
WHERE a.id > b.id AND a.github_id = b.github_id;

DELETE FROM github_repos a USING github_repos b
WHERE a.id > b.id AND a.github_id = b.github_id;

DELETE FROM github_top_repos a USING github_top_repos b
WHERE a.id > b.id AND a.url = b.url;

DELETE FROM github_trending_recent a USING github_trending_recent b
WHERE a.id > b.id AND a.url = b.url;
SQL
```


## Verify rows

```bash
psql "$PG_HOST" -c "
SELECT 'github_stars' AS table_name, COUNT(*) FROM github_stars
UNION ALL
SELECT 'github_repos' AS table_name, COUNT(*) FROM github_repos
UNION ALL
SELECT 'github_top_repos' AS table_name, COUNT(*) FROM github_top_repos
UNION ALL
SELECT 'github_trending_recent' AS table_name, COUNT(*) FROM github_trending_recent;
"
```
