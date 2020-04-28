# Securing C-lightning funds with PostgresQL reliability capabilities

*This is a new version of the guide much simpler and more straight to the point. If you need to refer to the old guide dealing with WAL shipping + streaming, please refer to [this branch][WAL+streaming]

This article assumes a good knowledge of how Lightning Network works. For a better understanding there are good [articles][Lightning Network] or the [Lightning Network RFC].

## The problem

Every Lightning Network node must keep track of every *state* of each of his channels in every moment in order to preserve the integrity of the funds it manages. 

If a node broadcast an old channel state and "force close" a channel, its peer have lag of time within which it has the right and will broadcast a [penalty transaction], spending the funds which whoud be destined to the node, to an address the peer instead control.

This implies that if we loose the data regarding the channel state (due e.g. to a system failure) and we have a non perfectly up-to-date backup to restore from, it is really dangerous to force close any of our channel, because we cannot be sure to broadcast the latest valid state and we could loose our share of bitcoins in the channel.

What it is really needed in case of data loss in the node, **is a reliable copy of the node's data, taken the instant before the loss has occurred**. Anything less than this means an high probability of loss of funds.

For the official explanation on the penalty transaction, please find here the [official definition in the protocol documentation][penalty transaction].

For a detailed report on how often these type of transaction occurs, please refers to the good [bitmex research report on the topic](https://blog.bitmex.com/lightning-network-justice/).

## The solution

All the developers working on Lightning Network have  tried to find a solution to this problem, but the reality is that the most promising  [solution][eltoo] would require some non trivial change at the base layer that is still under discussion.

In the meanwhile, the team behind c-lightning, the modular implementation of the protocol written in c, has found an architectural solution which is the object of the present guide.

## The choice of C-lightning

C-lightning characterizes itself as the most modular and lean solution in Lightning Network development. It follows strictly all the lessons learned by its developers in the Linux operating system environment.
C-lightning is made of small components that integrates with each other and with the OS.

Following this developing philosophy, the team has chosen not to directly address the problem, but to give the user the tools to build his own solution best fitted for his own environment and needs.

The developers has detached the node engine from the backend relational database designated to host all the application data. The reason is that relational databases have managed mission-critical data for more than 40 years now and have developed distinctive techniques that a single program would hardly be capable to compare with. Also it wouldn't be so well reviewed and tested in other mission critical environments.

The first database that has been chosen for its robustness as a possible backend of c-lightning is [PostgresQL].

The object of this guide is to lead you in setting up c-lightning with PostgresQL as its backend and to ensure that the database has a mechanism of replication which prevents the system from the loss of the critical data of the node in case of a failover.

What follows is a possible solution to data availability which can be adopted by a node owner, who has at least two physical machine available on the same LAN.

## The steps to secure c-lightning data with PostgresQL

There multiple ways you can ensure [high availability] to a postgresQL database.

This document focuses on synchronous *[streaming replication]*.
This solution has been chosen because it supports syncronous replication and low administration overhead compared to some other replication solutions and low performance load on the master and on the standby server:

Synchronous here means 
>*"that a data-modifying transaction is not considered committed until all servers have committed the transaction. This guarantees that a failover will not lose any data and that all load-balanced servers will return consistent results no matter which server is queried."* (https://www.postgresql.org/docs/12/different-replication-solutions.html)

Due to the way *punitive transactions* work in Lightning Network we cannot loose any channel state in any moment. the chosen solution also could allow to restart the standby server immediatly, provided it has a  sleeping c-lightning installation connected to the database with a compatible configuration, but for now we focus in just preserving the data. 

[Reliability] is a serious and multi-layered problem <sup id="a1">[1](#f1)</sup> and this brings us to an important warning: 

*I have no experience in mission critical application and, as stated above, [reliability] is a complex issue. Many things can go wrong from cache to non ECC memory, Operative systems particularities, etc. The article linked cover a large part of things to take care of and this guide can only refers to resources more deep into the matter.*

The streaming replication solution has these steps:

1. Install PostgresQL on two machines on the same LAN
2. Adopt PostgresQL as the database backend for the node
3. Set up the [high availability] the data through *[synchronous streaming replication]* between the two servers

### 1. Install PostgresQL on two machines on the same LAN<sup id="a2">[2](#f2)</sup>

For the purpose of the guide, Let's suppose we have two Linux machine on the same LAN:
One Master Server where we will run our c-lightning node and one Standby server which will have an up-to-date copy of our node's data in case the Master server suffers a crash.

Our machines have this IP address:

* Master server 192.168.0.5
* Standby server 192.168.0.6

this guide is based on version 12. This version has had a non trivial change [in the way the restore operation is done for replication][release notes for v.12],  so it is better to upgrade to it.

#### Download and install ver.12.x on both machine

The type of availability policy we have choosen requires that the two machine have the same major version.
Follow the instructions in https://www.postgresql.org/download/ for your system on both machines.

To test the installation on Linux you can follow this simple steps on both machine:

```bash
sudo -i -u postgres

psql
select version();
```

You should obtain one row of the database with the current version and some information about the system it is running on.

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

We will work only **on the Master Server for the moment**.

#### Create lightningd database and a user into postgresQL

When you connect your node an empty database for c-lightning should be present. It is also appropriate to manage the database with a specific *role* (user) we are going to create for the purpose.

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

remember to substitute `<password assigned to lightningusr>` with the actual password you have assigned to `lightningusr`.

If you prefer to run an explicative command line instead, add to the `lightningd` command.

```bash
--wallet=wallet=postgres://lightningusr:<password assigned to lightningusr>@localhost:5432/lightningdb
```

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

Find your `pg_hba.conf` file and add the following line:

```bash
sudo nano /etc/postgresql/12/main/pg_hba.conf
```

```bash
host    replication     lightningusr     192.168.0.6/32           md5
```

It means that the master server will accept TCP connections to the metadatabase called `replication` from the user authenticated as lightningusr when the request comes from the IP 192.168.0.6/32 with the authentication method md5.

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
listen_addresses = 'localhost,192.168.0.5' # required for streaming replication
wal_level = replica
wal_log_hints = on
max_wal_senders = 3
wal_keep_segments = 8
max_replication_slots = 2  
synchronous_standby_names = 'lightningd'    # gbd Mar 28 Gen 2020 16:13:46 CET
full_page_writes = on                   # gbd Ven  7 Feb 2020 10:58:33 CET https://www.postgresql.org/docs/12/app-pgbasebackup.html
```

#### Setting Up a Standby Server for [streaming replication]

##### Make a first backup of the master on the standby server

```bash
sudo systemctl stop postgresql # necessary to use the slot during the first backup
mkdir /p $HOME/master.backup
pg_basebackup -D /home/bitcoin/backup.cubi/initial_backup -l compressed -P -v -X stream -S 'node_a_slot' -F t -z  -d postgres://postgres:< password assigned to postgres >@10.2.206.136:5432
cp -R  /var/lib/postgresql/12/main/* /tmp
rm -R /var/lib/postgresql/12/main/*
cd /var/lib/postgresql/12/main  # the main cluster data directory
tar -zxvf /Users/Shared/backup.cubi/initial_backup/base.tar.gz
cd pg_wal
tar -zxvf /Users/Shared/backup.cubi/initial_backup/pg_wal.tar.gz
touch standby.signal    # it is the signaling file that make the server behave as a standby one.
```

###### edit the `postgresQL.conf`on the standby server

open the file `postgresql.conf` and add anywhere (I suggest at the top):

```bash
primary_conninfo = 'host=192.168.0.5 port=5432 user=lightningusr password=< password assigned to lightningusr > application_name=lightningd dbname=replication'
primary_slot_name = 'node_a_slot'
hot_standby = off
```

NOTE: One important setting is the `application_name` in the primary_conninfo parameter.

**The `application_name` will make the Master server activate the synchronous replication. I must have the same value of the `synchronous_standby_names` parameter on the master server.**

Now start postgresql on the master and on the client.

```bash
sudo systemctl start postgresql
```

On `postgresql.log` file on the standby server, you can see similar to:

```bash
tail /usr/local/var/log/postgres.log
2020-04-25 21:12:12.380 CEST [34763] LOG:  starting PostgreSQL 12.1 on x86_64-apple-darwin19.2.0, compiled by Apple clang version 11.0.0 (clang-1100.0.33.16), 64-bit
2020-04-25 21:12:12.381 CEST [34763] LOG:  listening on IPv6 address "::1", port 5432
2020-04-25 21:12:12.381 CEST [34763] LOG:  listening on IPv4 address "127.0.0.1", port 5432
2020-04-25 21:12:12.382 CEST [34763] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5432"
2020-04-25 21:12:12.391 CEST [34770] LOG:  database system was interrupted; last known up at 2020-04-25 21:05:12 CEST
2020-04-25 21:12:12.436 CEST [34770] LOG:  entering standby mode
2020-04-25 21:12:12.571 CEST [34771] LOG:  started streaming WAL from primary at 6/3F000000 on timeline 1
2020-04-25 21:12:12.625 CEST [34770] LOG:  redo starts at 6/3F000028
2020-04-25 21:12:12.626 CEST [34770] LOG:  consistent recovery state reached at 6/3F000138
```

In the `postgresql.log` file on the master you should see in the lines:

```bash
tail /usr/local/var/log/postgres.log
2020-04-25 21:12:14.252 CEST [31476] postgres@[sconosciuto] LOG: standby "lightningd" is now synchronous standby with priority 1
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
sync_state       | sync <<<<<<<< 
reply_time       | 2020-04-19 21:44:40.045909+02
```

Note in particular that the parameter `sync_state` has the value "sync".

It would be opportune to read also the detailed guide on [replication] for fine tuning.

Now you can start the `lightningd` on master:

```bash
sudo systemctl start lightningd
```

## Footnotes

<a name="f1">1</a>: Probably [logical replication] could also be an alternative to streaming replication. It focuses on just one database but it presumes an existing *replication identity* in every data object, which is not possible to assume now and for the future for `lightningd` database. [↩](#a1)

<a name="f2">2</a>: log shipping might be able to be skipped. If you try it, please let us know so we can update this section.
It should also be simple, according to this [HA mini-guide] (please read also the good [HA presentation] cited).
Please consider that the article has been written before version 12 which has dropped the `restore.conf` file. Now everything is in the `postgresql.conf` file and there is the new `standby.signal` signaling file. [↩](#a2)

<a name="f2">2</a>: To be on the same LAN is not great in terms of HA. We have chosen to focus on this configuration because, in synchronous replication, every transaction in the database must wait the standby server to confirm at least it has written it in the cache and this pose a bad impact in performances, if the machine is in a remote location. Maybe a setup with [wireguard] though could allow an acceptable setup for a lightning node.
[↩](#a2)

<a name="f3">3</a>: It is possible to adopt postgresQL [migrating the data in your existing sqlite database to postgresQL][sqlite postgres migration]. [↩](#a3)

[WAL+streaming]: https://github.com/gabridome/docs/blob/WAL+streaming/c-lightning_with_postgresql_reliability.md
[PostgrQL]: https://www.postgresql.org/
[eltoo]: https://blockstream.com/2018/04/30/en-eltoo-next-lightning/
[WAL shipping]: https://www.postgresql.org/docs/12/warm-standby.html#WARM-STANDBY
[streaming replication]: https://www.postgresql.org/docs/12/warm-standby.html#STREAMING-REPLICATION
[synchronous streaming replication]: https://www.postgresql.org/docs/12/warm-standby.html#SYNCHRONOUS-REPLICATION
[replication]: https://www.postgresql.org/docs/12/runtime-config-replication.html
[Reliability]: https://www.postgresql.org/docs/12/wal-reliability.html
[high availability]: https://www.postgresql.org/docs/12/different-replication-solutions.html
[HA miniguide]: https://scalegrid.io/blog/getting-started-with-postgresql-streaming-replication/
[HA presentation]: https://scalegrid.io/blog/getting-started-with-postgresql-streaming-replication/
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
[wireguard]: https://www.wireguard.com/
