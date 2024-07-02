## PostgreSQL Database Monitoring Script 

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
- Active session 
```sql
select pid as process_id,
usename as username,
datname as database_name,
client_addr as client_address,
application_name,
backend_start,
state,
state_change,query 
from pg_stat_activity where datname='xyz'
and state ='active';
```
### DATABASE MANAGEMENT

- Create a database in postgres
```sql 
create database DBATEST;
-- DB Create with Tablespace
create database DBATEST with tablespace ts_postgres;
-- DB Create with Tablespace and template
CREATE DATABASE "DBATEST"
WITH TABLESPACE ts_postgres
OWNER "postgres"
ENCODING 'UTF8'
LC_COLLATE = 'en_US.UTF-8'
LC_CTYPE = 'en_US.UTF-8'
TEMPLATE template0;

-- View database information:

select * from pg_database;
```
- How to connect to postgres db
```sql
psql -d edb -U postgres -h hostname/IP
-- Find current connection info
postgres=# \conninfo
select current_schema,current_user,session_user,current_database();
-- Switch to another database
postgres-# \c testdb
```
- Drop database from psql
```sql 
-- Note - while dropping a database, you need to connect to a database other than the db you are trying to drop.
drop database "testdb";

-- If Drop command failed, because the some sessions are already connected to the database. Lets clear them
select application_name,client_hostname,pid,usename from pg_stat_activity where datname='testdb';
-- Kill Session with session id 
select pg_terminate_backend(pid) from pg_stat_activity where pid='12755';
```
- Find the database details in postgres 
```sql 
postgres=# \list+
select datname from pg_database;

```
- How to get postgres db size 
```sql
SELECT pg_database.datname as "database_name", pg_size_pretty(pg_database_size(pg_database.datname)) AS size_in_mb FROM pg_database ORDER by size_in_mb DESC;
postgres=# \l+
```
- Timezone Information
```sql
show timezone;
SELECT current_setting('TIMEZONE');
select name,setting,short_desc,boot_val from pg_settings where name='TimeZone';
```

- Find postgres version
```sql
show server_version;
select version ();

```
- Enable archiving(wal) in postgres
```sql
---  Create directory for archiving
mkdir -p /archive/location
-- Update the postgres.conf file with below values
wal_level = replica
archive_mode = on
max_wal_senders=1
archive_command= 'test ! -f /archive/location/%f && cp %p /archive/location/%f'
-- Restart Database 
pg_ctl stop -D $PGDATA
pg_ctl start -D $PGDATA

-- Check archive status
select name,setting from pg_settings where name like 'archive%';
```
- Rotate server log in postgres
```sql 
- Below command signals the log-file manager to switch to a new output file immediately.it is just like an alert log
select pg_rotate_logfile() ;
```
- Find query execution time using pg_stat_statement
```sql 
-- Monitor query execution time

select substr(query,1,100) query,calls,min_time/1000 "min_time(in sec)" , max_time/1000 "max_time(in sec)", mean_time/1000 "avg_time(in sec)", rows from pg_stat_statements order by mean_time desc;

-- NOTE: If pg_stat_statements is not available in your database, then activate using below:
-- Add below parameters in postgres.conf file and restart the postgres cluster

shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all

sudo service postgresql restart

-- Now create extension:
create extension pg_stat_statements;
```

- DATA DIRECTORY LOCATION 
```sql
-- DATA_DIRECTORY - > Specifies the directory to use for data storage.
show data_directory;
select setting from pg_settings where name = 'data_directory';
-- This will show location of important files in postgres
SELECT name, setting FROM pg_settings WHERE category = 'File Locations';

```

- Find current sessions in the postgres
```sql
select pid as process_id,
usename as username,
datname as database_name,
client_addr as client_address,
application_name,
backend_start,
state,
state_change,query
from pg_stat_activity;

-- For specific database:

select pid as process_id,
usename as username,
datname as database_name,
client_addr as client_address,
application_name,
backend_start,
state,
state_change,query 
from pg_stat_activity where datname='testdb';
```

- Kill a session in postgres
```sql
-- First find the pid of the session:
SELECT datname as database, pid as pid, usename as username, application_name , client_addr , query FROM pg_stat_activity;

-- Lets say the pid=1124, in the below query pass the pid value to kill that particular session..

select pg_terminate_backend(pid) from pg_stat_activity where pid='1123';
```

- Cancel a session in postgres
```sql
-- First find the pid of the session:

dbaclass#SELECT datname as database, pid as pid, usename as username, application_name , client_addr , query FROM pg_stat_activity;

-- Lets say the pid=1124, in the below query pass the pid value to cancel that particular session query

select pg_cancel_backend(pid) from pg_stat_activity where pid='1124';
```

- Kill all session of a user in postgres
```sql
-- Here we want to kill all session of the user postgres

-- List all the session of that user.
select datname as database, pid as pid, usename as username, application_name , client_addr, query FROM pg_stat_activity where username='postgres';

-- Kill all the session of user postgres.

select pg_terminate_backend(pid) from pg_stat_activity where usename='postgres';
```

- Find locks present in postgres
```sql
select t.relname,l.locktype,page,virtualtransaction,pid,mode,granted from pg_locks l, pg_stat_all_tables t where l.relation=t.relid order by relation asc;
```

- Find blocking sessions in postgres
```sql
-- QUERY TO FIND BLOCKING SESSION DETAILS

select pid as blocked_pid, usename, pg_blocking_pids(pid) as "blocked_by(pid)", query as blocked_query from pg_stat_activity where cardinality(pg_blocking_pids(pid)) > 0;
```

- Find location of postgres conf files
```sql
SELECT name, setting FROM pg_settings WHERE category = 'File Locations';
show config_file;
```

- Find current data/time on postgres db
```sql
SELECT CURRENT_TIMESTAMP;
SELECT CURRENT_TIMESTAMP;
select statement_timestamp() ;
select timeofday() ;
select localtime(0);
select localtimestamp(0);
```

- Find extension details
```sql
psql# \dx
psql#\dx+
SELECT * FROM pg_extension;

-- For finding available extension in server:
SELECT * FROM pg_available_extensions;
```

- Find startup time and uptime postgres
```sql

--  Uptime of server

SELECT now() - pg_postmaster_start_time() "uptime";

-- Server startup time:

SELECT pg_postmaster_start_time();
```

- Find archiver process status
```sql
select * from pg_stat_archiver;
```

- Find postgres configuration values
```sql
select * from pg_settings;
select * from pg_settings where name='port';
show config_file;
```

- Find the last pg config reload time
```sql
select pg_conf_load_time() ;
select pg_reload_conf();
 select pg_conf_load_time() ;
```

- wal switch manually
```sql
select pg_switch_wal();
```

- Monitor Archiving Process
```sql

select pg_walfile_name(pg_current_wal_lsn()),last_archived_wal,last_failed_wal,
('x'||substring(pg_walfile_name(pg_current_wal_lsn()),9,8))::bit(32)::int*256 +
('x'||substring(pg_walfile_name(pg_current_wal_lsn()),17))::bit(32)::int -
('x'||substring(last_archived_wal,9,8))::bit(32)::int*256 -
('x'||substring(last_archived_wal,17))::bit(32)::int
as diff from pg_stat_archiver;

```

- View/modify connection limit of database

```sql
---View existing connection limit setting:( datconnlimit )

postgres=# select datname,datallowconn,datconnlimit from pg_database where datname='test_dev';
-[ RECORD 1 ]--+------------
datname        | test_dev
datallowconn  | t.
datconnlimit  | -1.       -- >Means unlimited connections allowed

-- To set a specific limit for connection

alter database test_dev connection limit 100;

-- To restrict all the connections to db

alter database test_dev connection limit 0;

-- NOTE: Even if connection limit is set to 0 , the superuser will be able to connect to database.
```

- Find wal file details and its size
```sql 
-- List down all the wal files present in pg_wal
select * from pg_ls_waldir();

-- Find total size of wal:
select sum(size) from pg_ls_waldir();

-- Find current wal file lsn:
select pg_current_wal_insert_lsn(),pg_current_wal_lsn();
```

- Find temp file usage of databases
```sql
SELECT datname, temp_files, temp_bytes, stats_reset FROM pg_stat_database;

```

### MAINTENANCE 

- Update statistics of a table using analyze

```sql
-- Analyze stats for a table testanalyze(schema is public)

analyze testanalyze;

-- For analyzing selected columns for emptab table ( schema is dbatest)

analyze dbatest.emptab (datname,datdba);
dbaclass=# select relname,reltuples from pg_class where relname in ('testanalyze','emptab');
select schemaname,relname,analyze_count,last_analyze,last_autoanalyze from pg_stat_user_tables where relname in ('testanalyze','emptab');

---Analyze command with verbose command

analyze verbose dbatest.emptab (datname,datdba);

---Analyze tables in the current schema that the user has access to.
 analyze ;

-- NOTE: ANALYZE requires only a read lock on the target table, so it can run in parallel with other activity on the table.
```

- Reorg a table using VACUUM command

```sql
-- VACUUM - >  REMOVES DEAD ROWS, AND MARK THEM FOR REUSE, BUT IT DOESN’T RETURN THE SPACE TO ORACLE,. IT DOESN'T NEED EXCLUSIVE LOCK ON THE TABLE.

-- vacuum a table:
vacuum dbatest.emptab;

-- both vacuum and analyze:

vacuum analyze dbatest.emptab;

-- with verbose
vacuum verbose analyze dbatest.emptab;
select schemaname,relname,last_vacuum,vacuum_count from pg_stat_user_tables where relname='emptab';
```

- Reorg a table using VACUUM FULL command

```sql
-- VACUUM FULL - > JUST LIKE MOVE COMMAND IN ORACLE . IT TAKES MORE TIME, BUT IT RETURNS THE SPACE TO OS BECAUSE OF ITS COMPLEX ALGORITHM. IT also requires additional disk space , which can store the new copy of the table., until the activity is completed. Also it locks the table exclusively, which block all operations on the table .

-- Command to run vacuum full command for table:
VACUUM FULL dbatest.emptab;

-- DEMO TO CHECK HOW IT RECLAIMS SPACE:

-- Check existing space and delete some data:
select pg_size_pretty(pg_relation_size('dbatest.emptab'));
delete from dbatest.emptab where oid=13634;
DELETE 131072

-- We can observe size is still same:

select pg_size_pretty(pg_relation_size('dbatest.emptab'));

-- Run vacuum full and observe the space usage:

VACUUM FULL dbatest.emptab;
select pg_size_pretty(pg_relation_size('dbatest.emptab'));

```

- Manage autovacuum process in postgres

```sql
-- Autovacuum methods automates the executions vacuum,freeze and analyze commands.
-- Find whether autovacuum is enabled or not:
select name,setting,short_desc,boot_val,pending_restart from pg_settings where name in ('autovacuum','track_counts');

-- Find other autovacuum related parameter settings
select name,setting,short_desc,min_val,max_val,enumvals,boot_val,pending_restart from pg_settings where category like 'Autovacuum';

-- Change autovacuum settings:( they need restart)
alter system set autovacuum_max_workers=10 ;

-- Now restart :

pg_ctl stop
pg_ctl start

```

- Rebuild indexes using REINDEX

```sql
-- REINDEX rebuilds an index using the data stored in the index's table, replacing the old copy of the index. There are several scenarios in which to use REINDEX:

--  Rebuild particular index:

REINDEX INDEX TEST_IDX2;

-- Rebuild all indexes on a table:
REINDEX TABLE TEST;

-- Rebuild all indexes of tables in a schema:
reindex schema public;

-- Rebuild all indexes in a database :
reindex database dbaclass;

-- Reindex with verbose option:
reindex (verbose) table test;

-- Rebuild index without causing lock on the table:( using concurrently option) 
REINDEX ( verbose) table concurrently test;
```

- Monitor index creation or rebuild

```sql
SELECT a.query,p.phase, p.blocks_total,p.blocks_done,p.tuples_total, p.tuples_done FROM pg_stat_progress_create_index p JOIN pg_stat_activity a ON p.pid = a.pid;

(or)

select pid,datname,command,phase,tuples_total,tuples_done,partitions_total,partitions_done from pg_stat_progress_create_index;

```

- Monitor vacuum operation 

```sql
select * from pg_stat_progress_vacuum;

```

- find and change statistics level of a column
```sql
-- Finding statistics level of a column ( orders.orderdate)
-- statistics level range is 1-10000 ( where 100 means 1 percent,10000 means 100 percent)

SELECT attname as column_name , attstattarget as stats_level FROM pg_attribute WHERE attrelid = (SELECT oid FROM pg_class WHERE relname = 'orders') and attname='orderdate';

-- To change statistics level of a column:
alter table orders alter column orderdate set statistics 1000;
```

- Find vaccum settings of tables

```sql
SELECT n.nspname, c.relname,
pg_catalog.array_to_string(c.reloptions || array(
select 'toast.' ||
x from pg_catalog.unnest(tc.reloptions) x),', ')
as relopts
FROM pg_catalog.pg_class c
LEFT JOIN
pg_catalog.pg_class tc ON (c.reltoastrelid = tc.oid)
JOIN
pg_namespace n ON c.relnamespace = n.oid
WHERE c.relkind = 'r'
AND nspname NOT IN ('pg_catalog', 'information_schema');

```

- Modify autovacuum setting of table/index

```sql
-- Disable autovacuum for a table:

alter table test2 set( autovacuum_enabled = off);

-- Enable autovacuum for a table

alter table test2 set( autovacuum_enabled = on);
```

- Find last vaccum/analyze details of a table

```sql 
select * from pg_stat_user_tables where relname='test';
```

- Find how much bloating a table has

```sql
-- Create the pgstattuple extension:
create extension pgstattuple;

-- bloating percentage of the table "test":
SELECT pg_size_pretty(pg_relation_size('test')) as table_size,(pgstattuple('test')).dead_tuple_percent;

-- bloating percentage of index "test_x_idx":

select pg_relation_size('test_x_idx') as index_size, 100-(pgstatindex('test_x_idx')).avg_leaf_density as bloat_ratio;
```

### TABLESPACE MANAGEMENT

- View tablespace info in postgres

```sql
-- VIEW TABLESPACE INFO IN POSTGRES:
select * from pg_tablespace;

(OR)

postgres=# \db+

(or)

-- For getting size of specific tablespace:
select pg_size_pretty(pg_tablespace_size('ts_dbaclass'));


-- Pre-configured tablespaces:( these are default tablespaces)

Pg_global - > PGDATA/global - > used for cluster wide table and system catalog
Pg_default - > PGDATA/base directory - > it stores databases and relations
```

- create/drop/rename tablespace in postgres
```sql
-- CREATE TABLESPACE:
create tablespace ts_postgres location '/Library/PostgreSQL/TEST/TS_POSTGRES';

-- RENAME TABLESPACE:
alter tablespace ts_postgres rename to ts_dbaclass;

-- DROP TABLESPACE:
drop tablespace ts_dbaclass;

-- Before dropping tablespace make sure it is emptry
```

- find/change default tablespace
```sql
show default_tablespace;

-- <<<< If output is blank means default is pg_default tablespace>>>>>

--To change the default tablespace at database level:

alter system set default_tablespace=ts_postgres;
select pg_reload_conf();
show default_tablespace;
SELECT name, setting FROM pg_settings where name='default_tablespace';

-- Steps to change default tablespace at session level:

set default_tablespace=ts_postgres;

```

- find/change default temp tablespace

```sql
-- VIEW DEFAULT TEMP TABLESPACE:
SELECT name, setting FROM pg_settings where name='temp_tablespaces';
show temp_tablespaces

-- CHANGE DEFAULT TEMP TABLESPACE

alter system set temp_tablespaces=TS_TEMP;
select pg_reload_conf();
show temp_tablespaces;
SELECT name, setting FROM pg_settings where name='temp_tablespaces';

```

- How to change tablespace owner
```sql
alter tablespace ts_postgres owner to dev_admin;
\db+
```

- Move table/index to different tablespace

```sql
-- move table to different tablespace
alter table TEST8 set tablespace pg_crm;

-- Move index to different tablespace

alter index TEST_ind set tablespace pg_crm;

```

- Move database to new tablespace in postgres

```sql
alter database prod_crm set tablespace crm_tblspc;

-- Before running this. make sure there are no active connections in the database.
-- You can kill the existing session using below query.
select pg_terminate_backend(pid) from pg_stat_activity where datname='DB_NAME';

```

### AUDITING & SECURITY

- Find pg_hba.conf file content

```sql
-- This provides a summary of contents of client authentication config file pg_hba.conf 

select * from pg_hba_file_rules;
```

- Enable auditing for ddl/dml statement

```sql
-- Check auditing setting :
show log_statement;

-- For logging all ddl activites:

alter system set log_statement=ddl;
select pg_reload_conf();

-- For logging all DDL DML activities:
alter system set log_statement=mod;
select pg_reload_conf();

-- For logging all statement( i.e ddl , dml and even select statements)
alter system set log_statement='all';
select pg_reload_conf();

```

- Enable audit for log on/log off to postgres

```sql
-- Enable audit for connection and disconnection to postgres.

select name,setting from pg_settings where name in ('log_disconnections','log_connections');

alter system set log_disconnections=off;
alter system set log_connections=on;
select pg_reload_conf();

-- Now all log on and log off will logged in the log file.

<<<<<<<<< cd /Library/PostgreSQL/10/data/log/ >>>>>>>
2020-07-06 12:51:39.042 IST [10212] LOG: connection received: host=[local]
2020-07-06 12:51:53.416 IST [10215] LOG: connection received: host=[local]
2020-07-06 12:51:53.420 IST [10215] LOG: connection authorized: user=postgres database=postgres
```

### REPLICATION
- Check streaming recovery status
```sql
-- Run this on hot standby server 

select pg_is_wal_replay_paused();

-- If the output is f then, streaming recovery is running, if t means not running.
```
- Check replication details on primary server

```sql
-- Run on this primary server for outgoing replication details
select * from pg_stat_replication;
```

- Get received /replayed WAL records on standby(replication)
```sql
-- Run on standby database
select pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn(), pg_last_xact_replay_timestamp();
select * from pg_stat_wal_receiver;
```

- How to stop /resume recovery in standy(replication)

```sql
-- To stop/pause recovery on replication server(standby)

select pg_wal_replay_pause();
select pg_is_wal_replay_paused();

-- To Resume recovery on replication server(standby)

select pg_wal_replay_resume();
select pg_is_wal_replay_paused();

```

- Find lag in streaming replication

```sql
-- Find lag in bytes( run on standby)

SELECT pg_wal_lsn_diff(sent_lsn, replay_lsn) from pg_stat_replication;

--- Find lag in seconds( run on standby)

SELECT CASE WHEN pg_last_wal_receive_lsn() =
pg_last_wal_replay_lsn()
THEN 0 ELSE
EXTRACT (EPOCH FROM now() - pg_last_xact_replay_timestamp()) END AS lag_seconds;
```

- Manage replication slots

```sql
-- Check existing replication slot details

SELECT redo_lsn, slot_name,restart_lsn, active,
round((redo_lsn-restart_lsn) / 1024 / 1024 / 1024, 2) AS GB_lag
FROM pg_control_checkpoint(), pg_replication_slots;

-- Create replication slots
SELECT pg_create_physical_replication_slot('slot_one');

-- Drop unused replication slots
SELECT pg_drop_replication_slot('slot_one');
```

- Find subscription details in logic replication

```sql
select * from pg_stat_subscription;
```

###  GENERIC

- Find autocommit setting in postgres

```sql
-- Bydefault autocommit is set to on in postgres. You can check the setting .

postgres# \echo :AUTOCOMMIT;
ON;

-- At session level you can change the autocommit setting :

postgres# \set AUTOCOMMIT OFF
```

- Run a query repeatedly automatically

```sql
-- You can use watch command to run a particular query repeatedly until you cancel it.
-- watch 3 , means for every 3 seconds, the previous query will be executed 

select count(*) from test;

postgres=# \watch 3
Tue 19 Apr 2022 08:18:17 PM +03 (every 3s)

count
-------
4226
(1 row)

Tue 19 Apr 2022 08:18:20 PM +03 (every 3s)

count
-------
4226
(1 row)
```


