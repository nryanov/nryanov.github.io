---
layout: single
title:  "PostgreSQL: Logical Replication"
date:   2022-02-04 22:50:00 +0300
categories: postgresql replication database high-availability logical-replication
---

# Table of contents
1. [Prerequisite](#prerequisite)
2. [Logical replication](#logical-replication)
3. [Setup (asynchronous mode)](#async-setup)
   1. [Publisher](#async-publisher)
   2. [Subscriber](#async-subscriber)
4. [Setup (synchronous mode)](#sync-setup)
   1. [Publisher](#sync-publicher)
   2. [Subscriber](#sync-subscriber)
   3. [Sync mode](#sync-mode)
5. [Conflict resolution](#conflict-resolution)
6. [Replica identity](#replica-identity)
7. [DDL](#ddl)
8. [Conclusion](#conclusion)

# Prerequisite <a name="prerequisite"></a>
All examples assumes that postgresql is already installed on your machine. 
Also all examples are created using `PostgreSQL 14.1 on aarch64-apple-darwin20.6.0, compiled by Apple clang version 13.0.0 (clang-1300.0.29.3), 64-bit`.

# Logical replication <a name="logical-replication"></a>
Logical replication is another method to replicate data between multiple nodes. This replication uses publish-subscribe model. 
Each publisher may have multiple subscribers and each subscriber can subscribe to multiple publisher. Also, each subscriber may 
be a publisher for another node which make it possible to create a cascading replication.

When new subscription is created it begins from taking the actual state of replicated data (or simply synchronize tables). 
After it's done the changes are sent to the subscriber as they occur in real-time. 
The subscriber applies the data in the same order as the publisher so that transactional consistency is guaranteed for publications 
within a single subscription.

# Setup (asynchronous mode) <a name="async-setup"></a>

## Publisher <a name="async-publisher"></a>
```
-- init new cluster
initdb -D /tmp/postgresql/db1
```

Update postgresql.conf:
- **wal_level**=`logical`
- **max_replication_slots**=`{ >= subscriber count + some reserve }`
- **max_wal_senders**=`{ max_replication_slots + the number of physical replicas that are connected at the same time }`

Update pg_hba.conf (for localhost configuration):
- host    replication    replication_user    127.0.0.1/32    md5

```
-- start cluster
pg_ctl -D /tmp/postgresql/db1 -o "-p 5432" -l /tmp/postgresql/db1/logs start
```

```sql
-- with explicit unique index (primary key)
CREATE TABLE t1
(
    id    int PRIMARY KEY,
    value text NOT NULL
);

-- without unique index
CREATE TABLE t2
(
    id    int,
    value text NOT NULL
);

-- create user for replication
CREATE USER replication_user REPLICATION;
GRANT ALL PRIVILEGES ON t1 TO replication_user;
GRANT ALL PRIVILEGES ON t2 TO replication_user;
-- add new user for replication in pg_hba.conf

-- generate some data
INSERT INTO t1
SELECT i AS id, ('value_' || i) AS value
FROM GENERATE_SERIES(1, 10) s(i);

-- create publication
CREATE PUBLICATION sample_publication FOR TABLE t1, t2 WITH (PUBLISH = 'insert, update, delete, truncate');
-- find existing publications
SELECT * FROM pg_publication;
```

## Subscriber <a name="async-subscriber"></a>
```
-- init new cluster
initdb -D /tmp/postgresql/db2
```

postgresql.conf:
- **max_replication_slots**={ at least the number of subscriptions that will be added to the subscriber, plus some reserve for table synchronization }
- **max_logical_replication_workers**={ at least the number of subscriptions, plus some reserve for the table synchronization }
- **max_worker_processes**={ may need to be adjusted to accommodate for replication workers, **at least (max_logical_replication_workers + 1)**. Note that some extensions and parallel queries also take worker slots from max_worker_processes }

```
-- start cluster
pg_ctl -D /tmp/postgresql/db2 -o "-p 5433" -l /tmp/postgresql/db2/logs start
```

```sql
CREATE SUBSCRIPTION sample_async_subscription
    CONNECTION 'dbname=postgres host=localhost user=replication_user port=5432 application_name=logical_replication_sample'
    PUBLICATION sample_publication;
-- we will get an error because we didn't create necessary tables. Logical replication doesn't replicate DDL:
-- [42P01] ERROR: relation "public.t1" does not exist

CREATE TABLE t1
(
    id    int PRIMARY KEY,
    value text NOT NULL
);

CREATE TABLE t2
(
    id    int,
    value text NOT NULL
);
-- now we can create a subscription
CREATE SUBSCRIPTION sample_async_subscription
    CONNECTION 'dbname=postgres host=localhost user=replication_user port=5432 application_name=logical_replication_sample'
    PUBLICATION sample_publication;

-- table t1 will be synchronized automatically:
SELECT * FROM t1;
```

That's it -- the basic logical replication setup is done.

# Setup (synchronous mode) <a name="sync-setup"></a>

## Publisher <a name="sync-publisher"></a>
```
initdb -D /tmp/postgresql/db1
```

postgresql.conf:
- **wal_level**=`logical`
- **max_replication_slots**=`{ >= subscriber count + some reserve }`
- **max_wal_senders**=`{ max_replication_slots + the number of physical replicas that are connected at the same time }`
- **synchronous_commit**=`on` -- this is a default value

pg_hba.conf (for localhost configuration):
- host    replication    replication_user    127.0.0.1/32    md5

```
pg_ctl -D /tmp/postgresql/db1 -o "-p 5432" -l /tmp/postgresql/db1/logs start
```

After initial setup we will create a table and publication:
```sql
CREATE TABLE t1
(
    id    int PRIMARY KEY,
    value text NOT NULL
);

-- create user for replication
CREATE USER replication_user REPLICATION;
GRANT ALL PRIVILEGES ON t1 TO replication_user;
-- add new user for replication in pg_hba.conf

-- generate some data
INSERT INTO t1
SELECT i AS id, ('value_' || i) AS value
FROM GENERATE_SERIES(1, 10) s(i);

-- create publication
CREATE PUBLICATION sample_publication FOR TABLE t1 WITH (PUBLISH = 'insert, update, delete, truncate');
```

## Subscriber <a name="sync-subscriber"></a>
```
initdb -D /tmp/postgresql/db2
```

postgresql.conf:
- **max_replication_slots**={ at least the number of subscriptions that will be added to the subscriber, plus some reserve for table synchronization }
- **max_logical_replication_workers**={ at least the number of subscriptions, plus some reserve for the table synchronization }
- **max_worker_processes**={ may need to be adjusted to accommodate for replication workers, **at least (max_logical_replication_workers + 1)**. Note that some extensions and parallel queries also take worker slots from max_worker_processes }

```
pg_ctl -D /tmp/postgresql/db2 -o "-p 5433" -l /tmp/postgresql/db2/logs start
```

```sql
CREATE TABLE t1
(
    id    int PRIMARY KEY,
    value text NOT NULL
);

CREATE SUBSCRIPTION sample_sync_subscription
    CONNECTION 'dbname=postgres host=localhost user=replication_user port=5432 application_name=replica_1'
    PUBLICATION sample_publication;
```

Note that in connection we use explicit application name: `application_name=replica_1`. 
By default publication name will be used as replica's name for synchronous mode. 

## Sync mode <a name="sync-mode"></a>
```sql
-- master
SELECT application_name, sync_state FROM pg_stat_replication;
-- [replica_1, async]
```

Initially, as we didn't specify standby names in `postgresql.conf` this publication is async. Let's change it by updating `postgresql.conf` on master:
- **synchronous_standby_names** = `'replica_1'`

Now reload config (no need for restart) and check state of replication:
```sql
-- master
SELECT pg_reload_conf(); -- reload config
SELECT application_name, sync_state FROM pg_stat_replication;
-- [replica_1, sync]

-- master
INSERT INTO t1 VALUES (11, 'value_11');
```

Now, each query will be committed only if all listed standby replicas also confirm transaction. This could be changed using `FIRST {number}` or `ANY {number}` modifiers 
to makes transaction commits wait until their WAL records are replicated to listed replicas. More information could be found in [docs](https://www.postgresql.org/docs/current/warm-standby.html#SYNCHRONOUS-REPLICATION).

If we stop replica:
```
pg_ctl -D /tmp/postgresql/db2 -o "-p 5433" -l /tmp/postgresql/db2/logs stop
```

And try to insert something new on master:
```sql
-- master
INSERT INTO t1 VALUES (12, 'value_12');
```

Transaction commit will wait until replica will be alive again:
```
pg_ctl -D /tmp/postgresql/db2 -o "-p 5433" -l /tmp/postgresql/db2/logs start
```

# Conflict resolution <a name="conflict-resolution"></a>
As replica doesn't need to be a read-only, it is possible to modify replicated table. Some modification may cause a conflicts in future
if master will have transaction which, for example, insert a row with already existing PK on replicated table:

```sql
-- replica:
INSERT INTO t1 VALUES (11, 'value_11'); -- (1)

-- master
INSERT INTO t1 VALUES (11, 'value_11'); -- (2)
INSERT INTO t1 VALUES (12, 'value_12'); -- (3)
INSERT INTO t1 VALUES (13, 'value_13'); -- (4)
INSERT INTO t1 VALUES (14, 'value_14'); -- (5)
```

Now master will have 14 rows while replica only 11. This is because `(2)` wasn't replicated due to conflict. 
Also replica's subscription will be stopped:
```sql
-- master
SELECT * FROM pg_stat_replication;
```

And in logs you can found something like that:
```text
2022-02-03 22:17:28.008 [3828] ERROR:  duplicate key value violates unique constraint "t1_pkey"
2022-02-03 22:17:28.008 [3828] DETAIL:  Key (id)=(11) already exists.
2022-02-03 22:17:28.009 [3194] LOG:  background worker "logical replication worker" (PID 3828) exited with exit code 1
2022-02-03 22:17:28.013 [3839] LOG:  logical replication apply worker for subscription "sample_async_subscription" has started
```
This will happen until conflict is fixed.

This conflict should be fixed manually:
- delete conflicting data in replica's side
- skip conflicting transaction

First solution is the simplest but only if you know exactly which rows cause a conflict:
```sql
DELETE FROM t1 WHERE id = 11;
-- restart subscription
ALTER SUBSCRIPTION {subscription_name} DISABLE;
ALTER SUBSCRIPTION {subscription_name} ENABLE;
```

Now let's consider the second solution: skipping conflicting transaction. To do this we need to know lsn to advance to.
Find the needed lsn by inspecting wal files or just get the current number:
```sql
-- master
select pg_current_wal_lsn();
```
And then on replica:
```sql
-- replica
SELECT 'pg_' || oid as origin FROM pg_subscription where subname = '{subscription_name}';

ALTER SUBSCRIPTION {subscription_name} DISABLE;
SELECT PG_REPLICATION_ORIGIN_ADVANCE('{origin}', '{lsn}');
ALTER SUBSCRIPTION {subscription_name} ENABLE;
```

# Replica identity <a name="replica-identity"></a>
Previous examples were done using `t1` table which has explicit primary key. Table `t2` doesn't have any unique indexes, but logical replication 
requires it: 
```sql
INSERT INTO t2 VALUES (1, 'value_1'); -- (1) 
-- update will cause error
UPDATE t2 SET value = 'updated_' || t2.id WHERE t2.id = 1; -- (2)
-- [55000] ERROR: cannot update table "t2" because it does not have a replica identity and publishes updates
-- Hint: To enable updating the table, set REPLICA IDENTITY using ALTER TABLE.
ALTER TABLE t2 REPLICA IDENTITY FULL; -- (3)
UPDATE t2 SET value = 'updated_' || t2.id WHERE t2.id = 1; -- (4)
```
Operation `(3)` will update table and set replica identity as a full row. This is not efficient, but now it is possible to make updates and deletes
to the `t2`. Possible values for replica identity:
```sql
ALTER TABLE {tablename} REPLICA IDENTITY { DEFAULT | USING INDEX index_name | FULL | NOTHING }
```

# DDL <a name="ddl"></a>
As mentioned before, table schema should be equal on master and replica and should be synchronized manually because logical replication does not replicate DDL operations.
Let's condier an example:
```sql
-- master
CREATE TABLE t3
(
    id    int PRIMARY KEY,
    field_1 text NOT NULL,
    field_2 text NOT NULL
);

INSERT INTO t3
SELECT i AS id, ('field_1_' || i) AS field_1, ('field_2_' || i) AS field_2
FROM GENERATE_SERIES(1, 10) s(i);

CREATE PUBLICATION {publication_name} FOR TABLE t3 WITH (PUBLISH = 'insert, update, delete, truncate');

-- replica
CREATE TABLE t3
(
    id    int PRIMARY KEY,
    field_1 text NOT NULL
);

CREATE SUBSCRIPTION {subscription_name}
    CONNECTION 'dbname=postgres host=localhost user=replication_user port=5432 application_name=replica_1'
    PUBLICATION {publication_name};
```

In this setup replica will not get any rows because it will stuck on table synchronization step.
Let's fix it:
```sql
-- replica
ALTER TABLE t1 ADD COLUMN field_2 text NOT NULL DEFAULT '';
ALTER SUBSCRIPTION {subscription_name} ENABLE;
```

Now replication will continue to work.

# Conclusion <a name="conclusion"></a>
So that was a brief run through the logical replication setup and how to solve some issues. 
There is still a lot of information which could also be described such as parameters for fine tuning, cascading replication and so on, 
but the main purpose of this article was to introduce [logical replication](https://www.postgresql.org/docs/current/logical-replication.html) as another one technique for data replication in PostgreSQL.
