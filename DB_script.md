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

### OBJECT MANAGEMENT

- Create/drop table in postgres

```sql
-- Create a simple table
Create table member_table ( mem_id integer, member_name varchar(100) , mobile integer not null);

-- Create table with primary key
Create table member_table ( mem_id integer primary key, member_name varchar(100) , mobile integer not null);

-- Create table under particular tablespace
Create table member_table ( mem_id integer primary key, member_name varchar(100) , mobile integer not null) tablespace pg_production_ts;

-- Create table with unique constraint
Create table member_table ( mem_id integer, member_name varchar(100) , mobile integer not null , constraint mem_id_cons unique(mem_id));

-- Create temporary table:
Create temporary table member_table ( mem_id integer primary key, member_name varchar(100) , mobile integer not null) tablespace pg_default;

-- Drop table
drop table member_table;
```

- Create/drop index commands in postgres

```sql
-- Simple create index:
create index tab_idx2 on scott.customer(emp_name);

-- Create index with tablespace:

CREATE INDEX tab_idx2 on scott.customer(emp_name) TABLESPACE IND_TS;

-- Create index without causing blocking:

create index concurrently tab_idx2 on scott.customer(emp_name) TABLESPACE IND_TS;

-- Create unique index:
create unique index tab_idx2 on scott.customer(emp_name)

-- Create functional index:
create index fun_idx on scott.customer(lower(emp_name));

-- Create multi column index:
create index multi_idx on scott.customer(emp_name,emp_id);

-- drop an index:

drop index fun_idx;

```

- Find list of schemas in postgres

```sql
-- Below of any commands can be used to find the schema details:

select schema_name,schema_owner from information_schema.schemata;

select nspname as schema_name , pg_get_userbyid(nspowner) as schema_owner from pg_catalog.pg_namespace;

postgres=# \dn+

```

- list of objects presents in a schema

```sql
--Below is for finding objects under schema scott: Replace your schema_name with scott

SELECT n.nspname as "Schema",
c.relname as "Name",
CASE c.relkind WHEN 'r' THEN 'table' WHEN 'v' THEN 'view' WHEN 'm' THEN 'materialized view' WHEN 'i' THEN 'index' WHEN 'S' THEN 'sequence' WHEN 's' THEN 'special' WHEN 'f' THEN 'foreign table' WHEN 'p' THEN 'partitioned table' WHEN 'I' THEN 'partitioned index' END as "Type",
pg_catalog.pg_get_userbyid(c.relowner) as "Owner"
FROM pg_catalog.pg_class c
LEFT JOIN pg_catalog.pg_namespace n ON n.oid = c.relnamespace
where n.nspname ='scott'
AND pg_catalog.pg_table_is_visible(c.oid)
ORDER BY 1,2;

-- NOTE: Make sure that, the schema_name for which you are looking for objects, is present in the search_path of that user. Otherwise it wont return any rows

show search_path;

```

- Find schema wise size in postgres db

```sql
-- Below queries can be used to get schema wise size in postgres db

select schemaname,pg_size_pretty(sum(pg_relation_size(quote_ident(schemaname) || '.' || quote_ident(tablename)))::bigint) as schema_size FROM pg_tables group by schemaname;

SELECT schemaname,
pg_size_pretty(sum(table_size)::bigint) as schema_size,
(sum(table_size) / pg_database_size(current_database())) * 100 as percentage_of_total_db
FROM (
SELECT pg_catalog.pg_namespace.nspname as schemaname,
pg_relation_size(pg_catalog.pg_class.oid) as table_size
FROM pg_catalog.pg_class
JOIN pg_catalog.pg_namespace ON relnamespace = pg_catalog.pg_namespace.oid
) t
GROUP BY schemaname
ORDER BY schemaname;

```

- Find top 10 big tables in postgres

```sql
-- Top 10 big tables in postgres

select schemaname as schema_owner,
relname as table_name,
pg_size_pretty(pg_total_relation_size(relid)) as total_size,
pg_size_pretty(pg_relation_size(relid)) as used_size,
pg_size_pretty(pg_total_relation_size(relid) - pg_relation_size(relid))
as free_space
from pg_catalog.pg_statio_user_tables
order by pg_total_relation_size(relid) desc,
pg_relation_size(relid) desc
limit 10;

(or)

SELECT
nspname as schema_name,relname as table_name,pg_size_pretty(pg_relation_size(c.oid)) as "table_size"
from pg_class c left join pg_namespace n on ( n.oid=c.relnamespace)
where nspname not in ('pg_catalog','information_schema')
order by pg_relation_size(c.oid) desc limit 10;

 
```

- Find tables and its index sizes

```sql
-- Find table sizes and its respective index sizes

SELECT
table_name,
pg_size_pretty(table_size) AS table_size,
pg_size_pretty(indexes_size) AS indexes_size,
pg_size_pretty(total_size) AS total_size
FROM (
SELECT
table_name,
pg_table_size(table_name) AS table_size,
pg_indexes_size(table_name) AS indexes_size,
pg_total_relation_size(table_name) AS total_size
FROM (
SELECT ('"' || table_schema || '"."' || table_name || '"') AS table_name
FROM information_schema.tables
) AS all_tables
ORDER BY total_size DESC
) AS pretty_sizes limit 10;
```

- List down index details in postgres

```sql
--- It wil find the indexes present on a table 'test'

select * from pg_indexes where tablename='test';

-- All indexes present in database:

select * from pg_indexes

-- It will show all index details including size:

postgres=# \di+

-- Find indexes with respective column name for table( here table name is test)

-- REFERENCE - https://stackoverflow.com/questions/2204058/list-columns-with-indexes-in-postgresql

select
t.relname as table_name,
i.relname as index_name,
array_to_string(array_agg(a.attname), ', ') as column_names
from
pg_class t,
pg_class i,
pg_index ix,
pg_attribute a
where
t.oid = ix.indrelid
and i.oid = ix.indexrelid
and a.attrelid = t.oid
and a.attnum = ANY(ix.indkey)
and t.relkind = 'r'
and t.relname ='test'
group by
t.relname,
i.relname
order by
t.relname,
i.relname;
```

- Find the size of a column 

```sql
-- Describe the table:

postgres=# \d test

-- Find the column size ( for sourcefile and sourceline)

select pg_size_pretty(sum(pg_column_size(sourcefile))) as total_size from test;

select pg_size_pretty(sum(pg_column_size(sourceline))) as total_size from test;

```

- Find respective physical file of a table/index

```sql
-- For getting the physical location of a table:
select pg_relation_filepath('test');

-- For getting the physical location of an index:

select pg_relation_filepath('test_idx');

```

- Find list of views present

```sql
select * from pg_views where schemaname not in ('pg_catalog','information_schema','sys');

postgres#\dv

```

- Find list of views present

```sql
select * from pg_views where schemaname not in ('pg_catalog','information_schema','sys');

postgres#\dv

```

- Manage sequences in postgres

```sql
-- Find the sequence details:

select * from pg_sequences;

(or)

\ds+

-- Create sequences:

CREATE SEQUENCE class_seq INCREMENT 1 MINVALUE 1 MAXVALUE 1000 START 1;

-- Create sequence in descending:

CREATE SEQUENCE class_seq INCREMENT -1 MINVALUE 1 MAXVALUE 1000 START 1000;

-- Alter sequence to change maxvalue:
alter sequence class_seq maxvalue 500;

-- Reset a sequence using alter command:
alter sequence class_seq restart with 1;

-- Find next_val and currval of a sequence:

select nextval('class_seq');
select currval('class_seq');

```

- Create Partial index in postgres

```sql
-- Partial index, means index will be created on a specific subset of data of a table.

create index part_emp_idx on orders(tax) where tax > 400;

edbstore=> \d part_emp_idx
Index "edbuser.part_emp_idx"
Column  | Type          | Key? | Definition
--------+---------------+------+------------
tax     | numeric(12,2) | yes  | tax
btree, for table "edbuser.orders", predicate (tax > 400::numeric)
```

-- Find foreign key details in postgres

```sql
SELECT
o.conname AS constraint_name,
(SELECT nspname FROM pg_namespace WHERE oid=m.relnamespace) AS source_schema,
m.relname AS source_table,
(SELECT a.attname FROM pg_attribute a WHERE a.attrelid = m.oid AND a.attnum = o.conkey[1] AND a.attisdropped = false) AS source_column,
(SELECT nspname FROM pg_namespace WHERE oid=f.relnamespace) AS target_schema,
f.relname AS target_table,
(SELECT a.attname FROM pg_attribute a WHERE a.attrelid = f.oid AND a.attnum = o.confkey[1] AND a.attisdropped = false) AS target_column
FROM
pg_constraint o LEFT JOIN pg_class f ON f.oid = o.confrelid LEFT JOIN pg_class m ON m.oid = o.conrelid
WHERE
o.contype = 'f' AND o.conrelid IN (SELECT oid FROM pg_class c WHERE c.relkind = 'r');

-- REFERENCE - https://stackoverflow.com/questions/1152260/postgres-sql-to-list-table-foreign-keys
```

- Find specific table/index size

```sql
postgres=# \d test
Table "public.test"

-- Find the table_size ( excluding the index_size)

SELECT pg_size_pretty (pg_relation_size('test'));

-- Find the total_index size of the table

SELECT pg_size_pretty ( pg_indexes_size('test'));

-- Find particular index size:

select pg_size_pretty(pg_total_relation_size('test_idx'));

select pg_size_pretty(pg_total_relation_size('test_idx2'));

-- Another method:

postgres=# \di+ "test_idx"

```

- Find list of partitioned table details

```sql
-- List down all partitioned tables present in db

SELECT
nmsp_parent.nspname AS parent_schema,
parent.relname AS parent,
nmsp_child.nspname AS child_schema,
child.relname AS child
FROM pg_inherits
JOIN pg_class parent ON pg_inherits.inhparent = parent.oid
JOIN pg_class child ON pg_inherits.inhrelid = child.oid
JOIN pg_namespace nmsp_parent ON nmsp_parent.oid = parent.relnamespace
JOIN pg_namespace nmsp_child ON nmsp_child.oid = child.relnamespace

-- List down all partitions of a single table:

SELECT
nmsp_parent.nspname AS parent_schema,
parent.relname AS parent,
nmsp_child.nspname AS child_schema,
child.relname AS child
FROM pg_inherits
JOIN pg_class parent ON pg_inherits.inhparent = parent.oid
JOIN pg_class child ON pg_inherits.inhrelid = child.oid
JOIN pg_namespace nmsp_parent ON nmsp_parent.oid = parent.relnamespace
JOIN pg_namespace nmsp_child ON nmsp_child.oid = child.relnamespace
WHERE parent.relname='parent_table_name';

-- Ref link - > https://dba.stackexchange.com/questions/40441/get-all-partition-names-for-a-table

```


### USER MANAGEMENT

- List users present in postgres

```sql
--- List users present in postgres:

select usename,usesuper,valuntil from pg_user;

select usename,usesuper,valuntil from pg_shadow;

select usename,usesuper,valuntil from pg_shadow;

postgres=# \du

-- NOTE - > \du command output includes both user and roles(custom created roles only).

-- postgres users are bydefault role, but roles are not bydefault user.
```

- List roles present in postgres

```sql
-- List roles :

select rolname,rolcanlogin,rolvaliduntil from pg_roles;

 
-- rolcanlogin - > If true mean they are role as well as user
--                 If false mean they are only role( they cannot login)

-- NOTE - > In postgres users are bydefault role, but roles are not bydefault user. i.e

-- Bydefault user come with login privilege, where as roles don’t come with login privilege.

```

- create/drop user in postgres

```sql
-- CREATE USER:
create user TEST_DBACLASS with password 'test123';

-- CREATE USER WITH VALID UNTIL:

create user TEST_dbuser1 with password 'test123' valid until '2020-08-08';

-- CREATE USER WITH SUPER USER PRIVILEGE

create user test_dbuser3 with password 'test123' CREATEDB SUPERUSER;


-- VIEW USERS:

select usename,valuntil,usecreatedb from pg_shadow;

select usename,usesuper,valuntil from pg_user;

dbaclass=# \du+

drop user DB_user1;
```

- Create/drop role in postgres

```sql
 -- Create role :

create role dev_admin;

create role dev_admin with valid until '10-oct-2020';

-- role with createdb and superuser privilege and login keyword mean it can login to db like a normal user

create role dev_admin with createdb createrole login ;

-- DROP ROLE:

drop role dev_admin;

select rolname,rolcanlogin,rolvaliduntil from pg_roles;
```
