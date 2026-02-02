## Backup and Restore Database 

### Configure bash_profile

```sh
export PATH=$PATH:/usr/pgsql-15/bin
export PGDATA=/var/lib/pgsql/15/data
export PGUSER=postgres
export PGPORT=5432
export PGDATABASE=postgres
export ARCHIVE_DIR=/backup/archive
export BACKUP_DIR=/backup/dump
```

#### Full Database Dump Backup 

```sh
pg_dump --port=5432 --dbname=abc --username=postgres --format=d --file=/backup/dump/full_db.dump -v
```

#### Single Schema Backup 

```sh 
pg_dump --port=5432 --dbname=abc --username=postgres --schema=abg --format=d --file=/backup/dump/schema.dump -v
```

#### Multiple schema Backup 

```sh
pg_dump --port=5432 --dbname=abc --username=postgres --schema=abg --schema=account --format=d --file=/backup/dump/multi_schema_db.dump -v
```

#### Single Table Backup 

```sh
pg_dump --port=5432 --dbname=abc --username=postgres --table=abg.customer --format=d --file=/backup/dump/single_table.dump -v
```

#### Multiple table Backup 

```sh 
pg_dump --port=5432 --dbname=abc --username=postgres --table=abg.customer --table=abg.cus --format=d --file=/backup/dump/multi_table.dump -v
```

### Database Restore 

#### Full Database restore 

####  Create database 
```sh 
CREATE DATABASE "abc"
WITH
  OWNER = "app_user"
  TEMPLATE = "template1"
  ENCODING = 'UTF8'
  TABLESPACE = "app_tbs"
  LC_COLLATE = 'en_US.UTF-8'
  LC_CTYPE = 'en_US.UTF-8'
;
```
#### create tablespace and directory 
```sh
### create directory 
mkdir -p /pgdata/ts_app
chown postgres:postgres /pgdata/ts_app
chmod 700 /pgdata/ts_app

### Create tablespace
CREATE TABLESPACE ts_app
LOCATION '/pgdata/ts_app';

### Grant Table space permission 
GRANT CREATE ON TABLESPACE ts_app TO app_user;

```

#### create user and give proper permission 

```sh
CREATE USER app_user WITH PASSWORD 'StrongPassword@123';
## Grant super user 
ALTER USER app_user WITH SUPERUSER;
### Normal grant 
GRANT CONNECT ON DATABASE app_db TO app_user;
```
#### Full Database Import 

```sh
pg_restore --port=5432 --dbname=abc --username=postgres -j 4 /backup/dump/pktwdbp_2026-01-28_15-35-45.dump -v

### j 4 ---> parallel import 4 core 
```

#### Restore table from full database 

```sh
pg_restore --port=5432  --dbname=abc --username=postgres --schema=abg --table=cus /backup/dump/full_db.dump -v
```

#### Restore schema from full database 

- create schema and give permission 
```sh
CREATE SCHEMA abg AUTHORIZATION app_user;
```

```sh
pg_restore --port=5432  --dbname=abc --username=postgres --schema=abg /backup/dump/full_db.dump -v
```
