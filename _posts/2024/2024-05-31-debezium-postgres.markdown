---
layout: single
title: "PostgreSQL: Log-based CDC using debezium"
date: 2024-05-10 04:30:00 +0300
categories: postgres replication cdc debezium logical-replication wal
---

# Table of contents
1. [Introduction](#introduction)
2. [CDC: Change Data Capture](#cdc)
3. [How Postgres handle data changes?](#how-postgres-handle-data-changes)
4. [Logical replication and WAL](#logical-replication-and-wal)
5. [Debezium](#debezium)
6. [Conclusion](#conclusion)

# Introduction <a name="introduction"></a>
In this little article I'll show different ways to set up debezium for log-based CDC.
Before diving into details about debezium, I'll shortly describe CDC and why it may be helpful in some tasks. 

# CDC: Change Data Capture <a name="cdc"></a>
In the Internet the `CDC` is described as a design pattern which allows to track data changes (deltas).
Let's consider this approach on table `user_balances`. Initial state of table is:

```sql
| user_id | balance | last_updated | 
| 1       | 100     | 2024-01-01   | 
| 2       | 100     | 2024-01-01   | 
| 3       | 100     | 2024-01-01   | 
```

We have three users and all of them have the same balances. After some time the table content has changed to:
```sql
| user_id | balance | last_updated | 
| 1       | 200     | 2024-01-02   | 
| 2       | 150     | 2024-01-03   | 
| 3       | 50      | 2024-01-04   | 
| 4       | 500     | 2024-01-05   | 
```

Each user were updated and also the new one was added. The question is `how to track these changes`? 
Imagine that you develop some kind of audit application, and you have to track each change of this table.
Your goal will be to get something like that:
```sql
| __operation__ | __timetamp__  | user_id | balance | last_updated | 
| UPDATE        | 2024-01-02    | 1       | 200     | 2024-01-02   |
| UPDATE        | 2024-01-03    | 2       | 150     | 2024-01-03   |
| UPDATE        | 2024-01-04    | 3       | 50      | 2024-01-04   |
| CREATE        | 2024-01-05    | 4       | 500     | 2024-01-05   | 
```

And `CDC` can help you with it. Let's move on and consider different ways to achieve this.

# How Postgres handle data changes? <a name="how-postgres-handle-data-changes"></a>
Postgres ensures durability using [WAL](https://www.postgresql.org/docs/current/wal-internals.html) (write-ahead log). WAL allows to recover from failures and using WAL files it is also possible to replicate data to other replicas.
In free interpretation, replication based on WAL is possible because each WAL file contains committed changes of each transaction. 
The last point that WAL allows to replicate data is crucial. As we know that each change is written in WAL we can somehow use it for CDC.

# Logical replication and WAL <a name="logical-replication-and-wal"></a>
WAL is enabled and to use it nothing more should be done. However, to be able to use WAL in a replication terms some additional setup must be done.
The most interesting setting for our case is `wal_level`. There are other settings which also important but for simplicity we'll consider only this one.
[wal_level](https://www.postgresql.org/docs/current/runtime-config-wal.html) may be set as:
- minimal
- replica
- logical

`minimal` level generates information that allow to recover DB from crash, but it's not sufficient to any kind of replication. `replica` level is enough 
to set up physical replication (e.g. log-shipping or streaming), due to WAL now will include information required for it, but it's still not enough for out task. Only one level is remain, and it's `logical` which as expected will help us to set up log-based CDC.
But why `replica` level wasn't enough?

The reason why we need `logical` level is because for log-based CDC we need information about updates not on physical level but on logical. 
In other words and very simplified, `replica` level generates records in WAL which contains information about `where` data was changed (like concrete pages), but `logical` contains information about `what` data was changed.

`logical` level is based on the same WAL files, but as it contains additional information data can be decoded using special plugins (like pgoutput, decoderbufs and others) 
and decoded data will contain:
- operation
- row identifier*
- before
- after

`operation` indicate `how` this event was generated: `DELETE`, `CREATE`, `UPDATE` and in some cases `TRUNCATE`. `row identifier` may be considered as a primary key of a concrete tuple which was updated
but that's not always like that. `logical` level requires that tuples should be distinguishable (because as we remember this info is used to indicate `what` data was changed and not `where`), 
and usually it is achieved using PK, but some tables may not have PK. In such cases unique indexes may be used (all columns should have `NOT NULL` constraint) or if
nor PK, nor unique index exists, full row must be used as identifier. Keep in mind that if full row is used as identifier it's also affect 
amount of data generated in WAL. Because of that it is highly recommended to avoid such cases. Postgres allows to choose what should be used as a row identifier per table:
```sql
ALTER TABLE {table} REPLICA IDENTITY DEFAULT -- when PK exists
ALTER TABLE {table} REPLICA IDENTITY USING INDEX {index} -- when unique index exists
ALTER TABLE {table} REPLICA IDENTITY FULL -- when nor PK, not unique index exist
```

Finally, `before` and `after` contains information about how row looked before operation and after it. 
Looking ahead, `before` not always included (for DEFAULT and INDEX bases replica identity). Also, not every column will be presented in `after` if it wasn't changed (TOASTed columns).

# Debezium <a name="debezium"></a>


# Conclusion <a name="conclusion"></a>