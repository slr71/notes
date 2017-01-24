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
postgres$ psql -c "CREATE USER grouper"
postgres$ psql -c "ALTER USER grouper WITH PASSWORD 'totallyfake'"
```

## Created the databases.

```
postgres$ psql -c "CREATE DATABASE de WITH OWNER de"
postgres$ psql -c "CREATE DATABASE metadata WITH OWNER de"
postgres$ psql -c "CREATE DATABASE notifications WITH OWNER de"
postgres$ psql -c "CREATE DATABASE permissions WITH OWNER de"
postgres$ psql -c "CREATE DATABASE grouper WITH OWNER grouper"
postgres$ psql -d de -c "ALTER SCHEMA public OWNER TO de"
postgres$ psql -d metadata -c "ALTER SCHEMA public OWNER TO de"
postgres$ psql -d notifications -c "ALTER SCHEMA public OWNER TO de"
postgres$ psql -d permissions -c "ALTER SCHEMA public OWNER TO de"
postgres$ psql -d grouper -c "ALTER SCHEMA public OWNER TO grouper"
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

This change required the service to be restarted:

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
myself$ ansible -i inventories/sobs docker-ready -u root -a "yum versionlock docker-engine"
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
that the keys are currently being generated for free, but they have to be renewed every month. I thought it might have
been easier to let Apache HTTPD manage the SSL connections instead simply because that's how it's currently set up. I
actually configured the system this way initially, but I was unable to push images to the registryw hen it was
configured this way. The alternative that I ultimately chose was to put the private registry on the DE UI host directly
and use the same SSL keys that Apache uses. Once this is done, I can simply modify the script that refreshes the keys so
that restarts the Docker registry once the keys have been refreshed.

### Set up a redis service.

For this step, I simply copied and modified the service definition from our private Docker registry host. The contents
of the service definition file, `/usr/lib/systemd/system/sobs-registry-redis.service`, are:

```
[Unit]
Description=Redis for SOBS Docker Registry
BindsTo=docker.service
PartOf=docker.service
After=docker.service
Requisite=docker.service

[Service]
ExecStartPre=-/usr/bin/docker rm sobs_registry_redis
ExecStart=/usr/bin/docker run --name sobs_registry_redis \
-v /etc/localtime:/etc/localtime \
--log-driver=journald \
redis:3.0.7
ExecStop=-/usr/bin/docker stop sobs_registry_redis
Restart=on-failure

SyslogIdentifier=sobs_registry_redis
SyslogFacility=local6

[Install]
WantedBy=multi-user.target
```

Once the service definition file was created, I enabled and started the service:

```
root# systemctl enable sobs-registry-redis
root# systemctl start sobs-registry-redis
```

### Set up the docker registry.

First, we needed a configuration file. The path to the file is `/etc/docker/registry/config.yml`:

```
version: 0.1
log:
  level: info
  fields:
    service: sobs_registry
    environment: production
storage:
    filesystem:
        rootdirectory: /registry_data
    cache:
        blobdescriptor: redis
http:
    addr: :5000
    secret: asecretforlocaldevelopment
    debug:
        addr: localhost:5001
    tls:
      certificate: /usr/local/etc/letsencrypt/sobs-de.sobs.arizona.edu/chained.pem
      key: /usr/local/etc/letsencrypt/sobs-de.sobs.arizona.edu/domain.key
redis:
    addr: redis:6379
```

Next, we needed the service definition file, `/usr/lib/systemd/system/sobs-registry.service`:

```
[Unit]
Description=SOBS Docker Registry
BindsTo=docker.service sobs-registry-redis.service
PartOf=docker.service sobs-registry-redis.service
After=docker.service sobs-registry-redis.service
Requires=sobs-registry-redis.service
Requisite=docker.service

[Service]
ExecStartPre=-/usr/bin/docker rm -v sobs_registry
ExecStart=/usr/bin/docker run --name sobs_registry \
-p 5000:5000 \
-v /etc/localtime:/etc/localtime \
-v /usr/share/docker-registry:/registry_data \
-v /etc/docker/registry/config.yml:/etc/docker/registry/config.yml \
-v /usr/local/etc/letsencrypt/sobs-de.sobs.arizona.edu:/usr/local/etc/letsencrypt/sobs-de.sobs.arizona.edu \
--link sobs_registry_redis:redis \
--log-driver=journald \
registry:2.5
ExecReload=-/usr/bin/docker kill -s HUP sobs_registry
ExecStop=-/usr/bin/docker stop sobs_registry
Restart=on-failure

SyslogIdentifier=sobs_registry
SyslogFacility=local6

[Install]
WantedBy=multi-user.target
```

Note: the `ExecReload` command isn't currently being used. I was considering using that to reload the configuration
file, but I couldn't find any documentation indicating whether or not sending a `SIGHUP` to the registry daemon would
cause the daemon to reload its configuration.

Once this was done, I enabled and started the service as usual:

```
root# systemctl enable sobs-registry
root# systemctl start sobs-registry
```

The final step in the registry deployment was to ensure that it's restarted whenever the SSL certificate is renewed. I
added a line to the certificate renewal script to do this:

```
systemctl restart sobs-registry
```

### Allow connections from sobs-de to port 5000.

This required me to simply add a firewall rule and restart iptables.

## Create the data container for SOBS.

This step required a little bit of backtracking because I was unable to push the new image after creating it. I ended up
redeploying the Docker registry to solve that problem.

The first step here was to generate the GPG keys. There were some instructions available in one of our private
repositories. The instructions didn't contain any sensitive information, though, so I duplicated
them [here](gpg-key.md). I then needed to generate a key pair that can be used to sign and validate the JWTs that the DE
UI uses to authenticate to the Terrain services. I typically use
the [buddy-sign instructions](https://funcool.github.io/buddy-sign/latest/#generate-keypairs) for generating RSA key
pairs for this.

The results of these commands are stored in the `docker-sobs-data` repository in GitLab, along with a Dockerfile that is
used to generate the data container image. Please refer to this repository for more information.

## Create the Grouper configuration container for SOBS.

This step was relatively simple although a bit time-consuming. It involved ensuring that all of the relevant
configuration settings were present in the `group_vars` file for SOBS and creating a Jenkins job to build the
image. To create the Jenkins job, I simply copied the Grouper configuration image job for the `de-2` environment and
changed the default values of the build parameters. This new job still pushes the image to the CyVerse private Docker
registry, so I pulled the image to my workstation, retagged it, and pushed it to the SOBS private Docker registry.

## Create /etc/timezone on all hosts.

It was easiest to do this using Ansible:

```
myself$ echo 'America/Phoenix' > timezone
myself$ ansible -i inventories/sobs all -u root -m copy -a "src=timezone dest=/etc/timezone"
myself$ ansible -i inventories/sobs all -u root -a "yum reinstall -y tzdata"
```

Once the files were created, I wanted to verify that they were all correct:

```
myself$ ansible -i inventories/sobs all -u root -a "cat /etc/timezone"
```

And just to make sure that the time zone was correct:

```
myself$ ansible -i inventories/sobs all -u root -a "rm /etc/localtime"
myself$ ansible -i inventories/sobs all -u root -a "ln -s /usr/share/zoneinfo/America/Phoenix /etc/localtime"
myself$ ansible -i inventories/sobs all -u root -a "ls -l /etc/localtime"
ansible -i inventories/sobs all -u root -a "date"
```

## Add Grouper to the docker compose file for the SOBS deployment.

Configs:

``` yaml
config_grouper:
  image: sobs-de.sobs.arizona.edu:5000/grouper-configs-sobs:${DE_TAG}
  container_name: config-grouper
  labels:
    org.cyverse.name: "grouper"
    org.cyverse.type: "configs"
```

Service:

``` yaml
grouper:
  image: discoenv/grouper:2.2.2
  container_name: grouper
  restart: "unless-stopped"
  ports:
    - "8080:8080"
  log_driver: "syslog"
  log_opt:
    tag: "{{.ImageName}}/{{.Name}}"
  volumes_from:
    - config_grouper
  volumes:
    - "/etc/localtime:/etc/localtime"
    - "/etc/timezone:/etc/timezone"
  labels:
    org.cyverse.name: "grouper"
    org.cyverse.type: "service"
```

## Remove all of the zero-downtime deployment containers.

This involved a lot of changes. The easiest way to describe the changes is to include the file contents:

``` yaml
# -*- mode: yaml -*-

###
# Data containers that don't contain service configurations.
###
iplant_data_apps:
  image: sobs-de.sobs.arizona.edu:5000/iplant_data:latest
  labels:
    org.cyverse.name: "apps"
    org.cyverse.type: "data"

iplant_data_de_ui:
  image: sobs-de.sobs.arizona.edu:5000/iplant_data:latest
  labels:
    org.cyverse.name: "ui"
    org.cyverse.type: "data"

iplant_data_terrain:
  image: sobs-de.sobs.arizona.edu:5000/iplant_data:latest
  labels:
    org.cyverse.name: "terrain"
    org.cyverse.type: "data"

###
# Set up the configuration containers
###
config_anon_files:
  image: sobs-de.sobs.arizona.edu:5000/de-configs-${DE_ENV}:${DE_TAG}
  labels:
    org.cyverse.name: "anon-files"
    org.cyverse.type: "configs"

config_apps:
  image: sobs-de.sobs.arizona.edu:5000/de-configs-${DE_ENV}:${DE_TAG}
  labels:
    org.cyverse.name: "apps"
    org.cyverse.type: "configs"

config_clockwork:
  image: sobs-de.sobs.arizona.edu:5000/de-configs-${DE_ENV}:${DE_TAG}
  labels:
    org.cyverse.name: "clockwork"
    org.cyverse.type: "configs"

config_data_info:
  image: sobs-de.sobs.arizona.edu:5000/de-configs-${DE_ENV}:${DE_TAG}
  labels:
    org.cyverse.name: "data-info"
    org.cyverse.type: "configs"

config_de_ui:
  image: sobs-de.sobs.arizona.edu:5000/de-configs-${DE_ENV}:${DE_TAG}
  labels:
    org.cyverse.name: "ui"
    org.cyverse.type: "configs"

config_dewey:
  image: sobs-de.sobs.arizona.edu:5000/de-configs-${DE_ENV}:${DE_TAG}
  labels:
    org.cyverse.name: "dewey"
    org.cyverse.type: "configs"

config_grouper:
  image: sobs-de.sobs.arizona.edu:5000/grouper-configs-sobs:${DE_TAG}
  container_name: config-grouper
  labels:
    org.cyverse.name: "grouper"
    org.cyverse.type: "configs"

config_image_janitor:
  image: sobs-de.sobs.arizona.edu:5000/de-configs-${DE_ENV}:${DE_TAG}
  labels:
    org.cyverse.name: "image-janitor"
    org.cyverse.type: "configs"

config_info_typer:
  image: sobs-de.sobs.arizona.edu:5000/de-configs-${DE_ENV}:${DE_TAG}
  labels:
    org.cyverse.name: "info-typer"
    org.cyverse.type: "configs"

config_infosquito:
  image: sobs-de.sobs.arizona.edu:5000/de-configs-${DE_ENV}:${DE_TAG}
  labels:
    org.cyverse.name: "infosquito"
    org.cyverse.type: "configs"

config_iplant_email:
  image: sobs-de.sobs.arizona.edu:5000/de-configs-${DE_ENV}:${DE_TAG}
  labels:
    org.cyverse.name: "iplant-email"
    org.cyverse.type: configs

config_iplant_groups:
  image: sobs-de.sobs.arizona.edu:5000/de-configs-${DE_ENV}:${DE_TAG}
  labels:
    org.cyverse.name: "iplant-groups"
    org.cyverse.type: "configs"

config_jex_adapter:
  image: sobs-de.sobs.arizona.edu:5000/de-configs-${DE_ENV}:${DE_TAG}
  labels:
    org.cyverse.name: "jex-adapter"
    org.cyverse.type: "configs"

config_job_status_to_apps_adapter:
  image: sobs-de.sobs.arizona.edu:5000/de-configs-${DE_ENV}:${DE_TAG}
  labels:
    org.cyverse.name: "job-status-to-apps-adapter"
    org.cyverse.type: "configs"

config_job_status_recorder:
  image: sobs-de.sobs.arizona.edu:5000/de-configs-${DE_ENV}:${DE_TAG}
  labels:
    org.cyverse.name: "job-status-recorder"
    org.cyverse.type: "configs"

config_kifshare:
  image: sobs-de.sobs.arizona.edu:5000/de-configs-${DE_ENV}:${DE_TAG}
  labels:
    org.cyverse.name: "kifshare"
    org.cyverse.type: "configs"

config_metadata:
  image: sobs-de.sobs.arizona.edu:5000/de-configs-${DE_ENV}:${DE_TAG}
  labels:
    org.cyverse.name: "metadata"
    org.cyverse.type: "configs"

config_monkey:
  image: sobs-de.sobs.arizona.edu:5000/de-configs-${DE_ENV}:${DE_TAG}
  labels:
    org.cyverse.name: "monkey"
    org.cyverse.type: "configs"

config_notification_agent:
  image: sobs-de.sobs.arizona.edu:5000/de-configs-${DE_ENV}:${DE_TAG}
  labels:
    org.cyverse.name: "notification-agent"
    org.cyverse.type: "configs"

config_permissions:
  image: sobs-de.sobs.arizona.edu:5000/de-configs-${DE_ENV}:${DE_TAG}
  labels:
    org.cyverse.name: "permissions"
    org.cyverse.type: "configs"

config_saved_searches:
  image: sobs-de.sobs.arizona.edu:5000/de-configs-${DE_ENV}:${DE_TAG}
  labels:
    org.cyverse.name: "saved-searches"
    org.cyverse.type: "configs"

config_templeton_periodic:
  image: sobs-de.sobs.arizona.edu:5000/de-configs-${DE_ENV}:${DE_TAG}
  labels:
    org.cyverse.name: "templeton-periodic"
    org.cyverse.type: "configs"

config_templeton_incremental:
  image: sobs-de.sobs.arizona.edu:5000/de-configs-${DE_ENV}:${DE_TAG}
  labels:
    org.cyverse.name: "templeton-incremental"
    org.cyverse.type: "configs"

config_terrain:
  image: sobs-de.sobs.arizona.edu:5000/de-configs-${DE_ENV}:${DE_TAG}
  labels:
    org.cyverse.name: "terrain"
    org.cyverse.type: "configs"

config_tree_urls:
  image: sobs-de.sobs.arizona.edu:5000/de-configs-${DE_ENV}:${DE_TAG}
  labels:
    org.cyverse.name: "tree-urls"
    org.cyverse.type: "configs"

config_user_preferences:
  image: sobs-de.sobs.arizona.edu:5000/de-configs-${DE_ENV}:${DE_TAG}
  labels:
    org.cyverse.name: "user-preferences"
    org.cyverse.type: "configs"

config_user_sessions:
  image: sobs-de.sobs.arizona.edu:5000/de-configs-${DE_ENV}:${DE_TAG}
  labels:
    org.cyverse.name: "user-sessions"
    org.cyverse.type: "configs"

###
# Service and UI definitions
###
anon_files:
  image: discoenv/anon-files:${DE_TAG}
  command: --config /etc/iplant/de/anon-files.properties
  net: de-${DE_ENV}
  restart: unless-stopped
  environment:
    - JAVA_TOOL_OPTIONS=-Xmx2G -Djava.net.preferIPv4Stack=true
  ports:
    - "31102:60000"
  log_driver: "syslog"
  log_opt:
    tag: "{{.ImageName}}/{{.Name}}"
  volumes:
    - /etc/localtime:/etc/localtime
    - /etc/timezone:/etc/timezone
  volumes_from:
    - config_anon_files
  labels:
    org.cyverse.name: "anon-files"
    org.cyverse.type: "service"

apps:
  image: discoenv/apps:${DE_TAG}
  command: --config /etc/iplant/de/apps.properties
  net: de-${DE_ENV}
  restart: unless-stopped
  environment:
    - JAVA_TOOL_OPTIONS=-Xmx2G -Djava.net.preferIPv4Stack=true
  ports:
    - "31323:60000"
  log_driver: "syslog"
  log_opt:
    tag: "{{.ImageName}}/{{.Name}}"
  volumes:
    - /etc/localtime:/etc/localtime
    - /etc/timezone:/etc/timezone
  volumes_from:
    - iplant_data_apps
    - config_apps
  labels:
    org.cyverse.name: "apps"
    org.cyverse.type: "service"

clockwork:
  image: discoenv/clockwork:${DE_TAG}
  command: --config /etc/iplant/de/clockwork.properties
  net: de-${DE_ENV}
  restart: unless-stopped
  environment:
    - JAVA_TOOL_OPTIONS=-Xmx1G -Djava.net.preferIPv4Stack=true
  log_driver: "syslog"
  log_opt:
    tag: "{{.ImageName}}/{{.Name}}"
  volumes_from:
    - config_clockwork
  volumes:
    - /etc/localtime:/etc/localtime
    - /etc/timezone:/etc/timezone
  labels:
    org.cyverse.name: "clockwork"
    org.cyverse.type: "service"

data_info:
  image: discoenv/data-info:${DE_TAG}
  command: --config /etc/iplant/de/data-info.properties
  net: de-${DE_ENV}
  restart: unless-stopped
  environment:
    - JAVA_TOOL_OPTIONS=-Xmx2G -Djava.net.preferIPv4Stack=true
  ports:
    - "31360:60000"
  log_driver: "syslog"
  log_opt:
    tag: "{{.ImageName}}/{{.Name}}"
  volumes:
    - /etc/localtime:/etc/localtime
    - /etc/timezone:/etc/timezone
  volumes_from:
    - config_data_info
  labels:
    org.cyverse.name: "data-info"
    org.cyverse.type: "service"

de_ui:
  image: discoenv/de:${DE_TAG}
  net: de-${DE_ENV}
  restart: unless-stopped
  ports:
    - "8081:8080"
  environment:
    - JAVA_TOOL_OPTIONS=-Djava.net.preferIPv4Stack=true
  log_driver: "syslog"
  log_opt:
    tag: "{{.ImageName}}/{{.Name}}"
  volumes:
    - /var/log/de/:/home/iplant/log/
    - /etc/localtime:/etc/localtime
    - /etc/timezone:/etc/timezone
  volumes_from:
    - iplant_data_de_ui
    - config_de_ui
  labels:
    org.cyverse.name: "ui"
    org.cyverse.type: "service"

dewey:
  image: discoenv/dewey:${DE_TAG}
  command: --config /etc/iplant/de/dewey.properties
  net: de-${DE_ENV}
  restart: unless-stopped
  environment:
    - JAVA_TOOL_OPTIONS=-Xmx1G -Djava.net.preferIPv4Stack=true
  log_driver: "syslog"
  log_opt:
    tag: "{{.ImageName}}/{{.Name}}"
  volumes_from:
    - config_dewey
  volumes:
    - /etc/localtime:/etc/localtime
    - /etc/timezone:/etc/timezone
  labels:
    org.cyverse.name: "dewey"
    org.cyverse.type: "service"

exim_sender:
  image: discoenv/exim-sender
  container_name: local-exim
  expose:
    - "25"
  net: de-${DE_ENV}
  restart: unless-stopped
  environment:
    - PRIMARY_HOST=sobs.arizona.edu
    - ALLOWED_HOSTS=*
  log_driver: "syslog"
  log_opt:
    tag: "{{.ImageName}}/{{.Name}}"
  volumes:
    - /etc/localtime:/etc/localtime
    - /etc/timezone:/etc/timezone
  labels:
    org.cyverse.name: "exim-sender"
    org.cyverse.type: "service"

grouper:
  image: discoenv/grouper:2.2.2
  container_name: grouper
  restart: "unless-stopped"
  ports:
    - "8080:8080"
  log_driver: "syslog"
  log_opt:
    tag: "{{.ImageName}}/{{.Name}}"
  volumes_from:
    - config_grouper
  volumes:
    - "/etc/localtime:/etc/localtime"
    - "/etc/timezone:/etc/timezone"
  labels:
    org.cyverse.name: "grouper"
    org.cyverse.type: "service"

image_janitor:
  image: discoenv/image-janitor:${DE_TAG}
  command: --config /etc/iplant/de/jobservices.yml
  net: de-${DE_ENV}
  restart: unless-stopped
  log_driver: "syslog"
  log_opt:
    tag: "{{.ImageName}}/{{.Name}}"
  volumes_from:
    - config_image_janitor
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock
    - /opt/image-janitor:/opt/image-janitor
  labels:
    org.cyverse.name: "image-janitor"
    org.cyverse.type: "service"

info_typer:
  image: discoenv/info-typer:${DE_TAG}
  command: --config /etc/iplant/de/info-typer.properties
  net: de-${DE_ENV}
  restart: unless-stopped
  environment:
    - JAVA_TOOL_OPTIONS=-Xmx1G -Djava.net.preferIPv4Stack=true
  log_driver: "syslog"
  log_opt:
    tag: "{{.ImageName}}/{{.Name}}"
  volumes_from:
    - config_info_typer
  volumes:
    - /etc/localtime:/etc/localtime
    - /etc/timezone:/etc/timezone
  labels:
    org.cyverse.name: "info-typer"
    org.cyverse.type: "service"

infosquito:
  image: discoenv/infosquito:${DE_TAG}
  command: --config /etc/iplant/de/infosquito.properties
  net: de-${DE_ENV}
  restart: unless-stopped
  environment:
    - JAVA_TOOL_OPTIONS=-Xmx1G -Djava.net.preferIPv4Stack=true
  log_driver: "syslog"
  log_opt:
    tag: "{{.ImageName}}/{{.Name}}"
  volumes_from:
    - config_infosquito
  volumes:
    - /etc/localtime:/etc/localtime
    - /etc/timezone:/etc/timezone
  labels:
    org.cyverse.name: "infosquito"
    org.cyverse.type: "service"

iplant_email:
  image: discoenv/iplant-email:${DE_TAG}
  command: --config /etc/iplant/de/iplant-email.properties
  net: de-${DE_ENV}
  restart: unless-stopped
  environment:
    - JAVA_TOOL_OPTIONS=-Xmx1G -Djava.net.preferIPv4Stack=true
  ports:
    - "31337:60000"
  log_driver: "syslog"
  log_opt:
    tag: "{{.ImageName}}/{{.Name}}"
  volumes:
    - /etc/localtime:/etc/localtime
    - /etc/timezone:/etc/timezone
  volumes_from:
    - config_iplant_email
  links:
    - exim_sender:local-exim
  labels:
    org.cyverse.name: "iplant-email"
    org.cyverse.type: "service"

iplant_groups:
  image: discoenv/iplant-groups:${DE_TAG}
  command: --config /etc/iplant/de/iplant-groups.properties
  net: de-${DE_ENV}
  restart: unless-stopped
  environment:
    - JAVA_TOOL_OPTIONS=-Xmx1G -Djava.net.preferIPv4Stack=true
  ports:
    - "31310:60000"
  log_driver: "syslog"
  log_opt:
    tag: "{{.ImageName}}/{{.Name}}"
  volumes:
    - /etc/localtime:/etc/localtime
    - /etc/timezone:/etc/timezone
  volumes_from:
    - config_iplant_groups
  labels:
    org.cyverse.name: "iplant-groups"
    org.cyverse.type: "service"

jex_adapter:
  image: discoenv/jex-adapter:${DE_TAG}
  command: --config /etc/iplant/de/jobservices.yml
  expose:
    - "60000"
  net: de-${DE_ENV}
  restart: unless-stopped
  log_driver: "syslog"
  log_opt:
    tag: "{{.ImageName}}/{{.Name}}"
  volumes:
    - /etc/localtime:/etc/localtime
    - /etc/timezone:/etc/timezone
  volumes_from:
    - config_jex_adapter
  labels:
    org.cyverse.name: "jex-adapter"
    org.cyverse.type: "service"

job_status_to_apps_adapter:
  image: discoenv/job-status-to-apps-adapter:${DE_TAG}
  command: --config /etc/iplant/de/jobservices.yml
  net: de-${DE_ENV}
  restart: unless-stopped
  log_driver: "syslog"
  log_opt:
    tag: "{{.ImageName}}/{{.Name}}"
  volumes_from:
    - config_job_status_to_apps_adapter
  volumes:
    - /etc/localtime:/etc/localtime
    - /etc/timezone:/etc/timezone
  labels:
    org.cyverse.name: "job-status-to-apps-adapter"
    org.cyverse.type: "service"

job_status_recorder:
  image: discoenv/job-status-recorder:${DE_TAG}
  command: --config /etc/iplant/de/jobservices.yml
  net: de-${DE_ENV}
  restart: unless-stopped
  log_driver: "syslog"
  log_opt:
    tag: "{{.ImageName}}/{{.Name}}"
  volumes_from:
    - config_job_status_recorder
  volumes:
    - /etc/localtime:/etc/localtime
    - /etc/timezone:/etc/timezone
  labels:
    org.cyverse.name: "job-status-recorder"
    org.cyverse.type: "service"

kifshare:
  image: discoenv/kifshare:${DE_TAG}
  command: --config /etc/iplant/de/kifshare.properties
  net: de-${DE_ENV}
  restart: unless-stopped
  environment:
    - JAVA_TOOL_OPTIONS=-Xmx2G -Djava.net.preferIPv4Stack=true
  ports:
    - "31380:60000"
  log_driver: "syslog"
  log_opt:
    tag: "{{.ImageName}}/{{.Name}}"
  volumes:
    - /etc/localtime:/etc/localtime
    - /etc/timezone:/etc/timezone
  volumes_from:
    - config_kifshare
  labels:
    org.cyverse.name: "kifshare"
    org.cyverse.type: "service"

metadata:
  image: discoenv/metadata:${DE_TAG}
  command: --config /etc/iplant/de/metadata.properties
  net: de-${DE_ENV}
  restart: unless-stopped
  environment:
    - JAVA_TOOL_OPTIONS=-Xmx2G -Djava.net.preferIPv4Stack=true
  ports:
    - "31331:60000"
  log_driver: "syslog"
  log_opt:
    tag: "{{.ImageName}}/{{.Name}}"
  volumes:
    - /etc/localtime:/etc/localtime
    - /etc/timezone:/etc/timezone
  volumes_from:
    - config_metadata
  labels:
    org.cyverse.name: "metadata"
    org.cyverse.type: "service"

monkey:
  image: discoenv/monkey:${DE_TAG}
  command: --config /etc/iplant/de/monkey.properties
  net: de-${DE_ENV}
  restart: unless-stopped
  environment:
    - JAVA_TOOL_OPTIONS=-Xmx1G -Djava.net.preferIPv4Stack=true
  log_driver: "syslog"
  log_opt:
    tag: "{{.ImageName}}/{{.Name}}"
  volumes_from:
    - config_monkey
  volumes:
    - /etc/localtime:/etc/localtime
    - /etc/timezone:/etc/timezone
  labels:
    org.cyverse.name: "monkey"
    org.cyverse.type: "service"

notification_agent:
  image: discoenv/notification-agent:${DE_TAG}
  command: --config /etc/iplant/de/notificationagent.properties
  net: de-${DE_ENV}
  restart: unless-stopped
  environment:
    - JAVA_TOOL_OPTIONS=-Xmx1G -Djava.net.preferIPv4Stack=true
  ports:
    - "31320:60000"
  log_driver: "syslog"
  log_opt:
    tag: "{{.ImageName}}/{{.Name}}"
  volumes:
    - /etc/localtime:/etc/localtime
    - /etc/timezone:/etc/timezone
  volumes_from:
    - config_notification_agent
  labels:
    org.cyverse.name: "notification-agent"
    org.cyverse.type: "service"

permissions:
  image: discoenv/permissions:${DE_TAG}
  command: --host 0.0.0.0 --port 60000
  net: de-${DE_ENV}
  restart: unless-stopped
  ports:
    - "31308:60000"
  log_driver: "syslog"
  log_opt:
    tag: "{{.ImageName}}/{{.Name}}"
  volumes:
    - /etc/localtime:/etc/localtime
    - /etc/timezone:/etc/timezone
  volumes_from:
    - config_permissions
  labels:
    org.cyverse.name: "permissions"
    org.cyverse.type: "service"

saved_searches:
  image: discoenv/saved-searches:${DE_TAG}
  command: --config /etc/iplant/de/jobservices.yml
  net: de-${DE_ENV}
  restart: unless-stopped
  ports:
    - "31306:60000"
  log_driver: "syslog"
  log_opt:
    tag: "{{.ImageName}}/{{.Name}}"
  volumes:
    - /etc/localtime:/etc/localtime
    - /etc/timezone:/etc/timezone
  volumes_from:
    - config_saved_searches
  labels:
    org.cyverse.name: "saved-searches"
    org.cyverse.type: "service"

templeton_periodic:
  image: discoenv/templeton:${DE_TAG}
  command: --mode periodic --config /etc/iplant/de/templeton-periodic.yaml
  restart: unless-stopped
  log_driver: "syslog"
  log_opt:
    tag: "{{.ImageName}}/{{.Name}}"
  volumes_from:
    - config_templeton_periodic
  volumes:
    - /etc/localtime:/etc/localtime
    - /etc/timezone:/etc/timezone
  labels:
    org.cyverse.name: "templeton-periodic"
    org.cyverse.type: "service"

templeton_incremental:
  image: discoenv/templeton:${DE_TAG}
  command: --mode incremental --config /etc/iplant/de/templeton-incremental.yaml
  restart: unless-stopped
  log_driver: "syslog"
  log_opt:
    tag: "{{.ImageName}}/{{.Name}}"
  volumes_from:
    - config_templeton_incremental
  volumes:
    - /etc/localtime:/etc/localtime
    - /etc/timezone:/etc/timezone
  labels:
    org.cyverse.name: "templeton-incremental"
    org.cyverse.type: "service"

terrain:
  image: discoenv/terrain:${DE_TAG}
  command: --config /etc/iplant/de/terrain.properties
  net: de-${DE_ENV}
  restart: unless-stopped
  environment:
    - JAVA_TOOL_OPTIONS=-Xmx2G -Djava.net.preferIPv4Stack=true
  ports:
    - "31325:60000"
  log_driver: "syslog"
  log_opt:
    tag: "{{.ImageName}}/{{.Name}}"
  volumes:
    - /etc/localtime:/etc/localtime
    - /etc/timezone:/etc/timezone
  volumes_from:
    - iplant_data_terrain
    - config_terrain
  labels:
    org.cyverse.name: "terrain"
    org.cyverse.type: "service"

tree_urls:
  image: discoenv/tree-urls:${DE_TAG}
  command: --config /etc/iplant/de/jobservices.yml
  net: de-${DE_ENV}
  restart: unless-stopped
  ports:
    - "31307:60000"
  log_driver: "syslog"
  log_opt:
    tag: "{{.ImageName}}/{{.Name}}"
  volumes:
    - /etc/localtime:/etc/localtime
    - /etc/timezone:/etc/timezone
  volumes_from:
    - config_tree_urls
  labels:
    org.cyverse.name: "tree-urls"
    org.cyverse.type: "service"

user_preferences:
  image: discoenv/user-preferences:${DE_TAG}
  command: --config /etc/iplant/de/jobservices.yml
  net: de-${DE_ENV}
  restart: unless-stopped
  ports:
    - "31305:60000"
  log_driver: "syslog"
  log_opt:
    tag: "{{.ImageName}}/{{.Name}}"
  volumes:
    - /etc/localtime:/etc/localtime
    - /etc/timezone:/etc/timezone
  volumes_from:
    - config_user_preferences
  labels:
    org.cyverse.name: "user-preferences"
    org.cyverse.type: "service"

user_sessions:
  image: discoenv/user-sessions:${DE_TAG}
  command: --config /etc/iplant/de/jobservices.yml
  net: de-${DE_ENV}
  restart: unless-stopped
  ports:
    - "31304:60000"
  log_driver: "syslog"
  log_opt:
    tag: "{{.ImageName}}/{{.Name}}"
  volumes:
    - /etc/localtime:/etc/localtime
    - /etc/timezone:/etc/timezone
  volumes_from:
    - config_user_sessions
  labels:
    org.cyverse.name: "user-sessions"
    org.cyverse.type: "service"
```

## Ensure that the SOBS group variables file has all required settings for generating DE config files.

This was one of the more tedious tasks associated with the SOBS deployment. I went through all of the templates in the
`util-cfg-service` role and verified that all of the group variables referenced in the templates would be set
correctly. One beneficial side-effect of this task was that I was able to remove some dead code.

## Add reverse proxies to Apache HTTPD.

I added this information in two separate sections, the first section is intended to add CORS support to all locations in
the HTTP virtual host.

```
# Allow cross-origin resource sharing.
Header set Access-Control-Allow-Origin "*"
Header set Access-Control-Allow-Credentials "true"
Header set Access-Control-Allow-Methods "GET, POST, OPTIONS"
Header set Access-Control-Allow-Headers "DNT,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Content-Range,Range"
Header set Access-Control-Expose-Headers "DNT,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Content-Range,Range"
```

The second section sets up the reverse proxies:

```
ProxyTimeout 3600
ProxyAddHeaders On

ProxyPass /anon-files/ http://localhost:31102/
ProxyPassReverse /anon-files/ http://localhost:31102/

ProxyPass /dl/ http://localhost:31380/
ProxyPassReverse /dl/ http://localhost:31380/

ProxyPass /de/agave-cb http://sobs-services.sobs.arizona.edu:31323/callbacks/agave-job
ProxyPassReverse /de/agave-cb http://sobs-services.sobs.arizona.edu:31323/callbacks/agave-job

<Location "/de/websocket">
    Header set Upgrade "websocket"
    Header set Connection "Upgrade"

    ProxyPass http://localhost:8081
    ProxyPassReverse http://localhost:8081
</Location>

ProxyPass / http://localhost:8081
ProxyPassReverse / http://localhost:8081
```

Both of these changes are in `/etc/httpd/conf.d/ssl.conf`.

## Install CAS.

I wanted to make CAS a little easier to deploy here than it is in other environments, so I decided to deploy it in a
Docker container. The first step was to create a CAS overlay repository for it. The repository, `sobs/sobs-cas`, in
GitLab. For this step, I copied and modified the CyVerse `cas-overlay` deployment. Most of the changes simply involved
updating the configuration settings and renaming some files. I'm not going to list the changes here simply because they
can be found by checking out the git repository.

The deployment on the server was also relatively easy once I figured out all of the hoops I had to jump through to get
LDAPS and HTTPS connections working. The first step was to create the service definition file, which is stored in
`/usr/lib/systemd/system/sobs-cas.service` on the host system:

```
[Unit]
Description=Apereo CAS for the SOBS DE Deployment
BindsTo=docker.service
PartOf=docker.service
After=docker.service
Requisite=docker.service

[Service]
ExecStartPre=-/usr/bin/docker rm sobs_cas
ExecStart=/usr/bin/docker run --name sobs_cas \
-v /etc/localtime:/etc/localtime:ro \
-v /etc/openldap:/etc/openldap:ro \
-v /usr/local/etc/letsencrypt/sobs-de.sobs.arizona.edu:/usr/local/etc/letsencrypt/sobs-de.sobs.arizona.edu:ro \
-v /etc/tomcat/server.xml:/usr/local/tomcat/conf/server.xml:ro \
-p 8082:8443 \
--log-driver=journald \
sobs-de.sobs.arizona.edu:5000/sobs-cas:latest
ExecStop=-/usr/bin/docker stop sobs_cas
Restart=on-failure

SyslogIdentifier=sobs_cas
SyslogFacility=local6

[Install]
WantedBy=multi-user.target
```

There's a few things of note in this file. First, bind-mounting `/etc/openldap` was required in order to get LDAPS to
work. If you examine `cas.properties` in the `sobs/sobs-cas` repository, you'll notice that the CAS configuration
references a certificate file in `/etc/openldap/cacerts`. Second, bind-mounting the LetsEncrypt directory was required
in order to enable SSL in Tomcat. We couldn't rely solely on Apache HTTPD to provide the SSL because CAS displays a
warning message any time its authentication pabge is served over a non-SSL connection. Finally, we're also bind-mounting
a local copy of Tomcat's `server.xml` configuration file in order to enable the SSL connection. The only change to this
file was to define the SSL connector:

```
    <Connector
           protocol="org.apache.coyote.http11.Http11AprProtocol"
           port="8443" maxThreads="200"
           scheme="https" secure="true" SSLEnabled="true"
           SSLCertificateFile="/usr/local/etc/letsencrypt/sobs-de.sobs.arizona.edu/signed.crt"
           SSLCertificateKeyFile="/usr/local/etc/letsencrypt/sobs-de.sobs.arizona.edu/domain.key"
           SSLVerifyClient="optional" SSLProtocol="TLSv1+TLSv1.1+TLSv1.2"/>
```

Because CAS will be using the certificates on the host system, it's also necessary to restart it when the certificates
are renewed. I added the following line to the script that renews the certificates:

```
systemctl restart sobs-cas
```

## Place `/etc/docker-compose.yml`.

Since we're not using consul for this deployment, I had to make a small change to the role that does this in order to
get the `de-dc` script to be generated correctly. Once I made the change, I simply ran the playbook:

```
myself$ ansible-playbook -i inventories/sobs -K infra-docker-compose.yml
```

## Initialize the Grouper database.

For this step, I need to launch a Grouper container with access to the configuration settings:

```
root# de-dc pull grouper
root# de-dc pull config_grouper
root# de-dc up -d config_grouper
root# docker run --rm -it --name grouper \
                 -v /etc/timezone:/etc/timezone \
                 -v /etc/localtime:/etc/localtime \
                 --volumes-from=grouper-config \
                 --entrypoint=sh \
                 discoenv/grouper:2.2.2
```

Once inside the container, I used the following command to initialize the database:

```
root# gsh -registry -init
```

This command generates an SQL file that can be loaded with GSH:

```
root# gsh -registry -runsqlfile /opt/grouper/api/ddlScripts/grouperDdl_20170113_14_04_10_546.sql
```

Note: the name of the SQL file will appear in the output from the previous command.

As an aside, I ran into some errors related to Grouper not being able to establish the SSL connection when I tried to
load the SQL file. I made several attempts to get the SSL working, but none of them worked. I eventually gave up and
switched back to a plain text LDAP connection. I can revisit this after everything is working if necessary.

## Initialize the databases used by the de.

This step was pretty straight-forward once I remembered exactly what had to be done. The `db-migrations` playbook with
the extra variable, `facepalm_mode` set to `init`. Is enough to do this:

```
myself$ ansible-playbook -K -i $DE_SCM_REPOS_DIR/de-ansible/inventories/sobs --extra-vars="facepalm_mode=init" \
        db-migrations.yaml
```

## Create the groups used by the DE.

I ran sharkbait from within a Docker container for this one:

```
root# docker run --rm -it --name sharkbait \
      -v /root/.pgpass:/root/.pgpass \
      -v /etc/timezone:/etc/timezone \
      -v /etc/localtime:/etc/localtime \
      -v /etc/openldap:/etc/openldap \
      -v /usr/local/etc/letsencrypt/sobs-de.sobs.arizona.edu:/etc/ssl \
      --volumes-from=config-grouper \
      discoenv/sharkbait:latest -h sobs-pgsql.sobs.arizona.edu -e sobs
```

## Updated data container names containing `iplant` in the `docker-compose.yaml` file for SOBS:

I did a selective search and replace for this. The data container names all begin with `sobs_data_` rather than
`iplant_data_`.

## Deployed the DE.

I used the `deploy-all` playbook for this:

```
myself$ ansible-playbook -i $DE_SCM_REPOS_DIR/de-ansible/inventories/sobs -K deploy-all.yaml
```

A few changes were necessary to get this to work.

## Checked for presence of UUIDs in iRODS.

```
root# iinit
root# imeta ls -c /sobs/home
```

No UUID was associated with the collection, so I checked the logs in `/var/lib/irods/iRODS/server/log/rodsLog*` to see
what errors were present. I found this error:

```
line 11, col 4, rule base ipc-uuid
    fail;
    ^

caused by: execMicroService3: error when executing microservice
line 5, col 22, rule base ipc-uuid
  *status = errorcode(msiExecCmd("generateuuid.sh", "", ipc_RE_HOST, "null", "null", *out));
                      ^

caused by: DEBUG: msiExecCmd: rsExecCmd failed for generateuuid.sh, status = -344000
```

### Assigning UUIDs to existing items.

I searched for the `generateuuid.sh` script on one of our other data store hosts and ended up copying the script from
one of our development environments. After copying the script, I had to generate UUIDs for all of the data objects and
collections in iRODS. A quick run of `ils -r /` revealed that only collections had been made so far, which made
generating UUIDs easy. The first step was to get the list of collection names:

```
root# ils -r / | grep '  C- ' | sed 's/  C- //' > collections.txt
```

The next step was to associate UUIDs with all of the collections:

```
root# for dir in $(< collections.txt); do imeta add -c "$dir" "ipc_UUID" $(uuidgen -t); done
```

The last step was to verify that UUIDs had been successfully assigned to all of the collections:

```
root# for dir in $(< collections.txt); do imeta ls -c "$dir"; done
```

### Testing the rules on a new collection.

For this step, I simply created a new collection and checked for the presence of a UUID, which was not present. The
following error showed up in the logs:

```
Jan 17 16:19:01 pid:11099 ERROR: [-]    iRODS/server/re/src/rules.cpp:670:actionTableLookUp :  status [PLUGIN_ERROR_MISSING_SHARED_OBJECT]  errno [] -- message []
        [-]     iRODS/server/re/src/irods_ms_plugin.cpp:110:load_microservice_plugin :  status [PLUGIN_ERROR_MISSING_SHARED_OBJECT]  errno [] -- message [Failed to create ms plugin entry.]
                [-]     iRODS/lib/core/include/irods_load_plugin.hpp:175:load_plugin :  status [PLUGIN_ERROR_MISSING_SHARED_OBJECT]  errno [] -- message [shared library does not exist [/var/lib/irods/plugins/microservices/libmsiSetAVU.so]]
```

I ended up fixing this problem by copying the shared object file from our development environment. After trying again,
the following error was generated:

```
Jan 17 16:37:42 pid:11986 NOTICE: writeLine: inString = Failed to send AMQP message: /usr/bin/env: python2.6: No such file or directory
```

This was easy enough to fix. Since the new iRODS server is running CentOS 7, `python` refers to version `2.7.5`, which
has all of the libraries that we need. I modified `amqptopicsend.py` to use `python` rather than `python2.6`. Upon
trying again, the following error was generated:

```
Jan 17 16:41:53 pid:12094 NOTICE: writeLine: inString = Failed to send AMQP message: ERROR: (406, "PRECONDITION_FAILED - cannot redeclare exchange 'irods' in vhost '/sobs/data-store' with different type, durable, internal or autodelete value")
```

It appears that `amqptopicsend.py` declares the exchange by default, so I simply deleted the exchange:

```
root# rmq -V /sobs/data-store delete exchange name=irods
```

After deleting and re-creating the folder, the UUID was successfully generated:

```
root# imeta ls -c foo
AVUs defined for collection foo:
attribute: ipc_UUID
value: b9d4ec08-ddc0-11e6-9e57-90e2baa983f5
units:
```

### Testing the roles on a new data object.

Just for good measure, I tested this with a data object as well:

```
root# echo foo > foo.txt
root# iput foo.txt
root# imeta ls -d foo.txt
AVUs defined for dataObj foo.txt:
attribute: ipc_UUID
value: 31b5786e-ddc1-11e6-9873-90e2baa983f5
units:
```

## Testing DE logins.

The first error that I encountered was a `NoRouteToHost` exception. I copied the exception into a file on my machine
then used `jq` to examine the stack trace:

```
myself$ jq -r .stack_trace err.json
java.lang.RuntimeException: java.net.NoRouteToHostException: Host is unreachable
    at org.jasig.cas.client.util.CommonUtils.getResponseFromServer(CommonUtils.java:407) ~[cas-client-core-3.3.3.jar!/:3.3.3]
    at org.jasig.cas.client.validation.AbstractCasProtocolUrlBasedTicketValidator.retrieveResponseFromServer(AbstractCasProtocolUrlBasedTicketValidator.java:45) ~[cas-client-core-3.3.3.jar!/:3.3.3]
    at org.jasig.cas.client.validation.AbstractUrlBasedTicketValidator.validate(AbstractUrlBasedTicketValidator.java:200) ~[cas-client-core-3.3.3.jar!/:3.3.3]
    at org.springframework.security.cas.authentication.CasAuthenticationProvider.authenticateNow(CasAuthenticationProvider.java:140) ~[spring-security-cas-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.cas.authentication.CasAuthenticationProvider.authenticate(CasAuthenticationProvider.java:126) ~[spring-security-cas-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.authentication.ProviderManager.authenticate(ProviderManager.java:156) ~[spring-security-core-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.cas.web.CasAuthenticationFilter.attemptAuthentication(CasAuthenticationFilter.java:242) ~[spring-security-cas-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.authentication.AbstractAuthenticationProcessingFilter.doFilter(AbstractAuthenticationProcessingFilter.java:211) ~[spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342) ~[spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.jasig.cas.client.session.SingleSignOutFilter.doFilter(SingleSignOutFilter.java:100) ~[cas-client-core-3.3.3.jar!/:3.3.3]
    at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342) ~[spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.authentication.logout.LogoutFilter.doFilter(LogoutFilter.java:110) ~[spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342) ~[spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.authentication.logout.LogoutFilter.doFilter(LogoutFilter.java:110) ~[spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342) ~[spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.authentication.logout.LogoutFilter.doFilter(LogoutFilter.java:110) ~[spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342) ~[spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.header.HeaderWriterFilter.doFilterInternal(HeaderWriterFilter.java:57) ~[spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107) ~[spring-web-4.1.6.RELEASE.jar!/:4.1.6.RELEASE]
    at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342) ~[spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.context.SecurityContextPersistenceFilter.doFilter(SecurityContextPersistenceFilter.java:87) ~[spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342) ~[spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter.doFilterInternal(WebAsyncManagerIntegrationFilter.java:50) ~[spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107) ~[spring-web-4.1.6.RELEASE.jar!/:4.1.6.RELEASE]
    at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342) ~[spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.FilterChainProxy.doFilterInternal(FilterChainProxy.java:192) ~[spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.FilterChainProxy.doFilter(FilterChainProxy.java:160) ~[spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:239) ~[tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:206) ~[tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:85) ~[spring-web-4.1.6.RELEASE.jar!/:4.1.6.RELEASE]
    at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107) ~[spring-web-4.1.6.RELEASE.jar!/:4.1.6.RELEASE]
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:239) ~[tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:206) ~[tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:219) ~[tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:106) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:501) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:142) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:79) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:88) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:516) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.coyote.http11.AbstractHttp11Processor.process(AbstractHttp11Processor.java:1086) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.coyote.AbstractProtocol$AbstractConnectionHandler.process(AbstractProtocol.java:659) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.coyote.http11.Http11NioProtocol$Http11ConnectionHandler.process(Http11NioProtocol.java:223) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1558) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.run(NioEndpoint.java:1515) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142) [na:1.8.0_92-internal]
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617) [na:1.8.0_92-internal]
    at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at java.lang.Thread.run(Thread.java:745) [na:1.8.0_92-internal]
Caused by: java.net.NoRouteToHostException: Host is unreachable
    at java.net.PlainSocketImpl.socketConnect(Native Method) ~[na:1.8.0_92-internal]
    at java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:350) ~[na:1.8.0_92-internal]
    at java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:206) ~[na:1.8.0_92-internal]
    at java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:188) ~[na:1.8.0_92-internal]
    at java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392) ~[na:1.8.0_92-internal]
    at java.net.Socket.connect(Socket.java:589) ~[na:1.8.0_92-internal]
    at sun.security.ssl.SSLSocketImpl.connect(SSLSocketImpl.java:668) ~[na:1.8.0_92-internal]
    at sun.security.ssl.BaseSSLSocketImpl.connect(BaseSSLSocketImpl.java:173) ~[na:1.8.0_92-internal]
    at sun.net.NetworkClient.doConnect(NetworkClient.java:180) ~[na:1.8.0_92-internal]
    at sun.net.www.http.HttpClient.openServer(HttpClient.java:432) ~[na:1.8.0_92-internal]
    at sun.net.www.http.HttpClient.openServer(HttpClient.java:527) ~[na:1.8.0_92-internal]
    at sun.net.www.protocol.https.HttpsClient.<init>(HttpsClient.java:264) ~[na:1.8.0_92-internal]
    at sun.net.www.protocol.https.HttpsClient.New(HttpsClient.java:367) ~[na:1.8.0_92-internal]
    at sun.net.www.protocol.https.AbstractDelegateHttpsURLConnection.getNewHttpClient(AbstractDelegateHttpsURLConnection.java:191) ~[na:1.8.0_92-internal]
    at sun.net.www.protocol.http.HttpURLConnection.plainConnect0(HttpURLConnection.java:1105) ~[na:1.8.0_92-internal]
    at sun.net.www.protocol.http.HttpURLConnection.plainConnect(HttpURLConnection.java:999) ~[na:1.8.0_92-internal]
    at sun.net.www.protocol.https.AbstractDelegateHttpsURLConnection.connect(AbstractDelegateHttpsURLConnection.java:177) ~[na:1.8.0_92-internal]
    at sun.net.www.protocol.http.HttpURLConnection.getInputStream0(HttpURLConnection.java:1513) ~[na:1.8.0_92-internal]
    at sun.net.www.protocol.http.HttpURLConnection.getInputStream(HttpURLConnection.java:1441) ~[na:1.8.0_92-internal]
    at sun.net.www.protocol.https.HttpsURLConnectionImpl.getInputStream(HttpsURLConnectionImpl.java:254) ~[na:1.8.0_92-internal]
    at org.jasig.cas.client.util.CommonUtils.getResponseFromServer(CommonUtils.java:393) ~[cas-client-core-3.3.3.jar!/:3.3.3]
    ... 48 common frames omitted
```

A quick examiniation of the stack trace revealed that this was a problem validating the CAS service ticket. I attempted
to connect to the CAS service from within the container:

```
root# docker exec -it etc_de_ui_1 sh
container-root# nc -v sobs-de.sobs.arizona.edu 443
nc: sobs-de.sobs.arizona.edu (128.196.59.20:443): Host is unreachable
```

This confirms that the host was unreachable, so I opened up the HTTPS port to the docker network and restarted
Docker. I also redeployed the DE to the SOBS environment. It's just easier to restart the services that way. Updating
the firewall rules and restarting Docker was enough to get past that error.

The next error that I encountered was a redirection error. I was being redirected back to `http` rather than `https`,
which prevented the login from working, partially because the SSL redirect had not been added to the Apache
configs. Adding the SSL redirect was enough to fix this error:

```
Redirect permanent /de https://sobs-de.sobs.arizona.edu/de
Redirect permanent /cas https://sobs-de.sobs.arizona.edu/cas
```

The next error that I encountered was a failure to load the JWT signing key. Once again, I copied the error to a file on
my local computer and used `jq` to get the stack trace:

```
myself$ jq -r .stack_trace err.json
java.lang.IllegalArgumentException: Unable to load JWT signing key
    at org.iplantc.de.server.auth.DefaultJwtBuilder.getPrivateKey(DefaultJwtBuilder.java:37) ~[de-lib-2.8.0.jar!/:na]
    at org.iplantc.de.server.auth.DefaultJwtBuilder.buildJwt(DefaultJwtBuilder.java:66) ~[de-lib-2.8.0.jar!/:na]
    at org.iplantc.de.server.auth.JwtUrlConnector.addHeaders(JwtUrlConnector.java:69) ~[de-lib-2.8.0.jar!/:na]
    at org.iplantc.de.server.auth.JwtUrlConnector.getRequest(JwtUrlConnector.java:33) ~[de-lib-2.8.0.jar!/:na]
    at org.iplantc.de.server.services.DEServiceImpl.getResponse(DEServiceImpl.java:215) [de-lib-2.8.0.jar!/:na]
    at org.iplantc.de.server.services.DEServiceImpl.getServiceData(DEServiceImpl.java:89) [de-lib-2.8.0.jar!/:na]
    at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:1.8.0_92-internal]
    at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[na:1.8.0_92-internal]
    at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:1.8.0_92-internal]
    at java.lang.reflect.Method.invoke(Method.java:498) ~[na:1.8.0_92-internal]
    at com.google.gwt.user.server.rpc.RPC.invokeAndEncodeResponse(RPC.java:587) [gwt-servlet-2.7.0.jar!/:na]
    at com.google.gwt.user.server.rpc.RPC.invokeAndEncodeResponse(RPC.java:571) [gwt-servlet-2.7.0.jar!/:na]
    at com.google.gwt.user.server.rpc.RPC.invokeAndEncodeResponse(RPC.java:533) [gwt-servlet-2.7.0.jar!/:na]
    at org.iplantc.de.server.rpc.GwtRpcController.processCall(GwtRpcController.java:72) [classes!/:na]
    at com.google.gwt.user.server.rpc.RemoteServiceServlet.processPost(RemoteServiceServlet.java:373) [gwt-servlet-2.7.0.jar!/:na]
    at com.google.gwt.user.server.rpc.AbstractRemoteServiceServlet.doPost(AbstractRemoteServiceServlet.java:62) [gwt-servlet-2.7.0.jar!/:na]
    at org.iplantc.de.server.rpc.GwtRpcController.handleRequest(GwtRpcController.java:60) [classes!/:na]
    at org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter.handle(SimpleControllerHandlerAdapter.java:50) [spring-webmvc-4.1.6.RELEASE.jar!/:4.1.6.RELEASE]
    at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:959) [spring-webmvc-4.1.6.RELEASE.jar!/:4.1.6.RELEASE]
    at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:893) [spring-webmvc-4.1.6.RELEASE.jar!/:4.1.6.RELEASE]
    at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:966) [spring-webmvc-4.1.6.RELEASE.jar!/:4.1.6.RELEASE]
    at org.springframework.web.servlet.FrameworkServlet.doPost(FrameworkServlet.java:868) [spring-webmvc-4.1.6.RELEASE.jar!/:4.1.6.RELEASE]
    at javax.servlet.http.HttpServlet.service(HttpServlet.java:707) [javax.servlet-api-3.1.0.jar!/:3.1.0]
    at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:842) [spring-webmvc-4.1.6.RELEASE.jar!/:4.1.6.RELEASE]
    at javax.servlet.http.HttpServlet.service(HttpServlet.java:790) [javax.servlet-api-3.1.0.jar!/:3.1.0]
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:291) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:206) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:52) [tomcat-embed-websocket-8.0.20.jar!/:8.0.20]
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:239) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:206) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.springframework.security.web.authentication.logout.LogoutFilter.doFilter(LogoutFilter.java:110) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:239) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:206) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.springframework.security.web.authentication.AbstractAuthenticationProcessingFilter.doFilter(AbstractAuthenticationProcessingFilter.java:199) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:239) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:206) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.springframework.security.web.authentication.logout.LogoutFilter.doFilter(LogoutFilter.java:110) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:239) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:206) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.springframework.security.web.authentication.AbstractAuthenticationProcessingFilter.doFilter(AbstractAuthenticationProcessingFilter.java:199) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:239) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:206) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:330) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.access.intercept.FilterSecurityInterceptor.invoke(FilterSecurityInterceptor.java:118) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.access.intercept.FilterSecurityInterceptor.doFilter(FilterSecurityInterceptor.java:84) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.access.ExceptionTranslationFilter.doFilter(ExceptionTranslationFilter.java:113) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.session.SessionManagementFilter.doFilter(SessionManagementFilter.java:103) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.authentication.AnonymousAuthenticationFilter.doFilter(AnonymousAuthenticationFilter.java:113) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter.doFilter(SecurityContextHolderAwareRequestFilter.java:154) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.savedrequest.RequestCacheAwareFilter.doFilter(RequestCacheAwareFilter.java:45) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.iplantc.de.server.MDCFilter.doFilter(MDCFilter.java:63) [classes!/:na]
    at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.iplantc.de.server.CacheControlFilter.doFilter(CacheControlFilter.java:51) [classes!/:na]
    at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.authentication.AbstractAuthenticationProcessingFilter.doFilter(AbstractAuthenticationProcessingFilter.java:199) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.jasig.cas.client.session.SingleSignOutFilter.doFilter(SingleSignOutFilter.java:100) [cas-client-core-3.3.3.jar!/:3.3.3]
    at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.authentication.logout.LogoutFilter.doFilter(LogoutFilter.java:110) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.authentication.logout.LogoutFilter.doFilter(LogoutFilter.java:110) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.authentication.logout.LogoutFilter.doFilter(LogoutFilter.java:110) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.header.HeaderWriterFilter.doFilterInternal(HeaderWriterFilter.java:57) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107) [spring-web-4.1.6.RELEASE.jar!/:4.1.6.RELEASE]
    at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.context.SecurityContextPersistenceFilter.doFilter(SecurityContextPersistenceFilter.java:87) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter.doFilterInternal(WebAsyncManagerIntegrationFilter.java:50) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107) [spring-web-4.1.6.RELEASE.jar!/:4.1.6.RELEASE]
    at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.FilterChainProxy.doFilterInternal(FilterChainProxy.java:192) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.springframework.security.web.FilterChainProxy.doFilter(FilterChainProxy.java:160) [spring-security-web-3.2.7.RELEASE.jar!/:3.2.7.RELEASE]
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:239) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:206) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:85) [spring-web-4.1.6.RELEASE.jar!/:4.1.6.RELEASE]
    at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107) [spring-web-4.1.6.RELEASE.jar!/:4.1.6.RELEASE]
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:239) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:206) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:219) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:106) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:501) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:142) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:79) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:88) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:516) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.coyote.http11.AbstractHttp11Processor.process(AbstractHttp11Processor.java:1086) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.coyote.AbstractProtocol$AbstractConnectionHandler.process(AbstractProtocol.java:659) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.coyote.http11.Http11NioProtocol$Http11ConnectionHandler.process(Http11NioProtocol.java:223) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1558) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.run(NioEndpoint.java:1515) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142) [na:1.8.0_92-internal]
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617) [na:1.8.0_92-internal]
    at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61) [tomcat-embed-core-8.0.20.jar!/:8.0.20]
    at java.lang.Thread.run(Thread.java:745) [na:1.8.0_92-internal]
Caused by: java.io.FileNotFoundException: /etc/sobs/crypto/private-key.pem (No such file or directory)
    at java.io.FileInputStream.open0(Native Method) ~[na:1.8.0_92-internal]
    at java.io.FileInputStream.open(FileInputStream.java:195) ~[na:1.8.0_92-internal]
    at java.io.FileInputStream.<init>(FileInputStream.java:138) ~[na:1.8.0_92-internal]
    at java.io.FileInputStream.<init>(FileInputStream.java:93) ~[na:1.8.0_92-internal]
    at java.io.FileReader.<init>(FileReader.java:58) ~[na:1.8.0_92-internal]
    at org.iplantc.de.server.auth.PemKeyUtils.loadPrivateKey(PemKeyUtils.java:42) ~[de-lib-2.8.0.jar!/:na]
    at org.iplantc.de.server.auth.DefaultJwtBuilder.getPrivateKey(DefaultJwtBuilder.java:35) ~[de-lib-2.8.0.jar!/:na]
    ... 101 common frames omitted
```

The missing file path probably means that I used a different set of paths when I generated the config and data images
than I did when I launched the services. Logging into the DE container and checking the path was enough to confirm that
the path does not exist:

```
root# docker exec -it etc_de_ui_1 sh
container-root# ls -l /etc/sobs
ls: /etc/sobs: No such file or directory
```

Examining the Dockerfile revealed that I'd forgotten to declare the volumes:

``` dockerfile
FROM clojure:alpine

USER root
RUN \
  mkdir -p /etc/sobs/crypto/accepted_keys \
           /etc/sobs/crypto/sobs \
           /etc/ssl/sobs

#############################################
## GPG files -- ADD auto-extracts tarballs ##
#############################################
ADD sobs.tar.gz /etc/sobs/crypto/sobs

##################
## Signing keys ##
##################
COPY private-key.pem /etc/sobs/crypto/private-key.pem
COPY public-key.pem /etc/sobs/crypto/public-key.pem

############################
## SSL files for Logstash ##
############################

#################
## PERMISSIONS ##
#################
RUN \
  chmod 0600 /etc/sobs/crypto/private-key.pem && \
  chmod 0644 /etc/sobs/crypto/public-key.pem

CMD ["/bin/sh"]
```

Adding the volumes should fix the problem:

``` dockerfile
FROM clojure:alpine

USER root
RUN \
  mkdir -p /etc/sobs/crypto/accepted_keys \
           /etc/sobs/crypto/sobs \
           /etc/ssl/sobs

#############################################
## GPG files -- ADD auto-extracts tarballs ##
#############################################
ADD sobs.tar.gz /etc/sobs/crypto/sobs

##################
## Signing keys ##
##################
COPY private-key.pem /etc/sobs/crypto/private-key.pem
COPY public-key.pem /etc/sobs/crypto/public-key.pem

############################
## SSL files for Logstash ##
############################

#################
## PERMISSIONS ##
#################
RUN \
  chmod 0600 /etc/sobs/crypto/private-key.pem && \
  chmod 0644 /etc/sobs/crypto/public-key.pem

VOLUME ["/etc/ssl", "/etc/sobs/crypto"]
CMD ["/bin/sh"]
```

After regenerating the image and restarting the service, this error stopped occurring. I restarted the other services
that use a data container at the same time since they're likely to encounter similar errors.

The next error that occurred was a `Connection refused` error message when the UI was attempting to make a call to
terrain. This was the same firewall issue that prevented the UI from validating the CAS service ticket above. I updated
the firewall rules on the services host to fix this problem.

The next error that occurred was a failure to get notifications. I looked at the notification agent's log messages to
determine why this was happening. The AMQP connection was failing. There were two problems here. The first was that the
firewall was not allowing connections to the AMQP port. I updated the firewall rules on the server running RabbitMQ and
tried again. The error was still occurring, but for a different reason this time. After chasing several red herrings, I
finally figured out that the problem was caused by a password that wasn't being URL encoded before being placed in a
URL. I updated the group variables file to fix this and tried again. This time, the notification agent started
successfully.

## Troubleshooting RabbitMQ connection issues.

RabbitMQ was quickly running out of available file descriptors. I stopped all of the services on the host where most of
the connections were coming from, and started them up one-by-one hoping to determine which services were causing
problems. To my surprise, the number of open connections didn't dramatically increase after starting one or more of the
services. I'll have to keep an eye out for this problem.

## Troubleshooting DE data operations.

Errors were being displayed when I opened up the data Window in the DE. The first error that I encountered was a missing
path, but the exception didn't indicate which path was missing:

```
org.ixxxxxxxx.jargon.core.exception.FileNotFoundException: unable to find file under path
    at org.ixxxxxxxx.jargon.core.pub.CollectionListingUtils.handleNoObjStatUnderRootOrHomeByLookingForPublicAndHome(CollectionListingUtils.java:292)
    at org.ixxxxxxxx.jargon.core.pub.CollectionAndDataObjectListAndSearchAOImpl.retrieveObjectStatForPathWithHeuristicPathGuessing(CollectionAndDataObjectListAndSearchAOImpl.java:1589)
    at org.ixxxxxxxx.jargon.core.pub.IRODSFileSystemAOImpl.getObjStat(IRODSFileSystemAOImpl.java:465)
    at clj_jargon.item_info$jargon_type_check.invokeStatic(item_info.clj:69)
    at clj_jargon.item_info$jargon_type_check.invoke(item_info.clj:67)
    at clj_jargon.item_info$is_dir_QMARK_.invokeStatic(item_info.clj:81)
    at clj_jargon.item_info$is_dir_QMARK_.invoke(item_info.clj:77)
    at clj_jargon.permissions$is_readable_QMARK_.invokeStatic(permissions.clj:380)
    at clj_jargon.permissions$is_readable_QMARK_.invoke(permissions.clj:367)
    at data_info.util.validators$path_readable.invokeStatic(validators.clj:147)
    at data_info.util.validators$path_readable.invoke(validators.clj:145)
    at data_info.services.root$get_root.invokeStatic(root.clj:24)
    at data_info.services.root$get_root.invoke(root.clj:22)
    at data_info.services.root$root_listing.invokeStatic(root.clj:50)
    at data_info.services.root$root_listing.invoke(root.clj:40)
    at data_info.services.root$do_root_listing.invokeStatic(root.clj:57)
    at data_info.services.root$do_root_listing.invoke(root.clj:55)
    at clojure.lang.AFn.applyToHelper(AFn.java:154)
    at clojure.lang.AFn.applyTo(AFn.java:144)
    at clojure.core$apply.invokeStatic(core.clj:646)
    at clojure.core$apply.invoke(core.clj:641)
    at dire.core$supervised_meta.invokeStatic(core.clj:240)
    at dire.core$supervised_meta.doInvoke(core.clj:235)
    at clojure.lang.RestFn.invoke(RestFn.java:442)
    at clojure.core$partial$fn__4759.invoke(core.clj:2516)
    at clojure.lang.AFn.applyToHelper(AFn.java:156)
    at clojure.lang.RestFn.applyTo(RestFn.java:132)
    at clojure.core$apply.invokeStatic(core.clj:648)
    at clojure.core$apply.invoke(core.clj:641)
    at robert.hooke$compose_hooks$fn__12473.doInvoke(hooke.clj:40)
    at clojure.lang.RestFn.applyTo(RestFn.java:137)
    at clojure.core$apply.invokeStatic(core.clj:646)
    at clojure.core$apply.invoke(core.clj:641)
    at robert.hooke$run_hooks.invokeStatic(hooke.clj:46)
    at robert.hooke$run_hooks.invoke(hooke.clj:45)
    at robert.hooke$prepare_for_hooks$fn__12478$fn__12479.doInvoke(hooke.clj:54)
    at clojure.lang.RestFn.applyTo(RestFn.java:137)
    at clojure.lang.AFunction$1.doInvoke(AFunction.java:29)
    at clojure.lang.RestFn.applyTo(RestFn.java:137)
    at clojure.core$apply.invokeStatic(core.clj:646)
    at clojure.core$apply.invoke(core.clj:641)
    at data_info.util.service$trap$fn__14374.invoke(service.clj:34)
    at clojure.lang.AFn.applyToHelper(AFn.java:152)
    at clojure.lang.AFn.applyTo(AFn.java:144)
    at clojure.core$apply.invokeStatic(core.clj:646)
    at clojure.core$apply.invoke(core.clj:641)
    at clojure_commons.error_codes$trap.invokeStatic(error_codes.clj:167)
    at clojure_commons.error_codes$trap.doInvoke(error_codes.clj:165)
    at clojure.lang.RestFn.invoke(RestFn.java:425)
    at data_info.util.service$trap.invokeStatic(service.clj:34)
    at data_info.util.service$trap.doInvoke(service.clj:31)
    at clojure.lang.RestFn.invoke(RestFn.java:442)
    at data_info.routes.navigation$fn__15493$fn__15501.invoke(navigation.clj:46)
    at compojure.core$make_route$fn__6000.invoke(core.clj:135)
    at compojure.core$pre_init$fn__6039.invoke(core.clj:268)
    at compojure.api.coerce$body_coercer_middleware$fn__11141.invoke(coerce.clj:51)
    at compojure.core$pre_init$fn__6041$fn__6042.invoke(core.clj:271)
    at compojure.core$wrap_route_middleware$fn__5993.invoke(core.clj:121)
    at compojure.core$wrap_route_info$fn__5997.invoke(core.clj:126)
    at compojure.core$if_route$fn__5955.invoke(core.clj:45)
    at compojure.core$if_method$fn__5945.invoke(core.clj:27)
    at compojure.core$wrap_routes$fn__6047.invoke(core.clj:279)
    at compojure.api.routes.Route.invoke(routes.clj:74)
    at compojure.core$routing$fn__6007.invoke(core.clj:151)
    at clojure.core$some.invokeStatic(core.clj:2592)
    at clojure.core$some.invoke(core.clj:2583)
    at compojure.core$routing.invokeStatic(core.clj:151)
    at compojure.core$routing.doInvoke(core.clj:148)
    at clojure.lang.RestFn.applyTo(RestFn.java:139)
    at clojure.core$apply.invokeStatic(core.clj:648)
    at clojure.core$apply.invoke(core.clj:641)
    at compojure.core$routes$fn__6011.invoke(core.clj:156)
    at compojure.core$routing$fn__6007.invoke(core.clj:151)
    at clojure.core$some.invokeStatic(core.clj:2592)
    at clojure.core$some.invoke(core.clj:2583)
    at compojure.core$routing.invokeStatic(core.clj:151)
    at compojure.core$routing.doInvoke(core.clj:148)
    at clojure.lang.RestFn.invoke(RestFn.java:423)
    at data_info.routes.navigation$fn__15493.invokeStatic(navigation.clj:13)
    at data_info.routes.navigation$fn__15493.invoke(navigation.clj:13)
    at compojure.core$if_context$fn__6029.invoke(core.clj:220)
    at compojure.api.routes.Route.invoke(routes.clj:74)
    at compojure.api.core$handle$fn__11278.invoke(core.clj:8)
    at clojure.core$some.invokeStatic(core.clj:2592)
    at clojure.core$some.invoke(core.clj:2583)
    at compojure.api.core$handle.invokeStatic(core.clj:8)
    at compojure.api.core$handle.invoke(core.clj:7)
    at clojure.core$partial$fn__4759.invoke(core.clj:2515)
    at compojure.api.routes.Route.invoke(routes.clj:74)
    at compojure.api.core$handle$fn__11278.invoke(core.clj:8)
    at clojure.core$some.invokeStatic(core.clj:2592)
    at clojure.core$some.invoke(core.clj:2583)
    at compojure.api.core$handle.invokeStatic(core.clj:8)
    at compojure.api.core$handle.invoke(core.clj:7)
    at clojure.core$partial$fn__4759.invoke(core.clj:2515)
    at compojure.api.routes.Route.invoke(routes.clj:74)
    at service_logging.middleware$log_validation_errors$fn__11758.invoke(middleware.clj:120)
    at data_info.util$req_logger$fn__16226.invoke(util.clj:14)
    at compojure.api.middleware$wrap_exceptions$fn__10107.invoke(middleware.clj:43)
    at ring.middleware.keyword_params$wrap_keyword_params$fn__7659.invoke(keyword_params.clj:35)
    at clojure_commons.lcase_params$wrap_lcase_params$fn__5567.invoke(lcase_params.clj:18)
    at clojure_commons.query_params$wrap_query_params$fn__5586.invoke(query_params.clj:62)
    at service_logging.middleware$add_user_to_context$fn__11752.invoke(middleware.clj:112)
    at compojure.api.routes.Route.invoke(routes.clj:74)
    at compojure.api.core$handle$fn__11278.invoke(core.clj:8)
    at clojure.core$some.invokeStatic(core.clj:2592)
    at clojure.core$some.invoke(core.clj:2583)
    at compojure.api.core$handle.invokeStatic(core.clj:8)
    at compojure.api.core$handle.invoke(core.clj:7)
    at clojure.core$partial$fn__4759.invoke(core.clj:2515)
    at compojure.api.routes.Route.invoke(routes.clj:74)
    at ring.middleware.http_response$wrap_http_response$fn__7629.invoke(http_response.clj:8)
    at ring.swagger.middleware$wrap_swagger_data$fn__9788.invoke(middleware.clj:33)
    at compojure.api.middleware$wrap_options$fn__10117.invoke(middleware.clj:74)
    at ring.middleware.format_params$wrap_format_params$fn__6840.invoke(format_params.clj:118)
    at ring.middleware.format_params$wrap_format_params$fn__6840.invoke(format_params.clj:119)
    at ring.middleware.format_params$wrap_format_params$fn__6840.invoke(format_params.clj:119)
    at ring.middleware.format_params$wrap_format_params$fn__6840.invoke(format_params.clj:119)
    at ring.middleware.format_params$wrap_format_params$fn__6840.invoke(format_params.clj:119)
    at compojure.api.middleware$wrap_exceptions$fn__10107.invoke(middleware.clj:43)
    at ring.middleware.format_response$wrap_format_response$fn__7539.invoke(format_response.clj:183)
    at ring.middleware.keyword_params$wrap_keyword_params$fn__7659.invoke(keyword_params.clj:35)
    at ring.middleware.nested_params$wrap_nested_params$fn__7703.invoke(nested_params.clj:86)
    at ring.middleware.params$wrap_params$fn__7759.invoke(params.clj:64)
    at compojure.api.middleware$wrap_options$fn__10117.invoke(middleware.clj:74)
    at compojure.api.routes.Route.invoke(routes.clj:74)
    at ring.adapter.jetty$proxy_handler$fn__117.invoke(jetty.clj:24)
    at ring.adapter.jetty.proxy$org.eclipse.jetty.server.handler.AbstractHandler$ff19274a.handle(Unknown Source)
    at org.eclipse.jetty.server.handler.HandlerWrapper.handle(HandlerWrapper.java:97)
    at org.eclipse.jetty.server.Server.handle(Server.java:497)
    at org.eclipse.jetty.server.HttpChannel.handle(HttpChannel.java:310)
    at org.eclipse.jetty.server.HttpConnection.onFillable(HttpConnection.java:257)
    at org.eclipse.jetty.io.AbstractConnection$2.run(AbstractConnection.java:540)
    at org.eclipse.jetty.util.thread.QueuedThreadPool.runJob(QueuedThreadPool.java:635)
    at org.eclipse.jetty.util.thread.QueuedThreadPool$3.run(QueuedThreadPool.java:555)
    at java.lang.Thread.run(Thread.java:745)
```

Since this was happening when the data window was first being loaded, I suspected that it was one of the paths that the
data window initially displays when it's launched. After looking over the `data-info` configuration file, I suspected
that the missing path was the path to the shared data folder: `/sobs/home/shared`. After creating this folder and
reopening the data window, another error occurred:

```
clojure.lang.ExceptionInfo: throw+: {:error_code "ERR_NOT_READABLE", :path "/sobs/home", :user "dennis"} {:error_code "ERR_NOT_READABLE", :path "/sobs/home", :user "dennis"}
    at slingshot.support$stack_trace.invoke(support.clj:201)
    at data_info.util.validators$path_readable.invokeStatic(validators.clj:148)
    at data_info.util.validators$path_readable.invoke(validators.clj:145)
    at data_info.services.root$get_root.invokeStatic(root.clj:24)
    at data_info.services.root$get_root.invoke(root.clj:22)
    at data_info.services.root$root_listing.invokeStatic(root.clj:51)
    at data_info.services.root$root_listing.invoke(root.clj:40)
    at data_info.services.root$do_root_listing.invokeStatic(root.clj:57)
    at data_info.services.root$do_root_listing.invoke(root.clj:55)
    at clojure.lang.AFn.applyToHelper(AFn.java:154)
    at clojure.lang.AFn.applyTo(AFn.java:144)
    at clojure.core$apply.invokeStatic(core.clj:646)
    at clojure.core$apply.invoke(core.clj:641)
    at dire.core$supervised_meta.invokeStatic(core.clj:240)
    at dire.core$supervised_meta.doInvoke(core.clj:235)
    at clojure.lang.RestFn.invoke(RestFn.java:442)
    at clojure.core$partial$fn__4759.invoke(core.clj:2516)
    at clojure.lang.AFn.applyToHelper(AFn.java:156)
    at clojure.lang.RestFn.applyTo(RestFn.java:132)
    at clojure.core$apply.invokeStatic(core.clj:648)
    at clojure.core$apply.invoke(core.clj:641)
    at robert.hooke$compose_hooks$fn__12473.doInvoke(hooke.clj:40)
    at clojure.lang.RestFn.applyTo(RestFn.java:137)
    at clojure.core$apply.invokeStatic(core.clj:646)
    at clojure.core$apply.invoke(core.clj:641)
    at robert.hooke$run_hooks.invokeStatic(hooke.clj:46)
    at robert.hooke$run_hooks.invoke(hooke.clj:45)
    at robert.hooke$prepare_for_hooks$fn__12478$fn__12479.doInvoke(hooke.clj:54)
    at clojure.lang.RestFn.applyTo(RestFn.java:137)
    at clojure.lang.AFunction$1.doInvoke(AFunction.java:29)
    at clojure.lang.RestFn.applyTo(RestFn.java:137)
    at clojure.core$apply.invokeStatic(core.clj:646)
    at clojure.core$apply.invoke(core.clj:641)
    at data_info.util.service$trap$fn__14374.invoke(service.clj:34)
    at clojure.lang.AFn.applyToHelper(AFn.java:152)
    at clojure.lang.AFn.applyTo(AFn.java:144)
    at clojure.core$apply.invokeStatic(core.clj:646)
    at clojure.core$apply.invoke(core.clj:641)
    at clojure_commons.error_codes$trap.invokeStatic(error_codes.clj:167)
    at clojure_commons.error_codes$trap.doInvoke(error_codes.clj:165)
    at clojure.lang.RestFn.invoke(RestFn.java:425)
    at data_info.util.service$trap.invokeStatic(service.clj:34)
    at data_info.util.service$trap.doInvoke(service.clj:31)
    at clojure.lang.RestFn.invoke(RestFn.java:442)
    at data_info.routes.navigation$fn__15493$fn__15501.invoke(navigation.clj:46)
    at compojure.core$make_route$fn__6000.invoke(core.clj:135)
    at compojure.core$pre_init$fn__6039.invoke(core.clj:268)
    at compojure.api.coerce$body_coercer_middleware$fn__11141.invoke(coerce.clj:51)
    at compojure.core$pre_init$fn__6041$fn__6042.invoke(core.clj:271)
    at compojure.core$wrap_route_middleware$fn__5993.invoke(core.clj:121)
    at compojure.core$wrap_route_info$fn__5997.invoke(core.clj:126)
    at compojure.core$if_route$fn__5955.invoke(core.clj:45)
    at compojure.core$if_method$fn__5945.invoke(core.clj:27)
    at compojure.core$wrap_routes$fn__6047.invoke(core.clj:279)
    at compojure.api.routes.Route.invoke(routes.clj:74)
    at compojure.core$routing$fn__6007.invoke(core.clj:151)
    at clojure.core$some.invokeStatic(core.clj:2592)
    at clojure.core$some.invoke(core.clj:2583)
    at compojure.core$routing.invokeStatic(core.clj:151)
    at compojure.core$routing.doInvoke(core.clj:148)
    at clojure.lang.RestFn.applyTo(RestFn.java:139)
    at clojure.core$apply.invokeStatic(core.clj:648)
    at clojure.core$apply.invoke(core.clj:641)
    at compojure.core$routes$fn__6011.invoke(core.clj:156)
    at compojure.core$routing$fn__6007.invoke(core.clj:151)
    at clojure.core$some.invokeStatic(core.clj:2592)
    at clojure.core$some.invoke(core.clj:2583)
    at compojure.core$routing.invokeStatic(core.clj:151)
    at compojure.core$routing.doInvoke(core.clj:148)
    at clojure.lang.RestFn.invoke(RestFn.java:423)
    at data_info.routes.navigation$fn__15493.invokeStatic(navigation.clj:13)
    at data_info.routes.navigation$fn__15493.invoke(navigation.clj:13)
    at compojure.core$if_context$fn__6029.invoke(core.clj:220)
    at compojure.api.routes.Route.invoke(routes.clj:74)
    at compojure.api.core$handle$fn__11278.invoke(core.clj:8)
    at clojure.core$some.invokeStatic(core.clj:2592)
    at clojure.core$some.invoke(core.clj:2583)
    at compojure.api.core$handle.invokeStatic(core.clj:8)
    at compojure.api.core$handle.invoke(core.clj:7)
    at clojure.core$partial$fn__4759.invoke(core.clj:2515)
    at compojure.api.routes.Route.invoke(routes.clj:74)
    at compojure.api.core$handle$fn__11278.invoke(core.clj:8)
    at clojure.core$some.invokeStatic(core.clj:2592)
    at clojure.core$some.invoke(core.clj:2583)
    at compojure.api.core$handle.invokeStatic(core.clj:8)
    at compojure.api.core$handle.invoke(core.clj:7)
    at clojure.core$partial$fn__4759.invoke(core.clj:2515)
    at compojure.api.routes.Route.invoke(routes.clj:74)
    at service_logging.middleware$log_validation_errors$fn__11758.invoke(middleware.clj:120)
    at data_info.util$req_logger$fn__16226.invoke(util.clj:14)
    at compojure.api.middleware$wrap_exceptions$fn__10107.invoke(middleware.clj:43)
    at ring.middleware.keyword_params$wrap_keyword_params$fn__7659.invoke(keyword_params.clj:35)
    at clojure_commons.lcase_params$wrap_lcase_params$fn__5567.invoke(lcase_params.clj:18)
    at clojure_commons.query_params$wrap_query_params$fn__5586.invoke(query_params.clj:62)
    at service_logging.middleware$add_user_to_context$fn__11752.invoke(middleware.clj:112)
    at compojure.api.routes.Route.invoke(routes.clj:74)
    at compojure.api.core$handle$fn__11278.invoke(core.clj:8)
    at clojure.core$some.invokeStatic(core.clj:2592)
    at clojure.core$some.invoke(core.clj:2583)
    at compojure.api.core$handle.invokeStatic(core.clj:8)
    at compojure.api.core$handle.invoke(core.clj:7)
    at clojure.core$partial$fn__4759.invoke(core.clj:2515)
    at compojure.api.routes.Route.invoke(routes.clj:74)
    at ring.middleware.http_response$wrap_http_response$fn__7629.invoke(http_response.clj:8)
    at ring.swagger.middleware$wrap_swagger_data$fn__9788.invoke(middleware.clj:33)
    at compojure.api.middleware$wrap_options$fn__10117.invoke(middleware.clj:74)
    at ring.middleware.format_params$wrap_format_params$fn__6840.invoke(format_params.clj:118)
    at ring.middleware.format_params$wrap_format_params$fn__6840.invoke(format_params.clj:119)
    at ring.middleware.format_params$wrap_format_params$fn__6840.invoke(format_params.clj:119)
    at ring.middleware.format_params$wrap_format_params$fn__6840.invoke(format_params.clj:119)
    at ring.middleware.format_params$wrap_format_params$fn__6840.invoke(format_params.clj:119)
    at compojure.api.middleware$wrap_exceptions$fn__10107.invoke(middleware.clj:43)
    at ring.middleware.format_response$wrap_format_response$fn__7539.invoke(format_response.clj:183)
    at ring.middleware.keyword_params$wrap_keyword_params$fn__7659.invoke(keyword_params.clj:35)
    at ring.middleware.nested_params$wrap_nested_params$fn__7703.invoke(nested_params.clj:86)
    at ring.middleware.params$wrap_params$fn__7759.invoke(params.clj:64)
    at compojure.api.middleware$wrap_options$fn__10117.invoke(middleware.clj:74)
    at compojure.api.routes.Route.invoke(routes.clj:74)
    at ring.adapter.jetty$proxy_handler$fn__117.invoke(jetty.clj:24)
    at ring.adapter.jetty.proxy$org.eclipse.jetty.server.handler.AbstractHandler$ff19274a.handle(Unknown Source)
    at org.eclipse.jetty.server.handler.HandlerWrapper.handle(HandlerWrapper.java:97)
    at org.eclipse.jetty.server.Server.handle(Server.java:497)
    at org.eclipse.jetty.server.HttpChannel.handle(HttpChannel.java:310)
    at org.eclipse.jetty.server.HttpConnection.onFillable(HttpConnection.java:257)
    at org.eclipse.jetty.io.AbstractConnection$2.run(AbstractConnection.java:540)
    at org.eclipse.jetty.util.thread.QueuedThreadPool.runJob(QueuedThreadPool.java:635)
    at org.eclipse.jetty.util.thread.QueuedThreadPool$3.run(QueuedThreadPool.java:555)
    at java.lang.Thread.run(Thread.java:745)
```

This error message was a little bit easier to decipher. After giving the `public` group read access to `/sobs/home`, I
was able to open the data window without error.

My next step was to create some files and verify that the info types were updated correctly. I was able to create files
without any problems, but info types weren't being assigned automatically. This error was showing up in the iRODS log
files again:

```
Jan 23 16:42:39 pid:9262 NOTICE: writeLine: inString = Failed to send AMQP message: ERROR: (406, "PRECONDITION_FAILED - cannot redeclare exchange 'irods' in vhost '/sobs/data-store' with different type, durable, internal or autodelete value")
```

The exchange had once again been declared as durable, which is causing the error. I suspect that the exchange is
actually being regenerated by one of the DE services:

```
root# rmq -V /sobs/data-store list exchanges
+------------------+--------------------+---------+-------------+---------+----------+
|      vhost       |        name        |  type   | auto_delete | durable | internal |
+------------------+--------------------+---------+-------------+---------+----------+
| /sobs/data-store |                    | direct  | False       | True    | False    |
| /sobs/data-store | amq.direct         | direct  | False       | True    | False    |
| /sobs/data-store | amq.fanout         | fanout  | False       | True    | False    |
| /sobs/data-store | amq.headers        | headers | False       | True    | False    |
| /sobs/data-store | amq.match          | headers | False       | True    | False    |
| /sobs/data-store | amq.rabbitmq.trace | topic   | False       | True    | True     |
| /sobs/data-store | amq.topic          | topic   | False       | True    | False    |
| /sobs/data-store | irods              | topic   | False       | True    | False    |
+------------------+--------------------+---------+-------------+---------+----------+
```

I stopped the DE services, deleted the `irods` exchange, created a new file and checked the exchange listing again:

```
# rmq -V /sobs/data-store list exchanges
+------------------+--------------------+---------+-------------+---------+----------+
|      vhost       |        name        |  type   | auto_delete | durable | internal |
+------------------+--------------------+---------+-------------+---------+----------+
| /sobs/data-store |                    | direct  | False       | True    | False    |
| /sobs/data-store | amq.direct         | direct  | False       | True    | False    |
| /sobs/data-store | amq.fanout         | fanout  | False       | True    | False    |
| /sobs/data-store | amq.headers        | headers | False       | True    | False    |
| /sobs/data-store | amq.match          | headers | False       | True    | False    |
| /sobs/data-store | amq.rabbitmq.trace | topic   | False       | True    | True     |
| /sobs/data-store | amq.topic          | topic   | False       | True    | False    |
| /sobs/data-store | irods              | topic   | True        | False   | False    |
+------------------+--------------------+---------+-------------+---------+----------+
```

This time, the exchange was declared the way that iRODS expects. Restarted the DE services and checked the exchange
listing again. Interestingly enough, this caused the number of connections to RabbitMQ to explode. After examining the
configuration on another one of our servers, I found that the problem was in `/etc/irods/ipc-env.re`. The value of
`ipc_AMQP_EPHEMERAL` was `true` when it should have been `false`. After fixing the file and forcing an AMQP message to
be sent, it appears that the exchange was declared correctly:

```
# rmq -V /sobs/data-store list exchanges
+------------------+--------------------+---------+-------------+---------+----------+
|      vhost       |        name        |  type   | auto_delete | durable | internal |
+------------------+--------------------+---------+-------------+---------+----------+
| /sobs/data-store |                    | direct  | False       | True    | False    |
| /sobs/data-store | amq.direct         | direct  | False       | True    | False    |
| /sobs/data-store | amq.fanout         | fanout  | False       | True    | False    |
| /sobs/data-store | amq.headers        | headers | False       | True    | False    |
| /sobs/data-store | amq.match          | headers | False       | True    | False    |
| /sobs/data-store | amq.rabbitmq.trace | topic   | False       | True    | True     |
| /sobs/data-store | amq.topic          | topic   | False       | True    | False    |
| /sobs/data-store | irods              | topic   | False       | True    | False    |
+------------------+--------------------+---------+-------------+---------+----------+
```

I restarted the DE services then tested info type detection again. This time automatic file detection worked
correctly. I briefly tested URL uploads. They weren't working, but I decided to table this problem. URL uploads are so
closely related to job submissions that I wanted to verify that job submissions were working before digging into this
too much.

I tried anonymous file sharing next. The link appears to have been created correctly, but the connection to the
anon-files service was refused. The first problem was that an HTTP URL was generated, but there was no reverse proxy for
the HTTP URL. I fixed this by creating some additional redirects:

```
Redirect permanent /de https://sobs-de.sobs.arizona.edu/de
Redirect permanent /cas https://sobs-de.sobs.arizona.edu/cas
Redirect permanent /anon-files https://sobs-de.sobs.arizona.edu/anon-files
Redirect permanent /dl https://sobs-de.sobs.arizona.edu/dl
```

I was still getting an error when I tried the URL again. This time it was because a firewall was blocking the port and
the reverse proxy was configured incorrectly. After fixing these settings, the download links started working correctly.
It was actually the reverse proxy setting that weas preventing the URL from working in this case, but the firewall would
have caused issues for `anon-files`.

The next step was to try to share a file with a collaborator. There was an error when I tried to do the first
collaborator search, which was caused by an incorrectly configured reverse proxy. After fixing the reverse proxy, the
collaborator search worked correctly. I ran into a bit of a problem when I tried to share with a collaborator at first,
but it turned out that these errors occurred because the user that I picked didn't have an iRODS account yet. Also,
while I was testing this, I noticed that I wasn't receiving notifications. It turned out that this was also an
incorrectly configured reverse proxy. The differences between Apache and nginx are just enough to cause confusion.

The next feature to test was the ability to share files with genome browsers via `anon-files`. The first time I tried
this, I got `ERR_NOT_A_USER`. This was because the `anonymous` user hadn't been created in iRODS. After creating the
user, this feature worked correctly as well.
