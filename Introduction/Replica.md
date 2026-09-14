## Configure PostgreSQL Streaming Replication 

### Prerequisite
- Two Or more PostgreSQL cluster with same version 
- Primary DB In Read Write Mode
- Replica Server only Binary Installed
- Both Server sudo or root access
- Ensure that Both servers can use ssh/scp passwordless communication
- Ensure Primary DB password base authentication active
- Install Package : rsync vim net-tools in Both server 
- Disabled Selinux 
- Open Port in Firewall 
- Primary IP: 192.168.10.10 Replica IP: 192.168.10.11

### Install PostgreSQL Binary on Replica Server 
```
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo dnf -qy module disable postgresql
sudo dnf install -y postgresql15-server
sudo systemctl enable postgresql-15
sudo passwd postgres
```

### If you want to give the postgres OS user sudo privileges
```
visudo
### add this line 
postgres ALL=(ALL) ALL      ### For password 
postgres ALL=(ALL) NOPASSWD: ALL        ### without password 
:x 
su - postgres
sudo -i
```


### Disabled Selinux and Install RPM package on Both Server 
```
sudo dnf install vim -y
sudo dnf install rsync -y 
sudo dnf install net-tools -y

sudo vim /etc/selinux/config
SELINUX=disabled
:x  ### save file 

sudo reboot 
```

### Configure ssh keybase authentication for postgres OS user in Both Servers
```
### Primary Server 
sudo su - postgres 
ssh-keygen
ssh-copy-id postgres@192.168.10.11
ssh postgres@192.168.10.11

### Replica site (If you have swithover plan )
sudo su - postgres 
ssh-keygen
ssh-copy-id postgres@192.168.10.10
ssh postgres@192.168.10.10
```
### configure firewall in both server
```
sudo firewall-cmd --permanent --add-port=5432/tcp
sudo firewall-cmd --reload 
```

### Primary Server Configuration 
- Enable Archive mode
- Configure postgresql.conf file
- Change archive log location (optional)

#### Enable Archive mode in Primary Server 
- check archive mode 
- create directory for archivelog 
```
### Check archive mode 
psql
show archive_mode; 
\q
### Create directory 
mkdir /var/lib/pgsql/15/archivelog  -p    ### (If you create directoy root or other location then give proper permission to postgres OS user and group)
sudo mkdir /archivelog
sudo chown -R postgres:postgres /archivelog
### Enable archivelog 
psql
show config_file;   ### postgresql.conf file location
\q
vim /var/lib/pgsql/15/data/postgresql.conf

promote_trigger_file = '/tmp/trigger_file.trg'          ### For swithover
archive_mode = on               ### archive mode on
archive_command = 'rsync -a %p postgres@192.168.10.11:/var/lib/pgsql/15/archivelog/%f'
wal_level = replica
max_wal_senders = 3
max_replication_slots = 4

:x save 

### Restart postgres service 
sudo systemctl restart postgresql-15.service
sudo systemctl status postgresql-15.service
```
#### Primary DB configure 
```
psql
show archive_mode;      #### Check archive mode 
CREATE ROLE replication WITH REPLICATION PASSWORD 'hello' LOGIN;        #### Create role for replication
\q

vim /var/lib/pgsql/15/data/postgresql.conf
listen_addresses = '*'		#### in production must use Production server ip 
:x 

vim /var/lib/edb/as15/data/pg_hba.conf
#### Replication server IP
host  replication     replication     192.168.10.11/32         scram-sha-256

:x 
sudo systemctl restart postgresql-15.service
sudo systemctl status postgresql-15.service

```
### Replica Server Configuration 
- Install postgreSQL binary Only 
- Configure Firewall and sudo access 
- Stop postgresql service (very Importance)
- Directory Stature should be same like data directory, archivelog location 

#### Update PostgreSQL PATH in postgres user
```
sudo su - postgres
vim .bash_profile
export PATH=$PATH:/usr/pgsql-15/bin/
### Now reload bash_profile or logout and login postgres user
```
#### Replication Server DB configuration 
```
sudo systemctl status postgresql-15.service
sudo systemctl stop postgresql-15.service

mkdir /var/lib/pgsql/15/archivelog  -p 

pg_basebackup -h 192.168.10.10 -U postgres -D /var/lib/pgsql/15/data -U replication -v -P --wal-method=stream --write-recovery-conf
### 192.168.10.10   primary server ip, 
### postgres Database user, 
### /var/lib/pgsql/15/data directory location replica server, replication DB user 
touch /var/lib/pgsql/15/data/standby.signal

sudo systemctl start postgresql-15.service
sudo systemctl status postgresql-15.service
```

### Replication Status Check from primary DB

```
psql
select usename, application_name, client_addr, state, sync_priority, sync_state from pg_stat_replication;
```

### Check replication in data level 
- Primary Server 
```
psql
create table test (id int,name varchar(30));
insert into test values (01,'raju');
```
- Replica Server 
```
psql
select * from test;
```

### Replication mode change async to sync replica
- Default replication mode is async

#### Primary Server 
```
### Append this line 
vim /var/lib/pgsql/15/data/postgresql.conf

synchronous_commit = remote_apply
synchronous_standby_names = 'replica1'      ### replica1 is replica server cluster name 

:x  ### save file 

sudo systemctl restart postgresql-15.service
sudo systemctl status postgresql-15.service
```

#### Replica Server 
```
### Append this line 
vim /var/lib/pgsql/15/data/postgresql.conf

cluster_name = 'replica1'			### cluser name of replaca server
hot_standby = on
primary_conninfo = 'host=192.168.10.10 port=5432 user=replication'

:x  ## save file 

sudo systemctl restart postgresql-15.service
sudo systemctl status postgresql-15.service
```

#### Check sync mode from primary server 
```
psql
select usename, application_name, client_addr, state, sync_priority, sync_state from pg_stat_replication;
```

### Post installation
- Check wal file move to replica server
- change archive command in replica server change server ip to primary ip 
- If you change your server IP address maybe your replication is not working, reconfigure replication again.

```
select pg_switch_wal();         ### switch wal file

```
