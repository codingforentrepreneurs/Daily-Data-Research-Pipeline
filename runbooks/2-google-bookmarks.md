---
title: Google Bookmarks
runtime: bash
---

# Google Bookmarks

Parse the Chrome/Google bookmarks export, load it into Ghost Postgres, and verify the rows exist.


## Setup Environment

### Location Config
```bash
export CWD="$(pwd)"
if [ "$(basename "$CWD")" = "runbooks" ]; then
    export REPO_ROOT="$(dirname "$CWD")"
else
    export REPO_ROOT="$CWD"
fi
```

### Data Config
```bash
export TABLE_NAME="google_bookmarks"
export CSV_PATH="$REPO_ROOT/raw/chrome/bookmarks.csv"
export GIST_PATH="$REPO_ROOT/gists/google_bookmarks_to_csv.py"
export GHOST_NAME="remember-me"
```

### Create Output Directories

```bash
mkdir -p "$(dirname "$CSV_PATH")"
mkdir -p "$(dirname "$GIST_PATH")"
```

### Google Bookmarks Parser Download

```bash
if [ ! -f "$GIST_PATH" ]; then
    curl -o "$GIST_PATH" https://gist.githubusercontent.com/codingforentrepreneurs/916be33f515589df486c80ca9d07ca0d/raw/6e25751789a6b7f98a1b9b0a4195e1133f6e5c02/google_bookmarks_to_csv.py
fi
```

## Find latest Bookmarks Export

```bash
export BOOKMARK_PATH=$(ls -t "$REPO_ROOT"/raw/chrome/bookmarks*.html 2>/dev/null | head -1)
echo "BOOKMARK_PATH: $BOOKMARK_PATH"
```

### Run Error if no export found
```bash
if [ -z "$BOOKMARK_PATH" ]; then
    echo "No bookmark export found. Please export your bookmarks from Chrome and try again."
    exit 1
fi
```

## Parse the export

```bash
uv run "$GIST_PATH" \
    "$BOOKMARK_PATH" \
    "$CSV_PATH"
```

## Verify CSV

```bash
head "$CSV_PATH"
```


## Ghost Database

```bash
export DB_ID=$(ghost list --json | jq -r --arg name "$GHOST_NAME" '.[] | select(.name == $name) | .id')
echo "DB_ID: $DB_ID"
```

```bash
ghost sql "$DB_ID" "
CREATE TABLE IF NOT EXISTS $TABLE_NAME (
  id BIGSERIAL PRIMARY KEY,
  folder_path TEXT,
  title TEXT NOT NULL,
  url TEXT NOT NULL,
  add_date TIMESTAMPTZ,
  db_added_at TIMESTAMPTZ DEFAULT NOW()
);
ALTER TABLE $TABLE_NAME ADD COLUMN IF NOT EXISTS db_added_at TIMESTAMPTZ DEFAULT NOW();
"
```

```bash
ghost psql "$DB_ID" -- <<PSQL
\copy $TABLE_NAME (folder_path, title, url, add_date) FROM '$CSV_PATH' WITH (FORMAT csv, HEADER true)
PSQL
```


## Remove duplicates

```bash
ghost sql "$DB_ID" "
DELETE FROM $TABLE_NAME a USING $TABLE_NAME b 
WHERE a.id > b.id AND a.url = b.url;
"
```

## Verify rows

```bash
ghost sql "$DB_ID" "SELECT COUNT(*) FROM $TABLE_NAME;"
```
