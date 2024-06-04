#### Postgresql Session 

- Number of active connections and remaining connections
```sql
select max_conn,used,res_for_super,max_conn-used-res_for_super res_for_normal 
from 
  (select count(*) used from pg_stat_activity) t1,
  (select setting::int res_for_super from pg_settings where name=$$superuser_reserved_connections$$) t2,
  (select setting::int max_conn from pg_settings where name=$$max_connections$$) t3;
```

- Terminate connections that have been idle for 15 minutes or longer.
```sql
SELECT pg_terminate_backend(pid) 
FROM   pg_stat_activity 
WHERE  state = 'idle' 
       AND state_change < now() - '15min'::interval; 
```
- Find PostgreSQL Queries running longer than 2 Minutes
```sql
SELECT pid, now() - pg_stat_activity.query_start AS duration, query 
FROM pg_stat_activity 
WHERE (now() - pg_stat_activity.query_start) > interval '2 minutes';
```
- Kill session Id 
```sql
SELECT pg_terminate_backend(8428);

select pg_terminate_backend(pid) 
from pg_stat_activity
where pid = '1344279';
```

- Find queries running longer than 5 minutes:
```sql
SELECT
  pid,
  user,
  pg_stat_activity.query_start,
  now() - pg_stat_activity.query_start AS query_time,
  query,
  state,
  wait_event_type,
  wait_event
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes' AND state = 'active';
```


- Kill long-running PostgreSQL query processes:

Where some queries look like they’re not going to finish, you can use the pid (process ID) from the pg_stat_activity or pg_locks views to terminate the running process.
```sql
---- attempt to gracefully kill a running query process.
pg_cancel_backend(pid) 

--- immediately kill the running query process, but potentially have side affects across additional queries running on your database server. The full connection may be 
--- reset when running pg_terminate_backend, so other running queries can be affected. Use as a last resort. -->

pg_terminate_backend(pid)
```