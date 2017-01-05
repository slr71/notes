# Database

## Install the Postgres YUM repository.

```
root# yum install https://download.postgresql.org/pub/repos/yum/9.5/redhat/rhel-7-x86_64/pgdg-centos95-9.5-3.noarch.rpm
```

## Install the Postgres server.

```
root# yum install postgresql95-server postgresql95-contrib
root# systemctl enable postgresql-9.5.service
postgres$ /usr/pgsql-9.5/bin/initdb -D /var/lib/pgsql/9.5/data/
root# systemctl start postgresql-9.5.service
```

## Set the passwords for the database users.

```
postgres$ psql -c "ALTER USER postgres WITH PASSWORD 'notreal'"
postgres$ psql -c "CREATE USER de"
postgres$ psql -c "ALTER USER de WITH PASSWORD 'reallyfake'"
```

## Created the databases.

```
postgres$ psql -c "CREATE DATABASE de WITH OWNER de"
postgres$ psql -c "CREATE DATABASE metadata WITH OWNER de"
postgres$ psql -c "CREATE DATABASE notifications WITH OWNER de"
postgres$ psql -c "CREATE DATABASE permissions WITH OWNER de"
postgres$ psql -d de -c "ALTER SCHEMA public OWNER TO de"
postgres$ psql -d metadata -c "ALTER SCHEMA public OWNER TO de"
postgres$ psql -d notifications -c "ALTER SCHEMA public OWNER TO de"
postgres$ psql -d permissions -c "ALTER SCHEMA public OWNER TO de"
```

## Updated `pg_hba.conf` and `postgresql.conf`.

The changes to `pg_hba.conf` were to allow connections from all hosts on the local subnet. The change to
`postgresql.conf` was to listen on all interfaces.

These changes required a restart:

```
root# systemctl restart postgresql-9.5.service
```

## Opened up the Postgres port in the `iptables` configuration settings.

This is just routine `iptables` configuration.

# Condor

Condor was already installed on the central manager and the execute nodes, but it wasn't installed on the submit node.

```
root# yum install condor-8.4.9-1
```

After Condor was installed, I copied the local configuration file from the central manager and modified it for the
submit node. The only difference between the submit node and the master node was the list of daemons to start.

# iRODS

## Install the Postgres YUM repository.

```
root# yum install https://download.postgresql.org/pub/repos/yum/9.3/redhat/rhel-7-x86_64/pgdg-centos93-9.3-3.noarch.rpm
```

## Install Postgres on the iRODS server.

```
root# yum installpostgresql93-server postgresql93-contrib
root# systemctl enable postgresql-9.3.service
postgres$ /usr/pgsql-9.3/bin/initdb -D /var/lib/pgsql/9.3/data/
root# systemctl start postgresql-9.3.service
```

## Set the passwords for the database users.

```
postgres$ psql -c "ALTER USER postgres WITH PASSWORD 'notreal'"
postgres$ psql -c "CREATE USER icat WITH PASSWORD 'reallyfake'"
postgres$ psql -c "CREATE USER icat_reader WITH PASSWORD 'stillfake'"
```

## Created the ICAT database.

```
postgres$ psql -c "CREATE DATABASE icat WITH OWNER icat"
postgres$ psql -d icat -c "ALTER SCHEMA public OWNER TO icat"
postgres$ psql -c "GRANT CONNECT ON DATABASE icat TO icat_reader"
postgres$ psql -c "GRANT TEMP ON DATABASE icat TO icat_reader"
postgres$ psql -c "REVOKE ALL PRIVILEGES ON DATABASE icat FROM PUBLIC"
postgres$ psql -c "GRANT ALL PRIVILEGES ON DATABASE icat TO postgres"
postgres$ psql -d icat -c "GRANT USAGE ON SCHEMA public TO icat_reader"
postgres$ psql -d icat -c "GRANT USAGE ON SCHEMA public TO icat_reader"
postgres$ psql -d icat -c "GRANT ALL PRIVILEGES ON SCHEMA public TO postgres"
postgres$ psql -d icat -c "REVOKE ALL PRIVILEGES ON SCHEMA public FROM PUBLIC"
```

## Updated `pg_hba.conf` and `postgresql.conf`.

The changes to `pg_hba.conf` were to allow connections from all hosts on the local subnet. The change to
`postgresql.conf` was to listen on all interfaces.

These changes required a restart:

```
root# systemctl restart postgresql-9.3.service
```

## Install iRODS ICAT server

```
root# yum install authd postgresql93-odbc unixODBC fuse-libs perl-JSON python-jsonschema python-psutil python-requests
root# rpm -i irods-database-plugin-postgres93-1.9-centos7-x86_64.rpm irods-icat-4.1.9-centos7-x86_64.rpm
```

The next step was to run an interactive setup script:

```
root# /var/lib/irods/packaging/setup_irods.sh
```

The program required me to specify a few keys to be used. These keys can be found in the iRODS configuration files, so
I'm not recording them anywhere else.

## Grant additional database access privileges.

The `icat_reader` account needs to have read access to the `icat` database.

```
postgres$ psql -d icat -c "GRANT SELECT ON ALL TABLES IN SCHEMA public TO icat_reader"
```

## Install RabbitMQ.

We have a playbook for this, but it's currently not working. I ended up doing most of the configuration for this
manually because it was easier to do that than to figure out how to configure RabbitMQ while at the same time updating
the playbook. Running the playbook was enough to get rabbitmq installed and to enable the management plugin. I did most
of the rest of the configuration manually.

### Update RabbitMQ configuration.

The path to this file is `/etc/rabbitmq/rabbitmq.config`:

```
%% -*- mode: erlang -*-
%% ----------------------------------------------------------------------------
%% RabbitMQ Sample Configuration File.
%%
%% See http://www.rabbitmq.com/configure.html for details.
%% ----------------------------------------------------------------------------
[
 {rabbit, [
   {tcp_listeners, [31333]}
  ]},

 {rabbitmq_management, [
   {listener, [{port, 31366}]}
 ]}
].
```

This change required the service to be restart:

```
root# systemctl restart rabbitmq-server
```

### Create the virual hosts.

```
root# rabbitmqctl add_vhost /sobs/de
root# rabbitmqctl add_vhost /sobs/data-store
root# rabbitmqctl set_permissions -p /sobs/de sobs_de ".*" ".*" ".*"
root# rabbitmqctl set_permissions -p /sobs/data-store sobs_de ".*" ".*" ".*"
```

### Install the `rabbitmqadmin` utility.

```
root# curl -o /sbin/rabbitmqadmin http://localhost:31366/cli/rabbitmqadmin
root# chmod +x /sbin/rabbitmqadmin
root# rabbitmqadmin --bash-completion > /etc/bash_completion.d/rabbitmqadmin
```

### Declare the exchanges.

The command used to connect to the management port was a bit of a pain to type repeatedly, so I created an alias for it:

```
root# alias rmq='rabbitmqadmin -P 31366 -u sobs_de -p "$PASSWORD"'
```

I set the environment variable in my session using the following command, which allows me to avoid having to repeatedly
type the password without the password showing up in the command history file:

```
root# read -s PASSWORD && export PASSWORD
```

Just to make sure that the command works correctly, I tried listing the exchanges with the alias that I set up:

```
root# rmq -V /sobs/de list exchanges
+----------+--------------------+---------+-------------+---------+----------+
|  vhost   |        name        |  type   | auto_delete | durable | internal |
+----------+--------------------+---------+-------------+---------+----------+
| /sobs/de |                    | direct  | False       | True    | False    |
| /sobs/de | amq.direct         | direct  | False       | True    | False    |
| /sobs/de | amq.fanout         | fanout  | False       | True    | False    |
| /sobs/de | amq.headers        | headers | False       | True    | False    |
| /sobs/de | amq.match          | headers | False       | True    | False    |
| /sobs/de | amq.rabbitmq.trace | topic   | False       | True    | True     |
| /sobs/de | amq.topic          | topic   | False       | True    | False    |
+----------+--------------------+---------+-------------+---------+----------+
```

I was then finally able to declare the exchanges:

```
root# rmq -V /sobs/de declare exchange name=de type=topic auto_delete=false durable=true internal=false
root# rmq -V /sobs/data-store declare exchange name=irods type=topic auto_delete=false durable=true internal=false
```

## Install iRODS rules.

An Ansible role was available for this, but it didn't quite suit the needs of the SOBS deployment because not all of the
settings that needed to be modified were configurable. Also, the default zone name was configurable, but the
configuration parameter wasn't used in all relevant places. I ended up using the role to generate the configuration,
editing the resulting files manually, and running the iRODS setup script once again for good measure.

### Created a playbook for deploying iRODS

``` yaml
# Note: this playbook was created for the SOBS deployment. It uses some additional iRODS settings that aren't
# currently included in the group variables for other deployments. Use with caution.
---
- hosts: irods
  become: true
  gather_facts: true
  vars:
    cfg: "{{ amqp.irods }}"
    amqp_uri: "amqp://{{ cfg.user }}:{{ cfg.password }}@{{ cfg.host }}:{{ cfg.port }}/{{ cfg.vhost_encoded }}"
  roles:
    - role: CyVerse-Ansible.cyverse-irods-cfg
      irods_amqp_uri: "{{ amqp_uri }}"
      irods_default_resource_name: "{{ irods.resource.name }}"
      irods_default_resource_directory: "{{ irods.resource.directory }}"
      irods_negotiation_key: "{{ irods.resource.negotiation_key }}"
      irods_server_control_plane_key: "{{ irods.resource.control_plane_key }}"
      irods_zone_key: "{{ irods.resource.zone_key }}"
      irods_db:
        host: "{{ icat.host }}"
        port: "{{ icat.port }}"
        username: "{{ icat.user }}"
        password: "{{ icat.password }}"
```

The custom iRODS settings mentioned in the header comments include the iRODS resource settings. These settings appear in
`group_vars/sobs`, but nowhere else.

### Installed the role on my workstation.

```
myself$ sudo ansible-galaxy install CyVerse-Ansible.cyverse-irods-cfg
```

### Ran the playbook on my workstation.

```
myself$ ansible-playbook -i inventories/sobs -K deployrods.yaml
```

### Fixed the configuration files.

- Replaced `iplant` with `sobs` wherever it was used to refer to a zone name.
- Corrected any incorrect values in the iRODS configuration files.
- Ran `/var/lib/irods/packaging/setup_irods.sh` again for good measure.
