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
```sh
SELECT pg_terminate_backend(pid) 
FROM   pg_stat_activity 
WHERE  state = 'idle' 
       AND state_change < now() - '15min'::interval; 
```
