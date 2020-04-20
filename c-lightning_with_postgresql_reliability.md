# Securing C-lightning funds with PostgresQL reliability capabilities

This article assumes a good knowledge of how Lightning Network works. for a better understanding there are good [articles][Lightning Network] or the [Lightning Network RFC].

## The problem

Every Lightning Network node must keep track of every *state* of each of his channels in every moment in order to preserve the integrity of the funds it manages. 

If a node broadcast an old channel state, force closing a channel, its peer have lag of time within which it has the right and will broadcast a [penalty transaction], spending the funds who whoud be destined to the node, to an address the peer control. 

This implies that if we loose the data regarding the channel state (due e.g. to a system failure), have a non perfectly up-to-date backup to restore from, it is really dangerous to force close any of our channel, because we cannot be sure to broadcast the latest valid state and we could loose our share of bitcoins in the channel.

What it is really needed in case of data loss in the node, **is a reliable copy of the node's data, taken the instant before the loss has occurred**. Anything less than this means an high probability of loss of funds.

For the official explanation on the penalty transaction, please find here the [official definition in the protocol documentation][penalty transaction].

For a detailed report on how often these type of transaction occurs, please refers to the good [bitmex research report on the topic](https://blog.bitmex.com/lightning-network-justice/).

## The solution

All the developers working on Lightning Network have  tried to find a solution to this problem, but the reality is that the most promising  [solution](https://blockstream.com/2018/04/30/en-eltoo-next-lightning/) would require some non trivial change at the base layer that is still under discussion.

In the meanwhile, the team behind c-lightning, the modular implementation of the protocol written in c, has found an architectural solution which is the object of the present guide.
 
## The choice of C-lightning

C-lightning characterizes itself as the most modular and lean solution in Lightning Network development. It follows strictly all the lessons learned by its developers in the Linux operating system environment.
C-lightning is made of small components that integrates with each other and with the OS.

Following this developing philosophy, the team has chosen not to directly address the problem, but to give the user the tools to build his own solution best fitted for his own environment and needs.

The developers has detached the node engine from the backend relational database designated to host all the application data. The reason is that relational databases have managed mission-critical data for more than 30 years now and have developed distinctive techniques that a single program would hardly be capable to compare with. Also it wouldn't be so well reviewed and tested in other mission critical environments.

The first database that was chosen for its robustness as a possible backend of c-lightning is PostgresQL. 

The object of this guide is to lead you in setting up c-lightning with PostgresQL as its backend and to ensure that the database has a mechanism of replication which prevents the system from the loss of the critical data of the node.

What follows is a possible solution to data availability which can be adopted by a node owner, who has at least two physical machine available on the same LAN.

## The steps to secure c-lightning data with PostgresQL

There multiple ways you can ensure [high availability] to a postgresQL database.

This document focuses on synchronous *[streaming replication]* with a fallback on *[WAL shipping]* AKA *warm standby*.
This solution has been chosen because it supports syncronous replication and low administration overhead compared to some other replication solutions and low performance load on the master: 

*"meaning that a data-modifying transaction is not considered committed until all servers have committed the transaction. This guarantees that a failover will not lose any data and that all load-balanced servers will return consistent results no matter which server is queried."* (https://www.postgresql.org/docs/12/different-replication-solutions.html)

Due to the way *punitive transactions* work in Lightning Network we cannot loose any data. the chosen solution also allow to restart the standby server immediatly, provided it has a c-lightning installation sleeping connected to the database but for now we focus in just preserving the data. [Reliability] is a serious and multi-layered problem <sup id="a1">[1](#f1)</sup>.

*This brings us to an important warning: I have no experience in mission critical application and, as stated above, [reliability] is a complex issue. Many things can go wrong from cache to non ECC memory, Operative systems particularities, etc. The article linked cover a large part of things to take care of and this guide can only refers to resources more deep into the matter<sup id="a2">[2](#f2)</sup>.

The streaming replication solution has these steps:

1. Install PostgresQL on two machines on the same LAN
2. Adopt PostgresQL as the database backend for the node
3. Set up [high availability] of the database through WAL shipping on the standby server
4. Set up the [high availability] the data through *[synchronous streaming replication]* between the two servers

### 1. Install PostgresQL on two machines on the same LAN

For the purpose of the guide, Let's suppose we have two Linux machine on the same LAN:
One Master Server where we will run our c-lightning node and one Standby server which will have an up-to-date copy of our node's data in case the Master server suffers a crash.

our machines have this IP address:
* Master server 192.168.0.5
* Standby server 192.168.0.6

this guide is based on version 12. This version has had a non trivial change [in the way the restore operation is done for replication][release notes for v.12],  so it is better to upgrade to it.

#### Download and install ver.12.x on both machine

The type of availability policy we have choosen requires that the two machine have the same major version.
Follow the instructions in https://www.postgresql.org/download/ for your system on both machines.

To test the installation on Linux you can follow this simple steps on both machine:

```bash
sudo -i -u postgres

$ psql
$ select version();
```

You should obtain one row of the database with the current version and some informations about the system it is running on.

Please note that you have to login interactively as the user postgres, because you haven't yet assigned any role to your usual user. We can live with it.
An other important step is to change postgres password and to write it in your favourite password manager.

```bash
sudo -i -u postgres
psql
postgres=# \password
\q
```

### 2. Adopt PostgresQL as the database backend for the node

This guide will assume that a new node is created from scatch<sup id="a3">[3](#f3)</sup>.

#### Create lightningd database into postgresQL

When you connect your node a database should be present a database for lightning should be present (empty).

```bash
sudo -i -u postgres
createuser --createdb --pwprompt --replication lightningusr # create user lightningusr (set a password)
createdb -O lightningusr lightningdb                        # create database lightningdb
exit
```

Please do this on both on master and standby servers.

#### Install c-lightning on the master server

[Here][c-lightning] the istructions.

#### Connect to the database

The strings to connect to a postgresQL are [well documented][postgres connection strings]. 

It is better to try to launch a simple command so when the node starts we have already tested the connection.

```bash
psql -U lightningusr --host=localhost --port=5432 "dbname=postgres" -t -c "SELECT version();"

```

If you see the version in the replay then you can add this line to [c-lightning configuration file]:

```bash
wallet=postgres://lightningusr:<password assigned to lightningusr>@localhost:5432/lightningdb
```

remember to substitute `<password assigned to lightningusr>` with the actual password you have assigned to lightningusr.

If you prefer to run an explicative command line instead, add 

```bash
--wallet=wallet=postgres://lightningusr:<passwordAssignedTolightningusr>@localhost:5432/lightningdb
```

to the `lightningd` command.

When the server starts, even looking close at the log, there's no way to guess if `lightningd` is effectively using postgresQL as its backend.
Please wait 20 minutes than try:

```bash
psql -U lightningusr --host=localhost --port=5432 "dbname=lightningdb" -t -c "SELECT max(height) from blocks;"
```

This should tell you the height of the last block as it results into postgresQL. if the number corresponds with the last block in the network you can see in your node, then you know that there's a new table (`blocks`) inside the database `lightningdb` and it is up-to-date.
You can guess `lightningd` is using postgresQL as its backend.

Stop lightningd and postgresql daemons:

```bash
sudo systemctl stop lightningd
sudo systemctl stop postgresql

```

### 3. Set up [high availability] of the database through WAL shipping on the standby server

#### WAL Shipping

WAL Shipping is based on a continuous copy of WAL files (WAL segments) from the master to the standby server.
When the master receive a request to alter the datatabase, it will first update the present WAL file with the transaction and then it will effectively alter the database. When the WAL file (WAL segment) has reached 16Mb, it will be closed and a new WAL file will be created or recycled.
It is in this very moment that the `archiving_command` can duplicate the WAL file on the standby machine where it will be taken to update the standby database of the local running instance.
It should be noted that the WAL file contains 16Mb of changes and that we will be alligned only in the moment in which all this changes have been transmitted to the standby server and they have been restored on that instance In the meanwhile and afterwards of this process any change to the master database will make it diverge from the standby. 

This is NOT obviuously what we want to ensure the reliability of our node's channel data. We implement this method just because it is used by *[streaming replication]* has a fallback system.

#### Prepare a standby server for the node's database

Now it is time to work on the *standby server* which is not required to be running the same OS than the master.
On this machine is higly suggested to install the same major version of postgresQL as the master.
This guide is written based on version 12.

#### Prepare a shared resource on the standby server accessible from the master

It could be an ftp resource or a network share or you could also choose to use `scp` to access this resource.
The result MUST be that you have the possibility of archive the WAL files on the standby machine or on a resource of a third machine easy accessible by it.

For we don't want to sacrifice too much in terms of latency, it is better to have master and standby server on the same LAN so we will use a shared NFS resource for the scope.

We will use NFS because it easy to set up and it allows to check for file existence through shell, etc.

On the standby machine create a file /etc/exports with this content:

```bash
/home 192.168.0.1/24(rw,sync,no_root_squash,no_subtree_check,insecure)
```

Where:

* `/home` is the directory we will share on the standby machine
* `192.168.0.1/24(rw,sync,no_root_squash,no_subtree_check,insecure)` is the range of ip numbers that will have the right to access to the shared directory. Please refer to this [NFS Tutorial] for the meaning of the flags.

```bash
sudo systemctl restart nfs-kernel-server
```

We skip the firewall part of the [NFS tutoria] because we assume the two servers being on the same network.

On the **master machine** you will have to mount the shared directory.

We’ll create two directories for our mounts:

```bash
sudo mkdir -p /nfs/home
```

And then the directory is mounted on the **master server** with the command:

```bash
sudo mount 192.168.0.5:/home /nfs/home
```

Now you should be able to see the directory /home/bitcoin/ on the standby server by doing:

```bash
ls /nfs/home/bitcoin
```

from the **master** and to write into it

```bash
touch /nfs/home/bitcoin/hello.world
```

On the **standby server** you should be able to perform:

```bash
ls -l /nfs/home/bitcoin/hello.world
-rw-r--r--  1 gbd  staff  0 19 Apr 12:40 /home/bitcoin/hello.world
rm /nfs/home/bitcoin/hello.world
```

build a place for the archived WAL files (WAL segments)

```bash
mkdir /nfs/home/bitcoin/master.backup/wals
```

#### Prepare master for [WAL shipping] reliability method

Set up authentication on the primary server to allow replication connections from the standby server.
Even if this is necessary just for [streaming replication] (our subsequent phase) on the official documentation they suggest to set it up now, so find you `pg_hba.conf` file and add the following line:

```bash
sudo nano /etc/postgresql/12/main/pg_hba.conf
```

```bash
host    replication     lightningusr     192.168.0.6/32           md5
```

It means that the master server will accept TCP connections to the metadatabase called `replication`from the user authenticated as lightningusr when the request comes from the IP 192.168.0.6/32 with the authentication method md5.


Create a replication slot

```bash
sudo -i -u postgres
psql
postgres=# SELECT * FROM pg_create_physical_replication_slot('node_a_slot');
  slot_name  | lsn
-------------+-----
 node_a_slot |

postgres=# SELECT slot_name, slot_type, active FROM pg_replication_slots;
  slot_name  | slot_type | active 
-------------+-----------+--------
 node_a_slot | physical  | f
(1 row)
\q
exit
```

Open the  postgresQL.conf file on **Master** and ensure that 

```bash
listen_addresses = 'localhost,192.168.0.6' # required for streaming replication
synchronous_commit = remote_write
wal_level = replica
full_page_writes = on  # https://www.postgresql.org/docs/12/app-pgbasebackup.html
wal_log_hints = on   # https://scalegrid.io/blog/getting-started-with-postgresql-streaming-replication/
archive_mode = on		
archive_command = 'test ! -f /nfs/home/bitcoin/master.backup/wals/%f && cp %p /nfs/home/bitcoin/master.backup/wals/%f'
max_wal_senders = 3    
wal_keep_segments = 6
max_replication_slots = 2  
cluster_name = 'lightningd'
```

#### Setting Up a Standby Server
#### Prepare standby server for logshipping

If you're setting up the standby server for high availability purposes, set up WAL archiving, connections and authentication like the Master server, because the standby server will work as a Master server after failover<sup id="a4">[4](#f4)</sup>.

Open `/etc/postgresql/12/main/postgresql.conf`on the standby server and set the following parameters:

```bash
restore_command = 'cp /home/bitcoin/wals/%f %p'  
archive_cleanup_command = '/usr/local/bin/pg_archivecleanup /home/bitcoin/backup.master/wals %r' 
primary_conninfo = 'host=192.168.0.5 port=5432 user=lightningusr password=<password of lightningusr> application_name=lightningd dbname=replication'    
primary_slot_name = 'node_a_slot' # https://www.postgresql.org/docs/12/warm-standby.html#STREAMING-REPLICATION-SLOTS
```

#### Prepare a backup procedure and a restore procedure of the master database from the standby server
The back up and restore procedures are rich and complex. Here a configuration that should work.
Prepare an home for the backup on standby server.

```bash
mkdir /p /home/bitcoin/master.backup
```

write a procedure for the backup of the master server .

```bash
#!/bin/sh
export POSTGRESQL_BKP_DIR=/home/bitcoin/master.backup/compressed-$(date "+%Y-%m-%dT%H:%M:%S") #check the last part for your system
# the date trick is necessary because the backup procedure do not normally overwrite an existing directory.
export POSTGRESQL_BKP_CMD=/usr/bin/pg_basebackup
time $POSTGRESQL_BKP_CMD -D $POSTGRESQL_BKP_DIR -l compressed -R -P -v -X f -F t -z -d postgres://postgres:<password of user postgres>@192.168.0.5:5432

#  The switch -R is necessary to make the standby server start as such in a real recovery it should probably be avoided.
# --write-recovery-conf
# Create standby.signal and append connection settings to postgresql.auto.conf in the output directory
# (or into the base archive file when using tar format) to ease setting up a standby server.
# The postgresql.auto.conf file will record the connection settings and, if specified, the replication
# slot that pg_basebackup is using, so that the streaming replication will use the same settings later on.
```

#### restore the master on the standby

Find out where is the main directory (cluster data directory) of postgresQL by looking into postgresql.conf file for a string like:

```bash
data_directory = '/var/lib/postgresql/12/main'
```

Write a procedure for the [restore] of a **standby server** in archiving mode:
The commands are:

```bash
sudo systemctl stop postgresql
cp -R  /tmp
rm -rf /var/lib/postgresql/12/main/*
cd  /usr/local/var/postgres/main     # or wherever the main cluster directory is.
tar -zxvf /home/bitcoin/master.backup/compressed-2020-02-06T15:29+01:00/base.tar.gz # substitute with the actual backup you need to restore.
cp /tmp/postgres/postgresql.conf .
rm -rf pg_wal/*
```

After having restored the system:
check the existence of `standby.signal`in the cluster data directory of the standby server (the restore procedure should have created it.

```bash
ls -l /var/lib/postgresql/12/main/standby.signal
```

if not create it

```bash
touch /var/lib/postgresql/12/main/standby.signal
```

Start postgresQL on the Master server
Start postgresQL on the Standby server

On Postgresql.log file, you can see similar to:

```bash
2020-04-19 17:39:00.974 CEST [99760] LOG:  starting PostgreSQL 12.1 on....
2020-04-19 17:39:00.976 CEST [99760] LOG:  listening on IPv6 address "::1", port 5432
2020-04-19 17:39:00.976 CEST [99760] LOG:  listening on IPv4 address "127.0.0.1", port 5432
2020-04-19 17:39:00.976 CEST [99760] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5432"
2020-04-19 17:39:00.993 CEST [99767] LOG:  database system was shut down in recovery at 2020-04-19 17:38:55 CEST
cp: /home/bitcoin/master.backup/wals/00000002.history: No such file or directory
2020-04-19 17:39:00.996 CEST [99767] LOG:  entering standby mode
cp: /Users/Shared/backup.cubi/wals/0000000100000005000000B1: No such file or directory
2020-04-19 17:39:01.001 CEST [99767] LOG:  redo starts at 5/B10000A0
2020-04-19 17:39:01.001 CEST [99767] LOG:  consistent recovery state reached at 5/B1000930
2020-04-19 17:39:01.001 CEST [99767] LOG:  invalid record length at 5/B1000930: wanted 24, got 0
2020-04-19 17:39:01.001 CEST [99760] LOG:  database system is ready to accept read only connections
```

In Every moment, you can use this command on the **master** to check the state of the replication.

```bash
psql -U postgres --host=localhost --port=5432 "dbname=postgres" -x -c "SELECT * from pg_stat_replication;"
-[ RECORD 1 ]----+------------------------------
pid              | 9239
usesysid         | 10
usename          | postgres
application_name | lightningd
client_addr      | 192.168.0.6
client_hostname  |
client_port      | 63017
backend_start    | 2020-04-19 21:31:48.538539+02
backend_xmin     |
state            | streaming
sent_lsn         | 5/B6A15910
write_lsn        | 5/B6A15910
flush_lsn        | 5/B6A15910
replay_lsn       | 5/B6A15910
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 1
sync_state       | async <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<
reply_time       | 2020-04-19 21:44:40.045909+02
```

### 4. Set up the [high availability] the data through *[synchronous streaming replication]* between the two servers

#### prepare the standby for asyncronous [streaming replication]

You already prepared everything ALSO for [streaming replication] at this point. Namely:

* You have set `primary_conninfo`on the **standby server**
* You have modified the `pg_hba.conf` file on **Master server**
It would be opportune to read also the detailed guide on [replication] for fine tuning.

#### Switch to Synchronous [streaming replication]

One only step is required to move to [streaming replication] mode:
Put only one more parameter into `postgresql.conf` file on the **Master server*:

```bash
synchronous_standby_names = 'lightningd'
```

The name `lightningd`corresponds to `application_name` value in the parameter `primary_conninfo` on the **standby** and to the `cluster_name` value in the `postgresql.conf` on the **master server**.

If everything has gone right, you will see something like this on the standby server:

```bash
2020-04-19 17:39:01.103 CEST [99773] LOG:  started streaming WAL from primary at 5/B1000000 on timeline 1
```

In Every moment, you can use this command on the **master** to check the state of the replication.

```bash
psql -U postgres --host=localhost --port=5432 "dbname=postgres" -x -c "SELECT * from pg_stat_replication;"
-[ RECORD 1 ]----+------------------------------
pid              | 9239
usesysid         | 10
usename          | postgres
application_name | lightningd
client_addr      | 192.168.0.6
client_hostname  |
client_port      | 63017
backend_start    | 2020-04-19 21:31:48.538539+02
backend_xmin     |
state            | streaming
sent_lsn         | 5/B6A15910
write_lsn        | 5/B6A15910
flush_lsn        | 5/B6A15910
replay_lsn       | 5/B6A15910
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 1
sync_state       | sync   <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<
reply_time       | 2020-04-19 21:44:40.045909+02
```

## Footnotes

<a name="f1">1</a>: Probably [logical replication] could also be an alternative to streaming replication. It focuses on just one database but it presumes an existing *replication identity* in every data object, which is not possible to assume now and for the future for `lightningd` database. [↩](#a1) 

<a name="f2">2</a>: log shipping might be able to be skipped. If you try it, please let us know so we can update this section. [↩](#a2)

<a name="f3">3</a>It is possible to adopt postgresQL [migrating the data in your existing sqlite database to postgresQL][sqlite postgres migration]. [↩](#a3)

<a name="f4">4</a>: I think that you can prepare ONE ONLY postgresql.conf file in which you set all the parameters for the master AND the standby server. In this way you can just prepare one machine (the master) and let the restore command do the job of building a postgres.auto.conf file that can be used after being copied in `/etc/postgresql/12/main/postgresql.conf`.
I cannot test this because my standby server is on a different OS and the cluster data directory is not the same. Maybe someone could try and contribute...;) [↩](#a4)

[WAL shipping]: https://www.postgresql.org/docs/12/warm-standby.html#WARM-STANDBY
[streaming replication]: https://www.postgresql.org/docs/12/warm-standby.html#STREAMING-REPLICATION
[synchronous streaming replication]: https://www.postgresql.org/docs/12/warm-standby.html#SYNCHRONOUS-REPLICATION
[replication]: https://www.postgresql.org/docs/12/runtime-config-replication.html
[Reliability]: https://www.postgresql.org/docs/12/wal-reliability.html
[high availability]: https://www.postgresql.org/docs/12/different-replication-solutions.html
[logical replication]: https://www.postgresql.org/docs/12/logical-replication.html
[release notes for v.12]: https://www.postgresql.org/docs/release/12.0/
[sqlite postgres migration]: https://github.com/fiatjaf/mcldsp
[postgres connection strings]: https://www.postgresql.org/docs/12/libpq-connect.html#LIBQ-CONNSTRING
[c-lightning]: https://github.com/ElementsProject/lightning
[c-lightning configuration file]: https://github.com/ElementsProject/lightning#configuration-file
[NFS tutorial]: https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nfs-mount-on-ubuntu-18-04
[restore]: https://www.postgresql.org/docs/12/continuous-archiving.html#BACKUP-PITR-RECOVERY
[Lightning Network]: https://bitcoinmagazine.com/articles/understanding-the-lightning-network-part-building-a-bidirectional-payment-channel-1464710791
[penalty transaction]: https://github.com/lightningnetwork/lightning-rfc/blob/master/00-introduction.md#penalty-transaction
[Lightning Network RFC]: https://github.com/lightningnetwork/lightning-rfc/blob/master/00-introduction.md
