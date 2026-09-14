# PostgreSQL Parameters

## `listen_addresses` Parameter

The `listen_addresses` parameter specifies the network interfaces on which the database server will listen for incoming connections. This parameter is defined in the `postgresql.conf` configuration file.

### Default Value
By default, PostgreSQL is configured to listen on all available network interfaces, and the `listen_addresses` parameter is commented out in the configuration file.

### Setting Values
You can set the `listen_addresses` parameter to a specific IP address or a comma-separated list of addresses to restrict PostgreSQL to listen only on those interfaces. If you want PostgreSQL to listen on all available interfaces, you can set it to '*'.

```conf
listen_addresses = '192.168.1.100'
listen_addresses = '*'
```

> **Note:** Remember to restart the PostgreSQL service after making changes to the `postgresql.conf` file for the modifications to take effect.

## `password_encryption` Parameter

The `password_encryption` parameter specifies the algorithm used for encrypting passwords stored in the database. It is an essential configuration option for enhancing security.

### Default Value
By default, PostgreSQL uses the MD5 algorithm to encrypt passwords.

### Setting Values
You can set the `password_encryption` parameter in the `postgresql.conf` configuration file:

```conf
password_encryption = 'scram-sha-256'
```

## `Port` Parameter

The port parameter specifies which port PostgreSQL should use for incoming connections.

### Default Value
By default, PostgreSQL uses port 5432.

### Configuration Steps

1. **Edit PostgreSQL Configuration File:**
   - Open the PostgreSQL configuration file in a text editor
   - Common locations:
     - Linux: `/etc/postgresql/{version}/main/postgresql.conf`
     - Windows: `C:\Program Files\PostgreSQL\{version}\data\postgresql.conf`

2. **Locate the Port Configuration:**
   - Look for the line that begins with `port =` in the configuration file
   - If it's commented out (with a # at the beginning), remove the # and change the port number if needed

```conf
port = 5432
# Change the number to the desired port, such as:
port = 5433
```

3. **Save and Restart:**
   - Save the configuration file
   - Restart PostgreSQL for changes to take effect

## `huge_pages` Parameter

Huge pages (also known as large pages) are a feature in PostgreSQL that can improve performance by reducing the overhead of managing page tables. This is achieved by using larger page sizes in the system's memory management.

### Configuration Steps

1. **Configure the Operating System:**
   - Enable huge pages at the operating system level
   - On Linux, modify the kernel parameters in `/etc/sysctl.conf` or use the `sysctl` command:

```bash
echo "vm.nr_hugepages = 2048" >> /etc/sysctl.conf
sysctl -p
```

> **Note:** The example sets the number of huge pages to 2048. Adjust this value based on your system's requirements.

2. **Configure PostgreSQL:**
   - In the `postgresql.conf` file, set the `huge_pages` parameter:

```conf
huge_pages = on

# Optionally set the huge page size (default is typically 2MB)
huge_page_size = 2MB
```

3. **Save and Restart:**
   - Save the configuration file
   - Restart PostgreSQL for changes to take effect
