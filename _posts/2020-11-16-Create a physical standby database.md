---
layout: post
title:  "Create a physical standby database with Oracle 12.1"
date:   2020-11-16 13:38:28 +0100
categories: Oracle DataGuard
---
# Create a physical standby database with Oracle 12.1

This guide will help you build a physical standby database for Oracle Database 12.1. We will start from a standalone instance, adding a standby database without any graphical interface tools. Initial setup will see us manually creating all the configuration needed. Later we advance to converting to a Data Guard Broker setup, allowing easy handling of switchover and failover tasks.

## Assumptions

For this guide I assume the following things:

- Two database servers, named "oracledb01" and "oracledb02"
- Oracle database software 12.1 already installed on both servers, having the same directory structure.
- The database file systems on each host are identically configured.
- One server already has a single instance database created, called "mytest1"
- Database "mytest1" is already running in archive log mode
- We will create a standby database named "mytest2"

------

## Prepare the network configuration files

Network configuration is very important to make the Data Guard setup work properly. I always apply some basic security settings to the network services. A good reference are the CIS benchmarks provided by the [Center of Internet Security](https://www.cisecurity.org). You can find their Oracle database guide here: https://www.cisecurity.org/benchmark/oracle_database/.

#### tnsnames.ora

This file is identical on primary and standby node. Although it is not possible to register services on the other nodes listener, it makes is easier to keep this file in sync for both databases.

##### oracledb01 and oracledb02

```
## Data Guard database mytest1
MYTEST1 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = oracledb01)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = mytest1)
    )
  )

## Data Guard database mytest2
MYTEST2 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = oracledb02)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = mytest2)
    )
  )

## Database service mytest
MYTEST  =
(DESCRIPTION = (CONNECT_TIMEOUT=5)(TRANSPORT_CONNECT_TIMEOUT=3)(RETRY_COUNT=3)
   (ADDRESS_LIST= (LOAD_BALANCE=off) (FAILOVER=on)
     (ADDRESS = (PROTOCOL = TCP)(HOST = oracledb01)(PORT = 1521))
     (ADDRESS = (PROTOCOL = TCP)(HOST = oracledb02)(PORT = 1521))
   )
   (CONNECT_DATA=(service_name=mytest))
)

## Listener entry for service registration on listener_mytest1
LISTENER_MYTEST1 =
 (ADDRESS = (PROTOCOL = IPC)(KEY = mytest1))
 
 ## Listener entry for service registration on listener_mytest2
LISTENER_MYTEST2 =
 (ADDRESS = (PROTOCOL = IPC)(KEY = mytest2))

```

#### listener.ora

We have on TCP listening port and IPC configured for registration of database services. Some basic CIS security settings are applied, as they are very common for Oracle database setups.

In comparison to the tnsnames.ora, files are different on each node, since they contain the hostname of the server. 

Also the databases must have a static entry to work for creation of the standby database and correct operation of Data Guard (?? Broker) ???  (static entries are needed for transient upgrade, but are they really required for Data Guard (Broker)) ???

##### oracledb01

```
SID_LIST_LISTENER_MYTEST1 =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = mytest1)
      (ORACLE_HOME = /u01/app/oracle/product/12.1.0/dbhome_1)
      (SID_NAME = mytest1)
    )
    (SID_DESC =
      (GLOBAL_DBNAME = mytest1_dgmgrl)
      (ORACLE_HOME = /u01/app/oracle/product/12.1.0/dbhome_1)
      (SID_NAME = mytest1)
    )
  )

LISTENER_MYTEST1 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = oracledb01)(PORT = 1521))
    (ADDRESS = (PROTOCOL = IPC)(KEY = mytest1))
  )

ADR_BASE_LISTENER_MYTEST1 = /u01/app/oracle

## CIS Oracle Benchmark 3.0.0 - 2.1.3
ADMIN_RESTRICTIONS_LISTENER=ON

## CIS Oracle Benchmark 3.0.0 - 2.1.1
SECURE_CONTROL_LISTENER=IPC

## CIS Oracle Benchmark 3.0.0 - 2.1.4
SECURE_REGISTER_LISTENER=IPC
```

##### oracledb02

```
SID_LIST_LISTENER_MYTEST2 =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = mytest2)
      (ORACLE_HOME = /u01/app/oracle/product/12.1.0/dbhome_1)
      (SID_NAME = mytest2)
    )
    (SID_DESC =
      (GLOBAL_DBNAME = mytest2_dgmgrl)
      (ORACLE_HOME = /u01/app/oracle/product/12.1.0/dbhome_1)
      (SID_NAME = mytest2)
    )

  )

LISTENER_MYTEST2 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = oracledb02)(PORT = 1521))
    (ADDRESS = (PROTOCOL = IPC)(KEY = mytest2))
  )

ADR_BASE_LISTENER_MYTEST2 = /u01/app/oracle

## CIS Oracle Benchmark 3.0.0 - 2.1.3
ADMIN_RESTRICTIONS_LISTENER=ON

## CIS Oracle Benchmark 3.0.0 - 2.1.1
SECURE_CONTROL_LISTENER=IPC

## CIS Oracle Benchmark 3.0.0 - 2.1.4
SECURE_REGISTER_LISTENER=IPC

```

------

## Prepare the primary database

- Enable forced logging
- Configure how the databases authenticate each other
- Configure standby redo logs
- Initialization parameters
- Enable archiving

### Enable forced logging

This enables the writing of redo records even when DDL is used with the `NOLOGGING`option.

```sql
SQL> ALTER DATABASE FORCE LOGGGING;
```

### Configure how the databased authenticate each other

In most setups authentication will be provided by the password file of the database. This means the primary database must have a password file. Check it with the following SQL:

```sql
SQL> select * from v$pwfile_users;

USERNAME                       SYSDB SYSOP SYSAS SYSBA SYSDG SYSKM     CON_ID
------------------------------ ----- ----- ----- ----- ----- ----- ----------
SYS                            TRUE  TRUE  FALSE FALSE FALSE FALSE          0
SYSDG                          FALSE FALSE FALSE FALSE TRUE  FALSE          0
SYSBACKUP                      FALSE FALSE FALSE TRUE  FALSE FALSE          0
SYSKM                          FALSE FALSE FALSE FALSE FALSE TRUE           0

```

### Configure standby redo logs

Whenever a database is running in standby mode, it stores the redo information received from the primary database in standby redo logs. The size of these logs must be at least as large as the largest redo log of the primary source database.

In my test setup they happen to have the default size of 50M

```sql
SQL> select member from v$logfile;

MEMBER
--------------------------------------------------------------------------------
/u02/oradata/mytest1/redo01.log
/u02/oradata/mytest1/redo02.log
/u02/oradata/mytest1/redo03.log

SQL> select group#, thread#, bytes/1024/1024 "SIZE_MB" from v$log;

    GROUP#    THREAD#    SIZE_MB
---------- ---------- ----------
         1          1         50
         2          1         50
         3          1         50

```

Best practice is to reflect the redo log configuration, having the same count of group and threads for the standby redo logs:

```sql
ALTER DATABASE ADD STANDBY LOGFILE THREAD 1 '/u02/oradata/mytest1/standby_redo01.log' SIZE 50M;
ALTER DATABASE ADD STANDBY LOGFILE THREAD 1 '/u02/oradata/mytest1/standby_redo02.log' SIZE 50M;
ALTER DATABASE ADD STANDBY LOGFILE THREAD 1 '/u02/oradata/mytest1/standby_redo03.log' SIZE 50M;
ALTER DATABASE ADD STANDBY LOGFILE THREAD 1 '/u02/oradata/mytest1/standby_redo04.log' SIZE 50M;
```

The final redo log configuration looks like this:

```sql
SQL> select group#, member, type from v$logfile

    GROUP# MEMBER                                   TYPE
---------- ---------------------------------------- -------
         1 /u02/oradata/mytest1/redo01.log          ONLINE
         2 /u02/oradata/mytest1/redo02.log          ONLINE
         3 /u02/oradata/mytest1/redo03.log          ONLINE
         4 /u02/oradata/mytest1/standby_redo01.log  STANDBY
         5 /u02/oradata/mytest1/standby_redo02.log  STANDBY
         6 /u02/oradata/mytest1/standby_redo03.log  STANDBY

6 rows selected.
```

### Initialization parameters

Now then necessary parameters for the redo transport are set. They do not require a database restart.

These parameters control the redo transport when the database is in primary mode:

```sql
alter system set LOG_ARCHIVE_CONFIG='DG_CONFIG=(mytest1,mytest2)';

alter system set LOG_ARCHIVE_DEST_1=
 'LOCATION=/u03/archive/mytest1 
  VALID_FOR=(ALL_LOGFILES,ALL_ROLES)
  DB_UNIQUE_NAME=mytest1';
  
alter system set LOG_ARCHIVE_DEST_2=
 'SERVICE=mytest2 ASYNC
  VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE) 
  DB_UNIQUE_NAME=mytest2';
```

These parameters are relevant if the database is in standby mode:

```sql
alter system set FAL_SERVER=mytest2;
alter system set STANDBY_FILE_MANAGEMENT=AUTO;
alter system set DB_FILE_NAME_CONVERT='/mytest2/','/mytest1/' scope=spfile;
alter system set LOG_FILE_NAME_CONVERT='/mytest2/','/mytest1/' scope=spfile;
```

### Archiving

The primary database must be in archive log mode:

```sql
SQL> archive log list;
Database log mode              Archive Mode
Automatic archival             Enabled
Archive destination            /u03/archive/mytest1
Oldest online log sequence     57
Next log sequence to archive   59
Current log sequence           59
```

------

## Create the standby database

- create an initialization parameter file
- copy password file
- create network configuration files
- startup listener on oracledb02

### Create the database parameter initialization file

A minimal parameter file for starting the standby instance in nomount mode must be created. Specifying these two naming parameters is enough. All other settings will be specified during RMAN clone operation:

```
db_name='mytest2'
db_unique_name='mytest2'
```

### Copy the password file

The password file must be copied (or created with the same password) from the primary database host to the secondary database host:

```bash
[oracle@oracledb01 dbs]$ scp orapwmytest1 oracle@oracledb02:/u01/app/oracle/product/12.1.0/dbhome_1/dbs/orapwmytest2
```

### Startup nomount

before startup nomount:

- create all directories on the filesystems:
  - datafiles, controlfiles, redologs directories
  - archive directory on disk /u03/archive/mytest2
  - create adump directory on disk /u01/app/oracle/admin/mytest2/adump
- tnsnames.ora must be in place in order to resolve LOCAL_LISTENER parameter

### Clone the primary to the standby database using RMAN

````
connect target sys@mytest1
connect auxiliary sys@mytest2
run {
allocate channel prmy1 type disk;
allocate channel prmy2 type disk;
allocate auxiliary channel stby type disk;

duplicate target database for standby from active database
spfile
set audit_file_dest='/u01/app/oracle/admin/mytest2/adump'
set control_files='/u02/oradata/mytest2/control01.ctl','/u03/oradata/mytest2/control02.ctl'
set db_unique_name='mytest2'
set diagnostic_dest='/u01/app/oracle'
set dispatchers='(PROTOCOL=TCP) (SERVICE=mytest2XDB)'
set fal_server='MYTEST1'
set local_listener='LISTENER_MYTEST2'
set log_archive_config='DG_CONFIG=(mytest1,mytest2)'
set log_archive_dest_1='LOCATION=/u03/archive/mytest2
  VALID_FOR=(ALL_LOGFILES,ALL_ROLES)
  DB_UNIQUE_NAME=mytest2'
set log_archive_dest_2='SERVICE=mytest1 ASYNC
  VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE)
  DB_UNIQUE_NAME=mytest1'
set service_names='mytest2,mytest'
set DB_FILE_NAME_CONVERT='/mytest1/','/mytest2/'
set LOG_FILE_NAME_CONVERT='/mytest1/','/mytest2/'
nofilenamecheck
;
}
````

### Start  redo apply on standby instance

```sql
SQL> ALTER DATABASE RECOVER MANAGED STANDBY DATABASE disconnect from session;

Database altered.
```

You can now check the status of redo transport (RFS) and redo apply (MRP0):

```sql
SQL> SELECT PROCESS, STATUS, THREAD#, SEQUENCE#, BLOCK#, BLOCKS FROM V$MANAGED_STANDBY;

PROCESS   STATUS          THREAD#  SEQUENCE#     BLOCK#     BLOCKS
--------- ------------ ---------- ---------- ---------- ----------
ARCH      CLOSING               1         67      65536        187
ARCH      CONNECTED             0          0          0          0
ARCH      CLOSING               1         66      73728       1870
ARCH      CONNECTED             0          0          0          0
RFS       IDLE                  0          0          0          0
RFS       IDLE                  1         68      46209          1
RFS       IDLE                  0          0          0          0
RFS       IDLE                  0          0          0          0
MRP0      APPLYING_LOG          1         68      46209     102400

9 rows selected.
```

------

## Data Guard Broker setup

- Create a static listener service for DGMGRL

### Create a static listener service for DGMGRL

Data Guard Broker manager needs a static service registered on the local listener of each instance. This service is used to restart instances, manually and also if fast-start failover is used.

#### mytest1 @ oracledb01

```
SID_LIST_LISTENER_MYTEST1 =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = mytest1)
      (ORACLE_HOME = /u01/app/oracle/product/12.1.0/dbhome_1)
      (SID_NAME = mytest1)
    )
   (SID_DESC =
      (GLOBAL_DBNAME = mytest1_dgmgrl)
      (ORACLE_HOME = /u01/app/oracle/product/12.1.0/dbhome_1)
      (SID_NAME = mytest1)
    )
  )
```

#### mytest2 @ oracledb02

```
SID_LIST_LISTENER_MYTEST2 =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = mytest2)
      (ORACLE_HOME = /u01/app/oracle/product/12.1.0/dbhome_1)
      (SID_NAME = mytest2)
    )
    (SID_DESC =
      (GLOBAL_DBNAME = mytest2_dgmgrl)
      (ORACLE_HOME = /u01/app/oracle/product/12.1.0/dbhome_1)
      (SID_NAME = mytest2)
    )
  )
```

### Broker configuration file

Next we have to set up the location of the Data Guard Broker configuration files. By default both copies are places in `$ORACLE_HOME/dbs` and should be changed to independent physical devices wherever possible. You can only change the location if the Broker is not started (`DG_BROKER_START=FALSE`).

So from the default state, I change the location on primary and secondary database:

- Primary database mytest1:

  ```sql
  SQL> alter system set dg_broker_config_file1='/u02/oradata/mytest1/dr1mytest1.dat';
  SQL> alter system set dg_broker_config_file2='/u03/oradata/mytest1/dr2mytest1.dat';
  ```

- Standby database mytest2:

  ```sql
  SQL> alter system set dg_broker_config_file1='/u02/oradata/mytest2/dr1mytest2.dat';
  SQL> alter system set dg_broker_config_file2='/u03/oradata/mytest2/dr2mytest2.dat';
  ```

Now you can start the Broker on both instances:

```sql
SQL> ALTER SYSTEM SET DG_BROKER_START=TRUE;
```

### Create a Broker configuration

Now we create a broker configuration including the primary and standby database.

For this we start the broker command line manager and authenticate us against the primary database:

```
[oracle@oracledb01 ~]$ dgmgrl
DGMGRL for Linux: Version 12.1.0.2.0 - 64bit Production

Copyright (c) 2000, 2013, Oracle. All rights reserved.

Welcome to DGMGRL, type "help" for information.
DGMGRL> connect sys
Password:
Connected as SYSDG.
DGMGRL>
```

Clear the redo transport destination (`log_archive_dest_1`) on primary and standby database. The primary location will stay as it is on both instances.

```
SQL> ALTER SYSTEM SET LOG_ARCHIVE_DEST_2=" ";
```

Create the broker configuration, starting with the primary instance:

```
DGMGRL> create configuration 'DR_mytest' AS
> primary database is 'mytest1'
> connect identifier is mytest1;
Configuration "DR_mytest" created with primary database "mytest1"
```

Check the initial configuration:

```
DGMGRL> show configuration;

Configuration - DR_mytest

  Protection Mode: MaxPerformance
  Members:
  mytest1 - Primary database

Fast-Start Failover: DISABLED

Configuration Status:
DISABLED
```

Now add the standby database:

```
DGMGRL> add database 'mytest2' as
> connect identifier is mytest2;
Database "mytest2" added
```

Check the configuration again:

```
DGMGRL> show configuration

Configuration - DR_mytest

  Protection Mode: MaxPerformance
  Members:
  mytest1 - Primary database
    mytest2 - Physical standby database

Fast-Start Failover: DISABLED

Configuration Status:
DISABLED
```

Finally, edit the property `StaticConnectIdentifier`. This makes sure that all databases can be properly controlled by `dgmgrl`:

```
edit database mytest2 set property
StaticConnectIdentifier='(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=oracledb02)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=mytest2_dgmgrl)(INSTANCE_NAME=mytest2)(SERVER=DEDICATED)))'; 

edit database mytest1 set property
StaticConnectIdentifier='(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=oracledb01)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=mytest1_dgmgrl)(INSTANCE_NAME=mytest1)(SERVER=DEDICATED)))'; 
```

------

## Perform a switchover

Switching the database roles is easy with Data Guard Broker. Pay attention to the fact that a switchover is not the same as a failover. A switchover is only allowed if the standby database is working properly, applying all redo as expected.

```
DGMGRL> switchover to mytest2;
Performing switchover NOW, please wait...
Operation requires a connection to instance "mytest2" on database "mytest2"
Connecting to instance "mytest2"...
Connected as SYSDBA.
New primary database "mytest2" is opening...
Operation requires start up of instance "mytest1" on database "mytest1"
Starting instance "mytest1"...
ORACLE instance started.
Database mounted.
Switchover succeeded, new primary is "mytest2"
```

Databases have now switched the roles. Primary database is instance mytest2 on host oracledb02. Former primary mytest1 on oracledb01 is now the standby database.

------

## References

- Creating a physical standby database with Oracle 12.1 - https://docs.oracle.com/database/121/SBYDB/create_ps.htm#SBYDB00200
- Settings the StaticConnectIdentifier - http://qdosmsq.dunbar-it.co.uk/blog/2014/01/interesting-data-guard-problem/