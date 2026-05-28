---
title: YouTube Sync
runtime: bash
timeouts:
  11: 10m
continue_on_error: [11]
---

# YouTube Sync

Append raw YouTube API responses to Ghost Postgres.

Two metadata tables (`youtube_channels`, `youtube_videos`) hold stable identity
and enrichment fields. Two append-only event tables (`youtube_channel_snapshots`,
`youtube_video_snapshots`) hold timestamped raw API payloads. Projection views
(`_current`, `_velocity`) give the agent a flat, queryable surface.

No transformation in the pipeline — every derived column is a SQL view.
Enrichment (link resolution, transcripts) fills nullable columns on
`youtube_videos` via separate runbooks.


## Setup Environment

### Location Config
```bash
export CWD="$(pwd)"
export PARENT="$(dirname "$CWD")"
cd "$PARENT"
```

### Data Config
```bash
export GHOST_NAME="ghost-research"
export RAW_DIR="raw/youtube"
export TMP_DIR="/tmp/delacruz-youtube"

export CHANNELS_YML="$RAW_DIR/channels.yml"
export CHANNELS_NDJSON="$RAW_DIR/channels.ndjson"
export VIDEOS_NDJSON="$RAW_DIR/videos.ndjson"

export CHANNELS_CSV="$TMP_DIR/channels_staging.csv"
export VIDEOS_CSV="$TMP_DIR/videos_staging.csv"
```

```bash
mkdir -p "$TMP_DIR"
mkdir -p "$RAW_DIR"
```


## Check Required Tools

```bash
for tool in jq psql uv; do
  if ! command -v "$tool" >/dev/null 2>&1; then
    echo "Missing required tool: $tool"
    exit 1
  fi
done

if [ -z "${YOUTUBE_API_KEY:-}" ]; then
  echo "Missing YOUTUBE_API_KEY"
  exit 1
fi
```


## Fetch YouTube Data

Resolves each seeded handle to a channel_id, pulls recent uploads, writes two
NDJSON files. Each line is a raw API item wrapped with `captured_at`.

```bash
uv run scripts/youtube/fetch.py
```


## Validate Inputs

```bash
for f in "$CHANNELS_NDJSON" "$VIDEOS_NDJSON"; do
  if [ ! -s "$f" ]; then
    echo "Missing or empty: $f"
    exit 1
  fi
done
```


## NDJSON -> Staging CSV

### Channels CSV
```bash
jq -rc '[
  .id,
  .seed_handle,
  ((.seed_niche // []) | join("|")),
  (.seed_priority // ""),
  .captured_at,
  tojson
] | @csv' "$CHANNELS_NDJSON" > "$CHANNELS_CSV"
```

### Videos CSV
```bash
jq -rc '[.id, .channel_id, .captured_at, tojson] | @csv' "$VIDEOS_NDJSON" > "$VIDEOS_CSV"
```

```bash
wc -l "$CHANNELS_CSV" "$VIDEOS_CSV"
```


## Ghost Database

```bash
export DB_ID=$(ghost list --json | jq -r --arg name "$GHOST_NAME" '.[] | select(.name == $name) | .id')
export PG_HOST=$(ghost connect $DB_ID)
echo "DB_ID: $DB_ID"
```


## Create Tables + Views

```bash
psql "$PG_HOST" <<'SQL'
-- Metadata tables (UPSERT target)

CREATE TABLE IF NOT EXISTS youtube_channels (
  channel_id      TEXT PRIMARY KEY,
  handle          TEXT,
  niche           TEXT[],
  priority        TEXT,
  first_seen_at   TIMESTAMPTZ DEFAULT NOW(),
  last_synced_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS youtube_videos (
  video_id              TEXT PRIMARY KEY,
  channel_id            TEXT REFERENCES youtube_channels(channel_id),
  first_seen_at         TIMESTAMPTZ DEFAULT NOW(),
  last_synced_at        TIMESTAMPTZ DEFAULT NOW(),
  links                 JSONB,
  links_enriched_at     TIMESTAMPTZ,
  transcript            TEXT,
  transcript_status     TEXT,
  transcript_fetched_at TIMESTAMPTZ
);

-- Event tables (INSERT only)

CREATE TABLE IF NOT EXISTS youtube_channel_snapshots (
  id           BIGSERIAL PRIMARY KEY,
  channel_id   TEXT NOT NULL REFERENCES youtube_channels(channel_id),
  captured_at  TIMESTAMPTZ NOT NULL,
  payload      JSONB NOT NULL
);

CREATE TABLE IF NOT EXISTS youtube_video_snapshots (
  id           BIGSERIAL PRIMARY KEY,
  video_id     TEXT NOT NULL REFERENCES youtube_videos(video_id),
  captured_at  TIMESTAMPTZ NOT NULL,
  payload      JSONB NOT NULL
);

CREATE INDEX IF NOT EXISTS youtube_channel_snapshots_ch_time_idx
  ON youtube_channel_snapshots(channel_id, captured_at DESC);
CREATE INDEX IF NOT EXISTS youtube_video_snapshots_v_time_idx
  ON youtube_video_snapshots(video_id, captured_at DESC);
CREATE INDEX IF NOT EXISTS youtube_videos_channel_id_idx
  ON youtube_videos(channel_id);
CREATE INDEX IF NOT EXISTS youtube_videos_links_null_idx
  ON youtube_videos(video_id) WHERE links IS NULL;
CREATE INDEX IF NOT EXISTS youtube_videos_transcript_null_idx
  ON youtube_videos(video_id) WHERE transcript_status IS NULL;

-- Projection views

CREATE OR REPLACE VIEW youtube_channels_current AS
SELECT DISTINCT ON (c.channel_id)
  c.channel_id, c.handle, c.niche, c.priority,
  c.first_seen_at, c.last_synced_at,
  s.captured_at                                                 AS snapshot_at,
  s.payload -> 'snippet'     ->> 'title'                        AS title,
  (s.payload -> 'statistics' ->> 'subscriberCount')::BIGINT     AS subscriber_count,
  (s.payload -> 'statistics' ->> 'videoCount')::BIGINT          AS video_count,
  (s.payload -> 'statistics' ->> 'viewCount')::BIGINT           AS view_count
FROM youtube_channels c
LEFT JOIN youtube_channel_snapshots s USING (channel_id)
ORDER BY c.channel_id, s.captured_at DESC NULLS LAST;

CREATE OR REPLACE VIEW youtube_videos_current AS
SELECT DISTINCT ON (v.video_id)
  v.video_id, v.channel_id, v.first_seen_at, v.last_synced_at,
  v.links, v.links_enriched_at,
  v.transcript, v.transcript_status, v.transcript_fetched_at,
  s.captured_at                                                 AS snapshot_at,
  s.payload -> 'snippet'        ->> 'title'                     AS title,
  s.payload -> 'snippet'        ->> 'description'               AS description,
  (s.payload -> 'snippet'       ->> 'publishedAt')::TIMESTAMPTZ AS published_at,
  s.payload -> 'contentDetails' ->> 'duration'                  AS duration_iso,
  s.payload -> 'snippet'        ->> 'categoryId'                AS category_id,
  s.payload -> 'snippet'         -> 'tags'                      AS tags,
  (s.payload -> 'statistics'    ->> 'viewCount')::BIGINT        AS view_count,
  (s.payload -> 'statistics'    ->> 'likeCount')::BIGINT        AS like_count,
  (s.payload -> 'statistics'    ->> 'commentCount')::BIGINT     AS comment_count
FROM youtube_videos v
LEFT JOIN youtube_video_snapshots s USING (video_id)
ORDER BY v.video_id, s.captured_at DESC NULLS LAST;

CREATE OR REPLACE VIEW youtube_video_velocity AS
SELECT
  video_id,
  captured_at,
  (payload -> 'statistics' ->> 'viewCount')::BIGINT    AS view_count,
  (payload -> 'statistics' ->> 'likeCount')::BIGINT    AS like_count,
  (payload -> 'statistics' ->> 'commentCount')::BIGINT AS comment_count,
  (payload -> 'statistics' ->> 'viewCount')::BIGINT
    - LAG((payload -> 'statistics' ->> 'viewCount')::BIGINT)
        OVER w                                              AS views_delta,
  (payload -> 'statistics' ->> 'likeCount')::BIGINT
    - LAG((payload -> 'statistics' ->> 'likeCount')::BIGINT)
        OVER w                                              AS likes_delta,
  (payload -> 'statistics' ->> 'commentCount')::BIGINT
    - LAG((payload -> 'statistics' ->> 'commentCount')::BIGINT)
        OVER w                                              AS comments_delta,
  EXTRACT(EPOCH FROM (captured_at - LAG(captured_at) OVER w))/3600
                                                             AS hours_since_prev
FROM youtube_video_snapshots
WINDOW w AS (PARTITION BY video_id ORDER BY captured_at);
SQL
```


## Load Staging + Append Events

```bash
psql "$PG_HOST" <<SQL
BEGIN;

CREATE TEMP TABLE staging_channels (
  channel_id   TEXT,
  handle       TEXT,
  niche_pipe   TEXT,
  priority     TEXT,
  captured_at  TIMESTAMPTZ,
  payload      TEXT
) ON COMMIT DROP;

CREATE TEMP TABLE staging_videos (
  video_id     TEXT,
  channel_id   TEXT,
  captured_at  TIMESTAMPTZ,
  payload      TEXT
) ON COMMIT DROP;

\copy staging_channels FROM '$CHANNELS_CSV' WITH (FORMAT csv)
\copy staging_videos   FROM '$VIDEOS_CSV'   WITH (FORMAT csv)

-- upsert channel metadata
INSERT INTO youtube_channels (channel_id, handle, niche, priority, last_synced_at)
SELECT
  channel_id,
  handle,
  CASE WHEN niche_pipe = '' THEN NULL ELSE string_to_array(niche_pipe, '|') END,
  NULLIF(priority, ''),
  NOW()
FROM staging_channels
ON CONFLICT (channel_id) DO UPDATE
SET handle         = COALESCE(EXCLUDED.handle,   youtube_channels.handle),
    niche          = COALESCE(EXCLUDED.niche,    youtube_channels.niche),
    priority       = COALESCE(EXCLUDED.priority, youtube_channels.priority),
    last_synced_at = NOW();

-- append channel snapshot events
INSERT INTO youtube_channel_snapshots (channel_id, captured_at, payload)
SELECT channel_id, captured_at, payload::JSONB
FROM staging_channels;

-- upsert video metadata (must precede snapshot FK)
INSERT INTO youtube_videos (video_id, channel_id, last_synced_at)
SELECT video_id, channel_id, NOW()
FROM staging_videos
ON CONFLICT (video_id) DO UPDATE SET last_synced_at = NOW();

-- append video snapshot events
INSERT INTO youtube_video_snapshots (video_id, captured_at, payload)
SELECT video_id, captured_at, payload::JSONB
FROM staging_videos;

COMMIT;
SQL
```


## Verify Rows

```bash
psql "$PG_HOST" -c "
SELECT 'youtube_channels'           AS t, COUNT(*) FROM youtube_channels
UNION ALL
SELECT 'youtube_channel_snapshots',   COUNT(*) FROM youtube_channel_snapshots
UNION ALL
SELECT 'youtube_videos',              COUNT(*) FROM youtube_videos
UNION ALL
SELECT 'youtube_video_snapshots',     COUNT(*) FROM youtube_video_snapshots;
"
```

```bash
psql "$PG_HOST" -c "
SELECT handle, title, subscriber_count, video_count, snapshot_at
FROM youtube_channels_current
ORDER BY handle;
"
```

```bash
psql "$PG_HOST" -c "
SELECT
  c.handle,
  v.title,
  v.view_count,
  vel.views_delta   AS d_views,
  vel.hours_since_prev::numeric(6,1) AS d_hours,
  v.published_at
FROM youtube_videos_current v
LEFT JOIN youtube_channels c USING (channel_id)
LEFT JOIN LATERAL (
  SELECT * FROM youtube_video_velocity
  WHERE video_id = v.video_id
  ORDER BY captured_at DESC
  LIMIT 1
) vel ON TRUE
ORDER BY vel.views_delta DESC NULLS LAST
LIMIT 20;
"
```
