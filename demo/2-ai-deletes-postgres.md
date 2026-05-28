---
title: AI Deletes Postgres Data
runtime: bash
---

# AI Deletes Postgres Data

Use the forked Ghost database from `demo/1-fork-db.md`, drop the public tables on the fork, and prove the original `remember-me` database still has its tables and data.

This simulates an agent making a broad destructive change while connected to a forked Postgres database instead of the source database.


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

### Load Fork State
```bash
export DEMO_STATE_FILE="$REPO_ROOT/demo/.ghost-fork.env"

if [ ! -f "$DEMO_STATE_FILE" ]; then
  echo "Missing $DEMO_STATE_FILE"
  echo "Run first: wb demo/1-fork-db.md"
  exit 1
fi

set -a
. "$DEMO_STATE_FILE"
set +a

echo "Source database: $GHOST_SOURCE_NAME ($GHOST_SOURCE_DB_ID)"
echo "Fork database: $GHOST_FORK_NAME ($GHOST_FORK_DB_ID)"
```

### Check required tools
```bash
for tool in ghost psql; do
  if ! command -v "$tool" >/dev/null 2>&1; then
    echo "Missing required tool: $tool"
    exit 1
  fi
done
```


## Connect To Source And Fork

```bash
export SOURCE_PG_HOST=$(ghost connect "$GHOST_SOURCE_DB_ID")
export FORK_PG_HOST=$(ghost connect "$GHOST_FORK_DB_ID")
```


## Show Row Counts Before Delete

### Source before delete
```bash
psql "$SOURCE_PG_HOST" <<'SQL'
CREATE TEMP TABLE demo_table_counts (
  database_name TEXT,
  schema_name TEXT,
  table_name TEXT,
  exact_rows BIGINT
);

DO $$
DECLARE
  table_record record;
  row_count BIGINT;
BEGIN
  FOR table_record IN
    SELECT schemaname, tablename
    FROM pg_tables
    WHERE schemaname = 'public'
    ORDER BY tablename
  LOOP
    EXECUTE format('SELECT COUNT(*) FROM %I.%I', table_record.schemaname, table_record.tablename) INTO row_count;
    INSERT INTO demo_table_counts VALUES ('source', table_record.schemaname, table_record.tablename, row_count);
  END LOOP;
END $$;

SELECT *
FROM demo_table_counts
ORDER BY table_name;
SQL
```

### Fork before delete
```bash
psql "$FORK_PG_HOST" <<'SQL'
CREATE TEMP TABLE demo_table_counts (
  database_name TEXT,
  schema_name TEXT,
  table_name TEXT,
  exact_rows BIGINT
);

DO $$
DECLARE
  table_record record;
  row_count BIGINT;
BEGIN
  FOR table_record IN
    SELECT schemaname, tablename
    FROM pg_tables
    WHERE schemaname = 'public'
    ORDER BY tablename
  LOOP
    EXECUTE format('SELECT COUNT(*) FROM %I.%I', table_record.schemaname, table_record.tablename) INTO row_count;
    INSERT INTO demo_table_counts VALUES ('fork', table_record.schemaname, table_record.tablename, row_count);
  END LOOP;
END $$;

SELECT *
FROM demo_table_counts
ORDER BY table_name;
SQL
```


## Simulate The Accidental Agent Drop

```bash
psql "$FORK_PG_HOST" <<'SQL'
DO $$
DECLARE
  table_names TEXT;
BEGIN
  SELECT string_agg(format('%I.%I', schemaname, tablename), ', ' ORDER BY tablename)
  INTO table_names
  FROM pg_tables
  WHERE schemaname = 'public';

  IF table_names IS NOT NULL THEN
    EXECUTE 'DROP TABLE ' || table_names || ' CASCADE';
  END IF;
END $$;
SQL
```


## Prove The Fork Was Deleted

```bash
psql "$FORK_PG_HOST" -c "
SELECT 'fork' AS database, COUNT(*) AS public_tables
FROM information_schema.tables
WHERE table_schema = 'public'
  AND table_type = 'BASE TABLE';
"

psql "$FORK_PG_HOST" <<'SQL'
CREATE TEMP TABLE demo_table_counts (
  database_name TEXT,
  schema_name TEXT,
  table_name TEXT,
  exact_rows BIGINT
);

DO $$
DECLARE
  table_record record;
  row_count BIGINT;
BEGIN
  FOR table_record IN
    SELECT schemaname, tablename
    FROM pg_tables
    WHERE schemaname = 'public'
    ORDER BY tablename
  LOOP
    EXECUTE format('SELECT COUNT(*) FROM %I.%I', table_record.schemaname, table_record.tablename) INTO row_count;
    INSERT INTO demo_table_counts VALUES ('fork', table_record.schemaname, table_record.tablename, row_count);
  END LOOP;
END $$;

SELECT *
FROM demo_table_counts
ORDER BY table_name;
SQL
```


## Prove The Source Database Still Has Data

```bash
psql "$SOURCE_PG_HOST" <<'SQL'
CREATE TEMP TABLE demo_table_counts (
  database_name TEXT,
  schema_name TEXT,
  table_name TEXT,
  exact_rows BIGINT
);

DO $$
DECLARE
  table_record record;
  row_count BIGINT;
BEGIN
  FOR table_record IN
    SELECT schemaname, tablename
    FROM pg_tables
    WHERE schemaname = 'public'
    ORDER BY tablename
  LOOP
    EXECUTE format('SELECT COUNT(*) FROM %I.%I', table_record.schemaname, table_record.tablename) INTO row_count;
    INSERT INTO demo_table_counts VALUES ('source', table_record.schemaname, table_record.tablename, row_count);
  END LOOP;
END $$;

SELECT *
FROM demo_table_counts
ORDER BY table_name;
SQL
```


## Compare Source And Fork

```bash
psql "$SOURCE_PG_HOST" -c "
SELECT 'source' AS database, COUNT(*) AS public_tables
FROM information_schema.tables
WHERE table_schema = 'public'
  AND table_type = 'BASE TABLE';
"

psql "$FORK_PG_HOST" -c "
SELECT 'fork' AS database, COUNT(*) AS public_tables
FROM information_schema.tables
WHERE table_schema = 'public'
  AND table_type = 'BASE TABLE';
"
```
