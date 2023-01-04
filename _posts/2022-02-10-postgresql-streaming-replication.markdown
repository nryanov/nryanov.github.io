---
layout: single
title:  "PostgreSQL: Streaming Replication"
date:   2022-02-10 22:50:00 +0300
categories: postgresql replication database streaming-replication
---

# Table of contents
1. [Prerequisite](#prerequisite)
2. [Streaming replication](#streaming-replication)
3. [Setup](#setup)
   1. [Primary server](#aprimary server)
   2. [Replica server](#replica)
   3. [Sync mode](#sync-mode)
4. [Standby promotion](#standby-promotion)
5. [Conclusion](#conclusion)

# Prerequisite <a name="prerequisite"></a>
All examples assumes that postgresql is already installed on your machine.
Also all examples are created using `PostgreSQL 14.1 on aarch64-apple-darwin20.6.0, compiled by Apple clang version 13.0.0 (clang-1300.0.29.3), 64-bit`.

# Streaming replication <a name="streaming-replication"></a>
Streaming replication is a built-in mechanism in PostgreSQL to replicate data between multiple servers.
It is a low-level replication mechanism as it streams WAL data from primary server to the replica through the physical replication slot,
so it is highly recommended to replicate data between servers using similar PostgreSQL major version (minor versions could be different).
Also it is a good idea to have equal servers in terms of server configuration such as CPU, RAM and Disks, especially if you consider to promote replica to master if primary server goes down.

If you need to replicate data between PostgreSQL servers which use different versions then consider Logical replication.

# Setup <a name="setup"></a>
To setup streaming replication we need at least two instances: one will be running as a primary server, another one as a replica.

## Primary server <a name="primary server"></a>
```sh
-- init new cluster
initdb -D /tmp/postgresql/db1
```

Update postgresql.conf:
- **wal_level**=`replica` (or `logical`)
- **max_replication_slots**=`{ >= subscriber count + some reserve }`
- **max_wal_senders**=`{ max_replication_slots + the number of physical replicas that are connected at the same time }`

```sh
-- start cluster
pg_ctl -D /tmp/postgresql/db1 -o "-p 5432" -l /tmp/postgresql/db1/logs start
```

Now we need to create a specific user for replication. You also can choose to use superuser, but it is not recommended because of security reasons.
```sql
CREATE USER streaming_replicator REPLICATION ENCRYPTED PASSWORD 'password';
```
After user creation we also need to update `pg_hba.conf` to allow replica to connect to the primary server using created user and password.
```
host    replication  streaming_replicator  127.0.0.1/32    md5
```
Restart primary server or execute query for reloading configs:
```sql
SELECT pg_reload_conf(); -- reload config
```

Now we can create a replication slot:
```sql
SELECT * FROM pg_create_physical_replication_slot('replication_slot');
-- get info about created replication slots
SELECT * FROM pg_replication_slots;
```

At this point setup for the primary server instance is done and we can start to setup replica.

## Replica server <a name="replica"></a>
For replica we need to synchronize data directory with primary server. To do this we can use [pg_basebackup](https://www.postgresql.org/docs/current/app-pgbasebackup.html) tool.
```sh
pg_basebackup -D /tmp/postgresql/db2 -F plain -R -X stream -S replication_slot -d application_name=replica1 -h localhost -p 5432 -U streaming_replicator -W
```
This tool has more flags but for now we will inspect only the most important for us:
- `-F` -- output format. By default it is `plain` but also `tar` can be chosen. For replication we need to copy data as plain files saving the same layout as the source server's data directory and tablespaces.
- `-R` -- to mark new cluster as replica we need an additional file `standby.signal`. Also, it is needed to append connection settings to the primary server in `postgresql.auto.conf`. It can be done manually, but using this flag it will be done automatically.
- `-X` -- as we want to create a standby replica we need to synchronize all data from the primary server. While creating backup primary server can continue to work so it will generate new WAL files. `-X stream` allows to stream WAL data while the backup is being taken.
- `-S` -- specify replication slot name. Can be used only with `-X stream`. If we don't specify it then temporary replication slot will be created. As we want to create a standby replica then it is important to save any necessary WAL data in the time between the end of the base backup and the start of streaming replication on the new standby.
- `-d` -- this flag allows as to specify connection string or just used to connect to the server. Any conflicting command line values will be overwritten. In this case we explicitly specify `application_name`.
- `-h`, `-p`, `-U`, `-W` -- host, port, username and password prompt

When `pg_basebackup` is done we can see that there are `standby.signal` and `postgresql.auto.conf` files. `standby.signal` is just a marker which indicates that this instance is a replica.
`postgresql.auto.conf` included parameters which needed for replication:
```sh
cat postgresql.auto.conf
```
```
primary_conninfo = 'user=streaming_replicator password=password channel_binding=prefer host=localhost port=5432 application_name=replica1 sslmode=prefer sslcompression=0 sslsni=1 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres target_session_attrs=any'
primary_slot_name = 'replication_slot'
```
As you can see, `primary_slot_name` has exactly the same value as we specified when executing `pg_basebackup` with `-S` flag. Also it includes `primary_conninfo` with `application_name=replica1`.

Now we can start replica:
```sh
-- start cluster
pg_ctl -D /tmp/postgresql/db2 -o "-p 5433" -l /tmp/postgresql/db2/logs start

-- logs
[14922] LOG:  database system is ready to accept read-only connections
[14927] LOG:  started streaming WAL from primary at 0/5000000 on timeline 1
```

On primary server we can check if replication works:
```sql
-- primary server
SELECT application_name, sync_state FROM pg_stat_replication;
-- -- replica1,async
```

By default streaming replication is async. To make it sync we need to update primary server's `postgresql.conf`.

## Sync mode <a name="sync-mode"></a>
By default streaming replication is async. To make it sync we need to update primary server's `postgresql.conf` and set `synchronous_standby_names`.
- `synchronous_standby_names = 'replica1'`

```sql
SELECT pg_reload_conf(); -- reload config
SELECT application_name, sync_state FROM pg_stat_replication;
-- [replica_1, sync]
```

Now if we try to execute query which modifies data transaction will wait until listed replica will commit it:
```sql
INSERT INTO t1
SELECT i AS id, ('value_' || i) AS value
FROM GENERATE_SERIES(11, 20) s(i);
```
If replica will be offline then transaction will be open until specified timeout.

Let's start replica:
```sh
 pg_ctl -D /tmp/postgresql/db2 -o "-p 5433" -l /tmp/postgresql/db2/logs start
```

Also we can check that now replication is synchronous:
```sql
-- primary server
SELECT application_name, sync_state FROM pg_stat_replication;
-- replica1,sync
```

# Examples <a name="examples"></a>
Let's try to execute some queries on primary server and then check what will happen on replica:
```sql
-- primary server
CREATE TABLE t1
(
    id    int PRIMARY KEY,
    value text NOT NULL
);

INSERT INTO t1
SELECT i AS id, ('value_' || i) AS value
FROM GENERATE_SERIES(1, 10) s(i);
```

DDL and DML queries will be replicated on replica:
```sql
-- replica
SELECt * FROM t1;
```
Also, keep in mind that replica is in read-only mode, so it is impossible to execute queries which modifies data:
```sql
INSERT INTO t1(id, value) VALUES (11, 'value_11');
-- [25006] ERROR: cannot execute INSERT in a read-only transaction
```

What will happen if primary server instance is stopped?
```sh
pg_ctl -D /tmp/postgresql/db1 -o "-p 5432" -l /tmp/postgresql/db1/logs stop
```
On replica's side we will see logs like these:
```sh
[14927] LOG:  replication terminated by primary server
[14927] DETAIL:  End of WAL reached on timeline 1 at 0/5026078.
[14927] FATAL:  could not send end-of-streaming message to primary: server closed the connection unexpectedly
		This probably means the server terminated abnormally
		before or while processing the request.
	no COPY in progress
[14923] LOG:  invalid record length at 0/5026078: wanted 24, got 0
[15022] FATAL:  could not connect to the primary server: connection to server at "localhost" (::1), port 5432 failed: Connection refused
		Is the server running on that host and accepting TCP/IP connections?
	connection to server at "localhost" (127.0.0.1), port 5432 failed: Connection refused
		Is the server running on that host and accepting TCP/IP connections?
```

After restarting primary server:
```sh
pg_ctl -D /tmp/postgresql/db1 -o "-p 5432" -l /tmp/postgresql/db1/logs start
-- replica's log
[15057] LOG:  started streaming WAL from primary at 0/5000000 on timeline 1
```

# Standby promotion <a name="standby-promotion"></a>
In case when primary server stopped temporarily it may be fine to just wait until it restarted and works again. But what if this is not possible to wait? In this case we have to promote standby to primary server:
```sql
-- replica
SELECT pg_promote();
INSERT INTO t1(id, value) VALUES (11, 'value_11');
```

```sh
[14923] LOG:  received promote request
[14923] LOG:  redo done at 0/50260B0 system usage: CPU: user: 0.00 s, system: 0.01 s, elapsed: 1579.03 s
[14923] LOG:  last completed transaction was at log time 2022-02-10 21:27:22.642367+03
[14923] LOG:  selected new timeline ID: 2
[14923] LOG:  archive recovery complete
[14922] LOG:  database system is ready to accept connections
```

Also after `SELECT pg_promote()` file `standby.signal` will be removed and replica becomes the new primary server.
To return old primary server instance we need to restore it and make it, for example, as a new replica.

# Conclusion <a name="conclusion"></a>
This was a quick example of how to setup streaming replication in PostgreSQL.
There is also a topic about high-availability but this will be covered in other articles.
