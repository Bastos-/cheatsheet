# Boated index

- [Boated index](#boated-index)
  - [Why](#why)
  - [Find bloated indexes](#find-bloated-indexes)
  - [Fix them](#fix)
    - [Reindex a single index.](#reindex-a-single-index)
    - [Reindex all indexes for a specific table.](#reindex-all-indexes-for-a-specific-table)
  - [Monitor during the reindex](#monitor)

## Why
Use space unnecessarily.

An index with too much bloat can deteriorate performance.

Only BTREE index can be reindexed.


## Find bloated indexes

```sql
SELECT current_database(), nspname AS schemaname, tblname, idxname, bs*(relpages)::bigint AS real_size,
  bs*(relpages-est_pages)::bigint AS extra_size,
  100 * (relpages-est_pages)::float / relpages AS extra_ratio,
  fillfactor, bs*(relpages-est_pages_ff) AS bloat_size,
  100 * (relpages-est_pages_ff)::float / relpages AS bloat_ratio,
  is_na
  -- , 100-(sub.pst).avg_leaf_density, est_pages, index_tuple_hdr_bm, 
  -- maxalign, pagehdr, nulldatawidth, nulldatahdrwidth, sub.reltuples, sub.relpages 
  -- (DEBUG INFO)
FROM (
  SELECT coalesce(1 +
       ceil(reltuples/floor((bs-pageopqdata-pagehdr)/(4+nulldatahdrwidth)::float)), 0 
       -- ItemIdData size + computed avg size of a tuple (nulldatahdrwidth)
    ) AS est_pages,
    coalesce(1 +
       ceil(reltuples/floor((bs-pageopqdata-pagehdr)*fillfactor/(100*(4+nulldatahdrwidth)::float))), 0
    ) AS est_pages_ff,
    bs, nspname, table_oid, tblname, idxname, relpages, fillfactor, is_na
    -- , stattuple.pgstatindex(quote_ident(nspname)||'.'||quote_ident(idxname)) AS pst, 
    -- index_tuple_hdr_bm, maxalign, pagehdr, nulldatawidth, nulldatahdrwidth, reltuples 
    -- (DEBUG INFO)
  FROM (
    SELECT maxalign, bs, nspname, tblname, idxname, reltuples, relpages, relam, table_oid, fillfactor,
      ( index_tuple_hdr_bm +
          maxalign - CASE -- Add padding to the index tuple header to align on MAXALIGN
            WHEN index_tuple_hdr_bm%maxalign = 0 THEN maxalign
            ELSE index_tuple_hdr_bm%maxalign
          END
        + nulldatawidth + maxalign - CASE -- Add padding to the data to align on MAXALIGN
            WHEN nulldatawidth = 0 THEN 0
            WHEN nulldatawidth::integer%maxalign = 0 THEN maxalign
            ELSE nulldatawidth::integer%maxalign
          END
      )::numeric AS nulldatahdrwidth, pagehdr, pageopqdata, is_na
      -- , index_tuple_hdr_bm, nulldatawidth -- (DEBUG INFO)
    FROM (
      SELECT
        i.nspname, i.tblname, i.idxname, i.reltuples, i.relpages, i.relam, a.attrelid AS table_oid,
        current_setting('block_size')::numeric AS bs, fillfactor,
        CASE -- MAXALIGN: 4 on 32bits, 8 on 64bits (and mingw32 ?)
          WHEN version() ~ 'mingw32' OR version() ~ '64-bit|x86_64|ppc64|ia64|amd64' THEN 8
          ELSE 4
        END AS maxalign,
        /* per page header, fixed size: 20 for 7.X, 24 for others */
        24 AS pagehdr,
        /* per page btree opaque data */
        16 AS pageopqdata,
        /* per tuple header: add IndexAttributeBitMapData if some cols are null-able */
        CASE WHEN max(coalesce(s.null_frac,0)) = 0
          THEN 2 -- IndexTupleData size
          ELSE 2 + (( 32 + 8 - 1 ) / 8) 
          -- IndexTupleData size + IndexAttributeBitMapData size ( max num filed per index + 8 - 1 /8)
        END AS index_tuple_hdr_bm,
        /* data len: we remove null values save space using it fractionnal part from stats */
        sum( (1-coalesce(s.null_frac, 0)) * coalesce(s.avg_width, 1024)) AS nulldatawidth,
        max( CASE WHEN a.atttypid = 'pg_catalog.name'::regtype THEN 1 ELSE 0 END ) > 0 AS is_na
      FROM pg_attribute AS a
        JOIN (
          SELECT nspname, tbl.relname AS tblname, idx.relname AS idxname, 
            idx.reltuples, idx.relpages, idx.relam,
            indrelid, indexrelid, indkey::smallint[] AS attnum,
            coalesce(substring(
              array_to_string(idx.reloptions, ' ')
               from 'fillfactor=([0-9]+)')::smallint, 90) AS fillfactor
          FROM pg_index
            JOIN pg_class idx ON idx.oid=pg_index.indexrelid
            JOIN pg_class tbl ON tbl.oid=pg_index.indrelid
            JOIN pg_namespace ON pg_namespace.oid = idx.relnamespace
          WHERE pg_index.indisvalid AND tbl.relkind = 'r' AND idx.relpages > 0
        ) AS i ON a.attrelid = i.indexrelid
        JOIN pg_stats AS s ON s.schemaname = i.nspname
          AND ((s.tablename = i.tblname AND s.attname = pg_catalog.pg_get_indexdef(a.attrelid, a.attnum, TRUE)) 
          -- stats from tbl
          OR  (s.tablename = i.idxname AND s.attname = a.attname))
          -- stats from functional cols
        JOIN pg_type AS t ON a.atttypid = t.oid
      WHERE a.attnum > 0
      GROUP BY 1, 2, 3, 4, 5, 6, 7, 8, 9
    ) AS s1
  ) AS s2
    JOIN pg_am am ON s2.relam = am.oid WHERE am.amname = 'btree'
) AS sub
-- WHERE NOT is_na
ORDER BY 2,3,4;
```
Shameless copy/paste to get the bloat status.


## FIX

### Reindex a single index.
```sql
REINDEX index CONCURRENTLY index_name;
```

### Reindex all indexes for a specific table.
```sql
REINDEX TABLE CONCURRENTLY table_name;
```

CONCURRENTLY parameter will rebuild the index without taking any locks that prevent concurrent inserts, updates, or deletes on the table. No downtime needed with this option.

## Monitor

```sql
SELECT 
       clock_timestamp() - a.xact_start                          AS duration_so_far,
       coalesce(a.wait_event_type ||'.'|| a.wait_event, 'false') AS waiting,
       a.state,
       p.phase,
       CASE p.phase
           WHEN 'initializing' THEN '1 of 12'
           WHEN 'waiting for writers before build' THEN '2 of 12'
           WHEN 'building index: scanning table' THEN '3 of 12'
           WHEN 'building index: sorting live tuples' THEN '4 of 12'
           WHEN 'building index: loading tuples in tree' THEN '5 of 12'
           WHEN 'waiting for writers before validation' THEN '6 of 12'
           WHEN 'index validation: scanning index' THEN '7 of 12'
           WHEN 'index validation: sorting tuples' THEN '8 of 12'
           WHEN 'index validation: scanning table' THEN '9 of 12'
           WHEN 'waiting for old snapshots' THEN '10 of 12'
           WHEN 'waiting for readers before marking dead' THEN '11 of 12'
           WHEN 'waiting for readers before dropping' THEN '12 of 12'
       END AS phase_progress,
       format(
           '%s (%s of %s)',
           coalesce(round(100.0 * p.blocks_done / nullif(p.blocks_total, 0), 2)::text || '%', 'not applicable'),
           p.blocks_done::text,
           p.blocks_total::text
       ) AS scan_progress,
       format(
           '%s (%s of %s)',
           coalesce(round(100.0 * p.tuples_done / nullif(p.tuples_total, 0), 2)::text || '%', 'not applicable'),
           p.tuples_done::text,
           p.tuples_total::text
       ) AS tuples_loading_progress,
       format(
           '%s (%s of %s)',
           coalesce((100 * p.lockers_done / nullif(p.lockers_total, 0))::text || '%', 'not applicable'),
           p.lockers_done::text,
           p.lockers_total::text
       ) AS lockers_progress,
       format(
           '%s (%s of %s)',
           coalesce((100 * p.partitions_done / nullif(p.partitions_total, 0))::text || '%', 'not applicable'),
           p.partitions_done::text,
           p.partitions_total::text
       ) AS partitions_progress,
       p.current_locker_pid,
       trim(trailing ';' from l.query) AS current_locker_query
  FROM pg_stat_progress_create_index   AS p
  JOIN pg_stat_activity                AS a ON a.pid = p.pid
  LEFT JOIN pg_stat_activity           AS l ON l.pid = p.current_locker_pid
 ORDER BY clock_timestamp() - a.xact_start DESC;
```
You call follow the process and its different phases, ([More info](https://www.postgresql.org/docs/current/progress-reporting.html#CREATE-INDEX-PROGRESS-REPORTING))


```sql
SELECT 
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocked_activity.query AS blocked_query,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocking_activity.query AS blocking_query,
    blocking_activity.state AS blocking_state,
    age(blocking_activity.query_start) AS blocking_age
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks 
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.DATABASE IS NOT DISTINCT FROM blocked_locks.DATABASE
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
    AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
    AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
    AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
    AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
    AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
    AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
    AND blocked_locks.GRANTED = false
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE blocked_activity.pid <> blocking_activity.pid;
```
The REINDEX CONCURRENTLY command can be blocked by another transaction. You can monitor such locks with the following query.