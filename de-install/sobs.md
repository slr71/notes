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
