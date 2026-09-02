# PostgreSQL VACUUM

PostgreSQL `VACUUM` is a maintenance operation that cleans up dead rows left behind by updates and deletes.

In PostgreSQL, when a row is updated or deleted, the old row version is not removed immediately because of MVCC (Multi-Version Concurrency Control). `VACUUM` removes those obsolete row versions so space can be reused, and table performance stays healthy.

`VACUUM` scans a table for dead tuples, which are old row versions left behind by `UPDATE`/`DELETE`, and marks that space as reusable by Postgres for future inserts and updates.

## Types of VACUUM

### 1. Plain / Standard VACUUM

Removes dead tuples and makes the space reusable inside PostgreSQL. It does **not** return disk space to the operating system.

```sql
VACUUM table_name;
```

### 2. VACUUM FULL

Rewrites the entire table and releases unused disk space back to the operating system. It is slower and requires an exclusive lock on the table, which makes it unsuitable for use on tables that need to stay available.

```sql
VACUUM FULL table_name;
```

### 3. AUTOVACUUM

Runs automatically in the background, cleaning dead tuples and keeping the database in shape with no manual effort needed. It works well for most cases, but large or heavily used tables may still need manual vacuuming or tuned settings.

## Why VACUUM Is Necessary

### Reclaims storage space

`VACUUM` removes dead tuples and marks their space as reusable for future inserts and updates. Without it, tables grow larger unnecessarily and consume more disk space than needed.

### Improves query performance

Dead tuples increase the number of rows PostgreSQL must scan, leading to slower `SELECT` queries and increased I/O.

`VACUUM` is necessary for the long-term health of a database, but it comes with trade-offs. `VACUUM FULL` locks a large table for the duration of the process, so other operations must wait until it completes. `AUTOVACUUM`, by contrast, runs in the background without significantly affecting performance. For large databases or highly active tables, default `AUTOVACUUM` settings may not be enough — it's common to tune them to run more frequently, or to manually vacuum during low-traffic periods.

### Prevents table bloat

Frequent `UPDATE` and `DELETE` operations can cause tables and indexes to grow much larger than needed. `VACUUM` reduces this bloat and keeps storage efficient.

### Updates statistics (with VACUUM ANALYZE)

```sql
VACUUM ANALYZE employees;
```

This updates table statistics used by the query planner, enabling PostgreSQL to choose better execution plans.

### Prevents transaction ID wraparound

Every transaction in PostgreSQL gets a transaction ID (XID). If old transaction IDs aren't cleaned up, PostgreSQL can eventually reach transaction ID wraparound, which may force the database into a protective shutdown. `VACUUM FREEZE` helps prevent this.

### Maintains index efficiency

Dead tuples can also leave unused entries in indexes. `VACUUM` helps keep indexes efficient and improves index scan performance.

### Example

Suppose a table contains 1 million rows:

```sql
DELETE FROM customers
WHERE status = 'inactive';
```

If 300,000 rows are deleted, they're marked as deleted but disk space isn't immediately freed, and queries still encounter these dead tuples. Running:

```sql
VACUUM customers;
```

cleans up the dead tuples and makes the space reusable.

## Monitoring VACUUM Processes

To ensure `VACUUM` is running smoothly, monitor:

- **Dead rows** — `pg_stat_user_tables` provides a breakdown of each table (`relname`) and how many dead rows (`n_dead_tup`) it holds. Tracking this, especially for frequently updated tables, shows whether vacuuming is keeping up.
- **Table disk usage** — unexpected disk growth (not explained by new data) usually points to a vacuuming problem, since without regular `VACUUM`, dead rows don't get reused and new data just consumes fresh disk space.
- **Last time autovacuum ran** — `pg_stat_user_tables` also shows the last time a manual or automatic vacuum successfully ran on each table.

```sql
SELECT relname, n_dead_tup, n_live_tup, last_vacuum, last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

## Common Issues and Solutions

### Issue 1: VACUUM runs but disk space doesn't shrink

Usually caused by another active transaction still holding a lock on the dead tuples.

Find long-running transactions:

```sql
SELECT pid, usename, query_start, now() - query_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY duration DESC;
```

If one is safe to stop, terminate it and run `VACUUM` again:

```sql
SELECT pg_terminate_backend(pid);
```

### Issue 2: VACUUM is slowing down production

Plain `VACUUM` can be heavy on I/O on large tables.

```sql
-- Lightweight, good for routine maintenance
VACUUM (ANALYZE) my_table;

-- Aggressive, reclaims disk space, but locks the table completely
VACUUM FULL my_table;
```

Use `VACUUM (ANALYZE)` for routine cleanup. Save `VACUUM FULL` for emergencies, since it takes an exclusive lock and blocks all reads/writes until it finishes.

### Issue 3: Not running VACUUM regularly

Skipping `VACUUM` lets dead tuples pile up, causing slowdowns and disk bloat. The fix is to let `AUTOVACUUM` handle it — it's enabled by default.

```sql
SHOW autovacuum;
```

For most databases, the default `AUTOVACUUM` settings are enough. Only tune them manually if it's clearly falling behind.

## Conclusion

`VACUUM` is a core part of keeping a PostgreSQL database healthy, not an optional maintenance task. It cleans up dead tuples left behind by `UPDATE` and `DELETE` operations, keeps disk usage under control, and protects the database from transaction ID wraparound.

For everyday use, standard `VACUUM` (or `VACUUM ANALYZE`) handles cleanup without disrupting normal operations, and `AUTOVACUUM` automates this so manual intervention is rarely needed. `VACUUM FULL` exists for severe bloat but should be used sparingly given its exclusive lock.

Monitoring tables through `pg_stat_user_tables` and watching disk usage trends helps catch vacuuming problems early, before they turn into performance issues. Understanding these tools and knowing when to step in manually is what separates a database that runs smoothly from one that slowly degrades over time.

## References

- What is Vacuum in PostgreSQL? — GeeksforGeeks
- PostgreSQL VACUUM: Common Issues and Solutions
- PostgreSQL VACUUM Processes: How to Monitor — Datadog
