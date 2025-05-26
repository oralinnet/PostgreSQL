# Changing Default PostgreSQL OS User in Linux

This guide explains how to change the default PostgreSQL OS user for PostgreSQL-15 on RHEL/CentOS/OEL systems.

## Prerequisites
- PostgreSQL-15 installed
- Root or sudo access
- System running RHEL/CentOS/OEL

## Steps

### 1. Create New User
```bash
adduser dbuser
passwd dbuser
```

### 2. Create Directory and Set Permissions
```bash
# Create data directory
mkdir /rnddatabase/pgsql/data -p

# Set ownership
chown -R dbuser:dbuser /rnddatabase/pgsql/data/
chown -R dbuser:dbuser /var/lib/pgsql/      # For renaming postgres user
```

### 3. Rename or Delete OS User
```bash
# Option 1: Rename postgres user
pkill -u postgres
usermod -l dbuser postgres

# Option 2: Delete postgres user (Use with caution!)
userdel -r postgres         # Warning: This will delete all files from home directory
```

### 4. Configure Service Override
```bash
# Create override directory and file
mkdir /etc/systemd/system/multi-user.target.wants/postgresql-15.service.d/
touch /etc/systemd/system/multi-user.target.wants/postgresql-15.service.d/overrideuser.conf

# Edit override file
vim /etc/systemd/system/multi-user.target.wants/postgresql-15.service.d/overrideuser.conf
```

Add the following content to the override file:
```ini
[Service]
User=dbuser
Group=dbuser
```

After saving the file, reboot your system:
```bash
reboot
```

### 5. Configure Socket Location Permissions
```bash
# Set ownership of socket directory
chown -R dbuser:dbuser /var/run/postgresql

# Edit tmpfiles.d configuration
vim /usr/lib/tmpfiles.d/postgresql-15.conf
```

Add this line if not present:
```ini
d /run/postgresql 0755 dbuser dbuser -
```

### 6. Reload and Start Service
```bash
# Enable and start PostgreSQL service
sudo systemctl enable postgresql-15.service
sudo systemctl daemon-reload
sudo systemctl restart postgresql-15.service

# Verify service status
sudo systemctl status postgresql-15.service -l
```

### 7. Create New Cluster
```bash
# Switch to dbuser
su - dbuser

# Connect to PostgreSQL
psql

# List databases
\l

# Create new cluster
initdb -D /rnddatabase/pgsql/data/

# Start cluster
pg_ctl -D /rnddatabase/pgsql/data/ -l logfile start
```

## Notes
- Make sure to backup your data before performing these changes
- The service location path may vary depending on your PostgreSQL version
- Always verify the service status after making changes
- Consider updating any application connection strings to reflect the new user
