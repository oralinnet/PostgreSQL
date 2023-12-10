### Postgresql Parameters 

- listen_addresses

```t
In PostgreSQL, the listen_addresses parameter is used to specify the network interfaces on which the
database server will listen for incoming connections. This parameter is defined in the **postgresql.conf** configuration file.
**Default Value:** By default, PostgreSQL is configured to listen on all available network interfaces,
 and the listen_addresses parameter is commented out in the configuration file.
**Setting Values:** You can set the listen_addresses parameter to a specific IP address or a 
comma-separated list of addresses to restrict PostgreSQL to listen only on those interfaces. If you want
 PostgreSQL to listen on all available interfaces, you can set it to '*'.
listen_addresses = '192.168.1.100'
listen_addresses = '*'
Remember to **restart the PostgreSQL service** after making changes to the postgresql.conf 
file for the modifications to take effect.
```