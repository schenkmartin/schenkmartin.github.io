---
layout: post
title:  "Running various PostgreSQL versions on a single CentOS/RedHat 8 host"
date:   2021-06-07 12:14:00 +0200
categories: postgresql installation
---

Running multiple versions of PostgreSQL server on one system can be handy for development, upgrade testing, or even production use. In this guide I'll setup different versions of the open source database on a single host, incorporating automatic system startup and a separate file system location for the most important files.<!--break-->

The database server is configured using a minimal memory footprint. No fancy things like replication or archiving will be configured. Only some extended logging is configured, so we get a better view on the things happening on our system.

Someone might object that applying a tight security to the database logins removes the ease of use. This can me mitigated by using a `pgpass`file.

I used the current software packages as I wrote this manual. There will be new version, but all installation steps should apply to future version without need to adapt.

This guide covers the following PostgreSQL versions:

* [9.6](#96)
* [12.5](#125)
* [13.1](#131)

---

## 9.6

### Packages source

Use the packages provided by the official PostgreSQL team - located at https://download.postgresql.org/pub/repos/yum/9.6/redhat/rhel-8-x86_64/

- [postgresql96-9.6.22-1PGDG.rhel8.x86_64.rpm](https://download.postgresql.org/pub/repos/yum/9.6/redhat/rhel-8-x86_64/postgresql96-9.6.22-1PGDG.rhel8.x86_64.rpm)
- [postgresql96-contrib-9.6.22-1PGDG.rhel8.x86_64.rpm](https://download.postgresql.org/pub/repos/yum/9.6/redhat/rhel-8-x86_64/postgresql96-contrib-9.6.22-1PGDG.rhel8.x86_64.rpm)
- [postgresql96-libs-9.6.22-1PGDG.rhel8.x86_64.rpm](https://download.postgresql.org/pub/repos/yum/9.6/redhat/rhel-8-x86_64/postgresql96-libs-9.6.22-1PGDG.rhel8.x86_64.rpm)
- [postgresql96-server-9.6.22-1PGDG.rhel8.x86_64.rpm](https://download.postgresql.org/pub/repos/yum/9.6/redhat/rhel-8-x86_64/postgresql96-server-9.6.22-1PGDG.rhel8.x86_64.rpm)

### Package installation

Download all packages to a directory. From this directory, install the packages using YUM. Dependencies are automatically installed during the process.

```
yum install postgresql96*.rpm
```

### OS user "postgres"

During package installation, the operating system user `postgres`will also be created. 

We want to recreate it with custom settings, so it is usable as administrative account for the database server:

```bash
## remove the user created by package installation
userdel -r postgres

## add the group "postgres"
groupadd postgres

## create the user
useradd -m -g postgres -c "PostgreSQL server" -s /bin/bash -d /home/postgres postgres
passwd postgres

## adapt permissions to the new user ID
chown postgres:postgres /var/run/postgresql
```



### Directory structure for the database cluster

- /postgres/9.6 - base directory for all database files
  - /postgres/9.6/data - the DATA directory of the database cluster
  - /postgres/9.6/log - logfiles are stored here

```bash
mkdir -p /postgres/9.6/data
mkdir -p /postgres/9.6/log
mkdir -p /postgres/9.6/wal
```

### Database initialization

Now it's time to actually create a PostgreSQL database cluster:

```bash
/usr/pgsql-9.6/bin/initdb -A md5 -D /postgres/9.6/data --pwprompt --data-checksums
```

Here is a short explanation of the parameters used:

- all users must authenticate with password `-A md5`
- Data directory will be created at `-D /postgres/9.6/data`
- The password for the database super-user `postgres` must be entered during initialization `--pwpromt`
- Detect I/O system corruptions with `--data-checksums`

### postgresql.conf

A minimal configuration file at `/postgres/9.6/data/postgresql.conf`replaces the default configuration file.

Note that we do not define the TCP port in postgresql.conf, since it is provided by the systemd customization file.

```
## the bare minimum required to start server
data_directory = '/postgres/9.6/data'
listen_addresses = '*'

## logging
log_destination = 'stderr'
logging_collector = on
log_directory = '/postgres/9.6/log/'
log_filename = 'postgresql-%Y%m%d.log'
log_line_prefix = '%m [%p]: [%l-line] user=%u,db=%d,remote=%h,app=%a '
log_file_mode = 0600
log_truncate_on_rotation = on
log_rotation_age = 1d
log_rotation_size = 0
log_checkpoints = on
log_connections = on
log_disconnections = on

# performance logging
log_lock_waits = on
log_min_duration_statement = 60000  # duration in milliseconds  -1=disabled 0=log all

## memory 
shared_buffers = 128MB
huge_pages = try
work_mem = 8MB
maintenance_work_mem = 8MB
autovacuum_work_mem = 8MB

fsync = on
full_page_writes = on

## Checkpoints
checkpoint_timeout = 15min
```

### Server startup and shutdown

We are now able to start the database server, giving it a manual try. So we can see if everything works as expected:

```bash
## Start
[postgres@dbmachine01 ~]$ /usr/pgsql-9.6/bin/pg_ctl start -D /postgres/9.6/data/
server starting
2020-11-24 14:48:06.112 GMT [5147]: [1-line] user=,db=,remote=,app= LOG:  redirecting log output to logging collector process
2020-11-24 14:48:06.112 GMT [5147]: [2-line] user=,db=,remote=,app= HINT:  Future log output will appear in directory "/postgres/9.6/log".
```

Shutdown the database server:

```bash
## Stop
[postgres@dbmachine01 log]$ /usr/pgsql-9.6/bin/pg_ctl stop -D /postgres/9.6/data/
waiting for server to shut down.... done
server stopped
```

Finishing this short test will make sure everything is set for the next step - automatic startup and shutdown.

#### Automatic startup and shutdown

RPM packages also provide startup and shutdown script for the systemd facility. These files also allow customization of the database location and network port. The customization file may be owned by the `postgres`OS user to allow a little bit customization during the startup process.

- Service control file - do not edit this file: `/usr/lib/systemd/system/postgresql-9.6.service`

- Customization file for systemd: `/etc/systemd/system/postgresql-9.6.service`
  Example content - set a custom TCP port and data directory

  ```
  .include /usr/lib/systemd/system/postgresql-9.6.service
  
  [Service]
  Environment=PGPORT=5433
  Environment=PGDATA=/postgres/9.6/data
  ```

Now we can control our database server using `systemctl`:

```bash
## Start
[root@dbmachine01 system]$ systemctl start postgresql-9.6

## Status
[root@dbmachine01 system]# systemctl status postgresql-9.6
● postgresql-9.6.service - PostgreSQL 9.6 database server
   Loaded: loaded (/etc/systemd/system/postgresql-9.6.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2021-01-13 11:33:17 CET; 6s ago
     Docs: https://www.postgresql.org/docs/9.6/static/
  Process: 77260 ExecStartPre=/usr/pgsql-9.6/bin/postgresql96-check-db-dir ${PGDATA} (code=exited, status=0/SUCCESS)
 Main PID: 77266 (postmaster)
    Tasks: 7 (limit: 25005)
   Memory: 9.8M
   CGroup: /system.slice/postgresql-9.6.service
           ├─77266 /usr/pgsql-9.6/bin/postmaster -D /postgres/9.6/data
           ├─77267 postgres: logger process
           ├─77269 postgres: checkpointer process
           ├─77270 postgres: writer process
           ├─77271 postgres: wal writer process
           ├─77272 postgres: autovacuum launcher process
           └─77273 postgres: stats collector process

## Stop
[root@dbmachine01 system]$ systemctl stop postgresql-9.6
```

Now enable (or disable, if needed) automatic service start at system boot:

```bash
### Enable automatic startup and shutdown
[root@dbmachine01 system]$ systemctl enable postgresql-9.6
Created symlink /etc/systemd/system/multi-user.target.wants/postgresql-9.6.service → /etc/systemd/system/postgresql-9.6.service.

## Disable automatic startup and shutdown
[root@dbmachine01 system]$ systemctl disable postgresql-9.6
Removed /etc/systemd/system/multi-user.target.wants/postgresql-9.6.service.
```

### A shell profile makes life easier

I use to create a shell profile, so I can switch between different installations of PostgreSQL easily.

The file is named `/home/postgres/pg96_profile` and contains settings for the database server's base directory and network port. Additionally it modifies the search path of our shell to make sure we are using the correct binaries. The last line modifies the shell prompt `PS1`.

```bash
export PGDATA=/postgres/9/data
export PGPORT=5433
export PG_HOME=/usr/pgsql-9.6

export PATH=$PG_HOME/bin:$PATH

PS1="PG9.6 - [\u@\h \W]\$ "
```

You can activate the settings by sourcing the file on the shell prompt: ```source ~/pg96_profile```

### Connecting with psql

When connecting to the database server with `psql`you can either use the easy way, by sourcing the shell profile:

```bash
PG12 - [postgres@dbmachine01 ~]$ source pg96_profile
PG12 - [postgres@dbmachine01 ~]$ psql
psql (9.6)
Type "help" for help.

postgres=#
```

Or you may specify the connection data as parameters. In our case it is sufficient to specify the network port (`-p 5433`), since we attach to a database server at the local host:

```bash
[postgres@dbmachine01 data]$ /usr/pgsql-9.6/bin/psql -p 5433
```

Enter the password and you are logged in. If you want to be logged in without entering the password, read on.

#### Using a password file to automatically login with psql

For many reasons, we might want to login without typing the password each time. For testing and development purposes, upgrading the database cluster, or to simplify usage of some of the system tools provided by PostgreSQL (e.g. pg_dumpall).

`psql`therefore supports reading authentication data from the file `~/.pgpass`. This file can contain multiple entries for different targets, but for my usage pattern I only have one entry. It will provide the password of the `postgres`database user no mater on which database on the local host at port 5433 I connect:

```
localhost:5433:*:postgres:secret-password
```

Entries are formatted in this way: *hostname:port:database:username:password*. Instead of a specific database, the asterisk `*` acts as wildcard for whichever database to connect to.

The file must be only read- and writeable by the owner, so apply a `chmod 600 ~/.pgpass`, or it will not be used by `psql`!



---

## 12.5

### Package sources

Use the RPM packages provided by the official PostgreSQL team - located at https://download.postgresql.org/pub/repos/yum/12/redhat/rhel-8-x86_64/. 

* https://download.postgresql.org/pub/repos/yum/12/redhat/rhel-8-x86_64/postgresql12-12.5-1PGDG.rhel8.x86_64.rpm
* https://download.postgresql.org/pub/repos/yum/12/redhat/rhel-8-x86_64/postgresql12-contrib-12.5-1PGDG.rhel8.x86_64.rpm
* https://download.postgresql.org/pub/repos/yum/12/redhat/rhel-8-x86_64/postgresql12-libs-12.5-1PGDG.rhel8.x86_64.rpm
* https://download.postgresql.org/pub/repos/yum/12/redhat/rhel-8-x86_64/postgresql12-server-12.5-1PGDG.rhel8.x86_64.rpm

### Package installation

Download all packages to a directory. From this directory, install the packages using YUM. Dependencies are automatically installed during the process.

```bash
yum install postgresql12*.rpm
```

### OS user "postgres"

During package installation, the operating system user `postgres`will also be created. 

We want to recreate it with custom settings, so it is usable as administrative account for the database server:

```bash
## remove the user created by package installation
userdel -r postgres

## add the group "postgres"
groupadd postgres

## create the user
useradd -m -g postgres -c "PostgreSQL server" -s /bin/bash -d /home/postgres postgres
passwd postgres

## adapt permissions to the new user ID
chown postgres:postgres /var/run/postgresql
```

### Directory structure for the database cluster

- /postgres/12 - base directory for all database files
  - /postgres/12/data - the DATA directory of the database cluster
  - /postgres/12/log - logfiles are stored here

```bash
mkdir -p /postgres/12/data
mkdir -p /postgres/12/log
mkdir -p /postgres/12/wal
```

### Database initialization

Now it's time to actually create a PostgreSQL database cluster:

```bash
/usr/pgsql-12/bin/initdb -A scram-sha-256 -D /postgres/12/data --pwprompt --data-checksums
```

Here is a short explanation of the parameters used:

- all users must authenticate with password, using the SCRAM-SHA-256 authentication  `-A scram-sha-256`
- Data directory will be created at `-D /postgres/12/data`
- The password for the database super-user `postgres` must be entered during initialization `--pwpromt`
- Detect I/O system corruptions with `--data-checksums`

### postgresql.conf 

Since we do not use the defaults that come with the package installation, it is necessary to edit the main configuration file `postgresql.conf`. It is located in the data directory.

But instead of cumbersome editing, I'm using a minimal configuration file. If different settings are needed, they can be changed or added any time later.

Here is my initial `/postgres/12/data/postgresql.conf`file: 

```
## the bare minimum required to start server
data_directory = '/postgres/12/data'
listen_addresses = '*'

## logging
log_destination = 'stderr'
logging_collector = on
log_directory = '/postgres/12/log/'
log_filename = 'postgresql-%Y%m%d.log'
log_line_prefix = '%m [%p]: [%l-line] user=%u,db=%d,remote=%h,app=%a '
log_file_mode = 0600
log_truncate_on_rotation = on
log_rotation_age = 1d
log_rotation_size = 0
log_checkpoints = on
log_connections = on
log_disconnections = on

# performance logging
log_lock_waits = on
log_min_duration_statement = 60000  # duration in milliseconds  -1=disabled 0=log all

## memory 
shared_buffers = 128MB
huge_pages = try
work_mem = 8MB
maintenance_work_mem = 8MB
autovacuum_work_mem = 8MB

fsync = on
full_page_writes = on

## Checkpoints
checkpoint_timeout = 15min
```

### Server startup and shutdown

We are now able to start the database server, giving it a manual try. So we can see if everything works as expected:

```bash
## Start
[postgres@dbmachine01 ~]$ /usr/pgsql-12/bin/pg_ctl start -D /postgres/12/data/
waiting for server to start....2020-11-24 14:48:06.112 GMT [5147]: [1-line] user=,db=,remote=,app= LOG:  redirecting log output to logging collector process
2020-11-24 14:48:06.112 GMT [5147]: [2-line] user=,db=,remote=,app= HINT:  Future log output will appear in directory "/postgres/12/log".
```

Shutdown the database server:

```bash
## Stop

[postgres@dbmachine01 log]$ /usr/pgsql-12/bin/pg_ctl stop -D /postgres/12/data/
waiting for server to shut down.... done
server stopped
```

Finishing this short test will make sure everything is set for the next step - automatic startup and shutdown.

#### Automatic startup and shutdown

RPM packages also provide startup and shutdown script for the systemd facility. These files also allow customization of the database location and network port. The customization file may be owned by the `postgres`OS user to allow a little bit customization during the startup process.

- Service control file - do not edit this file: `/usr/lib/systemd/system/postgresql-12.service`

- Customization file for systemd: `/etc/systemd/system/postgresql-12.service`

  Create this file (as `root`user) and add the following content:

  ```
  .include /usr/lib/systemd/system/postgresql-12.service
  
  [Service]
  Environment=PGDATA=/postgres/12/data
  Environment=PGPORT=5443
  ```

Now we can control our database server using `systemctl`:

```bash
## Start
[root@dbmachine01 system]$ systemctl start postgresql-12

## Status
[root@dbmachine01 system]# systemctl status postgresql-12
● postgresql-12.service - PostgreSQL 12 database server
   Loaded: loaded (/etc/systemd/system/postgresql-12.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2021-01-20 14:32:50 CET; 4s ago
     Docs: https://www.postgresql.org/docs/12/static/
  Process: 107154 ExecStartPre=/usr/pgsql-12/bin/postgresql-12-check-db-dir ${PGDATA} (code=exited, status=0/SUCCESS)
 Main PID: 107160 (postmaster)
    Tasks: 8 (limit: 25005)
   Memory: 11.4M
   CGroup: /system.slice/postgresql-12.service
           ├─107160 /usr/pgsql-12/bin/postmaster -D /postgres/12/data
           ├─107161 postgres: logger
           ├─107163 postgres: checkpointer
           ├─107164 postgres: background writer
           ├─107165 postgres: walwriter
           ├─107166 postgres: autovacuum launcher
           ├─107167 postgres: stats collector
           └─107168 postgres: logical replication launcher

## Stop
[root@dbmachine01 system]$ systemctl stop postgresql-12
```

Now enable (or disable, if needed) automatic service start at system boot:

```bash
### Enable automatic startup and shutdown
[root@dbmachine01 system]$ systemctl enable postgresql-12
Created symlink /etc/systemd/system/multi-user.target.wants/postgresql-12.service → /etc/systemd/system/postgresql-12.service.

## Disable automatic startup and shutdown
[root@dbmachine01 system]$ systemctl disable postgresql-12
Removed /etc/systemd/system/multi-user.target.wants/postgresql-12.service.
```

### A shell profile makes life easier

I use to create a shell profile, so I can switch between different installations of PostgreSQL easily.

The file is named `/home/postgres/pg12_profile` and contains settings for the database server's base directory and network port. Additionally it modifies the search path of our shell to make sure we are using the correct binaries. The last line modifies the shell prompt `PS1`.

```bash
export PGDATA=/postgres/12/data
export PGPORT=5443
export PG_HOME=/usr/pgsql-12

export PATH=$PG_HOME/bin:$PATH

PS1="PG12 - [\u@\h \W]\$ "
```

You can activate the settings by sourcing the file on the shell prompt: ```source ~/pg12_profile```

### Connecting with psql

When connecting to the database server with `psql`you can either use the easy way, by sourcing the shell profile:

```bash
PG12 - [postgres@dbmachine01 ~]$ source pg12_profile
PG12 - [postgres@dbmachine01 ~]$ psql
psql (12.5)
Type "help" for help.

postgres=#
```

Or you may specify the connection data as parameters. In our case it is sufficient to specify the network port (`-p 5443`), since we attach to a database server at the local host:

```bash
[postgres@dbmachine01 data]$ /usr/pgsql-12/bin/psql -p 5443
```

Enter the password and you are logged in. If you want to be logged in without entering the password, read on.

#### Using a password file to automatically login with psql

For many reasons, we might want to login without typing the password each time. For testing and development purposes, upgrading the database cluster, or to simplify usage of some of the system tools provided by PostgreSQL (e.g. pg_dumpall).

`psql`therefore supports reading authentication data from the file `~/.pgpass`. This file can contain multiple entries for different targets, but for my usage pattern I only have one entry. It will provide the password of the `postgres`database user no mater on which database on the local host at port 5443 I connect:

```
localhost:5443:*:postgres:secret-password
```

Entries are formatted in this way: *hostname:port:database:username:password*. Instead of a specific database, the asterisk `*` acts as wildcard for whichever database to connect to.

The file must be only read- and writeable by the owner, so apply a `chmod 600 ~/.pgpass`, or it will not be used by `psql`!



------

## 13.1

### Package sources

Use the RPM packages provided by the official PostgreSQL team - located at https://download.postgresql.org/pub/repos/yum/13/redhat/rhel-8-x86_64/

* https://download.postgresql.org/pub/repos/yum/13/redhat/rhel-8-x86_64/postgresql13-13.1-3PGDG.rhel8.x86_64.rpm
* https://download.postgresql.org/pub/repos/yum/13/redhat/rhel-8-x86_64/postgresql13-contrib-13.1-3PGDG.rhel8.x86_64.rpm
* https://download.postgresql.org/pub/repos/yum/13/redhat/rhel-8-x86_64/postgresql13-libs-13.1-3PGDG.rhel8.x86_64.rpm
* https://download.postgresql.org/pub/repos/yum/13/redhat/rhel-8-x86_64/postgresql13-server-13.1-3PGDG.rhel8.x86_64.rpm

### Package installation

Download all packages to a directory. From this directory, install the packages using YUM. Dependencies are automatically installed during the process.

```
yum install postgresql13*.rpm
```

### OS user "postgres"

During package installation, the operating system user `postgres`will also be created. 

We want to recreate it with custom settings, so it is usable as administrative account for the database server:

```bash
## remove the user created by package installation
userdel -r postgres

## add the group "postgres"
groupadd postgres

## create the user
useradd -m -g postgres -c "PostgreSQL server" -s /bin/bash -d /home/postgres postgres
passwd postgres

## adapt permissions to the new user ID
chown postgres:postgres /var/run/postgresql
```

### Directory structure for the database cluster

- /postgres/13 - base directory for all database files
  - /postgres/13/data - the DATA directory of the database cluster
  - /postgres/13/log - logfiles are stored here

```bash
mkdir -p /postgres/13/data
mkdir -p /postgres/13/log
mkdir -p /postgres/13/wal
```

### Database initialization

Now it's time to actually create a PostgreSQL database cluster:

```
/usr/pgsql-13/bin/initdb -A scram-sha-256 -D /postgres/13/data --username=admin --pwprompt --data-checksums
```

Here is a short explanation of the parameters used:

- all users must authenticate with password `-A scram-sha-256`
- Data directory will be created at `-D /postgres/13/data`
- The password for the database super-user `postgres` must be entered during initialization `--pwpromt`
- Detect I/O system corruptions with `--data-checksums`

### postgresql.conf

A minimal configuration file at `/postgres/13/data/postgresql.conf`replaces the default configuration file.

```
## the bare minimum required to start server
data_directory = '/postgres/13/data'
listen_addresses = '*'

## logging
log_destination = 'stderr'
logging_collector = on
log_directory = '/postgres/13/log/'
log_filename = 'postgresql-%Y%m%d.log'
log_line_prefix = '%m [%p]: [%l-line] user=%u,db=%d,remote=%h,app=%a '
log_file_mode = 0600
log_truncate_on_rotation = on
log_rotation_age = 1d
log_rotation_size = 0
log_checkpoints = on
log_connections = on
log_disconnections = on

# performance logging
log_lock_waits = on
log_min_duration_statement = 60000  # duration in milliseconds  -1=disabled 0=log all

## memory 
shared_buffers = 128MB
huge_pages = try
work_mem = 8MB
maintenance_work_mem = 8MB
autovacuum_work_mem = 8MB

fsync = on
full_page_writes = on

## Checkpoints
checkpoint_timeout = 15min
```

### Server startup and shutdown

We are now able to start the database server, giving it a manual try. So we can see if everything works as expected:

```bash
## Start
[postgres@dbmachine01 ~]$ /usr/pgsql-13/bin/pg_ctl start -D /postgres/13/data/
waiting for server to start....2021-01-20 13:25:26.122 GMT [107032]: [1-line] user=,db=,remote=,app= LOG:  redirecting log output to logging collector process
2021-01-20 13:25:26.122 GMT [107032]: [2-line] user=,db=,remote=,app= HINT:  Future log output will appear in directory "/postgres/13/log".
```

Shutdown the database server:

```bash
## Stop
[postgres@dbmachine01 log]$ /usr/pgsql-13/bin/pg_ctl stop -D /postgres/13/data/
waiting for server to shut down.... done
server stopped
```

Finishing this short test will make sure everything is set for the next step - automatic startup and shutdown.

#### Automatic startup and shutdown

RPM packages also provide startup and shutdown script for the systemd facility. These files also allow customization of the database location and network port. The customization file may be owned by the `postgres`OS user to allow a little bit customization during the startup process.

- Service control file - do not edit this file: `/usr/lib/systemd/system/postgresql-13.service`

- Customization file for systemd: `/etc/systemd/system/postgresql-13.service`
  Example content - set a custom TCP port and data directory

  ```
  .include /usr/lib/systemd/system/postgresql-13.service
  
  [Service]
  Environment=PGPORT=5453
  Environment=PGDATA=/postgres/13/data
  ```

Now we can control our database server using `systemctl`:

```bash
## Start
[root@dbmachine01 system]$ systemctl start postgresql-13

## Status
[root@dbmachine01 system]$ systemctl status postgresql-13
● postgresql-13.service - PostgreSQL 13 database server
   Loaded: loaded (/etc/systemd/system/postgresql-13.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2021-01-20 14:28:38 CET; 5s ago
     Docs: https://www.postgresql.org/docs/13/static/
  Process: 107073 ExecStartPre=/usr/pgsql-13/bin/postgresql-13-check-db-dir ${PGDATA} (code=exited, status=0/SUCCESS)
 Main PID: 107079 (postmaster)
    Tasks: 8 (limit: 25005)
   Memory: 11.5M
   CGroup: /system.slice/postgresql-13.service
           ├─107079 /usr/pgsql-13/bin/postmaster -D /postgres/13/data
           ├─107080 postgres: logger
           ├─107082 postgres: checkpointer
           ├─107083 postgres: background writer
           ├─107084 postgres: walwriter
           ├─107085 postgres: autovacuum launcher
           ├─107086 postgres: stats collector
           └─107087 postgres: logical replication launcher
## Stop
[root@dbmachine01 system]$ systemctl stop postgresql-13
```

Now enable (or disable, if needed) automatic service start at system boot:

```bash
### Enable automatic startup and shutdown
[root@dbmachine01 system]$ systemctl enable postgresql-13
Created symlink /etc/systemd/system/multi-user.target.wants/postgresql-13.service → /etc/systemd/system/postgresql-13.service.

## Disable automatic startup and shutdown
[root@dbmachine01 system]$ systemctl disable postgresql-13
Removed /etc/systemd/system/multi-user.target.wants/postgresql-12.service.
```

### A shell profile makes life easier

I use to create a shell profile, so I can switch between different installations of PostgreSQL easily. It eases use of client and server tools as well. For example, `pg_ctl`can be used without additional parameters.

The file is named `/home/postgres/pg13_profile` and contains settings for the database server's base directory and network port. Additionally it modifies the search path of our shell to make sure we are using the correct binaries. The last line modifies the shell prompt `PS1`.

```bash
export PGDATA=/postgres/13/data
export PGPORT=5453
export PG_HOME=/usr/pgsql-13

export PATH=$PG_HOME/bin:$PATH

PS1="PG13 - [\u@\h \W]\$ "
```

You can activate the settings by sourcing the file on the shell prompt: ```source ~/pg13_profile```

### Connecting with psql

When connecting to the database server with `psql`you can either use the easy way, by sourcing the shell profile:

```bash
PG12 - [postgres@dbmachine01 ~]$ source pg12_profile
PG12 - [postgres@dbmachine01 ~]$ psql
psql (12.5)
Type "help" for help.

postgres=#
```

Or you may specify the connection data as parameters. In our case it is sufficient to specify the network port (`-p 5443`), since we attach to a database server at the local host:

```bash
[postgres@dbmachine01 data]$ /usr/pgsql-13/bin/psql -p 5453
```

Enter the password and you are logged in. If you want to be logged in without entering the password, read on.

#### Using a password file to automatically login with psql

For many reasons, we might want to login without typing the password each time. For testing and development purposes, upgrading the database cluster, or to simplify usage of some of the system tools provided by PostgreSQL (e.g. pg_dumpall).

`psql`therefore supports reading authentication data from the file `~/.pgpass`. This file can contain multiple entries for different targets, but for my usage pattern I only have one entry. It will provide the password of the `postgres`database user no mater on which database on the local host at port 5453 I connect:

```
localhost:5453:*:postgres:secret-password
```

Entries are formatted in this way: *hostname:port:database:username:password*. Instead of a specific database, the asterisk `*` acts as wildcard for whichever database to connect to.

The file must be only read- and writeable by the owner, so apply a `chmod 600 ~/.pgpass`, or it will not be used by `psql`!



---



