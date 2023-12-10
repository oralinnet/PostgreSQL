### Postgresql Parameters 

- listen_addresses
```
In PostgreSQL, the listen_addresses parameter is used to specify the network interfaces on which the
database server will listen for incoming connections. This parameter is defined in the __postgresql.conf__ configuration file.
__Default Value:__ By default, PostgreSQL is configured to listen on all available network interfaces,
 and the listen_addresses parameter is commented out in the configuration file.
__Setting Values:__ You can set the listen_addresses parameter to a specific IP address or a 
comma-separated list of addresses to restrict PostgreSQL to listen only on those interfaces. If you want
 PostgreSQL to listen on all available interfaces, you can set it to '_'.
listen_addresses = '192.168.1.100'
listen_addresses = '*'
Remember to __restart the PostgreSQL service__ after making changes to the postgresql.conf 
file for the modifications to take effect.
```