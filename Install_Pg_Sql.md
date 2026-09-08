### Install Postgresql 15 on RHEL 8 

### Prerequisite
-  Internet Access
-  OS user with sudo privileges
- Disable Selinux

### Configure REPO access 
- Go to https://www.postgresql.org/download 
- Select your Linux OS version and PostgreSQL version 
```sh 
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo dnf -qy module disable postgresql
sudo dnf install -y postgresql15-server
sudo /usr/pgsql-15/bin/postgresql-15-setup initdb
sudo systemctl enable postgresql-15
sudo systemctl start postgresql-15
```
### Testing Database 
```
sudo su - postgres
psql
select version();
```
### Update PostgreSQL PATH in postgres user
```
sudo su - postgres
vim .bash_profile
export PATH=$PATH:/usr/pgsql-15/bin/
### Now reload bash_profile or logout and login postgres user
```

### Password base authentication
- login with postgres user in OS 
- First set or create passowd for database user
```
psql
ALTER USER postgres WITH PASSWORD 'postgres';
\q
```
- Enable Password base authentication in PostgreSQL
- replace peer or ident to scram-sha-256
```
cd $PGDATA
vim pg_hba.conf
local   all             all       scram-sha-256
:x  ## save file 
pg_ctl reload -D $PGDATA    ## Reload DB
psql    ### Now Need to password 
```

### Connect DB from Client 
- Open DB port firewall 
- listen_addresses Configure 

```
cd $PGDATA
vim postgresql.conf
listen_addresses = '*'  ### uncommit this line and change localhost to *
:x  ### save file 
vim $PGDATA/pg_hba.conf
host  all     all     192.168.5.241/32         scram-sha-256        ### cline ip address 
:x  ## savefile 
sudo systemctl restart postgresql-15.service    ### Restart service 


sudo firewall-cmd --permanent --add-port=5432/tcp   
sudo firewall-cmd --reload 
```
### Configure Database in Another Directory
- First create directory for DATA
- Give proper permission in postgres OS user

```
sudo mkdir /mydb/ -p
sudo chown -R postgres:postgres /mydb
chmod -R 775 /mydb
sudo su - postgres 
initdb -D /mydb -w                          ### install DB
pg_ctl -D /mydb -l /mydb/startlog start     ### start DB
pg_ctl -D /mydb/ status                     ### DB status
pg_ctl -D /mydb start                       ### Start DB
pg_ctl -D /mydb stop                        ### Stop DB
```
### post installation task
- Enable postgresql service for auto start after reboot
- Open DB port in firewall
- Accept Connection from out site in DB
- Make sure isten_addresses line is uncommented and set it's value to isten_addresses = '*' 
```
sudo systemctl enable postgresql-15
sudo firewall-cmd --permanent --add-port=5432/tcp
sudo firewall-cmd --reload 
sudo su - postgres 
vim $PGDATA/pg_hba.conf
host  all     all     192.168.5.241/32         scram-sha-256        ### cline ip address 
sudo systemctl restart postgresql-15.service
```


