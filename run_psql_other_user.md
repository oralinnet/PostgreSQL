# Running PostgreSQL Commands on Another User

This guide shows how to run SQL commands on another PostgreSQL user with password authentication.

## Steps

1. Edit the PostgreSQL authentication configuration:
```bash
vim $PGDATA/pg_hba.conf
```

2. Ensure the following line is present in the configuration:
```conf
local   all             all                                     scram-sha-256
```

3. Reload PostgreSQL configuration:
```bash
/usr/pgsql-15/bin/pg_ctl -D $PGDATA reload
```

4. Run SQL command using connection string with password:
```bash
psql postgresql://postgres:password@192.168.100.5/postgres <<EOF
SELECT pg_database.datname as "database_name", 
       pg_database_size(pg_database.datname)/1024/1024 AS size_in_mb 
FROM pg_database 
ORDER by size_in_mb DESC;
EOF
```

## Note
This example shows how to query database sizes across all databases in the PostgreSQL instance.