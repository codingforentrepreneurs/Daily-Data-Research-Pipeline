---
title: Hacker News Top Stories
runtime: bash
---

# Hacker News Top Stories

Collect top Hacker News stories from the official Firebase API, normalize to CSV, load into Ghost Postgres, and verify.


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
export RAW_DIR="raw/hacker-news"
export TMP_DIR="/tmp/delacruz-hacker-news"

export HN_API_BASE="https://hacker-news.firebaseio.com/v0"
export HN_LIMIT="100"
export HN_IDS_JSON="$RAW_DIR/topstories_ids.json"
export HN_ITEMS_NDJSON="$RAW_DIR/topstories_items.ndjson"
export HN_STORIES_JSON="$RAW_DIR/topstories.json"
export HN_TOP_STORIES_CSV="$TMP_DIR/topstories.csv"
```

```bash
mkdir -p "$TMP_DIR"
mkdir -p "$RAW_DIR"
```


## Download Hacker News Top Stories

### Check required tools
```bash
for tool in curl jq; do
  if ! command -v "$tool" >/dev/null 2>&1; then
    echo "Missing required tool: $tool"
    exit 1
  fi
done
```

### Export top story IDs
```bash
curl -fsSL "$HN_API_BASE/topstories.json" > "$HN_IDS_JSON"
```

### Export story items
```bash
: > "$HN_ITEMS_NDJSON"

jq -r ".[:$HN_LIMIT][]" "$HN_IDS_JSON" | while read -r HN_ID; do
  curl -fsSL "$HN_API_BASE/item/$HN_ID.json"
  echo
done > "$HN_ITEMS_NDJSON"

jq -s 'map(select(. != null))' "$HN_ITEMS_NDJSON" > "$HN_STORIES_JSON"
```


## Validate Inputs

```bash
for f in "$HN_IDS_JSON" "$HN_ITEMS_NDJSON" "$HN_STORIES_JSON"; do
  if [ ! -f "$f" ]; then
    echo "Missing required file: $f"
    exit 1
  fi
done
```


## Normalize JSON to CSV

### Top stories CSV
```bash
jq -r '
  def story_url:
    .url // ("https://news.ycombinator.com/item?id=" + (.id | tostring));

  def source:
    if (.url // "") == "" then
      "news.ycombinator.com"
    else
      (try (.url | capture("^(?:https?://)?(?:www\\.)?(?<host>[^/:?#]+)").host) catch null)
    end;

  [
    "hn_id","rank","title","url","source","hn_type","author","score",
    "comment_count","hn_time","comments_url","fetched_at"
  ],
  (
    to_entries[] | [
      .value.id,
      (.key + 1),
      .value.title,
      (.value | story_url),
      (.value | source),
      .value.type,
      .value.by,
      .value.score,
      .value.descendants,
      (if .value.time then (.value.time | strftime("%Y-%m-%dT%H:%M:%SZ")) else "" end),
      ("https://news.ycombinator.com/item?id=" + (.value.id | tostring)),
      (now | strftime("%Y-%m-%dT%H:%M:%SZ"))
    ]
  )
  | @csv
' "$HN_STORIES_JSON" > "$HN_TOP_STORIES_CSV"
```


## Verify CSV

```bash
head "$HN_TOP_STORIES_CSV"
```


## Ghost Database

```bash
export DB_ID=$(ghost list --json | jq -r --arg name "$GHOST_NAME" '.[] | select(.name == $name) | .id')
export PG_HOST=$(ghost connect $DB_ID)
echo "DB_ID: $DB_ID"
```

```bash
psql "$PG_HOST" <<SQL
CREATE TABLE IF NOT EXISTS hacker_news_top_stories (
  id BIGSERIAL PRIMARY KEY,
  hn_id BIGINT NOT NULL,
  rank INTEGER,
  title TEXT NOT NULL,
  url TEXT NOT NULL,
  source TEXT,
  hn_type TEXT,
  author TEXT,
  score INTEGER,
  comment_count INTEGER,
  hn_time TIMESTAMPTZ,
  comments_url TEXT NOT NULL,
  fetched_at TIMESTAMPTZ,
  db_added_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE hacker_news_top_stories ADD COLUMN IF NOT EXISTS db_added_at TIMESTAMPTZ DEFAULT NOW();
SQL
```

```bash
psql "$PG_HOST" <<PSQL
\copy hacker_news_top_stories (hn_id, rank, title, url, source, hn_type, author, score, comment_count, hn_time, comments_url, fetched_at) FROM '$HN_TOP_STORIES_CSV' WITH (FORMAT csv, HEADER true)
PSQL
```


## Remove duplicates

```bash
psql "$PG_HOST" <<SQL
DELETE FROM hacker_news_top_stories a USING hacker_news_top_stories b
WHERE a.id < b.id AND a.hn_id = b.hn_id;
SQL
```


## Verify rows

```bash
psql "$PG_HOST" -c "
SELECT 'hacker_news_top_stories' AS table_name, COUNT(*) FROM hacker_news_top_stories;
"
```

```bash
psql "$PG_HOST" -c "
SELECT rank, title, source, score, comment_count, hn_time, url
FROM hacker_news_top_stories
ORDER BY rank
LIMIT 10;
"
```
