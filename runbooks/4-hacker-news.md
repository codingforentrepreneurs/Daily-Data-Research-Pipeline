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
export GHOST_NAME="${GHOST_NAME:-daily-research}"
export RAW_DIR="raw/hacker-news"
export TMP_DIR="/tmp/delacruz-hacker-news"

export HN_API_BASE="https://hacker-news.firebaseio.com/v0"
export HN_SEARCH_API_BASE="https://hn.algolia.com/api/v1"
export HN_LOOKBACK="${HN_LOOKBACK:-today}"
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

### Choose story window
```bash
case "$HN_LOOKBACK" in
  today|current|0|0d)
    export HN_FETCH_MODE="current"
    export HN_LOOKBACK_DAYS="0"
    ;;
  1d|day)
    export HN_FETCH_MODE="historical"
    export HN_LOOKBACK_DAYS="1"
    ;;
  7d|1w|week)
    export HN_FETCH_MODE="historical"
    export HN_LOOKBACK_DAYS="7"
    ;;
  14d|2w)
    export HN_FETCH_MODE="historical"
    export HN_LOOKBACK_DAYS="14"
    ;;
  30d|month)
    export HN_FETCH_MODE="historical"
    export HN_LOOKBACK_DAYS="30"
    ;;
  90d|quarter)
    export HN_FETCH_MODE="historical"
    export HN_LOOKBACK_DAYS="90"
    ;;
  365d|year|1y|1yr|1yrs)
    export HN_FETCH_MODE="historical"
    export HN_LOOKBACK_DAYS="365"
    ;;
  2y|2yr|2yrs)
    export HN_FETCH_MODE="historical"
    export HN_LOOKBACK_DAYS="730"
    ;;
  *d)
    export HN_FETCH_MODE="historical"
    export HN_LOOKBACK_DAYS="${HN_LOOKBACK%d}"
    ;;
  *w)
    export HN_FETCH_MODE="historical"
    export HN_LOOKBACK_DAYS="$((${HN_LOOKBACK%w} * 7))"
    ;;
  *y)
    export HN_FETCH_MODE="historical"
    export HN_LOOKBACK_DAYS="$((${HN_LOOKBACK%y} * 365))"
    ;;
  *yr)
    export HN_FETCH_MODE="historical"
    export HN_LOOKBACK_DAYS="$((${HN_LOOKBACK%yr} * 365))"
    ;;
  *yrs)
    export HN_FETCH_MODE="historical"
    export HN_LOOKBACK_DAYS="$((${HN_LOOKBACK%yrs} * 365))"
    ;;
  *)
    echo "Unsupported HN_LOOKBACK: $HN_LOOKBACK"
    echo "Use one of: today, 1d, 7d, 14d, 30d, 90d, 365d, 2yrs, or a custom value like 3d, 2w, or 1y."
    exit 1
    ;;
esac

if ! [[ "$HN_LOOKBACK_DAYS" =~ ^[0-9]+$ ]]; then
  echo "HN_LOOKBACK must resolve to a whole number of days."
  exit 1
fi

export HN_NOW_TS="$(date -u +%s)"
export HN_START_TS="$((HN_NOW_TS - (HN_LOOKBACK_DAYS * 86400)))"

echo "HN_LOOKBACK=$HN_LOOKBACK"
echo "HN_FETCH_MODE=$HN_FETCH_MODE"
```

### Export top story IDs
```bash
if [ "$HN_FETCH_MODE" = "current" ]; then
  curl -fsSL "$HN_API_BASE/topstories.json" > "$HN_IDS_JSON"
else
  jq -n '[]' > "$HN_IDS_JSON"
fi
```

### Export story items
```bash
: > "$HN_ITEMS_NDJSON"

if [ "$HN_FETCH_MODE" = "current" ]; then
  jq -r ".[:$HN_LIMIT][]" "$HN_IDS_JSON" | while read -r HN_ID; do
    curl -fsSL "$HN_API_BASE/item/$HN_ID.json"
    echo
  done > "$HN_ITEMS_NDJSON"

  jq -s 'map(select(. != null))' "$HN_ITEMS_NDJSON" > "$HN_STORIES_JSON"
else
  curl -fsSLG "$HN_SEARCH_API_BASE/search" \
    --data-urlencode "tags=story" \
    --data-urlencode "hitsPerPage=$HN_LIMIT" \
    --data-urlencode "numericFilters=created_at_i>=$HN_START_TS,created_at_i<=$HN_NOW_TS" \
    > "$HN_STORIES_JSON"

  jq -c '.hits[]' "$HN_STORIES_JSON" > "$HN_ITEMS_NDJSON"
fi
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
  def hn_id:
    .id // .objectID;

  def story_url:
    .url // ("https://news.ycombinator.com/item?id=" + (hn_id | tostring));

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
    (if type == "array" then . else .hits end) | to_entries[] | [
      (.value | hn_id),
      (.key + 1),
      .value.title,
      (.value | story_url),
      (.value | source),
      (.value.type // "story"),
      (.value.by // .value.author),
      (.value.score // .value.points),
      (.value.descendants // .value.num_comments),
      (if .value.time then (.value.time | strftime("%Y-%m-%dT%H:%M:%SZ")) elif .value.created_at then .value.created_at else "" end),
      ("https://news.ycombinator.com/item?id=" + ((.value | hn_id) | tostring)),
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
export DB_ID=$(ghost list --json | jq -r --arg name "$GHOST_NAME" '.[] | select(.name == $name) | .id' | head -1)
if [ -z "$DB_ID" ]; then
  ghost create --name "$GHOST_NAME" --wait
  export DB_ID=$(ghost list --json | jq -r --arg name "$GHOST_NAME" '.[] | select(.name == $name) | .id' | head -1)
fi

if [ -z "$DB_ID" ]; then
  echo "Could not find or create Ghost database: $GHOST_NAME"
  exit 1
fi

export PG_HOST=$(ghost connect "$DB_ID")
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
