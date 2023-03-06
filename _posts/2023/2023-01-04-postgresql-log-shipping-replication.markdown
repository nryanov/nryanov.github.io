---
layout: single
title:  "PostgreSQL: Log shipping Replication"
date:   2023-01-04 22:50:00 +0300
categories: postgresql replication database high-availability log-shipping
---

# Table of contents
1. [Prerequisite](#prerequisite)
2. [Log shipping replication](#log-shipping-replication)
3. [Setup](#setup)
  1. [Primary server](#primary-server)
  2. [Replica server](#replica)
4. [Replication in action](#action)   
5. [Standby promotion](#standby-promotion)
6. [Conclusion](#conclusion)

# Prerequisite <a name="prerequisite"></a>
All examples assumes that postgresql is already installed on your machine.
Also all examples are created using `PostgreSQL 14.1 on aarch64-apple-darwin20.6.0, compiled by Apple clang version 13.0.0 (clang-1300.0.29.3), 64-bit`.

# Log shipping replication <a name="log-shipping-replication"></a>
Log shipping replication (i will use a short name for it `LSR`) is another one method to physically replicate data between multiple database clusters. As name says this method is about to replicate data through WAL-files (segment) which is transferred between instances. This is probably the most simple and straightforward method for data replication, but this simplicity comes with price and compromises which also should be accounted.

`LSR` has only async mode. There are multiple reasons for it:
1. The most obvious that it take some time to transfer WAL segments between instances;
2. Another reason is that not all commited transactions may be flushed immediately if `synchronous_commit=off`. In this case there is a possibility that some transactions may be lost if database was shutdown abnormally;
3. Also only archived segments may be transferred. This means that segment will be transferred only when it is full (16 mb by default) and after it was archived. There is a settings which may reduce a time for archiving: `archive_timeout`.

Because `LSR` is async standby replica may be not fully synced with primary instance and because of it such a replica also called `warm standby`.

As `LSR` is a physical replication there are some general limitations and recommendations:
1. Major versions of PostgreSQL should be the same on primary and standby servers. Minor versions may differ;
2. Hardware architecture must be the same;
3. It is a good practice to create primary and standby servers as similar as possible.

Theory is good, but let's try it on practice now.

# Setup <a name="setup"></a>
For simple sample we will setup two PostgreSQL instances: one as a primary servers and another one as a standby. We will create new clusters for each instances but if you already have some configured PostgreSQL instances, then you can skip commands like `initdb -D <path>`.

## Primary server <a name="primary server"></a>
First of all we need to init new database:
```sql
initdb -D /tmp/primary
```

After database initialization we should update its' `postgresql.conf`:
```
wal_level=replica (or higher)
archive_mode=on
archive_command='test ! -f /tmp/wal_archive/%f && cp %p /tmp/wal_archive/%f'
```

The setting `archive_command` is the key for replication. In this example we will archive WAL segments into some temporary directory, but in a production you can use any filesystem such as hdfs, s3, local and so on. Using hdfs or s3 also give you replication out-of-the-box for your WAL segments which is good for reliability. Don't forget to create directory for WAL archive:
```
mkdir /tmp/wal_archive
```

Now we are ready to start primary server:
```
pg_ctl -D /tmp/primary -o "-p 5432" -l /tmp/primary/logs start
```

To connect to the created primary instance you can use this command:
```
psql -h localhost -p 5432 -d postgres
```

As we didn't setup any user or database by default we can connect to the postgres database without any explicit credentials.

After connection to the new primary is established we can create something:
```sql
CREATE TABLE t1(
  id BIGSERIAL PRIMARY KEY,
  field TEXT NOT NULL
);

INSERT INTO t1(field) VALUES ('test');
```

Currently, directory `/tmp/wal_archive` is empty. This is because nothing was archived yet. In the meantime in the directory `/tmp/primary/pg_wal` will be at least two names:
- `archive_status` -- directory with names of WAL segments which are ready to be archived
- WAL segment

As earlier was mentioned by default WAL segment occupy 16 mb and it will not be archived until it's full. Let's generate some data:
```sql
INSERT INTO t1(field) SELECT ('value_' || i) AS field FROM GENERATE_SERIES(1, 200000) s(i);
```

Now we can see that some WAL segments were archived and saved into the `/tmp/wal_archive`. Let's move on to the replica server setup.

## Replica server <a name="replica"></a>
To setup replica we should backup it from the primary instance using [pg_basebackup](https://www.postgresql.org/docs/current/app-pgbasebackup.html) tool:
```sh
pg_basebackup -D /tmp/replica -R -h localhost -p 5432
```

and then configure it by updating `postgresql.conf`:
```
restore_command = 'cp /tmp/wal_archive/%f %p'
archive_cleanup_command = 'pg_archivecleanup /tmp/wal_archive %r'
```
[pg_basebackup](https://www.postgresql.org/docs/current/app-pgbasebackup.html) also will create a `postgresql.auto.conf` with `primary_conninfo` setting. This is required for streaming replication, but for log shipping we don't need it and we can remove this setting completely.


After all setup steps are done we can start it:
```
pg_ctl -D /tmp/replica -o "-p 5433" -l /tmp/replica/logs start
psql -h localhost -p 5433 -d postgres
```

# Replication in action <a name="action"></a>
Now we have primary and standby instances. Let's try to execute some queries and check what will happen.
First of all, let's check that backup was successful and at least rows count is the same on primary and standby:
```sql
SELECt COUNT(*) FROM t1;
```
On both servers result should be the same. Now we create ten records on primary (keep in mind that standby is read-only):
```sql
INSERT INTO t1(field) SELECT ('value_' || i) AS field FROM GENERATE_SERIES(1, 10) s(i);
```

After this query primary and standby will have different rows count. This is because WAL segment wasn't archived because it hasn't filled up yet.

Now we will create more enough records in primary instance to fill in WAL segment:
```sql
INSERT INTO t1(field) SELECT ('value_' || i) AS field FROM GENERATE_SERIES(1, 300000) s(i);
```

This should be enough to fill 16 mb of data and trigger WAL segment archivation. If we execute `count` query again we still may see that results are different but on standby it was also updated. Results are different because some updated were included in new WAL segment which is not archived yet. This is a `replication lag` which we discussed at the beginning. Replication lag may be reduced using streaming replication

# Standby promotion <a name="standby-promotion"></a>
In some cases you need to promote standby to primary. This may happen if primary server is down for  along time or, for example, your system can't operate normally if database was shutdown even for a short time and you have to promote standby fast. In this case you may just execute this query in standby:
```sql
SELECT pg_promote();
```

After this query standby instance became a new primary. To return old primary after promotion standby you need to restore it.

# Conclusion <a name="conclusion"></a>
This was a quick example of how to setup log shipping replication in PostgreSQL. More info could be found in the official [documentation](https://www.postgresql.org/docs/current/warm-standby.html).
