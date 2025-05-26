# Setting up Fail2ban with PostgreSQL

## PostgreSQL Configuration

By default, PostgreSQL doesn't include the client IP address in the log output. We need this to be able to use fail2ban. Modify the `postgresql.conf` to include two directives:

```conf
log_connections = on
log_line_prefix = '%m {%h} [%p] %q%u@%d '
```

## Fail2ban Configuration

### 1. Create PostgreSQL Filter

Create or edit the PostgreSQL filter configuration:

```bash
sudo vim /etc/fail2ban/filter.d/postgresql.conf
```

Add the following content:

```conf
[Definition]
failregex = \{<HOST>\} .+? FATAL:  password authentication failed for user .+$
            \{<HOST>\} .+? FATAL:  no pg_hba.conf entry for host .+$
ignoreregex =
```

### 2. Configure Jail Settings

Create or edit the jail configuration:

```bash
sudo vim /etc/fail2ban/jail.d/postgresql.conf
```

Add the following content:

```conf
[postgresql]
enabled = true
filter = postgresql
logpath = /data/log/edb*.log
maxretry = 3
bantime = 86400
port = 5432
```