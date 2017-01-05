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

### Create the virual hosts.

```
root# rabbitmqctl add_vhost /sobs/de
root# rabbitmqctl add_vhost /sobs/data-store
root# rabbitmqctl set_permissions -p /sobs/de sobs_de ".*" ".*" ".*"
root#rabbitmqctl set_permissions -p /sobs/data-store sobs_de ".*" ".*" ".*"
```
