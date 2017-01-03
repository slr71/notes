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
root# systemctl stop postgresql-9.5.service
root# systemctl start postgresql-9.5.service
```

## Opened up the Postgres port in the `iptables` configuration settings.

This is just routine `iptables` configuration.
