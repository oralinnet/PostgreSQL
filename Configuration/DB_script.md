# PostgreSQL Database Monitoring and Management Guide

This guide provides comprehensive SQL queries and commands for monitoring and managing PostgreSQL databases.

## Table of Contents
- [Connection Management](#connection-management)
- [Database Management](#database-management)
- [Maintenance](#maintenance)
- [Tablespace Management](#tablespace-management)
- [Auditing & Security](#auditing--security)
- [Replication](#replication)
- [Generic](#generic)
- [Object Management](#object-management)
- [User Management](#user-management)
- [Backup & Recovery](#backup--recovery)
- [Network](#network)
- [Performance](#performance)

## Connection Management

### Active Connections
```sql
select max_conn,used,res_for_super,max_conn-used-res_for_super res_for_normal 
from 
  (select count(*) used from pg_stat_activity) t1,
  (select setting::int res_for_super from pg_settings where name=$$superuser_reserved_connections$$) t2,
  (select setting::int max_conn from pg_settings where name=$$max_connections$$) t3;
```

### Terminate Idle Connections
```sql
SELECT pg_terminate_backend(pid) 
FROM   pg_stat_activity 
WHERE  state = 'idle' 
       AND state_change < now() - '15min'::interval; 
```

### Find Long-Running Queries
```sql
SELECT pid, now() - pg_stat_activity.query_start AS duration, query 
FROM pg_stat_activity 
WHERE (now() - pg_stat_activity.query_start) > interval '2 minutes';
```

### Kill Session
```sql
SELECT pg_terminate_backend(8428);

select pg_terminate_backend(pid) 
from pg_stat_activity
where pid = '1344279';
```

### Find Queries Running Longer Than 5 Minutes
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

### Kill Long-Running PostgreSQL Query Processes
```sql
-- Attempt to gracefully kill a running query process
pg_cancel_backend(pid) 

-- Immediately kill the running query process (use as last resort)
pg_terminate_backend(pid)
```

### Active Sessions
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

## Database Management

### Create Database
```sql
-- Basic database creation
create database DBATEST;

-- Create with Tablespace
create database DBATEST with tablespace ts_postgres;

-- Create with Tablespace and template
CREATE DATABASE "DBATEST"
WITH TABLESPACE ts_postgres
OWNER "postgres"
ENCODING 'UTF8'
LC_COLLATE = 'en_US.UTF-8'
LC_CTYPE = 'en_US.UTF-8'
TEMPLATE template0;

-- View database information
select * from pg_database;
```

### Connect to PostgreSQL
```sql
-- Connect to database
psql -d edb -U postgres -h hostname/IP

-- Find current connection info
postgres=# \conninfo
select current_schema,current_user,session_user,current_database();

-- Switch to another database
postgres-# \c testdb
```

### Drop Database
```sql
-- Note: Connect to a different database before dropping
drop database "testdb";

-- If drop fails due to active connections
select application_name,client_hostname,pid,usename from pg_stat_activity where datname='testdb';
-- Kill Session with session id 
select pg_terminate_backend(pid) from pg_stat_activity where pid='12755';
```

### Database Details
```sql
-- List databases
postgres=# \list+
select datname from pg_database;
```

### Database Size
```sql
SELECT pg_database.datname as "database_name", 
       pg_size_pretty(pg_database_size(pg_database.datname)) AS size_in_mb 
FROM pg_database 
ORDER by size_in_mb DESC;
postgres=# \l+
```

### Timezone Information
```sql
show timezone;
SELECT current_setting('TIMEZONE');
select name,setting,short_desc,boot_val from pg_settings where name='TimeZone';
```

### PostgreSQL Version
```sql
show server_version;
select version();
```

### Enable WAL Archiving
```sql
-- Create directory for archiving
mkdir -p /archive/location

-- Update postgres.conf
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

### Rotate Server Log
```sql
-- Signal log-file manager to switch to new output file
select pg_rotate_logfile();
```

### Query Execution Time
```sql
-- Monitor query execution time
select substr(query,1,100) query,
       calls,
       min_time/1000 "min_time(in sec)",
       max_time/1000 "max_time(in sec)",
       mean_time/1000 "avg_time(in sec)",
       rows 
from pg_stat_statements 
order by mean_time desc;

-- Enable pg_stat_statements
-- Add to postgres.conf:
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all

-- Restart PostgreSQL
sudo service postgresql restart

-- Create extension
create extension pg_stat_statements;
```

### Data Directory Location
```sql
-- Show data directory
show data_directory;
select setting from pg_settings where name = 'data_directory';

-- Show location of important files
SELECT name, setting FROM pg_settings WHERE category = 'File Locations';
```

## Maintenance

### Update Statistics
```sql
-- Analyze stats for a table
analyze testanalyze;

-- Analyze selected columns
analyze dbatest.emptab (datname,datdba);

-- Check analyze status
select relname,reltuples from pg_class where relname in ('testanalyze','emptab');
select schemaname,relname,analyze_count,last_analyze,last_autoanalyze 
from pg_stat_user_tables 
where relname in ('testanalyze','emptab');

-- Analyze with verbose
analyze verbose dbatest.emptab (datname,datdba);

-- Analyze all tables in current schema
analyze;
```

### Vacuum Operations
```sql
-- Basic vacuum
vacuum dbatest.emptab;

-- Vacuum and analyze
vacuum analyze dbatest.emptab;

-- Vacuum with verbose
vacuum verbose analyze dbatest.emptab;

-- Check vacuum status
select schemaname,relname,last_vacuum,vacuum_count 
from pg_stat_user_tables 
where relname='emptab';
```

### Vacuum Full
```sql
-- Vacuum full (requires exclusive lock)
VACUUM FULL dbatest.emptab;

-- Check space usage
select pg_size_pretty(pg_relation_size('dbatest.emptab'));
```

### Autovacuum Management
```sql
-- Check autovacuum status
select name,setting,short_desc,boot_val,pending_restart 
from pg_settings 
where name in ('autovacuum','track_counts');

-- View autovacuum settings
select name,setting,short_desc,min_val,max_val,enumvals,boot_val,pending_restart 
from pg_settings 
where category like 'Autovacuum';

-- Change autovacuum settings
alter system set autovacuum_max_workers=10;
```

### Index Management
```sql
-- Rebuild specific index
REINDEX INDEX TEST_IDX2;

-- Rebuild all indexes on table
REINDEX TABLE TEST;

-- Rebuild schema indexes
reindex schema public;

-- Rebuild database indexes
reindex database dbaclass;

-- Reindex with verbose
reindex (verbose) table test;

-- Rebuild index without locking
REINDEX (verbose) table concurrently test;
```

### Monitor Index Operations
```sql
SELECT a.query,
       p.phase,
       p.blocks_total,
       p.blocks_done,
       p.tuples_total,
       p.tuples_done 
FROM pg_stat_progress_create_index p 
JOIN pg_stat_activity a ON p.pid = a.pid;

-- Alternative query
select pid,
       datname,
       command,
       phase,
       tuples_total,
       tuples_done,
       partitions_total,
       partitions_done 
from pg_stat_progress_create_index;
```

### Monitor Vacuum Operations
```sql
select * from pg_stat_progress_vacuum;
```

### Column Statistics
```sql
-- Check column statistics level
SELECT attname as column_name,
       attstattarget as stats_level 
FROM pg_attribute 
WHERE attrelid = (SELECT oid FROM pg_class WHERE relname = 'orders') 
and attname='orderdate';

-- Change statistics level
alter table orders alter column orderdate set statistics 1000;
```

### Table Vacuum Settings
```sql
SELECT n.nspname,
       c.relname,
       pg_catalog.array_to_string(c.reloptions || array(
         select 'toast.' || x 
         from pg_catalog.unnest(tc.reloptions) x),', ') as relopts
FROM pg_catalog.pg_class c
LEFT JOIN pg_catalog.pg_class tc ON (c.reltoastrelid = tc.oid)
JOIN pg_namespace n ON c.relnamespace = n.oid
WHERE c.relkind = 'r'
AND nspname NOT IN ('pg_catalog', 'information_schema');
```

### Modify Autovacuum Settings
```sql
-- Disable autovacuum
alter table test2 set(autovacuum_enabled = off);

-- Enable autovacuum
alter table test2 set(autovacuum_enabled = on);
```

### Check Table Statistics
```sql
select * from pg_stat_user_tables where relname='test';
```

### Check Table Bloat
```sql
-- Create extension
create extension pgstattuple;

-- Check table bloat
SELECT pg_size_pretty(pg_relation_size('test')) as table_size,
       (pgstattuple('test')).dead_tuple_percent;

-- Check index bloat
select pg_relation_size('test_x_idx') as index_size,
       100-(pgstatindex('test_x_idx')).avg_leaf_density as bloat_ratio;
```

## Tablespace Management

### View Tablespace Info
```sql
-- List tablespaces
select * from pg_tablespace;
postgres=# \db+

-- Get tablespace size
select pg_size_pretty(pg_tablespace_size('ts_dbaclass'));
```

### Create/Drop/Rename Tablespace
```sql
-- Create tablespace
create tablespace ts_postgres location '/Library/PostgreSQL/TEST/TS_POSTGRES';

-- Rename tablespace
alter tablespace ts_postgres rename to ts_dbaclass;

-- Drop tablespace
drop tablespace ts_dbaclass;
```

### Default Tablespace
```sql
-- View default tablespace
show default_tablespace;

-- Change default tablespace
alter system set default_tablespace=ts_postgres;
select pg_reload_conf();
show default_tablespace;
SELECT name, setting FROM pg_settings where name='default_tablespace';

-- Set at session level
set default_tablespace=ts_postgres;
```

### Temp Tablespace
```sql
-- View temp tablespace
SELECT name, setting FROM pg_settings where name='temp_tablespaces';
show temp_tablespaces

-- Change temp tablespace
alter system set temp_tablespaces=TS_TEMP;
select pg_reload_conf();
show temp_tablespaces;
SELECT name, setting FROM pg_settings where name='temp_tablespaces';
```

### Tablespace Ownership
```sql
-- Change tablespace owner
alter tablespace ts_postgres owner to dev_admin;
\db+
```

### Move Objects Between Tablespaces
```sql
-- Move table
alter table TEST8 set tablespace pg_crm;

-- Move index
alter index TEST_ind set tablespace pg_crm;

-- Move database
alter database prod_crm set tablespace crm_tblspc;
```

## Auditing & Security

### View pg_hba.conf
```sql
select * from pg_hba_file_rules;
```

### Enable Statement Logging
```sql
-- Check current setting
show log_statement;

-- Log DDL activities
alter system set log_statement=ddl;
select pg_reload_conf();

-- Log DDL and DML activities
alter system set log_statement=mod;
select pg_reload_conf();

-- Log all statements
alter system set log_statement='all';
select pg_reload_conf();
```

### Connection Logging
```sql
-- Enable connection logging
select name,setting from pg_settings 
where name in ('log_disconnections','log_connections');

alter system set log_disconnections=off;
alter system set log_connections=on;
select pg_reload_conf();
```

## Replication

### Check Recovery Status
```sql
-- Run on standby server
select pg_is_wal_replay_paused();
```

### Replication Details
```sql
-- Run on primary server
select * from pg_stat_replication;
```

### WAL Status
```sql
-- Run on standby
select pg_last_wal_receive_lsn(),
       pg_last_wal_replay_lsn(),
       pg_last_xact_replay_timestamp();
select * from pg_stat_wal_receiver;
```

### Control Recovery
```sql
-- Pause recovery
select pg_wal_replay_pause();
select pg_is_wal_replay_paused();

-- Resume recovery
select pg_wal_replay_resume();
select pg_is_wal_replay_paused();
```

### Check Replication Lag
```sql
-- Lag in bytes
SELECT pg_wal_lsn_diff(sent_lsn, replay_lsn) 
from pg_stat_replication;

-- Lag in seconds
SELECT CASE 
         WHEN pg_last_wal_receive_lsn() = pg_last_wal_replay_lsn()
         THEN 0 
         ELSE EXTRACT(EPOCH FROM now() - pg_last_xact_replay_timestamp()) 
       END AS lag_seconds;
```

### Replication Slots
```sql
-- Check slots
SELECT redo_lsn,
       slot_name,
       restart_lsn,
       active,
       round((redo_lsn-restart_lsn) / 1024 / 1024 / 1024, 2) AS GB_lag
FROM pg_control_checkpoint(),
     pg_replication_slots;

-- Create slot
SELECT pg_create_physical_replication_slot('slot_one');

-- Drop slot
SELECT pg_drop_replication_slot('slot_one');
```

### Logical Replication
```sql
select * from pg_stat_subscription;
```

## Generic

### Autocommit Setting
```sql
-- Check setting
postgres# \echo :AUTOCOMMIT;
ON;

-- Change setting
postgres# \set AUTOCOMMIT OFF
```

### Repeated Query Execution
```sql
-- Execute query every 3 seconds
select count(*) from test;
postgres=# \watch 3
```

## Object Management

### Table Management
```sql
-- Create table
Create table member_table (
    mem_id integer,
    member_name varchar(100),
    mobile integer not null
);

-- Create table with primary key
Create table member_table (
    mem_id integer primary key,
    member_name varchar(100),
    mobile integer not null
);

-- Create table in tablespace
Create table member_table (
    mem_id integer primary key,
    member_name varchar(100),
    mobile integer not null
) tablespace pg_production_ts;

-- Create table with unique constraint
Create table member_table (
    mem_id integer,
    member_name varchar(100),
    mobile integer not null,
    constraint mem_id_cons unique(mem_id)
);

-- Create temporary table
Create temporary table member_table (
    mem_id integer primary key,
    member_name varchar(100),
    mobile integer not null
) tablespace pg_default;

-- Drop table
drop table member_table;
```

### Index Management
```sql
-- Create index
create index tab_idx2 on scott.customer(emp_name);

-- Create index in tablespace
CREATE INDEX tab_idx2 on scott.customer(emp_name) TABLESPACE IND_TS;

-- Create index without blocking
create index concurrently tab_idx2 on scott.customer(emp_name) TABLESPACE IND_TS;

-- Create unique index
create unique index tab_idx2 on scott.customer(emp_name);

-- Create functional index
create index fun_idx on scott.customer(lower(emp_name));

-- Create multi-column index
create index multi_idx on scott.customer(emp_name,emp_id);

-- Drop index
drop index fun_idx;
```

### Schema Management
```sql
-- List schemas
select schema_name,schema_owner from information_schema.schemata;
select nspname as schema_name,
       pg_get_userbyid(nspowner) as schema_owner 
from pg_catalog.pg_namespace;
postgres=# \dn+

-- List objects in schema
SELECT n.nspname as "Schema",
       c.relname as "Name",
       CASE c.relkind 
           WHEN 'r' THEN 'table' 
           WHEN 'v' THEN 'view' 
           WHEN 'm' THEN 'materialized view' 
           WHEN 'i' THEN 'index' 
           WHEN 'S' THEN 'sequence' 
           WHEN 's' THEN 'special' 
           WHEN 'f' THEN 'foreign table' 
           WHEN 'p' THEN 'partitioned table' 
           WHEN 'I' THEN 'partitioned index' 
       END as "Type",
       pg_catalog.pg_get_userbyid(c.relowner) as "Owner"
FROM pg_catalog.pg_class c
LEFT JOIN pg_catalog.pg_namespace n ON n.oid = c.relnamespace
where n.nspname ='scott'
AND pg_catalog.pg_table_is_visible(c.oid)
ORDER BY 1,2;
```

### Schema Size
```sql
select schemaname,
       pg_size_pretty(sum(pg_relation_size(quote_ident(schemaname) || '.' || quote_ident(tablename)))::bigint) as schema_size 
FROM pg_tables 
group by schemaname;

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

### Top Tables by Size
```sql
select schemaname as schema_owner,
       relname as table_name,
       pg_size_pretty(pg_total_relation_size(relid)) as total_size,
       pg_size_pretty(pg_relation_size(relid)) as used_size,
       pg_size_pretty(pg_total_relation_size(relid) - pg_relation_size(relid)) as free_space
from pg_catalog.pg_statio_user_tables
order by pg_total_relation_size(relid) desc,
         pg_relation_size(relid) desc
limit 10;
```

### Table and Index Sizes
```sql
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
) AS pretty_sizes 
limit 10;
```

### Index Details
```sql
-- List indexes on table
select * from pg_indexes where tablename='test';

-- List all indexes
select * from pg_indexes;
postgres=# \di+

-- List indexes with columns
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

### Column Size
```sql
-- Get column size
select pg_size_pretty(sum(pg_column_size(sourcefile))) as total_size from test;
select pg_size_pretty(sum(pg_column_size(sourceline))) as total_size from test;
```

### Physical File Location
```sql
-- Get table file location
select pg_relation_filepath('test');

-- Get index file location
select pg_relation_filepath('test_idx');
```

### View Management
```sql
-- List views
select * from pg_views 
where schemaname not in ('pg_catalog','information_schema','sys');
postgres#\dv
```

### Sequence Management
```sql
-- List sequences
select * from pg_sequences;
postgres#\ds+

-- Create sequence
CREATE SEQUENCE class_seq 
    INCREMENT 1 
    MINVALUE 1 
    MAXVALUE 1000 
    START 1;

-- Create descending sequence
CREATE SEQUENCE class_seq 
    INCREMENT -1 
    MINVALUE 1 
    MAXVALUE 1000 
    START 1000;

-- Alter sequence
alter sequence class_seq maxvalue 500;
alter sequence class_seq restart with 1;

-- Get sequence values
select nextval('class_seq');
select currval('class_seq');
```

### Partial Index
```sql
-- Create partial index
create index part_emp_idx on orders(tax) where tax > 400;
```

### Foreign Key Details
```sql
SELECT
    o.conname AS constraint_name,
    (SELECT nspname FROM pg_namespace WHERE oid=m.relnamespace) AS source_schema,
    m.relname AS source_table,
    (SELECT a.attname FROM pg_attribute a 
     WHERE a.attrelid = m.oid 
     AND a.attnum = o.conkey[1] 
     AND a.attisdropped = false) AS source_column,
    (SELECT nspname FROM pg_namespace WHERE oid=f.relnamespace) AS target_schema,
    f.relname AS target_table,
    (SELECT a.attname FROM pg_attribute a 
     WHERE a.attrelid = f.oid 
     AND a.attnum = o.confkey[1] 
     AND a.attisdropped = false) AS target_column
FROM
    pg_constraint o 
    LEFT JOIN pg_class f ON f.oid = o.confrelid 
    LEFT JOIN pg_class m ON m.oid = o.conrelid
WHERE
    o.contype = 'f' 
    AND o.conrelid IN (SELECT oid FROM pg_class c WHERE c.relkind = 'r');
```

### Object Size
```sql
-- Get table size
SELECT pg_size_pretty(pg_relation_size('test'));

-- Get index size
SELECT pg_size_pretty(pg_indexes_size('test'));

-- Get specific index size
select pg_size_pretty(pg_total_relation_size('test_idx'));
select pg_size_pretty(pg_total_relation_size('test_idx2'));
```

### Partitioned Tables
```sql
-- List all partitioned tables
SELECT
    nmsp_parent.nspname AS parent_schema,
    parent.relname AS parent,
    nmsp_child.nspname AS child_schema,
    child.relname AS child
FROM pg_inherits
JOIN pg_class parent ON pg_inherits.inhparent = parent.oid
JOIN pg_class child ON pg_inherits.inhrelid = child.oid
JOIN pg_namespace nmsp_parent ON nmsp_parent.oid = parent.relnamespace
JOIN pg_namespace nmsp_child ON nmsp_child.oid = child.relnamespace;

-- List partitions of specific table
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
```

## User Management

### List Users
```sql
-- List users
select usename,usesuper,valuntil from pg_user;
select usename,usesuper,valuntil from pg_shadow;
postgres=# \du
```

### List Roles
```sql
select rolname,rolcanlogin,rolvaliduntil from pg_roles;
```

### Create User
```sql
-- Create basic user
create user TEST_DBACLASS with password 'test123';

-- Create user with expiration
create user TEST_dbuser1 with password 'test123' valid until '2020-08-08';

-- Create superuser
create user test_dbuser3 with password 'test123' CREATEDB SUPERUSER;

-- View users
select usename,valuntil,usecreatedb from pg_shadow;
select usename,usesuper,valuntil from pg_user;
dbaclass=# \du+

-- Drop user
drop user DB_user1;
```

### Create Role
```sql
-- Create basic role
create role dev_admin;

-- Create role with expiration
create role dev_admin with valid until '10-oct-2020';

-- Create role with privileges
create role dev_admin with createdb createrole login;

-- Drop role
drop role dev_admin;

-- View roles
select rolname,rolcanlogin,rolvaliduntil from pg_roles;
```

### Modify User
```sql
-- Rename user
alter user dbatest rename to dbaprod;

-- Change password
alter user dbaprod password 'test';

-- Set expiration
alter user dbaprod valid until 'Feb 10 2021';
```

### Superuser Management
```sql
-- Check superuser status
select usename,usesuper from pg_user where usename='dbatest';

-- Grant superuser
alter user dbatest with superuser;

-- Revoke superuser
alter user dbatest with nosuperuser;
```

### Reset Password
```sql
-- Set initial password
alter user dbaprod password 'old';

-- Get encrypted password
SELECT rolname, rolpassword 
FROM pg_catalog.pg_authid 
where rolname='dbaprod';

-- Change password
alter user dbaprod password 'new';

-- Update with old encrypted password
update pg_catalog.pg_authid 
set rolpassword = 'md5bbb103edd695a83d45db75755e459a78' 
where rolname='dbaprod';
```

### Privilege Management
```sql
-- Grant privileges
GRANT CONNECT ON DATABASE PRIMDB to DBAUSER1;
GRANT USAGE ON SCHEMA CRM to DBAUSER1;
GRANT INSERT,UPDATE,DELETE ON TABLE CRM.EMPTAB TO DBAUSER1;
GRANT ALL ON TABLE CRM.EMPTAB TO DBAUSER1;
GRANT CREATE ALL ON DATABASE CRM to DBAUSER2;
GRANT CREATE ON TABLESPACE INV_TS to DBAUSER2;
GRANT ALL ON TABLESPACE INV_TS TO DBAUSER2;
GRANT CREATE ON TABLESPACE INV_TS to DBAUSER2 with grant option;
GRANT EXECUTE ON PROCEDURE PRIM_ID.TEST_PROC;
GRANT EXECUTE ON FUNCTION PRIM_ID.TEST_FUNC;

-- Revoke privileges
REVOKE CONNECT ON DATABASE PRIMDB FROM DBAUSER1;
REVOKE USAGE ON SCHEMA CRM FROM DBAUSER1;
REVOKE INSERT,UPDATE,DELETE ON TABLE CRM.EMPTAB FROM DBAUSER1;
REVOKE ALL ON TABLE CRM.EMPTAB FROM DBAUSER1;
REVOKE CREATE ALL ON DATABASE CRM FROM DBAUSER2;
REVOKE CREATE ON TABLESPACE INV_TS FROM DBAUSER2;
REVOKE ALL ON TABLESPACE INV_TS FROM DBAUSER2;
REVOKE CREATE ON TABLESPACE INV_TS FROM DBAUSER2;
REVOKE EXECUTE ON PROCEDURE PRIM_ID.TEST_PROC FROM DBAUSER2;
REVOKE EXECUTE ON FUNCTION PRIM_ID.TEST_FUNC FROM DBAUSER2;
```

### User Profile Management
```sql
-- Create profile
create profile REPORTING_PROFILE 
limit FAILED_LOGIN_ATTEMPTS 3 
PASSWORD_LIFE_TIME 90;

-- Alter profile
alter profile REPORTING_PROFILE 
limit FAILED_LOGIN_ATTEMPTS 1;

-- View profiles
select * from dba_profiles;
```

### Schema Management
```sql
-- Create schema
create schema dba_schema;

-- Create schema with owner
create schema dba_schema authorization raj2;

-- Drop schema
drop schema dba_schema;

-- List schemas
postgres=# \dn+
```

### Search Path Management
```sql
-- View search path
SELECT r.rolname,
       d.datname,
       drs.setconfig
FROM pg_db_role_setting drs
LEFT JOIN pg_roles r ON r.oid = drs.setrole
LEFT JOIN pg_database d ON d.oid = drs.setdatabase
WHERE d.datname = 'EDB';

-- Set search path
alter user prod_user in database "EDB" 
set search_path="$user", public, prim_db;

-- Set global search path
alter user prod_user 
set search_path="$user", public, prim_db;
```

### Privilege Information
```sql
-- Table privileges
SELECT table_catalog,
       table_schema,
       table_name,
       privilege_type
FROM information_schema.table_privileges
WHERE grantee = 'USER_NAME';

-- Usage privileges
select * from usage_privileges 
where grantee='USER_NAME';
```

### Role Membership
```sql
SELECT
    r.rolname,
    ARRAY(SELECT b.rolname
          FROM pg_catalog.pg_auth_members m
          JOIN pg_catalog.pg_roles b ON (m.roleid = b.oid)
          WHERE m.member = r.oid) as memberof
FROM pg_catalog.pg_roles r
ORDER BY 1;
```

## Backup & Recovery

### Export Data
```sql
-- Export specific columns
copy EMPLOYEE(EMP_NAME,EMP_ID) to '/tmp/emp.txt';

-- Export complete table
copy EMPLOYEE to '/tmp/emp.txt';

-- Export to CSV
copy EMPLOYEE to '/tmp/emp.csv' with csv headers;

-- Export query results
copy (select ename,depname from emp where depname='HR') 
to '/tmp/emp.csv' with csv headers;
```

## Network

### Foreign Server Management
```sql
-- List foreign servers
postgres=# \des+
select srvname,
       srvowner,
       srvoptions,
       fdwname,
       srvversion,
       srvtype 
from pg_foreign_server 
join pg_foreign_data_wrapper b on b.oid=srvfdw;

-- List foreign tables
postgres=# \det+

-- List foreign data wrappers
postgres=# \dew

-- List user mappings
postgres=# \deu+
```

### Database Link
```sql
-- Create extension
create extension dblink;

-- Create foreign server
CREATE SERVER oracle_dblink 
FOREIGN DATA WRAPPER dblink_fdw 
OPTIONS (host '10.21.120.131',
         dbname 'postgres',
         port '5444');

-- Create user mapping
CREATE USER MAPPING FOR enterprisedb 
SERVER oracle_dblink 
OPTIONS (user 'dba_raj',
         password 'dba_raj');

-- Test connection
SELECT dblink_connect('my_new_conn', 'oracle_dblink');

-- Query remote data
select * from dblink('oracle_dblink',
                    'select object_name from test') 
as test_object(object_name varchar);
```

### Foreign Server Configuration
```sql
-- Create server
CREATE SERVER oracle_dblink 
FOREIGN DATA WRAPPER dblink_fdw 
OPTIONS (host '10.21.120.131',
         dbname 'postgres',
         port '5444');

-- Add parameter
ALTER SERVER oracle_dblink 
options (ADD port '5444');

-- Modify parameter
ALTER SERVER oracle_dblink 
options (SET port '5432');

-- Drop server
DROP SERVER oracle_dblink CASCADE;
```

## Performance

### Foreign Server Details
```sql
postgres=# \des+
select srvname,
       srvowner,
       srvoptions,
       fdwname,
       srvversion,
       srvtype 
from pg_foreign_server 
join pg_foreign_data_wrapper b on b.oid=srvfdw;
```

