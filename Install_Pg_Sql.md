### Install Postgresql 15 on RHEL 8 

### Prerequisite
-  Internet Access
-  OS user with sudo privileges
- Disable Selinux

### Congigure REPO access 
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
