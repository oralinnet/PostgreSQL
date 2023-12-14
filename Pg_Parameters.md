## PostgreSQL Parameters 

### `listen_addresses` Parameter

In PostgreSQL, the `listen_addresses` parameter is used to specify the network interfaces on which the
database server will listen for incoming connections. This parameter is defined in the postgresql.conf configuration file.

- **Default Value:** By default, PostgreSQL is configured to listen on all available network interfaces,
and the listen_addresses parameter is commented out in the configuration file.

- **Setting Values:** You can set the listen_addresses parameter to a specific IP address or a 
comma-separated list of addresses to restrict PostgreSQL to listen only on those interfaces. If you want
PostgreSQL to listen on all available interfaces, you can set it to '*'.

```plaintext
listen_addresses = '192.168.1.100'
listen_addresses = '*'
```
Remember to `restart the PostgreSQL service` after making changes to the postgresql.conf 
file for the modifications to take effect.


### `password_encryption` Parameter

The `password_encryption` parameter in PostgreSQL is used to specify the algorithm used for encrypting passwords stored in the database. It is an essential configuration option for enhancing security.

- **Default Value:** By default, PostgreSQL uses the MD5 algorithm to encrypt passwords.

- **Setting Values:** You can set the `password_encryption` parameter in the `postgresql.conf` configuration file. For example:

```plaintext
  password_encryption = 'scram-sha-256'
```

### `Port` Parameter

To specify which port PostgreSQL should use, you need to configure the PostgreSQL server settings. By default, PostgreSQL uses port 5432. However, you can change this port if needed. Here's how you can do it:

- **Edit PostgreSQL Configuration File:**
Open the PostgreSQL configuration file in a text editor. The location of this file may vary depending on your operating system. Common locations include /etc/postgresql/{version}/main/postgresql.conf on Linux or C:\Program Files\PostgreSQL\{version}\data\postgresql.conf on Windows.

- **Locate the Port Configuration:**
Look for the line that begins with port = in the configuration file. If it's commented out (with a # at the beginning), remove the # and change the port number if needed. For example:

```sh
port = 5432
# Change the number to the desired port, such as:
port = 5433
```
- **Save and Close the Configuration File Restart PostgreSQL**

### `huge_pages` Parameter 

Huge pages, also known as large pages, are a feature in PostgreSQL that can be configured to improve performance by reducing the overhead of managing page tables. This is achieved by using larger page sizes in the system's memory management. Enabling huge pages can be beneficial for databases that handle large amounts of data and have significant memory requirements. Here's how you can configure huge pages in PostgreSQL:

- **Configure the Operating System:** 
Enable huge pages at the operating system level. The steps to do this depend on your operating system. On Linux, you typically need to modify the kernel parameters. This might involve configuring the vm.nr_hugepages parameter in the /etc/sysctl.conf file or using the sysctl command. For example:

```sh
echo "vm.nr_hugepages = 2048" >> /etc/sysctl.conf
sysctl -p
# This example sets the number of huge pages to 2048. 
# Adjust this value based on your system's requirements
```
- **Configure PostgreSQL:**
In the PostgreSQL configuration file (postgresql.conf), specify the huge_pages parameter. This parameter is a boolean that enables or disables the use of huge pages. Set it to on to enable huge pages:

```sh
huge_pages = on

# Additionally, you can set the huge_page_size parameter to specify the size of the huge pages. 
# The default is typically 2MB. For example
huge_page_size = 2MB
```
- **Save the changes to the configuration file and Restart PostgreSQL**

