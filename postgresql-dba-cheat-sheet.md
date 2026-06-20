# PostgreSQL DBA Cheat Sheet — Essential Functions & Commands

> Quick Reference | Copy-Paste Ready | Organized by Category

## Table of Contents

1. [Server Information & Configuration](#1-server-information--configuration)
2. [Database, Schema & Tablespace](#2-database-schema--tablespace)
3. [Table, Index & Relation Size Functions](#3-table-index--relation-size-functions)
4. [Session, Connection & Process Functions](#4-session-connection--process-functions)
5. [Backup, Recovery & WAL Functions](#5-backup-recovery--wal-functions)
6. [Maintenance & VACUUM Functions](#6-maintenance--vacuum-functions)
7. [Security, Roles & Privilege Functions](#7-security-roles--privilege-functions)
8. [Statistics & Monitoring Views](#8-statistics--monitoring-views)
9. [Essential Utility Functions for DBAs](#9-essential-utility-functions-for-dbas)
10. [Emergency & Troubleshooting](#10-emergency--troubleshooting)

---

## 1. Server Information & Configuration

### Server Version & Uptime

| Function / Command | Example | Description |
|---|---|---|
| `version()` | `SELECT version();` | Full server version string with OS and compiler info |
| `current_setting('name')` | `SELECT current_setting('server_version');` | Get value of any configuration parameter |
| `SHOW parameter` | `SHOW work_mem;` | Show current value of a config parameter |
| `pg_postmaster_start_time()` | `SELECT pg_postmaster_start_time();` | When the server was last started |
| `now() - pg_postmaster_start_time()` | `SELECT now() - pg_postmaster_start_time() AS uptime;` | Server uptime as interval |
| `pg_conf_load_time()` | `SELECT pg_conf_load_time();` | When config was last reloaded |
| `pg_is_in_recovery()` | `SELECT pg_is_in_recovery();` | True if server is a standby replica |
| `inet_server_addr()` | `SELECT inet_server_addr(), inet_server_port();` | Server IP address and port |

### Configuration Management

| Function / Command | Example | Description |
|---|---|---|
| `pg_reload_conf()` | `SELECT pg_reload_conf();` | Reload postgresql.conf and pg_hba.conf (no restart) |
| `set_config(name, value, local)` | `SELECT set_config('work_mem', '256MB', true);` | Set parameter for current transaction (`true`) or session (`false`) |
| `ALTER SYSTEM SET` | `ALTER SYSTEM SET shared_buffers = '4GB';` | Write parameter to postgresql.auto.conf (persists across restarts) |
| `ALTER SYSTEM RESET` | `ALTER SYSTEM RESET work_mem;` | Remove parameter override from postgresql.auto.conf |
| `pg_settings` view | `SELECT name, setting, unit, context FROM pg_settings WHERE name LIKE '%mem%';` | All parameters with context (when change takes effect) |

---

## 2. Database, Schema & Tablespace

### Database Functions

| Function / Command | Example | Description |
|---|---|---|
| `current_database()` | `SELECT current_database();` | Name of the current database |
| `current_schema()` | `SELECT current_schema();` | Current active schema (first in search_path) |
| `current_schemas(true)` | `SELECT current_schemas(true);` | All schemas in search_path (true = include implicit) |
| `pg_database_size(name)` | `SELECT pg_size_pretty(pg_database_size('mydb'));` | Total size of a database including indexes |
| `pg_tablespace_size(name)` | `SELECT pg_size_pretty(pg_tablespace_size('pg_default'));` | Size of a tablespace |
| `pg_database.datconnlimit` | `SELECT datname, datconnlimit FROM pg_database;` | Connection limit per database (-1 = unlimited) |
| `has_database_privilege()` | `SELECT has_database_privilege('bob', 'mydb', 'CONNECT');` | Check if user can connect to database |

### Schema Functions

| Function / Command | Example | Description |
|---|---|---|
| `has_schema_privilege()` | `SELECT has_schema_privilege('bob', 'public', 'USAGE');` | Check user access to a schema |
| `pg_namespace` | `SELECT nspname, nspacl FROM pg_namespace;` | List all schemas with their ACLs |
| `SHOW search_path` | `SHOW search_path;` | Current schema search order |

---

## 3. Table, Index & Relation Size Functions

### Size Functions
*(all return bytes — wrap in `pg_size_pretty()`)*

| Function / Command | Example | Description |
|---|---|---|
| `pg_relation_size(rel)` | `SELECT pg_size_pretty(pg_relation_size('orders'));` | Size of table data only (no indexes, no TOAST) |
| `pg_table_size(rel)` | `SELECT pg_size_pretty(pg_table_size('orders'));` | Table + TOAST + FSM (no indexes) |
| `pg_indexes_size(rel)` | `SELECT pg_size_pretty(pg_indexes_size('orders'));` | Total size of all indexes on the table |
| `pg_total_relation_size(rel)` | `SELECT pg_size_pretty(pg_total_relation_size('orders'));` | Table + indexes + TOAST = everything |
| `pg_size_pretty(bigint)` | `SELECT pg_size_pretty(1073741824);` | Convert bytes to human-readable (e.g., '1024 MB') |
| `pg_size_bytes(text)` | `SELECT pg_size_bytes('1 GB');` | Convert human-readable to bytes |
| `pg_column_size(value)` | `SELECT pg_column_size(row(t.*)) FROM orders t LIMIT 5;` | Bytes used by a specific value or row |

### Finding Largest Objects

**Top 10 largest tables**
```sql
SELECT relname, pg_size_pretty(pg_total_relation_size(oid)) AS size
FROM pg_class
WHERE relkind = 'r'
ORDER BY pg_total_relation_size(oid) DESC
LIMIT 10;
```

**Top 10 largest indexes**
```sql
SELECT indexrelname, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC
LIMIT 10;
```

**Unused indexes** (never scanned — candidates for dropping)
```sql
SELECT indexrelname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;
```

---

## 4. Session, Connection & Process Functions

### Current Session

| Function / Command | Example | Description |
|---|---|---|
| `current_user` | `SELECT current_user;` | Current effective role (after SET ROLE) |
| `session_user` | `SELECT session_user;` | Original login role (never changes) |
| `pg_backend_pid()` | `SELECT pg_backend_pid();` | Process ID of current session |
| `pg_blocking_pids(pid)` | `SELECT pg_blocking_pids(12345);` | Array of PIDs blocking the given PID |
| `pg_stat_activity` | `SELECT pid, usename, state, query, query_start FROM pg_stat_activity WHERE state = 'active';` | All active connections and their queries |

### Process Control

| Function / Command | Example | Description |
|---|---|---|
| `pg_cancel_backend(pid)` | `SELECT pg_cancel_backend(12345);` | Cancel a running query (graceful — like Ctrl+C) |
| `pg_terminate_backend(pid)` | `SELECT pg_terminate_backend(12345);` | Kill a connection entirely (forceful) |
| `pg_stat_get_activity(pid)` | `SELECT * FROM pg_stat_get_activity(12345);` | Detailed info about a specific backend |

### Lock Management

| Function / Command | Example | Description |
|---|---|---|
| `pg_locks` view | `SELECT locktype, relation::regclass, mode, granted, pid FROM pg_locks WHERE NOT granted;` | All waiting (ungranted) locks |
| `pg_advisory_lock(key)` | `SELECT pg_advisory_lock(42);` | Acquire session-level advisory lock (blocks until acquired) |
| `pg_try_advisory_lock(key)` | `SELECT pg_try_advisory_lock(42);` | Try to acquire advisory lock (returns false if taken) |
| `pg_advisory_unlock(key)` | `SELECT pg_advisory_unlock(42);` | Release advisory lock |

---

## 5. Backup, Recovery & WAL Functions

### WAL (Write-Ahead Log) Functions

| Function / Command | Example | Description |
|---|---|---|
| `pg_current_wal_lsn()` | `SELECT pg_current_wal_lsn();` | Current WAL write position (Log Sequence Number) |
| `pg_current_wal_insert_lsn()` | `SELECT pg_current_wal_insert_lsn();` | Current WAL insert position |
| `pg_current_wal_flush_lsn()` | `SELECT pg_current_wal_flush_lsn();` | Last WAL position flushed to disk |
| `pg_wal_lsn_diff(lsn1, lsn2)` | `SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0'));` | Bytes between two WAL positions |
| `pg_walfile_name(lsn)` | `SELECT pg_walfile_name(pg_current_wal_lsn());` | WAL filename containing the given LSN |
| `pg_switch_wal()` | `SELECT pg_switch_wal();` | Force a WAL segment switch (useful before backup) |
| `pg_last_wal_receive_lsn()` | `SELECT pg_last_wal_receive_lsn();` | Last WAL received on standby (replica only) |
| `pg_last_wal_replay_lsn()` | `SELECT pg_last_wal_replay_lsn();` | Last WAL replayed on standby (replica only) |
| `pg_last_xact_replay_timestamp()` | `SELECT pg_last_xact_replay_timestamp();` | Timestamp of last replayed transaction (replica lag) |

### Backup Functions

| Function / Command | Example | Description |
|---|---|---|
| `pg_backup_start(label)` | `SELECT pg_backup_start('my_backup');` | Signal start of online backup (PG 15+) |
| `pg_backup_stop()` | `SELECT * FROM pg_backup_stop();` | Signal end of online backup, returns WAL info |
| `pg_create_restore_point(name)` | `SELECT pg_create_restore_point('before_migration');` | Create a named restore point for PITR |
| `pg_is_in_backup()` | `SELECT pg_is_in_backup();` | True if a backup is in progress |

### Replication Functions

| Function / Command | Example | Description |
|---|---|---|
| `pg_create_physical_replication_slot()` | `SELECT pg_create_physical_replication_slot('replica1');` | Create a replication slot (prevents WAL removal) |
| `pg_drop_replication_slot(name)` | `SELECT pg_drop_replication_slot('replica1');` | Drop a replication slot |
| `pg_replication_slot_advance()` | `SELECT pg_replication_slot_advance('replica1', '0/1234');` | Advance slot position (skip WAL) |
| `pg_stat_replication` | `SELECT client_addr, state, sent_lsn, replay_lsn FROM pg_stat_replication;` | Status of all connected replicas |

**Replication lag query** (run on standby)
```sql
SELECT now() - pg_last_xact_replay_timestamp() AS lag;
```

---

## 6. Maintenance & VACUUM Functions

### VACUUM & ANALYZE

| Function / Command | Example | Description |
|---|---|---|
| `VACUUM` | `VACUUM orders;` | Reclaim dead tuple space (runs concurrently) |
| `VACUUM FULL` | `VACUUM FULL orders;` | Rewrite table to reclaim disk (EXCLUSIVE LOCK!) |
| `VACUUM ANALYZE` | `VACUUM ANALYZE orders;` | Vacuum + update planner statistics in one pass |
| `VACUUM (VERBOSE)` | `VACUUM (VERBOSE) orders;` | Show detailed dead tuple and page information |
| `VACUUM FREEZE` | `VACUUM FREEZE orders;` | Aggressively freeze transaction IDs (prevent wraparound) |
| `ANALYZE` | `ANALYZE orders (customer_id, order_date);` | Update column statistics for query planner |

### Index Maintenance

| Function / Command | Example | Description |
|---|---|---|
| `REINDEX INDEX` | `REINDEX INDEX idx_orders_date;` | Rebuild a single index (locks table briefly) |
| `REINDEX TABLE` | `REINDEX TABLE orders;` | Rebuild all indexes on a table |
| `REINDEX INDEX CONCURRENTLY` | `REINDEX INDEX CONCURRENTLY idx_orders_date;` | Online reindex — no blocking (PG 12+) |
| `REINDEX DATABASE` | `REINDEX DATABASE mydb;` | Rebuild all indexes in the database |
| `pg_stat_user_indexes` | `SELECT indexrelname, idx_scan, idx_tup_read FROM pg_stat_user_indexes WHERE relname = 'orders';` | Index usage statistics — find unused indexes |

### Bloat & Dead Tuple Monitoring

**Dead tuples per table + last vacuum time**
```sql
SELECT relname, n_live_tup, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

**Tables with highest bloat percentage**
```sql
SELECT relname,
       ROUND(n_dead_tup*100.0/NULLIF(n_live_tup+n_dead_tup,0),2) AS dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000;
```

| Function / Command | Example | Description |
|---|---|---|
| `pg_stat_progress_vacuum` | `SELECT * FROM pg_stat_progress_vacuum;` | Live progress of running VACUUM operations |
| `pg_stat_progress_create_index` | `SELECT * FROM pg_stat_progress_create_index;` | Live progress of running CREATE INDEX / REINDEX |

### Transaction ID & Wraparound

**Transaction age per database** (warn if > 1 billion)
```sql
SELECT datname, age(datfrozenxid)
FROM pg_database
ORDER BY age DESC;
```

**Table-level age** (closest to wraparound — need VACUUM FREEZE)
```sql
SELECT relname, age(relfrozenxid)
FROM pg_class
WHERE relkind = 'r'
ORDER BY age DESC
LIMIT 10;
```

| Function / Command | Example | Description |
|---|---|---|
| `txid_current()` | `SELECT txid_current();` | Current transaction ID |

---

## 7. Security, Roles & Privilege Functions

### Privilege Check Functions
*(all return boolean)*

| Function / Command | Example | Description |
|---|---|---|
| `has_table_privilege()` | `SELECT has_table_privilege('bob', 'orders', 'SELECT');` | Can user SELECT from this table? |
| `has_column_privilege()` | `SELECT has_column_privilege('bob', 'employees', 'salary', 'SELECT');` | Can user SELECT this specific column? |
| `has_schema_privilege()` | `SELECT has_schema_privilege('bob', 'public', 'CREATE');` | Can user CREATE objects in this schema? |
| `has_database_privilege()` | `SELECT has_database_privilege('bob', 'mydb', 'CONNECT');` | Can user CONNECT to this database? |
| `has_function_privilege()` | `SELECT has_function_privilege('bob', 'my_func(int)', 'EXECUTE');` | Can user EXECUTE this function? |
| `has_sequence_privilege()` | `SELECT has_sequence_privilege('bob', 'orders_id_seq', 'USAGE');` | Can user use this sequence? |
| `pg_has_role(user, role, priv)` | `SELECT pg_has_role('bob', 'admin_group', 'MEMBER');` | Is user a member of this role? |

### Role & Auth Information

| Function / Command | Example | Description |
|---|---|---|
| `pg_roles` view | `SELECT rolname, rolsuper, rolcreatedb, rolcanlogin, rolconnlimit FROM pg_roles;` | All roles with their attributes |
| `pg_hba_file_rules` | `SELECT line_number, type, database, user_name, address, auth_method FROM pg_hba_file_rules;` | Current pg_hba.conf rules (PG 15+) |

**Role membership** (who belongs to what group)
```sql
SELECT r.rolname AS grp, m.rolname AS member
FROM pg_auth_members am
JOIN pg_roles r ON r.oid = am.roleid
JOIN pg_roles m ON m.oid = am.member;
```

**Check password hash type for each login role**
```sql
SELECT rolname,
       CASE
           WHEN rolpassword LIKE 'SCRAM%' THEN 'scram'
           WHEN rolpassword LIKE 'md5%' THEN 'md5'
           ELSE 'none'
       END AS hash_type
FROM pg_authid
WHERE rolcanlogin;
```

### SSL / Encryption

| Function / Command | Example | Description |
|---|---|---|
| `pg_stat_ssl` | `SELECT pid, ssl, version, cipher, bits FROM pg_stat_ssl;` | SSL status of every connection |
| `SHOW ssl` | `SHOW ssl;` | Is SSL enabled on the server? |

**Find connections NOT using SSL**
```sql
SELECT sa.usename, ss.ssl, sa.client_addr
FROM pg_stat_ssl ss
JOIN pg_stat_activity sa ON ss.pid = sa.pid
WHERE NOT ss.ssl;
```

---

## 8. Statistics & Monitoring Views

### Connection & Query Monitoring

**All active sessions, their state, wait events, current query**
```sql
SELECT pid, usename, state, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE state != 'idle';
```

**Queries running longer than 5 minutes**
```sql
SELECT pid, now() - query_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active' AND now() - query_start > '5 min'::interval;
```

**Connections per user**
```sql
SELECT usename, count(*)
FROM pg_stat_activity
GROUP BY usename
ORDER BY count DESC;
```

**Top 10 resource-consuming queries** (requires `pg_stat_statements` extension)
```sql
SELECT query, calls, mean_exec_time::numeric(10,2), rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

### Table-Level Statistics

**Scans, inserts, updates, deletes, dead tuples per table**
```sql
SELECT relname, seq_scan, idx_scan, n_tup_ins, n_tup_upd, n_tup_del,
       n_dead_tup, last_autovacuum
FROM pg_stat_user_tables;
```

**Tables doing only sequential scans — need indexes?**
```sql
SELECT relname, seq_scan, idx_scan
FROM pg_stat_user_tables
WHERE seq_scan > 1000 AND COALESCE(idx_scan, 0) = 0;
```

**Buffer cache hit ratio per table**
```sql
SELECT relname, heap_blks_read, heap_blks_hit,
       ROUND(heap_blks_hit*100.0/NULLIF(heap_blks_hit+heap_blks_read,0),2) AS cache_hit_pct
FROM pg_statio_user_tables;
```

### Database-Level Statistics

**Overall database activity and cache performance**
```sql
SELECT datname, numbackends, xact_commit, xact_rollback,
       blks_read, blks_hit, tup_returned, tup_fetched
FROM pg_stat_database;
```

**Database-level buffer cache hit ratio** (aim for > 99%)
```sql
SELECT datname,
       ROUND(blks_hit*100.0/NULLIF(blks_hit+blks_read,0),2) AS cache_hit_pct
FROM pg_stat_database
WHERE datname = current_database();
```

**High rollback percentage may indicate application issues**
```sql
SELECT datname, xact_commit, xact_rollback,
       ROUND(xact_rollback*100.0/NULLIF(xact_commit+xact_rollback,0),2) AS rollback_pct
FROM pg_stat_database;
```

### Checkpoint & Background Writer

| View / Function | Example Query | What It Shows |
|---|---|---|
| `pg_stat_bgwriter` | `SELECT checkpoints_timed, checkpoints_req, buffers_checkpoint, buffers_clean, buffers_backend FROM pg_stat_bgwriter;` | Checkpoint frequency and buffer writes (high `buffers_backend` = bad) |
| `pg_stat_wal` | `SELECT wal_records, wal_bytes, wal_buffers_full FROM pg_stat_wal;` | WAL generation statistics (PG 14+) |

---

## 9. Essential Utility Functions for DBAs

### Object Information

| Function | Example | Description |
|---|---|---|
| `pg_typeof(value)` | `SELECT pg_typeof(42), pg_typeof('hello');` | Data type of any expression |
| `obj_description(oid)` | `SELECT obj_description('orders'::regclass);` | Comment on a database object |
| `col_description(table, col)` | `SELECT col_description('orders'::regclass, 1);` | Comment on a specific column |
| `pg_get_viewdef(view)` | `SELECT pg_get_viewdef('my_view', true);` | SQL definition of a view |
| `pg_get_indexdef(idx)` | `SELECT pg_get_indexdef('idx_orders_date'::regclass);` | SQL definition of an index |
| `pg_get_constraintdef(oid)` | `SELECT conname, pg_get_constraintdef(oid) FROM pg_constraint WHERE conrelid = 'orders'::regclass;` | Definition of all constraints on a table |
| `pg_get_functiondef(oid)` | `SELECT pg_get_functiondef('my_func'::regproc);` | Full CREATE FUNCTION statement |
| `::regclass` cast | `SELECT 'orders'::regclass::oid;` | Get OID of a table by name |

### Date/Time (common in DBA scripts)

| Function | Example | Description |
|---|---|---|
| `now()` / `CURRENT_TIMESTAMP` | `SELECT now();` | Current date+time with timezone |
| `clock_timestamp()` | `SELECT clock_timestamp();` | Actual current time (changes during statement) |
| `date_trunc('unit', ts)` | `SELECT date_trunc('hour', now());` | Truncate timestamp to hour/day/month/year |
| `extract(field FROM ts)` | `SELECT extract(epoch FROM now());` | Extract part of timestamp (epoch = Unix seconds) |
| `age(ts1, ts2)` | `SELECT age(now(), '2020-01-01');` | Interval between two timestamps |
| `generate_series(start,end,step)` | `SELECT generate_series('2026-01-01'::date, '2026-12-01'::date, '1 month');` | Generate a series of dates (great for reports) |

### System Catalog Shortcuts (psql)

| Command | Example | Description |
|---|---|---|
| `\dt` | `\dt public.*` | List all tables in a schema |
| `\di` | `\di+ orders*` | List indexes on tables matching pattern |
| `\df` | `\df+ my_func` | Show function signatures and source |
| `\du` | `\du` | List all roles |
| `\dp` | `\dp orders` | Show table privileges / ACLs |
| `\l` | `\l+` | List all databases with sizes |
| `\dx` | `\dx` | List installed extensions |
| `\x` | `\x auto` | Toggle expanded display (vertical output) |
| `\timing` | `\timing on` | Show query execution time |
| `\watch n` | `SELECT count(*) FROM orders; \watch 5` | Re-run query every n seconds |

---

## 10. Emergency & Troubleshooting

### Kill Queries & Connections

| Scenario | Command / Query | Notes |
|---|---|---|
| Cancel one query | `SELECT pg_cancel_backend(pid);` | Graceful — query gets ERROR, connection stays |
| Kill one connection | `SELECT pg_terminate_backend(pid);` | Forceful — entire connection dropped |

**Kill all connections to a DB** (nuclear option — drops everyone except you)
```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE datname = 'mydb' AND pid <> pg_backend_pid();
```

**Kill idle-in-transaction** (clean up abandoned transactions holding locks)
```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND now() - state_change > '10 min'::interval;
```

### Lock Troubleshooting

**Find blocked queries** (shows who is blocking whom)
```sql
SELECT blocked.pid, blocked.query,
       blocking.pid AS blocker_pid, blocking.query AS blocker_query
FROM pg_stat_activity blocked
JOIN pg_locks bl ON bl.pid = blocked.pid
JOIN pg_locks kl ON kl.locktype = bl.locktype
                 AND kl.relation = bl.relation
                 AND kl.pid <> bl.pid
                 AND NOT bl.granted
JOIN pg_stat_activity blocking ON blocking.pid = kl.pid;
```

**Find lock holders on a specific table**
```sql
SELECT locktype, relation::regclass, mode, granted, pid
FROM pg_locks
WHERE relation = 'orders'::regclass;
```

**Deadlock detection** — PostgreSQL auto-detects deadlocks and kills one transaction
```sql
SHOW deadlock_timeout; -- default 1s
```

### Disk Space Emergency

**Check disk usage** (how much disk the current database uses)
```sql
SELECT pg_size_pretty(pg_database_size(current_database()));
```

**Largest tables** (find what is consuming space)
```sql
SELECT relname, pg_size_pretty(pg_total_relation_size(oid))
FROM pg_class
WHERE relkind = 'r'
ORDER BY pg_total_relation_size(oid) DESC
LIMIT 5;
```

> ⚠️ **TRUNCATE for emergency** — instantly reclaims space (no MVCC overhead), but is **unrecoverable**:
> ```sql
> TRUNCATE TABLE audit_log;
> ```

**Check WAL accumulation** (each file = 16MB, too many = archiver stuck)
```sql
SELECT count(*) FROM pg_ls_waldir();
```

**List recent WAL files with sizes**
```sql
SELECT name, size, modification
FROM pg_ls_waldir()
ORDER BY modification DESC
LIMIT 10;
```

### Replication Emergency

**Check replica lag (run on primary)** — bytes of lag per replica
```sql
SELECT client_addr, state, pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes
FROM pg_stat_replication;
```

**Check replica lag (run on standby)** — time lag on the replica itself
```sql
SELECT now() - pg_last_xact_replay_timestamp() AS lag;
```

**Stuck replication slot** — inactive slots retain WAL indefinitely, disk fills up!
```sql
SELECT slot_name, active,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots;
```

**Drop stuck slot** (make sure replica is gone first)
```sql
SELECT pg_drop_replication_slot('stuck_slot');
```

---

*PostgreSQL DBA Cheat Sheet — Essential Functions & Commands*
