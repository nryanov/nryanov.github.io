---
layout: single
title: "PostgreSQL: how to stream data changes to kafka?"
date: 2024-05-10 04:30:00 +0300
categories: kafka-connect postgres replication cdc debezium logical-replication wal kafka
---

Before diving into details about streaming I want to start from the task which we will try to solve during this article.
Imagine that we have some table which represents user's balance:
```sql
CREATE TABLE user_balances
(
    user_id     BIGINT NOT NULL,
    balance     BIGINT NOT NULL,
    last_update DATE   NOT NULL,
    CONSTRAINT user_balances_pk PRIMARY KEY (user_id)
);
```

Now we want to handle each change of user's balance and according to actual balance notify or not the target user by its id.
How we can achieve this? This is the main question which we will consider and try to find the answer during this article.

# Table of contents
1. [CDC: Change Data Capture](#cdc)
2. [Postgres: NOTIFY](#postgres-notify)
3. [Postgres: outbox pattern](#postgres-outbox)
4. [Step back: how Postgres handle data changes?](#how-postgres-handle-data-changes)
5. [Logical replication and WAL](#logical-replication-and-wal)
6. [Debezium](#debezium)
7. [Real usages of CDC](#cdc-usages)
8. [Conclusion](#conclusion)

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

# Postgres: NOTIFY <a name="postgres-notify"></a>
As all examples will be done using postgres we can consider some specific features of this RDBMS.

`NOTIFY` is one of the [Postgres feature](https://www.postgresql.org/docs/current/sql-notify.html) which allows to make simple interprocess communication.
How to set up this? Firstly, let's start with a Postgres instance. We will use docker for it:
```shell
docker run -p 5432:5432 --name postgres-notify -e POSTGRES_PASSWORD=postgres -e POSTGRES_USER=postgres -e POSTGRES_DB=postgres -d postgres:14
```

And open two connections (in different terminals):
```shell
psql postgresql://postgres:postgres@localhost:5432/postgres
```

`NOTIFY` has some important limitations:
- Despite the fact, the all notifications will be saved in queue until all clients will receive each notification this queue has limited size. When the queue is full then transaction which try to send new notification will fail at the commit point. `pg_notification_queue_usage` allows to get information about queue usage.  