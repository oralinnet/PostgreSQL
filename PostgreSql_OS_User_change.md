### Rename or Change Default postgres user in linux OS 

- In this tutorial we change default postgres OS user for PostgreSql-15 on RHEL/Centos/OEL
- Setp 1: Create New User 
```sh
adduser dbuser
passwd dbuser
```
- Step 2: Create Directory and give proper permission 
```sh
mkdir /rnddatabase/pgsql/data -p
chown -R dbuser:dbuser /rnddatabase/pgsql/data/
chown -R dbuser:dbuser /var/lib/pgsql/      # For Rename postgres user 
```
- Step 3: Rename or Delete OS User 
```sh
### For Rename 
pkill -u postgres
usermod -l dbuser postgres
### For delete 
userdel -r postgres         ## Becareful for this command, Delete all file from Home directory
```
- Step 4: Find postgres service and create overrideuser.conf
```sh
/etc/systemd/system/multi-user.target.wants/postgresql-15.service       ## Service location for PostgreSql 15
mkdir /etc/systemd/system/multi-user.target.wants/postgresql-15.service.d/
touch /etc/systemd/system/multi-user.target.wants/postgresql-15.service.d/overrideuser.conf
vim /etc/systemd/system/multi-user.target.wants/postgresql-15.service.d/overrideuser.conf              ## append service Part

[Service]
User=dbuser
Group=dbuser

:x  ## Save and exit 
reboot          ## Reboot your system
```

- Step 5: Change Permission and owership of socket location, Edit tmpfiles.d file
```sh
chown -R dbuser:dbuser /var/run/postgresql
vim /usr/lib/tmpfiles.d/postgresql-15.conf
    d /run/postgresql 0755 dbuser dbuser -          ## append this line if it's not found
:x      ## save and exit 
```

- Step 6: Reload server demon, Enable and start service

```sh
sudo systemctl enable postgresql-15.service
sudo systemctl daemon-reload
sudo systemctl restart postgresql-15.service
sudo systemctl status postgresql-15.service -l
```

- Step 7: Check your service and create new cluster 
```sh
su - dbuser
psql
\l
initdb -D /rnddatabase/pgsql/data/      ### Create cluster 
pg_ctl -D /rnddatabase/pgsql/data/ -l logfile start         ### start cluster 

```
