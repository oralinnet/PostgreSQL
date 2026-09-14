# PostgreSQL

PostgreSQL is a powerful, open-source object-relational database system that has earned a strong reputation for reliability, feature robustness, and performance. It is known for its advanced features, extensibility, and standards compliance.

## Repository Structure

- [Introduction](/Configuration/Introduction) - Basic concepts and getting started
- [Install PostgreSQL](/Configuration//Install_Pg_Sql.md) - Install PostgreSQL in RHEL 8
- [Replication PostgreSQL](/Configuration//Replica.md) - Configure Replica 
- [SQL](/SQL) - SQL commands and examples
- [DB script](/Configuration//DB_script.md) - Database scripts and utilities
- [Pg Parameters](/Configuration//Pg_Parameters.md) - PostgreSQL configuration parameters
- [PostgreSql OS User_change](/Configuration//PostgreSql_OS_User_change.md) - OS user management
- [fail2ban Configure](/Configuration//fail2ban.md) - Security and fail2ban setup
- [Run SQL command from another OS user](/Configuration//run_psql_other_user.md) - Running psql as different users
- [PostgreSQL DBA Cheat Sheet](/Configuration//postgresql-dba-cheat-sheet.md) - Essential Functions & Commands

## Features

- **ACID Compliance**: Ensures data integrity and reliability
- **Complex Queries**: Support for complex SQL queries and advanced data types
- **Multi-Version Concurrency Control (MVCC)**: Enables concurrent access to data
- **Point-in-Time Recovery**: Allows recovery to any point in time
- **Table Partitioning**: Efficient handling of large tables
- **JSON Support**: Native JSON and JSONB data types
- **Full-Text Search**: Advanced text search capabilities
- **Geographic Objects**: PostGIS extension for geographic data
- **Extensibility**: Custom functions, operators, and data types
- **Replication**: Built-in replication for high availability
- **Security**: Row-level security and encryption

## Installation

### Windows
1. Download the installer from [PostgreSQL Official Website](https://www.postgresql.org/download/windows/)
2. Run the installer and follow the setup wizard
3. Remember the password you set for the postgres user
4. The default port is 5432
5. Optional: Install pgAdmin (GUI tool) during installation

### Linux (Ubuntu/Debian)
```bash
sudo apt update
sudo apt install postgresql postgresql-contrib
```

### macOS
```bash
brew install postgresql
```

## Basic Usage

### Starting PostgreSQL Service

#### Windows
- PostgreSQL service starts automatically after installation
- To manage service: Services app → PostgreSQL
- Default data directory: `C:\Program Files\PostgreSQL\[version]\data`

#### Linux
```bash
sudo service postgresql start
# or
sudo systemctl start postgresql
```

#### macOS
```bash
brew services start postgresql
```

### Connecting to PostgreSQL

```bash
# Connect as postgres user
psql -U postgres

# Connect to specific database
psql -U postgres -d database_name

# Connect with host and port
psql -h localhost -p 5432 -U postgres
```

### Basic Commands
```sql
-- List all databases
\l

-- Connect to a database
\c database_name

-- List all tables
\dt

-- Describe a table
\d table_name

-- Exit psql
\q
```

## Configuration

### Important Configuration Files
- `postgresql.conf`: Main configuration file
- `pg_hba.conf`: Client authentication configuration
- `pg_ident.conf`: User name mapping

### Common Settings
- `max_connections`: Maximum number of concurrent connections
- `shared_buffers`: Memory allocated for caching
- `work_mem`: Memory for sorting and joins
- `maintenance_work_mem`: Memory for maintenance operations

## Security

For detailed security configuration and fail2ban setup, refer to:
- [fail2ban.md](/fail2ban.md) - Security setup guide
- [run_psql_other_user.md](/run_psql_other_user.md) - User management
- [PostgreSql_OS_User_change.md](/PostgreSql_OS_User_change.md) - OS user configuration

## Useful Resources

- [Official Documentation](https://www.postgresql.org/docs/)
- [PostgreSQL Tutorial](https://www.postgresqltutorial.com/)
- [Stack Overflow PostgreSQL Tag](https://stackoverflow.com/questions/tagged/postgresql)
- [PostgreSQL Wiki](https://wiki.postgresql.org/wiki/Main_Page)
- [PostgreSQL Exercises](https://pgexercises.com/)

## Contributing

Feel free to contribute to this repository by:
1. Forking the repository
2. Creating a new branch
3. Making your changes
4. Submitting a pull request

## License

This project is licensed under the MIT License - see the LICENSE file for details.
