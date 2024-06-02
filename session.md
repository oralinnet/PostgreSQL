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
