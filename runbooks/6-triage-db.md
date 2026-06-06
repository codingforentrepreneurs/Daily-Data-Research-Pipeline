---
title: Triage DB
runtime: bash
---

# Triage DB

Verify, export, show, or apply daily-research triage data in Ghost Postgres.

This workbook does not call an LLM. It is the database pipeline only:

1. `TRIAGE_ACTION=verify` checks the selected source and prints sample rows. This is the default.
2. `TRIAGE_ACTION=export` writes rows to JSONL with blank `auto_category` and `auto_summary`.
3. Codex, Claude, Hermes, or another agent goes row-by-row through that JSONL and fills those fields using model judgment.
4. `TRIAGE_ACTION=apply` writes the agent-completed JSONL back to Postgres.
5. `TRIAGE_ACTION=show` prints existing triage line-by-line.

Do not add heuristic categorization logic to this workbook. The triage judgment belongs in the invoking agent.
Agent-generated summaries should be specific and useful. Do not use generic text like `The linked story covers ...`; bad summaries are rejected during apply and should be re-triaged.

```bash
export TRIAGE_SOURCE="${TRIAGE_SOURCE:-hn}"
export GHOST_NAME="${GHOST_NAME:-daily-research}"
export TRIAGE_ACTION="${TRIAGE_ACTION:-verify}"
export TRIAGE_LIMIT="${TRIAGE_LIMIT:-500}"
export TRIAGE_EXPORT="${TRIAGE_EXPORT:-/tmp/triage-db.jsonl}"
export TRIAGE_APPLY="${TRIAGE_APPLY:-/tmp/triage-db.done.jsonl}"
export TRIAGE_AGAIN="${TRIAGE_AGAIN:-}"
export TRIAGE_ARGS="${TRIAGE_ARGS:-}"
```

```bash
if [ -n "$TRIAGE_ARGS" ]; then
  set -- $TRIAGE_ARGS
  while [ "$#" -gt 0 ]; do
    case "$1" in
      --triage-again)
        export TRIAGE_AGAIN="$2"
        shift 2
        ;;
      --limit)
        export TRIAGE_LIMIT="$2"
        shift 2
        ;;
      --db)
        export GHOST_NAME="$2"
        shift 2
        ;;
      --export)
        export TRIAGE_ACTION="export"
        export TRIAGE_EXPORT="$2"
        shift 2
        ;;
      --apply)
        export TRIAGE_ACTION="apply"
        export TRIAGE_APPLY="$2"
        shift 2
        ;;
      --action)
        export TRIAGE_ACTION="$2"
        shift 2
        ;;
      --show)
        export TRIAGE_ACTION="show"
        shift
        ;;
      --*)
        echo "Unsupported triage option: $1"
        exit 1
        ;;
      *)
        export TRIAGE_SOURCE="$1"
        shift
        ;;
    esac
  done
fi
```

```bash
case "$TRIAGE_SOURCE" in
  hn|hacker-news|hacker_news|hacker_news_top_stories)
    export TRIAGE_TABLE="hacker_news_top_stories"
    ;;
  github-stars|github_stars)
    export TRIAGE_TABLE="github_stars"
    ;;
  github-repos|github_repos)
    export TRIAGE_TABLE="github_repos"
    ;;
  github-top|github_top_repos)
    export TRIAGE_TABLE="github_top_repos"
    ;;
  github-trending|github_trending_recent)
    export TRIAGE_TABLE="github_trending_recent"
    ;;
  *)
    export TRIAGE_TABLE="$TRIAGE_SOURCE"
    ;;
esac

if ! printf '%s' "$TRIAGE_TABLE" | grep -Eq '^[A-Za-z_][A-Za-z0-9_]*$'; then
  echo "Unsafe table name: $TRIAGE_TABLE"
  exit 1
fi

case "$TRIAGE_ACTION" in
  verify|export|apply|show) ;;
  *)
    echo "TRIAGE_ACTION must be verify, export, apply, or show"
    exit 1
    ;;
esac

if ! printf '%s' "$TRIAGE_LIMIT" | grep -Eq '^[0-9]+$'; then
  echo "TRIAGE_LIMIT must be a whole number"
  exit 1
fi
```

```bash
export PG_HOST="$(ghost connect "$GHOST_NAME")"
echo "TRIAGE_TABLE=$TRIAGE_TABLE"
echo "TRIAGE_ACTION=$TRIAGE_ACTION"
```

```bash
psql "$PG_HOST" <<SQL
ALTER TABLE "$TRIAGE_TABLE"
  ADD COLUMN IF NOT EXISTS auto_category TEXT,
  ADD COLUMN IF NOT EXISTS auto_summary TEXT,
  ADD COLUMN IF NOT EXISTS auto_triaged_at TIMESTAMPTZ;
SQL
```

## Verify

Print enough information to verify the workbook outside an LLM.

```bash
if [ "$TRIAGE_ACTION" = "verify" ]; then
  psql "$PG_HOST" -X -q -c "
  SELECT
    COUNT(*) AS total_rows,
    COUNT(*) FILTER (
      WHERE coalesce(auto_category, '') = ''
         OR coalesce(auto_summary, '') = ''
    ) AS rows_needing_triage,
    COUNT(*) FILTER (
      WHERE coalesce(auto_category, '') <> ''
        AND coalesce(auto_summary, '') <> ''
    ) AS triaged_rows
  FROM \"$TRIAGE_TABLE\";
  "

  echo ""
  echo "Rows needing triage:"
  psql "$PG_HOST" -X -q -t -A -c "
  SELECT concat_ws(
    ' | ',
    id::TEXT,
    coalesce(to_jsonb(src)->>'title', to_jsonb(src)->>'full_name', to_jsonb(src)->>'name', '(untitled)')
  ) AS row_preview
  FROM \"$TRIAGE_TABLE\" src
  WHERE coalesce(auto_category, '') = ''
     OR coalesce(auto_summary, '') = ''
  ORDER BY id DESC
  LIMIT LEAST($TRIAGE_LIMIT, 10);
  "

  echo ""
  echo "Existing triage:"
  psql "$PG_HOST" -X -q -t -A -c "
  SELECT concat_ws(
    ' | ',
    id::TEXT,
    coalesce(to_jsonb(src)->>'title', to_jsonb(src)->>'full_name', to_jsonb(src)->>'name', '(untitled)'),
    auto_category,
    auto_summary
  ) AS triage
  FROM \"$TRIAGE_TABLE\" src
  WHERE coalesce(auto_category, '') <> ''
    AND coalesce(auto_summary, '') <> ''
  ORDER BY auto_triaged_at DESC NULLS LAST, id
  LIMIT LEAST($TRIAGE_LIMIT, 10);
  "
fi
```

## Export Rows

This exports rows that need agent judgment. Without `TRIAGE_AGAIN`, it only exports rows missing `auto_category` or `auto_summary`. With `TRIAGE_AGAIN=30d`, it re-exports recent rows using the best available timestamp column.

```bash
if [ "$TRIAGE_ACTION" = "export" ]; then
  export TRIAGE_COLUMNS="$(psql "$PG_HOST" -X -q -t -A -c "
  SELECT string_agg(column_name, ',' ORDER BY ordinal_position)
  FROM information_schema.columns
  WHERE table_schema = 'public' AND table_name = '$TRIAGE_TABLE';
  ")"
else
  echo "Skipping export."
fi

choose_col() {
  for col in "$@"; do
    if printf ',%s,' "$TRIAGE_COLUMNS" | grep -q ",$col,"; then
      printf '%s' "$col"
      return 0
    fi
  done
}

if [ "$TRIAGE_ACTION" = "export" ]; then
  export TITLE_COL="$(choose_col title full_name name)"
  export RECENCY_COL="$(choose_col db_added_at fetched_at hn_time updated_at pushed_at created_at)"
  export ORDER_COL="$(choose_col score stargazers_count id)"

  if [ -z "$TITLE_COL" ]; then
    echo "No title-like column found for $TRIAGE_TABLE"
    exit 1
  fi

  if [ -n "$RECENCY_COL" ] && printf ',%s,' "$TRIAGE_COLUMNS" | grep -q ',rank,'; then
    export TRIAGE_ORDER="\"$RECENCY_COL\" DESC NULLS LAST, \"rank\" ASC NULLS LAST"
  elif [ -n "$RECENCY_COL" ]; then
    export TRIAGE_ORDER="\"$RECENCY_COL\" DESC NULLS LAST"
  elif printf ',%s,' "$TRIAGE_COLUMNS" | grep -q ',rank,'; then
    export TRIAGE_ORDER="\"rank\" ASC NULLS LAST"
  else
    if [ -z "$ORDER_COL" ]; then
      export ORDER_COL="id"
    fi
    export TRIAGE_ORDER="\"$ORDER_COL\" DESC NULLS LAST"
  fi

  export TRIAGE_WHERE="(auto_category IS NULL OR auto_summary IS NULL OR auto_category = '' OR auto_summary = '')"
  if [ -n "$TRIAGE_AGAIN" ]; then
    if ! printf '%s' "$TRIAGE_AGAIN" | grep -Eq '^[0-9]+[dwmy]$'; then
      echo "TRIAGE_AGAIN must look like 7d, 4w, 6m, or 1y"
      exit 1
    fi

    if [ -z "$RECENCY_COL" ]; then
      export TRIAGE_WHERE="TRUE"
    else
      export TRIAGE_AMOUNT="${TRIAGE_AGAIN%?}"
      export TRIAGE_UNIT="${TRIAGE_AGAIN#${TRIAGE_AMOUNT}}"
      case "$TRIAGE_UNIT" in
        d) export TRIAGE_DAYS="$TRIAGE_AMOUNT" ;;
        w) export TRIAGE_DAYS="$((TRIAGE_AMOUNT * 7))" ;;
        m) export TRIAGE_DAYS="$((TRIAGE_AMOUNT * 30))" ;;
        y) export TRIAGE_DAYS="$((TRIAGE_AMOUNT * 365))" ;;
      esac
      export TRIAGE_WHERE="\"$RECENCY_COL\" >= now() - interval '$TRIAGE_DAYS days'"
    fi
  fi

  psql "$PG_HOST" -X -q -t -A -F $'\t' -c "
  COPY (
    SELECT jsonb_build_object(
      'id', id,
      'item', to_jsonb(src) - 'auto_category' - 'auto_summary' - 'auto_triaged_at',
      'auto_category', '',
      'auto_summary', ''
    )::text
    FROM (
      SELECT *
      FROM \"$TRIAGE_TABLE\"
      WHERE $TRIAGE_WHERE
      ORDER BY $TRIAGE_ORDER
      LIMIT $TRIAGE_LIMIT
    ) src
  ) TO STDOUT WITH (FORMAT csv, DELIMITER E'\x02', QUOTE E'\x01', ESCAPE E'\x01');
  " > "$TRIAGE_EXPORT"

  echo "Exported $(wc -l < "$TRIAGE_EXPORT" | tr -d ' ') rows to $TRIAGE_EXPORT"
fi
```

## Apply Completed Triage

The active agent should read `TRIAGE_EXPORT`, fill `auto_category` and `auto_summary` for each JSONL row, write the completed rows to `TRIAGE_APPLY`, then run this workbook with `TRIAGE_ACTION=apply`.

Completed JSONL format:

```json
{"id":5333,"auto_category":"technology, browser, open source","auto_summary":"Ladybird describes changes to its development model for the open source browser project."}
```

```bash
if [ "$TRIAGE_ACTION" = "apply" ]; then
  if [ ! -s "$TRIAGE_APPLY" ]; then
    echo "Missing completed triage file: $TRIAGE_APPLY"
    exit 1
  fi
else
  echo "Skipping apply."
fi

if [ "$TRIAGE_ACTION" = "apply" ]; then
  psql "$PG_HOST" <<SQL
CREATE TEMP TABLE triage_decisions_raw (
  raw JSONB NOT NULL
);

\copy triage_decisions_raw (raw) FROM '$TRIAGE_APPLY' WITH (FORMAT csv, DELIMITER E'\x02', QUOTE E'\x01', ESCAPE E'\x01')

CREATE TEMP TABLE triage_applied_ids (
  id BIGINT PRIMARY KEY
);

DO \$\$
DECLARE
  bad_count INTEGER;
BEGIN
  SELECT COUNT(*)
  INTO bad_count
  FROM triage_decisions_raw
  WHERE coalesce(btrim(raw->>'auto_category'), '') = ''
     OR coalesce(btrim(raw->>'auto_summary'), '') = ''
     OR raw->>'auto_summary' ILIKE 'The linked story covers %'
     OR raw->>'auto_summary' ILIKE 'A GitHub project covers %'
     OR raw->>'auto_summary' ILIKE 'Show HN presents %'
     OR raw->>'auto_summary' ILIKE 'A research-oriented item explores %'
     OR raw->>'auto_summary' ILIKE 'This article covers %'
     OR raw->>'auto_summary' ILIKE 'This post covers %';

  IF bad_count > 0 THEN
    RAISE EXCEPTION 'Refusing to apply % empty or generic triage decisions. The agent must write specific row-by-row auto_category and auto_summary values.', bad_count;
  END IF;
END \$\$;

WITH updated AS (
UPDATE "$TRIAGE_TABLE" target
SET auto_category = btrim(decision.raw->>'auto_category'),
    auto_summary = btrim(decision.raw->>'auto_summary'),
    auto_triaged_at = now()
FROM triage_decisions_raw decision
WHERE target.id = (decision.raw->>'id')::BIGINT
  AND coalesce(btrim(decision.raw->>'auto_category'), '') <> ''
  AND coalesce(btrim(decision.raw->>'auto_summary'), '') <> ''
RETURNING target.id
)
INSERT INTO triage_applied_ids (id)
SELECT id FROM updated;

SELECT COUNT(*) AS applied_rows
FROM triage_applied_ids;

SELECT concat_ws(
  ' | ',
  target.id::TEXT,
  coalesce(to_jsonb(target)->>'title', to_jsonb(target)->>'full_name', to_jsonb(target)->>'name', '(untitled)'),
  target.auto_category,
  target.auto_summary
) AS applied_triage
FROM "$TRIAGE_TABLE" target
JOIN triage_applied_ids applied ON applied.id = target.id
ORDER BY target.id;
SQL
fi
```

## Show Triage

Print existing triage line-by-line without changing rows.

```bash
if [ "$TRIAGE_ACTION" = "show" ]; then
  psql "$PG_HOST" -X -q -t -A -c "
  SELECT concat_ws(
    ' | ',
    id::TEXT,
    coalesce(to_jsonb(src)->>'title', to_jsonb(src)->>'full_name', to_jsonb(src)->>'name', '(untitled)'),
    auto_category,
    auto_summary
  ) AS triage
  FROM \"$TRIAGE_TABLE\" src
  WHERE coalesce(auto_category, '') <> ''
    AND coalesce(auto_summary, '') <> ''
  ORDER BY auto_triaged_at DESC NULLS LAST, id
  LIMIT $TRIAGE_LIMIT;
  "
fi
```
