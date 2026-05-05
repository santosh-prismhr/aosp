# Lab — `perfetto-sql` (Day 75–80)

Curated Perfetto SQL queries for jank, CPU contention, dma-buf accounting, and binder latency. Drop these into the Perfetto UI's "Query (SQL)" tab or run via `trace_processor`.

---

## Q1 — Top janky frames per app

```sql
SELECT
  process.name                  AS app,
  COUNT(*)                       AS janks,
  ROUND(AVG(afs.dur - efs.dur)/1e6, 2) AS avg_overshoot_ms
FROM actual_frame_timeline_slice afs
JOIN expected_frame_timeline_slice efs USING (display_frame_token)
JOIN thread USING (utid)
JOIN process USING (upid)
WHERE afs.dur > efs.dur
GROUP BY app
ORDER BY janks DESC
LIMIT 20;
```

## Q2 — Long sched_blocked_reason on UI thread

```sql
SELECT ts, dur/1e6 AS dur_ms, blocked_function, name
FROM thread_state
WHERE state = 'D' AND dur > 8e6
  AND utid IN (SELECT utid FROM thread WHERE name = 'RenderThread')
ORDER BY dur DESC LIMIT 20;
```

## Q3 — Top binder transactions by latency

```sql
SELECT
  s.name                         AS txn,
  COUNT(*)                       AS n,
  ROUND(AVG(s.dur)/1e6, 2)       AS avg_ms,
  ROUND(MAX(s.dur)/1e6, 2)       AS max_ms
FROM slice s
WHERE s.name LIKE 'binder transaction%' OR s.name LIKE 'AIDL::%'
GROUP BY s.name
ORDER BY avg_ms DESC LIMIT 30;
```

## Q4 — CPU frequency residency by core

```sql
SELECT
  cpu,
  freq,
  SUM(dur)/1e9 AS seconds
FROM cpu_freq
GROUP BY cpu, freq
ORDER BY cpu, freq;
```

## Q5 — Cold-start breakdown for an app

```sql
WITH starts AS (
  SELECT ts, dur, name, upid FROM slice
  WHERE name = 'launching: com.example.app'
)
SELECT s.name, ROUND(s.dur/1e6, 2) AS ms
FROM slice s, starts
WHERE s.ts BETWEEN starts.ts AND starts.ts + starts.dur
  AND s.upid = starts.upid
  AND s.dur > 1e6
ORDER BY s.ts;
```

## Q6 — dma-buf usage by exporter

```sql
SELECT exporter, ROUND(SUM(size)/1024.0/1024.0, 1) AS mb
FROM dmabuf_alloc
GROUP BY exporter
ORDER BY mb DESC;
```

## Run

```bash
$ trace_processor t.perfetto-trace --run-metrics android_jank_cuj
$ trace_processor t.perfetto-trace -q queries/q1_jank.sql
```

