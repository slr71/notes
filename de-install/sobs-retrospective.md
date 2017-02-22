# Agenda

- [Overview](#overview)
- [Differences](#differences)
- [Database](#database)
- [Condor](#condor)
- [RabbitMQ](#rabbitmq)
- [iRODS](#irods)
  - [iRODS Rules](#irods-rules)
- [ElasticSearch](#elasticsearch)
- [Docker](#docker)
- [Docker Compose](#docker-compose)
- [Docker Registry](#docker-registry)
- [Data Container](#data-container)
- [Group Variables](#group-variables)
- [Reverse Proxies](#reverse-proxies)
- [CAS](#cas)
- [Grouper](#grouper)
- [DE](#de)
- [Troubleshooting](#troubleshooting)
- [Miscellanea](#miscellanea)
- [Annoyances](#annoyances)

# Overview

- Deploying the DE was fairly painless, as expected.
- Deploying the rest of the components was hard, as expected.

# Differences

- HTTPD
  - Already installed.
  - Used it for reverse proxies in case they had other plans for it.
- LetsEncrypt
  - SSL certificates renewed periodically via scripts.
  - Need to refresh some services when certificates are renewed.
- Grouper
  - No separate Grouper host.
- CAS
  - Runs on DE host.
- Consul
  - Not installed.
  - Zero-downtime was not a requirement for this deployment.

# Database

- Accounts and databases
- `postgresql.conf`
- `pg_hbas.conf`
- Firewall

# Condor

- Version 8.4.9, which was the latest at the time.
- Most of this was already done.
- Submit host (install, copy configs, tweak).

# RabbitMQ

- Playbook was sufficient for installation.
- Manual configuration.
- Manual virtual host creation.
- Manual `rabbitmqadmin` utility installation.
- Manual exchange declaration.


# iRODS

- iRODS has RPMs now!
- Usual firewall shenanigans.
- Additional database privileges for `icat_reader` account.

## iRODS Rules

- Playbook
- Manually fixed configuration files.
- Problems accessing RabbitMQ.

# ElasticSearch

- Java
- ElasticSearch RPMs
- Manual configuration.
  - Main ElasticSearch configuration.
  - JVM options.
- Firewall

# Docker

- YUM version lock plugin
- Docker YUM repository
  - Created locally
  - Copied to hosts using an Ansible ad-hoc command
- Update YUM cache and install.
- Enable and start service.

# Docker Compose

- Needed a more granular playbook

# Docker Registry

- Copied the setup on gims
- Defined and enabled the services.
- Problems
  - Reverse proxy and docker registry on different hosts didn't work.

# Data Container

- Mostly manual
- Modeled after our data container setup.

# Group Variables

- Cheated a bit.
- Much easier for us than for most people.

# Reverse Proxies

- Manual configuration.
- More flexibility because of manual configuration.

# CAS

- Copied and tweaked our `cas-overlay` repo.
- Used Docker for deployment - best thing ever.
- LDAPS actually fairly easy to set up.
- SSL needed in servlet container.

# Grouper

- Configuration container
  - Group vars
  - Build image
- Compose file
- LDAPS
  - Wasted time
  - Finally gave up and went with plain-text LDAP
- Database initialization

# DE

- Compose file
  - Many tiny changes
  - Deployed as usual.
- UI and Services
  - Deployed as usual.

# Troubleshooting

- Rules
  - Missing scripts and plugin.
  - Exchange being redeclared differently.
- CAS
  - Service ticket validation failure - Firewall
  - Borked redirections - HTTPD Redirect
- DE
  - JWT signing key error - volumes not declared in container
  - jargon.core.exception.FileNotFoundException - no shared data folder
  - ERR_NOT_REQADABLE for `/sobs/home` - incorrect iRODS permissions
  - PSQLException: syntax error near NULL - App permissions not initialized
  - jex-adapter: unknown error - no container name for JEX adapter
  - All jobs being held - Condor negotiator blocked by firewall
  - All jobs failing with shadow exception - permissions on `/opt/image-janitor`
  - Request validation failure for metadata templates - missing value types in database.
  - App publication failure - unconfigurable shared directory path in `de.properties` template.
  - Pipelines: redirection from `stdout` and `stderr` failing - workaround
- RabbitMQ
  - Running out of file descriptors - irods rule configuration
- Belphegor
  - Granted authority textual representation required - bracket shenanigans/DE code change


# Miscellanea

- Ad-hoc commands for `/etc/timezone` and `/etc/localtime`

# Annoyances

- Stopping and restarting services.
- Firewalls
