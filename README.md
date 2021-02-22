paulrentschler.mysql
====================

[![MIT licensed][mit-badge]][mit-link]

Installs and configures the MySQL database server on Ubuntu Linux


Requirements
------------

None.


Role Variables
--------------

A password for the root user is not specified by default and if left undefined a random one will be generated. To specify one, define the variable (ideally using Ansible Vault):

```yaml
mysql_root_password: "{{ mysql_root_password_vaulted }}"
```

**NOTE:** this is not needed with MySQL 5.7+


The following variables are available with defaults defined in `defaults/main.yml`:

Specify users who should be setup to access the server as root:

    mysql_admin_users: []

**NOTE:** this is not needed with MySQL 5.7+


These variables generally do not (and probably should not) be changed as they are pretty standard for all MySQL installations:

```yaml
mysql_port: 3306
mysql_bind_address: "127.0.0.1"
mysql_datadir: /var/lib/mysql
mysql_logdir: /var/log/mysql

mysql_packages:
  - python-selinux
  - mysql-server
  - python-pymysql

mysql_service: mysql
mysql_file_conf: /etc/mysql/my.cnf
mysql_pid_file: /var/run/mysqld/mysqld.pid
mysql_socket: /var/run/mysqld/mysqld.sock
mysql_binary: /usr/sbin/mysqld
```


### Firewall settings

**NOTE:** If the host supports the Ubuntu FireWall (UFW), these settings can be used  to allow access to the MySQL server through the firewall.

Set to "yes" to indicate the firewall should be configured.

    mysql_firewall: no

Specify a list of Ansible hosts that should be allowed access.

    mysql_firewall_allow_hosts: []

Specify a list of Ansible groups that should be allowed access.

    mysql_firewall_allow_groups: []


### Replication settings

To enable replication, several variables must be defined for all the hosts that will be involved in the replication process.

The replication flag must be changed to "yes".

    mysql_replication: no

The MySQL user who will handle the replication work must be defined.

    mysql_repl_user: replicant
    mysql_repl_password: replicant


And information about the primary host must be defined.

The primary's Ansible inventory name is needed.

    mysql_repl_primary_inventory_name: ""

Along with the primary's IP address and the port MySQL is listening on.

    mysql_repl_primary_ip: 127.0.0.1
    mysql_repl_primary_port: 3306


#### Per-host replication settings

Set to "yes" for the host that will be the primary server (handles reading and writing).

    mysql_repl_primary: no

Set to "yes" for the host that will be the replica server (handles reading only).

    mysql_repl_replica: no

**NOTE**: a host cannot have both `mysql_repl_primary: yes` and `mysql_repl_replica: yes`!


#### Replication configuration options

These settings control how the data is replicated and generally do not need to be changed.

The following are used by the primary to create a log of all the changes made.

```yaml
mysql_repl_auto_increment: 1
mysql_repl_increment_offset: 1
mysql_binary_log: "{{ mysql_logdir }}/binary-log"
mysql_expire_logs_days: 5
```

The following are used by the replica to know where to get the log of changes made on the primary.

```yaml
mysql_relay_log: "{{ mysql_logdir }}/relay-log"
mysql_relay_log_space_limit: "2G"
```


#### Replication watchdog

The role can optionally install a separate [MySQL replication watchdog script](https://github.com/paulrentschler/mysqlwatch) to watch and alert when the replica gets out of sync with the primary.

To install the watchdog, specify:

```yaml
mysql_repl_watchdog: yes
```


Indicate what email address the alerts will go to.

```yaml
mysql_repl_watchdog_mailto: root
```


Indicate when the watchdog will be run. Defaults to 4:00pm, 7 days a week.

```yaml
mysql_repl_watchdog_hour: 16
mysql_repl_watchdog_min: 0
mysql_repl_watchdog_days: "*"
```


Role Tags
---------

Many of the tasks are tagged to enable certain features to be run without every task the role being run.

`mysql_security` runs only the tasks associated with removing anonymous users, root users (other than root@localhost), and the test database to security harden the server.

`mysql_replication_setup` runs all of the tasks associated with setting a host up to replicate from a primary to a replica. This is useful if replication is being enabled **after** MySQL was already installed without replication configured.

`mysql_repair_replication` runs the tasks necessary to copy the data from the primary to the replica and start/restart the replication process. This is useful if the replica experiences an error and stops replicating from the primary.


Dependencies
------------

This role requires modules from the *community.mysql* collection.

To install it use: `ansible-galaxy collection install community.mysql`


Example Playbook
----------------

Minimal example that installs a stand-alone MySQL server and creates a random password for the root user.

```yaml
---
- hosts: all
  roles:
     - paulrentschler.mysql
```

More common example that includes specifying the root password from Ansible Vault.

```yaml
---
- hosts: all
  roles:
    - role: paulrentschler.mysql
      vars:
        mysql_root_password: "{{ mysql_root_password_vaulted }}"
```

Complex example that sets up replication and installs the watchdog on the replica.

```yaml
---
- hosts: db_primary
  roles:
    - role: paulrentschler.mysql
      vars:
        mysql_root_password: "{{ mysql_root_password_vaulted }}"
        mysql_replication: yes
        mysql_repl_password: "{{ mysql_replicant_password_vaulted }}"
        mysql_repl_primary_inventory_name: "db_primary"
        mysql_repl_primary_ip: 192.168.0.10
        mysql_repl_primary: yes

- hosts: db_replica
  roles:
    - role: paulrentschler.mysql
      vars:
        mysql_root_password: "{{ mysql_root_password_vaulted }}"
        mysql_replication: yes
        mysql_repl_password: "{{ mysql_replicant_password_vaulted }}"
        mysql_repl_primary_inventory_name: "db_primary"
        mysql_repl_primary_ip: 192.168.0.10
        mysql_repl_replica: yes
        mysql_repl_watchdog: yes
```


License
-------

[MIT][mit-link]


Author Information
------------------

Created by Paul Rentschler in 2021.


[mit-badge]: https://img.shields.io/badge/license-MIT-blue.svg
[mit-link]: https://github.com/paulrentschler/ansible-role-mysql/blob/master/LICENSE
