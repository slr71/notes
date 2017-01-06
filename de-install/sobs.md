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

## Install ElasticSearch.

There appears to be no playbook available for this at the moment.

### Install Java

```
root# yum install -y java-1.8.0-openjdk-devel
```

### Install ElasticSearch RPMs

This requires us to import ElasticSearch's GPG key then define the repository manually.

```
root# rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```

I put the repository definition in `/etc/yum.repos.d/elasticsearch.repo`.

```
[elasticsearch-5.x]
name=Elasticsearch repository for 5.x packages
baseurl=https://artifacts.elastic.co/packages/5.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
```

After creating the repository definition file, installing ElasticSearch itself was easy:

```
root# yum install -y elasticsearch
```

### Configuration Changes

Contents of `/etc/elasticsearch/elasticsearch.yml`:

``` yaml
# ======================== Elasticsearch Configuration =========================
#
# NOTE: Elasticsearch comes with reasonable defaults for most settings.
#       Before you set out to tweak and tune the configuration, make sure you
#       understand what are you trying to accomplish and the consequences.
#
# The primary way of configuring a node is via this file. This template lists
# the most important settings you may want to configure for a production cluster.
#
# Please see the documentation for further information on configuration options:
# <http://www.elastic.co/guide/en/elasticsearch/reference/current/setup-configuration.html>
#
# ---------------------------------- Cluster -----------------------------------
#
# Use a descriptive name for your cluster:
#
cluster.name: search-sobs
#
# ------------------------------------ Node ------------------------------------
#
# Use a descriptive name for the node:
#
node.name: search-sobs-1
#
# Add custom attributes to the node:
#
#node.attr.rack: r1
#
# ----------------------------------- Paths ------------------------------------
#
# Path to directory where to store the data (separate multiple locations by comma):
#
path.data: /var/data/elasticsearch
#
# Path to log files:
#
path.logs: /var/log/elasticsearch
#
# ----------------------------------- Memory -----------------------------------
#
# Lock the memory on startup:
#
bootstrap.memory_lock: true
#
# Make sure that the heap size is set to about half the memory available
# on the system and that the owner of the process is allowed to use this
# limit.
#
# Elasticsearch performs poorly when the system is swapping the memory.
#
# ---------------------------------- Network -----------------------------------
#
# Set the bind address to a specific IP (IPv4 or IPv6):
#
network.host: 0.0.0.0
#
# Set a custom port for HTTP:
#
http.port: 31338
#
# For more information, see the documentation at:
# <http://www.elastic.co/guide/en/elasticsearch/reference/current/modules-network.html>
#
# --------------------------------- Discovery ----------------------------------
#
# Pass an initial list of hosts to perform discovery when new node is started:
# The default list of hosts is ["127.0.0.1", "[::1]"]
#
#discovery.zen.ping.unicast.hosts: ["host1", "host2"]
#
# Prevent the "split brain" by configuring the majority of nodes (total number of nodes / 2 + 1):
#
#discovery.zen.minimum_master_nodes: 3
#
# For more information, see the documentation at:
# <http://www.elastic.co/guide/en/elasticsearch/reference/current/modules-discovery.html>
#
# ---------------------------------- Gateway -----------------------------------
#
# Block initial recovery after a full cluster restart until N nodes are started:
#
#gateway.recover_after_nodes: 3
#
# For more information, see the documentation at:
# <http://www.elastic.co/guide/en/elasticsearch/reference/current/modules-gateway.html>
#
# ---------------------------------- Various -----------------------------------
#
# Require explicit names when deleting indices:
#
#action.destructive_requires_name: true
```

Contents of `/etc/elasticsearch/jvm.options`:

```
## JVM configuration

################################################################
## IMPORTANT: JVM heap size
################################################################
##
## You should always set the min and max JVM heap
## size to the same value. For example, to set
## the heap to 4 GB, set:
##
## -Xms4g
## -Xmx4g
##
## See https://www.elastic.co/guide/en/elasticsearch/reference/current/heap-size.html
## for more information
##
################################################################

# Xms represents the initial size of total heap space
# Xmx represents the maximum size of total heap space

-Xms26g
-Xmx26g

################################################################
## Expert settings
################################################################
##
## All settings below this section are considered
## expert settings. Don't tamper with them unless
## you understand what you are doing
##
################################################################

## GC configuration
-XX:+UseConcMarkSweepGC
-XX:CMSInitiatingOccupancyFraction=75
-XX:+UseCMSInitiatingOccupancyOnly

## optimizations

# disable calls to System#gc
-XX:+DisableExplicitGC

# pre-touch memory pages used by the JVM during initialization
-XX:+AlwaysPreTouch

## basic

# force the server VM (remove on 32-bit client JVMs)
-server

# explicitly set the stack size (reduce to 320k on 32-bit client JVMs)
-Xss1m

# set to headless, just in case
-Djava.awt.headless=true

# ensure UTF-8 encoding by default (e.g. filenames)
-Dfile.encoding=UTF-8

# use our provided JNA always versus the system one
-Djna.nosys=true

# use old-style file permissions on JDK9
-Djdk.io.permissionsUseCanonicalPath=true

# flags to keep Netty from being unsafe
-Dio.netty.noUnsafe=true
-Dio.netty.noKeySetOptimization=true

# log4j 2
-Dlog4j.shutdownHookEnabled=false
-Dlog4j2.disable.jmx=true
-Dlog4j.skipJansi=true

## heap dumps

# generate a heap dump when an allocation from the Java heap fails
# heap dumps are created in the working directory of the JVM
-XX:+HeapDumpOnOutOfMemoryError

# specify an alternative path for heap dumps
# ensure the directory exists and has sufficient space
#-XX:HeapDumpPath=${heap.dump.path}

## GC logging

#-XX:+PrintGCDetails
#-XX:+PrintGCTimeStamps
#-XX:+PrintGCDateStamps
#-XX:+PrintClassHistogram
#-XX:+PrintTenuringDistribution
#-XX:+PrintGCApplicationStoppedTime

# log GC status to a file with time stamps
# ensure the directory exists
#-Xloggc:${loggc}

# Elasticsearch 5.0.0 will throw an exception on unquoted field names in JSON.
# If documents were already indexed with unquoted fields in a previous version
# of Elasticsearch, some operations may throw errors.
#
# WARNING: This option will be removed in Elasticsearch 6.0.0 and is provided
# only for migration purposes.
#-Delasticsearch.json.allow_unquoted_field_names=true
```

These changes required a restart:

```
root# systemctl restart elasticsearch
```

### Allow connections to ElasticSearch ports.

Updated the firewall to allow connections to ElasticSearch ports and restarted the firewall.

## Install Docker

I did this on my host using Ansible ad-hoc commands. I had to downgrade from Ansible version 2.2.0.0 to version 2.1.1.0
in order for this to work.

### Install the YUM version lock plugin.

```
myself$ ansible -i inventories/sobs docker-ready -u root -a "yum install -y yum-plugin-versionlock"

```

### Install the Docker YUM repository.

The easiest way to do that was to create the file locally then copy it over to the remote host. Here are the contents of
the local file (`docker.repo`):

```
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/7/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
```

The command to copy the file to each remote host was relatively simple:

```
myself$ ansible -i inventories/sobs docker-ready -u root -m copy -a "src=docker.repo dest=/etc/yum.repos.d/docker.repo"
```

### Update the YUM cache.

```
myself$ ansible -i inventories/sobs docker-ready -u root -a "yum makecache"
```

### Install `docker-engine`.

```
myself$ ansible -i inventories/sobs docker-ready -u root -a "yum install -y docker-engine-1.11.2"
```

### Lock the `docker-engine` package version.

```
myself$ ansible -i inventories/sobs docker-ready -u root -a "yum install -y docker-engine-1.11.2"
```

### Enable and start the Docker service.

```
myself$ ansible -i inventories/sobs docker-ready -u root -a "systemctl enable docker.service"
myself$ ansible -i inventories/sobs docker-ready -u root -a "systemctl start docker.service"
```

### Verify that Docker is running.

```
myself$ ansible -i inventories/sobs docker-ready -u root -a "docker run --rm hello-world"
```

## Install `docker-compose`.

We had a playbook that did this plus a few more things. I simply copied this playbook to a new file name, `blargh.yaml`,
and deleted the stuff that I didn't want. The result was this:

``` yaml
---
############################################
# Condor
############################################
- include: playbooks/infra-condor-exec-jar.yaml

############################################
# Docker Compose
############################################
- include: playbooks/infra-docker-compose.yml
```

Once the playbook was in place, I simply ran it, skipping the tags to install `docker-compose.yml` and `de-dc` because
I'm not quite ready for those yet:

```
myself$ ansible-playbook -i inventories/sobs -K blargh.yaml --skip-tags update-docker-compose-file,update-de-dc
```

Just for good measure, I ran another command to verify that docker-compose was installed:

```
myself$ ansible -i inventories/sobs docker-ready -u root -a "docker-compose --version"
```

## Install the private Docker registry.

When I was first considering how to do this, I wasn't entirely sure how I was going to manage the SSL keys. Andy said
that the keys are currently being generated for free, but they have to be renewed every month. This works, but having to
coordinate the SSL key updates on multiple hosts sounds a little painful. What I think I might do is use Apache HTTPD to
manage the SSL connections instead simply because that's how it's currently set up. The only problem that I can foresee
is that our Ansible scripts currently assume that nginx will be used instead. I don't know how painful it will be to
skip the nginx installation and use HTTPD instead.

Because of the slightly different HTTP reverse proxy setup in the SOBS deployment, the Docker registry will run on
`sobs-storage`, but all requests to the docker registry will go through `sobs-de`. This shouldn't pose a security risk
since the entire deployment has to be locked down. It will be something to keep in mind when we're troubleshooting
problems, however.
