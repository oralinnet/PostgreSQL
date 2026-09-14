## Replication Monitoring 

### On Primary Server 

#### Replication Status 
```sql
SELECT 
    client_addr,
    state,
    sync_state,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;
```
#### Replication Slot status 

```sql
SELECT * FROM pg_replication_slots;
```

### On Standby Server (Replica) 

#### Replica Status 

```sql 
SELECT pg_is_in_recovery();
```

#### Check replication delay 
```sql 
SELECT now() - pg_last_xact_replay_timestamp() AS delay;
```

#### Check WAL receiver status

```sql 
SELECT * FROM pg_stat_wal_receiver;
```

#### Check if WAL is being applied (Standby) 

```sql 
SELECT 
    pg_last_wal_receive_lsn(),
    pg_last_wal_replay_lsn(),
    pg_last_xact_replay_timestamp();
```

#### Measure actual replication delay 
```sql 
SELECT now() - pg_last_xact_replay_timestamp() AS delay;
```
#### One-Line Health Query 

```sql 
SELECT 
    pg_is_in_recovery(),
    now() - pg_last_xact_replay_timestamp() AS delay,
    pg_last_wal_receive_lsn() = pg_last_wal_replay_lsn() AS fully_synced;
```

#### Check replay status (Standby)

```sql
SELECT pg_is_wal_replay_paused();
```

### Compare data between Primary & Replica (REAL PROOF)

```sql 
SELECT count(*) FROM your_table;

-- IN primary DB insert data 
INSERT INTO test_replication (msg, created_at)
VALUES ('replication_test', now());

-- Now check in Standby DB 
SELECT * 
FROM test_replication 
ORDER BY created_at DESC 
LIMIT 1;
```