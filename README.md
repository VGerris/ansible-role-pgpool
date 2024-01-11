PgPool
=========

[![Build Status](https://travis-ci.com/fidanf/ansible-role-pgpool.svg?branch=master)](https://travis-ci.com/fidanf/ansible-role-pgpool)

Installs and configures PgPool-II for Debian/Ubuntu. The default _running mode_ is **streaming replication mode**.

Tested with :
  - Ubuntu 22.04.x :heavy_check_mark:

> :warning: Only the major 4.3 is currently being handled and tested by this role.  
Note on versioning ( from old repo regarding 4.1 )  
For example, to install the version [4.1.4](https://www.pgpool.net/docs/42/en/html/release-4-1-4.html), Debian 11 hosts should set `pgpool_version_debian: 4.1.4-6.pgdg110+1`, whereas Ubuntu 20.04 hosts should rather use `pgpool_version_debian: 4.1.4-6.pgdg20.04+1`.

---

- [PgPool](#pgpool)
  - [Requirements](#requirements)
  - [Role Variables](#role-variables)
  - [Dependencies](#dependencies)
  - [Example Playbook](#example-playbook)
  - [Configuration checking commands](#configuration-checking-commands)
  - [PCP commands](#pcp-commands)
  - [Handling server reboots](#handling-server-reboots)
  - [License](#license)

Requirements
------------

- Python >=3.8
- Ansible-core >=2.12

See [./requirements.txt](./requirements.txt) for detailled dependencies used to develop the role.

Role Variables
--------------

Check out the [defaults.yml](./defaults/main.yml) file to retrieve the extended list of this role's variables.

Dependencies
------------

None

Example Playbook
----------------

Check out both [inventory.yml](./inventory.yml) and [example.yml](./example.yml) to get a picture of how this role should be used to integrate with an exisiting postgreSQL cluster managed by repmgr.

If you're looking for a role solely dedicated to provide such an environment, take a look at [ansible-role-postgresql-ha](https://github.com/fidanf/ansible-role-postgresql-ha) 

Configuration checking commands
------------------------------

:warning: PgPool instance must be running

Show all configuration parameters
```bash
pgpool@pgpool01:~$ psql -h 192.168.56.30 -p 9999 -U admin -d testdb -c 'PGPOOL SHOW ALL' 
```
Or from Ansible:
```bash
ansible pgpool01 -b --become-user postgres -m shell -a "psql -h 192.168.56.30 -p 9999 -U admin -d testdb -c 'PGPOOL SHOW ALL' " -i invpgpool.yml 
```

Show pool status
```bash
pgpool@pgpool01:~$ psql -h 192.168.56.30 -p 9999 -U admin -d testdb -c 'SHOW POOL_NODES'
 node_id | hostname | port | status | lb_weight |  role   | select_cnt | load_balance_node | replication_delay | replication_state | replication_sync_state | last_status_change
---------+----------+------+--------+-----------+---------+------------+-------------------+-------------------+-------------------+------------------------+---------------------
 0       | pgsql01  | 5432 | up     | 0.333333  | primary | 30         | true              | 0                 |                   |                        | 2020-07-27 14:50:55
 1       | pgsql02  | 5432 | up     | 0.333333  | standby | 13         | false             | 0                 | streaming         | async                  | 2020-07-27 15:26:15
 2       | pgsql03  | 5432 | up     | 0.333333  | standby | 83         | false             | 0                 | streaming         | async                  | 2020-07-27 14:50:55
(3 rows)
```

Feel free to create a .pgpass file for pgpool user in order to skip repetitive password prompts

More details at : https://www.pgpool.net/docs/latest/en/html/sql-commands.html 

:warning: If for any reason PgPool's internal status is no longer in sync with Postgresql Replication' status, issue the following commands :
```bash
pgpool@pgpool01:~$ sudo systemctl stop pgpool2 && rm -f /var/log/pgpool/pgpool_status && sudo systemctl restart pgpool2
```

PCP commands
------------

Get pgpool status of backend node ID 0 
```bash
pgpool@pgpool01:~$ pcp_node_info -h 127.0.0.1 -U pgpool -w -v 0
Password:
Hostname               : pgsql01
Port                   : 5432
Status                 : 2
Weight                 : 0.333333
Status Name            : up
Role                   : primary
Replication Delay      : 0
Replication State      :
Replication Sync State :
Last Status Change     : 2020-07-27 14:50:55
```

Same thing against standby node
```bash
pgpool@pgpool01:~$ pcp_node_info -h 127.0.0.1 -U pgpool -w -v 1
Password:
Hostname               : pgsql02
Port                   : 5432
Status                 : 2
Weight                 : 0.333333
Status Name            : up
Role                   : standby
Replication Delay      : 0
Replication State      : streaming
Replication Sync State : async
Last Status Change     : 2020-07-27 14:50:55
```

Attach pgpool node and give it and ID
```bash
pgpool@pgpool01:~$ pcp_attach_node -h 127.0.0.1 -U pgpool -w -n 3
```

Display the parameter values as defined in pgpool.conf
```bash
pgpool@pgpool01:~$ pcp_pool_status -h /var/run/pcp -U pgpool -w
```

Display PgPool's cluster status
```bash
pcp_watchdog_info -h 127.0.0.1 -U pgpool -w -v
```

Handling server reboots
-----------------------

Although using the recommended paths from the official documentation, socket directories seems to have unsufficient privileges to be able to recreate PIDs upon server reboots.

If you're running PgPool and PostgreSQL on the same servers, you may use the following configuration instead which solves the problem :

```yaml
pgpool_pid_file_name: /var/run/postgresql/pgpool.pid
pgpool_socket_dir: /var/run/postgresql
pgpool_pcp_socket_dir: /var/run/postgresql
pgpool_wd_ipc_socket_dir: /var/run/postgresql
```
The included inventory.yml file has this setup.

More details at : https://www.pgpool.net/docs/latest/en/html/pcp-commands.html 

Security setup
--------------

Pgpool uses the pg_hba.conf file for access, details are here:
https://www.pgpool.net/docs/latest/en/html/auth-pool-hba-conf.html

The admin user is used as the 'superadmin' user. The pgpass file is used to set the password.
In the playbook one sets these as variables in the inventory:
 - pgpool_passwd_users_md5 - contains entries for the pgpass file
 - pgpool_pool_hba_entries - contains entries for pool_hba.conf

 By default the admin user is setup with password set to secret if not set otherwise - **change it for production**.

 The admin entry is setup ass hostsll so an SSL connection is required to connect.
 In production one can use IP white listing to the pgpool IP to further restrict this.
 This only applies to port 9999, the default port for pgpool.

 **Note**: between hosts in the network where pgpool/postgresql runs, the pg_hba rights do not apply, the postgres security rules do, so once one has access to a node in the internal network, like the bootstrap node, one can use for example repmgr and admin as defined in the playbook ( that this playbook can make use of ).
 This should be further restricted when required.

 For example, connecting to the external IP/DNS as follows from a range that is allowed will give:
 ```bash
 $ PGSSLMODE=verify-full psql -h yourpublicpgpoolip.yourexternal.domain  -p 9999 -U admin -d testdb -c 'SELECT * from pg_catalog.pg_stat_ssl'
Password for user admin: 
   pid   | ssl | version |         cipher         | bits | client_dn | client_serial | issuer_dn 
---------+-----+---------+------------------------+------+-----------+---------------+-----------
   10548 | t   | TLSv1.3 | TLS_AES_256_GCM_SHA384 |  256 |           |               | 
   10549 | t   | TLSv1.3 | TLS_AES_256_GCM_SHA384 |  256 |           |               | 
   11112 | t   | TLSv1.3 | TLS_AES_256_GCM_SHA384 |  256 |           |               | 
   11115 | t   | TLSv1.3 | TLS_AES_256_GCM_SHA384 |  256 |           |               | 
   11117 | t   | TLSv1.3 | TLS_AES_256_GCM_SHA384 |  256 |           |               | 
   11176 | t   | TLSv1.3 | TLS_AES_256_GCM_SHA384 |  256 |           |               | 
 1412856 | t   | TLSv1.3 | TLS_AES_256_GCM_SHA384 |  256 |           |               | 
 1474215 | t   | TLSv1.3 | TLS_AES_256_GCM_SHA384 |  256 |           |               | 
(8 rows)
```
but the following happens when using user repmgr:
```bash
$ PGSSLMODE=verify-full psql -h yourpublicpgpoolip.yourexternal.domain -p 9999 -U repmgr -d repmgr -c 'SELECT * from pg_catalog.pg_stat_ssl'
psql: error: connection to server at "yourpublicpgpoolip.yourexternal.domain" (a.b.c.d), port 9999 failed: FATAL:  client authentication failed
DETAIL:  no pool_hba.conf entry for host "w.x.y.z", user "repmgr", database "repmgr", SSL on
HINT:  see pgpool log for details
```

From the a node within the network the same query can look like this, assuming the used IP address is the internal one for yourpublicpgpoolip.yourexternal.domain:
```bash
$ psql -h 192.168.56.30  -p 5432 -U admin -d repmgr -c 'SELECT * from pg_catalog.pg_stat_ssl'
Password for user admin: 
   pid   | ssl | version |         cipher         | bits | client_dn | client_serial | issuer_dn 
---------+-----+---------+------------------------+------+-----------+---------------+-----------
   10915 | t   | TLSv1.3 | TLS_AES_256_GCM_SHA384 |  256 |           |               | 
 3777410 | t   | TLSv1.3 | TLS_AES_256_GCM_SHA384 |  256 |           |               | 
   10983 | t   | TLSv1.3 | TLS_AES_256_GCM_SHA384 |  256 |           |               | 
 3818533 | t   | TLSv1.3 | TLS_AES_256_GCM_SHA384 |  256 |           |               | 
(4 rows)

psql -h 192.168.56.30  -p 5432 -U repmgr -d repmgr -c 'SELECT * from pg_catalog.pg_stat_ssl'
Password for user repmgr: 
   pid   | ssl | version |         cipher         | bits | client_dn | client_serial | issuer_dn 
---------+-----+---------+------------------------+------+-----------+---------------+-----------
   10915 | t   | TLSv1.3 | TLS_AES_256_GCM_SHA384 |  256 |           |               | 
 3777410 | t   | TLSv1.3 | TLS_AES_256_GCM_SHA384 |  256 |           |               | 
   10983 | t   | TLSv1.3 | TLS_AES_256_GCM_SHA384 |  256 |           |               | 
 3818410 | t   | TLSv1.3 | TLS_AES_256_GCM_SHA384 |  256 |           |               | 
(4 rows)
```
Make sure to verify only the required ports and users are enabled as required by security policy.

License
-------

MIT / BSD
